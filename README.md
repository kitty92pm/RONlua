# Cathack Lua scripting

Full reference for the in-process Lua 5.4 scripting tab.

---

## Contents

1. [Overview](#overview)
2. [Where scripts live](#where-scripts-live)
3. [The Lua tab UI](#the-lua-tab-ui)
4. [Auto-run directive (`-- @autorun`)](#auto-run-directive---autorun)
5. [Threading model](#threading-model)
6. [Sandbox: what's available](#sandbox-whats-available)
7. [API reference: `cathack.*`](#api-reference-cathack)
   - [Console + notifications](#console--notifications)
   - [Settings (`get` / `set` / `settings`)](#settings)
   - [Local player state](#local-player-state)
   - [Camera](#camera)
   - [Engine / session info](#engine--session-info)
   - [Entities](#entities)
   - [Per-entity functions](#per-entity-functions)
   - [Raycasts / line-of-sight](#raycasts--line-of-sight)
   - [Math / vectors](#math--vectors)
   - [Color helpers](#color-helpers)
   - [Screen / projection](#screen--projection)
   - [Input](#input)
   - [Text editing helper](#text-editing-helper)
   - [Mouse + cursor + capture](#mouse--cursor--capture)
   - [Cheat menu state](#cheat-menu-state)
   - [Time / framerate](#time--framerate)
   - [Drawing primitives](#drawing-primitives)
   - [Sized & aligned text](#sized--aligned-text)
   - [Fancy primitives (gradients / shadows / glow)](#fancy-primitives-gradients--shadows--glow)
   - [Textures and images](#textures-and-images)
   - [Clipping & draw layers](#clipping--draw-layers)
   - [Clipboard](#clipboard)
   - [Theme + animation](#theme--animation)
   - [Viewport](#viewport)
   - [JSON](#json)
   - [Persistent storage](#persistent-storage)
   - [Filesystem](#filesystem)
   - [Dynamic execution](#dynamic-execution)
   - [HTTP](#http)
   - [ImGui](#imgui)
   - [Callbacks](#callbacks)
   - [Object reflection](#object-reflection)
   - [PlayerController helpers](#playercontroller-helpers)
8. [Full setting binding table](#full-setting-binding-table)
9. [Patterns and idioms](#patterns-and-idioms)
10. [Worked examples](#worked-examples)
11. [Limitations and gotchas](#limitations-and-gotchas)

---

## Overview

Cathack ships an embedded Lua 5.4 VM. Scripts you write in the **Lua** tab can:

- Read and write almost every cheat setting (aimbot, silent, chams, viewmodel, FOV, etc.).
- Enumerate every AI character / player in the level (`cathack.entities()`), with positions, bones, health, team, name, alive state, distance, etc.
- Read full camera state (pos, rot, forward / right / up vectors, FOV, aspect, near clip, velocity, view target, post-process blend).
- Override the camera per-frame: pin position, rotation, FOV — the foundation for freecam, zoom, cinematic camera, etc.
- Switch the active view target to spectate any actor.
- Trigger camera fades and stop active camera shakes.
- Read your own player's position, velocity, aim rotation, team, name, health.
- Cast world-space rays for line-of-sight or hit detection.
- Project world coordinates to screen.
- Draw text, rects, lines, circles, triangles, quads, polylines, beziers, and world-anchored text.
- Draw fancy UI primitives: rect gradients, soft drop-shadows, glow circles, sized + aligned text, and PNG/JPG images loaded from `%APPDATA%\cathack\lua\`.
- Build full **interactive UIs** — read mouse position / clicks / wheel / drag, capture the mouse and keyboard so clicks on a custom UI don't shoot or move the player, push/pop clip rects for scroll panels, and pick a draw layer (background / foreground).
- Build colors at runtime, including HSV; interpolate colors with `lerp_color`; ease-curve any 0..1 with `ease(...)`.
- Persist key/value state across reloads (`cathack.store` / `cathack.fetch`), including arbitrary tables via `cathack.json_encode` / `json_decode`.
- Read and write files inside the script directory.
- Poll any Windows virtual-key code, get edge-triggered "pressed this frame" / "released this frame" via ImGui, drain the per-frame typed-character queue.
- Subscribe to event callbacks: `on_key_pressed`, `on_key_released`, `on_mouse_pressed`, `on_mouse_released`, `on_mouse_wheel`, `on_char`.
- Read clipboard / write clipboard from a script.
- Match the cheat's accent and theme via `cathack.theme()` / `cathack.accent_color()`.
- Open / close / toggle the main cheat menu programmatically.
- Push toast notifications.
- Hook into a game-thread tick (~60 Hz) and a render-thread tick (every frame).
- **Reflect** on the local `PlayerController`, `Character`, `Pawn`, and `PlayerCameraManager` — list every property and function the engine exposes on those objects, read primitive properties (bool/int/float/FName/FString/FVector/FRotator/FVector2D), write whitelisted properties, and call whitelisted functions through `cathack.object_call`.
- Use higher-level convenience helpers (`cathack.pc_*`) for the most common PlayerController operations: mouse cursor, input mode, control rotation, view target, world ↔ screen projection.

Scripts are saved as `.lua` files on disk, can be flagged auto-run on inject, and can be edited / run / stopped from the menu.

---

## Where scripts live

```
%APPDATA%\cathack\lua\<name>.lua
```

Concretely on most setups: `C:\Users\<you>\AppData\Roaming\cathack\lua\`. Drop or write `.lua` files there and they'll show up in the tab on the next **Reload all**. The filename without the extension is the script's display name; valid characters are letters, digits, `_`, `-`, `.` (no spaces, no `..`).

The directory is auto-created on first run.

---

## Editor features at a glance

The Lua tab's editor has a small set of conveniences that compound on top of the plain `InputTextMultiline` widget:

| Action | What it does |
|---|---|
| Typing a partial identifier (e.g. `pri`, `cathack.cam`, `math.f`) | Opens a popup of prefix-matching candidates from Lua keywords, the standard library, and the full `cathack.*` API |
| **Tab** (with popup open) | Inserts the highlighted match |
| **↑ / ↓** (with popup open) | Cycle through matches |
| **↑ / ↓** (with popup closed) | Move cursor between lines (preserves column) |
| **Ctrl+S** | Save the current script to disk |
| **Ctrl+R** | Save and run the current script (errors go to the console) |
| Status bar | Shows `Ln X, Col Y`, total character count, and a hint line |
| Accent dot on a script in the list | Indicates unsaved changes (editor buffer differs from disk) |
| Green dot on a script in the list | Indicates the script's tick/render callbacks are live |

### Settings (cog button on the editor toolbar)

The **Settings** button next to `Save` / `Run` / `Stop` / `Delete` opens a small popup with editor-only quality-of-life options. Toggles are persisted to `%APPDATA%/cathack/lua/editor.cfg` so they survive restarts.

| Option | What it does |
|---|---|
| **Smooth typing** | Spawns a tiny accent-colored ripple at the cursor on each visible-character keystroke. Pure cosmetic. |
| **Keyboard click sound** | Plays a synthesized click on each keystroke. Volume slider + a `Test` button to preview. |
| **Auto-close brackets and quotes** | When you type `(`, `[`, `{`, `"` or `'`, drops the matching closer in and parks the cursor between them. Skipped when the next char is part of an identifier so `foo(name)` wraps cleanly. |
| **Auto-indent on Enter** | Copies the previous line's leading whitespace forward so blocks keep their indent. |
| **Highlight current line** | Tints the row your cursor is on so it's easy to spot at a glance. |
| **Reset to defaults** | Restores the popup to factory defaults and writes the file. |

The click sound is generated at runtime (~22 ms of decaying noise + a faint pitched pop), baked at the chosen volume, and played via `PlaySoundA(SND_MEMORY | SND_ASYNC)`. No external sound files are loaded.

The autocomplete pool is built once on first use from:

- 22 Lua 5.4 reserved keywords
- ~60 standard-library entries (`math.*`, `string.*`, `table.*`, `io.*`, `os.*` time helpers, `print` / `pairs` / `ipairs` / `pcall` / etc.)
- Every function exported on the `cathack.*` API table (~120 entries — runs from `CathackLua::GetApiNames()`)

Matches are case-insensitive prefix matches. Up to 8 are shown at once with a "+N more" hint when the list is longer.

> Tab is repurposed as the autocomplete-commit key; if you really need a literal `\t` in your source, paste it in.

---

## The Lua tab UI

```
+----------------+----------------------------+----------------+
| Scripts        | <name>.lua  Save Run Stop  | Console        |
| [auto-load on] |                            | Reset Clear … |
| my_esp     [x] |                            | [output…]      |
| toggle_aim []  |   editor (multiline)       |                |
| ...            |                            |                |
| Create Reload  |                            |                |
+----------------+----------------------------+----------------+
```

- **Left rail**
  - List of every `.lua` in `%APPDATA%\cathack\lua\`.
  - Per-row checkbox toggles `-- @autorun` in the source.
  - Green dot = the script is currently running (its top-level chunk has executed at least once and hasn't been stopped).
  - `*` prefix = editor buffer differs from on-disk content.
  - **Auto-load on inject** — global toggle. When ON, every script with `-- @autorun` runs automatically right after the cheat finishes loading saved settings.
  - **Create** — name the new script and click; a starter file with commented examples is written.
  - **Reload all** — re-scan the folder. Picks up files added externally and discards unsaved edits.

- **Center**
  - Filename header.
  - **Save** — flush the editor buffer to disk.
  - **Run** — flush to in-memory source + execute the chunk under the global VM. Errors land in the console.
  - **Stop** — flips the script's "running" flag. **Does not unregister callbacks** the script registered — Lua registry refs are anonymous, so to fully evict a stuck script's callbacks use **Reset VM** or have the script call `cathack.clear_callbacks()` / track its handles (see below).
  - **Find** — opens a small search bar above the editor. Distinct from the menu-wide Ctrl+F popup so neither steals focus from the other. Hotkey **Ctrl+G** (vim/codemirror convention). Inside the bar:
    - Enter cycles to the next match.
    - **F3** / **Shift+F3** navigate next / prev (works even when the editor itself has focus).
    - **Aa** toggles case sensitivity.
    - **Esc** closes the bar.
    - Matches are highlighted in the editor; the current match is brighter and outlined.
  - **Delete** — confirm popup → removes the file from disk.
  - **Settings** — small popup with editor cosmetics:
    - Smooth typing (particle ripple at the cursor on each keystroke).
    - Keyboard click sound (synthesized in memory, with volume slider).
    - Auto-close brackets and quotes (`(`, `[`, `{`, `"`, `'`).
    - Auto-indent on Enter (copies the previous line's leading whitespace forward).
    - Highlight current line.
    - **Syntax highlighting** — tokenizes Lua and renders keywords (purple), Lua builtins (cyan), `cathack.*` API calls (accent), numbers (orange), strings (green), and comments (muted blue) in their own text colors. The InputText glyphs are painted transparent and the colored text is redrawn on top so the cursor, selection rect, scroll, and find-match highlights all stay aligned. Tokenization is cached on a content hash and only the visible line range is repainted.
  - Multi-line editor (Tab is the autocomplete commit key, not a literal `\t` insert; paste a tab in for a literal one).

- **Right rail (console)**
  - **Reset VM** — close the lua_State, rebuild it fresh. Every callback every script registered is gone. Use when something is stuck.
  - **Clear log** — empty the console buffer (held in memory; not on disk).
  - **Run all** — runs every script flagged `@autorun`. Same code path as auto-load on inject, just on demand.
  - The console captures `print(...)` and `cathack.print(...)` calls plus any Lua error messages from chunk loads, runs, ticks, and renders.

Switching scripts in the list while you have unsaved changes stashes the in-memory buffer onto the previous script's `ScriptEntry` so you don't lose work bouncing around. Disk only updates when you click **Save**.

---

## Auto-run directive (`-- @autorun`)

Cathack flags a script for auto-run by looking for `@autorun` inside a Lua comment somewhere in the first ~20 lines:

```lua
-- @autorun

print("this runs on inject")
```

The per-script checkbox in the UI just rewrites this line at the top of the file. When you toggle it OFF the line is removed; toggling ON inserts `-- @autorun` as line 1.

When **Auto-load on inject** is OFF in the UI, the directive is ignored at startup — you have to click **Run** (or **Run all**) manually.

---

## Threading model

There's exactly one `lua_State`, protected by a recursive mutex. Every public API call holds the mutex for its full duration.

Two callback fans:

| Fan | Fired from | Rate | Safe to do |
|---|---|---|---|
| `on_tick`   | Game thread (post-`UObject::ProcessEvent` hook) | ~60 Hz (rate-limited internally) | Read game state, call `cathack.player_pos()`, `cathack.player_health()`, write settings. |
| `on_render` | Render thread (inside `IDXGISwapChain::Present`) | Every frame | Call `cathack.draw_*` (only valid inside this fan). |

Drawing primitives (`cathack.draw_*`) only work inside `on_render`. Calling them from `on_tick` is a silent no-op, not a crash.

The whole module is gated to authed users — scripts won't run before KeyAuth unlocks the menu.

---

## Sandbox: what's available

Lua 5.4 standard libraries that are loaded into every state:

- `_G` (base): `print` (redirected to the console), `type`, `pairs`, `ipairs`, `error`, `pcall`, `xpcall`, `tostring`, `tonumber`, `assert`, `select`, `setmetatable`, `getmetatable`, `rawset`, `rawget`, `rawequal`, `rawlen`, `next`, `unpack` (5.1 compat name absent — use `table.unpack`), `load`, `loadstring`, `dofile`, `loadfile`, `collectgarbage`, etc.
- `coroutine.*`
- `string.*` (including pattern matching)
- `math.*` (including `math.random`, `math.pi`, `math.huge`)
- `table.*`
- `utf8.*`
- `io.*` (full filesystem IO — local scripts only, you've already trusted them)
- `debug.*` (stack inspection / hooks)

**Removed for safety / portability:**

- `os.*` — `os.execute`, `os.exit`, `os.remove` etc. would let a script run arbitrary processes / kill the game / corrupt files. The `loslib.c` translation unit is removed from the build.
- `package` / `require` / `loadlib` — would let a script `require` a native DLL inside the cheat. The `loadlib.c` translation unit is removed.

If you absolutely need a few `os` functions you can add them back in `linit.c` and re-vendor `loslib.c`; they were dropped on principle, not because they don't work.

---

## API reference: `cathack.*`

Everything below lives on the global `cathack` table. The table is sealed for new fields per state — adding `cathack.foo = ...` works fine for your own state on top of the host bindings.

### Console + notifications

#### `cathack.print(...)`

Same arg shape as `print`. Pushes one space-joined line to the in-game console. The default `print` global is also redirected here so any library you `dofile` works without changes.

```lua
cathack.print("loaded", 1, true)   -- console: "loaded 1 true"
print("same thing")
```

#### `cathack.notify(message, force?)`

Pushes a toast in the bottom-right corner. `force = true` ignores the user's "toasts disabled" setting in Visuals → Notifications.

```lua
cathack.notify("script ready")
cathack.notify("override toasts off", true)
```

Returns nothing.

---

### Settings

Cathack exposes ~50 internal cheat settings under stable string names. See [the full table below](#full-setting-binding-table) for the canonical list.

#### `cathack.get(name) -> value`

Returns a `boolean`, `number`, or integer depending on the binding's kind. Errors with `unknown setting '<name>'` if the name doesn't exist.

#### `cathack.set(name, value)`

Type is inferred from the binding. Numbers are silently clamped to the binding's range (the same range the menu sliders use). Errors with `unknown setting '<name>'` if the name doesn't exist.

```lua
cathack.set("aimbot", true)
cathack.set("aimbot.fov", 45.0)
cathack.set("silent.method", 2)         -- integer settings take integers
cathack.set("aimbot", not cathack.get("aimbot"))   -- toggle
```

#### `cathack.settings() -> table`

Returns a fresh table mapping every binding name to its current value. Useful for diffing or saving snapshots.

```lua
local snap = cathack.settings()
print(snap["aimbot.fov"], snap["silent.hit_chance"])
```

---

### Local player state

All of these read game state from the local player. They return `nil` when there is no live local player (loading screens, alt-tab during transition, etc.).

#### `cathack.player_pos() -> x, y, z | nil`

World-space position in Unreal centimeters.

```lua
local x, y, z = cathack.player_pos()
if x then print(x, y, z) end
```

#### `cathack.player_health() -> current, max | nil`

```lua
local cur, max = cathack.player_health()
if cur and cur < max * 0.25 then
  cathack.notify("low health", true)
end
```

#### `cathack.is_alive() -> boolean`

`true` while the local player is alive. Returns `false` for not-alive **and** for any state where the actor isn't valid yet (mid-load).

#### `cathack.player_velocity() -> vx, vy, vz | nil`

World-space velocity (cm/s).

#### `cathack.player_speed() -> number | nil`

3D length of `player_velocity()` (cm/s). Convenient for "am I sprinting" tests.

#### `cathack.player_aim_rot() -> pitch, yaw, roll | nil`

Base aim rotation in degrees. Same source the engine uses for aim cone resolution.

#### `cathack.player_team() -> string | nil`

One of `"swat"`, `"suspect"`, `"civilian"`, `"unknown"`. The local player is always SWAT in normal coop play; classification is by Unreal class.

#### `cathack.player_name() -> string | nil`

Display name from `PlayerState`.

#### `cathack.local_id() -> integer | nil`

Returns the local player's entity id (the same kind of id `entities()` returns for AI). Useful for e.g. excluding yourself from custom logic that already iterates `entities()` (already done by default — `entities()` skips yourself — but `entity_*` accessors still accept this id if you want to introspect yourself).

---

### Camera

The cheat exposes the full Player Camera Manager surface — every read-only property, the three coordinate-space basis vectors, frame-to-frame velocity, view-target spectating, fade effects, shake control, and **per-frame overrides** for position/rotation/FOV (which is what you build a freecam on top of).

All reading functions return `nil` until the camera is initialized (typically a handful of frames after spawn). Override setters are silently ignored when there's no live camera; they "stick" and apply once the camera comes online.

#### Reading

##### `cathack.camera_pos() -> x, y, z | nil`

World-space camera location (cm).

##### `cathack.camera_rot() -> pitch, yaw, roll | nil`

Camera rotation in degrees.

##### `cathack.camera_forward() -> x, y, z | nil`

Unit vector aligned with the camera's look direction.

##### `cathack.camera_right() -> x, y, z | nil`

Unit vector pointing camera-right (useful for strafe-direction freecam logic).

##### `cathack.camera_up() -> x, y, z | nil`

Unit vector pointing camera-up.

##### `cathack.camera_fov() -> number | nil`

Current effective FOV in degrees (post-zoom, post-engine modifiers).

##### `cathack.camera_aspect() -> number | nil`

Aspect ratio the camera is rendering at.

##### `cathack.camera_near_clip() -> number | nil`

Perspective near-clip plane distance (cm). Anything inside this distance is clipped by the renderer.

##### `cathack.camera_velocity() -> vx, vy, vz`

Camera velocity in cm/s, computed from frame-to-frame position deltas. Returns `(0, 0, 0)` for the very first sample.

##### `cathack.camera_speed() -> number`

3D length of `camera_velocity()`.

##### `cathack.camera_view_target() -> entity_id | nil`

Returns the id of the actor the camera is currently following (the result of `SetViewTargetWithBlend`). Usually equals `cathack.local_id()` during normal play; differs while spectating or during cinematics. `nil` outside a live world.

##### `cathack.camera_post_process_blend() -> number | nil`

Current `PostProcessBlendWeight` (0..1) applied to the camera. Useful for syncing your own UI fades to camera fades.

#### Coordinate transforms

##### `cathack.screen_to_world_ray(sx, sy) -> ox, oy, oz, dx, dy, dz | nil`

Inverse of `world_to_screen`. Returns 6 numbers: a ray origin and a unit direction, both in world space, that pass through the given pixel. Best basic primitive for "what is the player aiming at" without the trigonometric work.

```lua
local ox, oy, oz, dx, dy, dz = cathack.screen_to_world_ray(sw / 2, sh / 2)
local r = cathack.line_trace(ox, oy, oz, ox + dx*5000, oy + dy*5000, oz + dz*5000)
if r and r.entity_id then
  print("crosshair points at entity", r.entity_id)
end
```

##### `cathack.world_to_camera_relative(wx, wy, wz) -> forward, right, up | nil`

Convert a world point into camera-relative coordinates, expressed as scalar projections onto the camera's forward/right/up vectors. Useful for "is this point in front of me?", "how far to my left is this?" without reaching for ProjectWorldLocationToScreen.

##### `cathack.camera_to_world(forward, right, up) -> wx, wy, wz | nil`

Inverse: reconstruct a world position from camera-local coordinates.

#### Per-frame overrides (freecam, FOV lock, cinematic camera)

The override functions stash values in atomics that the cheat reapplies on **every game-thread tick** *and* **every render frame**. That double application is what makes overrides survive UE's own camera updates — the engine still recomputes its view, then the cheat stomps it back to your values just before the renderer reads them.

Setting `nil` (or calling the matching `clear_*` function) cancels an override. Position, rotation, and FOV are independent — you can override just one and let the engine drive the others.

##### `cathack.set_camera_pos(x, y, z)` / `cathack.clear_camera_pos()`

Pin the camera at a fixed world position. Combine with `set_camera_rot` for a freecam.

##### `cathack.set_camera_rot(pitch, yaw, roll?)` / `cathack.clear_camera_rot()`

Pin the camera's orientation. `roll` defaults to 0.

##### `cathack.set_camera_fov(fov)` / `cathack.clear_camera_fov()`

Pin the rendered FOV. Clamped to `[1, 170]`. Stomps both `FOV` and `DesiredFOV` on the camera POV cache so zoom transitions can't briefly fight you.

##### `cathack.clear_camera_overrides()`

Clear all three overrides at once.

#### View target / spectating

##### `cathack.set_view_target(entity_id_or_nil, blend_time?)`

Switch the rendered camera to follow another actor. Pass an `entity_id` (from `entities()` or `local_id()`) or `nil` to revert to the local pawn. `blend_time` (seconds, default `0`) is a smooth blend between the old and new view.

```lua
-- Cycle through suspects on F
local idx = 0
cathack.on_tick(function()
  if cathack.key(0x46) and not held then
    held = true
    local list = {}
    for _, e in ipairs(cathack.entities()) do
      if e.kind == "suspect" and e.alive then list[#list+1] = e end
    end
    if #list > 0 then
      idx = (idx % #list) + 1
      cathack.set_view_target(list[idx].id, 0.5)
    end
  elseif not cathack.key(0x46) then
    held = false
  end
end)
```

#### Effects

##### `cathack.camera_fade(from_alpha, to_alpha, duration, color?, fade_audio?, hold?)`

Animated screen fade. Alphas are 0..1 (0 = no fade, 1 = full color). `color` is a packed ABGR int (use `cathack.color()`); default opaque black. `fade_audio` muxes audio. `hold` keeps the final alpha after the duration ends.

```lua
cathack.camera_fade(1.0, 0.0, 1.5, cathack.color(0, 0, 0), false, false) -- 1.5s fade-from-black
```

##### `cathack.camera_fade_stop()`

Stop and clear any running fade.

##### `cathack.camera_set_manual_fade(amount, color?, fade_audio?)`

Set the fade alpha directly (no animation). Useful for syncing with your own logic.

##### `cathack.camera_stop_shakes(immediate?)`

Stop every running camera shake. Pass `true` to stop instantly; otherwise the shakes blend out over their natural fade.

---

### Engine / session info

#### `cathack.is_in_game() -> boolean`

`true` once the world, level, player controller, and local character are all live and valid. Best gate for "we can safely call entity / camera / raycast functions".

#### `cathack.world_name() -> string | nil`

UObject name of the current `UWorld`. Examples: `"L_Hawkins"`, `"L_Brewery"`. `nil` outside any world.

---

### Entities

`cathack.entities()` is the workhorse for anything that touches AI: it enumerates every AI character (suspects, civilians, SWAT bots, other players) currently in the level. The local player is **excluded** — use `cathack.local_id()` to refer to yourself.

#### `cathack.entities() -> array of records`

Each record is a Lua table with these fields:

| Field | Type | Description |
|---|---|---|
| `id`             | integer | Stable handle for the rest of this frame. Pass to `entity_*` getters. |
| `kind`           | string  | `"player"` / `"swat"` / `"suspect"` / `"civilian"` / `"unknown"` |
| `pos`            | `{x, y, z}` | Actor pivot location (foot-height in UE). |
| `head`           | `{x, y, z}` | Best-effort "head" bone position; falls back to `pos` if unknown. |
| `health`         | number  | Current health. |
| `max_health`     | number  | Maximum health. |
| `distance`       | number  | Distance from local player in meters. |
| `alive`          | bool    | |
| `dead`           | bool    | |
| `suspect`        | bool    | |
| `civilian`       | bool    | |
| `swat`           | bool    | |
| `arrested`       | bool    | |
| `surrendered`    | bool    | |
| `incapacitated`  | bool    | |
| `being_arrested` | bool    | |
| `name`           | string? | Only populated for human players (PlayerState name). |

```lua
for _, e in ipairs(cathack.entities()) do
  if e.kind == "suspect" and e.alive and e.distance < 50 then
    print("close suspect:", e.id, e.distance, "m")
  end
end
```

The id is the underlying actor pointer cast to an integer. It's safe to keep around for one tick / frame — beyond that, the actor may have been freed and the id will fail validation when next used. Every `entity_*(id)` call validates the pointer before dereferencing, so a stale id returns `nil`, never crashes.

#### `cathack.entity_count() -> integer`

Equivalent to `#cathack.entities()` but doesn't allocate the table.

---

### Per-entity functions

All of these accept an entity `id` (as returned by `entities()` or `local_id()`). They return `nil` if the id is stale / invalid.

#### `cathack.entity_pos(id) -> x, y, z | nil`

Actor pivot position (cm).

#### `cathack.entity_velocity(id) -> vx, vy, vz | nil`

#### `cathack.entity_speed(id) -> number | nil`

3D length of velocity.

#### `cathack.entity_rotation(id) -> pitch, yaw, roll | nil`

Actor world rotation (the body-facing direction).

#### `cathack.entity_aim_rot(id) -> pitch, yaw, roll | nil`

Pawn aim rotation (the head/aim direction). Different from `entity_rotation` for AI that's looking around without turning their body.

#### `cathack.entity_health(id) -> cur, max | nil`

#### `cathack.entity_kind(id) -> string | nil`

Same as the `kind` field on the records returned by `entities()`.

#### `cathack.entity_alive(id) -> boolean`

#### `cathack.entity_distance(id) -> number | nil`

Distance from local player in **meters**.

#### `cathack.entity_name(id) -> string | nil`

Player-state name. `nil` for AI.

#### `cathack.entity_bone(id, name_or_index) -> x, y, z | nil`

World-space position of a bone. Pass either:

- a **string** name (e.g. `"Head"`, `"spine_03"`, `"hand_l"`) — case-sensitive match; or
- an **integer** index into the skeleton's bone array (`0`-based).

```lua
local id = cathack.entities()[1].id
local x, y, z = cathack.entity_bone(id, "Head")
local hx, hy = cathack.world_to_screen(x, y, z)
if hx then cathack.draw_circle_filled(hx, hy, 4, 0xFFFFFFFF) end
```

#### `cathack.entity_screen_pos(id, mode?) -> sx, sy, ok | nil`

Convenience: project the entity to screen coordinates. `mode` is `"feet"` (default) or `"head"`.

```lua
local sx, sy = cathack.entity_screen_pos(id, "head")
if sx then cathack.draw_text(sx + 6, sy - 6, "TGT") end
```

#### `cathack.entity_visible(id) -> boolean`

Line-of-sight from the camera to the entity's head, ignoring the local player. Cheap-ish but not free — caches nothing, traces every call.

---

### Raycasts / line-of-sight

#### `cathack.line_of_sight(x1, y1, z1, x2, y2, z2) -> boolean | nil`

`true` if a world-space line trace from A to B is unobstructed. Ignores the local player. `nil` outside a valid world.

#### `cathack.line_trace(x1, y1, z1, x2, y2, z2) -> table | nil`

Full result of a line trace. Returns `nil` outside a valid world, otherwise a record:

| Field | Type | Description |
|---|---|---|
| `hit`        | bool   | true if there was a blocking hit |
| `x`/`y`/`z`  | number | hit point (only meaningful when `hit`) |
| `distance`   | number | distance from start to hit |
| `entity_id`  | integer? | If the hit was on an AI character, its id |

```lua
local cx, cy, cz = cathack.camera_pos()
local fx, fy, fz = cathack.camera_forward()
local r = cathack.line_trace(cx, cy, cz, cx + fx*5000, cy + fy*5000, cz + fz*5000)
if r and r.hit and r.entity_id then
  print("looking at entity", r.entity_id, "at", r.distance, "cm")
end
```

---

### Math / vectors

Lightweight numeric helpers. All vectors are passed as 3 numbers (`x, y, z`) — no Lua table allocation.

#### `cathack.distance(x1, y1, z1, x2, y2, z2) -> number`

Euclidean distance in centimeters (whatever the input unit is).

#### `cathack.normalize(x, y, z) -> nx, ny, nz`

Unit-length version. Zero-length input returns `(0, 0, 0)`.

#### `cathack.dot(x1, y1, z1, x2, y2, z2) -> number`

#### `cathack.cross(x1, y1, z1, x2, y2, z2) -> nx, ny, nz`

#### `cathack.lerp(a, b, t) -> number`

Linear interpolation. `t` is unclamped, so you can extrapolate.

---

### Color helpers

The cheat's drawing functions take colors as packed 32-bit ImGui ABGR integers (`0xAABBGGRR`). These helpers build them at runtime.

#### `cathack.color(r, g, b, a?) -> integer`

Components are 0–255 ints. `a` defaults to 255. Out-of-range values are clamped.

```lua
local red = cathack.color(255, 0, 0)              -- 0xFF0000FF
local soft = cathack.color(255, 255, 255, 80)     -- low-alpha white
```

#### `cathack.color_hsv(h, s, v, a?) -> integer`

`h`, `s`, `v` are 0..1 floats. `a` is 0–255 (default 255). Useful for animating hues:

```lua
cathack.on_render(function()
  local hue = (cathack.time() % 5000) / 5000.0
  local c = cathack.color_hsv(hue, 1.0, 1.0)
  cathack.draw_rect_filled(20, 20, 40, 40, c)
end)
```

---

### Screen / projection

#### `cathack.screen_size() -> w, h`

Pixel dimensions of the game's render area, as ImGui sees them.

#### `cathack.world_to_screen(x, y, z) -> sx, sy, ok | nil`

Projects a world position to screen pixels using the local player's camera. Returns `nil` if there's no PlayerController, or three values when projection succeeded:

- `sx`, `sy` — screen-space pixels
- `ok` — `true`. Always `true` on the success path; the third return is reserved so future revisions can return `false` for behind-camera points without breaking calling code.

```lua
local px, py = cathack.world_to_screen(0, 0, 0) -- world origin
if px then
  cathack.draw_circle_filled(px, py, 4, 0xFFFFFFFF)
end
```

Returns `nil` when off-screen / behind camera.

---

### Text editing helper

Building a text input field by hand involves three subtle issues that bite every script author the first time:
- Arrow keys do not produce `WM_CHAR`, so they never appear in `chars_typed()` or `on_char` — `chars_typed()` alone can't move the caret.
- Backspace **does** appear in `chars_typed()` as raw byte `0x08`. The classic bug is `state.text = state.text .. cathack.chars_typed()` which concats a literal `\b` byte instead of erasing.
- Holding a key needs auto-repeat at the OS keyboard rate. `cathack.on_key_pressed` only fires once per discrete press; you have to either pass `repeat = true` to `cathack.key_pressed` or read events from the unified queue.

`cathack.text_input(state[, opts])` solves all three and lets a Lua text input be one call per frame.

#### `cathack.text_input(state[, opts]) -> state`

Mutates `state` in-place from this frame's input events and returns it for chaining.

**State fields** (all optional on input, always written on output):
- `text` — the buffer (string).
- `caret` — caret position in bytes, `0..#text`.
- `sel_anchor` — when set, marks the other end of a selection. The selection range is `[min(caret, sel_anchor), max(...)]`. `nil` means no selection.
- `submit` — set `true` on output when Enter was pressed this frame (single-line mode only).

**Opts fields**:
- `max_length` — int. Inserts beyond this length are truncated.
- `multiline` — bool. When `false` (default) Enter sets `submit` and is not inserted. When `true`, Enter inserts `\n`.

**What it handles**:
- Printable insert (UTF-8, multi-byte safe — Ctrl+letter chords are skipped because they're hotkeys).
- Backspace, Delete (deletes selection if there is one, otherwise the char on the relevant side of the caret).
- Left / Right arrow (UTF-8 boundary-aware).
- Ctrl+Left / Ctrl+Right (word-jump).
- Shift+arrow / Shift+Home / Shift+End (selection grow).
- Home, End.
- Ctrl+A (select all), Ctrl+C (copy), Ctrl+X (cut), Ctrl+V (paste — strips CRLF; drops `\n` when not multiline).
- Auto-repeat: every WM_KEYDOWN with the repeat flag set is consumed, so holding a key walks the caret / repeats the action at the OS repeat rate.

```lua
-- @autorun
local field = { text = "", caret = 0 }

cathack.on_render(function()
    cathack.capture_keyboard(true)            -- claim keys this frame
    cathack.text_input(field, { max_length = 64 })

    cathack.draw_rect_filled(40, 40, 320, 28, 0xC0202020, 4)
    cathack.draw_rect       (40, 40, 320, 28, cathack.accent_color(), 1)
    cathack.draw_text(48, 46, field.text, 0xFFFFFFFF)

    -- Caret
    local before = field.text:sub(1, field.caret)
    local cw     = cathack.measure_text(before)
    if cathack.frame_count() % 60 < 30 then
        cathack.draw_rect_filled(48 + cw, 44, 2, 18, 0xFFFFFFFF)
    end

    if field.submit then
        cathack.notify("submitted: " .. field.text)
        field.text, field.caret = "", 0
    end
end)
```

#### `cathack.input_events() -> array of event tables`

Returns the full per-frame ordered stream of input events — `WM_CHAR` codepoints and `WM_KEY*DOWN` / `WM_KEY*UP` transitions interleaved in arrival order. Same data the helper above eats. Use this when you want to roll your own editor or do something exotic.

Each event is:
```lua
{ kind = "char",     codepoint = 65, text = "A",
  repeated = false, shift = false, ctrl = false, alt = false }

{ kind = "key_down", vk = 0x25,        -- VK_LEFT
  repeated = true,  shift = false, ctrl = false, alt = false }

{ kind = "key_up",   vk = 0x25,
  repeated = false, shift = false, ctrl = false, alt = false }
```

The snapshot is taken at the top of `TickRender` — every call within the same frame returns the same array.

#### `cathack.key_events() -> array of event tables`

Same data as `input_events` but only the `key_down` / `key_up` rows. Matches what `on_key_pressed` sees, **plus** auto-repeat key-downs that `on_key_pressed` filters out. Use this when you specifically want navigation-key behavior (arrows / backspace / delete with hold-to-repeat) without dealing with the char rows.

```lua
for _, ev in ipairs(cathack.key_events()) do
  if ev.kind == "key_down" and ev.vk == 0x25 then  -- VK_LEFT
    cursor = math.max(0, cursor - 1)
  end
end
```

#### `cathack.chars_typed_printable() -> string`

Like `cathack.chars_typed()` but with C0 controls (`0x00..0x1F`) and DEL (`0x7F`) stripped. Use this when you want to append printable insertion text to a buffer and are reading navigation keys from `key_events()` separately.

---

### Input

#### `cathack.key(vk) -> boolean`

`true` while the given Windows virtual key is held. Backed by `GetAsyncKeyState`, so it reads the global key state regardless of whether the game window has focus (you'll usually want a foreground check too, but for in-game polling that's overkill).

Common VKs:

| Key | VK |
|---|---|
| Letters `A`–`Z` | `0x41`–`0x5A` |
| Digits `0`–`9` | `0x30`–`0x39` |
| Function `F1`–`F12` | `0x70`–`0x7B` |
| `Space`           | `0x20` |
| `Shift`           | `0x10` |
| `Ctrl`            | `0x11` |
| `Alt`             | `0x12` |
| Mouse `LMB / RMB` | `0x01` / `0x02` |
| Mouse `MMB`       | `0x04` |
| Mouse `X1 / X2`   | `0x05` / `0x06` |

```lua
if cathack.key(0x52) then    -- VK_R
  cathack.notify("R held")
end
```

For "edge-triggered" (fired once per press) use either the pattern in [Patterns and idioms](#patterns-and-idioms) or `cathack.key_pressed(vk)` / `cathack.key_released(vk)` (see below).

#### `cathack.key_pressed(vk[, repeat]) -> boolean`

`true` only on the frame the key transitioned from up → down. With `repeat=true`, also returns true while the key auto-repeats. Equivalent to ImGui's `IsKeyPressed`.

#### `cathack.key_released(vk) -> boolean`

`true` only on the frame the key transitioned from down → up.

#### `cathack.chars_typed() -> string`

Returns every character typed since the **last call** in this render frame. UTF-8 encoded; surrogate pairs already collapsed. Drained on read — calling it twice in the same frame returns the second batch as `""`.

```lua
cathack.on_render(function()
  local s = cathack.chars_typed()
  if #s > 0 then print("typed:", s) end
end)
```

For event-style char input use `cathack.on_char(fn)` (see [Callbacks](#callbacks)).

---

### Mouse + cursor + capture

All mouse APIs read from ImGui's per-frame state, so they're consistent with the menu and reset cleanly between frames. Button IDs are **0-indexed**:

| ID | Button |
|---|---|
| `0` | Left |
| `1` | Right |
| `2` | Middle |
| `3` | Mouse X1 (back) |
| `4` | Mouse X2 (forward) |

#### `cathack.mouse_pos() -> x, y`

Cursor position in screen pixels (top-left origin).

#### `cathack.mouse_delta() -> dx, dy`

Per-frame movement. Equal to `mouse_pos` minus the previous frame's `mouse_pos`.

#### `cathack.mouse_down(button) -> boolean`

`true` while `button` is held.

#### `cathack.mouse_clicked(button[, repeat]) -> boolean`

`true` only on the frame the button transitioned from up → down. `repeat=true` opts into ImGui's auto-repeat semantics.

#### `cathack.mouse_released(button) -> boolean`

`true` only on the frame of the up edge.

#### `cathack.mouse_double_clicked(button) -> boolean`

`true` on the frame ImGui considers a double-click. Backed by ImGui's per-button double-click timing.

#### `cathack.mouse_wheel() -> dv, dh`

Vertical and horizontal wheel delta this frame. `0` when not scrolling. `local v = cathack.mouse_wheel()` works because Lua just discards extra returns.

#### `cathack.mouse_drag_delta(button[, threshold]) -> dx, dy`

Distance dragged since the button went down, in screen pixels. `threshold` (default `-1` = ImGui's default, ~6 px) is the dead zone before the drag is considered "started."

#### `cathack.mouse_in_rect(x, y, w, h) -> boolean`

Convenience hover-test. Returns `true` if the cursor is inside the given rectangle.

```lua
local x, y, w, h = 100, 100, 200, 30
cathack.draw_rect_filled(x, y, w, h,
  cathack.mouse_in_rect(x, y, w, h) and 0xFF333333 or 0xFF222222)
```

#### `cathack.set_cursor_visible(bool)` / `cathack.cursor_visible() -> boolean`

Software cursor (the ImGui-rendered arrow). When the main cheat menu is open this is forced on; closing it restores whatever the script last set.

#### `cathack.capture_mouse([bool])` / `cathack.capture_keyboard([bool])`

> **THE crucial pair for interactive UI.**

When called with `true` (or no argument), the host swallows mouse / keyboard messages on the **next frame** so they never reach the game's window proc. That stops a click on a custom UI button from also firing your weapon, hitting reload, etc.

Both flags are **per-frame**: they reset to `false` at the start of every render frame. Scripts must keep calling `cathack.capture_mouse(true)` every frame they want the capture to apply (the same way ImGui's `SetNextFrameWantCaptureMouse` works). This is on purpose — if a script crashes mid-frame, the player isn't left input-locked.

```lua
local btn = { x=200, y=200, w=160, h=36 }

cathack.on_render(function()
  local hover = cathack.mouse_in_rect(btn.x, btn.y, btn.w, btn.h)
  if hover then cathack.capture_mouse(true) end  -- this frame only

  cathack.draw_rect_filled(btn.x, btn.y, btn.w, btn.h,
    hover and 0xFF555555 or 0xFF333333, 4)
  cathack.draw_text(btn.x + 12, btn.y + 10, "Click me")

  if hover and cathack.mouse_clicked(0) then
    cathack.notify("clicked!")
  end
end)
```

#### `cathack.want_capture_mouse() -> boolean` / `cathack.want_capture_keyboard() -> boolean`

Returns `true` if **either** ImGui or any Lua script has requested capture this frame. Useful for game-side logic (`on_tick`) that should pause when a UI is up:

```lua
cathack.on_tick(function()
  if cathack.want_capture_keyboard() then return end  -- skip when UI eats input
  -- … movement / aim logic …
end)
```

---

### Cheat menu state

#### `cathack.menu_open() -> boolean`

`true` when the cheat menu (the one you toggle with the menu key) is visible.

#### `cathack.set_menu_open(bool)`

Open or close the menu programmatically. Mirrors what the toggle key does.

#### `cathack.toggle_menu()`

Flip the visibility. Equivalent to `cathack.set_menu_open(not cathack.menu_open())`.

```lua
-- Auto-hide a custom HUD when the cheat menu is open
cathack.on_render(function()
  if cathack.menu_open() then return end
  draw_hud()
end)
```

---

### Time / framerate

#### `cathack.time() -> integer ms`

Steady monotonic clock in milliseconds. Same source the cheat itself uses internally; useful for animation timing and rate-limiting your own logic inside `on_render`.

```lua
local t0 = cathack.time()
-- … do work …
print("took", cathack.time() - t0, "ms")
```

#### `cathack.delta_time() -> number`

Last frame's delta in seconds (from ImGui IO). Convenient for time-based animations inside `on_render`:

```lua
cathack.on_render(function()
  -- 90°/sec spin
  rot = (rot or 0) + 90 * cathack.delta_time()
end)
```

#### `cathack.fps() -> number`

ImGui's smoothed framerate.

---

### Drawing primitives

**All drawing functions are no-ops outside `cathack.on_render`.** They write into the ImGui background draw list, which sits between the game render and the cheat menu (so drawings show under the menu but on top of the world).

Colors are 32-bit ABGR integers (`0xAABBGGRR`). The menu happens to use the same encoding ImGui uses, e.g.:

| Color | Literal |
|---|---|
| White (opaque) | `0xFFFFFFFF` |
| Black (opaque) | `0xFF000000` |
| Red            | `0xFF0000FF` |
| Green          | `0xFF00FF00` |
| Blue           | `0xFFFF0000` |
| Yellow         | `0xFF00FFFF` |
| 50% white      | `0x80FFFFFF` |

Helper to build colors at runtime:

```lua
local function rgba(r, g, b, a)
  return (a << 24) | (b << 16) | (g << 8) | r
end
```

#### `cathack.draw_text(x, y, text, color?)`

```lua
cathack.draw_text(20, 20, "hello", 0xFFFFFFFF)
```

#### `cathack.draw_rect(x, y, w, h, color?, thickness?)`

Outline only.

```lua
cathack.draw_rect(40, 40, 100, 60, 0xFF00FFFF, 2)
```

#### `cathack.draw_rect_filled(x, y, w, h, color?, rounding?)`

Solid fill, optional corner rounding in pixels.

```lua
cathack.draw_rect_filled(40, 110, 100, 60, 0x80FF0000, 8)
```

#### `cathack.draw_line(x1, y1, x2, y2, color?, thickness?)`

```lua
cathack.draw_line(0, 0, sw, sh, 0xFFFFFFFF, 1)
```

#### `cathack.draw_circle(cx, cy, radius, color?, segments?, thickness?)`

`segments` defaults to 32. Higher = smoother circle.

```lua
cathack.draw_circle(sw / 2, sh / 2, 60, 0xFF0000FF, 64, 1.5)
```

#### `cathack.draw_circle_filled(cx, cy, radius, color?, segments?)`

```lua
cathack.draw_circle_filled(sw / 2, sh / 2, 4, 0xFFFFFFFF)
```

#### `cathack.draw_triangle(x1, y1, x2, y2, x3, y3, color?, thickness?)`

Outline only.

#### `cathack.draw_triangle_filled(x1, y1, x2, y2, x3, y3, color?)`

Solid fill. Useful for direction arrows / aim indicators.

#### `cathack.draw_quad(x1, y1, x2, y2, x3, y3, x4, y4, color?, thickness?)`

Outline of a 4-vertex polygon (need not be axis-aligned or convex).

#### `cathack.draw_quad_filled(x1, y1, x2, y2, x3, y3, x4, y4, color?)`

#### `cathack.draw_polyline(points, color?, thickness?, closed?)`

`points` is a 1-indexed array of `{x, y}` pairs. `closed` connects the last point to the first.

```lua
cathack.draw_polyline({
  {100, 100}, {140, 80}, {180, 100}, {180, 140}, {140, 160}, {100, 140},
}, 0xFFFFFFFF, 2, true)
```

#### `cathack.draw_bezier(x1, y1, x2, y2, x3, y3, x4, y4, color?, thickness?, segments?)`

Cubic Bezier. `segments = 0` lets ImGui auto-pick.

#### `cathack.measure_text(text) -> w, h`

Returns the size the cheat menu's font will render `text` at. Useful for centering text or sizing background rects.

#### `cathack.world_text(x, y, z, text, color?, offset_y?) -> sx, sy, ok | nil`

Convenience: project a world point and draw text at the resulting screen pixel. Returns the screen coords (or `nil` if off-screen).

```lua
cathack.on_render(function()
  for _, e in ipairs(cathack.entities()) do
    if e.alive then
      cathack.world_text(e.head.x, e.head.y, e.head.z + 20,
        string.format("%s [%dm]", e.kind, math.floor(e.distance)),
        0xFFFFFF00, -16)
    end
  end
end)
```

---

### Sized & aligned text

The plain `cathack.draw_text(x, y, s, color)` always uses the menu's current font size. These helpers let you scale and align without measuring by hand.

#### `cathack.draw_text_sized(x, y, text[, color, size])`

Same as `draw_text` but with an optional `size` (pixels). `size` of `0` (or omitted) falls back to the menu font size.

#### `cathack.draw_text_centered(x, y, text[, color, size])`

`(x, y)` is the **center** of the text bounding box.

#### `cathack.draw_text_right(x, y, text[, color, size])`

`(x, y)` is the **top-right** of the text bounding box. Useful for right-aligned labels.

#### `cathack.measure_text_sized(text[, size]) -> w, h`

Like `cathack.measure_text` but accepts the same optional `size` so you measure at the size you'll draw at.

```lua
-- Center "PAUSED" 20% from the top in 32 px text
local w, h = cathack.screen_size()
cathack.draw_text_centered(w*0.5, h*0.2, "PAUSED", 0xFFFFFFFF, 32)
```

---

### Fancy primitives (gradients / shadows / glow)

#### `cathack.draw_rect_gradient(x, y, w, h, c1, c2[, vertical])`

Two-color filled rectangle. `vertical=true` (default) puts `c1` on top, `c2` on bottom; `false` runs the gradient horizontally.

```lua
cathack.draw_rect_gradient(20, 20, 200, 60,
  0xFF202020, 0xFF000000, true)
```

#### `cathack.draw_shadow_rect(x, y, w, h[, color, blur, rounding])`

Soft drop-shadow drawn as a stack of expanding rounded rects with falling alpha.

| Param | Default | Meaning |
|---|---|---|
| `color` | `0xC0000000` | Inner shadow color (alpha is the *peak*) |
| `blur` | `6` | Pixels the shadow expands outward; controls softness |
| `rounding` | `0` | Corner radius |

#### `cathack.draw_glow_circle(x, y, radius[, color, strength])`

Concentric circles fading outward — gives a glow halo.

| Param | Default | Meaning |
|---|---|---|
| `color` | `0xFFFFFFFF` | Glow color |
| `strength` | `1.0` | How far the glow extends past `radius` (multiplies expansion) |

```lua
cathack.draw_glow_circle(cx, cy, 8, 0xFF80FF80, 1.5)
cathack.draw_circle_filled(cx, cy, 4, 0xFFFFFFFF)
```

---

### Textures and images

Load PNG / JPG / BMP / GIF / TIFF / WebP files from the script directory and draw them.

#### `cathack.load_texture(name) -> id | nil`

Loads `%APPDATA%\cathack\lua\<name>` and returns an integer handle. Pass the handle to `draw_image` / `draw_image_rounded`. Returns `nil` if the file is missing or invalid.

> **Render-thread only** — the underlying D3D11 calls aren't safe from `on_tick`. Always call `load_texture` from `on_render` (or from the chunk top-level if your script runs from the **Run** button while the menu is open). Calling from `on_tick` returns `nil`.

The path is sandboxed to the lua directory: filenames may contain `/` for subfolders (e.g. `assets/logo.png`) but not `\` or `..`.

#### `cathack.free_texture(id) -> boolean`

Releases the GPU resource. Returns `true` if the id existed.

> The VM also frees every texture when you click **Reset VM** or when the cheat unloads, so you don't have to be perfect about cleanup.

#### `cathack.texture_size(id) -> w, h`

Pixel dimensions, or `nil` for an unknown id.

#### `cathack.draw_image(id, x, y, w, h[, color])`

Draws the texture stretched to the given rect. `color` is multiplicative (`0xFFFFFFFF` = unmodified).

#### `cathack.draw_image_rounded(id, x, y, w, h[, rounding, color])`

Same as `draw_image` but with rounded corners.

```lua
local logo
cathack.on_render(function()
  if not logo then logo = cathack.load_texture("assets/logo.png") end
  if logo then cathack.draw_image_rounded(logo, 20, 20, 64, 64, 8) end
end)
```

---

### Clipping & draw layers

#### `cathack.push_clip_rect(x, y, w, h[, intersect])`

Constrains every subsequent `cathack.draw_*` call to the given rect. `intersect=true` (default) intersects with the existing clip rect, so nested clips combine; `false` replaces the active clip.

#### `cathack.pop_clip_rect()`

Pops the most recent push. Pairs must match. Forgotten pops are auto-cleaned at the end of each `on_render` callback so they don't leak into the host's draw list.

#### `cathack.set_draw_layer(layer)`

Switches the draw list every subsequent `cathack.draw_*` call writes into. Resets to the default at the start of every `on_render` callback. Valid layers:

| Value | Meaning |
|---|---|
| `"background"` / `"bg"` | Behind the cheat menu (default — drawings sit between the game and the menu chrome). |
| `"foreground"` / `"fg"` / `"overlay"` / `"menu"` | Above the cheat menu (drawings sit on top of everything ImGui renders). |

```lua
cathack.on_render(function()
  -- Two-pane scroll list: clip everything to a 200x300 box.
  cathack.push_clip_rect(40, 40, 200, 300)
  for i = 1, 100 do
    cathack.draw_text(48, 48 + i * 18 - scroll, "Item " .. i)
  end
  cathack.pop_clip_rect()
end)
```

---

### Clipboard

#### `cathack.clipboard_get() -> string`

Returns the current text clipboard contents (`""` if empty / non-text).

#### `cathack.clipboard_set(text)`

Sets the clipboard to the given UTF-8 string.

```lua
local cfg = cathack.json_encode({ aimbot = cathack.get("aimbot"), fov = cathack.get("aimbot.fov") })
cathack.clipboard_set(cfg)
cathack.notify("Config copied to clipboard")
```

---

### Theme + animation

#### `cathack.accent_color() -> integer`

The current menu accent color, packed as ABGR (`0xAABBGGRR`). Updates live as the user edits the accent in **Settings**.

#### `cathack.theme() -> table`

Snapshot of the menu's palette. All colors are ABGR-packed integers, ready to drop into `draw_*` calls.

```lua
local t = cathack.theme()
-- t.accent     : current accent
-- t.background : main panel background
-- t.text       : default text color
-- t.border     : 1px border color
-- t.opacity    : 0..1 multiplier the menu uses
```

#### `cathack.lerp_color(c1, c2, t) -> integer`

Linearly interpolates each channel (R, G, B, A) between `c1` and `c2` at `t ∈ [0, 1]`. Result is the same ABGR-packed format.

```lua
local pulse = (math.sin(cathack.time() * 0.005) + 1) * 0.5  -- 0..1
local col = cathack.lerp_color(0xFFFF0000, 0xFFFFFF00, pulse)
```

#### `cathack.ease(type, t) -> number`

Standard easing curves over `t ∈ [0, 1]`. Supported types:

`"linear"`, `"in_quad"`, `"out_quad"`, `"in_out_quad"`, `"in_cubic"`, `"out_cubic"`, `"in_out_cubic"`, `"in_sine"`, `"out_sine"`, `"in_out_sine"`, `"in_back"`, `"out_back"`, `"in_out_back"`, `"in_bounce"`, `"out_bounce"`.

Unknown types fall back to `"linear"`.

#### `cathack.frame_count() -> integer`

Number of render frames the VM has dispatched since DLL load. Increments inside `on_render` (and once before the first `on_render` registration).

---

### Viewport

For correct scaling on multi-monitor / HiDPI setups.

#### `cathack.viewport_pos() -> x, y`

Top-left of the game's main viewport in screen coordinates. Usually `(0, 0)` for fullscreen.

#### `cathack.viewport_size() -> w, h`

Width / height of the viewport. Equivalent to `cathack.screen_size()` for fullscreen but more correct for windowed multi-monitor setups.

#### `cathack.dpi_scale() -> number`

Display framebuffer scale (typically `1.0` on a normal monitor, `1.5` / `2.0` on HiDPI setups). Multiply font sizes by this if you want pixel-perfect text on high-DPI displays.

---

### JSON

Hand-rolled encoder + decoder. Strings, numbers, booleans, `nil`/`null`, arrays, and objects all survive a round-trip. Tables that look like arrays (1-indexed contiguous integer keys 1..N) become JSON arrays; anything else becomes a JSON object.

#### `cathack.json_encode(value) -> string`

Returns the encoded JSON. Raises a Lua error if the value tree contains an unsupported type (e.g. a function). Cycles aren't detected — the watchdog (250 ms / call) will kill an infinite recursion if you hand it a self-referential table.

#### `cathack.json_decode(s) -> value`

Parses a JSON string. Numbers that are exactly representable as integers come back as Lua integers; everything else as floats. Raises a Lua error on parse failure.

```lua
-- Save/load a UI panel position
local function save_panel(p)  cathack.store("ui.panel", cathack.json_encode(p)) end
local function load_panel()
  local raw = cathack.fetch("ui.panel")
  return raw and cathack.json_decode(raw) or { x = 200, y = 200 }
end
```

---

### Persistent storage

Simple key-value store that survives Reset VM and game restarts. Backed by `%APPDATA%\cathack\lua\cathack.store`. Values can be **strings**, **numbers**, or **booleans**.

#### `cathack.store(key, value)`

Sets a key. Pass `nil` for `value` to delete. Strings can't contain tab or newline characters.

#### `cathack.fetch(key) -> string | number | boolean | nil`

Retrieves a previously stored value, or `nil` if the key doesn't exist.

#### `cathack.store_keys() -> array of strings`

Lists all stored keys.

```lua
-- Track total kills across sessions
local k = (cathack.fetch("kills") or 0) + 1
cathack.store("kills", k)
print("total kills:", k)
```

---

### Filesystem

Sandboxed file IO restricted to the script directory.

#### `cathack.script_dir() -> string`

Absolute path to `%APPDATA%\cathack\lua\`.

#### `cathack.read_file(name) -> string | nil`

Reads a single file out of the script directory. The name must be a plain filename — no `..`, no path separators. Returns `nil` if the file doesn't exist.

#### `cathack.write_file(name, contents) -> boolean`

Writes `contents` to `<script_dir>/<name>`. Returns `true` on success. Same name validation as `read_file`.

```lua
cathack.write_file("notes.txt", "hello")
print(cathack.read_file("notes.txt"))   -- "hello"
```

---

### Dynamic execution

#### `cathack.run_string(code, chunk_name?) -> bool [, error_string]`

Compile and execute a Lua chunk inline against the currently-running VM. On success returns `true` (and any values left on the stack by the chunk are discarded). On compile or runtime error returns `(false, error_string)`. Optional `chunk_name` shows up in stack traces (defaults to `"=run_string"`).

```lua
-- Build a binding override at runtime
local snippet = "cathack.set('aimbot.fov', 45)"
local ok, err = cathack.run_string(snippet)
if not ok then print(err) end
```

Useful for loading user-supplied snippets (e.g. via `cathack.read_file` / `cathack.fetch`) without using the file picker.

---

### HTTP

The HTTP module is built on **WinHTTP** with HTTP/2 + TLS 1.2/1.3 enabled and a process-wide connection pool. Repeat calls to the same host avoid the TCP/TLS handshake. All successful responses are cached in memory for 5 minutes by URL — most "load a remote script" patterns end up effectively free after the first call.

The Roblox-style `game:HttpGet(url)` global is also available; under the hood it routes to the same code as `cathack.http_get` so the two share the cache.

#### `cathack.http_get(url, opts?) -> body | nil, err`

Synchronous HTTP GET. Blocks the calling thread (game or render) until the response arrives. Default 5 second timeout, default 16 MB body cap, automatic 4xx/5xx → error conversion.

`opts` is an optional table:
- `timeout_ms` — number, override the receive timeout in milliseconds.
- `no_cache`   — bool, skip the in-memory cache (both lookup and store).

Returns the response body as a Lua string on success. Returns `nil, err` on network failure, parse failure, or non-2xx status (`err` will be `"HTTP 404"` etc.).

```lua
-- Canonical "load remote script" pattern
local code, err = cathack.http_get("https://example.com/util.lua")
if code then
    local fn, lerr = load(code, "@util")
    if fn then fn() else print(lerr) end
else
    print("download failed:", err)
end

-- Force a fresh fetch, no cache
local body = cathack.http_get(url, { no_cache = true, timeout_ms = 10000 })
```

#### `game:HttpGet(url) -> body`  (Roblox-style)

Mirrors Roblox's `game:HttpGet`. Synchronous; **raises a Lua error on failure** instead of returning `nil, err`. Use `pcall` if you don't want the script to crash on a bad URL.

```lua
loadstring(game:HttpGet("https://raw.githubusercontent.com/user/repo/main/lib.lua"))()
```

Both `game:HttpGet` (method form) and `game.HttpGet` (bare-call form) work. A lowercase alias `game.http_get` is also registered for the case-insensitive crowd.

#### `cathack.http_get_async(url, callback, opts?) -> bool`

Non-blocking GET. Returns immediately. The worker runs on a detached thread; the callback fires on the **render thread** during `TickRender`, before any drawing for that frame, so values it sets are visible to `on_render` immediately.

Callback signature: `function(body_or_nil, err_or_nil) end`

```lua
local code = nil
cathack.http_get_async("https://example.com/big.lua", function(body, err)
    if body then code = body else print("err:", err) end
end)

-- Some frames later, code is populated
cathack.on_render(function()
    if code then cathack.draw_text(20, 20, "got " .. #code .. " bytes", 0xFFFFFFFF) end
end)
```

Cache hits return through the same path on the next render frame (no thread spawned).

#### `cathack.http_clear_cache()`

Drops every cached response. Connection pool is left intact (dropping it would just re-cost the next TLS handshake).

---

### ImGui

Build full menus from Lua instead of just flat HUD overlays. Every wrapper is **render-thread only** — calling them from `on_tick` is a no-op (they don't error, they just don't do anything). The expected entry point is always `on_render`:

```lua
cathack.on_render(function()
    if cathack.imgui_begin("My script") then
        if cathack.imgui_button("Hi") then cathack.notify("clicked") end
    end
    cathack.imgui_end()
end)
```

#### Auto-capture and cursor

Mouse and keyboard capture is automatic. As soon as a Lua-built ImGui window is hovered or has an active text field, ImGui sets `io.WantCaptureMouse` / `io.WantCaptureKeyboard` / `io.WantTextInput`, the host's WndProc reads those and swallows the matching Win32 messages, so clicks on a Lua panel never bleed through to the game's shoot/move handlers. No `capture_mouse(true)` calls needed.

The cursor is force-shown for any frame that calls `imgui_begin` so users can see what they're clicking even when the cheat menu is closed. Override per-frame with `cathack.set_cursor_visible(false)` if you want a stealth UI.

#### Bracketing safety

ImGui's begin/end pairs come in two flavors:
- **Unconditional**: `Begin/End`, `BeginChild/EndChild`, `BeginGroup/EndGroup` — always end, even if begin returned `false`.
- **Conditional**: `BeginTabBar/EndTabBar`, `BeginTabItem/EndTabItem`, `BeginTable/EndTable`, `TreeNode/TreePop`, `BeginTooltip/EndTooltip` — only end if begin returned `true`.

The wrapper does the right thing automatically — wrap conditional pairs in an `if`:

```lua
if cathack.imgui_begin_tab_bar("##bar") then
    if cathack.imgui_begin_tab_item("Settings") then
        -- ...
        cathack.imgui_end_tab_item()
    end
    cathack.imgui_end_tab_bar()
end
```

If a script crashes / the watchdog longjmps out mid-tree, the host auto-closes any leftover scopes (windows, child windows, tabs, tables, tooltips, tree nodes, indents, ID stack, style colors, style vars) at the end of the callback. No stale state can leak into the next script's callback or into the host menu.

#### Constants — `cathack.imgui`

Flag enums and color/style indices are exposed as a nested table. Pass these to wrappers that take a flags argument or as the `idx` to `imgui_push_style_color/var`:

- `cathack.imgui.WindowFlags.{NoTitleBar, NoResize, NoMove, NoScrollbar, NoCollapse, AlwaysAutoResize, NoBackground, NoSavedSettings, MenuBar, NoDecoration, NoInputs, ...}`
- `cathack.imgui.InputTextFlags.{Password, ReadOnly, EnterReturnsTrue, CharsDecimal, CharsHexadecimal, CharsUppercase, CharsNoBlank, AutoSelectAll, ...}`
- `cathack.imgui.SelectableFlags.{SpanAllColumns, AllowDoubleClick, Disabled, AllowOverlap, ...}`
- `cathack.imgui.TreeNodeFlags.{Selected, Framed, DefaultOpen, OpenOnArrow, Leaf, Bullet, SpanAvailWidth, ...}`
- `cathack.imgui.TableFlags.{Resizable, Reorderable, Hideable, Sortable, RowBg, Borders, ScrollX, ScrollY, ...}`
- `cathack.imgui.TableColumnFlags.{WidthStretch, WidthFixed, NoResize, NoSort, ...}`
- `cathack.imgui.TabBarFlags.{Reorderable, AutoSelectNewTabs, NoTooltip, ...}`
- `cathack.imgui.TabItemFlags.{UnsavedDocument, SetSelected, NoReorder, Leading, Trailing, ...}`
- `cathack.imgui.Cond.{Always, Once, FirstUseEver, Appearing}` — used by `imgui_set_next_window_pos/size`.
- `cathack.imgui.Dir.{Left, Right, Up, Down}` — used by `imgui_arrow_button`.
- `cathack.imgui.Col.{Text, WindowBg, FrameBg, Button, Header, Tab, ...}` — used by `imgui_push_style_color`.
- `cathack.imgui.StyleVar.{Alpha, WindowPadding, FramePadding, FrameRounding, ItemSpacing, ...}` — used by `imgui_push_style_var`.
- `cathack.imgui.HoveredFlags.{ChildWindows, AllowWhenBlockedByPopup, DelayShort, ...}`

Combine flags with the bitwise OR operator `|` (Lua 5.4): `cathack.imgui.WindowFlags.NoTitleBar | cathack.imgui.WindowFlags.NoResize`.

#### API — windows, child panels, layout

| Function | Returns | Notes |
|---|---|---|
| `imgui_begin(name [, closable] [, flags])` | `visible [, open]` | If `closable=true`, returns a second value with the close-X state. **Always** call `imgui_end` regardless of `visible`. |
| `imgui_end()` | — | |
| `imgui_begin_child(id [, w] [, h] [, border] [, flags])` | `visible` | Always pair with `imgui_end_child`. |
| `imgui_end_child()` | — | |
| `imgui_set_next_window_pos(x, y [, cond])` | — | `cond` from `cathack.imgui.Cond`. |
| `imgui_set_next_window_size(w, h [, cond])` | — | |
| `imgui_same_line([offset] [, spacing])` | — | |
| `imgui_separator()`, `imgui_spacing()`, `imgui_new_line()` | — | |
| `imgui_dummy(w, h)` | — | |
| `imgui_indent([w])`, `imgui_unindent([w])` | — | Auto-unindented at end of frame. |
| `imgui_begin_group()`, `imgui_end_group()` | — | |
| `imgui_get_cursor_pos()` | `x, y` | Window-local cursor. |
| `imgui_set_cursor_pos(x, y)` | — | |
| `imgui_get_content_region_avail()` | `w, h` | |

#### API — text

```lua
cathack.imgui_text("plain text")
cathack.imgui_text_colored(0xFF00FF00, "green")
cathack.imgui_text_disabled("dimmed")
cathack.imgui_text_wrapped("a long line that wraps to the window's width...")
cathack.imgui_bullet_text("a bullet point")
cathack.imgui_label_text("Status", "online")     -- "Status: online" — label on the right
```

#### API — buttons

```lua
local clicked  = cathack.imgui_button("Press me" [, w] [, h])
local clicked2 = cathack.imgui_small_button("compact")
local clicked3 = cathack.imgui_invisible_button("##hidden", w, h)
local clicked4 = cathack.imgui_arrow_button("##back", cathack.imgui.Dir.Left)
```

#### API — toggles and value editors

```lua
local v, changed = cathack.imgui_checkbox("Enable", v)
local v, changed = cathack.imgui_radio_button("Pick me", v)

local n, changed = cathack.imgui_slider_int("count", n, 0, 10 [, "%d"])
local f, changed = cathack.imgui_slider_float("amount", f, 0.0, 1.0 [, "%.2f"])

local n, changed = cathack.imgui_drag_int("count", n [, speed=1] [, lo=0] [, hi=0] [, fmt="%d"])
local f, changed = cathack.imgui_drag_float("amount", f [, speed=1] [, lo=0] [, hi=0] [, fmt="%.3f"])

-- Color in/out is the same 32-bit ABGR packing the rest of the API uses.
local color, changed = cathack.imgui_color_edit("Tint", color [, flags])

local idx, changed   = cathack.imgui_combo("Mode", idx, { "A", "B", "C" })
-- idx is 1-indexed (Lua-style)
```

#### API — text input

```lua
local text, changed = cathack.imgui_input_text("Name", text [, max_len=256] [, flags])
local text, changed = cathack.imgui_input_text_multiline("Body", text [, max_len=4096] [, w=0] [, h=0] [, flags])

-- Common pattern: react only on Enter
local text, submitted = cathack.imgui_input_text("Search", text, 256,
    cathack.imgui.InputTextFlags.EnterReturnsTrue)
if submitted then run_search(text) end
```

#### API — selectable rows, headers, trees

```lua
-- Selectable returns (selected, just_clicked). Pass the current
-- selection state in; the wrapper updates it from the user's click.
local selected, clicked = cathack.imgui_selectable("Row 1", selected)
local selected, clicked = cathack.imgui_selectable("Row 2", selected,
    cathack.imgui.SelectableFlags.SpanAllColumns | cathack.imgui.SelectableFlags.AllowDoubleClick)

-- Collapsing header — no End. Returns true while open.
if cathack.imgui_collapsing_header("Section",
    cathack.imgui.TreeNodeFlags.DefaultOpen) then
    cathack.imgui_text("Stuff inside")
end

-- Tree node — call tree_pop() iff returned true.
if cathack.imgui_tree_node("Foo") then
    if cathack.imgui_tree_node("Bar") then
        cathack.imgui_bullet_text("nested")
        cathack.imgui_tree_pop()
    end
    cathack.imgui_tree_pop()
end
```

#### API — tab bars

```lua
if cathack.imgui_begin_tab_bar("##settings_tabs") then
    if cathack.imgui_begin_tab_item("General") then
        cathack.imgui_text("hi")
        cathack.imgui_end_tab_item()
    end
    if cathack.imgui_begin_tab_item("Advanced") then
        cathack.imgui_text("…")
        cathack.imgui_end_tab_item()
    end
    cathack.imgui_end_tab_bar()
end
```

#### API — tables

```lua
local TF = cathack.imgui.TableFlags
if cathack.imgui_begin_table("##t", 3,
    TF.RowBg | TF.Borders | TF.Resizable, 0, 240) then  -- 240px tall
    cathack.imgui_table_setup_column("Name")
    cathack.imgui_table_setup_column("HP")
    cathack.imgui_table_setup_column("Distance")
    cathack.imgui_table_headers_row()

    for _, e in ipairs(cathack.entities()) do
        cathack.imgui_table_next_row()
        cathack.imgui_table_set_column_index(0)
        cathack.imgui_text(e.name or "?")
        cathack.imgui_table_set_column_index(1)
        cathack.imgui_text(tostring(e.health or "-"))
        cathack.imgui_table_set_column_index(2)
        cathack.imgui_text(string.format("%.1fm", e.distance or 0))
    end
    cathack.imgui_end_table()
end
```

#### API — tooltips

```lua
cathack.imgui_button("Hover me")
if cathack.imgui_is_item_hovered() then
    cathack.imgui_set_tooltip("This does the thing")
end

-- Shorter:
cathack.imgui_button("Hover me 2")
cathack.imgui_set_item_tooltip("This does the thing")

-- Custom-laid-out tooltip:
if cathack.imgui_begin_tooltip() then
    cathack.imgui_text_colored(0xFF80FF80, "green")
    cathack.imgui_text("normal")
    cathack.imgui_end_tooltip()
end
```

#### API — item state queries

```lua
cathack.imgui_button("Edit")
local hovered = cathack.imgui_is_item_hovered()           -- bool
local clicked = cathack.imgui_is_item_clicked(0)          -- mouse button index
local active  = cathack.imgui_is_item_active()
local focused = cathack.imgui_is_item_focused()
local winhov  = cathack.imgui_is_window_hovered(
    cathack.imgui.HoveredFlags.ChildWindows)
```

#### API — style stacks

```lua
local IM = cathack.imgui
cathack.imgui_push_style_color(IM.Col.Button, 0xFF202050)
cathack.imgui_push_style_color(IM.Col.ButtonHovered, 0xFF303080)
cathack.imgui_push_style_var(IM.StyleVar.FrameRounding, 6)
cathack.imgui_push_style_var(IM.StyleVar.FramePadding, 8, 4)   -- ImVec2 form
cathack.imgui_button("Themed")
cathack.imgui_pop_style_var(2)
cathack.imgui_pop_style_color(2)
```

`pop_style_*` defaults to popping 1, accepts a count. Passing more than the script pushed is silently clamped.

#### API — ID stack

```lua
for i, item in ipairs(items) do
    cathack.imgui_push_id(i)
    if cathack.imgui_button("Delete") then table.remove(items, i) end
    cathack.imgui_pop_id()
end
```

ImGui IDs are derived from the label, so duplicate labels in the same window create conflicting widgets. `imgui_push_id(string|int)` namespaces them.

---

### Callbacks

Callback handles are integers (Lua registry references). Hold onto the value `on_tick` / `on_render` returns if you want to remove a single callback later.

#### `cathack.on_tick(fn) -> handle`

Register a function to run on the **game thread**, ~60 Hz. The function receives one argument:

- `dt` — seconds elapsed since the last tick (`number`).

```lua
local handle = cathack.on_tick(function(dt)
  -- dt is in seconds
end)
```

Safe to call:

- `cathack.player_pos`, `cathack.player_health`, `cathack.is_alive`
- `cathack.set` / `cathack.get` / `cathack.settings`
- `cathack.world_to_screen`
- `cathack.notify`, `cathack.print`
- Any pure-Lua code (math, tables, string ops, io)

NOT useful here:

- `cathack.draw_*` — the draw list is only set during render. They no-op.

#### `cathack.on_render(fn) -> handle`

Register a function to run on the **render thread**, every frame. No arguments.

```lua
local handle = cathack.on_render(function()
  cathack.draw_text(20, 20, "render frame")
end)
```

Safe to call: everything `on_tick` can, plus all `cathack.draw_*` primitives.

#### `cathack.untick(handle) -> boolean`

Remove a tick callback by handle. Returns `true` if the handle was actually a tick callback.

#### `cathack.unrender(handle) -> boolean`

Remove a render callback. Returns `true` if the handle was actually a render callback.

#### `cathack.unhook(handle) -> boolean`

Generic remove — checks every callback fan (tick, render, key/mouse/wheel/char events). Returns `true` if it found and removed the handle in any of them.

#### `cathack.clear_callbacks()`

Remove every registered callback at once across all fans. Lua globals / functions / tables your script defined are NOT touched — only the registered hooks.

```lua
cathack.clear_callbacks()
```

#### Event callbacks

Six per-frame event fans. All are dispatched at the start of each render frame, **before** `on_render` callbacks fire, so an `on_render` body can already see the side effects. Each registration returns a handle; pass it to `cathack.unhook` (or the matching unregister) to remove it. `Stop`, `Reset VM`, and `clear_callbacks` all clean them up too.

| Function | Fires when | Args passed to your callback |
|---|---|---|
| `cathack.on_key_pressed(fn)`   | A key transitions from up → down | `(vk)` — Win32 VK |
| `cathack.on_key_released(fn)`  | A key transitions from down → up | `(vk)` |
| `cathack.on_mouse_pressed(fn)` | A mouse button goes down | `(button, x, y)` — same button IDs as `mouse_clicked` |
| `cathack.on_mouse_released(fn)`| A mouse button goes up | `(button, x, y)` |
| `cathack.on_mouse_wheel(fn)`   | The wheel moved this frame | `(dv, dh)` — vertical / horizontal |
| `cathack.on_char(fn)`          | The user typed a printable character | `(char)` — UTF-8 string, one codepoint |

```lua
cathack.on_key_pressed(function(vk)
  if vk == 0x52 then          -- VK_R
    cathack.notify("R pressed")
  end
end)

cathack.on_mouse_pressed(function(btn, x, y)
  if btn == 0 and cathack.mouse_in_rect(panel.x, panel.y, panel.w, 24) then
    panel.dragging = true
    cathack.capture_mouse(true)
  end
end)

cathack.on_char(function(c)
  search_buffer = (search_buffer or "") .. c
end)
```

> **Safety**: Errors raised inside an event callback are caught and logged to the console — they don't kill the VM or stop other listeners.

---

### Object reflection

A reflection layer exposing UClass / FProperty / UFunction metadata to Lua. Every dereference goes through a liveness check so a stale handle returns `nil` instead of crashing.

#### Object handles

```lua
cathack.local_player_controller()  -- object_id | nil
cathack.local_character()          -- object_id | nil
cathack.local_pawn()               -- object_id | nil
cathack.player_camera_manager()    -- object_id | nil
```

Handles are integers (raw `UObject*` cast to `lua_Integer`). They stay valid as long as the underlying object lives — the engine periodically invalidates them on level transitions, possession changes, etc. — so always re-fetch at the start of each render frame instead of caching across reloads.

#### Inspectors

```lua
cathack.object_valid(object_id)  -> bool
cathack.object_name(object_id)   -> string         -- e.g. "PlayerController_0"
cathack.object_class(object_id)  -> string         -- e.g. "ReadyOrNotPlayerController_C"
cathack.object_path(object_id)   -> string         -- full UE path (Class /Game/.../...)
```

#### Property enumeration

```lua
local props = cathack.object_properties(pc)
for _, p in ipairs(props) do
  print(p.name, p.type, p.readable, p.writable, p.owner)
end
```

Each entry is a table:

| Field        | Type    | Notes                                                                                         |
|--------------|---------|-----------------------------------------------------------------------------------------------|
| `name`       | string  | Property name (e.g. `bShowMouseCursor`).                                                      |
| `type`       | string  | Categorized: `bool`, `byte`, `int8`/`int16`/`int`/`int64`, `uint16`/`uint32`/`uint64`, `float`, `double`, `name`, `string`, `text`, `object`, `class`, `enum`, `struct`, `array`, `map`, `set`. Empty string for unsupported. |
| `raw_type`   | string  | Raw FFieldClass name (`BoolProperty`, `IntProperty`, `StructProperty`, …).                    |
| `struct`     | string? | Present only when `type == "struct"`. The struct's name (`Vector`, `Rotator`, etc.).          |
| `offset`     | int     | Offset (bytes) into the object.                                                               |
| `size`       | int     | Element size in bytes.                                                                        |
| `array_dim`  | int     | Static array dimension (`1` for normal scalars).                                              |
| `readable`   | bool    | True when `object_get` can return a meaningful value for this property.                       |
| `writable`   | bool    | True when `object_set` will accept this property type.                                        |
| `owner`      | string  | Class that declared the property (handy when walking class hierarchies).                      |

`cathack.object_find_properties(object_id, query, type_filter?)` returns the same shape, prefiltered by case-insensitive name substring and (optionally) by category string (e.g. `"bool"`, `"int"`, `"struct"`).

#### Function enumeration

```lua
for _, fn in ipairs(cathack.object_functions(pc)) do
  print(fn.name, fn.callable, fn.safe, fn.owner)
  for _, p in ipairs(fn.params) do
    print("  param", p.name, p.type, p.out and "out" or "in")
  end
end
```

Each function entry has:

| Field       | Type     | Notes                                                                                       |
|-------------|----------|---------------------------------------------------------------------------------------------|
| `name`      | string   | Function name.                                                                              |
| `callable`  | bool     | Always `true` — the function exists in the class.                                           |
| `safe`      | bool     | True when every parameter is in the call whitelist (see below).                             |
| `reason`    | string?  | Set when `safe == false` — the rejected param type.                                         |
| `owner`     | string   | Owning class name.                                                                          |
| `params`    | table    | Array of parameter descriptors: `{ name, type, raw_type, struct?, out, ret, is_const }`.    |

`cathack.object_find_functions(object_id, query)` filters by case-insensitive name substring.

#### Reading and writing

```lua
local cur = cathack.object_get(pc, "bShowMouseCursor")  -- bool
local ok, err = cathack.object_set(pc, "bShowMouseCursor", true)
if not ok then print("set failed:", err) end
```

- `object_get` understands every `readable` type — primitives become Lua numbers / bools, `name` and `string` become Lua strings, `Vector`/`Rotator`/`Vector2D` come back as `{ x, y, z }` / `{ pitch, yaw, roll }` / `{ x, y }` tables. Object pointers come back as a child object handle (or `nil` if dead).
- `object_set` writes only `writable` types: bool, all integer / float kinds, enum, `string` (FString), and the three known structs. **Writes to `FName` are refused** because the comparison-index pool is engine-private. Pointer-typed properties are also refused — there's no safe way to construct a UObject pointer from Lua.

#### Calling functions

```lua
-- One-arg call: bool SetActorTickEnabled(bool)
local ok, _ = cathack.object_call(actor, "SetActorTickEnabled", { true })

-- Multi-arg call with a return value
local ok, dist = cathack.object_call(actor, "GetDistanceTo", { other_actor })
```

`object_call` packs the args table positionally into a UE param buffer and invokes `UObject::ProcessEvent`. Strict whitelist:
- Allowed param types: `bool`, every integer / float kind, `name`, `string`, `enum`, and the three structs (`Vector`, `Rotator`, `Vector2D`). Functions taking object pointers, classes, arrays, maps, sets, text, or other structs are refused with `safe = false` and an error string.
- Up to 4 KB of total param size — large struct-heavy functions are refused.
- The actual `ProcessEvent` call is wrapped in a structured-exception guard. A bad function still hangs the game; a *very* bad function returns `false, "ProcessEvent threw"` instead of crashing the DLL.

Returns `ok, value_or_error`:
- On success with a return parm: `true, return_value`.
- On success without a return parm: `true`.
- On failure: `false, "error message"`.

> **Object handles are not stable**. Always re-fetch `local_player_controller()` (or whatever you need) every render frame. Caching a handle into persistent storage between reloads is guaranteed to crash.

---

### PlayerController helpers

Convenience layer on top of the reflection API for the things people typically want from a PlayerController. Pre-resolved to `GVars.PlayerController` so scripts don't have to fetch a handle.

```lua
cathack.pc_mouse_cursor()                                  -- bool
cathack.pc_set_mouse_cursor(true|false)

cathack.pc_input_mode()                                    -- "game" | "ui" | "game_and_ui"
cathack.pc_set_input_mode("game" | "ui" | "game_and_ui")   -- best-effort; sets cursor + click flags

cathack.pc_control_rotation()                              -- pitch, yaw, roll  (3 numbers)
cathack.pc_set_control_rotation(pitch, yaw, roll?)         -- roll defaults to 0

cathack.pc_view_target()                                   -- entity_id | nil
cathack.pc_set_view_target(entity_id_or_nil, blend_time?)  -- nil = clear; blend_time>0 uses linear blend

cathack.pc_project_world_to_screen(x, y, z)                -- screen_x, screen_y | nil
cathack.pc_deproject_screen_to_world(sx, sy)               -- {x,y,z} origin, {x,y,z} direction | nil
```

Notes:
- `pc_input_mode` is a heuristic emulation of UE's `SetInputMode(Game/UI/GameAndUI)`: those static helpers aren't reflected in the Dumper7 SDK, so the helpers flip `bShowMouseCursor` + `bEnableClickEvents` + `bEnableMouseOverEvents` directly. Behavior matches the engine for all common cases.
- `pc_set_view_target` accepts any entity id (from `cathack.entities()`) or any object handle that resolves to an `AActor`. Pass `nil` to revert to "no override" (engine reverts to default).
- `pc_project_world_to_screen` returns `nil` when the point is behind the camera or off-screen.
- `pc_deproject_screen_to_world` is the inverse of project — useful for raycasting from the cursor: feed the returned origin + direction into `cathack.line_trace(...)`.

#### Example: a debug overlay over the player camera

```lua
cathack.on_render(function()
  local pc = cathack.local_player_controller()
  if not pc then return end
  local cur = cathack.object_get(pc, "bShowMouseCursor")
  cathack.draw_text(20, 20, "PC mouse cursor: " .. tostring(cur), cathack.color(1,1,1,1))

  local p, y = cathack.pc_control_rotation()
  cathack.draw_text(20, 38, string.format("control rot: %.1f / %.1f", p, y), cathack.color(1,1,1,1))

  local cam = cathack.player_camera_manager()
  if cam then
    cathack.draw_text(20, 56, "camera: " .. (cathack.object_class(cam) or "?"), cathack.color(1,1,1,1))
  end
end)
```

---

## Full setting binding table

`cathack.set(name, value)` and `cathack.get(name)` understand these names. Range column is the slider clamp.

### Top-level toggles

| Name | Kind | Range | Description |
|---|---|---|---|
| `aimbot`           | bool  |   | Master aimbot enable. |
| `esp`              | bool  |   | Master ESP enable. |
| `silent_aim`       | bool  |   | Silent aim master enable. |
| `trigger_bot`      | bool  |   | Trigger bot master enable. |
| `god_mode`         | bool  |   | God mode master enable. |
| `no_recoil`        | bool  |   | |
| `no_spread`        | bool  |   | |
| `no_sway`          | bool  |   | No weapon sway. |
| `wall_penetration` | bool  |   | |
| `no_clip`          | bool  |   | |
| `speed_enabled`    | bool  |   | |
| `speed`            | float | 0–20 | Movement speed multiplier. |
| `fov`              | float | 30–175 | Game FOV. |
| `bullet_time`      | bool  |   | |

### Aimbot

| Name | Kind | Range |
|---|---|---|
| `aimbot.fov`                | float | 1–180 |
| `aimbot.distance_max`       | float | 0–1000 |
| `aimbot.distance_min`       | float | 0–1000 |
| `aimbot.los`                | bool  | |
| `aimbot.smooth`             | bool  | |
| `aimbot.smoothing`          | float | 1–200 |
| `aimbot.target_civilians`   | bool  | |
| `aimbot.target_dead`        | bool  | |
| `aimbot.target_arrested`    | bool  | |
| `aimbot.target_surrendered` | bool  | |
| `aimbot.target_all`         | bool  | |
| `aimbot.require_key`        | bool  | |

### Silent aim

| Name | Kind | Range |
|---|---|---|
| `silent.fov`                | float | 1–180 |
| `silent.hit_chance`         | float | 0–100 |
| `silent.los`                | bool  | |
| `silent.target_civilians`   | bool  | |
| `silent.target_dead`        | bool  | |
| `silent.target_arrested`    | bool  | |
| `silent.target_surrendered` | bool  | |
| `silent.target_all`         | bool  | |
| `silent.method`             | int   | 0–3 (`0`=hard snap, `1`=soft RLerp, `2`=client gun rotate, `3`=server gun rotate) |

### Chams

| Name | Kind |
|---|---|
| `chams.viewmodel`     | bool |
| `chams.world`         | bool |
| `chams.through_walls` | bool |
| `chams.suspects`      | bool |
| `chams.civilians`     | bool |
| `chams.arrested`      | bool |
| `chams.surrendered`   | bool |
| `chams.dead`          | bool |
| `chams.teammates`     | bool |

### Viewmodel

| Name | Kind | Range |
|---|---|---|
| `viewmodel.enabled`      | bool  | |
| `viewmodel.forward`      | float | (cm) |
| `viewmodel.right`        | float | (cm) |
| `viewmodel.up`           | float | (cm) |
| `viewmodel.pitch`        | float | (deg) |
| `viewmodel.yaw`          | float | (deg) |
| `viewmodel.roll`         | float | (deg) |
| `viewmodel.fov_override` | bool  | |
| `viewmodel.fov`          | float | 1–175 |

### Misc / privacy

| Name | Kind |
|---|---|
| `reticle`              | bool |
| `cross_reticle`        | bool |
| `exclude_from_capture` | bool |

To add more bindings, edit `kSettingBindings` in `ReadyOrNot/CathackLua.cpp`. Each row is `{name, kind, &target}` plus optional clamp range.

---

## Patterns and idioms

### Edge-triggered key press

`cathack.key()` is level-triggered (true while held). For "fire once on press":

```lua
local prev = false
cathack.on_tick(function(dt)
  local now = cathack.key(0x52)        -- R
  if now and not prev then
    cathack.set("aimbot", not cathack.get("aimbot"))
    cathack.notify("aimbot " .. (cathack.get("aimbot") and "ON" or "off"))
  end
  prev = now
end)
```

### Re-runnable script (clean up old callbacks)

Each `Run` of a script that registers a callback adds a NEW one — old ones keep firing. Stash handles in a global so reruns clean themselves up:

```lua
-- @autorun
if _MY_HANDLES then
  for _, h in ipairs(_MY_HANDLES) do cathack.unhook(h) end
end
_MY_HANDLES = {
  cathack.on_tick(function(dt) end),
  cathack.on_render(function() end),
}
```

Or sledgehammer it on every run:

```lua
-- @autorun
cathack.clear_callbacks()
cathack.on_tick(function(dt) end)
cathack.on_render(function() end)
```

(Note: `clear_callbacks` removes EVERY callback, including ones from other scripts. Per-script handle bookkeeping is friendlier when multiple scripts are in play.)

### Rate-limit work inside `on_render`

`on_render` fires every frame (often 144+ Hz). For periodic work use `cathack.time()`:

```lua
local nextMs = 0
cathack.on_render(function()
  local t = cathack.time()
  if t < nextMs then return end
  nextMs = t + 250
  -- runs at ~4 Hz max
end)
```

### Read all settings into a snapshot

```lua
local before = cathack.settings()
cathack.set("aimbot", true)
local after = cathack.settings()
for k, v in pairs(after) do
  if before[k] ~= v then print(k, "changed:", before[k], "→", v) end
end
```

---

## Worked examples

### 1. ESP-style box around the world origin

```lua
-- @autorun
cathack.on_render(function()
  local sx, sy = cathack.world_to_screen(0, 0, 0)
  if not sx then return end
  cathack.draw_rect(sx - 30, sy - 30, 60, 60, 0xFFFF00FF, 2)
  cathack.draw_text(sx + 34, sy - 6, "world origin", 0xFFFFFFFF)
end)
```

### 2. Toggle aimbot with R, low-health warning ring

```lua
-- @autorun
local prev = false

cathack.on_tick(function(dt)
  local now = cathack.key(0x52)
  if now and not prev then
    cathack.set("aimbot", not cathack.get("aimbot"))
  end
  prev = now
end)

cathack.on_render(function()
  local cur, max = cathack.player_health()
  if not cur then return end
  if cur < max * 0.30 then
    local sw, sh = cathack.screen_size()
    local pulse = 1.0 + 0.2 * math.sin(cathack.time() / 200)
    cathack.draw_circle(sw / 2, sh / 2, 80 * pulse, 0xFF0000FF, 64, 2)
  end
end)
```

### 3. HUD with current settings

```lua
-- @autorun
cathack.on_render(function()
  local s = cathack.settings()
  local sw, _ = cathack.screen_size()
  local x = sw - 230
  local y = 30
  cathack.draw_rect_filled(x - 10, y - 8, 220, 110, 0xC0000000, 6)
  cathack.draw_text(x, y,        "AIM:    " .. tostring(s["aimbot"]),       0xFFFFFFFF) y = y + 18
  cathack.draw_text(x, y,        "SILENT: " .. tostring(s["silent_aim"]),   0xFFFFFFFF) y = y + 18
  cathack.draw_text(x, y,        "ESP:    " .. tostring(s["esp"]),          0xFFFFFFFF) y = y + 18
  cathack.draw_text(x, y,        string.format("FOV:    %.1f", s["fov"]),   0xFFFFFFFF) y = y + 18
  cathack.draw_text(x, y,        string.format("HP:     %d/%d",
    select(1, cathack.player_health()) or 0,
    select(2, cathack.player_health()) or 0),                                0xFFFFFFFF)
end)
```

### 4. Auto-disable wallhacks when teammates are nearby (silly but illustrative)

```lua
-- @autorun
cathack.on_tick(function(dt)
  -- Imagine you computed nearby_teammates here…
  local nearby = false
  if nearby and cathack.get("chams.world") then
    cathack.set("chams.world", false)
    cathack.notify("chams off — teammate nearby", true)
  end
end)
```

### 5. Self-cleaning rerunnable script

```lua
-- @autorun
if _ESP_HANDLE then cathack.unrender(_ESP_HANDLE) end

_ESP_HANDLE = cathack.on_render(function()
  cathack.draw_text(20, 20, "esp v2", 0xFFFFFF00)
end)
```

### 6. Entity-aware ESP — boxes, names, distance, visibility tinting

```lua
-- @autorun
if _ESP then cathack.unrender(_ESP) end

local function color_for(e)
  if e.dead then return 0xFF606060 end
  if e.kind == "suspect" then  return 0xFF0000FF end
  if e.kind == "civilian" then return 0xFF00FFFF end
  return 0xFF00FF00
end

_ESP = cathack.on_render(function()
  if not cathack.is_in_game() then return end
  for _, e in ipairs(cathack.entities()) do
    local fx, fy = cathack.world_to_screen(e.pos.x, e.pos.y, e.pos.z)
    local hx, hy = cathack.world_to_screen(e.head.x, e.head.y, e.head.z)
    if fx and hx then
      local h = math.abs(fy - hy)
      local w = h * 0.5
      local c = color_for(e)
      -- dim if not visible
      if not cathack.entity_visible(e.id) then
        c = (c & 0x00FFFFFF) | 0x60000000
      end
      cathack.draw_rect(hx - w / 2, hy, w, h, c, 1.5)
      local label = string.format("%s [%dm]", e.kind, math.floor(e.distance))
      cathack.draw_text(hx + w / 2 + 4, hy, label, c)
    end
  end
end)
```

### 7. Closest-target picker for hotkey usage

```lua
-- @autorun
local prev = false

local function closest_alive_suspect()
  local best, bestDist = nil, math.huge
  for _, e in ipairs(cathack.entities()) do
    if e.alive and e.kind == "suspect" and e.distance < bestDist then
      best, bestDist = e, e.distance
    end
  end
  return best
end

cathack.on_tick(function()
  local now = cathack.key(0x46)   -- F
  if now and not prev then
    local t = closest_alive_suspect()
    if t then
      cathack.notify(string.format("nearest suspect: %dm", math.floor(t.distance)), true)
    else
      cathack.notify("no suspects in range", true)
    end
  end
  prev = now
end)
```

### 8. Rainbow crosshair with HSV cycling

```lua
-- @autorun
cathack.on_render(function()
  if not cathack.is_in_game() then return end
  local sw, sh = cathack.screen_size()
  local hue = (cathack.time() % 4000) / 4000.0
  local c = cathack.color_hsv(hue, 1.0, 1.0)
  cathack.draw_line(sw/2 - 12, sh/2,      sw/2 + 12, sh/2,      c, 1.5)
  cathack.draw_line(sw/2,      sh/2 - 12, sw/2,      sh/2 + 12, c, 1.5)
end)
```

### 9. Persistent stats counter

```lua
-- @autorun
local was_alive = true
cathack.on_tick(function()
  local alive_now = cathack.is_alive()
  if was_alive and not alive_now then
    local d = (cathack.fetch("deaths") or 0) + 1
    cathack.store("deaths", d)
    cathack.notify("deaths: " .. d, true)
  end
  was_alive = alive_now
end)
```

### 10. Full freecam (toggle on F8)

Holds position with `set_camera_pos` / rotation with `set_camera_rot`, reads WASD + Q/E + mouse-look (Shift = sprint, Ctrl = slow). Uses the camera's basis vectors so movement is always relative to where you're looking.

```lua
-- @autorun
local fc = {
  on   = false,
  pos  = nil,    -- {x, y, z}
  rot  = nil,    -- {pitch, yaw, roll}
  speed = 600,   -- cm/s base
  prev_f8 = false,
  prev_mx = nil, prev_my = nil,
}

local function copy3(a, b, c) return {a, b, c} end

cathack.on_tick(function(dt)
  -- Toggle on F8
  local f8 = cathack.key(0x77)
  if f8 and not fc.prev_f8 then
    fc.on = not fc.on
    if fc.on then
      fc.pos = copy3(cathack.camera_pos())
      fc.rot = copy3(cathack.camera_rot())
      cathack.notify("freecam ON", true)
    else
      cathack.clear_camera_overrides()
      cathack.notify("freecam off", true)
    end
  end
  fc.prev_f8 = f8

  if not fc.on or not fc.pos then return end

  -- Mouse look (right mouse button held = look)
  -- WASD/QE always control movement, regardless of mouse state
  local mx, my = (function()
    -- ImGui doesn't expose mouse delta from on_tick; cheap proxy via
    -- keys for arrow keys.
    local dx, dy = 0, 0
    if cathack.key(0x25) then dy = dy - 90 * dt end -- ←
    if cathack.key(0x27) then dy = dy + 90 * dt end -- →
    if cathack.key(0x26) then dx = dx - 60 * dt end -- ↑
    if cathack.key(0x28) then dx = dx + 60 * dt end -- ↓
    return dx, dy
  end)()
  fc.rot[1] = math.max(-89, math.min(89, fc.rot[1] + mx))
  fc.rot[2] = fc.rot[2] + my

  -- Movement: WASD + space/ctrl, with shift sprint
  local sp = fc.speed * (cathack.key(0x10) and 4 or 1) * (cathack.key(0x11) and 0.25 or 1)
  local fx, fy, fz = cathack.camera_forward()
  local rx, ry, rz = cathack.camera_right()

  local mvf, mvr, mvu = 0, 0, 0
  if cathack.key(0x57) then mvf = mvf + 1 end -- W
  if cathack.key(0x53) then mvf = mvf - 1 end -- S
  if cathack.key(0x44) then mvr = mvr + 1 end -- D
  if cathack.key(0x41) then mvr = mvr - 1 end -- A
  if cathack.key(0x20) then mvu = mvu + 1 end -- Space
  if cathack.key(0x45) then mvu = mvu - 1 end -- E

  fc.pos[1] = fc.pos[1] + (fx * mvf + rx * mvr) * sp * dt
  fc.pos[2] = fc.pos[2] + (fy * mvf + ry * mvr) * sp * dt
  fc.pos[3] = fc.pos[3] + (fz * mvf + rz * mvr) * sp * dt + mvu * sp * dt

  cathack.set_camera_pos(fc.pos[1], fc.pos[2], fc.pos[3])
  cathack.set_camera_rot(fc.rot[1], fc.rot[2], fc.rot[3])
end)
```

### 11. Zoom toggle on right mouse (override FOV)

```lua
-- @autorun
cathack.on_tick(function()
  if cathack.key(0x02) then          -- RMB
    cathack.set_camera_fov(25)
  else
    cathack.clear_camera_fov()
  end
end)
```

### 12. Crosshair-targeted line trace + ESP

```lua
-- @autorun
cathack.on_render(function()
  if not cathack.is_in_game() then return end
  local sw, sh = cathack.screen_size()
  local ox, oy, oz, dx, dy, dz = cathack.screen_to_world_ray(sw / 2, sh / 2)
  if not ox then return end
  local r = cathack.line_trace(ox, oy, oz,
    ox + dx * 5000, oy + dy * 5000, oz + dz * 5000)
  if r and r.hit then
    local sx, sy = cathack.world_to_screen(r.x, r.y, r.z)
    if sx then
      cathack.draw_circle(sx, sy, 6, 0xFFFF00FF, 24, 1.5)
      cathack.draw_text(sx + 8, sy - 8,
        string.format("%.0f cm", r.distance), 0xFFFFFFFF)
    end
  end
end)
```

---

### 13. Draggable, clickable UI panel with mouse capture

Demonstrates the new mouse + capture APIs. The panel can be grabbed by its title bar and dragged anywhere; clicks on its button toggle aimbot. Game inputs are correctly suppressed while interacting so clicking the button doesn't also fire your weapon.

```lua
-- @autorun
local panel = { x = 200, y = 200, w = 220, h = 110, drag = nil }

local function draw_button(x, y, w, h, label)
  local hov = cathack.mouse_in_rect(x, y, w, h)
  local theme = cathack.theme()
  local bg = hov and cathack.lerp_color(theme.background, theme.accent, 0.35)
                  or theme.background
  cathack.draw_shadow_rect(x, y, w, h, 0x80000000, 4, 4)
  cathack.draw_rect_filled(x, y, w, h, bg, 4)
  cathack.draw_rect(x, y, w, h, theme.border, 1)
  cathack.draw_text_centered(x + w/2, y + h/2 - 7, label, theme.text)
  if hov then cathack.capture_mouse(true) end
  return hov and cathack.mouse_clicked(0)
end

cathack.on_render(function()
  -- Drag handle on the title bar
  local titleH = 22
  local mx, my = cathack.mouse_pos()
  local hovTitle = cathack.mouse_in_rect(panel.x, panel.y, panel.w, titleH)
  if hovTitle and cathack.mouse_clicked(0) then
    panel.drag = { dx = mx - panel.x, dy = my - panel.y }
    cathack.capture_mouse(true)
  end
  if panel.drag then
    cathack.capture_mouse(true)
    panel.x = mx - panel.drag.dx
    panel.y = my - panel.drag.dy
    if cathack.mouse_released(0) then panel.drag = nil end
  end

  -- Frame
  local theme = cathack.theme()
  cathack.draw_shadow_rect(panel.x, panel.y, panel.w, panel.h, 0xC0000000, 8, 6)
  cathack.draw_rect_filled(panel.x, panel.y, panel.w, panel.h, theme.background, 6)
  cathack.draw_rect_gradient(panel.x, panel.y, panel.w, titleH,
    theme.accent, cathack.lerp_color(theme.accent, theme.background, 0.5), true)
  cathack.draw_rect(panel.x, panel.y, panel.w, panel.h, theme.border, 1)
  cathack.draw_text(panel.x + 10, panel.y + 4, "My Lua Panel", theme.text)

  -- Toggle button
  local on = cathack.get("aimbot")
  if draw_button(panel.x + 14, panel.y + 40,
                 panel.w - 28, 28,
                 on and "Aimbot: ON" or "Aimbot: OFF") then
    cathack.set("aimbot", not on)
  end

  -- Hover/footer text
  cathack.draw_text_right(panel.x + panel.w - 10,
    panel.y + panel.h - 18,
    string.format("frame %d", cathack.frame_count()),
    cathack.lerp_color(theme.text, 0x00000000, 0.5))
end)
```

---

### 14. Clipped scrollable list

Mouse wheel scrolls a list clipped to a fixed area.

```lua
-- @autorun
local items = {}
for i = 1, 200 do items[i] = "Entry " .. i end

local list = { x = 60, y = 60, w = 220, h = 240, scroll = 0 }

cathack.on_render(function()
  if cathack.mouse_in_rect(list.x, list.y, list.w, list.h) then
    cathack.capture_mouse(true)
    list.scroll = list.scroll - cathack.mouse_wheel() * 24
    list.scroll = math.max(0, math.min(list.scroll, #items * 20 - list.h))
  end

  cathack.draw_rect_filled(list.x, list.y, list.w, list.h, 0xFF101010, 4)
  cathack.push_clip_rect(list.x, list.y, list.w, list.h)
  for i, name in ipairs(items) do
    local y = list.y + 4 + (i - 1) * 20 - list.scroll
    cathack.draw_text(list.x + 8, y, name)
  end
  cathack.pop_clip_rect()
  cathack.draw_rect(list.x, list.y, list.w, list.h, 0x40FFFFFF, 1)
end)
```

---

### 15. Search bar that consumes typing (with arrow-key navigation)

Uses `cathack.text_input` so arrows / backspace / delete / home / end / shift-select / Ctrl+A / C / X / V all work, plus auto-repeat from holding a key. Clicking the bar grabs keyboard focus so the player doesn't strafe while typing.

```lua
-- @autorun
local bar = {
  x = 80, y = 80, w = 280, h = 28, focus = false,
  state = { text = "", caret = 0 },
}

cathack.on_render(function()
  local hov = cathack.mouse_in_rect(bar.x, bar.y, bar.w, bar.h)
  if cathack.mouse_clicked(0) then
    bar.focus = hov
    if hov then cathack.capture_mouse(true) end
  end

  if bar.focus then
    cathack.capture_keyboard(true)
    cathack.text_input(bar.state, { max_length = 64 })

    -- Escape unfocuses without committing
    for _, ev in ipairs(cathack.key_events()) do
      if ev.kind == "key_down" and ev.vk == 0x1B then bar.focus = false end
    end
    if bar.state.submit then
      cathack.notify("search: " .. bar.state.text)
      bar.state.text, bar.state.caret = "", 0
    end
  end

  local theme = cathack.theme()
  cathack.draw_rect_filled(bar.x, bar.y, bar.w, bar.h, theme.background, 4)
  cathack.draw_rect(bar.x, bar.y, bar.w, bar.h,
    bar.focus and theme.accent or theme.border, 1)

  -- Selection highlight
  if bar.state.sel_anchor then
    local lo = math.min(bar.state.sel_anchor, bar.state.caret)
    local hi = math.max(bar.state.sel_anchor, bar.state.caret)
    local sx = cathack.measure_text(bar.state.text:sub(1, lo))
    local sw = cathack.measure_text(bar.state.text:sub(lo + 1, hi))
    cathack.draw_rect_filled(bar.x + 8 + sx, bar.y + 6, sw, 14,
      cathack.lerp_color(theme.accent, 0x00000000, 0.6))
  end

  local label = (#bar.state.text == 0 and not bar.focus) and "Search…" or bar.state.text
  cathack.draw_text(bar.x + 8, bar.y + 6, label,
    (#bar.state.text == 0 and not bar.focus) and 0x80FFFFFF or theme.text)

  -- Caret
  if bar.focus and (cathack.frame_count() % 60 < 30) then
    local cw = cathack.measure_text(bar.state.text:sub(1, bar.state.caret))
    cathack.draw_rect_filled(bar.x + 8 + cw, bar.y + 6, 2, 14, theme.text)
  end
end)
```

---

### 16. Full ImGui-driven settings panel

A real menu built entirely from Lua — windows, tabs, tables, sliders, color editor, tooltip, ID stack. Auto-captures input while hovered, no manual `capture_*` plumbing.

```lua
-- @autorun
local IM = cathack.imgui
local state = {
    open       = true,
    aim_fov    = 30.0,
    accent     = 0xFFE54AC2,
    target     = 1,
    notify_on  = true,
    note       = "",
}

cathack.on_render(function()
    if not state.open then return end

    cathack.imgui_set_next_window_size(420, 380, IM.Cond.FirstUseEver)
    cathack.imgui_set_next_window_pos (40, 60, IM.Cond.FirstUseEver)

    cathack.imgui_push_style_var(IM.StyleVar.WindowRounding, 6)
    cathack.imgui_push_style_var(IM.StyleVar.FrameRounding,  4)
    cathack.imgui_push_style_color(IM.Col.WindowBg, 0xE0151515)
    cathack.imgui_push_style_color(IM.Col.Header,   state.accent)

    local visible, open = cathack.imgui_begin("Cathack — script panel", true)
    state.open = open
    if visible and cathack.imgui_begin_tab_bar("##bar") then
        if cathack.imgui_begin_tab_item("Aim") then
            state.aim_fov = cathack.imgui_slider_float("FOV", state.aim_fov, 1, 180, "%.1f deg")
            cathack.imgui_set_item_tooltip("Max angle from crosshair the aim assist will pull to.")

            state.target = cathack.imgui_combo("Targets", state.target,
                { "Suspects", "Civilians", "Both" })
            cathack.imgui_end_tab_item()
        end

        if cathack.imgui_begin_tab_item("Theme") then
            state.accent = cathack.imgui_color_edit("Accent", state.accent)
            cathack.imgui_end_tab_item()
        end

        if cathack.imgui_begin_tab_item("Entities") then
            local TF = IM.TableFlags
            if cathack.imgui_begin_table("##ent", 3,
                TF.RowBg | TF.Borders | TF.ScrollY | TF.Resizable, 0, 200) then
                cathack.imgui_table_setup_column("Name")
                cathack.imgui_table_setup_column("HP",  IM.TableColumnFlags.WidthFixed, 60)
                cathack.imgui_table_setup_column("Distance",
                    IM.TableColumnFlags.WidthFixed, 90)
                cathack.imgui_table_headers_row()

                for i, e in ipairs(cathack.entities()) do
                    cathack.imgui_push_id(i)
                    cathack.imgui_table_next_row()
                    cathack.imgui_table_set_column_index(0)
                    cathack.imgui_text(e.name or "?")
                    cathack.imgui_table_set_column_index(1)
                    cathack.imgui_text(tostring(e.health or "-"))
                    cathack.imgui_table_set_column_index(2)
                    cathack.imgui_text(string.format("%.1fm", e.distance or 0))
                    cathack.imgui_pop_id()
                end
                cathack.imgui_end_table()
            end
            cathack.imgui_end_tab_item()
        end

        if cathack.imgui_begin_tab_item("Notes") then
            state.notify_on = cathack.imgui_checkbox("Notify on submit", state.notify_on)
            local submitted
            state.note, submitted = cathack.imgui_input_text("Note", state.note, 256,
                IM.InputTextFlags.EnterReturnsTrue)
            if submitted and state.notify_on then
                cathack.notify("Note: " .. state.note)
                state.note = ""
            end
            cathack.imgui_end_tab_item()
        end

        cathack.imgui_end_tab_bar()
    end
    cathack.imgui_end()

    cathack.imgui_pop_style_color(2)
    cathack.imgui_pop_style_var(2)
end)
```

---

### 17. Loadstring from a remote URL

Canonical "fetch and run a remote Lua file" pattern. The first call pays for the TLS handshake; subsequent calls hit the in-memory cache for ~free.

```lua
-- Roblox-style — raises on failure, simplest to read
loadstring(game:HttpGet("https://raw.githubusercontent.com/user/repo/main/lib.lua"))()

-- Defensive form — no errors, you decide what to do
local code, err = cathack.http_get("https://example.com/util.lua")
if not code then
    cathack.notify("download failed: " .. err)
    return
end

local fn, lerr = load(code, "@util")
if not fn then
    cathack.notify("compile failed: " .. lerr)
    return
end

local ok, run_err = pcall(fn)
if not ok then cathack.notify("run failed: " .. run_err) end
```

For a non-blocking variant (won't stall the frame on a cold connection):

```lua
cathack.http_get_async("https://example.com/util.lua", function(body, err)
    if not body then return cathack.notify(err) end
    local fn = load(body, "@util")
    if fn then pcall(fn) end
end)
```

---

## Limitations and gotchas

- **`Stop` now unregisters callbacks.** Stop sweeps every `on_tick` / `on_render` callback the script's last `Run` registered (tagged at registration time with the running script's name) and frees them. Lua globals / closures the script defined in the VM are not touched — those stay until you click **Reset VM** or another script reassigns them.

- **`Run` auto-stops first.** Re-running a script no longer accumulates duplicate callbacks. Each `Run` calls Stop on the same script first, so `cathack.on_tick(fn)` registered N times across N runs ends up with one live callback, not N.

- **Watchdog: 250 ms per call.** Every chunk execution and every callback invocation has a wall-clock budget (default 250 ms, see `kCallBudgetMs` in `CathackLua.cpp`). If a call exceeds it, the VM raises a Lua error with `call exceeded 250ms wall-clock budget — aborted to keep the game responsive` and longjmps out. The error lands in the console; the VM keeps running. Practical effect: a `while true do end` no longer freezes the game.

- **Persistent storage survives Reset VM.** `cathack.store` / `cathack.fetch` write to disk. Resetting the VM does NOT clear `cathack.store` (intentional — that's the whole point).

- **Reset VM clears callbacks AND globals.** Reset rebuilds the lua_State from scratch. Anything you stored in Lua globals is gone. Auto-run scripts re-fire after Reset.

- **No coroutine yielding from callbacks.** `on_tick` / `on_render` are called via `lua_pcall` with no continuation — `coroutine.yield` from inside them won't suspend the way you might expect. Use `cathack.time()` for time-based logic.

- **Drawing uses pixel coordinates** (top-left origin). `screen_size()` gives you the bounds. Coordinates outside the screen are clipped by ImGui automatically.

- **Float clamping.** `cathack.set("aimbot.fov", 9999)` silently clamps to 180 (the slider's max). The bind table's range column shows the clamps.

- **Concurrency.** All Lua execution is serialized behind a single mutex. A long callback in `on_render` will stall `on_tick` (and vice versa), which causes hitches. Keep callbacks short — and now if you can't, the watchdog will cut you off.

- **Errors don't kill the VM.** Any error inside a chunk run, tick callback, or render callback is caught, logged to the console, and execution moves on. The state is fine to keep using. If a script is in a bad state, **Reset VM** is always the bailout.

- **`io.*` is on.** A script can write/read anywhere your user can. That's intentional (you own these scripts) but worth knowing if you're sharing scripts from the internet — read them before you Run them.

- **`os.*` and `require`/`package` are off.** A script can't `os.execute("calc")`, can't `require("mylib")`. If you need a few of those, `linit.c` and `loslib.c` are the file-level changes.

- **Mouse / keyboard capture is per-frame.** `cathack.capture_mouse(true)` only applies to the **next** message after the call, and the flag clears at the start of every render frame. Scripts that want a UI to keep eating clicks must call `capture_mouse(true)` every frame. This matches ImGui's `SetNextFrameWantCaptureMouse` model and keeps a crashing script from leaving the game permanently input-locked.

- **Texture loading is render-thread only.** `cathack.load_texture` calls D3D11 directly on the immediate context. Calling it from `on_tick` is unsafe, so the binding short-circuits to `nil`. Always load textures from the chunk top-level or inside `on_render` (typically guarded with `if not tex then tex = cathack.load_texture(...) end`).

- **Event callbacks fire before `on_render`.** `on_key_pressed` / `on_mouse_*` / `on_char` all dispatch at the top of the render frame. Side effects they make to script-side state (e.g. appending to `search_buffer`) are visible to `on_render` callbacks during the same frame.

- **`on_key_pressed` doesn't auto-repeat — `key_events()` does.** The callback hook only fires on a discrete key transition, the same as `cathack.key_pressed(vk, false)`. For text editing where holding the arrow / backspace should repeat at the OS keyboard rate, either pass `repeat = true` to the polling form or read `cathack.key_events()` (which carries the repeat flag) — that's what `cathack.text_input` does internally.

- **`chars_typed()` includes control characters.** Backspace arrives as `0x08`, Enter as `0x0D`, etc. The classic foot-gun is `state.text = state.text .. cathack.chars_typed()` which appends the raw control byte. Either use `chars_typed_printable()` or `text_input(state)` — the helper reads navigation keys from the unified event queue and never interleaves them as raw bytes.

- **JSON is best-effort.** `json_encode` rejects functions / userdata / threads with a Lua error, doesn't detect cycles (the watchdog catches infinite recursion), and treats sparse-array tables as objects. Round-tripping numbers preserves integers when they're representable as such.

- **Object handles are not stable.** The integer returned from `local_player_controller()` / `object_get(obj, "SomeObjectProp")` / `pc_view_target()` is a raw `UObject*`. The engine garbage-collects and re-allocates these on level transitions, possession changes, view-target swaps, etc. **Always re-fetch handles every render frame** — do not cache them across reloads or even across multiple frames if you want robustness.

- **Reflection writes are bounded.** `object_set` refuses pointer types, `FName` writes, and any struct other than `Vector`/`Rotator`/`Vector2D`. `object_call` refuses functions whose params include object pointers, classes, arrays/maps/sets, text, or unknown structs. The whitelist exists so a malformed write can't corrupt UE's GC roots.

- **`object_call` runs on whichever thread invoked it.** From `on_tick` it runs on the game thread (safe for most UE calls). From `on_render` it runs on the render thread (still serialized through the Lua mutex but **not** safe for every UE call — engine code that touches actor lifetimes generally expects the game thread). When in doubt, do reflection work in `on_tick`.

- **`http_get` and `game:HttpGet` block.** The synchronous form holds the Lua mutex *and* the calling thread (game or render) for the entire round-trip. With cache hits this is microseconds; with a cold TLS connection to a slow CDN it can be a couple of hundred ms — long enough to register as a frame-time spike. Use `http_get_async` if you have a non-blocking path. The 5 s default timeout caps the worst case.

- **HTTP cache is in-memory only.** `cathack.store`/`fetch` is the path for cross-session persistence. The HTTP cache lives until the DLL unloads (or `cathack.http_clear_cache()`), and cache hits don't re-validate with the server — pass `no_cache = true` in opts to bypass. Failures are not cached.

- **`game:HttpGet` raises on non-2xx.** Roblox-style behavior: a 404 / network error is a Lua error, not a `(nil, err)` return. Wrap in `pcall` if you don't want script death on a bad URL. The `cathack.http_get` form returns `nil, err` instead and is generally easier to handle.

---

## Cheat sheet

```lua
-- Settings
cathack.get(name)
cathack.set(name, value)
cathack.settings() -> table
cathack.bindings() -> array of name strings

-- Local player
cathack.player_pos() -> x, y, z | nil
cathack.player_velocity() -> vx, vy, vz | nil
cathack.player_speed() -> n | nil
cathack.player_aim_rot() -> p, y, r | nil
cathack.player_health() -> cur, max | nil
cathack.player_team() -> string | nil
cathack.player_name() -> string | nil
cathack.is_alive() -> bool
cathack.local_id() -> integer | nil

-- Camera (read)
cathack.camera_pos() -> x, y, z | nil
cathack.camera_rot() -> p, y, r | nil
cathack.camera_forward() -> x, y, z | nil
cathack.camera_right() -> x, y, z | nil
cathack.camera_up() -> x, y, z | nil
cathack.camera_fov() -> n | nil
cathack.camera_aspect() -> n | nil
cathack.camera_near_clip() -> n | nil
cathack.camera_velocity() -> vx, vy, vz
cathack.camera_speed() -> n
cathack.camera_view_target() -> entity_id | nil
cathack.camera_post_process_blend() -> n | nil
-- Camera (transforms)
cathack.screen_to_world_ray(sx, sy) -> ox, oy, oz, dx, dy, dz | nil
cathack.world_to_camera_relative(wx, wy, wz) -> fwd, right, up | nil
cathack.camera_to_world(fwd, right, up) -> wx, wy, wz | nil
-- Camera (overrides — reapplied each tick/frame)
cathack.set_camera_pos(x, y, z) / cathack.clear_camera_pos()
cathack.set_camera_rot(p, y, r?) / cathack.clear_camera_rot()
cathack.set_camera_fov(fov) / cathack.clear_camera_fov()
cathack.clear_camera_overrides()
-- Camera (spectate / effects)
cathack.set_view_target(entity_id|nil, blend_time?)
cathack.camera_fade(from, to, dur, color?, fade_audio?, hold?)
cathack.camera_fade_stop()
cathack.camera_set_manual_fade(amount, color?, fade_audio?)
cathack.camera_stop_shakes(immediate?)

-- Engine / session
cathack.is_in_game() -> bool
cathack.world_name() -> string | nil

-- Entities
cathack.entities() -> array of records
cathack.entity_count() -> integer
cathack.entity_pos(id) -> x, y, z | nil
cathack.entity_velocity(id) -> vx, vy, vz | nil
cathack.entity_speed(id) -> n | nil
cathack.entity_rotation(id) -> p, y, r | nil
cathack.entity_aim_rot(id) -> p, y, r | nil
cathack.entity_health(id) -> cur, max | nil
cathack.entity_kind(id) -> string | nil
cathack.entity_alive(id) -> bool
cathack.entity_distance(id) -> meters | nil
cathack.entity_name(id) -> string | nil
cathack.entity_bone(id, name|index) -> x, y, z | nil
cathack.entity_screen_pos(id, "feet"|"head") -> sx, sy, ok | nil
cathack.entity_visible(id) -> bool

-- Raycast
cathack.line_of_sight(x1,y1,z1, x2,y2,z2) -> bool | nil
cathack.line_trace(x1,y1,z1, x2,y2,z2) -> {hit, x, y, z, distance, entity_id?} | nil

-- Math / vectors
cathack.distance(x1,y1,z1, x2,y2,z2) -> n
cathack.normalize(x, y, z) -> nx, ny, nz
cathack.dot(x1,y1,z1, x2,y2,z2) -> n
cathack.cross(x1,y1,z1, x2,y2,z2) -> nx, ny, nz
cathack.lerp(a, b, t) -> n

-- Color helpers
cathack.color(r, g, b, a?) -> integer        -- r/g/b/a are 0..255
cathack.color_hsv(h, s, v, a?) -> integer    -- h/s/v are 0..1; a is 0..255

-- Screen / projection
cathack.screen_size() -> w, h
cathack.world_to_screen(x, y, z) -> sx, sy, ok | nil

-- Input / time
cathack.key(vk) -> bool
cathack.key_pressed(vk[, repeat]) -> bool
cathack.key_released(vk) -> bool
cathack.chars_typed() -> string                            -- raw, includes 0x08/etc.
cathack.chars_typed_printable() -> string                  -- same but C0/DEL stripped
cathack.input_events() -> array of {kind,codepoint|vk,text?,repeated,shift,ctrl,alt}
cathack.key_events() -> array (same shape, key_down/key_up only — auto-repeat included)
cathack.text_input(state[, opts]) -> state                 -- full text editing, see below
cathack.time() -> ms
cathack.delta_time() -> seconds
cathack.fps() -> n
cathack.frame_count() -> integer

-- Mouse / cursor / capture
cathack.mouse_pos() -> x, y
cathack.mouse_delta() -> dx, dy
cathack.mouse_down(button) -> bool
cathack.mouse_clicked(button[, repeat]) -> bool
cathack.mouse_released(button) -> bool
cathack.mouse_double_clicked(button) -> bool
cathack.mouse_wheel() -> dv, dh
cathack.mouse_drag_delta(button[, threshold]) -> dx, dy
cathack.mouse_in_rect(x, y, w, h) -> bool
cathack.set_cursor_visible(bool) / cathack.cursor_visible() -> bool
cathack.capture_mouse([bool])    / cathack.want_capture_mouse() -> bool
cathack.capture_keyboard([bool]) / cathack.want_capture_keyboard() -> bool

-- Cheat menu
cathack.menu_open() -> bool
cathack.set_menu_open(bool)
cathack.toggle_menu()

-- Notifications / output
cathack.print(...)
cathack.notify(msg, force?)

-- Drawing (only inside on_render)
cathack.draw_text(x, y, text, color?)
cathack.draw_rect(x, y, w, h, color?, thickness?)
cathack.draw_rect_filled(x, y, w, h, color?, rounding?)
cathack.draw_line(x1, y1, x2, y2, color?, thickness?)
cathack.draw_circle(cx, cy, r, color?, segments?, thickness?)
cathack.draw_circle_filled(cx, cy, r, color?, segments?)
cathack.draw_triangle(x1,y1, x2,y2, x3,y3, color?, thickness?)
cathack.draw_triangle_filled(x1,y1, x2,y2, x3,y3, color?)
cathack.draw_quad(x1,y1, x2,y2, x3,y3, x4,y4, color?, thickness?)
cathack.draw_quad_filled(x1,y1, x2,y2, x3,y3, x4,y4, color?)
cathack.draw_polyline({{x,y},...}, color?, thickness?, closed?)
cathack.draw_bezier(x1,y1, x2,y2, x3,y3, x4,y4, color?, thickness?, segments?)
cathack.measure_text(text) -> w, h
cathack.world_text(x, y, z, text, color?, offset_y?) -> sx, sy, ok | nil

-- Sized & aligned text
cathack.draw_text_sized(x, y, text, color?, size?)
cathack.draw_text_centered(x, y, text, color?, size?)
cathack.draw_text_right(x, y, text, color?, size?)
cathack.measure_text_sized(text, size?) -> w, h

-- Fancy primitives
cathack.draw_rect_gradient(x, y, w, h, c1, c2, vertical?)
cathack.draw_shadow_rect(x, y, w, h, color?, blur?, rounding?)
cathack.draw_glow_circle(cx, cy, r, color?, strength?)

-- Textures (render-thread only)
local id = cathack.load_texture("logo.png")
cathack.draw_image(id, x, y, w, h, color?)
cathack.draw_image_rounded(id, x, y, w, h, rounding?, color?)
cathack.texture_size(id) -> w, h
cathack.free_texture(id)

-- Clipping & layers
cathack.push_clip_rect(x, y, w, h, intersect?)
cathack.pop_clip_rect()
cathack.set_draw_layer("foreground"|"background")

-- Clipboard
cathack.clipboard_get() -> string
cathack.clipboard_set(text)

-- Theme + animation
cathack.accent_color() -> integer
cathack.theme() -> { accent, background, text, border, opacity }
cathack.lerp_color(c1, c2, t) -> integer
cathack.ease(type, t) -> number

-- Viewport
cathack.viewport_pos() -> x, y
cathack.viewport_size() -> w, h
cathack.dpi_scale() -> number

-- JSON
cathack.json_encode(value) -> string
cathack.json_decode(string) -> value

-- Persistent storage
cathack.store(key, value)        -- value: string|number|bool, nil to delete
cathack.fetch(key) -> value | nil
cathack.store_keys() -> array of strings

-- Filesystem
cathack.script_dir() -> string
cathack.read_file(name) -> string | nil
cathack.write_file(name, contents) -> bool

-- Dynamic execution
cathack.run_string(code, chunk_name?) -> bool [, err]

-- HTTP
cathack.http_get(url, opts?) -> body | nil, err          -- sync GET, 5s default timeout, 5min cache
cathack.http_get_async(url, callback, opts?) -> bool      -- async; callback(body, err) on render thread
cathack.http_clear_cache()                                -- drop cached responses
game:HttpGet(url) -> body                                 -- Roblox-style sync GET (errors on failure)
-- opts table: { timeout_ms = 5000, no_cache = false }

-- ImGui (render-thread only — call from on_render)
cathack.imgui_begin(name [, closable] [, flags])           -- always call imgui_end
cathack.imgui_end()
cathack.imgui_begin_child(id [, w] [, h] [, border] [, flags])
cathack.imgui_end_child()
cathack.imgui_set_next_window_pos(x, y [, cond])
cathack.imgui_set_next_window_size(w, h [, cond])
cathack.imgui_same_line([offset] [, spacing])
cathack.imgui_separator() / spacing() / new_line() / dummy(w, h)
cathack.imgui_indent([w]) / unindent([w])
cathack.imgui_begin_group() / end_group()
cathack.imgui_get_cursor_pos() -> x, y         /  set_cursor_pos(x, y)
cathack.imgui_get_content_region_avail() -> w, h
cathack.imgui_text(s) / text_colored(c, s) / text_disabled(s) / text_wrapped(s)
cathack.imgui_bullet_text(s) / label_text(label, value)
cathack.imgui_button(label [, w] [, h]) -> clicked
cathack.imgui_small_button(label) / invisible_button(id, w, h) / arrow_button(id, dir) -> clicked
cathack.imgui_checkbox(label, value) -> value, changed
cathack.imgui_radio_button(label, active) -> active, clicked
cathack.imgui_slider_int(label, v, lo, hi [, fmt]) -> v, changed
cathack.imgui_slider_float(label, v, lo, hi [, fmt]) -> v, changed
cathack.imgui_drag_int  (label, v [, speed] [, lo] [, hi] [, fmt]) -> v, changed
cathack.imgui_drag_float(label, v [, speed] [, lo] [, hi] [, fmt]) -> v, changed
cathack.imgui_color_edit(label, color [, flags]) -> color, changed
cathack.imgui_combo(label, idx, items_table) -> idx (1-based), changed
cathack.imgui_input_text(label, text [, max_len] [, flags]) -> text, changed
cathack.imgui_input_text_multiline(label, text [, max_len] [, w] [, h] [, flags]) -> text, changed
cathack.imgui_selectable(label [, selected] [, flags] [, w] [, h]) -> selected, clicked
cathack.imgui_collapsing_header(label [, flags]) -> open
cathack.imgui_tree_node(label [, flags]) -> open / tree_pop()
cathack.imgui_begin_tab_bar(id [, flags]) / end_tab_bar()
cathack.imgui_begin_tab_item(label [, flags]) / end_tab_item()
cathack.imgui_begin_table(id, columns [, flags] [, w] [, h]) / end_table()
cathack.imgui_table_setup_column(label [, flags] [, width])
cathack.imgui_table_headers_row()
cathack.imgui_table_next_row([row_flags] [, min_height])
cathack.imgui_table_next_column() / table_set_column_index(idx)
cathack.imgui_set_tooltip(s) / set_item_tooltip(s)
cathack.imgui_begin_tooltip() / end_tooltip()
cathack.imgui_is_item_hovered([flags]) / is_item_clicked([btn]) / is_item_active() / is_item_focused()
cathack.imgui_is_window_hovered([flags])
cathack.imgui_push_style_color(idx, color) / pop_style_color([count])
cathack.imgui_push_style_var(idx, x [, y]) / pop_style_var([count])
cathack.imgui_push_id(string|int) / pop_id()
-- Constants: cathack.imgui.{ WindowFlags, InputTextFlags, SelectableFlags,
--   TreeNodeFlags, TableFlags, TableColumnFlags, TabBarFlags, TabItemFlags,
--   Cond, Dir, Col, StyleVar, HoveredFlags }

-- Callbacks (per-frame ticks)
local h = cathack.on_tick(function(dt) end)
local h = cathack.on_render(function() end)
cathack.untick(h)
cathack.unrender(h)
cathack.unhook(h)
cathack.clear_callbacks()

-- Callbacks (events)
cathack.on_key_pressed(function(vk) end)
cathack.on_key_released(function(vk) end)
cathack.on_mouse_pressed(function(button, x, y) end)
cathack.on_mouse_released(function(button, x, y) end)
cathack.on_mouse_wheel(function(dv, dh) end)
cathack.on_char(function(s) end)

-- Object reflection
cathack.local_player_controller() -> object_id | nil
cathack.local_character() -> object_id | nil
cathack.local_pawn() -> object_id | nil
cathack.player_camera_manager() -> object_id | nil
cathack.object_valid(id) -> bool
cathack.object_name(id) -> string
cathack.object_class(id) -> string
cathack.object_path(id) -> string
cathack.object_properties(id) -> array of { name, type, raw_type, struct?, offset, size, array_dim, readable, writable, owner }
cathack.object_functions(id) -> array of { name, callable, safe, reason?, owner, params }
cathack.object_find_properties(id, query, type_filter?) -> array
cathack.object_find_functions(id, query) -> array
cathack.object_get(id, prop_name) -> value | nil
cathack.object_set(id, prop_name, value) -> bool, err?
cathack.object_call(id, fn_name, args_table?) -> ok, result_or_error

-- PlayerController helpers
cathack.pc_mouse_cursor() -> bool
cathack.pc_set_mouse_cursor(bool)
cathack.pc_input_mode() -> "game" | "ui" | "game_and_ui"
cathack.pc_set_input_mode("game" | "ui" | "game_and_ui")
cathack.pc_control_rotation() -> p, y, r
cathack.pc_set_control_rotation(p, y, r?)
cathack.pc_view_target() -> entity_id | nil
cathack.pc_set_view_target(entity_id_or_nil, blend_time?)
cathack.pc_project_world_to_screen(x, y, z) -> sx, sy | nil
cathack.pc_deproject_screen_to_world(sx, sy) -> {x,y,z}, {x,y,z} | nil
```
