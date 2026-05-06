# NEON LEAP — Project Handover

A synthwave-themed browser platformer in two versions: a complete 2D tile-based platformer and a single-level 3D Mario 64-style follow-up. Both run as single self-contained HTML files with no build step.

---

## Files

| File | Description | Size |
|---|---|---|
| `neon-leap.html` | 2D platformer, 4 hand-designed levels, save state for unlocked levels and best times. | ~52 KB |
| `neon-leap-3d.html` | 3D platformer, single spiral-tower level, full free-look camera with presets. | ~51 KB |
| `HANDOVER.md` | This document. | — |

Both HTML files are standalone — open in a modern browser (Chrome, Firefox, Safari, Edge), no server needed.

---

## Tech Stack & Constraints

### Shared across both versions
- **Pure HTML / CSS / JS** in one file each, no build pipeline
- **Google Fonts:** Monoton (display), Orbitron (UI), VT323 (terminal/HUD)
- **Web Audio API** for all sound effects — synthesized at runtime, no audio files
- **`window.storage`** API for persistent save state. Do **not** swap to `localStorage` — it's not available in this environment and will throw
- All save/load wrapped in try/catch so the game still works if storage is missing

### 2D-specific
- HTML5 Canvas 2D rendering at 960×540 logical resolution
- Custom-drawn synthwave perspective grid in the background
- Fixed viewport with horizontal camera scrolling

### 3D-specific
- **Three.js r128** loaded from `cdnjs.cloudflare.com`. The version is locked: the artifact environment has issues with newer versions, and `THREE.CapsuleGeometry` (post-r142) must be avoided
- WebGL renderer with antialias, fog (`FogExp2`), grid helpers, ambient + directional lighting
- All glow effects are achieved with bright `MeshBasicMaterial` colors plus `EdgesGeometry` overlays — no post-processing bloom

---

## Game Design

### 2D — Four levels

Each level is a tile-grid character map, 18 rows tall, variable width. The parser walks the grid and emits collision and game objects.

Tile legend:

| Char | Meaning |
|---|---|
| `.` | Empty space |
| `#` | Solid block (walls/ground) |
| `=` | Solid platform (visually thinner accent, mechanically same as `#`) |
| `^` | Spike (must sit directly above a `#` or `=`) |
| `c` | Coin pickup |
| `P` | Player spawn (one per level) |
| `G` | Goal (one per level) |
| `e` | Patrol enemy (must sit directly above a `#` or `=`) |

Levels and their themes:

1. **AWAKENING** (44 wide) — Tutorial. No hazards. Optional aerial platforms with bonus coins.
2. **PULSE** (54 wide) — Introduces spikes on the ground row with 8-tile gaps for safe jump landings.
3. **ASCENT** (38 wide) — Vertical climb, single patrolling enemy on a wide platform.
4. **OVERDRIVE** (66 wide) — Combines spikes, enemies, and aerial coin paths.

### 3D — One level

A spiral tower: spawn ground (18×12), then nine 4×4 platforms rising in a counter-clockwise spiral, ending in a 7×7 goal platform topped by a glowing portal. 11 coins are placed along the path. Falling off any platform respawns the player at the spawn ground (timer and coin count preserved).

---

## Architecture

### 2D code map (`neon-leap.html`)

| Section | Purpose |
|---|---|
| `LEVELS = [...]` | All level tile maps |
| State machine | Scenes: `title`, `playing`, `paused`, `win`, `over`, `select`, `how`, `victory` |
| `parseLevel(idx)` | Converts char grid into platform/spike/coin/enemy objects; merges contiguous solid tiles for cleaner collision |
| `updatePlayer()` | Input, gravity, jumps, hazard checks |
| `moveAndCollidePlayer()` | AABB collision, axis-by-axis (X then Y) |
| `render()` | Layered draw: gradient sky → sun with stripes → stars → perspective grid → platforms → spikes → coins → goal → enemies → particles → player |
| Save layer | Loads/saves via `window.storage` with key `neon-leap-save-v1` |

### 3D code map (`neon-leap-3d.html`)

