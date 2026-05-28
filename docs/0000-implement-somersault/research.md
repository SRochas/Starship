# Somersault for On-Foot Fox — Research

## Goal

Implement a full 360° somersault for FORM_ON_FOOT. Fox pitches up, loops overhead
backward, and returns to forward-facing — matching the visual shape of the Arwing
somersault. Triggered by C-Left + stick-down while grounded.

---

## 1. Authoritative Reference: Player_PerformLoop

`fox_play.c:4784` is the Arwing somersault function. Everything below is derived
from reading it directly.

### State fields (sf64player.h)

| Field | Offset | Type | Role |
|---|---|---|---|
| `player->somersault` | 0x4DC | bool | Active-loop flag; set on trigger, cleared on exit |
| `player->aerobaticPitch` | 0x4D8 | f32 | Accumulated pitch angle 0→360° |

### Per-frame execution in Player_PerformLoop

**Step 1 — Y lift (line 4804)**
```c
if (player->aerobaticPitch < 180.0f) {
    player->pos.y += 2.0f;
}
```
Adds 2 units/frame during the first half-rotation. At 5°/frame, 0→180° takes ~36
frames = ~72 units of height gain. This is what lets the player clear the ground.

**Step 2 — Pitch advance (line 4820)**
```c
Math_SmoothStepToF(&player->aerobaticPitch, 360.0f, 0.1f, 5.0f, 0.001f);
```
Advances 5°/frame from 0–300° (maxStep clamped), then decelerates gently. Reaches
350° in roughly 70 frames (~1.2 seconds at 60fps).

**Step 3 — Exit condition (line 4821)**
```c
if (player->aerobaticPitch > 350.0f) {
    player->somersault = false;
    // Arwing: restores alternateView, sets unk_018/unk_014 — skip for on-foot
}
```
Note: the Arwing does NOT reset aerobaticPitch to 0 here. It decays back via
`Math_SmoothStepToAngle` in the normal flight update (line 4646). For on-foot we
must reset it explicitly on exit (see §5).

**Step 4 — Yaw steering (lines 4836–4838)**
```c
f32 temp = -gInputPress->stick_x * 0.68f;
Math_SmoothStepToF(&player->rot.y, temp, 0.1f, 2.0f, 0.0f);
```
Allows directional steering mid-loop. Should be preserved for on-foot.

**Step 5 — Velocity via matrix (lines 4842–4853)**
```c
Matrix_RotateY(gCalcMatrix, (player->yRot_114 + player->rot.y + 180.0f) * M_DTOR, MTXF_NEW);
Matrix_RotateX(gCalcMatrix, -((player->xRot_120 + player->rot.x + player->aerobaticPitch) * M_DTOR), MTXF_APPLY);

sp4C = { 0.0f, 0.0f, player->baseSpeed };
Matrix_MultVec3fNoTranslate(gCalcMatrix, &sp4C, &sp40);

player->vel.x = sp40.x;
player->vel.z = sp40.z;
player->vel.y = sp40.y;
player->pos.x += player->vel.x;
player->pos.y += player->vel.y;
```
The pitch rotation bends the forward vector up and over. At pitch=0 the vector
points forward (Z+). At pitch=90 it points straight up (Y+). At pitch=180 it
points backward (Z-). At pitch=270 it points straight down (Y-). This is what
produces the loop-up, back-over, down, and forward-again arc.

**Step 6 — Z advance + floor guard (lines 4860–4859)**
```c
player->pos.z += player->vel.z;   // line 4860 — present but easy to miss
player->trueZpos = player->pos.z; // line 4861
if (player->pos.y < player->pathFloor + player->yPath) {
    player->pos.y = player->pathFloor + player->yPath;
    player->vel.y = 0.0f;
}
```
`pos.z` IS advanced here inside PerformLoop; it does not rely on a separate
call later.

### Arwing-specific steps to OMIT for on-foot

- Lines 4799–4802: flap rotation — Fox has no wing flaps
- Lines 4808–4817: `boostCooldown = true` and `boostMeter` refill — on-foot
  jetpack meter is irrelevant to the somersault
- Lines 4827–4831: `alternateView` save/restore — on-foot is always 3rd-person
- Lines 4833–4834: `rot.z/rot.x` smooth-step to 0 — on-foot rot.x is already
  forced to 0 inside `Player_MoveOnFootRails`; since we replace that call during
  the loop, we can leave these out without issue

---

## 2. Floor Fields: Arwing vs. On-Foot

The Arwing floor guard uses `player->pathFloor + player->yPath`. For on-foot this
is wrong — `pathFloor` is only set meaningfully in Arwing/Landmaster init blocks
(confirmed at sf64player.h:0x0A4). On-foot grounding uses two different fields:

