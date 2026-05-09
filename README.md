// CathackLua.cpp — Lua 5.4 scripting host for the cheat.
//
// Main moving pieces:
//  - g_L          : the only lua_State. Created once in Init(); torn down in Shutdown().
//  - g_mu         : mutex serializing all access to g_L.
//  - g_scripts    : in-memory script registry (filename, source, autoRun flag, last error).
//  - g_console    : ring buffer of strings produced by print() and cathack.print().
//  - g_tickRefs / g_renderRefs : Lua registry references (LUA_REGISTRYINDEX) for callback fns.
//  - tl_currentDrawList : thread-local pointer set during TickRender(), used by cathack.draw_*.
//
// Sandbox notes:
//  - lua sources for loadlib (require) and loslib (os.execute / os.exit) were deleted from
//    the project, so even if a script tries to access them it'll fail at parse time when
//    the global doesn't exist.
//  - debug, io, coroutine, table, string, math, utf8 are all live.
//  - No dofile() / loadfile() / io.popen restrictions beyond that — scripts that want them
//    can have them. Anyone running a Lua script from the menu has already trusted it.
//
// Threading:
//  - g_mu is held for the entire duration of every public call. Lua VM is not re-entrant
//    so this is mandatory. Callbacks fire entirely under the mutex; if a callback calls
//    a long-running C function we lock everything else out for that long. In practice
//    on_tick fires at ~60 Hz and on_render fires at the framerate, both with very light
//    bodies, so contention is invisible.

#include <Windows.h>
#include <ShlObj.h>
#include <KnownFolders.h>
#include <winhttp.h>
#pragma comment(lib, "winhttp.lib")

#include <algorithm>
#include <atomic>
#include <cctype>
#include <chrono>
#include <climits>
#include <cmath>
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <deque>
#include <filesystem>
#include <fstream>
#include <memory>
#include <mutex>
#include <sstream>
#include <string>
#include <thread>
#include <unordered_map>
#include <unordered_set>
#include <vector>

#include "imgui.h"
#include "Cheats.h"
#include "Utils.h"
#include "Engine.h"
#include "CathackLua.h"

#include "Lua/lua.hpp"

#pragma comment(lib, "shell32.lib")

// === CathackHostBridge — defined in DLLMain.cpp ===========================
// Lua scripts ask the host (the cheat menu / WndProc) for things the Lua
// VM can't know on its own: whether the main menu is open, whether any
// script has requested mouse/keyboard capture this frame, and the queue of
// WM_CHAR codepoints typed since the last render frame. Forward-declared
// here so we don't need a separate header for these handful of functions.
//
// Capture-flag contract (mirrors ImGui::SetNextFrameWantCaptureMouse):
//   - Scripts call cathack.capture_mouse(true) inside on_render every frame
//     they want to claim the mouse. We reset the flag to false at the
//     START of every render frame so scripts that crash or stop calling
//     don't leave the game permanently input-blocked.
//   - WndProc reads the flag (atomically) when a mouse message arrives and
//     swallows the message instead of forwarding to the game.
namespace CathackHostBridge
{
	extern std::atomic<bool>   g_LuaCaptureMouseReq;
	extern std::atomic<bool>   g_LuaCaptureKbdReq;
	bool                       IsMenuOpen();
	void                       SetMenuOpen(bool);
	void                       ToggleMenu();
	std::vector<unsigned int>  DrainTypedChars();

	// Unified input event — kind 0 = char codepoint, 1 = WM_KEY*DOWN,
	// 2 = WM_KEY*UP. Mirrors the struct defined in DLLMain.cpp.
	struct LuaInputEvent
	{
		int          kind;
		unsigned int code;
		bool         repeated;
		bool         shift;
		bool         ctrl;
		bool         alt;
	};
	std::vector<LuaInputEvent> DrainInputEvents();
}

// ====================================================================
// Module-local state
// ====================================================================

namespace
{
	std::recursive_mutex            g_mu;
	lua_State*                      g_L = nullptr;
	// Each callback registration carries an ownership tag (the script name
	// that registered it, or empty for "registered outside any script run").
	// Stop(name) sweeps both lists by owner so a script's hooks actually
	// stop firing — used to be a footgun where Stop only flipped the UI
	// indicator while callbacks kept ticking.
	struct Callback
	{
		int         ref;        // LUA_REGISTRYINDEX reference
		std::string owner;      // script name (empty = registered from outside Run)
	};
	std::vector<Callback>           g_tickRefs;
	std::vector<Callback>           g_renderRefs;
	// Event-fan callback lists. Populated by the on_key_pressed /
	// on_mouse_* / on_char registration functions below; declared up
	// front so RemoveCallbacksByOwner_locked / unhook / clear_callbacks
	// (which all live further down) can reference them. Each fan
	// fires exactly once per matching event in TickRender.
	std::vector<Callback>           g_keyPressedRefs;
	std::vector<Callback>           g_keyReleasedRefs;
	std::vector<Callback>           g_mousePressedRefs;
	std::vector<Callback>           g_mouseReleasedRefs;
	std::vector<Callback>           g_mouseWheelRefs;
	std::vector<Callback>           g_charRefs;
	std::vector<CathackLua::ScriptEntry> g_scripts;
	std::deque<std::string>         g_console;
	constexpr size_t                kConsoleMax = 512;
	std::atomic<bool>               g_autoLoadEnabled{true};
	std::atomic<bool>               g_initialized{false};

	// Set during TickRender(). cathack.draw_* read this. Always nullptr outside
	// the render hook so on_tick / sync calls can't accidentally draw.
	thread_local ImDrawList*        tl_currentDrawList = nullptr;

	// Tracks which script is currently being run (for error reporting AND
	// callback ownership tagging). cathack.on_tick / on_render copy this
	// into the registered Callback's `owner` field at registration time
	// so Stop(name) can find them later.
	std::string                     g_runningScript;

	// Last tick timestamp (ms) — used to compute dt for on_tick and to
	// rate-limit the tick fan to ~16 ms regardless of how often the
	// game-thread caller invokes us.
	uint64_t                        g_lastTickMs = 0;

	// === Watchdog ==================================================
	// A wall-clock budget for any single Lua call (chunk run OR a single
	// callback). When the lua_sethook count-hook fires, it compares the
	// current time against `g_callDeadlineMs`; if we've blown past it,
	// luaL_error longjmps out of the running chunk so the game/render
	// thread isn't stuck. Each call site sets the deadline before invoking
	// pcall and clears it after.
	std::atomic<long long>          g_callDeadlineMs{0};
	// Soft cap on per-call wall time. 250ms is generous for legit work
	// (drawing 1000s of widgets, walking 100s of entities) but small
	// enough that a `while true do end` doesn't freeze a session.
	constexpr long long             kCallBudgetMs = 250;

	// ====================================================================
	// Path helpers — every script lives under %APPDATA%\cathack\lua\.
	// Layout:
	//   %APPDATA%\cathack\lua\<name>.lua
	// ====================================================================

	std::filesystem::path GetLuaDirectory()
	{
		PWSTR pAppData = nullptr;
		std::filesystem::path base;
		if (SUCCEEDED(SHGetKnownFolderPath(FOLDERID_RoamingAppData, 0, nullptr, &pAppData)) && pAppData)
		{
			base = pAppData;
			CoTaskMemFree(pAppData);
		}
		else
		{
			// Fallback to %APPDATA% env var or CWD.
			wchar_t buf[MAX_PATH] = {};
			if (GetEnvironmentVariableW(L"APPDATA", buf, MAX_PATH))
				base = buf;
			else
				base = std::filesystem::current_path();
		}
		base /= L"cathack";
		base /= L"lua";
		std::error_code ec;
		std::filesystem::create_directories(base, ec); // best effort
		return base;
	}

	bool IsValidScriptName(const std::string& name)
	{
		if (name.empty() || name.size() > 96) return false;
		for (char c : name)
		{
			// Restrict to a subset that's safe as a filename component.
			if (!(std::isalnum(static_cast<unsigned char>(c)) || c == '_' || c == '-' || c == '.'))
				return false;
		}
		// Don't allow leading dot or "..".
		if (name[0] == '.' || name == "..") return false;
		return true;
	}

	std::filesystem::path ScriptPath(const std::string& name)
	{
		return GetLuaDirectory() / (name + ".lua");
	}

	std::string ReadFileAll(const std::filesystem::path& path)
	{
		std::ifstream f(path, std::ios::binary);
		if (!f) return std::string();
		std::ostringstream ss;
		ss << f.rdbuf();
		return ss.str();
	}

	bool WriteFileAll(const std::filesystem::path& path, const std::string& content)
	{
		std::error_code ec;
		std::filesystem::create_directories(path.parent_path(), ec);
		std::ofstream f(path, std::ios::binary | std::ios::trunc);
		if (!f) return false;
		f.write(content.data(), static_cast<std::streamsize>(content.size()));
		return f.good();
	}

	// Detect "-- @autorun" in the first ~20 lines of a script. We use this as
	// the persisted "auto-run on inject" flag instead of a sidecar metadata
	// file — keeps everything in the .lua file the user already has open.
	bool DetectAutoRunDirective(const std::string& source)
	{
		const std::string needle = "@autorun";
		size_t scan = source.size() < 4096 ? source.size() : 4096;
		std::string head = source.substr(0, scan);
		// Lowercase for easier matching.
		std::string low; low.reserve(head.size());
		for (char c : head) low.push_back(static_cast<char>(std::tolower(static_cast<unsigned char>(c))));
		size_t p = low.find(needle);
		if (p == std::string::npos) return false;
		// Must appear inside a Lua comment ("--" earlier on the same line).
		size_t lineStart = low.rfind('\n', p);
		lineStart = (lineStart == std::string::npos) ? 0 : lineStart + 1;
		size_t commentPos = low.find("--", lineStart);
		return (commentPos != std::string::npos && commentPos < p);
	}

	// Strip / inject the "-- @autorun" line at the top of the script when the
	// user toggles the flag in the UI. We rewrite it as the very first line
	// for clarity. Returns the modified source.
	std::string SetAutoRunDirective(const std::string& source, bool autoRun)
	{
		// First, strip any existing "@autorun" line (case-insensitive). We
		// only consider the first 20 lines.
		std::vector<std::string> lines;
		{
			std::stringstream ss(source);
			std::string line;
			while (std::getline(ss, line)) lines.push_back(line);
			// std::getline drops the trailing \n; if source ends with \n we
			// preserve it on the final newline-join below.
		}
		const bool sourceEndsWithNewline = !source.empty() && source.back() == '\n';

		auto isAutoRunLine = [](const std::string& l) {
			std::string low; low.reserve(l.size());
			for (char c : l) low.push_back(static_cast<char>(std::tolower(static_cast<unsigned char>(c))));
			size_t commentPos = low.find("--");
			if (commentPos == std::string::npos) return false;
			size_t arPos = low.find("@autorun", commentPos);
			return arPos != std::string::npos;
		};

		const size_t scanLines = lines.size() < 20 ? lines.size() : 20;
		for (size_t i = 0; i < scanLines; ++i)
		{
			if (isAutoRunLine(lines[i]))
			{
				lines.erase(lines.begin() + i);
				break;
			}
		}

		if (autoRun)
		{
			// Prepend the directive as line 1.
			lines.insert(lines.begin(), "-- @autorun");
		}

		std::string out;
		for (size_t i = 0; i < lines.size(); ++i)
		{
			out += lines[i];
			if (i + 1 < lines.size() || sourceEndsWithNewline) out += '\n';
		}
		return out;
	}

	// ====================================================================
	// Console plumbing
	// ====================================================================

	void PushConsole(const std::string& s)
	{
		g_console.push_back(s);
		if (g_console.size() > kConsoleMax)
			g_console.pop_front();
	}

	// Lua: print(...) — replaces the default print so it goes into our console
	// instead of stdout (which the game discards).
	int l_print(lua_State* L)
	{
		const int n = lua_gettop(L);
		std::string out;
		for (int i = 1; i <= n; ++i)
		{
			size_t len;
			const char* s = luaL_tolstring(L, i, &len);
			if (i > 1) out.push_back('\t');
			out.append(s, len);
			lua_pop(L, 1);
		}
		PushConsole(out);
		return 0;
	}

	// ====================================================================
	// Setting binding table — name -> getter/setter pair against the
	// existing C++ globals (CVars, AimbotSettings, etc.).
	//
	// Each binding is one of: bool, float, int. The Lua API uses
	// cathack.set(name, value) and cathack.get(name) and dispatches via
	// the binding type.
	//
	// To add a new setting, just add a row to kSettingBindings.
	// ====================================================================

	enum class BindKind { Bool, Float, Int };

	struct SettingBinding
	{
		const char* name;
		BindKind    kind;
		void*       addr;
		float       fmin = 0.f;
		float       fmax = 0.f; // 0,0 = no clamp
	};

	// Note on ordering: anything the user is likely to bind to first should
	// appear before the deeper menu-only knobs. We don't need to expose
	// every setting — just the gameplay-affecting ones. Users who want
	// more can ask, easy to add a row.
	const SettingBinding kSettingBindings[] = {
		// Top-level toggles
		{"aimbot",            BindKind::Bool,  &CVars.Aimbot},
		{"esp",               BindKind::Bool,  &CVars.ESP},
		{"silent_aim",        BindKind::Bool,  &CVars.SilentAim},
		{"trigger_bot",       BindKind::Bool,  &CVars.TriggerBot},
		{"god_mode",          BindKind::Bool,  &CVars.GodMode},
		{"no_recoil",         BindKind::Bool,  &CVars.NoRecoil},
		{"no_spread",         BindKind::Bool,  &CVars.NoSpread},
		{"no_sway",           BindKind::Bool,  &CVars.NoWeaponSway},
		{"wall_penetration",  BindKind::Bool,  &CVars.WallPenetration},
		{"no_clip",           BindKind::Bool,  &CVars.NoClip},
		{"speed_enabled",     BindKind::Bool,  &CVars.SpeedEnabled},
		{"speed",             BindKind::Float, &CVars.Speed,                 0.0f, 20.0f},
		{"fov_changer",       BindKind::Bool,  &CVars.FOVChanger},
		{"fov",               BindKind::Float, &CVars.FOV,                   30.0f, 175.0f},
		{"spam_yell",         BindKind::Bool,  &CVars.SpamYell},
		{"jerk_off",          BindKind::Bool,  &CVars.JerkOff},
		{"bullet_time",       BindKind::Bool,  &CVars.BulletTime},

		// Aimbot
		{"aimbot.fov",                 BindKind::Float, &AimbotSettings.MaxFOV,           1.0f, 180.0f},
		{"aimbot.distance_max",        BindKind::Float, &AimbotSettings.MaxDistance,      0.0f, 1000.0f},
		{"aimbot.distance_min",        BindKind::Float, &AimbotSettings.MinDistance,      0.0f, 1000.0f},
		{"aimbot.los",                 BindKind::Bool,  &AimbotSettings.LOS},
		{"aimbot.smooth",              BindKind::Bool,  &AimbotSettings.Smooth},
		{"aimbot.smoothing",           BindKind::Float, &AimbotSettings.SmoothingVector,  1.0f, 200.0f},
		{"aimbot.target_civilians",    BindKind::Bool,  &AimbotSettings.TargetCivilians},
		{"aimbot.target_dead",         BindKind::Bool,  &AimbotSettings.TargetDead},
		{"aimbot.target_arrested",     BindKind::Bool,  &AimbotSettings.TargetArrested},
		{"aimbot.target_surrendered",  BindKind::Bool,  &AimbotSettings.TargetSurrendered},
		{"aimbot.target_all",          BindKind::Bool,  &AimbotSettings.TargetAll},
		{"aimbot.require_key",         BindKind::Bool,  &AimbotSettings.RequireKeyHeld},

		// Silent aim
		{"silent.fov",                 BindKind::Float, &SilentAimSettings.MaxFOV,        1.0f, 180.0f},
		{"silent.hit_chance",          BindKind::Float, &SilentAimSettings.HitChance,     0.0f, 100.0f},
		{"silent.los",                 BindKind::Bool,  &SilentAimSettings.RequiresLOS},
		{"silent.target_civilians",    BindKind::Bool,  &SilentAimSettings.TargetCivilians},
		{"silent.target_dead",         BindKind::Bool,  &SilentAimSettings.TargetDead},
		{"silent.target_arrested",     BindKind::Bool,  &SilentAimSettings.TargetArrested},
		{"silent.target_surrendered",  BindKind::Bool,  &SilentAimSettings.TargetSurrendered},
		{"silent.target_all",          BindKind::Bool,  &SilentAimSettings.TargetAll},
		{"silent.method",              BindKind::Int,   &SilentAimSettings.AimOutputMode, 0.0f, 3.0f},

		// Chams
		{"chams.viewmodel",            BindKind::Bool,  &ChamsSettings.ViewmodelEnabled},
		{"chams.world",                BindKind::Bool,  &ChamsSettings.WorldEnabled},
		{"chams.through_walls",        BindKind::Bool,  &ChamsSettings.RenderThroughWalls},
		{"chams.suspects",             BindKind::Bool,  &ChamsSettings.WorldChamSuspects},
		{"chams.civilians",            BindKind::Bool,  &ChamsSettings.WorldChamCivilians},
		{"chams.arrested",             BindKind::Bool,  &ChamsSettings.WorldChamArrested},
		{"chams.surrendered",          BindKind::Bool,  &ChamsSettings.WorldChamSurrendered},
		{"chams.dead",                 BindKind::Bool,  &ChamsSettings.WorldChamDead},
		{"chams.teammates",            BindKind::Bool,  &ChamsSettings.WorldChamTeammates},

		// Viewmodel
		{"viewmodel.enabled",          BindKind::Bool,  &ViewmodelSettings.Enabled},
		{"viewmodel.forward",          BindKind::Float, &ViewmodelSettings.OffsetForward},
		{"viewmodel.right",            BindKind::Float, &ViewmodelSettings.OffsetRight},
		{"viewmodel.up",               BindKind::Float, &ViewmodelSettings.OffsetUp},
		{"viewmodel.pitch",            BindKind::Float, &ViewmodelSettings.RotPitch},
		{"viewmodel.yaw",              BindKind::Float, &ViewmodelSettings.RotYaw},
		{"viewmodel.roll",             BindKind::Float, &ViewmodelSettings.RotRoll},
		{"viewmodel.fov_override",     BindKind::Bool,  &ViewmodelSettings.FOVOverride},
		{"viewmodel.fov",              BindKind::Float, &ViewmodelSettings.FOV,           1.0f, 175.0f},

		// Misc / privacy
		{"reticle",                    BindKind::Bool,  &MiscSettings.Reticle},
		{"cross_reticle",              BindKind::Bool,  &MiscSettings.CrossReticle},
		{"exclude_from_capture",       BindKind::Bool,  &MiscSettings.ExcludeFromCapture},

		{nullptr, BindKind::Bool, nullptr},
	};

	const SettingBinding* FindBinding(const char* name)
	{
		for (const SettingBinding* b = kSettingBindings; b->name; ++b)
		{
			if (std::strcmp(b->name, name) == 0) return b;
		}
		return nullptr;
	}

	// ====================================================================
	// Lua API: cathack.* functions
	// ====================================================================

	int l_cathack_print(lua_State* L)
	{
		const int n = lua_gettop(L);
		std::string out;
		for (int i = 1; i <= n; ++i)
		{
			size_t len;
			const char* s = luaL_tolstring(L, i, &len);
			if (i > 1) out.push_back(' ');
			out.append(s, len);
			lua_pop(L, 1);
		}
		PushConsole(out);
		return 0;
	}

	int l_cathack_notify(lua_State* L)
	{
		const char* msg = luaL_checkstring(L, 1);
		bool force = lua_toboolean(L, 2) != 0;
		// Push a toast. Force=true ignores the user's "toasts disabled" setting.
		// CathackNotifications namespace is declared in Cheats.h.
		CathackNotifications::Push(msg, force);
		return 0;
	}

	int l_cathack_set(lua_State* L)
	{
		const char* name = luaL_checkstring(L, 1);
		const SettingBinding* b = FindBinding(name);
		if (!b) return luaL_error(L, "cathack.set: unknown setting '%s'", name);

		switch (b->kind)
		{
		case BindKind::Bool:
		{
			bool v = lua_toboolean(L, 2) != 0;
			*static_cast<bool*>(b->addr) = v;
			break;
		}
		case BindKind::Float:
		{
			float v = static_cast<float>(luaL_checknumber(L, 2));
			if (b->fmax > b->fmin)
			{
				if (v < b->fmin) v = b->fmin;
				if (v > b->fmax) v = b->fmax;
			}
			*static_cast<float*>(b->addr) = v;
			break;
		}
		case BindKind::Int:
		{
			int v = static_cast<int>(luaL_checkinteger(L, 2));
			if (b->fmax > b->fmin)
			{
				int lo = static_cast<int>(b->fmin);
				int hi = static_cast<int>(b->fmax);
				if (v < lo) v = lo;
				if (v > hi) v = hi;
			}
			*static_cast<int*>(b->addr) = v;
			break;
		}
		}
		return 0;
	}

	int l_cathack_get(lua_State* L)
	{
		const char* name = luaL_checkstring(L, 1);
		const SettingBinding* b = FindBinding(name);
		if (!b) return luaL_error(L, "cathack.get: unknown setting '%s'", name);

		switch (b->kind)
		{
		case BindKind::Bool:
			lua_pushboolean(L, *static_cast<bool*>(b->addr));
			return 1;
		case BindKind::Float:
			lua_pushnumber(L, *static_cast<float*>(b->addr));
			return 1;
		case BindKind::Int:
			lua_pushinteger(L, *static_cast<int*>(b->addr));
			return 1;
		}
		return 0;
	}

	int l_cathack_settings(lua_State* L)
	{
		// Returns a table { name1=true|false|number, name2=..., ... } of every binding.
		lua_newtable(L);
		for (const SettingBinding* b = kSettingBindings; b->name; ++b)
		{
			switch (b->kind)
			{
			case BindKind::Bool:  lua_pushboolean(L, *static_cast<bool*>(b->addr)); break;
			case BindKind::Float: lua_pushnumber(L, *static_cast<float*>(b->addr)); break;
			case BindKind::Int:   lua_pushinteger(L, *static_cast<int*>(b->addr)); break;
			}
			lua_setfield(L, -2, b->name);
		}
		return 1;
	}

	int l_cathack_player_pos(lua_State* L)
	{
		auto* PC = GVars.ReadyOrNotChar;
		if (!PC || !Utils::IsValidActor(PC))
		{
			lua_pushnil(L);
			return 1;
		}
		FVector p = PC->K2_GetActorLocation();
		lua_pushnumber(L, p.X);
		lua_pushnumber(L, p.Y);
		lua_pushnumber(L, p.Z);
		return 3;
	}

	int l_cathack_player_health(lua_State* L)
	{
		auto* PC = GVars.ReadyOrNotChar;
		if (!PC || !Utils::IsValidActor(PC))
		{
			lua_pushnil(L);
			return 1;
		}
		const float cur = PC->GetCurrentHealth();
		const float max = PC->GetMaxHealth();
		lua_pushnumber(L, cur);
		lua_pushnumber(L, max);
		return 2;
	}

	int l_cathack_is_alive(lua_State* L)
	{
		auto* PC = GVars.ReadyOrNotChar;
		if (!PC || !Utils::IsValidActor(PC))
		{
			lua_pushboolean(L, 0);
			return 1;
		}
		lua_pushboolean(L, PC->IsDeadNotUnconscious() ? 0 : 1);
		return 1;
	}

	int l_cathack_screen_size(lua_State* L)
	{
		lua_pushnumber(L, GVars.ScreenSize.x);
		lua_pushnumber(L, GVars.ScreenSize.y);
		return 2;
	}

	int l_cathack_world_to_screen(lua_State* L)
	{
		const float wx = static_cast<float>(luaL_checknumber(L, 1));
		const float wy = static_cast<float>(luaL_checknumber(L, 2));
		const float wz = static_cast<float>(luaL_checknumber(L, 3));
		auto* C = GVars.PlayerController;
		if (!C)
		{
			lua_pushnil(L);
			return 1;
		}
		FVector wp{ wx, wy, wz };
		FVector2D sp;
		bool ok = C->ProjectWorldLocationToScreen(wp, &sp, true);
		if (!ok)
		{
			lua_pushnil(L);
			return 1;
		}
		lua_pushnumber(L, sp.X);
		lua_pushnumber(L, sp.Y);
		lua_pushboolean(L, 1);
		return 3;
	}

	int l_cathack_key(lua_State* L)
	{
		const int vk = static_cast<int>(luaL_checkinteger(L, 1));
		// GetAsyncKeyState is fine here: scripts polling key state every render
		// frame is exactly the use case it was built for.
		const SHORT s = GetAsyncKeyState(vk);
		lua_pushboolean(L, (s & 0x8000) ? 1 : 0);
		return 1;
	}

	int l_cathack_time(lua_State* L)
	{
		const auto now = std::chrono::steady_clock::now();
		const auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(now.time_since_epoch()).count();
		lua_pushinteger(L, static_cast<lua_Integer>(ms));
		return 1;
	}

	// ====================================================================
	// Math / vector helpers — convenient wrappers for the math you'll keep
	// re-implementing in Lua otherwise. All vectors are passed as 3 numbers
	// (x, y, z) rather than Lua tables to avoid the table-allocation cost.
	// ====================================================================

	int l_cathack_distance(lua_State* L)
	{
		const float ax = static_cast<float>(luaL_checknumber(L, 1));
		const float ay = static_cast<float>(luaL_checknumber(L, 2));
		const float az = static_cast<float>(luaL_checknumber(L, 3));
		const float bx = static_cast<float>(luaL_checknumber(L, 4));
		const float by = static_cast<float>(luaL_checknumber(L, 5));
		const float bz = static_cast<float>(luaL_checknumber(L, 6));
		const float dx = ax - bx, dy = ay - by, dz = az - bz;
		lua_pushnumber(L, std::sqrt(dx*dx + dy*dy + dz*dz));
		return 1;
	}

	int l_cathack_normalize(lua_State* L)
	{
		const float x = static_cast<float>(luaL_checknumber(L, 1));
		const float y = static_cast<float>(luaL_checknumber(L, 2));
		const float z = static_cast<float>(luaL_checknumber(L, 3));
		const float len = std::sqrt(x*x + y*y + z*z);
		if (len <= 0.0001f)
		{
			lua_pushnumber(L, 0.0); lua_pushnumber(L, 0.0); lua_pushnumber(L, 0.0);
		}
		else
		{
			lua_pushnumber(L, x / len);
			lua_pushnumber(L, y / len);
			lua_pushnumber(L, z / len);
		}
		return 3;
	}

	int l_cathack_dot(lua_State* L)
	{
		const float ax = static_cast<float>(luaL_checknumber(L, 1));
		const float ay = static_cast<float>(luaL_checknumber(L, 2));
		const float az = static_cast<float>(luaL_checknumber(L, 3));
		const float bx = static_cast<float>(luaL_checknumber(L, 4));
		const float by = static_cast<float>(luaL_checknumber(L, 5));
		const float bz = static_cast<float>(luaL_checknumber(L, 6));
		lua_pushnumber(L, ax*bx + ay*by + az*bz);
		return 1;
	}

	int l_cathack_cross(lua_State* L)
	{
		const float ax = static_cast<float>(luaL_checknumber(L, 1));
		const float ay = static_cast<float>(luaL_checknumber(L, 2));
		const float az = static_cast<float>(luaL_checknumber(L, 3));
		const float bx = static_cast<float>(luaL_checknumber(L, 4));
		const float by = static_cast<float>(luaL_checknumber(L, 5));
		const float bz = static_cast<float>(luaL_checknumber(L, 6));
		lua_pushnumber(L, ay*bz - az*by);
		lua_pushnumber(L, az*bx - ax*bz);
		lua_pushnumber(L, ax*by - ay*bx);
		return 3;
	}

	int l_cathack_lerp(lua_State* L)
	{
		const lua_Number a = luaL_checknumber(L, 1);
		const lua_Number b = luaL_checknumber(L, 2);
		const lua_Number t = luaL_checknumber(L, 3);
		lua_pushnumber(L, a + (b - a) * t);
		return 1;
	}

	// ====================================================================
	// Color helpers — return packed ABGR ints in the same encoding ImGui
	// (and therefore cathack.draw_*) expects.
	// ====================================================================

	int l_cathack_color(lua_State* L)
	{
		// rgba(r, g, b, a?) — components are 0..255 ints, with `a` defaulting to 255.
		const int r = static_cast<int>(luaL_checkinteger(L, 1));
		const int g = static_cast<int>(luaL_checkinteger(L, 2));
		const int b = static_cast<int>(luaL_checkinteger(L, 3));
		const int a = static_cast<int>(luaL_optinteger(L, 4, 255));
		auto cl = [](int v) { return v < 0 ? 0 : (v > 255 ? 255 : v); };
		const lua_Integer packed =
			(static_cast<lua_Integer>(cl(a) & 0xFF) << 24) |
			(static_cast<lua_Integer>(cl(b) & 0xFF) << 16) |
			(static_cast<lua_Integer>(cl(g) & 0xFF) << 8)  |
			 static_cast<lua_Integer>(cl(r) & 0xFF);
		lua_pushinteger(L, packed);
		return 1;
	}

	int l_cathack_color_hsv(lua_State* L)
	{
		// hsva(h[0..1], s[0..1], v[0..1], a[0..255]?). Standard HSV → RGB.
		const float h = static_cast<float>(luaL_checknumber(L, 1));
		const float s = static_cast<float>(luaL_checknumber(L, 2));
		const float v = static_cast<float>(luaL_checknumber(L, 3));
		const int a   = static_cast<int>(luaL_optinteger(L, 4, 255));
		float r = 0, g = 0, b = 0;
		ImGui::ColorConvertHSVtoRGB(h, s, v, r, g, b);
		auto to8 = [](float f) -> int { int v = static_cast<int>(f * 255.0f + 0.5f); return v < 0 ? 0 : (v > 255 ? 255 : v); };
		const int ai = a < 0 ? 0 : (a > 255 ? 255 : a);
		const lua_Integer packed =
			(static_cast<lua_Integer>(ai)     << 24) |
			(static_cast<lua_Integer>(to8(b)) << 16) |
			(static_cast<lua_Integer>(to8(g)) << 8)  |
			 static_cast<lua_Integer>(to8(r));
		lua_pushinteger(L, packed);
		return 1;
	}

	// ====================================================================
	// Engine / session info.
	// ====================================================================

	int l_cathack_is_in_game(lua_State* L)
	{
		const bool ok = (GVars.World != nullptr) && (GVars.Level != nullptr)
			&& (GVars.PlayerController != nullptr) && (GVars.ReadyOrNotChar != nullptr)
			&& Utils::IsValidActor(GVars.ReadyOrNotChar);
		lua_pushboolean(L, ok ? 1 : 0);
		return 1;
	}

	int l_cathack_world_name(lua_State* L)
	{
		if (!GVars.World) { lua_pushnil(L); return 1; }
		// UObject::GetName() is C++-side, no UFunction call.
		const std::string nm = GVars.World->GetName();
		lua_pushstring(L, nm.c_str());
		return 1;
	}

	int l_cathack_fps(lua_State* L)
	{
		ImGuiIO* io = ImGui::GetCurrentContext() ? &ImGui::GetIO() : nullptr;
		lua_pushnumber(L, io ? static_cast<lua_Number>(io->Framerate) : 0.0);
		return 1;
	}

	int l_cathack_delta_time(lua_State* L)
	{
		ImGuiIO* io = ImGui::GetCurrentContext() ? &ImGui::GetIO() : nullptr;
		lua_pushnumber(L, io ? static_cast<lua_Number>(io->DeltaTime) : 0.0);
		return 1;
	}

	int l_cathack_bindings(lua_State* L)
	{
		// Returns array of binding name strings. Useful for autocomplete-ish
		// menus or debugging "what can I set?"
		lua_newtable(L);
		int idx = 1;
		for (const SettingBinding* b = kSettingBindings; b->name; ++b)
		{
			lua_pushstring(L, b->name);
			lua_rawseti(L, -2, idx++);
		}
		return 1;
	}

	// ====================================================================
	// Player extras — beyond pos / health / alive that already exist.
	// ====================================================================

	int l_cathack_player_velocity(lua_State* L)
	{
		auto* PC = GVars.ReadyOrNotChar;
		if (!PC || !Utils::IsValidActor(PC)) { lua_pushnil(L); return 1; }
		FVector v = PC->GetVelocity();
		lua_pushnumber(L, v.X); lua_pushnumber(L, v.Y); lua_pushnumber(L, v.Z);
		return 3;
	}

	int l_cathack_player_speed(lua_State* L)
	{
		auto* PC = GVars.ReadyOrNotChar;
		if (!PC || !Utils::IsValidActor(PC)) { lua_pushnil(L); return 1; }
		FVector v = PC->GetVelocity();
		lua_pushnumber(L, std::sqrt(double(v.X)*v.X + double(v.Y)*v.Y + double(v.Z)*v.Z));
		return 1;
	}

	int l_cathack_player_aim_rot(lua_State* L)
	{
		auto* PC = GVars.ReadyOrNotChar;
		if (!PC || !Utils::IsValidActor(PC)) { lua_pushnil(L); return 1; }
		FRotator r = PC->GetBaseAimRotation();
		lua_pushnumber(L, r.Pitch); lua_pushnumber(L, r.Yaw); lua_pushnumber(L, r.Roll);
		return 3;
	}

	int l_cathack_player_team(lua_State* L)
	{
		auto* PC = GVars.ReadyOrNotChar;
		if (!PC || !Utils::IsValidActor(PC)) { lua_pushnil(L); return 1; }
		// Local player is always SWAT in RoN coop. We still classify by
		// IsA to be future-proof for game updates / mods.
		const char* kind = "unknown";
		if (PC->IsA(APlayerCharacter::StaticClass()) || PC->IsA(ASWATCharacter::StaticClass())) kind = "swat";
		else if (PC->IsA(ASuspectCharacter::StaticClass())) kind = "suspect";
		else if (PC->IsA(ACivilianCharacter::StaticClass())) kind = "civilian";
		lua_pushstring(L, kind);
		return 1;
	}

	int l_cathack_player_name(lua_State* L)
	{
		if (!GVars.PlayerController) { lua_pushnil(L); return 1; }
		auto* PS = GVars.PlayerController->PlayerState;
		if (!PS) { lua_pushnil(L); return 1; }
		const std::string nm = PS->GetPlayerName().ToString();
		lua_pushstring(L, nm.c_str());
		return 1;
	}

	// ====================================================================
	// Camera info — the local player's POV. Returns nil whenever the
	// camera cache hasn't been populated yet (the cheat sets GVars.POV
	// every tick whenever PlayerCameraManager is live).
	//
	// Reading is done directly from the FMinimalViewInfo cache (a struct
	// nested in APlayerCameraManager); writing goes through a tiny
	// "override" subsystem (see ApplyCameraOverrides at the bottom of
	// this section) so that values get re-asserted each tick / frame
	// even after the engine has re-computed its own pose.
	// ====================================================================

	// ----- Override state -----------------------------------------------
	// Every flagged-on override below is reapplied to GVars.POV from BOTH
	// the game-thread tick and the render-thread frame. We use atomics so
	// scripts can flip them from on_tick (game thread) without locking
	// out the render-thread reapplication (which holds no Lua mutex).
	std::atomic<bool>  g_camHasPosOverride{false};
	std::atomic<float> g_camOverridePosX{0.f};
	std::atomic<float> g_camOverridePosY{0.f};
	std::atomic<float> g_camOverridePosZ{0.f};
	std::atomic<bool>  g_camHasRotOverride{false};
	std::atomic<float> g_camOverridePitch{0.f};
	std::atomic<float> g_camOverrideYaw{0.f};
	std::atomic<float> g_camOverrideRoll{0.f};
	std::atomic<bool>  g_camHasFovOverride{false};
	std::atomic<float> g_camOverrideFov{90.f};

	// Frame-to-frame velocity sampling. Updated from both ticks but the
	// render-thread sampling is the authoritative one (most consistent
	// timestamp). Saved with millisecond precision; velocity is cm/s.
	std::atomic<bool>     g_camHadPrevSample{false};
	std::atomic<long long> g_camLastSampleMs{0};
	std::atomic<float> g_camLastX{0.f}, g_camLastY{0.f}, g_camLastZ{0.f};
	std::atomic<float> g_camVelX{0.f},  g_camVelY{0.f},  g_camVelZ{0.f};

	int l_cathack_camera_pos(lua_State* L)
	{
		if (!GVars.POV) { lua_pushnil(L); return 1; }
		const FVector p = GVars.POV->Location;
		lua_pushnumber(L, p.X); lua_pushnumber(L, p.Y); lua_pushnumber(L, p.Z);
		return 3;
	}

	int l_cathack_camera_rot(lua_State* L)
	{
		if (!GVars.POV) { lua_pushnil(L); return 1; }
		const FRotator r = GVars.POV->Rotation;
		lua_pushnumber(L, r.Pitch); lua_pushnumber(L, r.Yaw); lua_pushnumber(L, r.Roll);
		return 3;
	}

	int l_cathack_camera_forward(lua_State* L)
	{
		if (!GVars.POV) { lua_pushnil(L); return 1; }
		const FVector f = Utils::FRotatorToVector(GVars.POV->Rotation);
		lua_pushnumber(L, f.X); lua_pushnumber(L, f.Y); lua_pushnumber(L, f.Z);
		return 3;
	}

	int l_cathack_camera_right(lua_State* L)
	{
		if (!GVars.POV) { lua_pushnil(L); return 1; }
		const FVector r = UKismetMathLibrary::GetRightVector(GVars.POV->Rotation);
		lua_pushnumber(L, r.X); lua_pushnumber(L, r.Y); lua_pushnumber(L, r.Z);
		return 3;
	}

	int l_cathack_camera_up(lua_State* L)
	{
		if (!GVars.POV) { lua_pushnil(L); return 1; }
		const FVector u = UKismetMathLibrary::GetUpVector(GVars.POV->Rotation);
		lua_pushnumber(L, u.X); lua_pushnumber(L, u.Y); lua_pushnumber(L, u.Z);
		return 3;
	}

	int l_cathack_camera_fov(lua_State* L)
	{
		if (!GVars.POV) { lua_pushnil(L); return 1; }
		lua_pushnumber(L, GVars.POV->FOV);
		return 1;
	}

	int l_cathack_camera_aspect(lua_State* L)
	{
		if (!GVars.POV) { lua_pushnil(L); return 1; }
		lua_pushnumber(L, GVars.POV->AspectRatio);
		return 1;
	}

	int l_cathack_camera_near_clip(lua_State* L)
	{
		if (!GVars.POV) { lua_pushnil(L); return 1; }
		lua_pushnumber(L, GVars.POV->PerspectiveNearClipPlane);
		return 1;
	}

	int l_cathack_camera_velocity(lua_State* L)
	{
		// Returned velocity is whatever the most recent TickCameraSampling
		// computed. If sampling hasn't run (early frames) returns zeros.
		lua_pushnumber(L, g_camVelX.load());
		lua_pushnumber(L, g_camVelY.load());
		lua_pushnumber(L, g_camVelZ.load());
		return 3;
	}

	int l_cathack_camera_speed(lua_State* L)
	{
		const float vx = g_camVelX.load();
		const float vy = g_camVelY.load();
		const float vz = g_camVelZ.load();
		lua_pushnumber(L, std::sqrt(double(vx)*vx + double(vy)*vy + double(vz)*vz));
		return 1;
	}

	int l_cathack_camera_view_target(lua_State* L)
	{
		// Whatever actor the camera is currently following. Usually the
		// local player's pawn, but can be any actor when SetViewTarget
		// has been called (spectating, cinematics, etc.).
		if (!GVars.PlayerController) { lua_pushnil(L); return 1; }
		auto* cam = GVars.PlayerController->PlayerCameraManager;
		if (!cam) { lua_pushnil(L); return 1; }
		AActor* target = cam->ViewTarget.Target;
		if (!target || !Utils::IsValidActor(target)) { lua_pushnil(L); return 1; }
		lua_pushinteger(L, static_cast<lua_Integer>(reinterpret_cast<uintptr_t>(target)));
		return 1;
	}

	int l_cathack_camera_post_process_blend(lua_State* L)
	{
		if (!GVars.POV) { lua_pushnil(L); return 1; }
		lua_pushnumber(L, GVars.POV->PostProcessBlendWeight);
		return 1;
	}

	// ----- Coordinate transforms ----------------------------------------

	int l_cathack_screen_to_world_ray(lua_State* L)
	{
		// Engine API: APlayerController::DeprojectScreenPositionToWorld.
		// Returns the ray origin (world-space) and a unit direction. Useful
		// for "what am I looking at under cursor" raycasts.
		if (!GVars.PlayerController) { lua_pushnil(L); return 1; }
		const float sx = static_cast<float>(luaL_checknumber(L, 1));
		const float sy = static_cast<float>(luaL_checknumber(L, 2));
		FVector origin{0,0,0};
		FVector direction{0,0,0};
		if (!GVars.PlayerController->DeprojectScreenPositionToWorld(sx, sy, &origin, &direction))
		{
			lua_pushnil(L);
			return 1;
		}
		lua_pushnumber(L, origin.X);    lua_pushnumber(L, origin.Y);    lua_pushnumber(L, origin.Z);
		lua_pushnumber(L, direction.X); lua_pushnumber(L, direction.Y); lua_pushnumber(L, direction.Z);
		return 6;
	}

	int l_cathack_world_to_camera_relative(lua_State* L)
	{
		// Convert world-space point P to camera-relative coordinates
		// (forward, right, up). Useful for pretty-printing distances or
		// driving HUD elements that follow look direction.
		if (!GVars.POV) { lua_pushnil(L); return 1; }
		const float wx = static_cast<float>(luaL_checknumber(L, 1));
		const float wy = static_cast<float>(luaL_checknumber(L, 2));
		const float wz = static_cast<float>(luaL_checknumber(L, 3));
		const FVector cam = GVars.POV->Location;
		const FVector dx{wx - cam.X, wy - cam.Y, wz - cam.Z};
		const FVector fwd = UKismetMathLibrary::GetForwardVector(GVars.POV->Rotation);
		const FVector rgt = UKismetMathLibrary::GetRightVector(GVars.POV->Rotation);
		const FVector upv = UKismetMathLibrary::GetUpVector(GVars.POV->Rotation);
		lua_pushnumber(L, dx.X*fwd.X + dx.Y*fwd.Y + dx.Z*fwd.Z);
		lua_pushnumber(L, dx.X*rgt.X + dx.Y*rgt.Y + dx.Z*rgt.Z);
		lua_pushnumber(L, dx.X*upv.X + dx.Y*upv.Y + dx.Z*upv.Z);
		return 3;
	}

	int l_cathack_camera_to_world(lua_State* L)
	{
		// Inverse of world_to_camera_relative.
		if (!GVars.POV) { lua_pushnil(L); return 1; }
		const float fwd_d = static_cast<float>(luaL_checknumber(L, 1));
		const float rgt_d = static_cast<float>(luaL_checknumber(L, 2));
		const float up_d  = static_cast<float>(luaL_checknumber(L, 3));
		const FVector cam = GVars.POV->Location;
		const FVector fwd = UKismetMathLibrary::GetForwardVector(GVars.POV->Rotation);
		const FVector rgt = UKismetMathLibrary::GetRightVector(GVars.POV->Rotation);
		const FVector upv = UKismetMathLibrary::GetUpVector(GVars.POV->Rotation);
		lua_pushnumber(L, cam.X + fwd.X*fwd_d + rgt.X*rgt_d + upv.X*up_d);
		lua_pushnumber(L, cam.Y + fwd.Y*fwd_d + rgt.Y*rgt_d + upv.Y*up_d);
		lua_pushnumber(L, cam.Z + fwd.Z*fwd_d + rgt.Z*rgt_d + upv.Z*up_d);
		return 3;
	}

	// ----- Override setters ---------------------------------------------

	int l_cathack_set_camera_pos(lua_State* L)
	{
		// (x, y, z) sets a persistent override; (nil) clears it.
		if (lua_isnoneornil(L, 1))
		{
			g_camHasPosOverride.store(false);
			return 0;
		}
		g_camOverridePosX.store(static_cast<float>(luaL_checknumber(L, 1)));
		g_camOverridePosY.store(static_cast<float>(luaL_checknumber(L, 2)));
		g_camOverridePosZ.store(static_cast<float>(luaL_checknumber(L, 3)));
		g_camHasPosOverride.store(true);
		return 0;
	}

	int l_cathack_set_camera_rot(lua_State* L)
	{
		// (pitch, yaw, roll) — same persistence scheme as set_camera_pos.
		if (lua_isnoneornil(L, 1))
		{
			g_camHasRotOverride.store(false);
			return 0;
		}
		g_camOverridePitch.store(static_cast<float>(luaL_checknumber(L, 1)));
		g_camOverrideYaw.store(static_cast<float>(luaL_checknumber(L, 2)));
		g_camOverrideRoll.store(static_cast<float>(luaL_optnumber(L, 3, 0.0)));
		g_camHasRotOverride.store(true);
		return 0;
	}

	int l_cathack_set_camera_fov(lua_State* L)
	{
		if (lua_isnoneornil(L, 1))
		{
			g_camHasFovOverride.store(false);
			return 0;
		}
		float fov = static_cast<float>(luaL_checknumber(L, 1));
		// Clamp to "anything that isn't going to crash the renderer". UE
		// generally accepts 1..170; outside that you get fish-eye + numeric
		// instability.
		if (fov < 1.0f)   fov = 1.0f;
		if (fov > 170.0f) fov = 170.0f;
		g_camOverrideFov.store(fov);
		g_camHasFovOverride.store(true);
		return 0;
	}

	int l_cathack_clear_camera_pos(lua_State* L)
	{
		g_camHasPosOverride.store(false);
		return 0;
	}

	int l_cathack_clear_camera_rot(lua_State* L)
	{
		g_camHasRotOverride.store(false);
		return 0;
	}

	int l_cathack_clear_camera_fov(lua_State* L)
	{
		g_camHasFovOverride.store(false);
		return 0;
	}

	int l_cathack_clear_camera_overrides(lua_State* L)
	{
		g_camHasPosOverride.store(false);
		g_camHasRotOverride.store(false);
		g_camHasFovOverride.store(false);
		return 0;
	}

	// ----- View target / spectate ---------------------------------------

	int l_cathack_set_view_target(lua_State* L)
	{
		// (entity_id, blend_time?) — passing nil reverts to the local pawn.
		if (!GVars.PlayerController) { lua_pushboolean(L, 0); return 1; }
		AActor* target = nullptr;
		if (lua_isnoneornil(L, 1))
		{
			target = reinterpret_cast<AActor*>(GVars.ReadyOrNotChar);
		}
		else
		{
			lua_Integer id = luaL_checkinteger(L, 1);
			AActor* a = reinterpret_cast<AActor*>(static_cast<uintptr_t>(id));
			if (!a || !Utils::IsValidActor(a)) { lua_pushboolean(L, 0); return 1; }
			target = a;
		}
		const float blend = static_cast<float>(luaL_optnumber(L, 2, 0.0));
		// Linear blend, exp=0 (no easing), don't lock outgoing — matches
		// what "spectate" implementations in UE games typically use.
		GVars.PlayerController->SetViewTargetWithBlend(
			target,
			blend,
			EViewTargetBlendFunction::VTBlend_Linear,
			0.0f,
			false);
		lua_pushboolean(L, 1);
		return 1;
	}

	// ----- Camera effects -----------------------------------------------

	int l_cathack_camera_fade(lua_State* L)
	{
		// (from_alpha, to_alpha, duration, color_int?, fade_audio?, hold?)
		if (!GVars.PlayerController || !GVars.PlayerController->PlayerCameraManager)
		{
			lua_pushboolean(L, 0);
			return 1;
		}
		const float fromA = static_cast<float>(luaL_checknumber(L, 1));
		const float toA   = static_cast<float>(luaL_checknumber(L, 2));
		const float dur   = static_cast<float>(luaL_checknumber(L, 3));
		const lua_Integer color = luaL_optinteger(L, 4, 0xFF000000LL); // default opaque black
		const bool fadeAudio = lua_toboolean(L, 5) != 0;
		const bool hold      = lua_toboolean(L, 6) != 0;
		// Unpack ImGui ABGR int → FLinearColor 0..1 floats.
		FLinearColor c;
		c.R = (color & 0xFF) / 255.0f;
		c.G = ((color >> 8) & 0xFF) / 255.0f;
		c.B = ((color >> 16) & 0xFF) / 255.0f;
		c.A = ((color >> 24) & 0xFF) / 255.0f;
		GVars.PlayerController->PlayerCameraManager->StartCameraFade(
			fromA, toA, dur, c, fadeAudio, hold);
		lua_pushboolean(L, 1);
		return 1;
	}

	int l_cathack_camera_fade_stop(lua_State* L)
	{
		if (!GVars.PlayerController || !GVars.PlayerController->PlayerCameraManager)
			return 0;
		GVars.PlayerController->PlayerCameraManager->StopCameraFade();
		return 0;
	}

	int l_cathack_camera_set_manual_fade(lua_State* L)
	{
		// (amount, color_int?, fade_audio?) — instant fade with no blend.
		if (!GVars.PlayerController || !GVars.PlayerController->PlayerCameraManager)
			return 0;
		const float amount = static_cast<float>(luaL_checknumber(L, 1));
		const lua_Integer color = luaL_optinteger(L, 2, 0xFF000000LL);
		const bool fadeAudio = lua_toboolean(L, 3) != 0;
		FLinearColor c;
		c.R = (color & 0xFF) / 255.0f;
		c.G = ((color >> 8) & 0xFF) / 255.0f;
		c.B = ((color >> 16) & 0xFF) / 255.0f;
		c.A = ((color >> 24) & 0xFF) / 255.0f;
		GVars.PlayerController->PlayerCameraManager->SetManualCameraFade(amount, c, fadeAudio);
		return 0;
	}

	int l_cathack_camera_stop_shakes(lua_State* L)
	{
		if (!GVars.PlayerController || !GVars.PlayerController->PlayerCameraManager)
			return 0;
		const bool immediately = lua_toboolean(L, 1) != 0;
		GVars.PlayerController->PlayerCameraManager->StopAllCameraShakes(immediately);
		return 0;
	}

	// ====================================================================
	// Entity enumeration.
	//
	// Entity IDs are AActor* cast to lua_Integer. To make sure scripts
	// can't smash random pointers, every entity_*(id) call:
	//   1. Casts back to AActor*.
	//   2. Checks Utils::IsValidActor (vtable + GObjects + level + class).
	//   3. Re-confirms IsA(AReadyOrNotCharacter::StaticClass()).
	// If any of that fails, the call returns nil.
	// ====================================================================

	static AReadyOrNotCharacter* ResolveEntity(lua_Integer id)
	{
		if (id == 0) return nullptr;
		AActor* a = reinterpret_cast<AActor*>(static_cast<uintptr_t>(id));
		if (!a || !Utils::IsValidActor(a)) return nullptr;
		if (!a->IsA(AReadyOrNotCharacter::StaticClass())) return nullptr;
		return reinterpret_cast<AReadyOrNotCharacter*>(a);
	}

	static const char* ClassifyEntity(AReadyOrNotCharacter* e)
	{
		if (!e) return "unknown";
		if (e->IsA(APlayerCharacter::StaticClass())) return "player";
		if (e->IsA(ASWATCharacter::StaticClass()))   return "swat";
		if (e->IsA(ASuspectCharacter::StaticClass())) return "suspect";
		if (e->IsA(ACivilianCharacter::StaticClass())) return "civilian";
		// Fallback: use IsSuspect/IsCivilian methods.
		if (e->IsSuspect()) return "suspect";
		if (e->IsCivilian()) return "civilian";
		return "unknown";
	}

	// FName is opaque (a 32-bit comparison index into the global FName
	// pool) and the SDK doesn't expose a string-based constructor — to
	// resolve a name like "Head" we have to walk the mesh's bone list
	// and ToString-compare each entry. Real character rigs in RoN have
	// ~80–120 bones so this is cheap.
	static int FindBoneIndexByName(USkeletalMeshComponent* M, const char* needle)
	{
		if (!M || !Utils::IsLikelyLiveUObject(M) || !needle) return -1;
		const int32 n = M->GetNumBones();
		if (n <= 0 || n > 1024) return -1;
		for (int32 i = 0; i < n; ++i)
		{
			FName bn = M->GetBoneName(i);
			if (bn.ToString() == needle) return i;
		}
		return -1;
	}

	// Try a few common bone names for "head" to be robust across rigs.
	// Returns the world position; falls back to the actor location when
	// no head bone exists / mesh is invalid.
	static FVector EntityHeadPos(AReadyOrNotCharacter* e)
	{
		FVector pos = e->K2_GetActorLocation();
		USkeletalMeshComponent* M = e->Mesh;
		if (!M || !Utils::IsLikelyLiveUObject(M)) return pos;
		static const char* kHeadNames[] = { "Head", "head", "head_1", "Bip01-Head", "neck_1" };
		for (const char* nm : kHeadNames)
		{
			const int idx = FindBoneIndexByName(M, nm);
			if (idx < 0) continue;
			FName fn = M->GetBoneName(idx);
			FVector p = M->GetBoneTransform(fn, ERelativeTransformSpace::RTS_World).Translation;
			if (p.X != 0 || p.Y != 0 || p.Z != 0) return p;
		}
		return pos;
	}

	int l_cathack_entities(lua_State* L)
	{
		lua_newtable(L);
		if (!GVars.Level) return 1;

		const TArray<AActor*>& Actors = GVars.Level->Actors;
		const int n = Actors.Num();
		if (n < 0 || n > 8192) return 1; // sanity

		const FVector myPos = GVars.ReadyOrNotChar
			? GVars.ReadyOrNotChar->K2_GetActorLocation()
			: FVector{0.f, 0.f, 0.f};

		int outIdx = 1;
		for (int i = 0; i < n; ++i)
		{
			AActor* A = Actors[i];
			if (!A || !Utils::IsValidActor(A)) continue;
			if (!A->IsA(AReadyOrNotCharacter::StaticClass())) continue;
			auto* e = reinterpret_cast<AReadyOrNotCharacter*>(A);

			// Skip the local player by default — entity_count() and
			// entities() consistently mean "everyone else". Scripts that
			// want the local player can use cathack.local_id().
			if (e == GVars.ReadyOrNotChar) continue;

			const FVector p = e->K2_GetActorLocation();
			const FVector h = EntityHeadPos(e);
			const float dx = p.X - myPos.X;
			const float dy = p.Y - myPos.Y;
			const float dz = p.Z - myPos.Z;
			const float distMeters = std::sqrt(dx*dx + dy*dy + dz*dz) / 100.0f;

			lua_newtable(L);

			lua_pushinteger(L, static_cast<lua_Integer>(reinterpret_cast<uintptr_t>(e)));
			lua_setfield(L, -2, "id");
			lua_pushstring(L, ClassifyEntity(e));
			lua_setfield(L, -2, "kind");

			lua_newtable(L);
			lua_pushnumber(L, p.X); lua_setfield(L, -2, "x");
			lua_pushnumber(L, p.Y); lua_setfield(L, -2, "y");
			lua_pushnumber(L, p.Z); lua_setfield(L, -2, "z");
			lua_setfield(L, -2, "pos");

			lua_newtable(L);
			lua_pushnumber(L, h.X); lua_setfield(L, -2, "x");
			lua_pushnumber(L, h.Y); lua_setfield(L, -2, "y");
			lua_pushnumber(L, h.Z); lua_setfield(L, -2, "z");
			lua_setfield(L, -2, "head");

			lua_pushnumber(L, e->GetCurrentHealth()); lua_setfield(L, -2, "health");
			lua_pushnumber(L, e->GetMaxHealth());     lua_setfield(L, -2, "max_health");
			lua_pushnumber(L, distMeters);            lua_setfield(L, -2, "distance");

			const bool dead = e->IsDeadOrUnconscious();
			lua_pushboolean(L, dead ? 0 : 1);                         lua_setfield(L, -2, "alive");
			lua_pushboolean(L, dead ? 1 : 0);                         lua_setfield(L, -2, "dead");
			lua_pushboolean(L, e->IsSuspect() ? 1 : 0);               lua_setfield(L, -2, "suspect");
			lua_pushboolean(L, e->IsCivilian() ? 1 : 0);              lua_setfield(L, -2, "civilian");
			lua_pushboolean(L, (e->IsA(APlayerCharacter::StaticClass())
				|| e->IsA(ASWATCharacter::StaticClass())) ? 1 : 0);   lua_setfield(L, -2, "swat");
			lua_pushboolean(L, e->IsArrested() ? 1 : 0);              lua_setfield(L, -2, "arrested");
			lua_pushboolean(L, e->IsArrestedOrSurrendered() ? 1 : 0); lua_setfield(L, -2, "surrendered");
			lua_pushboolean(L, e->IsIncapacitated() ? 1 : 0);         lua_setfield(L, -2, "incapacitated");
			lua_pushboolean(L, e->bIsBeingArrested ? 1 : 0);          lua_setfield(L, -2, "being_arrested");

			// Optional name — only present for players (PlayerState).
			if (e->IsA(APlayerCharacter::StaticClass()))
			{
				auto* P = static_cast<APlayerCharacter*>(static_cast<AActor*>(e));
				if (P && P->PlayerState)
				{
					const std::string nm = P->PlayerState->GetPlayerName().ToString();
					lua_pushstring(L, nm.c_str());
					lua_setfield(L, -2, "name");
				}
			}

			lua_rawseti(L, -2, outIdx++);
		}
		return 1;
	}

	int l_cathack_entity_count(lua_State* L)
	{
		int count = 0;
		if (GVars.Level)
		{
			const TArray<AActor*>& Actors = GVars.Level->Actors;
			const int n = Actors.Num();
			if (n >= 0 && n <= 8192)
			{
				for (int i = 0; i < n; ++i)
				{
					AActor* A = Actors[i];
					if (!A || !Utils::IsValidActor(A)) continue;
					if (!A->IsA(AReadyOrNotCharacter::StaticClass())) continue;
					if (A == GVars.ReadyOrNotChar) continue;
					++count;
				}
			}
		}
		lua_pushinteger(L, count);
		return 1;
	}

	int l_cathack_local_id(lua_State* L)
	{
		auto* PC = GVars.ReadyOrNotChar;
		if (!PC || !Utils::IsValidActor(PC)) { lua_pushnil(L); return 1; }
		lua_pushinteger(L, static_cast<lua_Integer>(reinterpret_cast<uintptr_t>(PC)));
		return 1;
	}

	int l_cathack_entity_pos(lua_State* L)
	{
		auto* e = ResolveEntity(luaL_checkinteger(L, 1));
		if (!e) { lua_pushnil(L); return 1; }
		const FVector p = e->K2_GetActorLocation();
		lua_pushnumber(L, p.X); lua_pushnumber(L, p.Y); lua_pushnumber(L, p.Z);
		return 3;
	}

	int l_cathack_entity_velocity(lua_State* L)
	{
		auto* e = ResolveEntity(luaL_checkinteger(L, 1));
		if (!e) { lua_pushnil(L); return 1; }
		const FVector v = e->GetVelocity();
		lua_pushnumber(L, v.X); lua_pushnumber(L, v.Y); lua_pushnumber(L, v.Z);
		return 3;
	}

	int l_cathack_entity_speed(lua_State* L)
	{
		auto* e = ResolveEntity(luaL_checkinteger(L, 1));
		if (!e) { lua_pushnil(L); return 1; }
		const FVector v = e->GetVelocity();
		lua_pushnumber(L, std::sqrt(double(v.X)*v.X + double(v.Y)*v.Y + double(v.Z)*v.Z));
		return 1;
	}

	int l_cathack_entity_rotation(lua_State* L)
	{
		auto* e = ResolveEntity(luaL_checkinteger(L, 1));
		if (!e) { lua_pushnil(L); return 1; }
		const FRotator r = e->K2_GetActorRotation();
		lua_pushnumber(L, r.Pitch); lua_pushnumber(L, r.Yaw); lua_pushnumber(L, r.Roll);
		return 3;
	}

	int l_cathack_entity_aim_rot(lua_State* L)
	{
		auto* e = ResolveEntity(luaL_checkinteger(L, 1));
		if (!e) { lua_pushnil(L); return 1; }
		const FRotator r = e->GetBaseAimRotation();
		lua_pushnumber(L, r.Pitch); lua_pushnumber(L, r.Yaw); lua_pushnumber(L, r.Roll);
		return 3;
	}

	int l_cathack_entity_health(lua_State* L)
	{
		auto* e = ResolveEntity(luaL_checkinteger(L, 1));
		if (!e) { lua_pushnil(L); return 1; }
		lua_pushnumber(L, e->GetCurrentHealth());
		lua_pushnumber(L, e->GetMaxHealth());
		return 2;
	}

	int l_cathack_entity_kind(lua_State* L)
	{
		auto* e = ResolveEntity(luaL_checkinteger(L, 1));
		if (!e) { lua_pushnil(L); return 1; }
		lua_pushstring(L, ClassifyEntity(e));
		return 1;
	}

	int l_cathack_entity_alive(lua_State* L)
	{
		auto* e = ResolveEntity(luaL_checkinteger(L, 1));
		if (!e) { lua_pushboolean(L, 0); return 1; }
		lua_pushboolean(L, e->IsDeadOrUnconscious() ? 0 : 1);
		return 1;
	}

	int l_cathack_entity_distance(lua_State* L)
	{
		auto* e = ResolveEntity(luaL_checkinteger(L, 1));
		if (!e) { lua_pushnil(L); return 1; }
		auto* PC = GVars.ReadyOrNotChar;
		if (!PC || !Utils::IsValidActor(PC)) { lua_pushnil(L); return 1; }
		const FVector a = PC->K2_GetActorLocation();
		const FVector b = e->K2_GetActorLocation();
		const float dx = a.X - b.X, dy = a.Y - b.Y, dz = a.Z - b.Z;
		lua_pushnumber(L, std::sqrt(double(dx)*dx + double(dy)*dy + double(dz)*dz) / 100.0);
		return 1;
	}

	int l_cathack_entity_name(lua_State* L)
	{
		auto* e = ResolveEntity(luaL_checkinteger(L, 1));
		if (!e) { lua_pushnil(L); return 1; }
		if (e->IsA(APlayerCharacter::StaticClass()))
		{
			auto* P = static_cast<APlayerCharacter*>(static_cast<AActor*>(e));
			if (P && P->PlayerState)
			{
				const std::string nm = P->PlayerState->GetPlayerName().ToString();
				lua_pushstring(L, nm.c_str());
				return 1;
			}
		}
		lua_pushnil(L);
		return 1;
	}

	int l_cathack_entity_bone(lua_State* L)
	{
		auto* e = ResolveEntity(luaL_checkinteger(L, 1));
		if (!e) { lua_pushnil(L); return 1; }
		USkeletalMeshComponent* M = e->Mesh;
		if (!M || !Utils::IsLikelyLiveUObject(M)) { lua_pushnil(L); return 1; }
		// Accept either bone name (string) or bone index (integer).
		FVector bonePos;
		int boneIndex = -1;
		if (lua_type(L, 2) == LUA_TNUMBER)
		{
			boneIndex = static_cast<int>(luaL_checkinteger(L, 2));
		}
		else
		{
			const char* s = luaL_checkstring(L, 2);
			boneIndex = FindBoneIndexByName(M, s);
		}
		if (boneIndex < 0) { lua_pushnil(L); return 1; }
		FName n = M->GetBoneName(boneIndex);
		bonePos = M->GetBoneTransform(n, ERelativeTransformSpace::RTS_World).Translation;
		// Engine returns (0,0,0) for unknown bones — easy to filter on the
		// Lua side (`if x ~= 0 or y ~= 0 or z ~= 0 then`) but we still
		// pass it through so the caller's behavior is predictable.
		lua_pushnumber(L, bonePos.X);
		lua_pushnumber(L, bonePos.Y);
		lua_pushnumber(L, bonePos.Z);
		return 3;
	}

	int l_cathack_entity_screen_pos(lua_State* L)
	{
		auto* e = ResolveEntity(luaL_checkinteger(L, 1));
		if (!e || !GVars.PlayerController) { lua_pushnil(L); return 1; }
		// Optional second arg picks "head" vs "feet" — defaults to "feet"
		// (the actor's pivot, which sits at foot height in UE).
		const char* mode = luaL_optstring(L, 2, "feet");
		FVector world = e->K2_GetActorLocation();
		if (std::strcmp(mode, "head") == 0)
			world = EntityHeadPos(e);
		FVector2D sp;
		if (!GVars.PlayerController->ProjectWorldLocationToScreen(world, &sp, true))
		{
			lua_pushnil(L);
			return 1;
		}
		lua_pushnumber(L, sp.X);
		lua_pushnumber(L, sp.Y);
		lua_pushboolean(L, 1);
		return 3;
	}

	int l_cathack_entity_visible(lua_State* L)
	{
		// Convenience: line-of-sight from the camera to the entity's head bone.
		auto* e = ResolveEntity(luaL_checkinteger(L, 1));
		if (!e || !GVars.World || !GVars.POV) { lua_pushboolean(L, 0); return 1; }
		const FVector start = GVars.POV->Location;
		const FVector end   = EntityHeadPos(e);
		TArray<AActor*> ignore;
		if (GVars.ReadyOrNotChar) ignore.Add(GVars.ReadyOrNotChar);
		FHitResult hit;
		UKismetSystemLibrary::LineTraceSingle(
			GVars.World, start, end,
			ETraceTypeQuery::TraceTypeQuery1, true, ignore,
			EDrawDebugTrace::None, &hit, true,
			FLinearColor(), FLinearColor(), 1.0f);
		AActor* hitActor = nullptr;
		if (UPrimitiveComponent* hitPrim = hit.Component.Get())
			hitActor = hitPrim->GetOwner();
		const bool visible = !hit.bBlockingHit || hitActor == reinterpret_cast<AActor*>(e);
		lua_pushboolean(L, visible ? 1 : 0);
		return 1;
	}

	// ====================================================================
	// Raycast / LOS — direct world-space queries. The two share a worker.
	// ====================================================================

	struct LineTraceOut
	{
		bool        hit;
		FVector     pos;
		float       distance;
		AActor*     actor;
		bool        valid;
	};

	static LineTraceOut DoLineTrace(const FVector& start, const FVector& end)
	{
		LineTraceOut out{ false, FVector{0,0,0}, 0.f, nullptr, false };
		if (!GVars.World) return out;
		TArray<AActor*> ignore;
		if (GVars.ReadyOrNotChar) ignore.Add(GVars.ReadyOrNotChar);
		FHitResult hit;
		UKismetSystemLibrary::LineTraceSingle(
			GVars.World, start, end,
			ETraceTypeQuery::TraceTypeQuery1, true, ignore,
			EDrawDebugTrace::None, &hit, true,
			FLinearColor(), FLinearColor(), 1.0f);
		out.valid = true;
		out.hit = hit.bBlockingHit;
		if (out.hit)
		{
			out.pos = hit.Location;
			out.distance = hit.Distance;
			if (UPrimitiveComponent* hitPrim = hit.Component.Get())
				out.actor = hitPrim->GetOwner();
		}
		return out;
	}

	int l_cathack_line_of_sight(lua_State* L)
	{
		const float ax = static_cast<float>(luaL_checknumber(L, 1));
		const float ay = static_cast<float>(luaL_checknumber(L, 2));
		const float az = static_cast<float>(luaL_checknumber(L, 3));
		const float bx = static_cast<float>(luaL_checknumber(L, 4));
		const float by = static_cast<float>(luaL_checknumber(L, 5));
		const float bz = static_cast<float>(luaL_checknumber(L, 6));
		LineTraceOut r = DoLineTrace(FVector{ax, ay, az}, FVector{bx, by, bz});
		if (!r.valid) { lua_pushnil(L); return 1; }
		lua_pushboolean(L, r.hit ? 0 : 1);
		return 1;
	}

	int l_cathack_line_trace(lua_State* L)
	{
		const float ax = static_cast<float>(luaL_checknumber(L, 1));
		const float ay = static_cast<float>(luaL_checknumber(L, 2));
		const float az = static_cast<float>(luaL_checknumber(L, 3));
		const float bx = static_cast<float>(luaL_checknumber(L, 4));
		const float by = static_cast<float>(luaL_checknumber(L, 5));
		const float bz = static_cast<float>(luaL_checknumber(L, 6));
		LineTraceOut r = DoLineTrace(FVector{ax, ay, az}, FVector{bx, by, bz});
		if (!r.valid) { lua_pushnil(L); return 1; }
		lua_newtable(L);
		lua_pushboolean(L, r.hit ? 1 : 0);   lua_setfield(L, -2, "hit");
		lua_pushnumber(L, r.pos.X);          lua_setfield(L, -2, "x");
		lua_pushnumber(L, r.pos.Y);          lua_setfield(L, -2, "y");
		lua_pushnumber(L, r.pos.Z);          lua_setfield(L, -2, "z");
		lua_pushnumber(L, r.distance);       lua_setfield(L, -2, "distance");
		// If the hit primitive belongs to a known character, expose its id
		// so scripts can match it back to entities() entries.
		if (r.actor && Utils::IsValidActor(r.actor) &&
			r.actor->IsA(AReadyOrNotCharacter::StaticClass()))
		{
			lua_pushinteger(L, static_cast<lua_Integer>(reinterpret_cast<uintptr_t>(r.actor)));
			lua_setfield(L, -2, "entity_id");
		}
		return 1;
	}

	// ====================================================================
	// Drawing primitives — only valid during on_render. Outside of that
	// window they no-op (returning 0) so a buggy script that calls draw_*
	// from on_tick doesn't crash, just silently doesn't draw.
	// ====================================================================

	int l_cathack_draw_text(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x = static_cast<float>(luaL_checknumber(L, 1));
		const float y = static_cast<float>(luaL_checknumber(L, 2));
		const char* s = luaL_checkstring(L, 3);
		const lua_Integer color = luaL_optinteger(L, 4, 0xFFFFFFFFLL);
		dl->AddText(ImVec2(x, y), static_cast<ImU32>(color), s);
		return 0;
	}

	int l_cathack_draw_rect(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x = static_cast<float>(luaL_checknumber(L, 1));
		const float y = static_cast<float>(luaL_checknumber(L, 2));
		const float w = static_cast<float>(luaL_checknumber(L, 3));
		const float h = static_cast<float>(luaL_checknumber(L, 4));
		const lua_Integer color = luaL_optinteger(L, 5, 0xFFFFFFFFLL);
		const float thickness = static_cast<float>(luaL_optnumber(L, 6, 1.0));
		dl->AddRect(ImVec2(x, y), ImVec2(x + w, y + h), static_cast<ImU32>(color), 0.f, 0, thickness);
		return 0;
	}

	int l_cathack_draw_rect_filled(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x = static_cast<float>(luaL_checknumber(L, 1));
		const float y = static_cast<float>(luaL_checknumber(L, 2));
		const float w = static_cast<float>(luaL_checknumber(L, 3));
		const float h = static_cast<float>(luaL_checknumber(L, 4));
		const lua_Integer color = luaL_optinteger(L, 5, 0xFFFFFFFFLL);
		const float rounding = static_cast<float>(luaL_optnumber(L, 6, 0.0));
		dl->AddRectFilled(ImVec2(x, y), ImVec2(x + w, y + h), static_cast<ImU32>(color), rounding);
		return 0;
	}

	int l_cathack_draw_line(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x1 = static_cast<float>(luaL_checknumber(L, 1));
		const float y1 = static_cast<float>(luaL_checknumber(L, 2));
		const float x2 = static_cast<float>(luaL_checknumber(L, 3));
		const float y2 = static_cast<float>(luaL_checknumber(L, 4));
		const lua_Integer color = luaL_optinteger(L, 5, 0xFFFFFFFFLL);
		const float thickness = static_cast<float>(luaL_optnumber(L, 6, 1.0));
		dl->AddLine(ImVec2(x1, y1), ImVec2(x2, y2), static_cast<ImU32>(color), thickness);
		return 0;
	}

	int l_cathack_draw_circle(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float cx = static_cast<float>(luaL_checknumber(L, 1));
		const float cy = static_cast<float>(luaL_checknumber(L, 2));
		const float r  = static_cast<float>(luaL_checknumber(L, 3));
		const lua_Integer color = luaL_optinteger(L, 4, 0xFFFFFFFFLL);
		const int segments = static_cast<int>(luaL_optinteger(L, 5, 32));
		const float thickness = static_cast<float>(luaL_optnumber(L, 6, 1.0));
		dl->AddCircle(ImVec2(cx, cy), r, static_cast<ImU32>(color), segments, thickness);
		return 0;
	}

	int l_cathack_draw_circle_filled(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float cx = static_cast<float>(luaL_checknumber(L, 1));
		const float cy = static_cast<float>(luaL_checknumber(L, 2));
		const float r  = static_cast<float>(luaL_checknumber(L, 3));
		const lua_Integer color = luaL_optinteger(L, 4, 0xFFFFFFFFLL);
		const int segments = static_cast<int>(luaL_optinteger(L, 5, 32));
		dl->AddCircleFilled(ImVec2(cx, cy), r, static_cast<ImU32>(color), segments);
		return 0;
	}

	int l_cathack_draw_triangle(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x1 = static_cast<float>(luaL_checknumber(L, 1));
		const float y1 = static_cast<float>(luaL_checknumber(L, 2));
		const float x2 = static_cast<float>(luaL_checknumber(L, 3));
		const float y2 = static_cast<float>(luaL_checknumber(L, 4));
		const float x3 = static_cast<float>(luaL_checknumber(L, 5));
		const float y3 = static_cast<float>(luaL_checknumber(L, 6));
		const lua_Integer color = luaL_optinteger(L, 7, 0xFFFFFFFFLL);
		const float thickness   = static_cast<float>(luaL_optnumber(L, 8, 1.0));
		dl->AddTriangle(ImVec2(x1, y1), ImVec2(x2, y2), ImVec2(x3, y3),
			static_cast<ImU32>(color), thickness);
		return 0;
	}

	int l_cathack_draw_triangle_filled(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x1 = static_cast<float>(luaL_checknumber(L, 1));
		const float y1 = static_cast<float>(luaL_checknumber(L, 2));
		const float x2 = static_cast<float>(luaL_checknumber(L, 3));
		const float y2 = static_cast<float>(luaL_checknumber(L, 4));
		const float x3 = static_cast<float>(luaL_checknumber(L, 5));
		const float y3 = static_cast<float>(luaL_checknumber(L, 6));
		const lua_Integer color = luaL_optinteger(L, 7, 0xFFFFFFFFLL);
		dl->AddTriangleFilled(ImVec2(x1, y1), ImVec2(x2, y2), ImVec2(x3, y3),
			static_cast<ImU32>(color));
		return 0;
	}

	int l_cathack_draw_quad(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x1 = static_cast<float>(luaL_checknumber(L, 1));
		const float y1 = static_cast<float>(luaL_checknumber(L, 2));
		const float x2 = static_cast<float>(luaL_checknumber(L, 3));
		const float y2 = static_cast<float>(luaL_checknumber(L, 4));
		const float x3 = static_cast<float>(luaL_checknumber(L, 5));
		const float y3 = static_cast<float>(luaL_checknumber(L, 6));
		const float x4 = static_cast<float>(luaL_checknumber(L, 7));
		const float y4 = static_cast<float>(luaL_checknumber(L, 8));
		const lua_Integer color = luaL_optinteger(L, 9, 0xFFFFFFFFLL);
		const float thickness   = static_cast<float>(luaL_optnumber(L, 10, 1.0));
		dl->AddQuad(ImVec2(x1, y1), ImVec2(x2, y2), ImVec2(x3, y3), ImVec2(x4, y4),
			static_cast<ImU32>(color), thickness);
		return 0;
	}

	int l_cathack_draw_quad_filled(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x1 = static_cast<float>(luaL_checknumber(L, 1));
		const float y1 = static_cast<float>(luaL_checknumber(L, 2));
		const float x2 = static_cast<float>(luaL_checknumber(L, 3));
		const float y2 = static_cast<float>(luaL_checknumber(L, 4));
		const float x3 = static_cast<float>(luaL_checknumber(L, 5));
		const float y3 = static_cast<float>(luaL_checknumber(L, 6));
		const float x4 = static_cast<float>(luaL_checknumber(L, 7));
		const float y4 = static_cast<float>(luaL_checknumber(L, 8));
		const lua_Integer color = luaL_optinteger(L, 9, 0xFFFFFFFFLL);
		dl->AddQuadFilled(ImVec2(x1, y1), ImVec2(x2, y2), ImVec2(x3, y3), ImVec2(x4, y4),
			static_cast<ImU32>(color));
		return 0;
	}

	int l_cathack_draw_polyline(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		// First arg is a 1-indexed array of {x, y} sub-tables. Walk it,
		// accumulating into a stack-allocated vector. ImGui's AddPolyline
		// takes a contiguous ImVec2*, so we need to collect first.
		luaL_checktype(L, 1, LUA_TTABLE);
		const lua_Integer color = luaL_optinteger(L, 2, 0xFFFFFFFFLL);
		const float thickness   = static_cast<float>(luaL_optnumber(L, 3, 1.0));
		const bool closed       = lua_toboolean(L, 4) != 0;

		std::vector<ImVec2> pts;
		const int n = static_cast<int>(lua_rawlen(L, 1));
		pts.reserve(n);
		for (int i = 1; i <= n; ++i)
		{
			lua_rawgeti(L, 1, i);
			if (lua_type(L, -1) == LUA_TTABLE)
			{
				lua_rawgeti(L, -1, 1); float x = static_cast<float>(luaL_optnumber(L, -1, 0.0)); lua_pop(L, 1);
				lua_rawgeti(L, -1, 2); float y = static_cast<float>(luaL_optnumber(L, -1, 0.0)); lua_pop(L, 1);
				pts.push_back(ImVec2(x, y));
			}
			lua_pop(L, 1);
		}
		if (pts.size() >= 2)
		{
			dl->AddPolyline(pts.data(), static_cast<int>(pts.size()),
				static_cast<ImU32>(color),
				closed ? ImDrawFlags_Closed : 0,
				thickness);
		}
		return 0;
	}

	int l_cathack_draw_bezier(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		// Cubic bezier: 4 control points + color/thickness/segments.
		const float x1 = static_cast<float>(luaL_checknumber(L, 1));
		const float y1 = static_cast<float>(luaL_checknumber(L, 2));
		const float x2 = static_cast<float>(luaL_checknumber(L, 3));
		const float y2 = static_cast<float>(luaL_checknumber(L, 4));
		const float x3 = static_cast<float>(luaL_checknumber(L, 5));
		const float y3 = static_cast<float>(luaL_checknumber(L, 6));
		const float x4 = static_cast<float>(luaL_checknumber(L, 7));
		const float y4 = static_cast<float>(luaL_checknumber(L, 8));
		const lua_Integer color = luaL_optinteger(L, 9, 0xFFFFFFFFLL);
		const float thickness   = static_cast<float>(luaL_optnumber(L, 10, 1.0));
		const int segments      = static_cast<int>(luaL_optinteger(L, 11, 0));
		dl->AddBezierCubic(
			ImVec2(x1, y1), ImVec2(x2, y2), ImVec2(x3, y3), ImVec2(x4, y4),
			static_cast<ImU32>(color), thickness, segments);
		return 0;
	}

	int l_cathack_measure_text(lua_State* L)
	{
		const char* s = luaL_checkstring(L, 1);
		const ImVec2 sz = ImGui::CalcTextSize(s);
		lua_pushnumber(L, sz.x);
		lua_pushnumber(L, sz.y);
		return 2;
	}

	int l_cathack_world_text(lua_State* L)
	{
		// Convenience: project a world point and draw text at its screen
		// pixel. Returns the pixel position (x, y, true) on success or nil
		// on off-screen so the caller can early-out.
		ImDrawList* dl = tl_currentDrawList;
		if (!dl || !GVars.PlayerController) { lua_pushnil(L); return 1; }
		const float wx = static_cast<float>(luaL_checknumber(L, 1));
		const float wy = static_cast<float>(luaL_checknumber(L, 2));
		const float wz = static_cast<float>(luaL_checknumber(L, 3));
		const char* text = luaL_checkstring(L, 4);
		const lua_Integer color = luaL_optinteger(L, 5, 0xFFFFFFFFLL);
		const float ofy = static_cast<float>(luaL_optnumber(L, 6, 0.0));

		FVector2D sp;
		if (!GVars.PlayerController->ProjectWorldLocationToScreen(FVector{wx, wy, wz}, &sp, true))
		{
			lua_pushnil(L);
			return 1;
		}
		dl->AddText(ImVec2(sp.X, sp.Y + ofy), static_cast<ImU32>(color), text);
		lua_pushnumber(L, sp.X);
		lua_pushnumber(L, sp.Y);
		lua_pushboolean(L, 1);
		return 3;
	}

	// ====================================================================
	// Persistent storage. Simple TSV: "key\tT\tvalue\n" where T is one of
	// 's' (string), 'n' (number), or 'b' (bool 0/1). We re-write the whole
	// file on every store(), which is fine for the small KV workloads
	// these scripts deal with.
	// ====================================================================

	std::filesystem::path GetStorePath()
	{
		return GetLuaDirectory() / "cathack.store";
	}

	std::unordered_map<std::string, std::string> g_storeCache;
	bool g_storeLoaded = false;

	void StoreLoad_locked()
	{
		if (g_storeLoaded) return;
		g_storeLoaded = true;
		g_storeCache.clear();
		const std::string raw = ReadFileAll(GetStorePath());
		if (raw.empty()) return;
		std::stringstream ss(raw);
		std::string line;
		while (std::getline(ss, line))
		{
			if (line.empty() || line[0] == '#') continue;
			g_storeCache[line] = ""; // placeholder; we'll overwrite below
			// Rewrite as: key | T | value
			size_t t1 = line.find('\t');
			if (t1 == std::string::npos) continue;
			size_t t2 = line.find('\t', t1 + 1);
			if (t2 == std::string::npos) continue;
			std::string key = line.substr(0, t1);
			std::string typed = line.substr(t1 + 1, t2 - t1 - 1) + "\t" + line.substr(t2 + 1);
			g_storeCache[key] = typed;
		}
	}

	void StoreFlush_locked()
	{
		std::ofstream f(GetStorePath(), std::ios::binary | std::ios::trunc);
		if (!f) return;
		for (const auto& kv : g_storeCache)
			f << kv.first << "\t" << kv.second << "\n";
	}

	int l_cathack_store(lua_State* L)
	{
		const char* key = luaL_checkstring(L, 1);
		StoreLoad_locked();

		if (lua_isnoneornil(L, 2))
		{
			g_storeCache.erase(key);
			StoreFlush_locked();
			return 0;
		}

		std::string typed;
		switch (lua_type(L, 2))
		{
		case LUA_TBOOLEAN:
			typed = std::string("b\t") + (lua_toboolean(L, 2) ? "1" : "0");
			break;
		case LUA_TNUMBER:
		{
			char buf[64];
			snprintf(buf, sizeof(buf), "n\t%.17g", lua_tonumber(L, 2));
			typed = buf;
			break;
		}
		default:
		{
			size_t len = 0;
			const char* s = luaL_checklstring(L, 2, &len);
			// Tabs / newlines in the value would corrupt the format. Reject
			// them so the script gets a clear error rather than mysterious
			// missing entries on next load.
			for (size_t i = 0; i < len; ++i)
			{
				if (s[i] == '\t' || s[i] == '\n' || s[i] == '\r')
					return luaL_error(L, "cathack.store: value must not contain tab/newline characters");
			}
			typed = "s\t" + std::string(s, len);
			break;
		}
		}
		g_storeCache[key] = typed;
		StoreFlush_locked();
		return 0;
	}

	int l_cathack_fetch(lua_State* L)
	{
		const char* key = luaL_checkstring(L, 1);
		StoreLoad_locked();
		auto it = g_storeCache.find(key);
		if (it == g_storeCache.end()) { lua_pushnil(L); return 1; }
		const std::string& typed = it->second;
		if (typed.size() < 2) { lua_pushnil(L); return 1; }
		const char tag = typed[0];
		const std::string val = typed.substr(2);
		switch (tag)
		{
		case 'b': lua_pushboolean(L, val == "1" ? 1 : 0); return 1;
		case 'n': lua_pushnumber(L, std::strtod(val.c_str(), nullptr)); return 1;
		case 's': lua_pushlstring(L, val.data(), val.size()); return 1;
		default:  lua_pushnil(L); return 1;
		}
	}

	int l_cathack_store_keys(lua_State* L)
	{
		StoreLoad_locked();
		lua_newtable(L);
		int idx = 1;
		for (const auto& kv : g_storeCache)
		{
			lua_pushstring(L, kv.first.c_str());
			lua_rawseti(L, -2, idx++);
		}
		return 1;
	}

	// ====================================================================
	// Filesystem (sandboxed). All paths are interpreted relative to
	// %APPDATA%\cathack\lua\ — scripts can't escape via "..\\..".
	// ====================================================================

	bool IsSafeFsName(const char* name)
	{
		if (!name || !*name) return false;
		for (const char* p = name; *p; ++p)
		{
			if (*p == '/' || *p == '\\') return false;
			if (*p == ':' || *p == '*' || *p == '?' || *p == '"' || *p == '<' || *p == '>' || *p == '|')
				return false;
		}
		// Reject ".." and leading-dot files.
		if (name[0] == '.') return false;
		return true;
	}

	int l_cathack_script_dir(lua_State* L)
	{
		const std::string p = GetLuaDirectory().string();
		lua_pushstring(L, p.c_str());
		return 1;
	}

	int l_cathack_read_file(lua_State* L)
	{
		const char* nm = luaL_checkstring(L, 1);
		if (!IsSafeFsName(nm)) { lua_pushnil(L); return 1; }
		const std::string s = ReadFileAll(GetLuaDirectory() / nm);
		if (s.empty())
		{
			std::error_code ec;
			if (!std::filesystem::exists(GetLuaDirectory() / nm, ec))
			{
				lua_pushnil(L);
				return 1;
			}
		}
		lua_pushlstring(L, s.data(), s.size());
		return 1;
	}

	int l_cathack_write_file(lua_State* L)
	{
		const char* nm = luaL_checkstring(L, 1);
		if (!IsSafeFsName(nm)) { lua_pushboolean(L, 0); return 1; }
		size_t len = 0;
		const char* data = luaL_checklstring(L, 2, &len);
		const bool ok = WriteFileAll(GetLuaDirectory() / nm, std::string(data, len));
		lua_pushboolean(L, ok ? 1 : 0);
		return 1;
	}

	// ====================================================================
	// Callback registration
	//
	// Both on_tick and on_render return the registry ref number as a
	// "handle". Pass that handle back to untick() / unrender() (or the
	// generic unhook(), which checks both lists) to remove a single
	// callback without nuking the whole VM.
	// ====================================================================

	int l_cathack_on_tick(lua_State* L)
	{
		luaL_checktype(L, 1, LUA_TFUNCTION);
		lua_pushvalue(L, 1);
		const int ref = luaL_ref(L, LUA_REGISTRYINDEX);
		// Tag the registration with the currently-running script so Stop()
		// can find it later. Empty owner = called outside any script run
		// (e.g. from the console or a programmatic injection).
		g_tickRefs.push_back({ref, g_runningScript});
		lua_pushinteger(L, ref);
		return 1;
	}

	int l_cathack_on_render(lua_State* L)
	{
		luaL_checktype(L, 1, LUA_TFUNCTION);
		lua_pushvalue(L, 1);
		const int ref = luaL_ref(L, LUA_REGISTRYINDEX);
		g_renderRefs.push_back({ref, g_runningScript});
		lua_pushinteger(L, ref);
		return 1;
	}

	// Common worker — drop `ref` from `vec` if present and free the Lua
	// registry slot so the closure becomes GC-able. Returns true if the
	// ref was actually in the list.
	bool RemoveRef(std::vector<Callback>& vec, int ref, lua_State* L)
	{
		auto it = std::find_if(vec.begin(), vec.end(),
			[ref](const Callback& c) { return c.ref == ref; });
		if (it == vec.end()) return false;
		vec.erase(it);
		luaL_unref(L, LUA_REGISTRYINDEX, ref);
		return true;
	}

	// Drop every callback owned by `name`. Returns the number freed.
	// Used by Stop() and Run() (the latter auto-stops before re-running
	// so users don't accumulate duplicate hooks across runs). Has to walk
	// every callback fan we've added since this was first introduced
	// (tick / render plus the new event fans for keys / mouse / chars).
	int RemoveCallbacksByOwner_locked(const std::string& name)
	{
		if (!g_L) return 0;
		int n = 0;
		auto sweep = [&](std::vector<Callback>& vec) {
			for (auto it = vec.begin(); it != vec.end(); )
			{
				if (it->owner == name)
				{
					luaL_unref(g_L, LUA_REGISTRYINDEX, it->ref);
					it = vec.erase(it);
					++n;
				}
				else
				{
					++it;
				}
			}
		};
		sweep(g_tickRefs);
		sweep(g_renderRefs);
		sweep(g_keyPressedRefs);
		sweep(g_keyReleasedRefs);
		sweep(g_mousePressedRefs);
		sweep(g_mouseReleasedRefs);
		sweep(g_mouseWheelRefs);
		sweep(g_charRefs);
		return n;
	}

	int l_cathack_untick(lua_State* L)
	{
		const int ref = static_cast<int>(luaL_checkinteger(L, 1));
		const bool ok = RemoveRef(g_tickRefs, ref, L);
		lua_pushboolean(L, ok ? 1 : 0);
		return 1;
	}

	int l_cathack_unrender(lua_State* L)
	{
		const int ref = static_cast<int>(luaL_checkinteger(L, 1));
		const bool ok = RemoveRef(g_renderRefs, ref, L);
		lua_pushboolean(L, ok ? 1 : 0);
		return 1;
	}

	// Generic unhook — handy when you don't remember which list a handle
	// came from (or you stored a mixed bag of handles in one table).
	// Walks every callback fan; RemoveRef short-circuits on misses so the
	// chain of calls is cheap even when the handle isn't registered.
	int l_cathack_unhook(lua_State* L)
	{
		const int ref = static_cast<int>(luaL_checkinteger(L, 1));
		bool any = false;
		any |= RemoveRef(g_tickRefs,           ref, L);
		any |= RemoveRef(g_renderRefs,         ref, L);
		any |= RemoveRef(g_keyPressedRefs,     ref, L);
		any |= RemoveRef(g_keyReleasedRefs,    ref, L);
		any |= RemoveRef(g_mousePressedRefs,   ref, L);
		any |= RemoveRef(g_mouseReleasedRefs,  ref, L);
		any |= RemoveRef(g_mouseWheelRefs,     ref, L);
		any |= RemoveRef(g_charRefs,           ref, L);
		lua_pushboolean(L, any ? 1 : 0);
		return 1;
	}

	// Drop EVERY registered callback across every fan. Same semantics as
	// "Reset VM" minus blowing the lua_State away — globals + functions
	// stay defined, but no callbacks fire until something re-registers.
	int l_cathack_clear_callbacks(lua_State* L)
	{
		auto clearVec = [&](std::vector<Callback>& v) {
			for (const auto& cb : v) luaL_unref(L, LUA_REGISTRYINDEX, cb.ref);
			v.clear();
		};
		clearVec(g_tickRefs);
		clearVec(g_renderRefs);
		clearVec(g_keyPressedRefs);
		clearVec(g_keyReleasedRefs);
		clearVec(g_mousePressedRefs);
		clearVec(g_mouseReleasedRefs);
		clearVec(g_mouseWheelRefs);
		clearVec(g_charRefs);
		return 0;
	}

	int l_cathack_run_string(lua_State* L)
	{
		// Compile and run an arbitrary Lua chunk under the current state.
		// Returns true on success, otherwise (false, error_message). Useful
		// for hot-loading snippets without going through the file picker —
		// a script can build code as a string and run it.
		const char* src = luaL_checkstring(L, 1);
		const char* name = luaL_optstring(L, 2, "=run_string");
		if (luaL_loadbuffer(L, src, std::strlen(src), name) != LUA_OK)
		{
			lua_pushboolean(L, 0);
			lua_insert(L, -2); // (false, error_string)
			return 2;
		}
		// run_string is invoked from inside another pcall (the script that
		// called it), so the watchdog deadline is already armed by the
		// caller. We don't (re-)arm it here — that would falsely extend
		// the original call's budget.
		if (lua_pcall(L, 0, 0, 0) != LUA_OK)
		{
			lua_pushboolean(L, 0);
			lua_insert(L, -2);
			return 2;
		}
		lua_pushboolean(L, 1);
		return 1;
	}

	int l_cathack_script_dir_path(lua_State* L)
	{
		// Alias for script_dir() — scripts that prefer the verbose name.
		const std::string p = GetLuaDirectory().string();
		lua_pushstring(L, p.c_str());
		return 1;
	}

	// =====================================================================
	// UI — mouse / keyboard / menu / capture / events / textures / clipping
	//
	// Everything in this block is meant to support scripts that build their
	// own interactive UI on top of the existing draw_* primitives. Most of
	// the read-side state comes straight from ImGuiIO so it's already
	// frame-synchronized with the menu the rest of the cheat draws.
	// =====================================================================

	// --- Frame counter (read-only, ticks once per TickRender) ---------------
	std::atomic<lua_Integer> g_frameCount{0};

	int l_cathack_frame_count(lua_State* L)
	{
		lua_pushinteger(L, g_frameCount.load());
		return 1;
	}

	// --- Mouse polling ------------------------------------------------------
	// Button IDs match the public API documented in the cathack.* surface:
	// 0=left, 1=right, 2=middle, 3=X1, 4=X2 (which is also exactly how
	// ImGui orders them in its Mouse* APIs, so we forward straight through).
	int l_cathack_mouse_pos(lua_State* L)
	{
		const ImGuiIO& io = ImGui::GetIO();
		lua_pushnumber(L, io.MousePos.x);
		lua_pushnumber(L, io.MousePos.y);
		return 2;
	}

	int l_cathack_mouse_delta(lua_State* L)
	{
		const ImGuiIO& io = ImGui::GetIO();
		lua_pushnumber(L, io.MouseDelta.x);
		lua_pushnumber(L, io.MouseDelta.y);
		return 2;
	}

	int l_cathack_mouse_down(lua_State* L)
	{
		const int btn = static_cast<int>(luaL_optinteger(L, 1, 0));
		if (btn < 0 || btn >= IM_ARRAYSIZE(ImGui::GetIO().MouseDown))
		{
			lua_pushboolean(L, 0);
			return 1;
		}
		lua_pushboolean(L, ImGui::IsMouseDown(btn) ? 1 : 0);
		return 1;
	}

	int l_cathack_mouse_clicked(lua_State* L)
	{
		const int btn = static_cast<int>(luaL_optinteger(L, 1, 0));
		const bool repeat = lua_toboolean(L, 2) != 0;
		if (btn < 0 || btn >= 5) { lua_pushboolean(L, 0); return 1; }
		lua_pushboolean(L, ImGui::IsMouseClicked(btn, repeat) ? 1 : 0);
		return 1;
	}

	int l_cathack_mouse_released(lua_State* L)
	{
		const int btn = static_cast<int>(luaL_optinteger(L, 1, 0));
		if (btn < 0 || btn >= 5) { lua_pushboolean(L, 0); return 1; }
		lua_pushboolean(L, ImGui::IsMouseReleased(btn) ? 1 : 0);
		return 1;
	}

	int l_cathack_mouse_double_clicked(lua_State* L)
	{
		const int btn = static_cast<int>(luaL_optinteger(L, 1, 0));
		if (btn < 0 || btn >= 5) { lua_pushboolean(L, 0); return 1; }
		lua_pushboolean(L, ImGui::IsMouseDoubleClicked(btn) ? 1 : 0);
		return 1;
	}

	int l_cathack_mouse_wheel(lua_State* L)
	{
		const ImGuiIO& io = ImGui::GetIO();
		// Vertical wheel as primary return; horizontal as secondary so
		// scripts that care about both can read both, while the simple
		// `local w = cathack.mouse_wheel()` idiom still works.
		lua_pushnumber(L, io.MouseWheel);
		lua_pushnumber(L, io.MouseWheelH);
		return 2;
	}

	int l_cathack_mouse_drag_delta(lua_State* L)
	{
		const int   btn       = static_cast<int>(luaL_optinteger(L, 1, 0));
		const float threshold = static_cast<float>(luaL_optnumber(L, 2, -1.0));
		if (btn < 0 || btn >= 5) { lua_pushnumber(L, 0.0); lua_pushnumber(L, 0.0); return 2; }
		const ImVec2 d = ImGui::GetMouseDragDelta(btn, threshold);
		lua_pushnumber(L, d.x);
		lua_pushnumber(L, d.y);
		return 2;
	}

	int l_cathack_mouse_in_rect(lua_State* L)
	{
		const float x = static_cast<float>(luaL_checknumber(L, 1));
		const float y = static_cast<float>(luaL_checknumber(L, 2));
		const float w = static_cast<float>(luaL_checknumber(L, 3));
		const float h = static_cast<float>(luaL_checknumber(L, 4));
		const ImVec2 p = ImGui::GetIO().MousePos;
		const bool in = (p.x >= x) && (p.x < x + w) && (p.y >= y) && (p.y < y + h);
		lua_pushboolean(L, in ? 1 : 0);
		return 1;
	}

	// --- Cursor visibility --------------------------------------------------
	// Software cursor (io.MouseDrawCursor) is what ImGui uses to render its
	// own arrow when the host hides the OS cursor. Lua scripts that want
	// to show the cursor for a custom UI flip this on; the OS cursor is
	// handled separately by the WndProc so we don't fight it.
	int l_cathack_set_cursor_visible(lua_State* L)
	{
		const bool v = lua_toboolean(L, 1) != 0;
		if (ImGui::GetCurrentContext())
			ImGui::GetIO().MouseDrawCursor = v;
		return 0;
	}

	int l_cathack_cursor_visible(lua_State* L)
	{
		bool v = false;
		if (ImGui::GetCurrentContext())
			v = ImGui::GetIO().MouseDrawCursor;
		lua_pushboolean(L, v ? 1 : 0);
		return 1;
	}

	// --- Input capture ------------------------------------------------------
	// Both flags are *per-frame requests* — same semantics as
	// ImGui::SetNextFrameWantCaptureMouse. Reset to false at the start of
	// every render frame, scripts re-set them each frame they want to
	// claim input. The host's WndProc reads these (plus ImGui's own
	// WantCapture flags) and swallows the corresponding messages so the
	// game underneath doesn't see them.
	int l_cathack_capture_mouse(lua_State* L)
	{
		const bool v = lua_isnoneornil(L, 1) ? true : (lua_toboolean(L, 1) != 0);
		CathackHostBridge::g_LuaCaptureMouseReq.store(v);
		ImGui::SetNextFrameWantCaptureMouse(v);
		return 0;
	}

	int l_cathack_capture_keyboard(lua_State* L)
	{
		const bool v = lua_isnoneornil(L, 1) ? true : (lua_toboolean(L, 1) != 0);
		CathackHostBridge::g_LuaCaptureKbdReq.store(v);
		ImGui::SetNextFrameWantCaptureKeyboard(v);
		return 0;
	}

	int l_cathack_want_capture_mouse(lua_State* L)
	{
		bool v = CathackHostBridge::g_LuaCaptureMouseReq.load();
		if (ImGui::GetCurrentContext() && ImGui::GetIO().WantCaptureMouse)
			v = true;
		lua_pushboolean(L, v ? 1 : 0);
		return 1;
	}

	int l_cathack_want_capture_keyboard(lua_State* L)
	{
		bool v = CathackHostBridge::g_LuaCaptureKbdReq.load();
		if (ImGui::GetCurrentContext() && ImGui::GetIO().WantCaptureKeyboard)
			v = true;
		lua_pushboolean(L, v ? 1 : 0);
		return 1;
	}

	// --- Main cheat menu ----------------------------------------------------
	int l_cathack_menu_open(lua_State* L)
	{
		lua_pushboolean(L, CathackHostBridge::IsMenuOpen() ? 1 : 0);
		return 1;
	}

	int l_cathack_set_menu_open(lua_State* L)
	{
		const bool v = lua_toboolean(L, 1) != 0;
		CathackHostBridge::SetMenuOpen(v);
		return 0;
	}

	int l_cathack_toggle_menu(lua_State* L)
	{
		(void)L;
		CathackHostBridge::ToggleMenu();
		return 0;
	}

	// --- Keyboard polling (edge-triggered) ---------------------------------
	// `cathack.key()` already exists for IsKeyDown semantics; these add
	// "pressed this frame" / "released this frame" so scripts can build
	// click handlers without storing previous state themselves.
	ImGuiKey VkToImGuiKey(int vk)
	{
		// ImGui's helper is the official one, but it only takes WPARAM
		// values directly via the impl backend. The keycode arg here is
		// the same Win32 VK_* the rest of the cheat uses, which is what
		// ImGui_ImplWin32_KeyEventToImGuiKey takes. The header doesn't
		// publicly export it, so we reimplement the small subset we need.
		// Most ASCII / function / arrow / modifier keys map 1:1.
		if (vk >= '0' && vk <= '9') return static_cast<ImGuiKey>(ImGuiKey_0 + (vk - '0'));
		if (vk >= 'A' && vk <= 'Z') return static_cast<ImGuiKey>(ImGuiKey_A + (vk - 'A'));
		if (vk >= VK_F1 && vk <= VK_F12) return static_cast<ImGuiKey>(ImGuiKey_F1 + (vk - VK_F1));
		switch (vk)
		{
		case VK_TAB:     return ImGuiKey_Tab;
		case VK_LEFT:    return ImGuiKey_LeftArrow;
		case VK_RIGHT:   return ImGuiKey_RightArrow;
		case VK_UP:      return ImGuiKey_UpArrow;
		case VK_DOWN:    return ImGuiKey_DownArrow;
		case VK_PRIOR:   return ImGuiKey_PageUp;
		case VK_NEXT:    return ImGuiKey_PageDown;
		case VK_HOME:    return ImGuiKey_Home;
		case VK_END:     return ImGuiKey_End;
		case VK_INSERT:  return ImGuiKey_Insert;
		case VK_DELETE:  return ImGuiKey_Delete;
		case VK_BACK:    return ImGuiKey_Backspace;
		case VK_SPACE:   return ImGuiKey_Space;
		case VK_RETURN:  return ImGuiKey_Enter;
		case VK_ESCAPE:  return ImGuiKey_Escape;
		case VK_SHIFT:   return ImGuiKey_LeftShift;
		case VK_CONTROL: return ImGuiKey_LeftCtrl;
		case VK_MENU:    return ImGuiKey_LeftAlt;
		case VK_LWIN:    return ImGuiKey_LeftSuper;
		case VK_RWIN:    return ImGuiKey_RightSuper;
		case VK_CAPITAL: return ImGuiKey_CapsLock;
		case VK_NUMLOCK: return ImGuiKey_NumLock;
		case VK_OEM_1:   return ImGuiKey_Semicolon;
		case VK_OEM_PLUS:return ImGuiKey_Equal;
		case VK_OEM_COMMA:return ImGuiKey_Comma;
		case VK_OEM_MINUS:return ImGuiKey_Minus;
		case VK_OEM_PERIOD:return ImGuiKey_Period;
		case VK_OEM_2:   return ImGuiKey_Slash;
		case VK_OEM_3:   return ImGuiKey_GraveAccent;
		case VK_OEM_4:   return ImGuiKey_LeftBracket;
		case VK_OEM_5:   return ImGuiKey_Backslash;
		case VK_OEM_6:   return ImGuiKey_RightBracket;
		case VK_OEM_7:   return ImGuiKey_Apostrophe;
		default: return ImGuiKey_None;
		}
	}

	int l_cathack_key_pressed(lua_State* L)
	{
		const int vk = static_cast<int>(luaL_checkinteger(L, 1));
		const bool repeat = lua_toboolean(L, 2) != 0;
		const ImGuiKey k = VkToImGuiKey(vk);
		bool pressed = false;
		if (k != ImGuiKey_None)
			pressed = ImGui::IsKeyPressed(k, repeat);
		lua_pushboolean(L, pressed ? 1 : 0);
		return 1;
	}

	int l_cathack_key_released(lua_State* L)
	{
		const int vk = static_cast<int>(luaL_checkinteger(L, 1));
		const ImGuiKey k = VkToImGuiKey(vk);
		bool released = false;
		if (k != ImGuiKey_None)
			released = ImGui::IsKeyReleased(k);
		lua_pushboolean(L, released ? 1 : 0);
		return 1;
	}

	// --- Char input ---------------------------------------------------------
	// chars_typed() drains the per-frame WM_CHAR queue maintained by the
	// host's WndProc and returns it as a UTF-8 string. Idempotent within a
	// single frame: subsequent calls in the same frame see an empty
	// string (the queue was already drained). Scripts can also subscribe
	// via on_char(...) to get one callback per codepoint; chars_typed() is
	// just the polling-style alternative.
	std::string Utf32ToUtf8(unsigned int cp)
	{
		std::string out;
		if (cp < 0x80) { out.push_back(static_cast<char>(cp)); return out; }
		if (cp < 0x800)
		{
			out.push_back(static_cast<char>(0xC0 | (cp >> 6)));
			out.push_back(static_cast<char>(0x80 | (cp & 0x3F)));
			return out;
		}
		if (cp < 0x10000)
		{
			out.push_back(static_cast<char>(0xE0 | (cp >> 12)));
			out.push_back(static_cast<char>(0x80 | ((cp >> 6) & 0x3F)));
			out.push_back(static_cast<char>(0x80 | (cp & 0x3F)));
			return out;
		}
		out.push_back(static_cast<char>(0xF0 | (cp >> 18)));
		out.push_back(static_cast<char>(0x80 | ((cp >> 12) & 0x3F)));
		out.push_back(static_cast<char>(0x80 | ((cp >> 6) & 0x3F)));
		out.push_back(static_cast<char>(0x80 | (cp & 0x3F)));
		return out;
	}

	// Lua-side char queue. WndProc-side queue is drained at the top of
	// TickRender; we hold a copy here so chars_typed() (polling) and
	// on_char (callbacks) both see the same set of codepoints.
	std::vector<unsigned int> g_charsThisFrame;

	// Per-frame ordered input events (chars + key transitions). Drained
	// from the host bridge at the top of TickRender. Used by
	// cathack.input_events(), cathack.key_events(), and cathack.text_input().
	std::vector<CathackHostBridge::LuaInputEvent> g_inputEventsThisFrame;

	int l_cathack_chars_typed(lua_State* L)
	{
		std::string out;
		for (unsigned int cp : g_charsThisFrame)
			out += Utf32ToUtf8(cp);
		lua_pushlstring(L, out.data(), out.size());
		return 1;
	}

	// chars_typed_printable() — same payload as chars_typed() but with
	// every C0/DEL control byte stripped. Backspace, tab, carriage
	// return, linefeed, escape and friends arrive as WM_CHAR codepoints
	// 0x00..0x1F / 0x7F; in a text-input loop you almost never want
	// those concatenated raw onto the buffer (the classic bug of
	// `state.text = state.text .. cathack.chars_typed()` appending a
	// literal '\b' instead of erasing). Use this helper when you only
	// want printable insertion text and are reading navigation keys
	// from key_events() / text_input().
	int l_cathack_chars_typed_printable(lua_State* L)
	{
		std::string out;
		for (unsigned int cp : g_charsThisFrame)
		{
			if (cp < 0x20 || cp == 0x7F) continue;
			out += Utf32ToUtf8(cp);
		}
		lua_pushlstring(L, out.data(), out.size());
		return 1;
	}

	// Helper for the input_events / key_events bindings. Pushes a single
	// event row table at the top of the stack. Layout:
	//   { kind = "char"|"key_down"|"key_up",
	//     codepoint? = int,                -- only for kind == "char"
	//     vk?        = int,                -- only for kind == "key_down"|"key_up"
	//     text?      = "x",                -- only for kind == "char"
	//     repeated   = bool,
	//     shift / ctrl / alt = bool }
	void PushInputEventRow(lua_State* L, const CathackHostBridge::LuaInputEvent& ev)
	{
		lua_createtable(L, 0, 7);
		const char* kind = (ev.kind == 0) ? "char" : (ev.kind == 1 ? "key_down" : "key_up");
		lua_pushstring(L, kind);                            lua_setfield(L, -2, "kind");
		if (ev.kind == 0)
		{
			lua_pushinteger(L, static_cast<lua_Integer>(ev.code)); lua_setfield(L, -2, "codepoint");
			const std::string utf8 = Utf32ToUtf8(ev.code);
			lua_pushlstring(L, utf8.data(), utf8.size());          lua_setfield(L, -2, "text");
		}
		else
		{
			lua_pushinteger(L, static_cast<lua_Integer>(ev.code)); lua_setfield(L, -2, "vk");
		}
		lua_pushboolean(L, ev.repeated ? 1 : 0); lua_setfield(L, -2, "repeated");
		lua_pushboolean(L, ev.shift    ? 1 : 0); lua_setfield(L, -2, "shift");
		lua_pushboolean(L, ev.ctrl     ? 1 : 0); lua_setfield(L, -2, "ctrl");
		lua_pushboolean(L, ev.alt      ? 1 : 0); lua_setfield(L, -2, "alt");
	}

	// cathack.input_events() -> array of { kind, codepoint|vk, text?, repeated, shift, ctrl, alt }
	//
	// Returns the full per-frame ordered stream so a script doing custom
	// text editing or hotkey logic can interleave printable insertion and
	// caret navigation correctly. Idempotent: subsequent calls in the
	// same frame return the same array (the snapshot is taken at the top
	// of TickRender).
	int l_cathack_input_events(lua_State* L)
	{
		const auto& events = g_inputEventsThisFrame;
		lua_createtable(L, static_cast<int>(events.size()), 0);
		for (size_t i = 0; i < events.size(); ++i)
		{
			PushInputEventRow(L, events[i]);
			lua_rawseti(L, -2, static_cast<int>(i + 1));
		}
		return 1;
	}

	// cathack.key_events() -> array of { vk, kind = "key_down"|"key_up", repeated, shift, ctrl, alt }
	//
	// Same data as input_events but filtered to key transitions only —
	// the right tool for "did the user press an arrow / backspace /
	// delete this frame, including auto-repeat?" Hold-to-repeat works
	// because every WM_KEYDOWN with the repeat bit set lands here.
	int l_cathack_key_events(lua_State* L)
	{
		int idx = 0;
		lua_newtable(L);
		for (const auto& ev : g_inputEventsThisFrame)
		{
			if (ev.kind == 0) continue;
			PushInputEventRow(L, ev);
			lua_rawseti(L, -2, ++idx);
		}
		return 1;
	}

	// ----------------------------------------------------------------
	// cathack.text_input(state, opts?)
	//
	// Drop-in helper that turns this frame's input events into a single
	// updated text-edit state. Designed so a Lua text input field is one
	// call per frame:
	//
	//   state = state or { text = "", caret = 0 }
	//   cathack.text_input(state)
	//
	// State fields (all optional on input, always set on output):
	//   text       = string             — current buffer
	//   caret      = int (0..#text)     — cursor position
	//   sel_anchor = int|nil            — if set, marks the other end of
	//                                     a selection (caret is the live
	//                                     end). The selection range is
	//                                     [min(caret, sel_anchor), max(...)].
	//   submit     = bool|nil           — set true on output when Enter
	//                                     was pressed this frame
	//
	// Opts:
	//   max_length = int                — clamp inserts past this length
	//   multiline  = bool               — when false (default) Enter sets
	//                                     `submit` and never inserts a '\n';
	//                                     when true, Enter inserts '\n'
	//
	// Handles printable insert, backspace, delete, left/right arrows
	// (with shift = grow selection, ctrl = word jump), home/end,
	// ctrl+A select-all, ctrl+C copy, ctrl+X cut, ctrl+V paste. All
	// auto-repeat naturally because they consume key-down events
	// (initial AND repeat-flag-set) from the frame queue.
	//
	// The function mutates `state` in-place AND returns it for chaining.
	// ----------------------------------------------------------------
	int l_cathack_text_input(lua_State* L)
	{
		luaL_checktype(L, 1, LUA_TTABLE);

		// Read state.
		lua_getfield(L, 1, "text");
		std::string text = (lua_type(L, -1) == LUA_TSTRING) ? lua_tostring(L, -1) : "";
		lua_pop(L, 1);

		auto clampInt = [](long long v, long long lo, long long hi) -> long long {
			if (v < lo) return lo; if (v > hi) return hi; return v;
		};

		long long caret = 0;
		lua_getfield(L, 1, "caret");
		if (lua_type(L, -1) == LUA_TNUMBER) caret = lua_tointeger(L, -1);
		lua_pop(L, 1);
		caret = clampInt(caret, 0, static_cast<long long>(text.size()));

		long long anchor = -1; // -1 = no selection
		lua_getfield(L, 1, "sel_anchor");
		if (lua_type(L, -1) == LUA_TNUMBER)
			anchor = clampInt(lua_tointeger(L, -1), 0, static_cast<long long>(text.size()));
		lua_pop(L, 1);

		// Read opts.
		long long maxLength = -1;
		bool multiline = false;
		if (lua_type(L, 2) == LUA_TTABLE)
		{
			lua_getfield(L, 2, "max_length");
			if (lua_type(L, -1) == LUA_TNUMBER) maxLength = lua_tointeger(L, -1);
			lua_pop(L, 1);
			lua_getfield(L, 2, "multiline");
			if (lua_type(L, -1) == LUA_TBOOLEAN) multiline = lua_toboolean(L, -1) != 0;
			lua_pop(L, 1);
		}

		bool submit = false;

		// --- Local helpers ----------------------------------------
		auto hasSelection = [&]() { return anchor >= 0 && anchor != caret; };
		auto selRange     = [&]() -> std::pair<size_t, size_t> {
			const long long a = anchor < 0 ? caret : anchor;
			const long long lo = a < caret ? a : caret;
			const long long hi = a < caret ? caret : a;
			return { static_cast<size_t>(lo), static_cast<size_t>(hi) };
		};
		auto deleteSelection = [&]() {
			if (!hasSelection()) return;
			const auto [lo, hi] = selRange();
			text.erase(lo, hi - lo);
			caret = static_cast<long long>(lo);
			anchor = -1;
		};
		auto insertText = [&](const std::string& ins) {
			deleteSelection();
			std::string s = ins;
			if (maxLength >= 0)
			{
				const long long room = maxLength - static_cast<long long>(text.size());
				if (room <= 0) return;
				if (static_cast<long long>(s.size()) > room) s.resize(static_cast<size_t>(room));
			}
			text.insert(static_cast<size_t>(caret), s);
			caret += static_cast<long long>(s.size());
			anchor = -1;
		};
		// "Word boundary" for ctrl-arrow movement. Walks past whitespace
		// then word chars (or vice versa) for one Ctrl+Left/Ctrl+Right step.
		auto isWord = [](unsigned char c) {
			return (c >= '0' && c <= '9') || (c >= 'A' && c <= 'Z')
				|| (c >= 'a' && c <= 'z') || c == '_' || c >= 0x80;
		};
		auto wordPrev = [&](long long pos) -> long long {
			while (pos > 0 && std::isspace(static_cast<unsigned char>(text[pos - 1]))) --pos;
			while (pos > 0 && isWord(static_cast<unsigned char>(text[pos - 1])))      --pos;
			if (pos == 0) return 0;
			// One non-word punctuation char if we didn't move — keeps
			// "foo!!!|" + ctrl+left from sticking on the punctuation.
			if (!std::isspace(static_cast<unsigned char>(text[pos - 1]))
				&& !isWord(static_cast<unsigned char>(text[pos - 1])))
				--pos;
			return pos;
		};
		auto wordNext = [&](long long pos) -> long long {
			const long long n = static_cast<long long>(text.size());
			while (pos < n && std::isspace(static_cast<unsigned char>(text[pos]))) ++pos;
			while (pos < n && isWord(static_cast<unsigned char>(text[pos])))      ++pos;
			if (pos < n
				&& !std::isspace(static_cast<unsigned char>(text[pos]))
				&& !isWord(static_cast<unsigned char>(text[pos])))
				++pos;
			return pos;
		};
		auto move = [&](long long newCaret, bool shift) {
			newCaret = clampInt(newCaret, 0, static_cast<long long>(text.size()));
			if (shift)
			{
				if (anchor < 0) anchor = caret;
				caret = newCaret;
				if (anchor == caret) anchor = -1;
			}
			else
			{
				caret = newCaret;
				anchor = -1;
			}
		};
		auto utf8PrevBoundary = [&](long long pos) -> long long {
			if (pos <= 0) return 0;
			--pos;
			while (pos > 0 && (static_cast<unsigned char>(text[pos]) & 0xC0) == 0x80) --pos;
			return pos;
		};
		auto utf8NextBoundary = [&](long long pos) -> long long {
			const long long n = static_cast<long long>(text.size());
			if (pos >= n) return n;
			++pos;
			while (pos < n && (static_cast<unsigned char>(text[pos]) & 0xC0) == 0x80) ++pos;
			return pos;
		};

		// --- Walk the per-frame events in arrival order. ----------
		for (const auto& ev : g_inputEventsThisFrame)
		{
			if (ev.kind == 0)
			{
				const unsigned int cp = ev.code;

				// Skip control codes and DEL — those are produced as
				// WM_CHAR siblings of WM_KEYDOWN navigation events and
				// the key_down branch below handles them properly.
				if (cp < 0x20 || cp == 0x7F) continue;

				// Ctrl+letter is a chord, not insertion — let the
				// key_down branch handle Ctrl+A/C/V/X.
				if (ev.ctrl && !ev.alt) continue;

				insertText(Utf32ToUtf8(cp));
			}
			else if (ev.kind == 1)
			{
				const unsigned int vk = ev.code;
				switch (vk)
				{
				case VK_BACK:
					if (hasSelection()) { deleteSelection(); break; }
					if (caret > 0)
					{
						const long long prev = utf8PrevBoundary(caret);
						text.erase(static_cast<size_t>(prev),
							static_cast<size_t>(caret - prev));
						caret = prev;
					}
					break;
				case VK_DELETE:
					if (hasSelection()) { deleteSelection(); break; }
					if (caret < static_cast<long long>(text.size()))
					{
						const long long nxt = utf8NextBoundary(caret);
						text.erase(static_cast<size_t>(caret),
							static_cast<size_t>(nxt - caret));
					}
					break;
				case VK_LEFT:
					if (ev.ctrl) move(wordPrev(caret), ev.shift);
					else         move(utf8PrevBoundary(caret), ev.shift);
					break;
				case VK_RIGHT:
					if (ev.ctrl) move(wordNext(caret), ev.shift);
					else         move(utf8NextBoundary(caret), ev.shift);
					break;
				case VK_HOME:
					move(0, ev.shift);
					break;
				case VK_END:
					move(static_cast<long long>(text.size()), ev.shift);
					break;
				case 'A':
					if (ev.ctrl)
					{
						anchor = 0;
						caret  = static_cast<long long>(text.size());
					}
					break;
				case 'C':
					if (ev.ctrl && hasSelection())
					{
						const auto [lo, hi] = selRange();
						const std::string sel = text.substr(lo, hi - lo);
						if (OpenClipboard(nullptr))
						{
							EmptyClipboard();
							const HGLOBAL h = GlobalAlloc(GMEM_MOVEABLE, sel.size() + 1);
							if (h)
							{
								if (auto* dst = static_cast<char*>(GlobalLock(h)))
								{
									std::memcpy(dst, sel.data(), sel.size());
									dst[sel.size()] = 0;
									GlobalUnlock(h);
									SetClipboardData(CF_TEXT, h);
								}
								else GlobalFree(h);
							}
							CloseClipboard();
						}
					}
					break;
				case 'X':
					if (ev.ctrl && hasSelection())
					{
						const auto [lo, hi] = selRange();
						const std::string sel = text.substr(lo, hi - lo);
						if (OpenClipboard(nullptr))
						{
							EmptyClipboard();
							const HGLOBAL h = GlobalAlloc(GMEM_MOVEABLE, sel.size() + 1);
							if (h)
							{
								if (auto* dst = static_cast<char*>(GlobalLock(h)))
								{
									std::memcpy(dst, sel.data(), sel.size());
									dst[sel.size()] = 0;
									GlobalUnlock(h);
									SetClipboardData(CF_TEXT, h);
								}
								else GlobalFree(h);
							}
							CloseClipboard();
						}
						deleteSelection();
					}
					break;
				case 'V':
					if (ev.ctrl)
					{
						std::string paste;
						if (OpenClipboard(nullptr))
						{
							if (HANDLE h = GetClipboardData(CF_TEXT))
							{
								if (auto* src = static_cast<const char*>(GlobalLock(h)))
								{
									paste.assign(src);
									GlobalUnlock(h);
								}
							}
							CloseClipboard();
						}
						if (!paste.empty())
						{
							// Strip CRLF -> LF (Windows clipboard pattern)
							// when not multiline, also drop newlines entirely.
							std::string clean;
							clean.reserve(paste.size());
							for (size_t i = 0; i < paste.size(); ++i)
							{
								const char c = paste[i];
								if (c == '\r') continue;
								if (c == '\n' && !multiline) continue;
								clean.push_back(c);
							}
							insertText(clean);
						}
					}
					break;
				case VK_RETURN:
					if (multiline) insertText("\n");
					else           submit = true;
					break;
				case VK_TAB:
					// Tab is usually a focus-traversal control in UI — skip
					// inserting it. Scripts can detect it via key_events()
					// if they want.
					break;
				default:
					break;
				}
			}
			// kind == 2 (key_up) is informational; text_input ignores it.
		}

		// Re-clamp caret in case events shrank the buffer below caret.
		caret = clampInt(caret, 0, static_cast<long long>(text.size()));
		if (anchor >= 0)
			anchor = clampInt(anchor, 0, static_cast<long long>(text.size()));

		// Write state back into the table.
		lua_pushlstring(L, text.data(), text.size()); lua_setfield(L, 1, "text");
		lua_pushinteger(L, static_cast<lua_Integer>(caret)); lua_setfield(L, 1, "caret");
		if (anchor >= 0)
		{
			lua_pushinteger(L, static_cast<lua_Integer>(anchor));
			lua_setfield(L, 1, "sel_anchor");
		}
		else
		{
			lua_pushnil(L);
			lua_setfield(L, 1, "sel_anchor");
		}
		lua_pushboolean(L, submit ? 1 : 0); lua_setfield(L, 1, "submit");

		// Return state for chaining: state = cathack.text_input(state).
		lua_pushvalue(L, 1);
		return 1;
	}

	// --- Clip rects ---------------------------------------------------------
	// Tiny stack of (x, y, w, h) push/pop pairs. Forwarded straight to the
	// active draw list. Calling pop without a matching push is a no-op (we
	// guard with a counter so we don't pop into the host's own clip stack).
	thread_local int g_luaClipStackDepth = 0;

	int l_cathack_push_clip_rect(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x = static_cast<float>(luaL_checknumber(L, 1));
		const float y = static_cast<float>(luaL_checknumber(L, 2));
		const float w = static_cast<float>(luaL_checknumber(L, 3));
		const float h = static_cast<float>(luaL_checknumber(L, 4));
		const bool intersect = lua_isnoneornil(L, 5) ? true : (lua_toboolean(L, 5) != 0);
		dl->PushClipRect(ImVec2(x, y), ImVec2(x + w, y + h), intersect);
		++g_luaClipStackDepth;
		return 0;
	}

	int l_cathack_pop_clip_rect(lua_State* L)
	{
		(void)L;
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		if (g_luaClipStackDepth <= 0) return 0;
		dl->PopClipRect();
		--g_luaClipStackDepth;
		return 0;
	}

	// --- Draw layer (foreground / background) ------------------------------
	// Default is foreground (the same draw list TickRender hands us). Some
	// scripts want to draw "below" the cheat menu — switching to the
	// background list lets them. State is per-render-frame: at the start
	// of each TickRender we re-default to foreground.
	int l_cathack_set_draw_layer(lua_State* L)
	{
		const char* mode = luaL_checkstring(L, 1);
		if (!ImGui::GetCurrentContext()) return 0;
		if (std::strcmp(mode, "foreground") == 0 || std::strcmp(mode, "fg") == 0
			|| std::strcmp(mode, "overlay") == 0 || std::strcmp(mode, "menu") == 0)
		{
			tl_currentDrawList = ImGui::GetForegroundDrawList();
		}
		else if (std::strcmp(mode, "background") == 0 || std::strcmp(mode, "bg") == 0)
		{
			tl_currentDrawList = ImGui::GetBackgroundDrawList();
		}
		return 0;
	}

	// --- Better text drawing -----------------------------------------------
	// Adds size-overrides + alignment helpers on top of the existing
	// draw_text. The (font, size) overload of ImDrawList::AddText is what
	// makes scaling work without a separate font asset; size 0 means
	// "use the current GetFontSize()".
	int l_cathack_draw_text_sized(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x = static_cast<float>(luaL_checknumber(L, 1));
		const float y = static_cast<float>(luaL_checknumber(L, 2));
		const char* s = luaL_checkstring(L, 3);
		const lua_Integer color = luaL_optinteger(L, 4, 0xFFFFFFFFLL);
		const float size = static_cast<float>(luaL_optnumber(L, 5, 0.0));
		ImFont* font = ImGui::GetFont();
		const float useSize = (size > 0.f) ? size : ImGui::GetFontSize();
		dl->AddText(font, useSize, ImVec2(x, y), static_cast<ImU32>(color), s);
		return 0;
	}

	int l_cathack_draw_text_centered(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x = static_cast<float>(luaL_checknumber(L, 1));
		const float y = static_cast<float>(luaL_checknumber(L, 2));
		const char* s = luaL_checkstring(L, 3);
		const lua_Integer color = luaL_optinteger(L, 4, 0xFFFFFFFFLL);
		const float size = static_cast<float>(luaL_optnumber(L, 5, 0.0));
		ImFont* font = ImGui::GetFont();
		const float useSize = (size > 0.f) ? size : ImGui::GetFontSize();
		const ImVec2 sz = font->CalcTextSizeA(useSize, FLT_MAX, 0.f, s);
		dl->AddText(font, useSize, ImVec2(x - sz.x * 0.5f, y - sz.y * 0.5f),
			static_cast<ImU32>(color), s);
		return 0;
	}

	int l_cathack_draw_text_right(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x = static_cast<float>(luaL_checknumber(L, 1));
		const float y = static_cast<float>(luaL_checknumber(L, 2));
		const char* s = luaL_checkstring(L, 3);
		const lua_Integer color = luaL_optinteger(L, 4, 0xFFFFFFFFLL);
		const float size = static_cast<float>(luaL_optnumber(L, 5, 0.0));
		ImFont* font = ImGui::GetFont();
		const float useSize = (size > 0.f) ? size : ImGui::GetFontSize();
		const ImVec2 sz = font->CalcTextSizeA(useSize, FLT_MAX, 0.f, s);
		dl->AddText(font, useSize, ImVec2(x - sz.x, y), static_cast<ImU32>(color), s);
		return 0;
	}

	int l_cathack_measure_text_sized(lua_State* L)
	{
		const char* s = luaL_checkstring(L, 1);
		const float size = static_cast<float>(luaL_optnumber(L, 2, 0.0));
		ImFont* font = ImGui::GetFont();
		const float useSize = (size > 0.f) ? size : ImGui::GetFontSize();
		const ImVec2 sz = font->CalcTextSizeA(useSize, FLT_MAX, 0.f, s);
		lua_pushnumber(L, sz.x);
		lua_pushnumber(L, sz.y);
		return 2;
	}

	// --- Fancy primitives --------------------------------------------------
	int l_cathack_draw_rect_gradient(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x = static_cast<float>(luaL_checknumber(L, 1));
		const float y = static_cast<float>(luaL_checknumber(L, 2));
		const float w = static_cast<float>(luaL_checknumber(L, 3));
		const float h = static_cast<float>(luaL_checknumber(L, 4));
		const lua_Integer c1 = luaL_checkinteger(L, 5);
		const lua_Integer c2 = luaL_checkinteger(L, 6);
		const bool vertical  = lua_isnoneornil(L, 7) ? true : (lua_toboolean(L, 7) != 0);
		const ImU32 ca = static_cast<ImU32>(c1);
		const ImU32 cb = static_cast<ImU32>(c2);
		if (vertical)
		{
			dl->AddRectFilledMultiColor(ImVec2(x, y), ImVec2(x + w, y + h),
				ca, ca, cb, cb);
		}
		else
		{
			dl->AddRectFilledMultiColor(ImVec2(x, y), ImVec2(x + w, y + h),
				ca, cb, cb, ca);
		}
		return 0;
	}

	int l_cathack_draw_shadow_rect(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float x = static_cast<float>(luaL_checknumber(L, 1));
		const float y = static_cast<float>(luaL_checknumber(L, 2));
		const float w = static_cast<float>(luaL_checknumber(L, 3));
		const float h = static_cast<float>(luaL_checknumber(L, 4));
		const lua_Integer color = luaL_optinteger(L, 5, 0xC0000000LL); // black-ish
		const float blur = static_cast<float>(luaL_optnumber(L, 6, 6.0));
		const float rounding = static_cast<float>(luaL_optnumber(L, 7, 0.0));
		// Approximate a soft shadow with N concentric rounded rects, each
		// with reduced alpha and slightly larger area. Cheap and looks
		// nice enough for menu backgrounds. blur=0 collapses to a flat
		// shadow.
		const int passes = blur <= 0.f ? 1 : (static_cast<int>(blur) > 8 ? 8 : static_cast<int>(blur));
		ImU32 baseCol = static_cast<ImU32>(color);
		const ImU32 baseAlpha = (baseCol >> 24) & 0xFF;
		for (int i = 0; i < passes; ++i)
		{
			const float pad = (passes <= 1) ? 0.f : (blur * (static_cast<float>(i) / (passes - 1)));
			const float a = static_cast<float>(baseAlpha) * (1.f - static_cast<float>(i) / static_cast<float>(passes));
			const ImU32 stepCol = (baseCol & 0x00FFFFFF) | (static_cast<ImU32>(a) << 24);
			dl->AddRectFilled(
				ImVec2(x - pad, y - pad),
				ImVec2(x + w + pad, y + h + pad),
				stepCol, rounding + pad);
		}
		return 0;
	}

	int l_cathack_draw_glow_circle(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const float cx = static_cast<float>(luaL_checknumber(L, 1));
		const float cy = static_cast<float>(luaL_checknumber(L, 2));
		const float r  = static_cast<float>(luaL_checknumber(L, 3));
		const lua_Integer color = luaL_optinteger(L, 4, 0xFFFFFFFFLL);
		const float strength    = static_cast<float>(luaL_optnumber(L, 5, 1.0));
		// Draw N nested circles outward with falling alpha to fake a glow.
		const ImU32 baseCol = static_cast<ImU32>(color);
		const ImU32 baseAlpha = (baseCol >> 24) & 0xFF;
		const int passes = 6;
		for (int i = passes; i >= 0; --i)
		{
			const float t = static_cast<float>(i) / passes;
			const float rr = r * (1.f + t * 0.9f * strength);
			const float a = static_cast<float>(baseAlpha) * (1.f - t) * (1.f - t);
			const ImU32 stepCol = (baseCol & 0x00FFFFFF) | (static_cast<ImU32>(a) << 24);
			dl->AddCircleFilled(ImVec2(cx, cy), rr, stepCol, 32);
		}
		return 0;
	}

	// --- Texture loading + image drawing -----------------------------------
	// Each loaded texture gets a small handle (an integer key) that's used
	// from Lua. Textures live as long as the script holds the handle OR
	// until the VM is restarted; we free everything in DestroyState.
	struct LuaTexture
	{
		ID3D11ShaderResourceView* srv = nullptr;
		int                       w   = 0;
		int                       h   = 0;
		std::string               owner;
	};
	std::unordered_map<lua_Integer, LuaTexture> g_textures;
	lua_Integer                                 g_nextTextureId = 1;

	void FreeAllTextures_locked()
	{
		for (auto& kv : g_textures)
			if (kv.second.srv) kv.second.srv->Release();
		g_textures.clear();
		g_nextTextureId = 1;
	}

	int l_cathack_load_texture(lua_State* L)
	{
		const char* nm = luaL_checkstring(L, 1);
		// load_texture is only safe to call from the render thread (the
		// underlying D3D11 calls touch the immediate context). We gate on
		// tl_currentDrawList — it's set only during TickRender — so a
		// script that calls load_texture from on_tick gets a clear nil
		// rather than a sporadic crash.
		if (!tl_currentDrawList) { lua_pushnil(L); return 1; }
		// Sandbox the path: only filenames inside the lua dir (or its
		// 'assets' subdir) are allowed. Same name-validation the existing
		// read_file / write_file uses, plus an optional 'assets/' prefix.
		const std::string n = nm;
		bool valid = !n.empty() && n.size() < 128;
		for (char c : n)
		{
			if (c == '\\' || (c == '.' && n.find("..") != std::string::npos))
			{ valid = false; break; }
		}
		if (!valid) { lua_pushnil(L); return 1; }
		std::filesystem::path full = GetLuaDirectory() / n;
		std::error_code ec;
		if (!std::filesystem::exists(full, ec)) { lua_pushnil(L); return 1; }
		ID3D11ShaderResourceView* srv = nullptr;
		int w = 0, h = 0;
		const std::wstring wpath = full.wstring();
		if (!Engine::LoadTextureFromFile(wpath.c_str(), &srv, &w, &h) || !srv)
		{
			lua_pushnil(L);
			return 1;
		}
		LuaTexture t;
		t.srv = srv;
		t.w = w;
		t.h = h;
		t.owner = g_runningScript;
		const lua_Integer id = g_nextTextureId++;
		g_textures.emplace(id, std::move(t));
		lua_pushinteger(L, id);
		return 1;
	}

	int l_cathack_free_texture(lua_State* L)
	{
		const lua_Integer id = luaL_checkinteger(L, 1);
		auto it = g_textures.find(id);
		if (it == g_textures.end()) { lua_pushboolean(L, 0); return 1; }
		if (it->second.srv) it->second.srv->Release();
		g_textures.erase(it);
		lua_pushboolean(L, 1);
		return 1;
	}

	int l_cathack_texture_size(lua_State* L)
	{
		const lua_Integer id = luaL_checkinteger(L, 1);
		auto it = g_textures.find(id);
		if (it == g_textures.end()) { lua_pushnil(L); return 1; }
		lua_pushinteger(L, it->second.w);
		lua_pushinteger(L, it->second.h);
		return 2;
	}

	int l_cathack_draw_image(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const lua_Integer id = luaL_checkinteger(L, 1);
		const float x = static_cast<float>(luaL_checknumber(L, 2));
		const float y = static_cast<float>(luaL_checknumber(L, 3));
		const float w = static_cast<float>(luaL_checknumber(L, 4));
		const float h = static_cast<float>(luaL_checknumber(L, 5));
		const lua_Integer color = luaL_optinteger(L, 6, 0xFFFFFFFFLL);
		auto it = g_textures.find(id);
		if (it == g_textures.end() || !it->second.srv) return 0;
		dl->AddImage(
			(ImTextureID)it->second.srv,
			ImVec2(x, y), ImVec2(x + w, y + h),
			ImVec2(0, 0), ImVec2(1, 1),
			static_cast<ImU32>(color));
		return 0;
	}

	int l_cathack_draw_image_rounded(lua_State* L)
	{
		ImDrawList* dl = tl_currentDrawList;
		if (!dl) return 0;
		const lua_Integer id = luaL_checkinteger(L, 1);
		const float x = static_cast<float>(luaL_checknumber(L, 2));
		const float y = static_cast<float>(luaL_checknumber(L, 3));
		const float w = static_cast<float>(luaL_checknumber(L, 4));
		const float h = static_cast<float>(luaL_checknumber(L, 5));
		const float rounding = static_cast<float>(luaL_optnumber(L, 6, 0.0));
		const lua_Integer color = luaL_optinteger(L, 7, 0xFFFFFFFFLL);
		auto it = g_textures.find(id);
		if (it == g_textures.end() || !it->second.srv) return 0;
		dl->AddImageRounded(
			(ImTextureID)it->second.srv,
			ImVec2(x, y), ImVec2(x + w, y + h),
			ImVec2(0, 0), ImVec2(1, 1),
			static_cast<ImU32>(color),
			rounding, ImDrawFlags_RoundCornersAll);
		return 0;
	}

	// --- Clipboard ----------------------------------------------------------
	int l_cathack_clipboard_get(lua_State* L)
	{
		if (!ImGui::GetCurrentContext()) { lua_pushstring(L, ""); return 1; }
		const char* s = ImGui::GetClipboardText();
		lua_pushstring(L, s ? s : "");
		return 1;
	}

	int l_cathack_clipboard_set(lua_State* L)
	{
		const char* s = luaL_checkstring(L, 1);
		if (ImGui::GetCurrentContext())
			ImGui::SetClipboardText(s);
		return 0;
	}

	// --- Theme / accent helpers --------------------------------------------
	// Read-only access to MiscSettings's menu palette so Lua UIs can match
	// the cheat's accent. All values come back as packed ImU32 so they
	// drop straight into draw_* calls.
	ImU32 PackedFromVec4(const ImVec4& c)
	{
		auto cl = [](float f) { int v = static_cast<int>(f * 255.f + 0.5f); return v < 0 ? 0 : (v > 255 ? 255 : v); };
		return (static_cast<ImU32>(cl(c.w)) << 24)
		     | (static_cast<ImU32>(cl(c.z)) << 16)
		     | (static_cast<ImU32>(cl(c.y)) << 8)
		     |  static_cast<ImU32>(cl(c.x));
	}

	int l_cathack_accent_color(lua_State* L)
	{
		lua_pushinteger(L, static_cast<lua_Integer>(PackedFromVec4(MiscSettings.MenuAccent)));
		return 1;
	}

	int l_cathack_theme(lua_State* L)
	{
		lua_newtable(L);
		auto setKey = [&](const char* k, lua_Integer v) {
			lua_pushinteger(L, v);
			lua_setfield(L, -2, k);
		};
		setKey("accent",     PackedFromVec4(MiscSettings.MenuAccent));
		setKey("background", PackedFromVec4(MiscSettings.MenuBackgroundColor));
		setKey("text",       PackedFromVec4(MiscSettings.MenuTextColor));
		setKey("border",     PackedFromVec4(MiscSettings.MenuBorderColor));
		// Opacity multiplier (0..1) the rest of the menu uses.
		lua_pushnumber(L, MiscSettings.MenuOpacity);
		lua_setfield(L, -2, "opacity");
		return 1;
	}

	// --- Animation / color helpers -----------------------------------------
	int l_cathack_lerp_color(lua_State* L)
	{
		const lua_Integer ca = luaL_checkinteger(L, 1);
		const lua_Integer cb = luaL_checkinteger(L, 2);
		const float t = static_cast<float>(luaL_checknumber(L, 3));
		auto chan = [](lua_Integer c, int shift) { return static_cast<float>((c >> shift) & 0xFF); };
		const float ar = chan(ca, 0),  ag = chan(ca, 8),  ab = chan(ca, 16), aa = chan(ca, 24);
		const float br = chan(cb, 0),  bg = chan(cb, 8),  bb = chan(cb, 16), ba = chan(cb, 24);
		auto mix = [](float x, float y, float k) { return x + (y - x) * k; };
		const int r  = static_cast<int>(mix(ar, br, t) + 0.5f);
		const int g  = static_cast<int>(mix(ag, bg, t) + 0.5f);
		const int b2 = static_cast<int>(mix(ab, bb, t) + 0.5f);
		const int a  = static_cast<int>(mix(aa, ba, t) + 0.5f);
		auto cl = [](int v) { return v < 0 ? 0 : (v > 255 ? 255 : v); };
		const lua_Integer packed =
			(static_cast<lua_Integer>(cl(a)) << 24) |
			(static_cast<lua_Integer>(cl(b2)) << 16) |
			(static_cast<lua_Integer>(cl(g)) << 8)  |
			 static_cast<lua_Integer>(cl(r));
		lua_pushinteger(L, packed);
		return 1;
	}

	int l_cathack_ease(lua_State* L)
	{
		// ease(type, t) -> number. t is clamped to [0, 1] for friendliness.
		// Types: linear, in_quad, out_quad, in_out_quad, in_cubic, out_cubic,
		// in_out_cubic, in_sine, out_sine, in_out_sine, in_back, out_back,
		// in_out_back, in_bounce, out_bounce.
		const char* type = luaL_optstring(L, 1, "linear");
		float t = static_cast<float>(luaL_checknumber(L, 2));
		if (t < 0.f) t = 0.f;
		if (t > 1.f) t = 1.f;
		float r = t;
		auto outBounce = [](float x) {
			const float n1 = 7.5625f, d1 = 2.75f;
			if (x < 1.f / d1)        return n1 * x * x;
			if (x < 2.f / d1)        { x -= 1.5f / d1; return n1 * x * x + 0.75f; }
			if (x < 2.5f / d1)       { x -= 2.25f / d1; return n1 * x * x + 0.9375f; }
			x -= 2.625f / d1; return n1 * x * x + 0.984375f;
		};
		const float pi = 3.14159265358979323846f;
		if      (std::strcmp(type, "linear") == 0)        r = t;
		else if (std::strcmp(type, "in_quad") == 0)       r = t * t;
		else if (std::strcmp(type, "out_quad") == 0)      r = 1.f - (1.f - t) * (1.f - t);
		else if (std::strcmp(type, "in_out_quad") == 0)   r = (t < 0.5f) ? 2.f * t * t : 1.f - std::pow(-2.f * t + 2.f, 2.f) * 0.5f;
		else if (std::strcmp(type, "in_cubic") == 0)      r = t * t * t;
		else if (std::strcmp(type, "out_cubic") == 0)     r = 1.f - std::pow(1.f - t, 3.f);
		else if (std::strcmp(type, "in_out_cubic") == 0)  r = (t < 0.5f) ? 4.f * t * t * t : 1.f - std::pow(-2.f * t + 2.f, 3.f) * 0.5f;
		else if (std::strcmp(type, "in_sine") == 0)       r = 1.f - std::cos((t * pi) * 0.5f);
		else if (std::strcmp(type, "out_sine") == 0)      r = std::sin((t * pi) * 0.5f);
		else if (std::strcmp(type, "in_out_sine") == 0)   r = -(std::cos(pi * t) - 1.f) * 0.5f;
		else if (std::strcmp(type, "in_back") == 0)
		{
			const float c1 = 1.70158f, c3 = c1 + 1.f;
			r = c3 * t * t * t - c1 * t * t;
		}
		else if (std::strcmp(type, "out_back") == 0)
		{
			const float c1 = 1.70158f, c3 = c1 + 1.f;
			r = 1.f + c3 * std::pow(t - 1.f, 3.f) + c1 * std::pow(t - 1.f, 2.f);
		}
		else if (std::strcmp(type, "in_out_back") == 0)
		{
			const float c1 = 1.70158f, c2 = c1 * 1.525f;
			r = (t < 0.5f)
				? (std::pow(2.f * t, 2.f) * ((c2 + 1.f) * 2.f * t - c2)) * 0.5f
				: (std::pow(2.f * t - 2.f, 2.f) * ((c2 + 1.f) * (t * 2.f - 2.f) + c2) + 2.f) * 0.5f;
		}
		else if (std::strcmp(type, "out_bounce") == 0) r = outBounce(t);
		else if (std::strcmp(type, "in_bounce") == 0)  r = 1.f - outBounce(1.f - t);
		else r = t;
		lua_pushnumber(L, r);
		return 1;
	}

	// --- Viewport helpers --------------------------------------------------
	int l_cathack_viewport_pos(lua_State* L)
	{
		// In a normal fullscreen game, the viewport sits at (0, 0) and
		// matches DisplaySize. Multi-viewport setups (rare here) report
		// the main viewport's actual position through ImGui.
		ImGuiViewport* vp = ImGui::GetMainViewport();
		if (vp)
		{
			lua_pushnumber(L, vp->Pos.x);
			lua_pushnumber(L, vp->Pos.y);
		}
		else
		{
			lua_pushnumber(L, 0.0); lua_pushnumber(L, 0.0);
		}
		return 2;
	}

	int l_cathack_viewport_size(lua_State* L)
	{
		ImGuiViewport* vp = ImGui::GetMainViewport();
		if (vp)
		{
			lua_pushnumber(L, vp->Size.x);
			lua_pushnumber(L, vp->Size.y);
		}
		else
		{
			const ImGuiIO& io = ImGui::GetIO();
			lua_pushnumber(L, io.DisplaySize.x);
			lua_pushnumber(L, io.DisplaySize.y);
		}
		return 2;
	}

	int l_cathack_dpi_scale(lua_State* L)
	{
		// ImGuiViewport::DpiScale is only available on newer ImGui
		// builds with the multi-viewport system enabled. Our copy
		// doesn't expose that yet, so fall back to io.DisplayFramebufferScale
		// (set by the impl backend when the host window's DPI
		// changes). Defaults to 1 when nothing's set.
		float s = 1.f;
		if (ImGui::GetCurrentContext())
		{
			const ImGuiIO& io = ImGui::GetIO();
			if (io.DisplayFramebufferScale.x > 0.f) s = io.DisplayFramebufferScale.x;
		}
		lua_pushnumber(L, s);
		return 1;
	}

	// --- Tiny JSON encode / decode ------------------------------------------
	// Hand-rolled so we don't pull in a JSON library. Tables that look like
	// arrays (1-indexed, contiguous integer keys) become JSON arrays;
	// other tables become JSON objects. Cycles are not detected; a script
	// that hands us a self-referential table will recurse until the watchdog
	// aborts it (250ms budget) — acceptable for a UI-config use case.
	void JsonEscape(std::string& out, const char* s, size_t n)
	{
		out.push_back('"');
		for (size_t i = 0; i < n; ++i)
		{
			unsigned char c = static_cast<unsigned char>(s[i]);
			switch (c)
			{
			case '"':  out += "\\\""; break;
			case '\\': out += "\\\\"; break;
			case '\b': out += "\\b";  break;
			case '\f': out += "\\f";  break;
			case '\n': out += "\\n";  break;
			case '\r': out += "\\r";  break;
			case '\t': out += "\\t";  break;
			default:
				if (c < 0x20)
				{
					char buf[8];
					snprintf(buf, sizeof(buf), "\\u%04x", c);
					out += buf;
				}
				else
				{
					out.push_back(static_cast<char>(c));
				}
				break;
			}
		}
		out.push_back('"');
	}

	bool JsonEncodeValue(lua_State* L, int idx, std::string& out, int depth);

	bool JsonEncodeTable(lua_State* L, int idx, std::string& out, int depth)
	{
		if (depth > 32) return false;
		// Detect array shape: lua_rawlen reflects the table's # operator,
		// which is the array length for sequential integer keys 1..N.
		const int absIdx = lua_absindex(L, idx);
		const lua_Integer arrLen = static_cast<lua_Integer>(lua_rawlen(L, absIdx));
		bool isArray = arrLen > 0;
		if (isArray)
		{
			// Verify keys 1..arrLen are present and there are no other keys.
			for (lua_Integer i = 1; i <= arrLen; ++i)
			{
				lua_rawgeti(L, absIdx, i);
				const bool exists = !lua_isnil(L, -1);
				lua_pop(L, 1);
				if (!exists) { isArray = false; break; }
			}
		}
		if (isArray)
		{
			out.push_back('[');
			for (lua_Integer i = 1; i <= arrLen; ++i)
			{
				if (i > 1) out.push_back(',');
				lua_rawgeti(L, absIdx, i);
				if (!JsonEncodeValue(L, -1, out, depth + 1)) { lua_pop(L, 1); return false; }
				lua_pop(L, 1);
			}
			out.push_back(']');
		}
		else
		{
			out.push_back('{');
			lua_pushnil(L);
			bool first = true;
			while (lua_next(L, absIdx) != 0)
			{
				// stack: ... key value
				if (!first) out.push_back(',');
				first = false;
				// Key must be string or number; coerce numbers to strings.
				if (lua_type(L, -2) == LUA_TSTRING)
				{
					size_t kl = 0;
					const char* k = lua_tolstring(L, -2, &kl);
					JsonEscape(out, k, kl);
				}
				else if (lua_type(L, -2) == LUA_TNUMBER)
				{
					char buf[64];
					if (lua_isinteger(L, -2))
						snprintf(buf, sizeof(buf), "%lld", (long long)lua_tointeger(L, -2));
					else
						snprintf(buf, sizeof(buf), "%.17g", lua_tonumber(L, -2));
					JsonEscape(out, buf, std::strlen(buf));
				}
				else
				{
					lua_pop(L, 2); return false;
				}
				out.push_back(':');
				if (!JsonEncodeValue(L, -1, out, depth + 1)) { lua_pop(L, 2); return false; }
				lua_pop(L, 1); // pop value, keep key for lua_next
			}
			out.push_back('}');
		}
		return true;
	}

	bool JsonEncodeValue(lua_State* L, int idx, std::string& out, int depth)
	{
		const int t = lua_type(L, idx);
		switch (t)
		{
		case LUA_TNIL:
			out += "null";
			return true;
		case LUA_TBOOLEAN:
			out += lua_toboolean(L, idx) ? "true" : "false";
			return true;
		case LUA_TNUMBER:
		{
			char buf[64];
			if (lua_isinteger(L, idx))
				snprintf(buf, sizeof(buf), "%lld", (long long)lua_tointeger(L, idx));
			else
				snprintf(buf, sizeof(buf), "%.17g", lua_tonumber(L, idx));
			out += buf;
			return true;
		}
		case LUA_TSTRING:
		{
			size_t n = 0;
			const char* s = lua_tolstring(L, idx, &n);
			JsonEscape(out, s, n);
			return true;
		}
		case LUA_TTABLE:
			return JsonEncodeTable(L, idx, out, depth);
		default:
			return false;
		}
	}

	int l_cathack_json_encode(lua_State* L)
	{
		std::string out;
		if (!JsonEncodeValue(L, 1, out, 0))
			return luaL_error(L, "json_encode: unsupported value");
		lua_pushlstring(L, out.data(), out.size());
		return 1;
	}

	// Decoder. Recursive descent over the input string. Pushes one Lua
	// value on success; raises a Lua error on parse failure.
	struct JsonDecoder
	{
		const char* s;
		const char* end;

		void skipWs()
		{
			while (s < end && (*s == ' ' || *s == '\t' || *s == '\n' || *s == '\r')) ++s;
		}

		bool match(char c) { if (s < end && *s == c) { ++s; return true; } return false; }

		bool parseValue(lua_State* L, int depth);

		bool parseString(std::string& out)
		{
			if (!match('"')) return false;
			out.clear();
			while (s < end && *s != '"')
			{
				if (*s == '\\')
				{
					if (++s >= end) return false;
					char c = *s++;
					switch (c)
					{
					case '"':  out.push_back('"');  break;
					case '\\': out.push_back('\\'); break;
					case '/':  out.push_back('/');  break;
					case 'b':  out.push_back('\b'); break;
					case 'f':  out.push_back('\f'); break;
					case 'n':  out.push_back('\n'); break;
					case 'r':  out.push_back('\r'); break;
					case 't':  out.push_back('\t'); break;
					case 'u':
					{
						if (end - s < 4) return false;
						unsigned int cp = 0;
						for (int i = 0; i < 4; ++i)
						{
							const char hc = s[i];
							int v = 0;
							if (hc >= '0' && hc <= '9')      v = hc - '0';
							else if (hc >= 'a' && hc <= 'f') v = 10 + (hc - 'a');
							else if (hc >= 'A' && hc <= 'F') v = 10 + (hc - 'A');
							else return false;
							cp = (cp << 4) | static_cast<unsigned int>(v);
						}
						s += 4;
						out += Utf32ToUtf8(cp);
						break;
					}
					default: return false;
					}
				}
				else
				{
					out.push_back(*s++);
				}
			}
			return match('"');
		}
	};

	bool JsonDecoder::parseValue(lua_State* L, int depth)
	{
		if (depth > 64) return false;
		skipWs();
		if (s >= end) return false;
		char c = *s;
		if (c == '"')
		{
			std::string str;
			if (!parseString(str)) return false;
			lua_pushlstring(L, str.data(), str.size());
			return true;
		}
		if (c == '{')
		{
			++s;
			lua_newtable(L);
			skipWs();
			if (match('}')) return true;
			while (s < end)
			{
				skipWs();
				std::string key;
				if (!parseString(key)) return false;
				skipWs();
				if (!match(':')) return false;
				if (!parseValue(L, depth + 1)) return false;
				lua_setfield(L, -2, key.c_str());
				skipWs();
				if (match(',')) continue;
				return match('}');
			}
			return false;
		}
		if (c == '[')
		{
			++s;
			lua_newtable(L);
			skipWs();
			if (match(']')) return true;
			lua_Integer idx = 1;
			while (s < end)
			{
				if (!parseValue(L, depth + 1)) return false;
				lua_rawseti(L, -2, idx++);
				skipWs();
				if (match(',')) continue;
				return match(']');
			}
			return false;
		}
		if (c == 't')
		{
			if (end - s >= 4 && std::strncmp(s, "true", 4) == 0)
			{ s += 4; lua_pushboolean(L, 1); return true; }
			return false;
		}
		if (c == 'f')
		{
			if (end - s >= 5 && std::strncmp(s, "false", 5) == 0)
			{ s += 5; lua_pushboolean(L, 0); return true; }
			return false;
		}
		if (c == 'n')
		{
			if (end - s >= 4 && std::strncmp(s, "null", 4) == 0)
			{ s += 4; lua_pushnil(L); return true; }
			return false;
		}
		// Number. Take a permissive slice — strtod handles the spec.
		const char* p = s;
		if (*p == '-') ++p;
		while (p < end && (std::isdigit(static_cast<unsigned char>(*p))
			|| *p == '.' || *p == 'e' || *p == 'E' || *p == '+' || *p == '-')) ++p;
		if (p == s) return false;
		std::string buf(s, p);
		s = p;
		char* endp = nullptr;
		double d = std::strtod(buf.c_str(), &endp);
		if (!endp || *endp != '\0') return false;
		// Push integer when the number is exactly representable as one.
		if (std::floor(d) == d && d >= static_cast<double>(LLONG_MIN) && d <= static_cast<double>(LLONG_MAX))
			lua_pushinteger(L, static_cast<lua_Integer>(d));
		else
			lua_pushnumber(L, d);
		return true;
	}

	int l_cathack_json_decode(lua_State* L)
	{
		size_t n = 0;
		const char* s = luaL_checklstring(L, 1, &n);
		JsonDecoder dec{ s, s + n };
		if (!dec.parseValue(L, 0))
			return luaL_error(L, "json_decode: parse error");
		return 1;
	}

	// --- Per-frame event callbacks ------------------------------------------
	// Six new fans in the same shape as the existing tick / render fans:
	//   on_key_pressed(function(vk) end)
	//   on_key_released(function(vk) end)
	//   on_mouse_pressed(function(button, x, y) end)
	//   on_mouse_released(function(button, x, y) end)
	//   on_mouse_wheel(function(delta_v, delta_h) end)
	//   on_char(function(string) end)
	// Same handle pattern as on_tick / on_render: returns an integer that
	// can be passed to unhook() to drop the registration. The host's
	// Stop() / Restart() / clear_callbacks tear them down too. Storage
	// itself lives at the top of this anonymous namespace so the cleanup
	// helpers above can see it.

	int RegisterEventCallback(lua_State* L, std::vector<Callback>& sink)
	{
		luaL_checktype(L, 1, LUA_TFUNCTION);
		lua_pushvalue(L, 1);
		const int ref = luaL_ref(L, LUA_REGISTRYINDEX);
		sink.push_back({ ref, g_runningScript });
		lua_pushinteger(L, ref);
		return 1;
	}

	int l_cathack_on_key_pressed(lua_State* L)   { return RegisterEventCallback(L, g_keyPressedRefs); }
	int l_cathack_on_key_released(lua_State* L)  { return RegisterEventCallback(L, g_keyReleasedRefs); }
	int l_cathack_on_mouse_pressed(lua_State* L) { return RegisterEventCallback(L, g_mousePressedRefs); }
	int l_cathack_on_mouse_released(lua_State* L){ return RegisterEventCallback(L, g_mouseReleasedRefs); }
	int l_cathack_on_mouse_wheel(lua_State* L)   { return RegisterEventCallback(L, g_mouseWheelRefs); }
	int l_cathack_on_char(lua_State* L)          { return RegisterEventCallback(L, g_charRefs); }

	// =====================================================================
	// End of UI block
	// =====================================================================

	// =====================================================================
	// Object reflection
	//
	// A small reflection layer exposing UClass/FProperty/UFunction metadata
	// to Lua. Object handles are raw pointers cast to lua_Integer, the
	// same model used by the entity API — every dereference goes through
	// IsLikelyLiveUObject() so a stale handle returns nil instead of
	// crashing.
	//
	// Read-side (object_get / object_properties / object_functions) is
	// permissive: it understands all the FProperty types the SDK exposes
	// for primitive values, FName, FString, FVector, FRotator, FVector2D,
	// and tagged enums.
	//
	// Write-side (object_set / object_call) is whitelist-based:
	//   - object_set only writes to FProperty types whose binary layout is
	//     fixed-size and well-defined (bool/int/float/double/byte/FName/
	//     FString/FVector/FRotator/FVector2D). Pointer types are refused.
	//   - object_call only invokes UFunctions whose every parameter is in
	//     the same whitelist. Functions taking object pointers, structs
	//     other than the listed vectors, arrays, maps, sets, or text are
	//     refused with a descriptive error.
	//
	// PlayerController convenience helpers (pc_*) sit on top of this and
	// are pre-resolved against GVars.PlayerController so scripts don't
	// have to fish for an object handle for the most common case.
	// =====================================================================

	// Common UE5 PropertyFlags bits we care about. Defining as constexpr
	// here so we don't depend on the SDK's enum naming (which differs
	// between Dumper7 versions).
	static constexpr uint64_t kCPF_Edit              = 0x0000000000000001ull;
	static constexpr uint64_t kCPF_ConstParm         = 0x0000000000000002ull;
	static constexpr uint64_t kCPF_BlueprintVisible  = 0x0000000000000004ull;
	static constexpr uint64_t kCPF_BlueprintReadOnly = 0x0000000000000010ull;
	static constexpr uint64_t kCPF_Parm              = 0x0000000000000080ull;
	static constexpr uint64_t kCPF_OutParm           = 0x0000000000000100ull;
	static constexpr uint64_t kCPF_ReturnParm        = 0x0000000000000400ull;

	// Resolve an integer Lua argument back into a (validated) UObject*.
	// Returns nullptr on any failure — the call site should push nil.
	static UObject* ResolveObject(lua_Integer id)
	{
		if (id == 0) return nullptr;
		UObject* obj = reinterpret_cast<UObject*>(static_cast<uintptr_t>(id));
		if (!Utils::IsLikelyLiveUObject(obj)) return nullptr;
		return obj;
	}

	// Push an object handle (or nil if invalid).
	static void PushObjectId(lua_State* L, UObject* obj)
	{
		if (!obj || !Utils::IsLikelyLiveUObject(obj)) { lua_pushnil(L); return; }
		lua_pushinteger(L, static_cast<lua_Integer>(reinterpret_cast<uintptr_t>(obj)));
	}

	// Read the FFieldClass name (e.g. "ObjectProperty", "BoolProperty") for a FField.
	static std::string FieldTypeName(const FField* f)
	{
		if (!f || !f->ClassPrivate) return {};
		return f->ClassPrivate->Name.ToString();
	}

	// Look up a property by name on `obj`'s class, walking SuperStruct
	// ancestors. Returns the FProperty* or nullptr if no match.
	static const FProperty* FindPropertyByName(UObject* obj, const char* name)
	{
		if (!obj || !name) return nullptr;
		UClass* cls = obj->Class;
		while (cls)
		{
			FField* f = cls->ChildProperties;
			while (f)
			{
				if (f->Name.ToString() == name)
					return reinterpret_cast<const FProperty*>(f);
				f = f->Next;
			}
			UStruct* super_ = cls->SuperStruct;
			cls = (super_ && super_ != cls) ? reinterpret_cast<UClass*>(super_) : nullptr;
		}
		return nullptr;
	}

	// Look up a function by name on `obj`'s class, walking SuperStruct
	// ancestors. Functions live on the (UField-based) Children list, not
	// ChildProperties.
	static UFunction* FindFunctionByName(UObject* obj, const char* name)
	{
		if (!obj || !name) return nullptr;
		UClass* cls = obj->Class;
		while (cls)
		{
			UField* fld = cls->Children;
			while (fld)
			{
				if (fld->IsA(UFunction::StaticClass()))
				{
					UFunction* fn = reinterpret_cast<UFunction*>(fld);
					if (fn->Name.ToString() == name)
						return fn;
				}
				fld = fld->Next;
			}
			UStruct* super_ = cls->SuperStruct;
			cls = (super_ && super_ != cls) ? reinterpret_cast<UClass*>(super_) : nullptr;
		}
		return nullptr;
	}

	// Map an FProperty's type name to one of our small set of "supported"
	// kinds. Returns "" for unsupported / opaque types so the caller can
	// surface them as readable=false / callable=false.
	static const char* CategorizeProperty(const std::string& tname)
	{
		if (tname == "BoolProperty")   return "bool";
		if (tname == "ByteProperty")   return "byte";
		if (tname == "Int8Property")   return "int8";
		if (tname == "Int16Property")  return "int16";
		if (tname == "IntProperty")    return "int";
		if (tname == "Int64Property")  return "int64";
		if (tname == "UInt16Property") return "uint16";
		if (tname == "UInt32Property") return "uint32";
		if (tname == "UInt64Property") return "uint64";
		if (tname == "FloatProperty")  return "float";
		if (tname == "DoubleProperty") return "double";
		if (tname == "NameProperty")   return "name";
		if (tname == "StrProperty")    return "string";
		if (tname == "TextProperty")   return "text";
		if (tname == "ObjectProperty") return "object";
		if (tname == "ClassProperty")  return "class";
		if (tname == "EnumProperty")   return "enum";
		if (tname == "StructProperty") return "struct";
		if (tname == "ArrayProperty")  return "array";
		if (tname == "MapProperty")    return "map";
		if (tname == "SetProperty")    return "set";
		return "";
	}

	// Decide whether a property is writable from Lua. Pointer/object/
	// container types are read-only; primitives + the small set of
	// well-known structs (FVector / FRotator / FVector2D) are writable.
	// `structHint` may be empty for non-struct properties.
	static bool PropertyIsLuaWritable(const std::string& tname, const std::string& structHint)
	{
		if (tname == "BoolProperty"
			|| tname == "ByteProperty"
			|| tname == "Int8Property"
			|| tname == "Int16Property"
			|| tname == "IntProperty"
			|| tname == "Int64Property"
			|| tname == "UInt16Property"
			|| tname == "UInt32Property"
			|| tname == "UInt64Property"
			|| tname == "FloatProperty"
			|| tname == "DoubleProperty"
			|| tname == "NameProperty"
			|| tname == "StrProperty"
			|| tname == "EnumProperty")
			return true;
		if (tname == "StructProperty")
			return structHint == "Vector" || structHint == "Rotator" || structHint == "Vector2D";
		return false;
	}

	// Read a property value from `obj` and push it on the Lua stack.
	// Returns true on success, false (and pushes nil) on failure.
	static bool PropertyGetAndPush(lua_State* L, UObject* obj, const FProperty* prop, const std::string& tname)
	{
		if (!obj || !prop) { lua_pushnil(L); return false; }
		uint8_t* base = reinterpret_cast<uint8_t*>(obj) + prop->Offset;

		if (tname == "BoolProperty")
		{
			// Bools store their bit mask in the first byte after the
			// FProperty header (Pad_48 is overloaded with FieldSize +
			// ByteOffset + ByteMask + FieldMask). The property layout
			// matches UE5: ByteOffset @ 0x49, ByteMask @ 0x4A, FieldMask
			// @ 0x4B from the FField base + FProperty offset 0x30.
			const uint8_t* meta = reinterpret_cast<const uint8_t*>(prop) + 0x48;
			const uint8_t  byteOffset = meta[1];
			const uint8_t  fieldMask  = meta[3];
			const uint8_t* byteAddr = reinterpret_cast<uint8_t*>(obj) + prop->Offset + byteOffset;
			lua_pushboolean(L, ((*byteAddr) & fieldMask) ? 1 : 0);
			return true;
		}
		if (tname == "ByteProperty")    { lua_pushinteger(L, static_cast<lua_Integer>(*reinterpret_cast<const uint8_t*>(base))); return true; }
		if (tname == "Int8Property")    { lua_pushinteger(L, static_cast<lua_Integer>(*reinterpret_cast<const int8_t*>(base))); return true; }
		if (tname == "Int16Property")   { lua_pushinteger(L, static_cast<lua_Integer>(*reinterpret_cast<const int16_t*>(base))); return true; }
		if (tname == "IntProperty")     { lua_pushinteger(L, static_cast<lua_Integer>(*reinterpret_cast<const int32_t*>(base))); return true; }
		if (tname == "Int64Property")   { lua_pushinteger(L, static_cast<lua_Integer>(*reinterpret_cast<const int64_t*>(base))); return true; }
		if (tname == "UInt16Property")  { lua_pushinteger(L, static_cast<lua_Integer>(*reinterpret_cast<const uint16_t*>(base))); return true; }
		if (tname == "UInt32Property")  { lua_pushinteger(L, static_cast<lua_Integer>(*reinterpret_cast<const uint32_t*>(base))); return true; }
		if (tname == "UInt64Property")  { lua_pushinteger(L, static_cast<lua_Integer>(*reinterpret_cast<const uint64_t*>(base))); return true; }
		if (tname == "FloatProperty")   { lua_pushnumber(L,  *reinterpret_cast<const float*>(base)); return true; }
		if (tname == "DoubleProperty")  { lua_pushnumber(L,  *reinterpret_cast<const double*>(base)); return true; }
		if (tname == "EnumProperty")    { lua_pushinteger(L, static_cast<lua_Integer>(*reinterpret_cast<const int64_t*>(base))); return true; }

		if (tname == "NameProperty")
		{
			FName* n = reinterpret_cast<FName*>(base);
			const std::string s = n->ToString();
			lua_pushstring(L, s.c_str());
			return true;
		}
		if (tname == "StrProperty")
		{
			FString* fs = reinterpret_cast<FString*>(base);
			const std::string s = fs->ToString();
			lua_pushstring(L, s.c_str());
			return true;
		}
		if (tname == "ObjectProperty" || tname == "ClassProperty")
		{
			UObject* ref = *reinterpret_cast<UObject**>(base);
			if (!ref || !Utils::IsLikelyLiveUObject(ref)) { lua_pushnil(L); return true; }
			lua_pushinteger(L, static_cast<lua_Integer>(reinterpret_cast<uintptr_t>(ref)));
			return true;
		}
		if (tname == "StructProperty")
		{
			// We can only round-trip the small set of well-known structs.
			// The struct *type* lives on FStructProperty::Struct (offset
			// 0x70 in the SDK FProperty layout). Read it and dispatch.
			const UStruct* s = *reinterpret_cast<const UStruct* const*>(reinterpret_cast<const uint8_t*>(prop) + 0x70);
			std::string sn;
			if (s) sn = const_cast<UStruct*>(s)->GetName();
			if (sn == "Vector")
			{
				const FVector* v = reinterpret_cast<const FVector*>(base);
				lua_createtable(L, 0, 3);
				lua_pushnumber(L, v->X); lua_setfield(L, -2, "x");
				lua_pushnumber(L, v->Y); lua_setfield(L, -2, "y");
				lua_pushnumber(L, v->Z); lua_setfield(L, -2, "z");
				return true;
			}
			if (sn == "Rotator")
			{
				const FRotator* r = reinterpret_cast<const FRotator*>(base);
				lua_createtable(L, 0, 3);
				lua_pushnumber(L, r->Pitch); lua_setfield(L, -2, "pitch");
				lua_pushnumber(L, r->Yaw);   lua_setfield(L, -2, "yaw");
				lua_pushnumber(L, r->Roll);  lua_setfield(L, -2, "roll");
				return true;
			}
			if (sn == "Vector2D")
			{
				const FVector2D* v = reinterpret_cast<const FVector2D*>(base);
				lua_createtable(L, 0, 2);
				lua_pushnumber(L, v->X); lua_setfield(L, -2, "x");
				lua_pushnumber(L, v->Y); lua_setfield(L, -2, "y");
				return true;
			}
		}

		// Anything else: give the caller a graceful nil rather than
		// crashing on a memcpy of an unknown layout.
		lua_pushnil(L);
		return false;
	}

	// Write a Lua value into a property. Returns true on success or false
	// (and leaves the property unchanged) if the conversion is rejected.
	static bool PropertySetFromLua(lua_State* L, int valueIdx, UObject* obj, const FProperty* prop, const std::string& tname, std::string* err)
	{
		if (!obj || !prop) { if (err) *err = "invalid object/property"; return false; }
		uint8_t* base = reinterpret_cast<uint8_t*>(obj) + prop->Offset;

		auto writeBoolBit = [&](bool v) {
			const uint8_t* meta = reinterpret_cast<const uint8_t*>(prop) + 0x48;
			const uint8_t  byteOffset = meta[1];
			const uint8_t  fieldMask  = meta[3];
			uint8_t* byteAddr = reinterpret_cast<uint8_t*>(obj) + prop->Offset + byteOffset;
			if (v) *byteAddr |= fieldMask;
			else   *byteAddr &= static_cast<uint8_t>(~fieldMask);
		};

		if (tname == "BoolProperty")
		{
			writeBoolBit(lua_toboolean(L, valueIdx) != 0);
			return true;
		}
		auto asInt = [&]() -> lua_Integer {
			if (lua_type(L, valueIdx) == LUA_TBOOLEAN) return lua_toboolean(L, valueIdx) ? 1 : 0;
			return luaL_optinteger(L, valueIdx, 0);
		};
		auto asNum = [&]() -> double { return luaL_optnumber(L, valueIdx, 0.0); };

		if (tname == "ByteProperty")    { *reinterpret_cast<uint8_t*>(base)  = static_cast<uint8_t>(asInt());  return true; }
		if (tname == "Int8Property")    { *reinterpret_cast<int8_t*>(base)   = static_cast<int8_t>(asInt());   return true; }
		if (tname == "Int16Property")   { *reinterpret_cast<int16_t*>(base)  = static_cast<int16_t>(asInt());  return true; }
		if (tname == "IntProperty")     { *reinterpret_cast<int32_t*>(base)  = static_cast<int32_t>(asInt());  return true; }
		if (tname == "Int64Property")   { *reinterpret_cast<int64_t*>(base)  = static_cast<int64_t>(asInt());  return true; }
		if (tname == "UInt16Property")  { *reinterpret_cast<uint16_t*>(base) = static_cast<uint16_t>(asInt()); return true; }
		if (tname == "UInt32Property")  { *reinterpret_cast<uint32_t*>(base) = static_cast<uint32_t>(asInt()); return true; }
		if (tname == "UInt64Property")  { *reinterpret_cast<uint64_t*>(base) = static_cast<uint64_t>(asInt()); return true; }
		if (tname == "FloatProperty")   { *reinterpret_cast<float*>(base)    = static_cast<float>(asNum());    return true; }
		if (tname == "DoubleProperty")  { *reinterpret_cast<double*>(base)   = asNum();                         return true; }
		if (tname == "EnumProperty")    { *reinterpret_cast<int64_t*>(base)  = static_cast<int64_t>(asInt());  return true; }

		if (tname == "StrProperty")
		{
			const char* s = luaL_optstring(L, valueIdx, "");
			FString* fs = reinterpret_cast<FString*>(base);
			// Round-trip via wstring — FString is wide.
			std::wstring w;
			while (*s) { w.push_back(static_cast<wchar_t>(static_cast<unsigned char>(*s++))); }
			*fs = FString(w.c_str());
			return true;
		}
		// FName writes are not safe in general (the comparison-index pool
		// is engine-private). Skip — read works fine.
		if (tname == "NameProperty") { if (err) *err = "FName writes are read-only from Lua"; return false; }

		if (tname == "StructProperty")
		{
			const UStruct* s = *reinterpret_cast<const UStruct* const*>(reinterpret_cast<const uint8_t*>(prop) + 0x70);
			std::string sn = s ? const_cast<UStruct*>(s)->GetName() : std::string();
			if (lua_type(L, valueIdx) != LUA_TTABLE) { if (err) *err = "expected table for struct"; return false; }
			auto getNum = [&](const char* k) -> double {
				lua_getfield(L, valueIdx, k);
				const double v = (lua_type(L, -1) == LUA_TNUMBER) ? lua_tonumber(L, -1) : 0.0;
				lua_pop(L, 1);
				return v;
			};
			if (sn == "Vector")   { auto* v = reinterpret_cast<FVector*>(base);   v->X = getNum("x"); v->Y = getNum("y"); v->Z = getNum("z"); return true; }
			if (sn == "Rotator")  { auto* v = reinterpret_cast<FRotator*>(base);  v->Pitch = getNum("pitch"); v->Yaw = getNum("yaw"); v->Roll = getNum("roll"); return true; }
			if (sn == "Vector2D") { auto* v = reinterpret_cast<FVector2D*>(base); v->X = getNum("x"); v->Y = getNum("y"); return true; }
			if (err) *err = "unsupported struct type: " + sn;
			return false;
		}

		if (err) *err = "unsupported property type: " + tname;
		return false;
	}

	// --- Object handle accessors ----------------------------------------

	int l_cathack_local_player_controller(lua_State* L) { PushObjectId(L, GVars.PlayerController); return 1; }
	int l_cathack_local_character(lua_State* L)         { PushObjectId(L, GVars.Character);        return 1; }
	int l_cathack_local_pawn(lua_State* L)              { PushObjectId(L, GVars.Pawn);             return 1; }
	int l_cathack_player_camera_manager(lua_State* L)
	{
		auto* PC = GVars.PlayerController;
		if (!PC || !Utils::IsLikelyLiveUObject(PC)) { lua_pushnil(L); return 1; }
		PushObjectId(L, PC->PlayerCameraManager);
		return 1;
	}

	int l_cathack_object_valid(lua_State* L)
	{
		auto* o = ResolveObject(luaL_checkinteger(L, 1));
		lua_pushboolean(L, o ? 1 : 0);
		return 1;
	}
	int l_cathack_object_name(lua_State* L)
	{
		auto* o = ResolveObject(luaL_checkinteger(L, 1));
		if (!o) { lua_pushnil(L); return 1; }
		const std::string s = o->GetName();
		lua_pushstring(L, s.c_str());
		return 1;
	}
	int l_cathack_object_class(lua_State* L)
	{
		auto* o = ResolveObject(luaL_checkinteger(L, 1));
		if (!o || !o->Class) { lua_pushnil(L); return 1; }
		const std::string s = o->Class->GetName();
		lua_pushstring(L, s.c_str());
		return 1;
	}
	int l_cathack_object_path(lua_State* L)
	{
		auto* o = ResolveObject(luaL_checkinteger(L, 1));
		if (!o) { lua_pushnil(L); return 1; }
		const std::string s = o->GetFullName();
		lua_pushstring(L, s.c_str());
		return 1;
	}

	// Walk UClass chain and emit one descriptor table per FProperty.
	int l_cathack_object_properties(lua_State* L)
	{
		auto* o = ResolveObject(luaL_checkinteger(L, 1));
		lua_newtable(L);
		if (!o || !o->Class) return 1;

		int outIdx = 1;
		UClass* cls = o->Class;
		while (cls)
		{
			const std::string ownerClass = cls->GetName();
			FField* f = cls->ChildProperties;
			while (f)
			{
				const FProperty* p = reinterpret_cast<const FProperty*>(f);
				const std::string nm = f->Name.ToString();
				const std::string tn = FieldTypeName(f);
				const char*       cat = CategorizeProperty(tn);
				std::string       structHint;
				if (tn == "StructProperty")
				{
					const UStruct* s = *reinterpret_cast<const UStruct* const*>(reinterpret_cast<const uint8_t*>(p) + 0x70);
					if (s) structHint = const_cast<UStruct*>(s)->GetName();
				}
				const bool writable = PropertyIsLuaWritable(tn, structHint);
				const bool readable = (*cat) != 0;

				lua_newtable(L);
				lua_pushstring(L, nm.c_str());        lua_setfield(L, -2, "name");
				lua_pushstring(L, tn.c_str());        lua_setfield(L, -2, "raw_type");
				lua_pushstring(L, cat);               lua_setfield(L, -2, "type");
				if (!structHint.empty()) {
					lua_pushstring(L, structHint.c_str()); lua_setfield(L, -2, "struct");
				}
				lua_pushinteger(L, p->Offset);        lua_setfield(L, -2, "offset");
				lua_pushinteger(L, p->ElementSize);   lua_setfield(L, -2, "size");
				lua_pushinteger(L, p->ArrayDim);      lua_setfield(L, -2, "array_dim");
				lua_pushboolean(L, readable ? 1 : 0); lua_setfield(L, -2, "readable");
				lua_pushboolean(L, writable ? 1 : 0); lua_setfield(L, -2, "writable");
				lua_pushstring(L, ownerClass.c_str()); lua_setfield(L, -2, "owner");
				lua_rawseti(L, -2, outIdx++);
				f = f->Next;
			}
			UStruct* super_ = cls->SuperStruct;
			cls = (super_ && super_ != cls) ? reinterpret_cast<UClass*>(super_) : nullptr;
		}
		return 1;
	}

	// Pack a UFunction's parameter list into a Lua table of descriptors.
	// Used by both object_functions and the safety check inside object_call.
	static void PackParamsTable(lua_State* L, UFunction* fn)
	{
		lua_newtable(L);
		if (!fn) return;
		FField* f = fn->ChildProperties;
		int idx = 1;
		while (f)
		{
			const FProperty* p = reinterpret_cast<const FProperty*>(f);
			if (p->PropertyFlags & kCPF_Parm)
			{
				const std::string tn = FieldTypeName(f);
				const char*       cat = CategorizeProperty(tn);
				std::string structHint;
				if (tn == "StructProperty")
				{
					const UStruct* s = *reinterpret_cast<const UStruct* const*>(reinterpret_cast<const uint8_t*>(p) + 0x70);
					if (s) structHint = const_cast<UStruct*>(s)->GetName();
				}

				lua_newtable(L);
				lua_pushstring(L, f->Name.ToString().c_str()); lua_setfield(L, -2, "name");
				lua_pushstring(L, cat);                        lua_setfield(L, -2, "type");
				lua_pushstring(L, tn.c_str());                 lua_setfield(L, -2, "raw_type");
				if (!structHint.empty()) { lua_pushstring(L, structHint.c_str()); lua_setfield(L, -2, "struct"); }
				lua_pushboolean(L, (p->PropertyFlags & kCPF_OutParm) ? 1 : 0);    lua_setfield(L, -2, "out");
				lua_pushboolean(L, (p->PropertyFlags & kCPF_ReturnParm) ? 1 : 0); lua_setfield(L, -2, "ret");
				lua_pushboolean(L, (p->PropertyFlags & kCPF_ConstParm) ? 1 : 0);  lua_setfield(L, -2, "is_const");
				lua_rawseti(L, -2, idx++);
			}
			f = f->Next;
		}
	}

	// Whether a UFunction's full signature is safe to call from Lua —
	// every input parameter (and the return value, if any) must lie in
	// our typed whitelist. Functions taking object pointers, arrays,
	// maps, sets, classes, or unsupported structs are refused.
	static bool FunctionIsLuaSafe(UFunction* fn, std::string* whyNot)
	{
		if (!fn) { if (whyNot) *whyNot = "null function"; return false; }
		FField* f = fn->ChildProperties;
		while (f)
		{
			const FProperty* p = reinterpret_cast<const FProperty*>(f);
			if (p->PropertyFlags & kCPF_Parm)
			{
				const std::string tn = FieldTypeName(f);
				bool ok =
					tn == "BoolProperty" || tn == "ByteProperty"
					|| tn == "Int8Property" || tn == "Int16Property"
					|| tn == "IntProperty" || tn == "Int64Property"
					|| tn == "UInt16Property" || tn == "UInt32Property" || tn == "UInt64Property"
					|| tn == "FloatProperty" || tn == "DoubleProperty"
					|| tn == "NameProperty" || tn == "StrProperty"
					|| tn == "EnumProperty";
				if (!ok && tn == "StructProperty")
				{
					const UStruct* s = *reinterpret_cast<const UStruct* const*>(reinterpret_cast<const uint8_t*>(p) + 0x70);
					std::string sn = s ? const_cast<UStruct*>(s)->GetName() : std::string();
					ok = (sn == "Vector" || sn == "Rotator" || sn == "Vector2D");
				}
				if (!ok)
				{
					if (whyNot) *whyNot = "unsafe parameter type: " + tn;
					return false;
				}
			}
			f = f->Next;
		}
		return true;
	}

	int l_cathack_object_functions(lua_State* L)
	{
		auto* o = ResolveObject(luaL_checkinteger(L, 1));
		lua_newtable(L);
		if (!o || !o->Class) return 1;

		int outIdx = 1;
		UClass* cls = o->Class;
		while (cls)
		{
			const std::string ownerClass = cls->GetName();
			UField* fld = cls->Children;
			while (fld)
			{
				if (fld->IsA(UFunction::StaticClass()))
				{
					UFunction* fn = reinterpret_cast<UFunction*>(fld);
					std::string why;
					const bool safe = FunctionIsLuaSafe(fn, &why);
					lua_newtable(L);
					lua_pushstring(L, fn->Name.ToString().c_str()); lua_setfield(L, -2, "name");
					lua_pushboolean(L, 1);                          lua_setfield(L, -2, "callable");
					lua_pushboolean(L, safe ? 1 : 0);               lua_setfield(L, -2, "safe");
					if (!safe) { lua_pushstring(L, why.c_str()); lua_setfield(L, -2, "reason"); }
					lua_pushstring(L, ownerClass.c_str());          lua_setfield(L, -2, "owner");
					PackParamsTable(L, fn);                         lua_setfield(L, -2, "params");
					lua_rawseti(L, -2, outIdx++);
				}
				fld = fld->Next;
			}
			UStruct* super_ = cls->SuperStruct;
			cls = (super_ && super_ != cls) ? reinterpret_cast<UClass*>(super_) : nullptr;
		}
		return 1;
	}

	int l_cathack_object_find_properties(lua_State* L)
	{
		const lua_Integer  oid    = luaL_checkinteger(L, 1);
		const char*        query  = luaL_optstring(L, 2, "");
		const char*        filter = luaL_optstring(L, 3, "");
		// Reuse object_properties + filter in Lua via our own loop. We do
		// the filtering inline to avoid building the full table only to
		// throw most of it away.
		auto* o = ResolveObject(oid);
		lua_newtable(L);
		if (!o || !o->Class) return 1;
		int outIdx = 1;
		std::string qLow = query;
		for (auto& c : qLow) c = static_cast<char>(std::tolower(static_cast<unsigned char>(c)));
		UClass* cls = o->Class;
		while (cls)
		{
			const std::string ownerClass = cls->GetName();
			FField* f = cls->ChildProperties;
			while (f)
			{
				const FProperty* p = reinterpret_cast<const FProperty*>(f);
				const std::string nm = f->Name.ToString();
				const std::string tn = FieldTypeName(f);
				const char*       cat = CategorizeProperty(tn);
				bool match = true;
				if (*query) {
					std::string nmLow = nm;
					for (auto& c : nmLow) c = static_cast<char>(std::tolower(static_cast<unsigned char>(c)));
					match = nmLow.find(qLow) != std::string::npos;
				}
				if (match && *filter && std::strcmp(filter, cat) != 0) match = false;
				if (match)
				{
					std::string structHint;
					if (tn == "StructProperty")
					{
						const UStruct* s = *reinterpret_cast<const UStruct* const*>(reinterpret_cast<const uint8_t*>(p) + 0x70);
						if (s) structHint = const_cast<UStruct*>(s)->GetName();
					}
					lua_newtable(L);
					lua_pushstring(L, nm.c_str());        lua_setfield(L, -2, "name");
					lua_pushstring(L, cat);               lua_setfield(L, -2, "type");
					lua_pushstring(L, tn.c_str());        lua_setfield(L, -2, "raw_type");
					if (!structHint.empty()) { lua_pushstring(L, structHint.c_str()); lua_setfield(L, -2, "struct"); }
					lua_pushinteger(L, p->Offset);        lua_setfield(L, -2, "offset");
					lua_pushboolean(L, (*cat) != 0 ? 1 : 0); lua_setfield(L, -2, "readable");
					lua_pushboolean(L, PropertyIsLuaWritable(tn, structHint) ? 1 : 0); lua_setfield(L, -2, "writable");
					lua_pushstring(L, ownerClass.c_str()); lua_setfield(L, -2, "owner");
					lua_rawseti(L, -2, outIdx++);
				}
				f = f->Next;
			}
			UStruct* super_ = cls->SuperStruct;
			cls = (super_ && super_ != cls) ? reinterpret_cast<UClass*>(super_) : nullptr;
		}
		return 1;
	}

	int l_cathack_object_find_functions(lua_State* L)
	{
		const lua_Integer oid   = luaL_checkinteger(L, 1);
		const char*       query = luaL_optstring(L, 2, "");
		auto* o = ResolveObject(oid);
		lua_newtable(L);
		if (!o || !o->Class) return 1;
		int outIdx = 1;
		std::string qLow = query;
		for (auto& c : qLow) c = static_cast<char>(std::tolower(static_cast<unsigned char>(c)));
		UClass* cls = o->Class;
		while (cls)
		{
			const std::string ownerClass = cls->GetName();
			UField* fld = cls->Children;
			while (fld)
			{
				if (fld->IsA(UFunction::StaticClass()))
				{
					UFunction* fn = reinterpret_cast<UFunction*>(fld);
					const std::string nm = fn->Name.ToString();
					bool match = true;
					if (*query) {
						std::string nmLow = nm;
						for (auto& c : nmLow) c = static_cast<char>(std::tolower(static_cast<unsigned char>(c)));
						match = nmLow.find(qLow) != std::string::npos;
					}
					if (match)
					{
						std::string why;
						const bool safe = FunctionIsLuaSafe(fn, &why);
						lua_newtable(L);
						lua_pushstring(L, nm.c_str());          lua_setfield(L, -2, "name");
						lua_pushboolean(L, 1);                  lua_setfield(L, -2, "callable");
						lua_pushboolean(L, safe ? 1 : 0);       lua_setfield(L, -2, "safe");
						if (!safe) { lua_pushstring(L, why.c_str()); lua_setfield(L, -2, "reason"); }
						lua_pushstring(L, ownerClass.c_str());  lua_setfield(L, -2, "owner");
						PackParamsTable(L, fn);                 lua_setfield(L, -2, "params");
						lua_rawseti(L, -2, outIdx++);
					}
				}
				fld = fld->Next;
			}
			UStruct* super_ = cls->SuperStruct;
			cls = (super_ && super_ != cls) ? reinterpret_cast<UClass*>(super_) : nullptr;
		}
		return 1;
	}

	int l_cathack_object_get(lua_State* L)
	{
		auto* o = ResolveObject(luaL_checkinteger(L, 1));
		const char* prop = luaL_checkstring(L, 2);
		if (!o) { lua_pushnil(L); return 1; }
		const FProperty* p = FindPropertyByName(o, prop);
		if (!p) { lua_pushnil(L); return 1; }
		const std::string tn = FieldTypeName(reinterpret_cast<const FField*>(p));
		PropertyGetAndPush(L, o, p, tn);
		return 1;
	}

	int l_cathack_object_set(lua_State* L)
	{
		auto* o = ResolveObject(luaL_checkinteger(L, 1));
		const char* prop = luaL_checkstring(L, 2);
		if (!o) { lua_pushboolean(L, 0); lua_pushstring(L, "invalid object"); return 2; }
		const FProperty* p = FindPropertyByName(o, prop);
		if (!p) { lua_pushboolean(L, 0); lua_pushstring(L, "property not found"); return 2; }
		const std::string tn = FieldTypeName(reinterpret_cast<const FField*>(p));
		std::string structHint;
		if (tn == "StructProperty")
		{
			const UStruct* s = *reinterpret_cast<const UStruct* const*>(reinterpret_cast<const uint8_t*>(p) + 0x70);
			if (s) structHint = const_cast<UStruct*>(s)->GetName();
		}
		if (!PropertyIsLuaWritable(tn, structHint))
		{
			lua_pushboolean(L, 0);
			lua_pushfstring(L, "type %s is not writable from Lua", tn.c_str());
			return 2;
		}
		std::string err;
		const bool ok = PropertySetFromLua(L, 3, o, p, tn, &err);
		lua_pushboolean(L, ok ? 1 : 0);
		if (!ok) lua_pushstring(L, err.empty() ? "set failed" : err.c_str());
		return ok ? 1 : 2;
	}

	// Pack arguments from a Lua table into a UFunction call buffer and
	// invoke ProcessEvent. Returns 2 values to Lua: ok (bool) and either
	// a result value (when there's a return parm) or an error string.
	int l_cathack_object_call(lua_State* L)
	{
		auto* o = ResolveObject(luaL_checkinteger(L, 1));
		const char* fname = luaL_checkstring(L, 2);
		if (!o) { lua_pushboolean(L, 0); lua_pushstring(L, "invalid object"); return 2; }
		UFunction* fn = FindFunctionByName(o, fname);
		if (!fn) { lua_pushboolean(L, 0); lua_pushstring(L, "function not found"); return 2; }
		std::string why;
		if (!FunctionIsLuaSafe(fn, &why)) { lua_pushboolean(L, 0); lua_pushstring(L, why.c_str()); return 2; }

		// Build the Params buffer: each FProperty contributes ElementSize
		// bytes laid out in declaration order. The buffer starts as zero.
		// Cap the size so a malicious / huge function descriptor can't
		// blow the stack — anything that legitimately needs >4 KB is
		// already outside what Lua reflection should be doing.
		struct ParmInfo { const FProperty* p; std::string tn; std::string structHint; bool isRet; bool isOut; };
		std::vector<ParmInfo> parms;
		size_t totalSize = 0;
		FField* f = fn->ChildProperties;
		while (f)
		{
			const FProperty* p = reinterpret_cast<const FProperty*>(f);
			if (p->PropertyFlags & kCPF_Parm)
			{
				ParmInfo pi;
				pi.p  = p;
				pi.tn = FieldTypeName(f);
				pi.isRet = (p->PropertyFlags & kCPF_ReturnParm) != 0;
				pi.isOut = (p->PropertyFlags & kCPF_OutParm) != 0;
				if (pi.tn == "StructProperty")
				{
					const UStruct* s = *reinterpret_cast<const UStruct* const*>(reinterpret_cast<const uint8_t*>(p) + 0x70);
					if (s) pi.structHint = const_cast<UStruct*>(s)->GetName();
				}
				parms.push_back(std::move(pi));
				const size_t end = static_cast<size_t>(p->Offset) + static_cast<size_t>(p->ElementSize);
				if (end > totalSize) totalSize = end;
			}
			f = f->Next;
		}
		if (totalSize > 4096) { lua_pushboolean(L, 0); lua_pushstring(L, "param buffer too large"); return 2; }
		std::vector<uint8_t> buf(totalSize, 0);

		// Optional Lua-side args table sits at index 3. Walk in
		// declaration order, mapping table[1]→first input param, etc.
		const bool hasArgs = lua_type(L, 3) == LUA_TTABLE;
		int inIdx = 1;
		for (const auto& pi : parms)
		{
			if (pi.isRet) continue; // skip — ProcessEvent fills it in
			if (!hasArgs)            continue; // leave zero default
			lua_rawgeti(L, 3, inIdx++);
			const int valIdx = lua_gettop(L);

			// Build a tiny "obj" pretending to be a UObject pointed at
			// `buf` so PropertySetFromLua's offset math (obj+offset)
			// lands inside our buffer. The function never reads any
			// fields off this fake UObject, so the trick is safe.
			UObject* fakeObj = reinterpret_cast<UObject*>(buf.data() - pi.p->Offset);
			std::string err;
			const bool ok = PropertySetFromLua(L, valIdx, fakeObj, pi.p, pi.tn, &err);
			lua_pop(L, 1);
			if (!ok)
			{
				lua_pushboolean(L, 0);
				lua_pushfstring(L, "arg pack failed: %s", err.empty() ? "?" : err.c_str());
				return 2;
			}
		}

		// Bracket the actual UE call in a SEH-only helper. The trivial
		// helper has no destructible locals so MSVC under /EHsc accepts
		// the __try; we get a soft failure (returns false) on access
		// violations from a stale function pointer or a mis-shaped
		// param buffer instead of taking the whole DLL down.
		if (!Engine::SafeProcessEvent(o, fn, buf.data()))
		{
			lua_pushboolean(L, 0);
			lua_pushstring(L, "ProcessEvent threw");
			return 2;
		}

		// If there's a return parm, push it. Otherwise just push true.
		lua_pushboolean(L, 1);
		for (const auto& pi : parms)
		{
			if (!pi.isRet) continue;
			UObject* fakeObj = reinterpret_cast<UObject*>(buf.data() - pi.p->Offset);
			PropertyGetAndPush(L, fakeObj, pi.p, pi.tn);
			return 2;
		}
		return 1;
	}

	// --- PlayerController convenience helpers ---------------------------

	int l_cathack_pc_mouse_cursor(lua_State* L)
	{
		auto* PC = GVars.PlayerController;
		if (!PC || !Utils::IsLikelyLiveUObject(PC)) { lua_pushnil(L); return 1; }
		lua_pushboolean(L, PC->bShowMouseCursor ? 1 : 0);
		return 1;
	}
	int l_cathack_pc_set_mouse_cursor(lua_State* L)
	{
		auto* PC = GVars.PlayerController;
		if (!PC || !Utils::IsLikelyLiveUObject(PC)) { lua_pushboolean(L, 0); return 1; }
		const bool v = lua_toboolean(L, 1) != 0;
		PC->bShowMouseCursor = v;
		lua_pushboolean(L, 1);
		return 1;
	}

	// Best-effort input-mode emulation. UE has SetInputMode(Game/UI/GameAndUI)
	// helpers that flip several flags at once; those static helpers aren't
	// reflected in Dumper7's SDK, so we set the same flags by hand.
	int l_cathack_pc_input_mode(lua_State* L)
	{
		auto* PC = GVars.PlayerController;
		if (!PC || !Utils::IsLikelyLiveUObject(PC)) { lua_pushnil(L); return 1; }
		const bool cur = PC->bShowMouseCursor != 0;
		const bool clicks = PC->bEnableClickEvents != 0;
		// Heuristic mapping — the engine itself only cares about the bit
		// flags, not the original mode.
		if (cur && clicks) lua_pushstring(L, "ui");
		else if (cur)      lua_pushstring(L, "game_and_ui");
		else               lua_pushstring(L, "game");
		return 1;
	}
	int l_cathack_pc_set_input_mode(lua_State* L)
	{
		auto* PC = GVars.PlayerController;
		const char* mode = luaL_checkstring(L, 1);
		if (!PC || !Utils::IsLikelyLiveUObject(PC)) { lua_pushboolean(L, 0); return 1; }
		if (!std::strcmp(mode, "game"))
		{
			PC->bShowMouseCursor       = false;
			PC->bEnableClickEvents     = false;
			PC->bEnableMouseOverEvents = false;
		}
		else if (!std::strcmp(mode, "ui"))
		{
			PC->bShowMouseCursor       = true;
			PC->bEnableClickEvents     = true;
			PC->bEnableMouseOverEvents = true;
		}
		else if (!std::strcmp(mode, "game_and_ui"))
		{
			PC->bShowMouseCursor       = true;
			PC->bEnableClickEvents     = false;
			PC->bEnableMouseOverEvents = false;
		}
		else
		{
			lua_pushboolean(L, 0);
			return 1;
		}
		lua_pushboolean(L, 1);
		return 1;
	}

	int l_cathack_pc_control_rotation(lua_State* L)
	{
		auto* PC = GVars.PlayerController;
		if (!PC || !Utils::IsLikelyLiveUObject(PC)) return 0;
		const FRotator r = PC->GetControlRotation();
		lua_pushnumber(L, r.Pitch);
		lua_pushnumber(L, r.Yaw);
		lua_pushnumber(L, r.Roll);
		return 3;
	}
	int l_cathack_pc_set_control_rotation(lua_State* L)
	{
		auto* PC = GVars.PlayerController;
		if (!PC || !Utils::IsLikelyLiveUObject(PC)) { lua_pushboolean(L, 0); return 1; }
		FRotator r;
		r.Pitch = static_cast<float>(luaL_checknumber(L, 1));
		r.Yaw   = static_cast<float>(luaL_checknumber(L, 2));
		r.Roll  = static_cast<float>(luaL_optnumber(L, 3, 0.0));
		PC->SetControlRotation(r);
		lua_pushboolean(L, 1);
		return 1;
	}

	int l_cathack_pc_view_target(lua_State* L)
	{
		auto* PC = GVars.PlayerController;
		if (!PC || !Utils::IsLikelyLiveUObject(PC)) { lua_pushnil(L); return 1; }
		AActor* a = PC->GetViewTarget();
		PushObjectId(L, a);
		return 1;
	}
	int l_cathack_pc_set_view_target(lua_State* L)
	{
		auto* PC = GVars.PlayerController;
		if (!PC || !Utils::IsLikelyLiveUObject(PC)) { lua_pushboolean(L, 0); return 1; }
		AActor* target = nullptr;
		if (lua_type(L, 1) == LUA_TNUMBER)
		{
			auto* o = ResolveObject(lua_tointeger(L, 1));
			if (o && o->IsA(AActor::StaticClass()))
				target = reinterpret_cast<AActor*>(o);
		}
		const float blendTime = static_cast<float>(luaL_optnumber(L, 2, 0.0));
		if (blendTime > 0.f)
			PC->SetViewTargetWithBlend(target, blendTime, EViewTargetBlendFunction::VTBlend_Linear, 0.f, false);
		else
		{
			FViewTargetTransitionParams t{};
			PC->ClientSetViewTarget(target, t);
		}
		lua_pushboolean(L, 1);
		return 1;
	}

	int l_cathack_pc_project_world_to_screen(lua_State* L)
	{
		auto* PC = GVars.PlayerController;
		if (!PC || !Utils::IsLikelyLiveUObject(PC)) { lua_pushnil(L); return 1; }
		FVector wp;
		wp.X = static_cast<double>(luaL_checknumber(L, 1));
		wp.Y = static_cast<double>(luaL_checknumber(L, 2));
		wp.Z = static_cast<double>(luaL_checknumber(L, 3));
		FVector2D sp{};
		const bool ok = PC->ProjectWorldLocationToScreen(wp, &sp, false);
		if (!ok) { lua_pushnil(L); return 1; }
		lua_pushnumber(L, sp.X);
		lua_pushnumber(L, sp.Y);
		return 2;
	}
	int l_cathack_pc_deproject_screen_to_world(lua_State* L)
	{
		auto* PC = GVars.PlayerController;
		if (!PC || !Utils::IsLikelyLiveUObject(PC)) { lua_pushnil(L); return 1; }
		const float sx = static_cast<float>(luaL_checknumber(L, 1));
		const float sy = static_cast<float>(luaL_checknumber(L, 2));
		FVector wl{}, wd{};
		const bool ok = PC->DeprojectScreenPositionToWorld(sx, sy, &wl, &wd);
		if (!ok) { lua_pushnil(L); return 1; }
		// origin
		lua_createtable(L, 0, 3);
		lua_pushnumber(L, wl.X); lua_setfield(L, -2, "x");
		lua_pushnumber(L, wl.Y); lua_setfield(L, -2, "y");
		lua_pushnumber(L, wl.Z); lua_setfield(L, -2, "z");
		// direction
		lua_createtable(L, 0, 3);
		lua_pushnumber(L, wd.X); lua_setfield(L, -2, "x");
		lua_pushnumber(L, wd.Y); lua_setfield(L, -2, "y");
		lua_pushnumber(L, wd.Z); lua_setfield(L, -2, "z");
		return 2;
	}

	// =====================================================================
	// ImGui wrappers — let scripts build full menus instead of only
	// flat HUD overlays. Every wrapper is render-thread-only (gated on
	// tl_currentDrawList being set, which is true only while we're
	// inside on_render dispatch). Calling any of these from on_tick is
	// a no-op; they don't error so a misplaced call doesn't kill the
	// script.
	//
	// Bracketing safety: ImGui requires every Begin* to have a matching
	// End*. Some pairs (Begin/End, BeginChild/EndChild) are
	// unconditional; others (BeginTabBar, BeginTabItem, BeginTable,
	// TreeNode, BeginTooltip) only end if the begin returned true.
	// Counters track open scopes per frame and TickRender auto-closes
	// any that the script left open (because the watchdog longjmp'd
	// out, the script crashed, etc.). This keeps a buggy Lua script
	// from taking down the host menu's ImGui state.
	//
	// Mouse / keyboard capture is automatic. Once ImGui sees a Lua
	// window hovered or active it sets io.WantCaptureMouse /
	// WantCaptureKeyboard / WantTextInput. The host's WndProc already
	// reads those flags and swallows the matching Win32 messages, so
	// clicks on a Lua-built panel never bleed through to the game's
	// shoot / move handlers. Cursor visibility is auto-forced on while
	// any imgui_begin fires this frame so users can see what they're
	// clicking even when the cheat menu is closed.
	// =====================================================================

	// Per-frame depth counters. Kept simple ints — only touched from
	// the render thread under the Lua mutex, no synchronization needed.
	static int  g_imBeginDepth        = 0;  // Begin / End          (unconditional pair)
	static int  g_imChildDepth        = 0;  // BeginChild / EndChild (unconditional pair)
	static int  g_imGroupDepth        = 0;  // BeginGroup / EndGroup
	static int  g_imTabBarDepth       = 0;  // BeginTabBar / EndTabBar (only if true)
	static int  g_imTabItemDepth      = 0;  // BeginTabItem / EndTabItem (only if true)
	static int  g_imTableDepth        = 0;  // BeginTable / EndTable (only if true)
	static int  g_imTreeDepth         = 0;  // TreeNode / TreePop (only if true)
	static int  g_imTooltipDepth      = 0;  // BeginTooltip / EndTooltip (only if true)
	static int  g_imStyleColorDepth   = 0;  // PushStyleColor — popped en masse with count
	static int  g_imStyleVarDepth     = 0;  // PushStyleVar — same
	static int  g_imIDDepth           = 0;  // PushID / PopID
	static int  g_imIndentLevel       = 0;  // Indent / Unindent
	static bool g_imAnyBeginThisFrame = false;

	// Reset per-frame state. Called at the top of TickRender's render
	// dispatch loop. Doesn't touch ImGui itself — just our counters.
	void ResetImGuiPerFrame()
	{
		g_imBeginDepth = g_imChildDepth = g_imGroupDepth = 0;
		g_imTabBarDepth = g_imTabItemDepth = g_imTableDepth = 0;
		g_imTreeDepth = g_imTooltipDepth = 0;
		g_imStyleColorDepth = g_imStyleVarDepth = g_imIDDepth = 0;
		g_imIndentLevel = 0;
		g_imAnyBeginThisFrame = false;
	}

	// Auto-close anything the script forgot. Order matters — innermost
	// scopes first. Anything ImGui says "only call End if Begin returned
	// true" is gated on its own depth counter so we won't double-close.
	void CleanupImGuiPerFrame()
	{
		while (g_imTreeDepth > 0)      { ImGui::TreePop();       --g_imTreeDepth; }
		while (g_imTabItemDepth > 0)   { ImGui::EndTabItem();    --g_imTabItemDepth; }
		while (g_imTabBarDepth > 0)    { ImGui::EndTabBar();     --g_imTabBarDepth; }
		while (g_imTableDepth > 0)     { ImGui::EndTable();      --g_imTableDepth; }
		while (g_imTooltipDepth > 0)   { ImGui::EndTooltip();    --g_imTooltipDepth; }
		while (g_imGroupDepth > 0)     { ImGui::EndGroup();      --g_imGroupDepth; }
		while (g_imChildDepth > 0)     { ImGui::EndChild();      --g_imChildDepth; }
		while (g_imBeginDepth > 0)     { ImGui::End();           --g_imBeginDepth; }
		// Unindent before popping IDs/styles — Indent is tied to
		// surrounding window context which we just closed above.
		while (g_imIndentLevel > 0)    { ImGui::Unindent();      --g_imIndentLevel; }
		while (g_imIDDepth > 0)        { ImGui::PopID();         --g_imIDDepth; }
		if (g_imStyleVarDepth > 0)     { ImGui::PopStyleVar(g_imStyleVarDepth);   g_imStyleVarDepth = 0; }
		if (g_imStyleColorDepth > 0)   { ImGui::PopStyleColor(g_imStyleColorDepth); g_imStyleColorDepth = 0; }
	}

	// Tiny helper — every wrapper is render-thread-only.
	static inline bool ImRenderGuard() { return tl_currentDrawList != nullptr; }

	// ---- Windows ------------------------------------------------------
	int l_cathack_imgui_begin(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const char* name = luaL_checkstring(L, 1);
		const bool wantClose = lua_type(L, 2) == LUA_TBOOLEAN
			? (lua_toboolean(L, 2) != 0)
			: false;
		const int flags = static_cast<int>(luaL_optinteger(L, 3, 0));

		bool isOpen = true;
		const bool visible = ImGui::Begin(name, wantClose ? &isOpen : nullptr,
			static_cast<ImGuiWindowFlags>(flags));
		++g_imBeginDepth;
		g_imAnyBeginThisFrame = true;

		lua_pushboolean(L, visible ? 1 : 0);
		if (wantClose)
		{
			lua_pushboolean(L, isOpen ? 1 : 0);
			return 2;
		}
		return 1;
	}

	int l_cathack_imgui_end(lua_State* L)
	{
		(void)L;
		if (!ImRenderGuard()) return 0;
		if (g_imBeginDepth <= 0) return 0;
		ImGui::End();
		--g_imBeginDepth;
		return 0;
	}

	// ---- Child windows -----------------------------------------------
	int l_cathack_imgui_begin_child(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const char* id = luaL_checkstring(L, 1);
		const float w = static_cast<float>(luaL_optnumber(L, 2, 0.0));
		const float h = static_cast<float>(luaL_optnumber(L, 3, 0.0));
		const bool border = lua_isnoneornil(L, 4) ? false : (lua_toboolean(L, 4) != 0);
		const int flags = static_cast<int>(luaL_optinteger(L, 5, 0));

		const bool visible = ImGui::BeginChild(id, ImVec2(w, h),
			border ? ImGuiChildFlags_Border : 0,
			static_cast<ImGuiWindowFlags>(flags));
		++g_imChildDepth;
		lua_pushboolean(L, visible ? 1 : 0);
		return 1;
	}

	int l_cathack_imgui_end_child(lua_State* L)
	{
		(void)L;
		if (!ImRenderGuard()) return 0;
		if (g_imChildDepth <= 0) return 0;
		ImGui::EndChild();
		--g_imChildDepth;
		return 0;
	}

	// ---- Layout / cursor ---------------------------------------------
	int l_cathack_imgui_same_line(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		const float xOffset = static_cast<float>(luaL_optnumber(L, 1, 0.0));
		const float spacing = static_cast<float>(luaL_optnumber(L, 2, -1.0));
		ImGui::SameLine(xOffset, spacing);
		return 0;
	}
	int l_cathack_imgui_separator(lua_State* L)   { (void)L; if (ImRenderGuard()) ImGui::Separator(); return 0; }
	int l_cathack_imgui_spacing(lua_State* L)     { (void)L; if (ImRenderGuard()) ImGui::Spacing();   return 0; }
	int l_cathack_imgui_new_line(lua_State* L)    { (void)L; if (ImRenderGuard()) ImGui::NewLine();   return 0; }
	int l_cathack_imgui_dummy(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		const float w = static_cast<float>(luaL_optnumber(L, 1, 0.0));
		const float h = static_cast<float>(luaL_optnumber(L, 2, 0.0));
		ImGui::Dummy(ImVec2(w, h));
		return 0;
	}
	int l_cathack_imgui_indent(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		const float w = static_cast<float>(luaL_optnumber(L, 1, 0.0));
		ImGui::Indent(w);
		++g_imIndentLevel;
		return 0;
	}
	int l_cathack_imgui_unindent(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		if (g_imIndentLevel <= 0) return 0;
		const float w = static_cast<float>(luaL_optnumber(L, 1, 0.0));
		ImGui::Unindent(w);
		--g_imIndentLevel;
		return 0;
	}
	int l_cathack_imgui_begin_group(lua_State* L) { (void)L; if (ImRenderGuard()) { ImGui::BeginGroup(); ++g_imGroupDepth; } return 0; }
	int l_cathack_imgui_end_group(lua_State* L)
	{
		(void)L;
		if (!ImRenderGuard() || g_imGroupDepth <= 0) return 0;
		ImGui::EndGroup();
		--g_imGroupDepth;
		return 0;
	}
	int l_cathack_imgui_set_next_window_pos(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		const float x = static_cast<float>(luaL_checknumber(L, 1));
		const float y = static_cast<float>(luaL_checknumber(L, 2));
		const int cond = static_cast<int>(luaL_optinteger(L, 3, 0));
		ImGui::SetNextWindowPos(ImVec2(x, y), static_cast<ImGuiCond>(cond));
		return 0;
	}
	int l_cathack_imgui_set_next_window_size(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		const float w = static_cast<float>(luaL_checknumber(L, 1));
		const float h = static_cast<float>(luaL_checknumber(L, 2));
		const int cond = static_cast<int>(luaL_optinteger(L, 3, 0));
		ImGui::SetNextWindowSize(ImVec2(w, h), static_cast<ImGuiCond>(cond));
		return 0;
	}

	// ---- Text / labels -----------------------------------------------
	int l_cathack_imgui_text(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		ImGui::TextUnformatted(luaL_checkstring(L, 1));
		return 0;
	}
	int l_cathack_imgui_text_colored(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		const lua_Integer color = luaL_checkinteger(L, 1);
		const char* s = luaL_checkstring(L, 2);
		ImGui::PushStyleColor(ImGuiCol_Text, static_cast<ImU32>(color));
		ImGui::TextUnformatted(s);
		ImGui::PopStyleColor();
		return 0;
	}
	int l_cathack_imgui_text_disabled(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		ImGui::TextDisabled("%s", luaL_checkstring(L, 1));
		return 0;
	}
	int l_cathack_imgui_text_wrapped(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		ImGui::TextWrapped("%s", luaL_checkstring(L, 1));
		return 0;
	}
	int l_cathack_imgui_bullet_text(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		ImGui::BulletText("%s", luaL_checkstring(L, 1));
		return 0;
	}
	int l_cathack_imgui_label_text(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		const char* label = luaL_checkstring(L, 1);
		const char* text  = luaL_checkstring(L, 2);
		ImGui::LabelText(label, "%s", text);
		return 0;
	}

	// ---- Buttons / clickables ----------------------------------------
	int l_cathack_imgui_button(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const char* label = luaL_checkstring(L, 1);
		const float w = static_cast<float>(luaL_optnumber(L, 2, 0.0));
		const float h = static_cast<float>(luaL_optnumber(L, 3, 0.0));
		lua_pushboolean(L, ImGui::Button(label, ImVec2(w, h)) ? 1 : 0);
		return 1;
	}
	int l_cathack_imgui_small_button(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		lua_pushboolean(L, ImGui::SmallButton(luaL_checkstring(L, 1)) ? 1 : 0);
		return 1;
	}
	int l_cathack_imgui_invisible_button(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const char* id = luaL_checkstring(L, 1);
		const float w = static_cast<float>(luaL_checknumber(L, 2));
		const float h = static_cast<float>(luaL_checknumber(L, 3));
		lua_pushboolean(L, ImGui::InvisibleButton(id, ImVec2(w, h)) ? 1 : 0);
		return 1;
	}
	int l_cathack_imgui_arrow_button(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const char* id  = luaL_checkstring(L, 1);
		const int   dir = static_cast<int>(luaL_optinteger(L, 2, ImGuiDir_Right));
		lua_pushboolean(L, ImGui::ArrowButton(id, static_cast<ImGuiDir>(dir)) ? 1 : 0);
		return 1;
	}

	// ---- Checkbox / toggles ------------------------------------------
	int l_cathack_imgui_checkbox(lua_State* L)
	{
		if (!ImRenderGuard())
		{
			lua_pushboolean(L, lua_toboolean(L, 2));
			lua_pushboolean(L, 0);
			return 2;
		}
		const char* label = luaL_checkstring(L, 1);
		bool value = lua_toboolean(L, 2) != 0;
		const bool changed = ImGui::Checkbox(label, &value);
		lua_pushboolean(L, value ? 1 : 0);
		lua_pushboolean(L, changed ? 1 : 0);
		return 2;
	}
	int l_cathack_imgui_radio_button(lua_State* L)
	{
		if (!ImRenderGuard())
		{
			lua_pushboolean(L, lua_toboolean(L, 2));
			lua_pushboolean(L, 0);
			return 2;
		}
		const char* label = luaL_checkstring(L, 1);
		const bool active = lua_toboolean(L, 2) != 0;
		const bool clicked = ImGui::RadioButton(label, active);
		lua_pushboolean(L, clicked ? 1 : (active ? 1 : 0));
		lua_pushboolean(L, clicked ? 1 : 0);
		return 2;
	}

	// ---- Sliders / drags ---------------------------------------------
	int l_cathack_imgui_slider_int(lua_State* L)
	{
		if (!ImRenderGuard())
		{
			lua_pushinteger(L, luaL_optinteger(L, 2, 0));
			lua_pushboolean(L, 0);
			return 2;
		}
		const char* label = luaL_checkstring(L, 1);
		int value = static_cast<int>(luaL_checkinteger(L, 2));
		const int lo = static_cast<int>(luaL_checkinteger(L, 3));
		const int hi = static_cast<int>(luaL_checkinteger(L, 4));
		const char* fmt = luaL_optstring(L, 5, "%d");
		const bool changed = ImGui::SliderInt(label, &value, lo, hi, fmt);
		lua_pushinteger(L, value);
		lua_pushboolean(L, changed ? 1 : 0);
		return 2;
	}
	int l_cathack_imgui_slider_float(lua_State* L)
	{
		if (!ImRenderGuard())
		{
			lua_pushnumber(L, luaL_optnumber(L, 2, 0.0));
			lua_pushboolean(L, 0);
			return 2;
		}
		const char* label = luaL_checkstring(L, 1);
		float value = static_cast<float>(luaL_checknumber(L, 2));
		const float lo = static_cast<float>(luaL_checknumber(L, 3));
		const float hi = static_cast<float>(luaL_checknumber(L, 4));
		const char* fmt = luaL_optstring(L, 5, "%.3f");
		const bool changed = ImGui::SliderFloat(label, &value, lo, hi, fmt);
		lua_pushnumber(L, value);
		lua_pushboolean(L, changed ? 1 : 0);
		return 2;
	}
	int l_cathack_imgui_drag_int(lua_State* L)
	{
		if (!ImRenderGuard())
		{
			lua_pushinteger(L, luaL_optinteger(L, 2, 0));
			lua_pushboolean(L, 0);
			return 2;
		}
		const char* label = luaL_checkstring(L, 1);
		int value = static_cast<int>(luaL_checkinteger(L, 2));
		const float speed = static_cast<float>(luaL_optnumber(L, 3, 1.0));
		const int lo = static_cast<int>(luaL_optinteger(L, 4, 0));
		const int hi = static_cast<int>(luaL_optinteger(L, 5, 0));
		const char* fmt = luaL_optstring(L, 6, "%d");
		const bool changed = ImGui::DragInt(label, &value, speed, lo, hi, fmt);
		lua_pushinteger(L, value);
		lua_pushboolean(L, changed ? 1 : 0);
		return 2;
	}
	int l_cathack_imgui_drag_float(lua_State* L)
	{
		if (!ImRenderGuard())
		{
			lua_pushnumber(L, luaL_optnumber(L, 2, 0.0));
			lua_pushboolean(L, 0);
			return 2;
		}
		const char* label = luaL_checkstring(L, 1);
		float value = static_cast<float>(luaL_checknumber(L, 2));
		const float speed = static_cast<float>(luaL_optnumber(L, 3, 1.0));
		const float lo = static_cast<float>(luaL_optnumber(L, 4, 0.0));
		const float hi = static_cast<float>(luaL_optnumber(L, 5, 0.0));
		const char* fmt = luaL_optstring(L, 6, "%.3f");
		const bool changed = ImGui::DragFloat(label, &value, speed, lo, hi, fmt);
		lua_pushnumber(L, value);
		lua_pushboolean(L, changed ? 1 : 0);
		return 2;
	}

	// ---- Color editors -----------------------------------------------
	// Color in/out is a 32-bit ABGR integer (matches the rest of the
	// cathack draw API). ImGui internally wants float[4], so we convert.
	int l_cathack_imgui_color_edit(lua_State* L)
	{
		if (!ImRenderGuard())
		{
			lua_pushinteger(L, luaL_optinteger(L, 2, 0xFFFFFFFFLL));
			lua_pushboolean(L, 0);
			return 2;
		}
		const char* label = luaL_checkstring(L, 1);
		const lua_Integer color = luaL_checkinteger(L, 2);
		const int flags = static_cast<int>(luaL_optinteger(L, 3, 0));
		float c[4] = {
			((color >>  0) & 0xFF) / 255.f,
			((color >>  8) & 0xFF) / 255.f,
			((color >> 16) & 0xFF) / 255.f,
			((color >> 24) & 0xFF) / 255.f,
		};
		const bool changed = ImGui::ColorEdit4(label, c, static_cast<ImGuiColorEditFlags>(flags));
		const lua_Integer out =
			(static_cast<lua_Integer>(c[0] * 255.f + 0.5f) <<  0) |
			(static_cast<lua_Integer>(c[1] * 255.f + 0.5f) <<  8) |
			(static_cast<lua_Integer>(c[2] * 255.f + 0.5f) << 16) |
			(static_cast<lua_Integer>(c[3] * 255.f + 0.5f) << 24);
		lua_pushinteger(L, out);
		lua_pushboolean(L, changed ? 1 : 0);
		return 2;
	}

	// ---- Combo -------------------------------------------------------
	int l_cathack_imgui_combo(lua_State* L)
	{
		if (!ImRenderGuard())
		{
			lua_pushinteger(L, luaL_optinteger(L, 2, 1));
			lua_pushboolean(L, 0);
			return 2;
		}
		const char* label = luaL_checkstring(L, 1);
		int current = static_cast<int>(luaL_checkinteger(L, 2)) - 1; // Lua is 1-indexed
		luaL_checktype(L, 3, LUA_TTABLE);

		// Read the items array.
		std::vector<std::string> items;
		const int n = static_cast<int>(lua_rawlen(L, 3));
		items.reserve(n);
		for (int i = 1; i <= n; ++i)
		{
			lua_rawgeti(L, 3, i);
			items.emplace_back(lua_isstring(L, -1) ? lua_tostring(L, -1) : "?");
			lua_pop(L, 1);
		}
		if (items.empty())
		{
			lua_pushinteger(L, 1);
			lua_pushboolean(L, 0);
			return 2;
		}
		std::vector<const char*> ptrs;
		ptrs.reserve(items.size());
		for (auto& s : items) ptrs.push_back(s.c_str());

		if (current < 0)                       current = 0;
		if (current >= static_cast<int>(items.size())) current = static_cast<int>(items.size()) - 1;

		const bool changed = ImGui::Combo(label, &current,
			ptrs.data(), static_cast<int>(ptrs.size()));

		lua_pushinteger(L, current + 1); // back to 1-indexed
		lua_pushboolean(L, changed ? 1 : 0);
		return 2;
	}

	// ---- Input text --------------------------------------------------
	// Input/output value is a Lua string. ImGui wants a writable char
	// buffer with a fixed capacity, so we copy in/out each frame. Worth
	// it for the API simplicity — scripts don't need to manage a
	// per-field char[] state.
	int l_cathack_imgui_input_text(lua_State* L)
	{
		if (!ImRenderGuard())
		{
			lua_pushvalue(L, 2);
			lua_pushboolean(L, 0);
			return 2;
		}
		const char* label = luaL_checkstring(L, 1);
		size_t len = 0;
		const char* s = luaL_optlstring(L, 2, "", &len);
		int maxLen = static_cast<int>(luaL_optinteger(L, 3, 256));
		const int flags = static_cast<int>(luaL_optinteger(L, 4, 0));

		if (maxLen < 16)     maxLen = 16;
		if (maxLen > 65536)  maxLen = 65536;

		std::vector<char> buf(static_cast<size_t>(maxLen), 0);
		const size_t copy = (len < buf.size() - 1) ? len : (buf.size() - 1);
		if (copy > 0) std::memcpy(buf.data(), s, copy);
		buf[copy] = 0;

		const bool changed = ImGui::InputText(label, buf.data(), buf.size(),
			static_cast<ImGuiInputTextFlags>(flags));

		lua_pushstring(L, buf.data());
		lua_pushboolean(L, changed ? 1 : 0);
		return 2;
	}
	int l_cathack_imgui_input_text_multiline(lua_State* L)
	{
		if (!ImRenderGuard())
		{
			lua_pushvalue(L, 2);
			lua_pushboolean(L, 0);
			return 2;
		}
		const char* label = luaL_checkstring(L, 1);
		size_t len = 0;
		const char* s = luaL_optlstring(L, 2, "", &len);
		int maxLen = static_cast<int>(luaL_optinteger(L, 3, 4096));
		const float w = static_cast<float>(luaL_optnumber(L, 4, 0.0));
		const float h = static_cast<float>(luaL_optnumber(L, 5, 0.0));
		const int flags = static_cast<int>(luaL_optinteger(L, 6, 0));

		if (maxLen < 64)        maxLen = 64;
		if (maxLen > (1 << 20)) maxLen = 1 << 20;

		std::vector<char> buf(static_cast<size_t>(maxLen), 0);
		const size_t copy = (len < buf.size() - 1) ? len : (buf.size() - 1);
		if (copy > 0) std::memcpy(buf.data(), s, copy);
		buf[copy] = 0;

		const bool changed = ImGui::InputTextMultiline(label, buf.data(), buf.size(),
			ImVec2(w, h), static_cast<ImGuiInputTextFlags>(flags));

		lua_pushstring(L, buf.data());
		lua_pushboolean(L, changed ? 1 : 0);
		return 2;
	}

	// ---- Selectable / list rows --------------------------------------
	int l_cathack_imgui_selectable(lua_State* L)
	{
		if (!ImRenderGuard())
		{
			lua_pushboolean(L, lua_toboolean(L, 2));
			lua_pushboolean(L, 0);
			return 2;
		}
		const char* label = luaL_checkstring(L, 1);
		bool selected = lua_toboolean(L, 2) != 0;
		const int flags = static_cast<int>(luaL_optinteger(L, 3, 0));
		const float w = static_cast<float>(luaL_optnumber(L, 4, 0.0));
		const float h = static_cast<float>(luaL_optnumber(L, 5, 0.0));
		const bool clicked = ImGui::Selectable(label, &selected,
			static_cast<ImGuiSelectableFlags>(flags), ImVec2(w, h));
		lua_pushboolean(L, selected ? 1 : 0);
		lua_pushboolean(L, clicked ? 1 : 0);
		return 2;
	}

	// ---- Headers / trees ---------------------------------------------
	int l_cathack_imgui_collapsing_header(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const char* label = luaL_checkstring(L, 1);
		const int flags = static_cast<int>(luaL_optinteger(L, 2, 0));
		// CollapsingHeader has no End — bool result is enough.
		lua_pushboolean(L,
			ImGui::CollapsingHeader(label, static_cast<ImGuiTreeNodeFlags>(flags)) ? 1 : 0);
		return 1;
	}
	int l_cathack_imgui_tree_node(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const char* label = luaL_checkstring(L, 1);
		const int flags = static_cast<int>(luaL_optinteger(L, 2, 0));
		const bool open = ImGui::TreeNodeEx(label, static_cast<ImGuiTreeNodeFlags>(flags));
		if (open) ++g_imTreeDepth;
		lua_pushboolean(L, open ? 1 : 0);
		return 1;
	}
	int l_cathack_imgui_tree_pop(lua_State* L)
	{
		(void)L;
		if (!ImRenderGuard() || g_imTreeDepth <= 0) return 0;
		ImGui::TreePop();
		--g_imTreeDepth;
		return 0;
	}

	// ---- Tab bars ----------------------------------------------------
	int l_cathack_imgui_begin_tab_bar(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const char* id = luaL_checkstring(L, 1);
		const int flags = static_cast<int>(luaL_optinteger(L, 2, 0));
		const bool open = ImGui::BeginTabBar(id, static_cast<ImGuiTabBarFlags>(flags));
		if (open) ++g_imTabBarDepth;
		lua_pushboolean(L, open ? 1 : 0);
		return 1;
	}
	int l_cathack_imgui_end_tab_bar(lua_State* L)
	{
		(void)L;
		if (!ImRenderGuard() || g_imTabBarDepth <= 0) return 0;
		ImGui::EndTabBar();
		--g_imTabBarDepth;
		return 0;
	}
	int l_cathack_imgui_begin_tab_item(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const char* label = luaL_checkstring(L, 1);
		const int flags = static_cast<int>(luaL_optinteger(L, 2, 0));
		const bool open = ImGui::BeginTabItem(label, nullptr, static_cast<ImGuiTabItemFlags>(flags));
		if (open) ++g_imTabItemDepth;
		lua_pushboolean(L, open ? 1 : 0);
		return 1;
	}
	int l_cathack_imgui_end_tab_item(lua_State* L)
	{
		(void)L;
		if (!ImRenderGuard() || g_imTabItemDepth <= 0) return 0;
		ImGui::EndTabItem();
		--g_imTabItemDepth;
		return 0;
	}

	// ---- Tables ------------------------------------------------------
	int l_cathack_imgui_begin_table(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const char* id = luaL_checkstring(L, 1);
		const int columns = static_cast<int>(luaL_checkinteger(L, 2));
		const int flags = static_cast<int>(luaL_optinteger(L, 3, 0));
		const float outerW = static_cast<float>(luaL_optnumber(L, 4, 0.0));
		const float outerH = static_cast<float>(luaL_optnumber(L, 5, 0.0));
		const bool open = ImGui::BeginTable(id, columns,
			static_cast<ImGuiTableFlags>(flags),
			ImVec2(outerW, outerH));
		if (open) ++g_imTableDepth;
		lua_pushboolean(L, open ? 1 : 0);
		return 1;
	}
	int l_cathack_imgui_end_table(lua_State* L)
	{
		(void)L;
		if (!ImRenderGuard() || g_imTableDepth <= 0) return 0;
		ImGui::EndTable();
		--g_imTableDepth;
		return 0;
	}
	int l_cathack_imgui_table_setup_column(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		const char* label = luaL_checkstring(L, 1);
		const int flags = static_cast<int>(luaL_optinteger(L, 2, 0));
		const float widthOrWeight = static_cast<float>(luaL_optnumber(L, 3, 0.0));
		ImGui::TableSetupColumn(label,
			static_cast<ImGuiTableColumnFlags>(flags),
			widthOrWeight);
		return 0;
	}
	int l_cathack_imgui_table_headers_row(lua_State* L)   { (void)L; if (ImRenderGuard()) ImGui::TableHeadersRow(); return 0; }
	int l_cathack_imgui_table_next_row(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		const int rowFlags = static_cast<int>(luaL_optinteger(L, 1, 0));
		const float minRowHeight = static_cast<float>(luaL_optnumber(L, 2, 0.0));
		ImGui::TableNextRow(static_cast<ImGuiTableRowFlags>(rowFlags), minRowHeight);
		return 0;
	}
	int l_cathack_imgui_table_next_column(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		lua_pushboolean(L, ImGui::TableNextColumn() ? 1 : 0);
		return 1;
	}
	int l_cathack_imgui_table_set_column_index(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const int idx = static_cast<int>(luaL_checkinteger(L, 1));
		lua_pushboolean(L, ImGui::TableSetColumnIndex(idx) ? 1 : 0);
		return 1;
	}

	// ---- Tooltips ----------------------------------------------------
	int l_cathack_imgui_set_tooltip(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		ImGui::SetTooltip("%s", luaL_checkstring(L, 1));
		return 0;
	}
	int l_cathack_imgui_set_item_tooltip(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		// SetItemTooltip is a 1.90+ shorthand; if not available we
		// fall back to the standard "hovered + delay → SetTooltip" pattern.
#if IMGUI_VERSION_NUM >= 19000
		ImGui::SetItemTooltip("%s", luaL_checkstring(L, 1));
#else
		if (ImGui::IsItemHovered(ImGuiHoveredFlags_DelayShort | ImGuiHoveredFlags_NoSharedDelay))
			ImGui::SetTooltip("%s", luaL_checkstring(L, 1));
		else
			(void)luaL_checkstring(L, 1);
#endif
		return 0;
	}
	int l_cathack_imgui_begin_tooltip(lua_State* L)
	{
		(void)L;
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const bool open = ImGui::BeginTooltip();
		if (open) ++g_imTooltipDepth;
		lua_pushboolean(L, open ? 1 : 0);
		return 1;
	}
	int l_cathack_imgui_end_tooltip(lua_State* L)
	{
		(void)L;
		if (!ImRenderGuard() || g_imTooltipDepth <= 0) return 0;
		ImGui::EndTooltip();
		--g_imTooltipDepth;
		return 0;
	}

	// ---- Item state queries ------------------------------------------
	int l_cathack_imgui_is_item_hovered(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const int flags = static_cast<int>(luaL_optinteger(L, 1, 0));
		lua_pushboolean(L, ImGui::IsItemHovered(static_cast<ImGuiHoveredFlags>(flags)) ? 1 : 0);
		return 1;
	}
	int l_cathack_imgui_is_item_clicked(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const int btn = static_cast<int>(luaL_optinteger(L, 1, 0));
		lua_pushboolean(L, ImGui::IsItemClicked(btn) ? 1 : 0);
		return 1;
	}
	int l_cathack_imgui_is_item_active(lua_State* L)    { (void)L; lua_pushboolean(L, (ImRenderGuard() && ImGui::IsItemActive())   ? 1 : 0); return 1; }
	int l_cathack_imgui_is_item_focused(lua_State* L)   { (void)L; lua_pushboolean(L, (ImRenderGuard() && ImGui::IsItemFocused())  ? 1 : 0); return 1; }
	int l_cathack_imgui_is_window_hovered(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushboolean(L, 0); return 1; }
		const int flags = static_cast<int>(luaL_optinteger(L, 1, 0));
		lua_pushboolean(L, ImGui::IsWindowHovered(static_cast<ImGuiHoveredFlags>(flags)) ? 1 : 0);
		return 1;
	}
	int l_cathack_imgui_get_cursor_pos(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushnumber(L, 0); lua_pushnumber(L, 0); return 2; }
		const ImVec2 p = ImGui::GetCursorPos();
		lua_pushnumber(L, p.x);
		lua_pushnumber(L, p.y);
		return 2;
	}
	int l_cathack_imgui_set_cursor_pos(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		const float x = static_cast<float>(luaL_checknumber(L, 1));
		const float y = static_cast<float>(luaL_checknumber(L, 2));
		ImGui::SetCursorPos(ImVec2(x, y));
		return 0;
	}
	int l_cathack_imgui_get_content_region_avail(lua_State* L)
	{
		if (!ImRenderGuard()) { lua_pushnumber(L, 0); lua_pushnumber(L, 0); return 2; }
		const ImVec2 p = ImGui::GetContentRegionAvail();
		lua_pushnumber(L, p.x);
		lua_pushnumber(L, p.y);
		return 2;
	}

	// ---- Style push/pop ----------------------------------------------
	int l_cathack_imgui_push_style_color(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		const int idx = static_cast<int>(luaL_checkinteger(L, 1));
		if (idx < 0 || idx >= ImGuiCol_COUNT)
			return luaL_error(L, "imgui_push_style_color: invalid color index %d", idx);
		const lua_Integer color = luaL_checkinteger(L, 2);
		ImGui::PushStyleColor(static_cast<ImGuiCol>(idx), static_cast<ImU32>(color));
		++g_imStyleColorDepth;
		return 0;
	}
	int l_cathack_imgui_pop_style_color(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		int count = static_cast<int>(luaL_optinteger(L, 1, 1));
		if (count <= 0)                       return 0;
		if (count > g_imStyleColorDepth)      count = g_imStyleColorDepth;
		if (count > 0)
		{
			ImGui::PopStyleColor(count);
			g_imStyleColorDepth -= count;
		}
		return 0;
	}
	int l_cathack_imgui_push_style_var(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		const int idx = static_cast<int>(luaL_checkinteger(L, 1));
		if (idx < 0 || idx >= ImGuiStyleVar_COUNT)
			return luaL_error(L, "imgui_push_style_var: invalid var index %d", idx);
		// Two-arg → ImVec2; one-arg → float. Most ImGui style vars are
		// floats; the ImVec2 variants (WindowPadding, FramePadding, etc.)
		// take two numbers.
		if (lua_isnumber(L, 3))
		{
			const float x = static_cast<float>(luaL_checknumber(L, 2));
			const float y = static_cast<float>(luaL_checknumber(L, 3));
			ImGui::PushStyleVar(static_cast<ImGuiStyleVar>(idx), ImVec2(x, y));
		}
		else
		{
			const float v = static_cast<float>(luaL_checknumber(L, 2));
			ImGui::PushStyleVar(static_cast<ImGuiStyleVar>(idx), v);
		}
		++g_imStyleVarDepth;
		return 0;
	}
	int l_cathack_imgui_pop_style_var(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		int count = static_cast<int>(luaL_optinteger(L, 1, 1));
		if (count <= 0)                     return 0;
		if (count > g_imStyleVarDepth)      count = g_imStyleVarDepth;
		if (count > 0)
		{
			ImGui::PopStyleVar(count);
			g_imStyleVarDepth -= count;
		}
		return 0;
	}

	// ---- ID stack ----------------------------------------------------
	int l_cathack_imgui_push_id(lua_State* L)
	{
		if (!ImRenderGuard()) return 0;
		if (lua_isnumber(L, 1))
			ImGui::PushID(static_cast<int>(lua_tointeger(L, 1)));
		else
			ImGui::PushID(luaL_checkstring(L, 1));
		++g_imIDDepth;
		return 0;
	}
	int l_cathack_imgui_pop_id(lua_State* L)
	{
		(void)L;
		if (!ImRenderGuard() || g_imIDDepth <= 0) return 0;
		ImGui::PopID();
		--g_imIDDepth;
		return 0;
	}

	// =====================================================================
	// HTTP — `cathack.http_get` and Roblox-style `game:HttpGet`.
	//
	// Backend is WinHTTP, opened once with HTTP/2 + TLS 1.2/1.3 enabled and
	// the session handle cached process-wide so subsequent calls don't
	// re-cost the TLS handshake. Per-host connections are also cached for
	// the duration of the DLL session — that's where the "fast" comes from
	// when a script HttpGet's the same host (e.g. raw.githubusercontent.com)
	// repeatedly.
	//
	// Two execution modes:
	//   - Synchronous: cathack.http_get(url[, opts]) and game:HttpGet(url).
	//     Blocks the calling thread (game or render) until the body comes
	//     back. Default 5 s timeout. Cached results return immediately.
	//   - Async: cathack.http_get_async(url, callback[, opts]). Spawns a
	//     detached worker, returns instantly. The callback fires in
	//     `TickRender` once the result lands; signature is (body, err).
	//
	// Cache: in-memory only. Default TTL 5 minutes; pass `no_cache=true`
	// in opts to bypass. cathack.http_clear_cache() drops everything.
	// =====================================================================
	struct HttpCachedResponse
	{
		std::string body;
		std::chrono::steady_clock::time_point fetchedAt;
		bool        ok = false;
	};
	struct HttpConn
	{
		HINTERNET handle = nullptr;
		std::wstring host;
		INTERNET_PORT port = 0;
		bool https = false;
	};

	std::mutex                                  g_httpMu;
	std::unordered_map<std::string, HttpCachedResponse> g_httpCache;
	HINTERNET                                   g_httpSession = nullptr;
	std::vector<HttpConn>                       g_httpConnPool;

	// Async job queue + result drain.
	struct HttpAsyncJob
	{
		std::string url;
		int  callbackRef = LUA_NOREF;
		std::string ownerScript;
		int  timeoutMs = 5000;
		std::atomic<bool> done{false};
		bool ok = false;
		std::string body;
		std::string err;
	};
	std::mutex                                          g_httpJobsMu;
	std::vector<std::shared_ptr<HttpAsyncJob>>          g_httpJobs;

	static void HttpInitSession_locked()
	{
		if (g_httpSession) return;
		g_httpSession = WinHttpOpen(
			L"cathack/1.0",
			WINHTTP_ACCESS_TYPE_AUTOMATIC_PROXY,
			WINHTTP_NO_PROXY_NAME,
			WINHTTP_NO_PROXY_BYPASS,
			0);
		if (!g_httpSession) return;

		// 5 s connect, 5 s send, 15 s receive (the receive cap is what most
		// "stuck" HTTP calls hit; tighter than that breaks slow CDNs).
		WinHttpSetTimeouts(g_httpSession, 5000, 5000, 15000, 15000);

		// Force modern TLS only. WinHTTP defaults to TLS 1.0 + SSL3 on
		// older Windows builds, both of which most servers reject.
		DWORD secureProtocols = 0;
#ifdef WINHTTP_FLAG_SECURE_PROTOCOL_TLS1_2
		secureProtocols |= WINHTTP_FLAG_SECURE_PROTOCOL_TLS1_2;
#endif
#ifdef WINHTTP_FLAG_SECURE_PROTOCOL_TLS1_3
		secureProtocols |= WINHTTP_FLAG_SECURE_PROTOCOL_TLS1_3;
#endif
		if (secureProtocols)
			WinHttpSetOption(g_httpSession, WINHTTP_OPTION_SECURE_PROTOCOLS,
				&secureProtocols, sizeof(secureProtocols));

		// HTTP/2 multiplexing — saves a couple of round-trips on every
		// hot host. Available on Windows 10 1607+; the option just no-ops
		// on older builds.
#ifdef WINHTTP_PROTOCOL_FLAG_HTTP2
		DWORD enableHttp2 = WINHTTP_PROTOCOL_FLAG_HTTP2;
		WinHttpSetOption(g_httpSession, WINHTTP_OPTION_ENABLE_HTTP_PROTOCOL,
			&enableHttp2, sizeof(enableHttp2));
#endif
	}

	// Look up a cached connection for (host, port, https) or open a new
	// one and remember it. Connection pooling is cheap and gives back the
	// TCP RTT on every repeat call to the same host.
	static HINTERNET HttpGetOrOpenConnection_locked(const std::wstring& host, INTERNET_PORT port, bool https)
	{
		for (const auto& c : g_httpConnPool)
			if (c.host == host && c.port == port && c.https == https && c.handle)
				return c.handle;
		HINTERNET h = WinHttpConnect(g_httpSession, host.c_str(), port, 0);
		if (h) g_httpConnPool.push_back({ h, host, port, https });
		return h;
	}

	static bool HttpParseUrl(const std::string& url,
		bool& outHttps, std::wstring& outHost, INTERNET_PORT& outPort, std::wstring& outPath)
	{
		// Widen byte-by-byte. URLs are ASCII for any sane case; this would
		// truncate weird Punycode/IDN inputs, but that's a non-goal here.
		std::wstring wurl;
		wurl.reserve(url.size());
		for (char c : url)
			wurl.push_back(static_cast<wchar_t>(static_cast<unsigned char>(c)));

		URL_COMPONENTSW uc{};
		uc.dwStructSize = sizeof(uc);
		// Pass capacity in dwHostNameLength / dwUrlPathLength fields by
		// supplying *non-zero* lengths and matching buffer pointers — see
		// the MSDN remarks on WinHttpCrackUrl.
		wchar_t hostBuf[512] = {};
		wchar_t pathBuf[2048] = {};
		uc.lpszHostName = hostBuf;     uc.dwHostNameLength = ARRAYSIZE(hostBuf);
		uc.lpszUrlPath  = pathBuf;     uc.dwUrlPathLength  = ARRAYSIZE(pathBuf);
		if (!WinHttpCrackUrl(wurl.c_str(), 0, 0, &uc)) return false;

		outHttps = uc.nScheme == INTERNET_SCHEME_HTTPS;
		outHost.assign(uc.lpszHostName, uc.dwHostNameLength);
		outPort  = uc.nPort;
		// Copy path + query together — pathBuf only has the path; the
		// query lives at lpszExtraInfo. Easier to just slice the original
		// wide URL after the host:port portion.
		outPath.assign(uc.lpszUrlPath, uc.dwUrlPathLength);
		if (uc.lpszExtraInfo && uc.dwExtraInfoLength)
			outPath.append(uc.lpszExtraInfo, uc.dwExtraInfoLength);
		if (outPath.empty()) outPath = L"/";
		return true;
	}

	// Issue one synchronous GET. Returns true + body on success, false +
	// error string on failure. `timeoutMs` overrides the session-default
	// receive timeout for this single request.
	static bool HttpDoGet(const std::string& url, std::string& outBody, std::string& outErr, int timeoutMs)
	{
		std::lock_guard<std::mutex> lk(g_httpMu);
		HttpInitSession_locked();
		if (!g_httpSession) { outErr = "winhttp session init failed"; return false; }

		bool https; std::wstring host, path; INTERNET_PORT port;
		if (!HttpParseUrl(url, https, host, port, path))
		{
			outErr = "bad URL";
			return false;
		}

		HINTERNET conn = HttpGetOrOpenConnection_locked(host, port, https);
		if (!conn) { outErr = "WinHttpConnect failed"; return false; }

		HINTERNET req = WinHttpOpenRequest(
			conn, L"GET", path.c_str(), nullptr,
			WINHTTP_NO_REFERER,
			WINHTTP_DEFAULT_ACCEPT_TYPES,
			https ? WINHTTP_FLAG_SECURE : 0u);
		if (!req) { outErr = "WinHttpOpenRequest failed"; return false; }

		if (timeoutMs > 0)
			WinHttpSetTimeouts(req, 5000, 5000, timeoutMs, timeoutMs);

		// Some servers (notably Cloudflare with strict bot rules) require
		// a User-Agent. WinHttpOpen sets one but it gets dropped per
		// request — re-add explicitly.
		const wchar_t* ua = L"User-Agent: cathack/1.0\r\n";
		WinHttpAddRequestHeaders(req, ua, static_cast<DWORD>(-1L), WINHTTP_ADDREQ_FLAG_ADD);

		BOOL ok = WinHttpSendRequest(req,
			WINHTTP_NO_ADDITIONAL_HEADERS, 0,
			WINHTTP_NO_REQUEST_DATA, 0, 0, 0);
		if (!ok)
		{
			outErr = "WinHttpSendRequest failed (err " + std::to_string(GetLastError()) + ")";
			WinHttpCloseHandle(req);
			return false;
		}

		ok = WinHttpReceiveResponse(req, nullptr);
		if (!ok)
		{
			outErr = "WinHttpReceiveResponse failed (err " + std::to_string(GetLastError()) + ")";
			WinHttpCloseHandle(req);
			return false;
		}

		// Surface 4xx / 5xx as errors. Returning a non-2xx body silently
		// would let scripts loadstring() an HTML 404 page and produce
		// confusing parser errors.
		DWORD statusCode = 0;
		DWORD scLen = sizeof(statusCode);
		WinHttpQueryHeaders(req,
			WINHTTP_QUERY_STATUS_CODE | WINHTTP_QUERY_FLAG_NUMBER,
			WINHTTP_HEADER_NAME_BY_INDEX, &statusCode, &scLen,
			WINHTTP_NO_HEADER_INDEX);

		outBody.clear();
		outBody.reserve(8192);
		DWORD bytesAvail = 0;
		do
		{
			if (!WinHttpQueryDataAvailable(req, &bytesAvail))
			{
				outErr = "WinHttpQueryDataAvailable failed";
				WinHttpCloseHandle(req);
				return false;
			}
			if (bytesAvail == 0) break;
			std::vector<char> buf(bytesAvail);
			DWORD bytesRead = 0;
			if (!WinHttpReadData(req, buf.data(), bytesAvail, &bytesRead))
			{
				outErr = "WinHttpReadData failed";
				WinHttpCloseHandle(req);
				return false;
			}
			if (bytesRead == 0) break;
			outBody.append(buf.data(), bytesRead);
			// Hard cap at 16 MB — protects against runaway streams.
			if (outBody.size() > (16u * 1024u * 1024u))
			{
				outErr = "response body exceeded 16 MB limit";
				WinHttpCloseHandle(req);
				return false;
			}
		} while (bytesAvail > 0);

		WinHttpCloseHandle(req);

		if (statusCode >= 400)
		{
			outErr = "HTTP " + std::to_string(statusCode);
			return false;
		}
		return true;
	}

	// Read the optional opts table sitting at `optIdx` in the Lua stack.
	// Recognized fields: timeout_ms (number), no_cache (bool).
	static void HttpReadOpts(lua_State* L, int optIdx, int& timeoutMs, bool& noCache)
	{
		if (lua_type(L, optIdx) != LUA_TTABLE) return;
		lua_getfield(L, optIdx, "timeout_ms");
		if (lua_type(L, -1) == LUA_TNUMBER) timeoutMs = static_cast<int>(lua_tointeger(L, -1));
		lua_pop(L, 1);
		lua_getfield(L, optIdx, "no_cache");
		if (lua_type(L, -1) == LUA_TBOOLEAN) noCache = lua_toboolean(L, -1) != 0;
		lua_pop(L, 1);
	}

	// Implementation shared by cathack.http_get and game:HttpGet.
	static int HttpGetSyncImpl(lua_State* L, int urlIdx, int optsIdx)
	{
		const char* url = luaL_checkstring(L, urlIdx);
		int timeoutMs = 5000;
		bool noCache = false;
		HttpReadOpts(L, optsIdx, timeoutMs, noCache);

		// Cache lookup. 5-minute TTL. Negative entries (failures) aren't
		// cached so a transient outage doesn't poison the slot.
		if (!noCache)
		{
			std::lock_guard<std::mutex> lk(g_httpMu);
			auto it = g_httpCache.find(url);
			if (it != g_httpCache.end() && it->second.ok)
			{
				const auto age = std::chrono::steady_clock::now() - it->second.fetchedAt;
				if (age < std::chrono::minutes(5))
				{
					lua_pushlstring(L, it->second.body.data(), it->second.body.size());
					return 1;
				}
			}
		}

		std::string body, err;
		const bool ok = HttpDoGet(url, body, err, timeoutMs);
		if (ok)
		{
			std::lock_guard<std::mutex> lk(g_httpMu);
			g_httpCache[url] = { body, std::chrono::steady_clock::now(), true };
			lua_pushlstring(L, body.data(), body.size());
			return 1;
		}
		lua_pushnil(L);
		lua_pushstring(L, err.c_str());
		return 2;
	}

	// cathack.http_get(url[, opts])  ->  body | nil, err
	int l_cathack_http_get(lua_State* L)
	{
		return HttpGetSyncImpl(L, 1, 2);
	}

	// game:HttpGet(url) — Roblox-style. Roblox raises on failure; mirror
	// that by lua_error'ing on a non-2xx so `local s = game:HttpGet(...)`
	// behaves correctly when callers don't bother checking.
	int l_game_http_get(lua_State* L)
	{
		// Method form passes self as arg 1; bare-call form skips it.
		// Distinguish by type at index 1.
		const int urlIdx = (lua_type(L, 1) == LUA_TSTRING) ? 1 : 2;
		const int optsIdx = urlIdx + 1;
		const int rets = HttpGetSyncImpl(L, urlIdx, optsIdx);
		if (rets == 1) return 1;
		// Two returns means (nil, err). Convert to a Lua error to match
		// Roblox semantics.
		const char* err = lua_tostring(L, -1);
		return luaL_error(L, "HttpGet failed: %s", err ? err : "?");
	}

	// cathack.http_get_async(url, callback[, opts])
	//
	// Spawns a detached worker that runs the GET. Callback fires inside
	// the next TickRender once the worker returns. Callback signature:
	//   function(body_or_nil, err_or_nil) end
	int l_cathack_http_get_async(lua_State* L)
	{
		const char* url = luaL_checkstring(L, 1);
		luaL_checktype(L, 2, LUA_TFUNCTION);

		int timeoutMs = 5000;
		bool noCache = false;
		HttpReadOpts(L, 3, timeoutMs, noCache);

		auto job = std::make_shared<HttpAsyncJob>();
		job->url        = url;
		job->timeoutMs  = timeoutMs;
		job->ownerScript = g_runningScript;
		// Pin the callback in the registry so it survives until we pop it.
		lua_pushvalue(L, 2);
		job->callbackRef = luaL_ref(L, LUA_REGISTRYINDEX);

		// Cache hit? Mark the job done immediately so TickRender fires
		// the callback on the next frame without spinning a thread.
		if (!noCache)
		{
			std::lock_guard<std::mutex> lk(g_httpMu);
			auto it = g_httpCache.find(url);
			if (it != g_httpCache.end() && it->second.ok
				&& std::chrono::steady_clock::now() - it->second.fetchedAt < std::chrono::minutes(5))
			{
				job->ok = true;
				job->body = it->second.body;
				job->done.store(true);
				{
					std::lock_guard<std::mutex> jlk(g_httpJobsMu);
					g_httpJobs.push_back(job);
				}
				lua_pushboolean(L, 1);
				return 1;
			}
		}

		{
			std::lock_guard<std::mutex> jlk(g_httpJobsMu);
			g_httpJobs.push_back(job);
		}

		std::thread([job, noCache]()
		{
			std::string body, err;
			const bool ok = HttpDoGet(job->url, body, err, job->timeoutMs);
			job->ok   = ok;
			job->body = std::move(body);
			job->err  = std::move(err);
			if (ok && !noCache)
			{
				std::lock_guard<std::mutex> lk(g_httpMu);
				g_httpCache[job->url] = { job->body, std::chrono::steady_clock::now(), true };
			}
			job->done.store(true);
		}).detach();

		lua_pushboolean(L, 1);
		return 1;
	}

	// cathack.http_clear_cache() — drops everything cached. Connection
	// pool is left intact (dropping it would just re-cost the TLS
	// handshake on the next call).
	int l_cathack_http_clear_cache(lua_State* L)
	{
		(void)L;
		std::lock_guard<std::mutex> lk(g_httpMu);
		g_httpCache.clear();
		return 0;
	}

	// Drain finished async jobs and fire their Lua callbacks. Called from
	// TickRender — that's the only place we know the Lua mutex is held
	// AND we have a stable lua_State pointer to push args onto.
	void HttpDrainCompletedJobs_locked(lua_State* L)
	{
		std::vector<std::shared_ptr<HttpAsyncJob>> ready;
		{
			std::lock_guard<std::mutex> lk(g_httpJobsMu);
			for (auto it = g_httpJobs.begin(); it != g_httpJobs.end(); )
			{
				if ((*it)->done.load())
				{
					ready.push_back(*it);
					it = g_httpJobs.erase(it);
				}
				else ++it;
			}
		}
		for (auto& job : ready)
		{
			lua_rawgeti(L, LUA_REGISTRYINDEX, job->callbackRef);
			if (job->ok)
			{
				lua_pushlstring(L, job->body.data(), job->body.size());
				lua_pushnil(L);
			}
			else
			{
				lua_pushnil(L);
				lua_pushstring(L, job->err.c_str());
			}
			if (lua_pcall(L, 2, 0, 0) != LUA_OK)
			{
				const char* e = lua_tostring(L, -1);
				PushConsole(std::string("[http_get_async callback] ") + (e ? e : "?"));
				lua_pop(L, 1);
			}
			luaL_unref(L, LUA_REGISTRYINDEX, job->callbackRef);
		}
	}

	// Registry table layout. Add new functions here.
	const luaL_Reg kCathackFuncs[] = {
		// Console / notifications
		{"print",                 l_cathack_print},
		{"notify",                l_cathack_notify},

		// Settings
		{"set",                   l_cathack_set},
		{"get",                   l_cathack_get},
		{"settings",              l_cathack_settings},
		{"bindings",              l_cathack_bindings},

		// Local player (basic)
		{"player_pos",            l_cathack_player_pos},
		{"player_health",         l_cathack_player_health},
		{"is_alive",              l_cathack_is_alive},
		// Local player (extras)
		{"player_velocity",       l_cathack_player_velocity},
		{"player_speed",          l_cathack_player_speed},
		{"player_aim_rot",        l_cathack_player_aim_rot},
		{"player_team",           l_cathack_player_team},
		{"player_name",           l_cathack_player_name},
		{"local_id",              l_cathack_local_id},

		// Camera — reading
		{"camera_pos",                l_cathack_camera_pos},
		{"camera_rot",                l_cathack_camera_rot},
		{"camera_forward",            l_cathack_camera_forward},
		{"camera_right",              l_cathack_camera_right},
		{"camera_up",                 l_cathack_camera_up},
		{"camera_fov",                l_cathack_camera_fov},
		{"camera_aspect",             l_cathack_camera_aspect},
		{"camera_near_clip",          l_cathack_camera_near_clip},
		{"camera_velocity",           l_cathack_camera_velocity},
		{"camera_speed",              l_cathack_camera_speed},
		{"camera_view_target",        l_cathack_camera_view_target},
		{"camera_post_process_blend", l_cathack_camera_post_process_blend},
		// Camera — coordinate transforms
		{"screen_to_world_ray",       l_cathack_screen_to_world_ray},
		{"world_to_camera_relative",  l_cathack_world_to_camera_relative},
		{"camera_to_world",           l_cathack_camera_to_world},
		// Camera — overrides (per-frame reapplied)
		{"set_camera_pos",            l_cathack_set_camera_pos},
		{"set_camera_rot",            l_cathack_set_camera_rot},
		{"set_camera_fov",            l_cathack_set_camera_fov},
		{"clear_camera_pos",          l_cathack_clear_camera_pos},
		{"clear_camera_rot",          l_cathack_clear_camera_rot},
		{"clear_camera_fov",          l_cathack_clear_camera_fov},
		{"clear_camera_overrides",    l_cathack_clear_camera_overrides},
		// Camera — view target / spectate
		{"set_view_target",           l_cathack_set_view_target},
		// Camera — effects
		{"camera_fade",               l_cathack_camera_fade},
		{"camera_fade_stop",          l_cathack_camera_fade_stop},
		{"camera_set_manual_fade",    l_cathack_camera_set_manual_fade},
		{"camera_stop_shakes",        l_cathack_camera_stop_shakes},

		// Screen / projection
		{"screen_size",           l_cathack_screen_size},
		{"world_to_screen",       l_cathack_world_to_screen},

		// Input / time
		{"key",                   l_cathack_key},
		{"time",                  l_cathack_time},
		{"delta_time",            l_cathack_delta_time},
		{"fps",                   l_cathack_fps},

		// Engine info
		{"is_in_game",            l_cathack_is_in_game},
		{"world_name",            l_cathack_world_name},

		// Math / vectors
		{"distance",              l_cathack_distance},
		{"normalize",             l_cathack_normalize},
		{"dot",                   l_cathack_dot},
		{"cross",                 l_cathack_cross},
		{"lerp",                  l_cathack_lerp},

		// Color helpers
		{"color",                 l_cathack_color},
		{"color_hsv",             l_cathack_color_hsv},

		// Entities
		{"entities",              l_cathack_entities},
		{"entity_count",          l_cathack_entity_count},
		{"entity_pos",            l_cathack_entity_pos},
		{"entity_velocity",       l_cathack_entity_velocity},
		{"entity_speed",          l_cathack_entity_speed},
		{"entity_rotation",       l_cathack_entity_rotation},
		{"entity_aim_rot",        l_cathack_entity_aim_rot},
		{"entity_health",         l_cathack_entity_health},
		{"entity_kind",           l_cathack_entity_kind},
		{"entity_alive",          l_cathack_entity_alive},
		{"entity_distance",       l_cathack_entity_distance},
		{"entity_name",           l_cathack_entity_name},
		{"entity_bone",           l_cathack_entity_bone},
		{"entity_screen_pos",     l_cathack_entity_screen_pos},
		{"entity_visible",        l_cathack_entity_visible},

		// Raycast / LOS
		{"line_of_sight",         l_cathack_line_of_sight},
		{"line_trace",            l_cathack_line_trace},

		// Drawing — basics
		{"draw_text",             l_cathack_draw_text},
		{"draw_rect",             l_cathack_draw_rect},
		{"draw_rect_filled",      l_cathack_draw_rect_filled},
		{"draw_line",             l_cathack_draw_line},
		{"draw_circle",           l_cathack_draw_circle},
		{"draw_circle_filled",    l_cathack_draw_circle_filled},
		// Drawing — extras
		{"draw_triangle",         l_cathack_draw_triangle},
		{"draw_triangle_filled",  l_cathack_draw_triangle_filled},
		{"draw_quad",             l_cathack_draw_quad},
		{"draw_quad_filled",      l_cathack_draw_quad_filled},
		{"draw_polyline",         l_cathack_draw_polyline},
		{"draw_bezier",           l_cathack_draw_bezier},
		{"measure_text",          l_cathack_measure_text},
		{"world_text",            l_cathack_world_text},

		// Persistent storage
		{"store",                 l_cathack_store},
		{"fetch",                 l_cathack_fetch},
		{"store_keys",            l_cathack_store_keys},

		// Filesystem (sandboxed)
		{"script_dir",            l_cathack_script_dir},
		{"read_file",             l_cathack_read_file},
		{"write_file",            l_cathack_write_file},

		// Dynamic execution
		{"run_string",            l_cathack_run_string},

		// Callbacks
		{"on_tick",               l_cathack_on_tick},
		{"on_render",             l_cathack_on_render},
		{"untick",                l_cathack_untick},
		{"unrender",              l_cathack_unrender},
		{"unhook",                l_cathack_unhook},
		{"clear_callbacks",       l_cathack_clear_callbacks},

		// Mouse polling
		{"mouse_pos",             l_cathack_mouse_pos},
		{"mouse_delta",           l_cathack_mouse_delta},
		{"mouse_down",            l_cathack_mouse_down},
		{"mouse_clicked",         l_cathack_mouse_clicked},
		{"mouse_released",        l_cathack_mouse_released},
		{"mouse_double_clicked",  l_cathack_mouse_double_clicked},
		{"mouse_wheel",           l_cathack_mouse_wheel},
		{"mouse_drag_delta",      l_cathack_mouse_drag_delta},
		{"mouse_in_rect",         l_cathack_mouse_in_rect},
		// Cursor + capture
		{"set_cursor_visible",    l_cathack_set_cursor_visible},
		{"cursor_visible",        l_cathack_cursor_visible},
		{"capture_mouse",         l_cathack_capture_mouse},
		{"capture_keyboard",      l_cathack_capture_keyboard},
		{"want_capture_mouse",    l_cathack_want_capture_mouse},
		{"want_capture_keyboard", l_cathack_want_capture_keyboard},
		// Cheat menu state
		{"menu_open",             l_cathack_menu_open},
		{"set_menu_open",         l_cathack_set_menu_open},
		{"toggle_menu",           l_cathack_toggle_menu},
		// Keyboard / char input
		{"key_pressed",           l_cathack_key_pressed},
		{"key_released",          l_cathack_key_released},
		{"chars_typed",           l_cathack_chars_typed},
		// Clip rects + draw layers
		{"push_clip_rect",        l_cathack_push_clip_rect},
		{"pop_clip_rect",         l_cathack_pop_clip_rect},
		{"set_draw_layer",        l_cathack_set_draw_layer},
		// Better text
		{"draw_text_sized",       l_cathack_draw_text_sized},
		{"draw_text_centered",    l_cathack_draw_text_centered},
		{"draw_text_right",       l_cathack_draw_text_right},
		{"measure_text_sized",    l_cathack_measure_text_sized},
		// Fancy primitives
		{"draw_rect_gradient",    l_cathack_draw_rect_gradient},
		{"draw_shadow_rect",      l_cathack_draw_shadow_rect},
		{"draw_glow_circle",      l_cathack_draw_glow_circle},
		// Textures + images
		{"load_texture",          l_cathack_load_texture},
		{"free_texture",          l_cathack_free_texture},
		{"texture_size",          l_cathack_texture_size},
		{"draw_image",            l_cathack_draw_image},
		{"draw_image_rounded",    l_cathack_draw_image_rounded},
		// Clipboard
		{"clipboard_get",         l_cathack_clipboard_get},
		{"clipboard_set",         l_cathack_clipboard_set},
		// Theme + animation
		{"accent_color",          l_cathack_accent_color},
		{"theme",                 l_cathack_theme},
		{"lerp_color",            l_cathack_lerp_color},
		{"ease",                  l_cathack_ease},
		{"frame_count",           l_cathack_frame_count},
		// Viewport
		{"viewport_pos",          l_cathack_viewport_pos},
		{"viewport_size",         l_cathack_viewport_size},
		{"dpi_scale",             l_cathack_dpi_scale},
		// JSON
		{"json_encode",           l_cathack_json_encode},
		{"json_decode",           l_cathack_json_decode},
		// Event callbacks
		{"on_key_pressed",        l_cathack_on_key_pressed},
		{"on_key_released",       l_cathack_on_key_released},
		{"on_mouse_pressed",      l_cathack_on_mouse_pressed},
		{"on_mouse_released",     l_cathack_on_mouse_released},
		{"on_mouse_wheel",        l_cathack_on_mouse_wheel},
		{"on_char",               l_cathack_on_char},

		// Object reflection
		{"local_player_controller",    l_cathack_local_player_controller},
		{"local_character",            l_cathack_local_character},
		{"local_pawn",                 l_cathack_local_pawn},
		{"player_camera_manager",      l_cathack_player_camera_manager},
		{"object_valid",               l_cathack_object_valid},
		{"object_name",                l_cathack_object_name},
		{"object_class",               l_cathack_object_class},
		{"object_path",                l_cathack_object_path},
		{"object_properties",          l_cathack_object_properties},
		{"object_functions",           l_cathack_object_functions},
		{"object_find_properties",     l_cathack_object_find_properties},
		{"object_find_functions",      l_cathack_object_find_functions},
		{"object_get",                 l_cathack_object_get},
		{"object_set",                 l_cathack_object_set},
		{"object_call",                l_cathack_object_call},

		// PlayerController helpers
		{"pc_mouse_cursor",            l_cathack_pc_mouse_cursor},
		{"pc_set_mouse_cursor",        l_cathack_pc_set_mouse_cursor},
		{"pc_input_mode",              l_cathack_pc_input_mode},
		{"pc_set_input_mode",          l_cathack_pc_set_input_mode},
		{"pc_control_rotation",        l_cathack_pc_control_rotation},
		{"pc_set_control_rotation",    l_cathack_pc_set_control_rotation},
		{"pc_view_target",             l_cathack_pc_view_target},
		{"pc_set_view_target",         l_cathack_pc_set_view_target},
		{"pc_project_world_to_screen", l_cathack_pc_project_world_to_screen},
		{"pc_deproject_screen_to_world", l_cathack_pc_deproject_screen_to_world},

		// HTTP
		{"http_get",                   l_cathack_http_get},
		{"http_get_async",             l_cathack_http_get_async},
		{"http_clear_cache",           l_cathack_http_clear_cache},

		// Input — unified events + text editing
		{"chars_typed_printable",      l_cathack_chars_typed_printable},
		{"input_events",               l_cathack_input_events},
		{"key_events",                 l_cathack_key_events},
		{"text_input",                 l_cathack_text_input},

		// ImGui — windows
		{"imgui_begin",                l_cathack_imgui_begin},
		{"imgui_end",                  l_cathack_imgui_end},
		{"imgui_begin_child",          l_cathack_imgui_begin_child},
		{"imgui_end_child",            l_cathack_imgui_end_child},
		{"imgui_set_next_window_pos",  l_cathack_imgui_set_next_window_pos},
		{"imgui_set_next_window_size", l_cathack_imgui_set_next_window_size},
		// ImGui — layout
		{"imgui_same_line",            l_cathack_imgui_same_line},
		{"imgui_separator",            l_cathack_imgui_separator},
		{"imgui_spacing",              l_cathack_imgui_spacing},
		{"imgui_new_line",             l_cathack_imgui_new_line},
		{"imgui_dummy",                l_cathack_imgui_dummy},
		{"imgui_indent",               l_cathack_imgui_indent},
		{"imgui_unindent",             l_cathack_imgui_unindent},
		{"imgui_begin_group",          l_cathack_imgui_begin_group},
		{"imgui_end_group",            l_cathack_imgui_end_group},
		{"imgui_get_cursor_pos",       l_cathack_imgui_get_cursor_pos},
		{"imgui_set_cursor_pos",       l_cathack_imgui_set_cursor_pos},
		{"imgui_get_content_region_avail", l_cathack_imgui_get_content_region_avail},
		// ImGui — text
		{"imgui_text",                 l_cathack_imgui_text},
		{"imgui_text_colored",         l_cathack_imgui_text_colored},
		{"imgui_text_disabled",        l_cathack_imgui_text_disabled},
		{"imgui_text_wrapped",         l_cathack_imgui_text_wrapped},
		{"imgui_bullet_text",          l_cathack_imgui_bullet_text},
		{"imgui_label_text",           l_cathack_imgui_label_text},
		// ImGui — buttons / clickables
		{"imgui_button",               l_cathack_imgui_button},
		{"imgui_small_button",         l_cathack_imgui_small_button},
		{"imgui_invisible_button",     l_cathack_imgui_invisible_button},
		{"imgui_arrow_button",         l_cathack_imgui_arrow_button},
		// ImGui — toggles / values
		{"imgui_checkbox",             l_cathack_imgui_checkbox},
		{"imgui_radio_button",         l_cathack_imgui_radio_button},
		{"imgui_slider_int",           l_cathack_imgui_slider_int},
		{"imgui_slider_float",         l_cathack_imgui_slider_float},
		{"imgui_drag_int",             l_cathack_imgui_drag_int},
		{"imgui_drag_float",           l_cathack_imgui_drag_float},
		{"imgui_color_edit",           l_cathack_imgui_color_edit},
		{"imgui_combo",                l_cathack_imgui_combo},
		{"imgui_input_text",           l_cathack_imgui_input_text},
		{"imgui_input_text_multiline", l_cathack_imgui_input_text_multiline},
		// ImGui — selectable / headers / trees
		{"imgui_selectable",           l_cathack_imgui_selectable},
		{"imgui_collapsing_header",    l_cathack_imgui_collapsing_header},
		{"imgui_tree_node",            l_cathack_imgui_tree_node},
		{"imgui_tree_pop",             l_cathack_imgui_tree_pop},
		// ImGui — tabs
		{"imgui_begin_tab_bar",        l_cathack_imgui_begin_tab_bar},
		{"imgui_end_tab_bar",          l_cathack_imgui_end_tab_bar},
		{"imgui_begin_tab_item",       l_cathack_imgui_begin_tab_item},
		{"imgui_end_tab_item",         l_cathack_imgui_end_tab_item},
		// ImGui — tables
		{"imgui_begin_table",          l_cathack_imgui_begin_table},
		{"imgui_end_table",            l_cathack_imgui_end_table},
		{"imgui_table_setup_column",   l_cathack_imgui_table_setup_column},
		{"imgui_table_headers_row",    l_cathack_imgui_table_headers_row},
		{"imgui_table_next_row",       l_cathack_imgui_table_next_row},
		{"imgui_table_next_column",    l_cathack_imgui_table_next_column},
		{"imgui_table_set_column_index", l_cathack_imgui_table_set_column_index},
		// ImGui — tooltips
		{"imgui_set_tooltip",          l_cathack_imgui_set_tooltip},
		{"imgui_set_item_tooltip",     l_cathack_imgui_set_item_tooltip},
		{"imgui_begin_tooltip",        l_cathack_imgui_begin_tooltip},
		{"imgui_end_tooltip",          l_cathack_imgui_end_tooltip},
		// ImGui — item state queries
		{"imgui_is_item_hovered",      l_cathack_imgui_is_item_hovered},
		{"imgui_is_item_clicked",      l_cathack_imgui_is_item_clicked},
		{"imgui_is_item_active",       l_cathack_imgui_is_item_active},
		{"imgui_is_item_focused",      l_cathack_imgui_is_item_focused},
		{"imgui_is_window_hovered",    l_cathack_imgui_is_window_hovered},
		// ImGui — style + IDs
		{"imgui_push_style_color",     l_cathack_imgui_push_style_color},
		{"imgui_pop_style_color",      l_cathack_imgui_pop_style_color},
		{"imgui_push_style_var",       l_cathack_imgui_push_style_var},
		{"imgui_pop_style_var",        l_cathack_imgui_pop_style_var},
		{"imgui_push_id",              l_cathack_imgui_push_id},
		{"imgui_pop_id",               l_cathack_imgui_pop_id},

		{nullptr, nullptr},
	};

	// Push an integer field { name = value } into a table at -1.
	static inline void PushIntField(lua_State* L, const char* name, lua_Integer v)
	{
		lua_pushinteger(L, v); lua_setfield(L, -2, name);
	}

	// Build the cathack.imgui constants table — flag enums and color
	// indices scripts pass to imgui_* functions. We only ship the names
	// most scripts will actually need; rare flags can still be passed
	// as raw integers.
	static void BuildImGuiConstantsTable(lua_State* L)
	{
		lua_newtable(L); // -- cathack.imgui

		// --- Window flags --------------------------------------------
		lua_newtable(L);
		PushIntField(L, "None",                   ImGuiWindowFlags_None);
		PushIntField(L, "NoTitleBar",             ImGuiWindowFlags_NoTitleBar);
		PushIntField(L, "NoResize",               ImGuiWindowFlags_NoResize);
		PushIntField(L, "NoMove",                 ImGuiWindowFlags_NoMove);
		PushIntField(L, "NoScrollbar",            ImGuiWindowFlags_NoScrollbar);
		PushIntField(L, "NoScrollWithMouse",      ImGuiWindowFlags_NoScrollWithMouse);
		PushIntField(L, "NoCollapse",             ImGuiWindowFlags_NoCollapse);
		PushIntField(L, "AlwaysAutoResize",       ImGuiWindowFlags_AlwaysAutoResize);
		PushIntField(L, "NoBackground",           ImGuiWindowFlags_NoBackground);
		PushIntField(L, "NoSavedSettings",        ImGuiWindowFlags_NoSavedSettings);
		PushIntField(L, "NoMouseInputs",          ImGuiWindowFlags_NoMouseInputs);
		PushIntField(L, "MenuBar",                ImGuiWindowFlags_MenuBar);
		PushIntField(L, "HorizontalScrollbar",    ImGuiWindowFlags_HorizontalScrollbar);
		PushIntField(L, "NoFocusOnAppearing",     ImGuiWindowFlags_NoFocusOnAppearing);
		PushIntField(L, "NoBringToFrontOnFocus",  ImGuiWindowFlags_NoBringToFrontOnFocus);
		PushIntField(L, "AlwaysVerticalScrollbar",   ImGuiWindowFlags_AlwaysVerticalScrollbar);
		PushIntField(L, "AlwaysHorizontalScrollbar", ImGuiWindowFlags_AlwaysHorizontalScrollbar);
		PushIntField(L, "NoNavInputs",            ImGuiWindowFlags_NoNavInputs);
		PushIntField(L, "NoNavFocus",             ImGuiWindowFlags_NoNavFocus);
		PushIntField(L, "UnsavedDocument",        ImGuiWindowFlags_UnsavedDocument);
		PushIntField(L, "NoDecoration",           ImGuiWindowFlags_NoDecoration);
		PushIntField(L, "NoInputs",               ImGuiWindowFlags_NoInputs);
		lua_setfield(L, -2, "WindowFlags");

		// --- Input text flags ----------------------------------------
		lua_newtable(L);
		PushIntField(L, "None",                ImGuiInputTextFlags_None);
		PushIntField(L, "CharsDecimal",        ImGuiInputTextFlags_CharsDecimal);
		PushIntField(L, "CharsHexadecimal",    ImGuiInputTextFlags_CharsHexadecimal);
		PushIntField(L, "CharsUppercase",      ImGuiInputTextFlags_CharsUppercase);
		PushIntField(L, "CharsNoBlank",        ImGuiInputTextFlags_CharsNoBlank);
		PushIntField(L, "AutoSelectAll",       ImGuiInputTextFlags_AutoSelectAll);
		PushIntField(L, "EnterReturnsTrue",    ImGuiInputTextFlags_EnterReturnsTrue);
		PushIntField(L, "AllowTabInput",       ImGuiInputTextFlags_AllowTabInput);
		PushIntField(L, "CtrlEnterForNewLine", ImGuiInputTextFlags_CtrlEnterForNewLine);
		PushIntField(L, "NoHorizontalScroll",  ImGuiInputTextFlags_NoHorizontalScroll);
		PushIntField(L, "AlwaysOverwrite",     ImGuiInputTextFlags_AlwaysOverwrite);
		PushIntField(L, "ReadOnly",            ImGuiInputTextFlags_ReadOnly);
		PushIntField(L, "Password",            ImGuiInputTextFlags_Password);
		PushIntField(L, "NoUndoRedo",          ImGuiInputTextFlags_NoUndoRedo);
		PushIntField(L, "CharsScientific",     ImGuiInputTextFlags_CharsScientific);
		PushIntField(L, "EscapeClearsAll",     ImGuiInputTextFlags_EscapeClearsAll);
		lua_setfield(L, -2, "InputTextFlags");

		// --- Selectable flags ----------------------------------------
		lua_newtable(L);
		PushIntField(L, "None",              ImGuiSelectableFlags_None);
		PushIntField(L, "DontClosePopups",   ImGuiSelectableFlags_NoAutoClosePopups);
		PushIntField(L, "SpanAllColumns",    ImGuiSelectableFlags_SpanAllColumns);
		PushIntField(L, "AllowDoubleClick",  ImGuiSelectableFlags_AllowDoubleClick);
		PushIntField(L, "Disabled",          ImGuiSelectableFlags_Disabled);
		PushIntField(L, "AllowOverlap",      ImGuiSelectableFlags_AllowOverlap);
		lua_setfield(L, -2, "SelectableFlags");

		// --- TreeNode flags (CollapsingHeader uses the same set) -----
		lua_newtable(L);
		PushIntField(L, "None",               ImGuiTreeNodeFlags_None);
		PushIntField(L, "Selected",           ImGuiTreeNodeFlags_Selected);
		PushIntField(L, "Framed",             ImGuiTreeNodeFlags_Framed);
		PushIntField(L, "AllowOverlap",       ImGuiTreeNodeFlags_AllowOverlap);
		PushIntField(L, "NoTreePushOnOpen",   ImGuiTreeNodeFlags_NoTreePushOnOpen);
		PushIntField(L, "NoAutoOpenOnLog",    ImGuiTreeNodeFlags_NoAutoOpenOnLog);
		PushIntField(L, "DefaultOpen",        ImGuiTreeNodeFlags_DefaultOpen);
		PushIntField(L, "OpenOnDoubleClick",  ImGuiTreeNodeFlags_OpenOnDoubleClick);
		PushIntField(L, "OpenOnArrow",        ImGuiTreeNodeFlags_OpenOnArrow);
		PushIntField(L, "Leaf",               ImGuiTreeNodeFlags_Leaf);
		PushIntField(L, "Bullet",             ImGuiTreeNodeFlags_Bullet);
		PushIntField(L, "FramePadding",       ImGuiTreeNodeFlags_FramePadding);
		PushIntField(L, "SpanAvailWidth",     ImGuiTreeNodeFlags_SpanAvailWidth);
		PushIntField(L, "SpanFullWidth",      ImGuiTreeNodeFlags_SpanFullWidth);
		PushIntField(L, "CollapsingHeader",   ImGuiTreeNodeFlags_CollapsingHeader);
		lua_setfield(L, -2, "TreeNodeFlags");

		// --- Table flags ---------------------------------------------
		lua_newtable(L);
		PushIntField(L, "None",               ImGuiTableFlags_None);
		PushIntField(L, "Resizable",          ImGuiTableFlags_Resizable);
		PushIntField(L, "Reorderable",        ImGuiTableFlags_Reorderable);
		PushIntField(L, "Hideable",           ImGuiTableFlags_Hideable);
		PushIntField(L, "Sortable",           ImGuiTableFlags_Sortable);
		PushIntField(L, "NoSavedSettings",    ImGuiTableFlags_NoSavedSettings);
		PushIntField(L, "ContextMenuInBody",  ImGuiTableFlags_ContextMenuInBody);
		PushIntField(L, "RowBg",              ImGuiTableFlags_RowBg);
		PushIntField(L, "BordersInnerH",      ImGuiTableFlags_BordersInnerH);
		PushIntField(L, "BordersOuterH",      ImGuiTableFlags_BordersOuterH);
		PushIntField(L, "BordersInnerV",      ImGuiTableFlags_BordersInnerV);
		PushIntField(L, "BordersOuterV",      ImGuiTableFlags_BordersOuterV);
		PushIntField(L, "BordersH",           ImGuiTableFlags_BordersH);
		PushIntField(L, "BordersV",           ImGuiTableFlags_BordersV);
		PushIntField(L, "BordersInner",       ImGuiTableFlags_BordersInner);
		PushIntField(L, "BordersOuter",       ImGuiTableFlags_BordersOuter);
		PushIntField(L, "Borders",            ImGuiTableFlags_Borders);
		PushIntField(L, "NoBordersInBody",    ImGuiTableFlags_NoBordersInBody);
		PushIntField(L, "SizingFixedFit",     ImGuiTableFlags_SizingFixedFit);
		PushIntField(L, "SizingFixedSame",    ImGuiTableFlags_SizingFixedSame);
		PushIntField(L, "SizingStretchProp",  ImGuiTableFlags_SizingStretchProp);
		PushIntField(L, "SizingStretchSame",  ImGuiTableFlags_SizingStretchSame);
		PushIntField(L, "ScrollX",            ImGuiTableFlags_ScrollX);
		PushIntField(L, "ScrollY",            ImGuiTableFlags_ScrollY);
		PushIntField(L, "PadOuterX",          ImGuiTableFlags_PadOuterX);
		PushIntField(L, "NoPadOuterX",        ImGuiTableFlags_NoPadOuterX);
		PushIntField(L, "NoPadInnerX",        ImGuiTableFlags_NoPadInnerX);
		PushIntField(L, "NoClip",             ImGuiTableFlags_NoClip);
		lua_setfield(L, -2, "TableFlags");

		// --- TableColumn flags ---------------------------------------
		lua_newtable(L);
		PushIntField(L, "None",                  ImGuiTableColumnFlags_None);
		PushIntField(L, "DefaultHide",           ImGuiTableColumnFlags_DefaultHide);
		PushIntField(L, "DefaultSort",           ImGuiTableColumnFlags_DefaultSort);
		PushIntField(L, "WidthStretch",          ImGuiTableColumnFlags_WidthStretch);
		PushIntField(L, "WidthFixed",            ImGuiTableColumnFlags_WidthFixed);
		PushIntField(L, "NoResize",              ImGuiTableColumnFlags_NoResize);
		PushIntField(L, "NoReorder",             ImGuiTableColumnFlags_NoReorder);
		PushIntField(L, "NoHide",                ImGuiTableColumnFlags_NoHide);
		PushIntField(L, "NoClip",                ImGuiTableColumnFlags_NoClip);
		PushIntField(L, "NoSort",                ImGuiTableColumnFlags_NoSort);
		PushIntField(L, "NoSortAscending",       ImGuiTableColumnFlags_NoSortAscending);
		PushIntField(L, "NoSortDescending",      ImGuiTableColumnFlags_NoSortDescending);
		PushIntField(L, "NoHeaderLabel",         ImGuiTableColumnFlags_NoHeaderLabel);
		PushIntField(L, "NoHeaderWidth",         ImGuiTableColumnFlags_NoHeaderWidth);
		PushIntField(L, "PreferSortAscending",   ImGuiTableColumnFlags_PreferSortAscending);
		PushIntField(L, "PreferSortDescending",  ImGuiTableColumnFlags_PreferSortDescending);
		PushIntField(L, "IndentEnable",          ImGuiTableColumnFlags_IndentEnable);
		PushIntField(L, "IndentDisable",         ImGuiTableColumnFlags_IndentDisable);
		lua_setfield(L, -2, "TableColumnFlags");

		// --- TabBar / TabItem flags ----------------------------------
		lua_newtable(L);
		PushIntField(L, "None",                       ImGuiTabBarFlags_None);
		PushIntField(L, "Reorderable",                ImGuiTabBarFlags_Reorderable);
		PushIntField(L, "AutoSelectNewTabs",          ImGuiTabBarFlags_AutoSelectNewTabs);
		PushIntField(L, "TabListPopupButton",         ImGuiTabBarFlags_TabListPopupButton);
		PushIntField(L, "NoCloseWithMiddleMouseButton", ImGuiTabBarFlags_NoCloseWithMiddleMouseButton);
		PushIntField(L, "NoTabListScrollingButtons",  ImGuiTabBarFlags_NoTabListScrollingButtons);
		PushIntField(L, "NoTooltip",                  ImGuiTabBarFlags_NoTooltip);
		PushIntField(L, "FittingPolicyResizeDown",    ImGuiTabBarFlags_FittingPolicyResizeDown);
		PushIntField(L, "FittingPolicyScroll",        ImGuiTabBarFlags_FittingPolicyScroll);
		lua_setfield(L, -2, "TabBarFlags");
		lua_newtable(L);
		PushIntField(L, "None",                       ImGuiTabItemFlags_None);
		PushIntField(L, "UnsavedDocument",            ImGuiTabItemFlags_UnsavedDocument);
		PushIntField(L, "SetSelected",                ImGuiTabItemFlags_SetSelected);
		PushIntField(L, "NoCloseWithMiddleMouseButton", ImGuiTabItemFlags_NoCloseWithMiddleMouseButton);
		PushIntField(L, "NoPushId",                   ImGuiTabItemFlags_NoPushId);
		PushIntField(L, "NoTooltip",                  ImGuiTabItemFlags_NoTooltip);
		PushIntField(L, "NoReorder",                  ImGuiTabItemFlags_NoReorder);
		PushIntField(L, "Leading",                    ImGuiTabItemFlags_Leading);
		PushIntField(L, "Trailing",                   ImGuiTabItemFlags_Trailing);
		lua_setfield(L, -2, "TabItemFlags");

		// --- Conditions (used by SetNextWindow*) ---------------------
		lua_newtable(L);
		PushIntField(L, "None",         ImGuiCond_None);
		PushIntField(L, "Always",       ImGuiCond_Always);
		PushIntField(L, "Once",         ImGuiCond_Once);
		PushIntField(L, "FirstUseEver", ImGuiCond_FirstUseEver);
		PushIntField(L, "Appearing",    ImGuiCond_Appearing);
		lua_setfield(L, -2, "Cond");

		// --- Direction (arrow buttons) -------------------------------
		lua_newtable(L);
		PushIntField(L, "Left",  ImGuiDir_Left);
		PushIntField(L, "Right", ImGuiDir_Right);
		PushIntField(L, "Up",    ImGuiDir_Up);
		PushIntField(L, "Down",  ImGuiDir_Down);
		lua_setfield(L, -2, "Dir");

		// --- Style colors --------------------------------------------
		lua_newtable(L);
		PushIntField(L, "Text",                  ImGuiCol_Text);
		PushIntField(L, "TextDisabled",          ImGuiCol_TextDisabled);
		PushIntField(L, "WindowBg",              ImGuiCol_WindowBg);
		PushIntField(L, "ChildBg",               ImGuiCol_ChildBg);
		PushIntField(L, "PopupBg",               ImGuiCol_PopupBg);
		PushIntField(L, "Border",                ImGuiCol_Border);
		PushIntField(L, "BorderShadow",          ImGuiCol_BorderShadow);
		PushIntField(L, "FrameBg",               ImGuiCol_FrameBg);
		PushIntField(L, "FrameBgHovered",        ImGuiCol_FrameBgHovered);
		PushIntField(L, "FrameBgActive",         ImGuiCol_FrameBgActive);
		PushIntField(L, "TitleBg",               ImGuiCol_TitleBg);
		PushIntField(L, "TitleBgActive",         ImGuiCol_TitleBgActive);
		PushIntField(L, "TitleBgCollapsed",      ImGuiCol_TitleBgCollapsed);
		PushIntField(L, "MenuBarBg",             ImGuiCol_MenuBarBg);
		PushIntField(L, "ScrollbarBg",           ImGuiCol_ScrollbarBg);
		PushIntField(L, "ScrollbarGrab",         ImGuiCol_ScrollbarGrab);
		PushIntField(L, "ScrollbarGrabHovered",  ImGuiCol_ScrollbarGrabHovered);
		PushIntField(L, "ScrollbarGrabActive",   ImGuiCol_ScrollbarGrabActive);
		PushIntField(L, "CheckMark",             ImGuiCol_CheckMark);
		PushIntField(L, "SliderGrab",            ImGuiCol_SliderGrab);
		PushIntField(L, "SliderGrabActive",      ImGuiCol_SliderGrabActive);
		PushIntField(L, "Button",                ImGuiCol_Button);
		PushIntField(L, "ButtonHovered",         ImGuiCol_ButtonHovered);
		PushIntField(L, "ButtonActive",          ImGuiCol_ButtonActive);
		PushIntField(L, "Header",                ImGuiCol_Header);
		PushIntField(L, "HeaderHovered",         ImGuiCol_HeaderHovered);
		PushIntField(L, "HeaderActive",          ImGuiCol_HeaderActive);
		PushIntField(L, "Separator",             ImGuiCol_Separator);
		PushIntField(L, "SeparatorHovered",      ImGuiCol_SeparatorHovered);
		PushIntField(L, "SeparatorActive",       ImGuiCol_SeparatorActive);
		PushIntField(L, "ResizeGrip",            ImGuiCol_ResizeGrip);
		PushIntField(L, "ResizeGripHovered",     ImGuiCol_ResizeGripHovered);
		PushIntField(L, "ResizeGripActive",      ImGuiCol_ResizeGripActive);
		PushIntField(L, "Tab",                   ImGuiCol_Tab);
		PushIntField(L, "TabHovered",            ImGuiCol_TabHovered);
		PushIntField(L, "TabActive",             ImGuiCol_TabActive);
		PushIntField(L, "TabUnfocused",          ImGuiCol_TabUnfocused);
		PushIntField(L, "TabUnfocusedActive",    ImGuiCol_TabUnfocusedActive);
		PushIntField(L, "PlotLines",             ImGuiCol_PlotLines);
		PushIntField(L, "PlotLinesHovered",      ImGuiCol_PlotLinesHovered);
		PushIntField(L, "PlotHistogram",         ImGuiCol_PlotHistogram);
		PushIntField(L, "PlotHistogramHovered",  ImGuiCol_PlotHistogramHovered);
		PushIntField(L, "TableHeaderBg",         ImGuiCol_TableHeaderBg);
		PushIntField(L, "TableBorderStrong",     ImGuiCol_TableBorderStrong);
		PushIntField(L, "TableBorderLight",      ImGuiCol_TableBorderLight);
		PushIntField(L, "TableRowBg",            ImGuiCol_TableRowBg);
		PushIntField(L, "TableRowBgAlt",         ImGuiCol_TableRowBgAlt);
		PushIntField(L, "TextSelectedBg",        ImGuiCol_TextSelectedBg);
		PushIntField(L, "DragDropTarget",        ImGuiCol_DragDropTarget);
		PushIntField(L, "NavHighlight",          ImGuiCol_NavHighlight);
		lua_setfield(L, -2, "Col");

		// --- Style vars ----------------------------------------------
		lua_newtable(L);
		PushIntField(L, "Alpha",                ImGuiStyleVar_Alpha);
		PushIntField(L, "DisabledAlpha",        ImGuiStyleVar_DisabledAlpha);
		PushIntField(L, "WindowPadding",        ImGuiStyleVar_WindowPadding);
		PushIntField(L, "WindowRounding",       ImGuiStyleVar_WindowRounding);
		PushIntField(L, "WindowBorderSize",     ImGuiStyleVar_WindowBorderSize);
		PushIntField(L, "WindowMinSize",        ImGuiStyleVar_WindowMinSize);
		PushIntField(L, "WindowTitleAlign",     ImGuiStyleVar_WindowTitleAlign);
		PushIntField(L, "ChildRounding",        ImGuiStyleVar_ChildRounding);
		PushIntField(L, "ChildBorderSize",      ImGuiStyleVar_ChildBorderSize);
		PushIntField(L, "PopupRounding",        ImGuiStyleVar_PopupRounding);
		PushIntField(L, "PopupBorderSize",      ImGuiStyleVar_PopupBorderSize);
		PushIntField(L, "FramePadding",         ImGuiStyleVar_FramePadding);
		PushIntField(L, "FrameRounding",        ImGuiStyleVar_FrameRounding);
		PushIntField(L, "FrameBorderSize",      ImGuiStyleVar_FrameBorderSize);
		PushIntField(L, "ItemSpacing",          ImGuiStyleVar_ItemSpacing);
		PushIntField(L, "ItemInnerSpacing",     ImGuiStyleVar_ItemInnerSpacing);
		PushIntField(L, "IndentSpacing",        ImGuiStyleVar_IndentSpacing);
		PushIntField(L, "CellPadding",          ImGuiStyleVar_CellPadding);
		PushIntField(L, "ScrollbarSize",        ImGuiStyleVar_ScrollbarSize);
		PushIntField(L, "ScrollbarRounding",    ImGuiStyleVar_ScrollbarRounding);
		PushIntField(L, "GrabMinSize",          ImGuiStyleVar_GrabMinSize);
		PushIntField(L, "GrabRounding",         ImGuiStyleVar_GrabRounding);
		PushIntField(L, "TabRounding",          ImGuiStyleVar_TabRounding);
		PushIntField(L, "ButtonTextAlign",      ImGuiStyleVar_ButtonTextAlign);
		PushIntField(L, "SelectableTextAlign",  ImGuiStyleVar_SelectableTextAlign);
		lua_setfield(L, -2, "StyleVar");

		// --- Hovered flags -------------------------------------------
		lua_newtable(L);
		PushIntField(L, "None",            ImGuiHoveredFlags_None);
		PushIntField(L, "ChildWindows",    ImGuiHoveredFlags_ChildWindows);
		PushIntField(L, "RootWindow",      ImGuiHoveredFlags_RootWindow);
		PushIntField(L, "AnyWindow",       ImGuiHoveredFlags_AnyWindow);
		PushIntField(L, "AllowWhenBlockedByPopup",   ImGuiHoveredFlags_AllowWhenBlockedByPopup);
		PushIntField(L, "AllowWhenBlockedByActiveItem", ImGuiHoveredFlags_AllowWhenBlockedByActiveItem);
		PushIntField(L, "AllowWhenOverlapped", ImGuiHoveredFlags_AllowWhenOverlapped);
		PushIntField(L, "AllowWhenDisabled",  ImGuiHoveredFlags_AllowWhenDisabled);
		PushIntField(L, "RectOnly",        ImGuiHoveredFlags_RectOnly);
		PushIntField(L, "RootAndChildWindows", ImGuiHoveredFlags_RootAndChildWindows);
		PushIntField(L, "DelayShort",      ImGuiHoveredFlags_DelayShort);
		PushIntField(L, "DelayNormal",     ImGuiHoveredFlags_DelayNormal);
		PushIntField(L, "NoSharedDelay",   ImGuiHoveredFlags_NoSharedDelay);
		lua_setfield(L, -2, "HoveredFlags");

		// Stash on the cathack table.
		lua_setfield(L, -2, "imgui");
	}

	void OpenCathackLib(lua_State* L)
	{
		lua_createtable(L, 0, sizeof(kCathackFuncs) / sizeof(kCathackFuncs[0]) - 1);
		luaL_setfuncs(L, kCathackFuncs, 0);

		// cathack.imgui = { WindowFlags = {...}, Col = {...}, ... }
		BuildImGuiConstantsTable(L);

		lua_setglobal(L, "cathack");

		// Roblox-style `game:HttpGet(url)` global. Implemented as a tiny
		// table with a single method so the canonical idiom
		//     loadstring(game:HttpGet("https://..."))()
		// works without anyone needing to know about the cathack
		// namespace. The C function handles both `:` (method) and `.`
		// (bare-call) forms by sniffing arg-1 type at runtime.
		lua_newtable(L);
		lua_pushcfunction(L, l_game_http_get);
		lua_setfield(L, -2, "HttpGet");
		// Lowercase alias for the case-insensitive crowd.
		lua_pushcfunction(L, l_game_http_get);
		lua_setfield(L, -2, "http_get");
		lua_setglobal(L, "game");
	}

	// Replace the global print() so write-to-stdout calls land in our console.
	void OverridePrint(lua_State* L)
	{
		lua_pushcfunction(L, l_print);
		lua_setglobal(L, "print");
	}

	// Called when Lua emits an unhandled error during a chunk run. We just
	// stash it in the console — no message box. Ensures bad scripts can't
	// kill the cheat.
	int LuaPanic(lua_State* L)
	{
		const char* msg = lua_tostring(L, -1);
		PushConsole(std::string("[lua panic] ") + (msg ? msg : "(no message)"));
		return 0;
	}

	// ====================================================================
	// State construction. Called from Init() and Restart().
	// ====================================================================

	void DestroyState_locked()
	{
		// Free any textures the scripts loaded so the D3D11 SRVs aren't
		// leaked across a Restart. Has to happen BEFORE we close the
		// lua_State because the texture map is keyed by Lua-issued ids
		// but the SRVs themselves are pure D3D objects.
		FreeAllTextures_locked();
		if (g_L)
		{
			lua_close(g_L);
			g_L = nullptr;
		}
		g_tickRefs.clear();
		g_renderRefs.clear();
		g_keyPressedRefs.clear();
		g_keyReleasedRefs.clear();
		g_mousePressedRefs.clear();
		g_mouseReleasedRefs.clear();
		g_mouseWheelRefs.clear();
		g_charRefs.clear();
		g_charsThisFrame.clear();
		g_luaClipStackDepth = 0;
	}

	// === Watchdog hook ============================================
	// Wired via lua_sethook(LUA_MASKCOUNT, N): the VM calls this every
	// N instructions. We treat it as a wall-clock cancel switch — if the
	// per-call deadline (set by StartCallDeadline) has passed, raise a
	// catchable error to longjmp out of the running chunk. That keeps a
	// `while true do end` from freezing the game thread.
	void DeadlineHook(lua_State* L, lua_Debug* /*ar*/)
	{
		const long long deadline = g_callDeadlineMs.load(std::memory_order_relaxed);
		if (!deadline) return;
		const auto now = std::chrono::duration_cast<std::chrono::milliseconds>(
			std::chrono::steady_clock::now().time_since_epoch()).count();
		if (now > deadline)
		{
			// luaL_error raises a Lua error and longjmps. The pcall above
			// us catches it. We clear the deadline here so the message
			// formatting doesn't trip the hook again on its way out.
			g_callDeadlineMs.store(0, std::memory_order_relaxed);
			luaL_error(L,
				"call exceeded %lldms wall-clock budget — aborted to keep the game responsive",
				(long long)kCallBudgetMs);
		}
	}

	void StartCallDeadline_locked()
	{
		const auto now = std::chrono::duration_cast<std::chrono::milliseconds>(
			std::chrono::steady_clock::now().time_since_epoch()).count();
		g_callDeadlineMs.store(now + kCallBudgetMs, std::memory_order_relaxed);
	}

	void ClearCallDeadline_locked()
	{
		g_callDeadlineMs.store(0, std::memory_order_relaxed);
	}

	void CreateState_locked()
	{
		if (g_L) return;
		g_L = luaL_newstate();
		if (!g_L)
		{
			PushConsole("[lua] luaL_newstate failed");
			return;
		}
		lua_atpanic(g_L, LuaPanic);
		luaL_openlibs(g_L);
		OpenCathackLib(g_L);
		OverridePrint(g_L);
		// Install the watchdog. Counts every ~10k VM instructions; cheap
		// (the hook is a single atomic load when no deadline is armed).
		lua_sethook(g_L, DeadlineHook, LUA_MASKCOUNT, 10000);
		PushConsole("[lua] Lua 5.4 VM ready.");
	}

	// Run a single chunk under the global state. Returns true on success.
	// Errors are pushed to the console and the script's lastError field.
	bool RunChunk_locked(const std::string& name, const std::string& source, std::string* errOut)
	{
		if (!g_L)
		{
			if (errOut) *errOut = "Lua VM not initialized.";
			return false;
		}

		const std::string chunkName = std::string("@") + name + ".lua";
		const int loadStatus = luaL_loadbuffer(g_L, source.data(), source.size(), chunkName.c_str());
		if (loadStatus != LUA_OK)
		{
			const char* err = lua_tostring(g_L, -1);
			std::string e = std::string("[lua load] ") + (err ? err : "(no error)");
			PushConsole(e);
			if (errOut) *errOut = e;
			lua_pop(g_L, 1);
			return false;
		}

		g_runningScript = name;
		StartCallDeadline_locked();
		const int callStatus = lua_pcall(g_L, 0, LUA_MULTRET, 0);
		ClearCallDeadline_locked();
		g_runningScript.clear();

		if (callStatus != LUA_OK)
		{
			const char* err = lua_tostring(g_L, -1);
			std::string e = std::string("[lua run] ") + (err ? err : "(no error)");
			PushConsole(e);
			if (errOut) *errOut = e;
			lua_pop(g_L, 1);
			return false;
		}
		return true;
	}
}

// ====================================================================
// Public API
// ====================================================================

void CathackLua::Init()
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	if (g_initialized.load()) return;
	g_initialized.store(true);

	CreateState_locked();
	RescanScripts();

	if (g_autoLoadEnabled.load())
		RunAllAutoRun();
}

void CathackLua::Shutdown()
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	g_initialized.store(false);
	DestroyState_locked();
}

void CathackLua::Restart()
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	DestroyState_locked();
	CreateState_locked();
	for (auto& s : g_scripts) s.running = false;
}

void CathackLua::RescanScripts()
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	g_scripts.clear();
	std::error_code ec;
	const auto dir = GetLuaDirectory();
	for (const auto& entry : std::filesystem::directory_iterator(dir, ec))
	{
		if (ec) break;
		if (!entry.is_regular_file()) continue;
		auto p = entry.path();
		if (p.extension() != ".lua") continue;
		ScriptEntry s;
		s.name = p.stem().string();
		s.source = ReadFileAll(p);
		s.autoRun = DetectAutoRunDirective(s.source);
		s.running = false;
		s.dirty = false;
		g_scripts.push_back(std::move(s));
	}
	std::sort(g_scripts.begin(), g_scripts.end(),
		[](const ScriptEntry& a, const ScriptEntry& b) { return a.name < b.name; });
}

std::vector<CathackLua::ScriptEntry> CathackLua::GetScripts()
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	return g_scripts;
}

bool CathackLua::GetScript(const std::string& name, ScriptEntry* out)
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	for (const auto& s : g_scripts) if (s.name == name) { if (out) *out = s; return true; }
	return false;
}

bool CathackLua::SetScriptSource(const std::string& name, const std::string& source)
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	for (auto& s : g_scripts)
	{
		if (s.name == name)
		{
			s.source = source;
			s.dirty = true;
			s.autoRun = DetectAutoRunDirective(s.source);
			return true;
		}
	}
	return false;
}

bool CathackLua::SetScriptAutoRun(const std::string& name, bool autoRun)
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	for (auto& s : g_scripts)
	{
		if (s.name == name)
		{
			s.source = SetAutoRunDirective(s.source, autoRun);
			s.autoRun = autoRun;
			s.dirty = true;
			return true;
		}
	}
	return false;
}

bool CathackLua::CreateScript(const std::string& name)
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	if (!IsValidScriptName(name)) return false;
	for (const auto& s : g_scripts) if (s.name == name) return false;

	const std::filesystem::path p = ScriptPath(name);
	const std::string boilerplate =
		"-- " + name + ".lua\n"
		"-- Cathack Lua script. Uncomment the line below to auto-run on inject.\n"
		"-- @autorun\n"
		"\n"
		"print(\"" + name + " loaded\")\n"
		"\n"
		"-- cathack.on_tick(function(dt)\n"
		"--   if cathack.key(0x52) then -- VK_R\n"
		"--     cathack.set(\"aimbot\", not cathack.get(\"aimbot\"))\n"
		"--     cathack.notify(\"toggled aimbot\", true)\n"
		"--   end\n"
		"-- end)\n"
		"\n"
		"-- cathack.on_render(function()\n"
		"--   local sw, sh = cathack.screen_size()\n"
		"--   cathack.draw_text(sw / 2, 24, \"hello from " + name + "\", 0xFFFFFFFF)\n"
		"-- end)\n";
	if (!WriteFileAll(p, boilerplate)) return false;

	ScriptEntry s;
	s.name = name;
	s.source = boilerplate;
	s.autoRun = false;
	s.running = false;
	s.dirty = false;
	g_scripts.push_back(std::move(s));
	std::sort(g_scripts.begin(), g_scripts.end(),
		[](const ScriptEntry& a, const ScriptEntry& b) { return a.name < b.name; });
	return true;
}

bool CathackLua::DeleteScript(const std::string& name)
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	auto it = std::find_if(g_scripts.begin(), g_scripts.end(),
		[&](const ScriptEntry& s) { return s.name == name; });
	if (it == g_scripts.end()) return false;
	std::error_code ec;
	std::filesystem::remove(ScriptPath(name), ec);
	g_scripts.erase(it);
	return true;
}

bool CathackLua::SaveScript(const std::string& name)
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	for (auto& s : g_scripts)
	{
		if (s.name == name)
		{
			if (!WriteFileAll(ScriptPath(name), s.source)) return false;
			s.dirty = false;
			return true;
		}
	}
	return false;
}

bool CathackLua::ReloadScript(const std::string& name)
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	for (auto& s : g_scripts)
	{
		if (s.name == name)
		{
			s.source = ReadFileAll(ScriptPath(name));
			s.autoRun = DetectAutoRunDirective(s.source);
			s.dirty = false;
			return true;
		}
	}
	return false;
}

bool CathackLua::Run(const std::string& name)
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	for (auto& s : g_scripts)
	{
		if (s.name == name)
		{
			// Auto-evict any callbacks the script registered on a previous
			// run. Without this, every Run accumulates a fresh set of
			// hooks and `cathack.on_tick(...)` ends up firing N times per
			// tick after N runs — the classic "why is my ESP rendering
			// once per Run-click I clicked?" bug.
			RemoveCallbacksByOwner_locked(name);

			std::string err;
			const bool ok = RunChunk_locked(s.name, s.source, &err);
			s.running = ok;
			s.lastError = ok ? "" : err;
			return ok;
		}
	}
	return false;
}

bool CathackLua::Stop(const std::string& name)
{
	// Stop now actually evicts every callback registered by `name`'s
	// previous Run() — used to be a footgun where Stop only flipped the
	// UI indicator while the script's hooks kept firing.
	//
	// Globals / closures the script defined in the VM are NOT touched —
	// for that, click Reset VM. Persistent storage (cathack.store) and
	// any *other* script's hooks survive untouched.
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	for (auto& s : g_scripts)
	{
		if (s.name == name)
		{
			const int freed = RemoveCallbacksByOwner_locked(name);
			s.running = false;
			if (freed > 0)
			{
				char buf[96];
				snprintf(buf, sizeof(buf),
					"[lua] stopped %s (%d callback%s removed)",
					name.c_str(), freed, freed == 1 ? "" : "s");
				PushConsole(buf);
			}
			return true;
		}
	}
	return false;
}

void CathackLua::RunAllAutoRun()
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	for (auto& s : g_scripts)
	{
		if (!s.autoRun) continue;
		std::string err;
		const bool ok = RunChunk_locked(s.name, s.source, &err);
		s.running = ok;
		s.lastError = ok ? "" : err;
	}
}

void CathackLua::StopAll()
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	for (auto& s : g_scripts)
	{
		if (s.running)
			RemoveCallbacksByOwner_locked(s.name);
		s.running = false;
	}
}

void CathackLua::ConsolePush(const std::string& line)
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	PushConsole(line);
}

std::vector<std::string> CathackLua::ConsoleSnapshot()
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	return std::vector<std::string>(g_console.begin(), g_console.end());
}

void CathackLua::ConsoleClear()
{
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	g_console.clear();
}

bool CathackLua::GetAutoLoadEnabled()       { return g_autoLoadEnabled.load(); }
void CathackLua::SetAutoLoadEnabled(bool e) { g_autoLoadEnabled.store(e); }

// ====================================================================
// Tick fans
// ====================================================================

void CathackLua::TickGame()
{
	if (!g_initialized.load()) return;

	// Rate limit: ~60 Hz. Caller (post-ProcessEvent) fires at hundreds of Hz
	// during dense scenes so we cap explicitly. dt for the script is the
	// real elapsed since our last fire so simulations don't speed up when
	// the cap kicks in.
	const auto nowMs = std::chrono::duration_cast<std::chrono::milliseconds>(
		std::chrono::steady_clock::now().time_since_epoch()).count();
	const uint64_t now64 = static_cast<uint64_t>(nowMs);
	const uint64_t lastMs = g_lastTickMs;
	if (lastMs && (now64 - lastMs) < 16) return;
	const double dt = lastMs ? (now64 - lastMs) / 1000.0 : 0.0;
	g_lastTickMs = now64;

	std::lock_guard<std::recursive_mutex> lk(g_mu);
	if (!g_L || g_tickRefs.empty()) return;

	// Snapshot the callback list so a callback that calls cathack.untick
	// or cathack.clear_callbacks (mutating g_tickRefs mid-iteration)
	// doesn't invalidate our cursor.
	std::vector<int> refs;
	refs.reserve(g_tickRefs.size());
	for (const auto& cb : g_tickRefs) refs.push_back(cb.ref);

	for (int ref : refs)
	{
		// The callback might have been removed (untick / clear_callbacks
		// from a previous iteration) — skip if so.
		bool stillRegistered = false;
		for (const auto& cb : g_tickRefs) { if (cb.ref == ref) { stillRegistered = true; break; } }
		if (!stillRegistered) continue;
		lua_rawgeti(g_L, LUA_REGISTRYINDEX, ref);
		lua_pushnumber(g_L, dt);
		StartCallDeadline_locked();
		const int rc = lua_pcall(g_L, 1, 0, 0);
		ClearCallDeadline_locked();
		if (rc != LUA_OK)
		{
			const char* err = lua_tostring(g_L, -1);
			PushConsole(std::string("[lua tick] ") + (err ? err : "(no error)"));
			lua_pop(g_L, 1);
		}
	}
}

std::vector<std::string> CathackLua::GetApiNames()
{
	std::vector<std::string> out;
	for (const luaL_Reg* r = kCathackFuncs; r && r->name; ++r)
		out.emplace_back(std::string("cathack.") + r->name);
	return out;
}

std::vector<std::string> CathackLua::GetBindingNames()
{
	std::vector<std::string> out;
	for (const SettingBinding* b = kSettingBindings; b && b->name; ++b)
		out.emplace_back(b->name);
	return out;
}

void CathackLua::TickRender(ImDrawList* dl)
{
	if (!g_initialized.load()) return;
	std::lock_guard<std::recursive_mutex> lk(g_mu);
	if (!g_L) return;

	// Frame counter ticks even if nothing's hooked — scripts that just want
	// a "this is frame N" via cathack.frame_count() shouldn't depend on
	// having an on_render registration.
	g_frameCount.fetch_add(1, std::memory_order_relaxed);

	// Drain finished async HTTP jobs first. Their callbacks are user code
	// and we want them to land before any draw commands so a script that
	// does `cathack.http_get_async(url, function(body) state = ... end)`
	// can see `state` during the same frame's on_render.
	HttpDrainCompletedJobs_locked(g_L);

	// Reset the per-frame capture-request flags FIRST so scripts that
	// fall over (or stop calling capture_*) don't leave the game with
	// no input. on_render below gets to re-set them inside the same
	// frame; the WndProc only sees them on the next message after
	// on_render returns.
	CathackHostBridge::g_LuaCaptureMouseReq.store(false);
	CathackHostBridge::g_LuaCaptureKbdReq.store(false);

	// Drain typed-char queue maintained by the host's WndProc into our
	// own per-frame copy. on_char callbacks below get one fire per
	// codepoint; chars_typed() (polling) reads the same set.
	g_charsThisFrame.clear();
	{
		auto v = CathackHostBridge::DrainTypedChars();
		g_charsThisFrame = std::move(v);
	}

	// Drain the unified ordered input-event queue (chars + key
	// transitions in arrival order). Powers cathack.input_events,
	// cathack.key_events, and cathack.text_input.
	g_inputEventsThisFrame = CathackHostBridge::DrainInputEvents();

	// Snapshot the keys / mouse buttons that transitioned this frame, so
	// the event callbacks have stable inputs even if the dispatch itself
	// somehow re-enters. Only the named-key range is interesting; that's
	// every key ImGui knows about (ImGuiKey_NamedKey_BEGIN..END).
	std::vector<ImGuiKey> keysPressedThisFrame, keysReleasedThisFrame;
	if (!g_keyPressedRefs.empty() || !g_keyReleasedRefs.empty())
	{
		for (int k = ImGuiKey_NamedKey_BEGIN; k < ImGuiKey_NamedKey_END; ++k)
		{
			const ImGuiKey ik = static_cast<ImGuiKey>(k);
			if (!g_keyPressedRefs.empty()  && ImGui::IsKeyPressed(ik, false))
				keysPressedThisFrame.push_back(ik);
			if (!g_keyReleasedRefs.empty() && ImGui::IsKeyReleased(ik))
				keysReleasedThisFrame.push_back(ik);
		}
	}

	struct MouseEdge { int btn; ImVec2 pos; };
	std::vector<MouseEdge> mousePressed, mouseReleased;
	if (!g_mousePressedRefs.empty() || !g_mouseReleasedRefs.empty())
	{
		const ImVec2 p = ImGui::GetIO().MousePos;
		for (int b = 0; b < 5; ++b)
		{
			if (!g_mousePressedRefs.empty()  && ImGui::IsMouseClicked(b, false)) mousePressed.push_back({b, p});
			if (!g_mouseReleasedRefs.empty() && ImGui::IsMouseReleased(b))       mouseReleased.push_back({b, p});
		}
	}
	const float wheelV = ImGui::GetIO().MouseWheel;
	const float wheelH = ImGui::GetIO().MouseWheelH;
	const bool  hasWheelEvent = (!g_mouseWheelRefs.empty()) && (wheelV != 0.f || wheelH != 0.f);

	tl_currentDrawList = dl;

	// Helper: dispatch a single callback list, pushing N args onto the
	// stack via a caller-provided functor. Mirrors the existing on_tick /
	// on_render walker, including the still-registered re-check that
	// makes mid-iteration unhook safe.
	auto dispatchAll = [&](std::vector<Callback>& list, auto pushArgs, int nargs, const char* tag)
	{
		if (list.empty()) return;
		std::vector<int> refs;
		refs.reserve(list.size());
		for (const auto& cb : list) refs.push_back(cb.ref);
		for (int ref : refs)
		{
			bool stillRegistered = false;
			for (const auto& cb : list) { if (cb.ref == ref) { stillRegistered = true; break; } }
			if (!stillRegistered) continue;
			lua_rawgeti(g_L, LUA_REGISTRYINDEX, ref);
			pushArgs(g_L);
			StartCallDeadline_locked();
			const int rc = lua_pcall(g_L, nargs, 0, 0);
			ClearCallDeadline_locked();
			if (rc != LUA_OK)
			{
				const char* err = lua_tostring(g_L, -1);
				PushConsole(std::string("[lua ") + tag + "] " + (err ? err : "(no error)"));
				lua_pop(g_L, 1);
			}
		}
	};

	// on_char(c): one fire per codepoint. We feed UTF-8 strings (one per
	// codepoint) so scripts don't have to deal with surrogate pairs.
	for (unsigned int cp : g_charsThisFrame)
	{
		const std::string utf8 = Utf32ToUtf8(cp);
		dispatchAll(g_charRefs, [&](lua_State* L) { lua_pushlstring(L, utf8.data(), utf8.size()); }, 1, "char");
	}

	// on_key_pressed(vk) / on_key_released(vk). We translate the ImGuiKey
	// back to a Win32 VK so scripts that already use cathack.key(vk) /
	// VK_* see the same identifiers across both APIs.
	auto imguiKeyToVk = [](ImGuiKey k) -> int {
		// Inverse of VkToImGuiKey. Most direct ranges map back cleanly.
		if (k >= ImGuiKey_0 && k <= ImGuiKey_9) return '0' + (k - ImGuiKey_0);
		if (k >= ImGuiKey_A && k <= ImGuiKey_Z) return 'A' + (k - ImGuiKey_A);
		if (k >= ImGuiKey_F1 && k <= ImGuiKey_F12) return VK_F1 + (k - ImGuiKey_F1);
		switch (k)
		{
		case ImGuiKey_Tab:        return VK_TAB;
		case ImGuiKey_LeftArrow:  return VK_LEFT;
		case ImGuiKey_RightArrow: return VK_RIGHT;
		case ImGuiKey_UpArrow:    return VK_UP;
		case ImGuiKey_DownArrow:  return VK_DOWN;
		case ImGuiKey_PageUp:     return VK_PRIOR;
		case ImGuiKey_PageDown:   return VK_NEXT;
		case ImGuiKey_Home:       return VK_HOME;
		case ImGuiKey_End:        return VK_END;
		case ImGuiKey_Insert:     return VK_INSERT;
		case ImGuiKey_Delete:     return VK_DELETE;
		case ImGuiKey_Backspace:  return VK_BACK;
		case ImGuiKey_Space:      return VK_SPACE;
		case ImGuiKey_Enter:      return VK_RETURN;
		case ImGuiKey_Escape:     return VK_ESCAPE;
		case ImGuiKey_LeftShift: case ImGuiKey_RightShift: return VK_SHIFT;
		case ImGuiKey_LeftCtrl:  case ImGuiKey_RightCtrl:  return VK_CONTROL;
		case ImGuiKey_LeftAlt:   case ImGuiKey_RightAlt:   return VK_MENU;
		case ImGuiKey_LeftSuper:                           return VK_LWIN;
		case ImGuiKey_RightSuper:                          return VK_RWIN;
		case ImGuiKey_CapsLock:   return VK_CAPITAL;
		case ImGuiKey_NumLock:    return VK_NUMLOCK;
		case ImGuiKey_Semicolon:  return VK_OEM_1;
		case ImGuiKey_Equal:      return VK_OEM_PLUS;
		case ImGuiKey_Comma:      return VK_OEM_COMMA;
		case ImGuiKey_Minus:      return VK_OEM_MINUS;
		case ImGuiKey_Period:     return VK_OEM_PERIOD;
		case ImGuiKey_Slash:      return VK_OEM_2;
		case ImGuiKey_GraveAccent:return VK_OEM_3;
		case ImGuiKey_LeftBracket:return VK_OEM_4;
		case ImGuiKey_Backslash:  return VK_OEM_5;
		case ImGuiKey_RightBracket:return VK_OEM_6;
		case ImGuiKey_Apostrophe: return VK_OEM_7;
		default: return 0;
		}
	};
	for (ImGuiKey k : keysPressedThisFrame)
	{
		const int vk = imguiKeyToVk(k);
		if (vk == 0) continue;
		dispatchAll(g_keyPressedRefs, [&](lua_State* L) { lua_pushinteger(L, vk); }, 1, "key_pressed");
	}
	for (ImGuiKey k : keysReleasedThisFrame)
	{
		const int vk = imguiKeyToVk(k);
		if (vk == 0) continue;
		dispatchAll(g_keyReleasedRefs, [&](lua_State* L) { lua_pushinteger(L, vk); }, 1, "key_released");
	}

	// on_mouse_pressed(btn, x, y) / on_mouse_released(btn, x, y).
	for (const MouseEdge& e : mousePressed)
	{
		dispatchAll(g_mousePressedRefs, [&](lua_State* L) {
			lua_pushinteger(L, e.btn); lua_pushnumber(L, e.pos.x); lua_pushnumber(L, e.pos.y);
		}, 3, "mouse_pressed");
	}
	for (const MouseEdge& e : mouseReleased)
	{
		dispatchAll(g_mouseReleasedRefs, [&](lua_State* L) {
			lua_pushinteger(L, e.btn); lua_pushnumber(L, e.pos.x); lua_pushnumber(L, e.pos.y);
		}, 3, "mouse_released");
	}
	if (hasWheelEvent)
	{
		dispatchAll(g_mouseWheelRefs, [&](lua_State* L) {
			lua_pushnumber(L, wheelV); lua_pushnumber(L, wheelH);
		}, 2, "mouse_wheel");
	}

	// === on_render ===
	// Same snapshot pattern as TickGame — protects against unhook/clear
	// being called from inside a callback.
	if (!g_renderRefs.empty())
	{
		std::vector<int> refs;
		refs.reserve(g_renderRefs.size());
		for (const auto& cb : g_renderRefs) refs.push_back(cb.ref);

		for (int ref : refs)
		{
			bool stillRegistered = false;
			for (const auto& cb : g_renderRefs) { if (cb.ref == ref) { stillRegistered = true; break; } }
			if (!stillRegistered) continue;
			// Reset the draw layer to foreground at the start of every
			// on_render call so a script that switched to background
			// last frame doesn't accidentally affect another script's
			// drawing this frame. set_draw_layer is sticky within a
			// single callback's body but not across callbacks.
			tl_currentDrawList = dl;
			// Reset ImGui-bracket counters at the *start* of every
			// callback. Each on_render gets a clean slate — if callback
			// A leaves a Begin open by accident, we close it before
			// callback B runs (handled below), so B doesn't end up
			// drawing inside A's window or trip ImGui asserts.
			ResetImGuiPerFrame();
			lua_rawgeti(g_L, LUA_REGISTRYINDEX, ref);
			StartCallDeadline_locked();
			const int rc = lua_pcall(g_L, 0, 0, 0);
			ClearCallDeadline_locked();
			if (rc != LUA_OK)
			{
				const char* err = lua_tostring(g_L, -1);
				PushConsole(std::string("[lua render] ") + (err ? err : "(no error)"));
				lua_pop(g_L, 1);
			}
			// If a script left clip rects pushed (forgot to pop), clean
			// up now so we don't leak into the host's draw list. This is
			// pure hygiene — a well-behaved script never trips it.
			while (g_luaClipStackDepth > 0)
			{
				if (tl_currentDrawList) tl_currentDrawList->PopClipRect();
				--g_luaClipStackDepth;
			}
			// Auto-close any ImGui scopes the script forgot. Done per
			// callback (not just per-frame) so if the script crashed
			// mid-Begin / mid-tab / mid-table, the next script doesn't
			// see broken ImGui state.
			CleanupImGuiPerFrame();
			// Force the cursor visible if any imgui_begin fired this
			// callback — otherwise users can't see what they're
			// clicking when the cheat menu is closed. Scripts that
			// want stealth can manually set_cursor_visible(false) after
			// their imgui_begin.
			if (g_imAnyBeginThisFrame && ImGui::GetCurrentContext())
				ImGui::GetIO().MouseDrawCursor = true;
		}
	}
	tl_currentDrawList = nullptr;
}

// =====================================================================
// Camera override hot path. Both of these run outside the Lua mutex —
// they touch only the atomic override flags and the UE camera POV cache,
// so there's no need to serialize against script execution.
// =====================================================================

void CathackLua::ApplyCameraOverrides()
{
	auto* pov = GVars.POV;
	if (!pov) return;
	// Cheap shortcut: if nothing's flagged, just return.
	const bool hasPos = g_camHasPosOverride.load(std::memory_order_relaxed);
	const bool hasRot = g_camHasRotOverride.load(std::memory_order_relaxed);
	const bool hasFov = g_camHasFovOverride.load(std::memory_order_relaxed);
	if (!hasPos && !hasRot && !hasFov) return;

	if (hasPos)
	{
		pov->Location.X = g_camOverridePosX.load(std::memory_order_relaxed);
		pov->Location.Y = g_camOverridePosY.load(std::memory_order_relaxed);
		pov->Location.Z = g_camOverridePosZ.load(std::memory_order_relaxed);
	}
	if (hasRot)
	{
		pov->Rotation.Pitch = g_camOverridePitch.load(std::memory_order_relaxed);
		pov->Rotation.Yaw   = g_camOverrideYaw.load(std::memory_order_relaxed);
		pov->Rotation.Roll  = g_camOverrideRoll.load(std::memory_order_relaxed);
	}
	if (hasFov)
	{
		// Stomp both FOV and DesiredFOV — the engine uses DesiredFOV to
		// drive zoom transitions, so writing only FOV would briefly snap
		// back next frame.
		const float f = g_camOverrideFov.load(std::memory_order_relaxed);
		pov->FOV        = f;
		pov->DesiredFOV = f;
	}
}

void CathackLua::ClearCameraOverrides()
{
	g_camHasPosOverride.store(false);
	g_camHasRotOverride.store(false);
	g_camHasFovOverride.store(false);
}

void CathackLua::TickCameraSampling()
{
	auto* pov = GVars.POV;
	if (!pov) return;
	using namespace std::chrono;
	const long long now = duration_cast<milliseconds>(
		steady_clock::now().time_since_epoch()).count();
	const float curX = static_cast<float>(pov->Location.X);
	const float curY = static_cast<float>(pov->Location.Y);
	const float curZ = static_cast<float>(pov->Location.Z);
	if (g_camHadPrevSample.load(std::memory_order_relaxed))
	{
		const long long lastT = g_camLastSampleMs.load(std::memory_order_relaxed);
		// Plain ternary instead of std::max — Windows.h's `max` macro
		// (#include <Windows.h> at the top of this file) makes the
		// std:: form choke unless we wrap in extra parens.
		const float dtRaw = (now - lastT) / 1000.0f;
		const float dt = dtRaw < 0.001f ? 0.001f : dtRaw;
		g_camVelX.store((curX - g_camLastX.load()) / dt, std::memory_order_relaxed);
		g_camVelY.store((curY - g_camLastY.load()) / dt, std::memory_order_relaxed);
		g_camVelZ.store((curZ - g_camLastZ.load()) / dt, std::memory_order_relaxed);
	}
	g_camLastX.store(curX, std::memory_order_relaxed);
	g_camLastY.store(curY, std::memory_order_relaxed);
	g_camLastZ.store(curZ, std::memory_order_relaxed);
	g_camLastSampleMs.store(now, std::memory_order_relaxed);
	g_camHadPrevSample.store(true, std::memory_order_relaxed);
}
