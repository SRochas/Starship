# Plan: Implement On-Foot Fox Somersault

## Context

StarFoot adds on-foot Fox gameplay to the Star Fox 64 decompilation. A somersault
(full 360° backwards loop — up, over, backward, down, forward) is a core Arwing
mechanic that on-foot Fox is missing. A previous attempt existed and was discarded;
it had three concrete problems:

1. **Camera didn't chase Fox** — caused entirely by `Math_SmoothStepToF(&player->camDist, -300.0f, ...)` in the loop function, which pulled `Camera_UpdateOnFoot`'s `eye.z` from ~110 to ~400+ units back, making Fox appear tiny. No other camera changes are needed: `Camera_UpdateOnFoot` sets `eye.y = player->pos.y + 50` directly every frame (no lag); `Camera_UpdateOnFoot360` smooth-steps `eye.y` with maxStep=65/frame, which easily covers Fox's max ~17 units/frame vertical speed.

2. **Fox didn't jump high enough** — the Y math was actually correct (peak height ~415 units above start at loopSpeed=15). The complaint was visual: `aerobaticPitch` was not included in the display matrix RotateX, so the model never visually rotated — Fox just appeared to float up and down. Fix: add `aerobaticPitch` to the RotateX call in `fox_display.c`.

3. **Fox didn't reverse direction** — The +180° in the Y rotation and negative sign on the X rotation together produce the correct backwards arc (pitch=0°→forward, 90°→up, 180°→backward, 270°→down, 360°→forward). This was architecturally correct in the old attempt too, but invisible because the display matrix wasn't updated.

Additionally, the old attempt used a `gFootSomersault` global instead of the
existing `player->somersault` struct field — breaking multiplayer. The correct
field already exists at `sf64player.h` offset 0x4DC.

---

## Physical Movement During the Loop

Fox physically travels through the air — this is not a rotation-in-place. The
mechanism:

The pitch matrix bends the forward velocity vector through 360°. At each frame:
- `vel.y = loopSpeed * sin(aerobaticPitch)` — genuine upward thrust peaking at loopSpeed units/frame when pitch = 90°
- `vel.z = -loopSpeed * cos(aerobaticPitch)` — forward/backward oscillation (backward at pitch=0°, forward at pitch=180°, backward again at 360°)
- `pos.x/y/z += vel.x/y/z` — these are real world-position changes every frame

At loopSpeed=15.0f, peak height above start ≈ **415 units** (343 from arc + 72
from Y-lift). Rings at 100–200 units above ground are easily reachable. Gravity
is not applied during the loop (Player_MoveOnFootRails is replaced for the
duration). After the loop ends, Fox is 72 units above start and falls naturally.

loopSpeed=15.0f is tunable — higher values produce a wider, taller arc.

---

## Template: Player_PerformLoop (fox_play.c:4784)

The Arwing somersault is the reference. On-foot reuses the same core mechanics:
- `Math_SmoothStepToF(&player->aerobaticPitch, 360.0f, 0.1f, 5.0f, 0.001f)` — ~70 frames
- Y-lift: `if (aerobaticPitch < 180.0f) player->pos.y += 2.0f`
- Matrix: `RotateY(yRot_114 + rot.y + 180°)` then `RotateX(-(xRot_120 + rot.x + aerobaticPitch))` applied to `(0, 0, loopSpeed)` → vel.x/y/z
- `pos.x += vel.x; pos.y += vel.y; pos.z += vel.z; trueZpos = pos.z`
- Exit: `aerobaticPitch > 350° → somersault = false; aerobaticPitch = 0.0f`

Omit from on-foot: flap rotation, boostMeter refill, boostCooldown, alternateView save/restore.

---

## Changes Made

### 1. `src/engine/fox_play.c` — Player_OnFootUpdateSpeed

Restructured C-Left block: down+C-Left while grounded triggers somersault (takes
priority over sprint); plain C-Left still toggles sprint.

### 2. `src/engine/fox_play.c` — Player_PerformFootLoop (new function)

Inserted after `Player_MoveOnFoot360`. Based on `Player_PerformLoop` (4784) with:
- Fixed `loopSpeed = 15.0f`
- Floor guards using `groundPos.y` and `yPath` (not `pathFloor + yPath`)
- `aerobaticPitch` reset to 0 on exit
- Steering via stick_x → rot.y
- No camDist modification

### 3 & 4. `src/engine/fox_play.c` — Dispatch blocks

Both `Player_UpdateOnRails` and `Player_Update360` FORM_ON_FOOT cases now
dispatch to `Player_PerformFootLoop` when `player->somersault`, else the normal
movement function.

### 5. `src/engine/fox_play.c` — Player_Shoot

Added `if (player->somersault) { break; }` at top of FORM_ON_FOOT case.

### 6. `src/engine/fox_play.c` — Player_Setup

Resets `player->somersault = false` and `player->aerobaticPitch = 0.0f` after
the `gFootModeEnabled` block.

### 7. `src/engine/fox_display.c` — Display_Player_Update (~line 1761)

Added `player->aerobaticPitch` to the on-foot RotateX so the model visually
rotates during the loop.

---

## What Is NOT Changed

- `sf64context.h` / `fox_context.c` — no new globals needed
- Camera functions — already track `player->pos.y` adequately
- `player->camDist` — not modified during the loop

---

## Verification

Build the project and run any on-foot level (Corneria recommended). While
running, hold stick down and press C-Left. Fox should:
1. Launch into a backwards arc (nose pitches up)
2. Rotate fully overhead (upside-down at the top)
3. Pitch forward through the downward arc
4. Return to forward-facing and fall to ground
5. Camera stays behind Fox — no zooming or drifting
6. Fox cannot fire during the loop