| Field | Source | Used In |
|---|---|---|
| `player->groundPos.y` (0x064) | Shadow/collision system | Grounding in `Player_MoveOnFootRails:6294` — planet/Meteo only |
| `player->yPath` (0x0B0) | Platform offset | Grounding in `Player_MoveOnFootRails:6299` — always checked |

The on-foot somersault floor guard must match what the on-foot movement code
already uses:

```c
// Mirror of Player_MoveOnFootRails lines 6294–6302
if (player->pos.y < player->groundPos.y) {
    player->pos.y = player->groundPos.y;
    player->vel.y = 0.0f;
    player->somersault = false;
    player->aerobaticPitch = 0.0f;
}
if (player->pos.y < player->yPath) {
    player->pos.y = player->yPath;
    player->vel.y = 0.0f;
    player->somersault = false;
    player->aerobaticPitch = 0.0f;
}
```

Canceling the somersault on ground-hit is the right behavior — the player clipped
the floor before completing the loop, so abort cleanly.

---

## 3. Loop Speed

The Arwing uses `player->baseSpeed` (varies by throttle, ~60–150 units/frame
during flight). On-foot `baseSpeed` during walking is ~3–5 units/frame — far too
slow to produce a visible arc. Use a fixed speed of **15.0f** units/frame, which
gives an arc roughly 15 units wide, large enough to see without being absurd.

---

## 4. Trigger Design

### Input

```c
bool cLeftHeld  = (gInputHold->right_stick_x < -40) || ((gInputHold->button & L_CBUTTONS) != 0);
bool stickDown  = (gInputHold->stick_y <= -50);
bool edgePress  = cLeftHeld && !gPrevCLeft;
```

`edgePress` is already computed in `Player_OnFootUpdateSpeed` (fox_play.c:5277).
Add a branch **before** the sprint toggle:

```c
if (edgePress && stickDown && player->grounded && !player->somersault) {
    player->somersault = true;
    player->aerobaticPitch = 0.0f;
    Player_PlaySfx(player->sfxSource, NA_SE_ARWING_BOOST, player->num);
    // Do not set gSuperSprint; fall through so gPrevCLeft is updated
}
```

The `else if` guard on the sprint branch (`&& !player->somersault`) then prevents
sprint from activating at the same time.

Require `player->grounded == true` so the player cannot cancel a jetpack ascent
into a somersault mid-air.

---

## 5. Player_PerformFootLoop — Complete Design

New function to add in `fox_play.c` after `Player_MoveOnFoot360`:

```c
void Player_PerformFootLoop(Player* player) {
    f32 loopSpeed = 15.0f;
    Vec3f sp4C;
    Vec3f sp40;
    f32 temp;

    // Rise during first half (mirrors Player_PerformLoop line 4804)
    if (player->aerobaticPitch < 180.0f) {
        player->pos.y += 2.0f;
    }

    // Advance pitch toward 360° (identical params to Arwing, line 4820)
    Math_SmoothStepToF(&player->aerobaticPitch, 360.0f, 0.1f, 5.0f, 0.001f);

    // Exit: pitch complete
    if (player->aerobaticPitch > 350.0f) {
        player->somersault = false;
        player->aerobaticPitch = 0.0f;  // explicit reset needed; on-foot has no decay path
    }

    // Steering input during loop (mirrors lines 4836-4838)
    temp = -gInputPress->stick_x * 0.68f;
    Math_SmoothStepToF(&player->rot.y, temp, 0.1f, 2.0f, 0.0f);

    // Velocity from pitch matrix
    Matrix_RotateY(gCalcMatrix, (player->yRot_114 + player->rot.y + 180.0f) * M_DTOR, MTXF_NEW);
    Matrix_RotateX(gCalcMatrix, -((player->xRot_120 + player->rot.x + player->aerobaticPitch) * M_DTOR), MTXF_APPLY);

    sp4C.x = 0.0f;
    sp4C.y = 0.0f;
    sp4C.z = loopSpeed;

    Matrix_MultVec3fNoTranslate(gCalcMatrix, &sp4C, &sp40);

    player->vel.x = sp40.x;
    player->vel.y = sp40.y;
    player->vel.z = sp40.z;  // NOT zeroed — must be set and applied

    player->pos.x += player->vel.x;
    player->pos.y += player->vel.y;
    player->pos.z += player->vel.z;  // required; mirrors line 4860
    player->trueZpos = player->pos.z;

    // Floor guards — on-foot fields, NOT pathFloor
    if (player->pos.y < player->groundPos.y) {
        player->pos.y = player->groundPos.y;
        player->vel.y = 0.0f;
        player->somersault = false;
        player->aerobaticPitch = 0.0f;
    }
    if (player->pos.y < player->yPath) {
        player->pos.y = player->yPath;
        player->vel.y = 0.0f;
        player->somersault = false;
        player->aerobaticPitch = 0.0f;
    }
}
```

---