| Section | Purpose |
|---|---|
| `LEVEL = {...}` | Spawn point, platform list, coin list, goal — all in 3D coords |
| Scene setup | `THREE.Scene`, fog, ambient + directional light, two grid helpers, a sun disc with stripe occluders, ~400 stars in upper hemisphere |
| `buildPlatforms()` | Each platform: dark `MeshLambertMaterial` box + cyan `EdgesGeometry` wireframe + bright top accent stripe |
| `buildCoins()` / `buildGoal()` / `buildPlayer()` | Object meshes; player is a `THREE.Group` (body + visor + face tab + halo + point light) |
| `moveAxis(axis, delta)` | AABB collision per axis — moves on one axis, resolves overlap, then advances to next |
| `updateCamera()` | Spherical orbit camera around player using `(yaw, pitch, distance)`; targets are smoothed via `CAM_LERP` |
| `cyclePreset(dir)` | Steps through `CAM_PRESETS` |
| Particle system | Lightweight pool of small box meshes; geometry/material disposed when life expires |

---

## Physics Tuning

### 2D (pixel space, frame-based)

| Constant | Value | Notes |
|---|---|---|
| `GRAVITY` | 0.55 px/frame² | |
| `JUMP_VEL` | -13 | Max jump height ≈ 154 px ≈ 4.8 tiles |
| `DOUBLE_JUMP_VEL` | -11 | |
| `MAX_SPEED` | 4.5 px/frame | Max horizontal jump ≈ 213 px ≈ 6.7 tiles |
| `COYOTE` | 7 frames | |
| `JUMP_BUFFER` | 7 frames | |

Reachability: a single jump clears a 4-tile-up platform with margin. Spikes need ~8 tile spacing to allow chained jump landings without overshooting.

### 3D (meters, delta-time-based)

| Constant | Value | Notes |
|---|---|---|
| `GRAVITY` | 28 m/s² | |
| `JUMP_VEL` | 11 m/s | Max jump height ≈ 2.16 m |
| `DOUBLE_JUMP_VEL` | 9.5 m/s | Combined max ≈ 3.77 m |
| `MOVE_SPEED` | 7 m/s | Max horizontal jump ≈ 5.5 m |
| `ACCEL` | 35 m/s² | Smooth acceleration toward target velocity |
| `FRICTION_GROUND` | 14 | Decay when no input on ground |
| `COYOTE_TIME` | 0.12 s | |
| `JUMP_BUFFER` | 0.12 s | |
| `FALL_DEATH_Y` | -30 | Below this y, player respawns |

The level is laid out so adjacent platforms are 2 m higher and 3–5 m apart — comfortably within reach with a single jump, and the double-jump gives recovery margin.

---

## Camera (3D)

Spherical orbit around player using `(yaw, pitch, distance)`. Smoothed via lerp.

**Presets** (cycled with `C` key or ◇ button):

| Name | Distance | Pitch (rad) | Feel |
|---|---|---|---|
| FOLLOW | 10 | 0.50 | Default Mario-64-like |
| FAR | 16 | 0.70 | Pulled back, see more of level |
| CLOSE | 6 | 0.25 | Tight third-person |
| OVERHEAD | 14 | 1.15 | Almost top-down for planning |

**Limits:**
- `CAM_PITCH_MIN = -0.15`, `CAM_PITCH_MAX = 1.35` rad (≈ -8° to 77°)
- `CAM_DIST_MIN = 4`, `CAM_DIST_MAX = 22`

**Smoothing:** `CAM_LERP = 11` — controls how fast camera position and angle catch up to target. Higher = snappier (less perceived input lag), lower = smoother (more cinematic).

**Sensitivity:**
- Mouse drag: `0.012` rad per pixel
- Touch drag: `0.020` rad per pixel
- Scroll wheel: `1.4` units per notch
- Pinch zoom: `0.04` units per pixel of pinch delta

---

## Controls

### 2D
- **Move:** A/D or ←/→
- **Jump:** Space / W / ↑ (tap again midair for double jump)
- **Pause:** Esc / P
- Touch devices get on-screen ◀ ▶ ▲ buttons

### 3D
| Action | Desktop | Mobile |
|---|---|---|
| Move | WASD / arrows | Joystick (left bottom) |
| Jump | Space | ▲ button |
| Camera yaw | Q / E or drag | Drag screen |
| Camera pitch | R / F or drag | Drag screen |
| Zoom | Z / X or scroll | Two-finger pinch |
| Cycle preset | C / V (back) | ◇ button |
| Pause | Esc / P | (tap menu via gesture — TBD) |

The mobile cam-area covers the full screen with `pointer-events: auto`; the joystick zone (bottom-left) sits on top of it via DOM order, so dragging in the joystick zone moves, dragging anywhere else looks. Buttons sit above both with their own touch handlers.