## 6. Dispatch: Replace Movement Call During Somersault

### Player_UpdateOnRails — FORM_ON_FOOT (fox_play.c:7657)

```c
case FORM_ON_FOOT:
    Player_OnFootUpdateSpeed(player);
    if (player->somersault) {
        Player_PerformFootLoop(player);
    } else {
        Player_MoveOnFootRails(player);
    }
    Player_UpdatePath(player);
    Player_Shoot(player);
    Player_FootCollisionCheck(player);
    Player_FloorCheck(player);
    Player_WaterEffects(player);
    Player_LowHealthAlarm(player);
    ...
```

### Player_Update360 — FORM_ON_FOOT (fox_play.c:7744)

```c
case FORM_ON_FOOT:
    Player_OnFootUpdateSpeed(player);
    if (player->somersault) {
        Player_PerformFootLoop(player);
    } else {
        Player_MoveOnFoot360(player);
    }
    Player_Shoot(player);
    Player_FootCollisionCheck(player);
    Player_FloorCheck(player);
    Player_LowHealthAlarm(player);
    ...
```

---

## 7. Display Fix

`fox_display.c:1761` currently excludes `aerobaticPitch` from the on-foot
rotation matrix. The model will not visually pitch during the loop without this:

```c
// Current (line 1761):
Matrix_RotateX(gCalcMatrix, -((player->xRot_120 + player->rot.x + player->damageShake) * M_DTOR), MTXF_APPLY);

// Required:
Matrix_RotateX(gCalcMatrix, -((player->xRot_120 + player->rot.x + player->aerobaticPitch + player->damageShake) * M_DTOR), MTXF_APPLY);
```

This block is inside the `FORM_ON_FOOT` branch of `Display_Player_Update`
(confirmed by the surrounding `else if (player->form == FORM_ON_FOOT)` at
line 1746).

---

## 8. Shoot Gate

`Player_Shoot` (fox_play.c:4373) has no somersault check for `FORM_ON_FOOT`.
Fox can fire continuously during the loop, unlike the Arwing which blocks input.
Add a guard at the top of the on-foot case:

```c
case FORM_ON_FOOT:
    if (player->somersault) { break; }   // ADD: block firing during loop
    if (gInputHold->button & A_BUTTON) {
```

---

## 9. Init

Near `fox_play.c:6440` where `gFootModeEnabled` forces `FORM_ON_FOOT`, ensure
these are reset on level load:

```c
player->somersault = false;
player->aerobaticPitch = 0.0f;
```

`player->somersault` is a field on the Player struct and does not need a new
global. No changes to `sf64context.h` or `fox_context.c` are needed.

---

## 10. What Went Wrong in the Previous Attempt

The discarded implementation had these concrete bugs:

| Bug | Problem | Fix |
|---|---|---|
| Used `gFootSomersault` global | Breaks versus mode (all players share one flag) | Use `player->somersault` — already on the struct |
| `player->vel.z = 0.0f` then `player->pos.z += sp40.z` | Contradictory: zeroes vel.z but uses sp40.z directly | Set `vel.z = sp40.z` and do `pos.z += vel.z` |
| `Math_SmoothStepToF(&player->camDist, -300.0f, ...)` | Arwing loop does not change camDist; caused camera drift | Remove; camDist is only moved during sprint |
| Missing `player->trueZpos = player->pos.z` | trueZpos drifts from pos.z | Add after pos.z update |
| Missing steering input | No yaw control during loop | Add stick_x → rot.y smooth-step |
| No shoot gate | Fox fires freely during animation | Add `if (player->somersault) break;` in Player_Shoot |
| aerobaticPitch not in display matrix | Model doesn't rotate visually | Add to RotateX in fox_display.c:1761 |

---

## 11. Files to Change

| File | Location | Change |
|---|---|---|
| `src/engine/fox_play.c` | After `Player_MoveOnFoot360` (~line 5766) | Add `Player_PerformFootLoop` |
| `src/engine/fox_play.c` | `Player_OnFootUpdateSpeed` (~line 5277) | Add somersault trigger before sprint toggle |
| `src/engine/fox_play.c` | `Player_UpdateOnRails` FORM_ON_FOOT (~line 7659) | Dispatch to `Player_PerformFootLoop` when somersault |
| `src/engine/fox_play.c` | `Player_Update360` FORM_ON_FOOT (~line 7746) | Dispatch to `Player_PerformFootLoop` when somersault |
| `src/engine/fox_play.c` | `Player_Shoot` FORM_ON_FOOT (~line 4373) | Add somersault guard |
| `src/engine/fox_play.c` | `Player_Setup` (~line 6440) | Init `somersault = false`, `aerobaticPitch = 0.0f` |
| `src/engine/fox_display.c` | `Display_Player_Update` (~line 1761) | Add `aerobaticPitch` to on-foot RotateX |