---

## Save Schema

### 2D — key `neon-leap-save-v1`
```json
{
  "unlocked": 2,
  "bestTime":  { "0": 23.45, "1": 41.20 },
  "bestCoins": { "0": 8,    "1": 11 },
  "totalDeaths": 14,
  "totalCoins":  93
}
```

### 3D — key `neon-leap-3d-save-v1`
```json
{
  "bestTime":  62.34,
  "bestCoins": 11
}
```

Both paths defensively `Object.assign(defaultSave(), parsed)` on load so adding new fields won't break older saves.

---

## How to Extend

### Add a 2D level
Append to `LEVELS` array. Constraints:
- Exactly 18 rows tall
- All rows the same width
- Every `^` and `e` must have a `#` or `=` directly below in the next row
- Player can clear 4 tiles vertical and ~6 tiles horizontal in a single jump — design platforms within those bounds

After adding, update `buildLevelGrid()` automatically picks up the new level. Save schema is keyed by index, so existing saves stay valid.

### Add a 3D platform
Append to `LEVEL.platforms`:
```js
{ x: 5, y: 8, z: -10, w: 4, h: 1, d: 4 }
```
- `x`, `y`, `z` are the **center** of the box
- `w`, `h`, `d` are full extents
- Keep within ~3.5 m vertical of the previous platform; max ~5 m horizontal gap

Add coins similarly; the goal is a single `{x,y,z}` near the top.

### Tune physics
Edit the constants block at the top of the script. After changes, re-verify the level is still beatable — especially platform reachability if `JUMP_VEL` or `MOVE_SPEED` changes.

### Add a camera preset (3D)
Append to `CAM_PRESETS`:
```js
{ name: 'CINEMATIC', distance: 18, pitch: 0.35 }
```
The cycle order is array order. `C` advances, `V` goes back.

### Adjust drag sensitivity (3D)
Change the multipliers in two places:
- Mouse: `dx * 0.012` and `dy * 0.012` in the `pointermove` handler
- Touch: `dx * 0.020` and `dy * 0.020` in the cam-area `touchmove` handler

---

## Tuning History (decisions worth remembering)

- **2D jump velocity** went from -11.5 → -12 → -13. The first value didn't quite reach 4-tile platforms; -13 gives comfortable margin at 4.8 tiles max.
- **2D spike spacing** widened from 5 to 8 tiles after jump bump — tighter spacing caused players to land directly on the next spike during chained jumps.
- **2D lives** — always reset to 3 when starting any level, simpler than tracking carryover.
- **3D camera** started as fixed `CAM_HEIGHT/CAM_DISTANCE` constants. Refactored to spherical `(yaw, pitch, distance)` to support presets and free-look.
- **3D drag sensitivity** raised from 0.005/0.008 to 0.012/0.020 — original felt sluggish.
- **3D `CAM_LERP`** raised 7 → 11 — removes a noticeable lag between dragging and seeing rotation.
- **3D mobile drag area** initially limited to right half. Expanded to full screen (with joystick zone winning in its corner via DOM order) so players don't have to remember which half is which.

---

## Known Limitations / Future Work

- **3D has only one level.** Adding more requires level data + a level select UI similar to the 2D version.
- **3D has no enemies.** Falling off platforms is the only failure state.
- **No music** — only synth SFX. A simple looping background track via Web Audio oscillators would be straightforward to add.
- **Save data is unsigned.** Trivial to edit. Acceptable for a single-player game.
- **No mouse keyboard fallback on touch devices** — if you're on a tablet with a Bluetooth keyboard, the touch controls take over and keyboard input still works alongside, but there's no special handling.
- **No accessibility features** — no remappable keys, no colorblind palette, no audio cues for visual events.
- **Performance** — the 3D version creates per-particle geometry/material and disposes them. Fine for casual play; would need pooling if particle counts grew significantly.

---

## Quick Reference

**To play:** open the HTML file in any modern browser.

**To modify:** the entire game is in one HTML file — search for the `<script>` block, then look for the `// =====` section markers (CONFIG, LEVELS, STATE, etc.).

**To debug:** browser devtools console works normally. The `state`, `player`, and `cam` objects are global within the script's IIFE-less scope, so you can inspect them via the console while the game runs.

**To check structural health:** the file should have balanced braces/parens, balanced `<script>` tags, and end with `</html>`. A quick `python3 -c` check covers that.
