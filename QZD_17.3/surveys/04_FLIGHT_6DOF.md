# 04 — Flight and 6DOF movement

**Engine:** `qzd17.3`, `github.com/emawind84/gzdoom` (QuestZDoom PC edition). Read-only.
All citations are repo-relative at that tag.
**Scope rule obeyed:** sprite layer only (weapon / psprite / inventory / EventHandler ZScript).
No map changes, no PlayerPawn class replacement. Where a design needs a `PlayerPawn`
override I say so and label it a WALL *under this rule*, not an engine wall.

---

## 1. Verdict

Flight itself is not the problem: GZDoom's flying path (`bFLY` + `bNOGRAVITY`) is intact in
this fork, reachable from a psprite, and it already steers off **head pitch** —
`ForwardThrust` converts forward stick into vertical thrust by `sin(Pitch)` whenever
`bNoGravity` is set (`wadsrc/static/zscript/actors/player/player.zs:1247-1257`). Player
pitch in VR is the HMD's pitch, clamped to `±maxviewpitch`, which defaults to **90°** on the
hardware renderer (`src/rendering/r_utility.cpp:1240`, `src/playsim/p_user.cpp:682-684`), so
"fly where you look", straight up and straight down, works today with no engine change.
**What does not work is script controlling the view.** In VR the rendered orientation is
overwritten from the raw HMD pose *after* `R_SetupFrame`, so writing `Pitch` or `Roll` on
the player changes aiming and physics but is **discarded from the image**
(`src/rendering/gl/stereo3d/gl_openvr.cpp:1467,1471`;
`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:740-741`). The engine's own comment
says it outright: *"Roll can be local, because it doesn't affect gameplay"*
(`gl_openvr.cpp:1464`). That kills every banking/barrel-roll/Descent fantasy at the root —
not slowly, not partially. **One premise in the brief is wrong and I have to correct it:**
both hands are freely readable. `AttackPos` / `AttackPitch` / `AttackAngle` / `AttackRoll`
are declared `native readonly` in `wadsrc/static/zscript/actors/actor.zs:268-271` and
thunked at `src/scripting/vmthunks_actors.cpp:2187-2190`, and the OpenVR path fills them
from the main-hand controller transform (`gl_openvr.cpp:2523-2536`), exactly as it fills the
offhand ones at `:2540-2553`. Iron Man palms are therefore **BUILDABLE, not a hack** — this
is the single biggest finding in this report.

---

## 2. What already exists

### 2.1 The flying movement path — EXISTS

| Piece | Citation |
|---|---|
| `bFLY` = `MF2_FLY`, `bNOGRAVITY` = `MF_NOGRAVITY`, `bFLOAT` = `MF_FLOAT` | `src/scripting/thingdef_data.cpp:88,127,130` |
| Engine fly toggle from the up/down axis | `player.zs:1564-1595` (`CheckMoveUpDown`) |
| `Vel.Z = Speed * cmd.upmove / 128.`, `upmove` clamped to `±0x300` | `player.zs:1578-1581` |
| Entering fly sets **both** flags | `player.zs:1584-1587` (`bFly = true; bNoGravity = true;`) |
| `upmove == -32768` (the Land command) clears `bNoGravity` only | `player.zs:1568-1573` |
| Flight drag: `Vel.Z *= FRICTION_FLY` each tic while `NOGRAVITY` and >2 units off the floor | `src/playsim/p_mobj.cpp:2631-2644` |
| `FRICTION_FLY = 0xeb00/65536 ≈ 0.9180` | `src/doomdef.h:277` |
| Fly-bob sine wobble, amplitude `FlyBob` / 8 | `p_mobj.cpp:2638`; property at `player.zs:81,133` |
| Gravity is zero while `NOGRAVITY`; otherwise `Level->gravity * Sector->gravity * Gravity * 0.00125` | `src/playsim/actorinlines.h:86-90` |
| Pitch-steered thrust ("fly where you look") | `player.zs:1247-1257` (`ForwardThrust`) |
| Classic-flight userinfo disables pitch steering | `player.zs:1249`, `src/playsim/d_player.h:282` |
| Air-control penalty is **skipped** while `bNoGravity` | `player.zs:1404-1409` — the `ApplyAirControl` call is guarded by `!bNoGravity` |
| Player-mobj physics integration order each tic | `p_mobj.cpp:4344` (`P_XYMovement`) then `:4372`/`:4435` (`P_ZMovement`) |

### 2.2 A vertical input axis already reaches the playsim — EXISTS

`fly += joyint(joyaxes[JOYAXIS_Up] * 2048)` → `cmd.ucmd.upmove`
(`src/g_game.cpp:827`). On the OpenVR build the **weapon-hand stick Y** is bound to
`JOYAXIS_Up` by default (`src/win32/i_openVR.cpp:54-64`, entries `ON_HAND_PAD_Y` /
`ON_HAND_STICK_Y`). So a throttle axis in the weapon hand is not something a mod has to
invent — it is the shipped default binding.

Caveat: `VR_GetMove` returns `*up = 0.0f` unconditionally
(`src/QzDoom/qzdoom_common.cpp:117`), so the VR locomotion path contributes no up axis of
its own; the vertical axis comes only through the joystick device above.

### 2.3 Both VR hands, full pose, readable from script — EXISTS

| ZScript field | Declared | Thunk | Filled from |
|---|---|---|---|
| `AttackPos` (vector3) | `actor.zs:268` | `vmthunks_actors.cpp:2187` | `gl_openvr.cpp:2523-2525` |
| `AttackPitch` | `actor.zs:269` | `:2188` | `gl_openvr.cpp:2531-2533` |
| `AttackRoll` | `actor.zs:270` | `:2189` | `gl_openvr.cpp:2536` |
| `AttackAngle` | `actor.zs:271` | `:2190` | `gl_openvr.cpp:2535` |
| `OffhandPos` / `Pitch` / `Roll` / `Angle` | `actor.zs:272-275` | `:2191-2194` | `gl_openvr.cpp:2540-2553` |
| `HmdPosition` | `actor.zs:277` | `:2195` | `hw_vrmodes.cpp:445` |

**This contradicts `_BASELINE_FACTS.md` §1a**, which states there is no main-hand pose field.
There is. `AttackPos`/`AttackPitch`/`AttackAngle`/`AttackRoll` are the main hand, and they
are populated from `GetWeaponTransform(&mat, VR_MAINHAND)` — the controller matrix — not from
the viewpoint, whenever the VR path runs (`gl_openvr.cpp:2520-2536`). The baseline's
underlying observation is still half true and still matters: in the **base**
`VRMode::SetUp()` the same four fields are reset to the viewpoint
(`hw_vrmodes.cpp:442-444`, `AttackPos = OffhandPos = PosAtZ(shootz)`,
`AttackAngle = OffhandAngle = viewpoint yaw - 90`), and `OpenVRMode::SetUp()` calls
`super::SetUp()` first (`gl_openvr.cpp:2327`) before overwriting them. So on a build or a
frame where the OpenVR block does not run — menu open, `gamestate != GS_LEVEL`
(`gl_openvr.cpp:2505`), or a non-OpenVR VR mode — **both** hands collapse to the viewpoint and
read identically. Design for that: compare `AttackPos` against `OffhandPos`; if they are
equal you are in the degenerate case and must not drive flight off them.

Angle conventions, from the assignments above:
- `OffhandAngle` / `AttackAngle` are already **world yaw** (`-90 + getViewpointYaw() + (handYaw - playerYaw)`).
- `OffhandPitch` / `AttackPitch` are **raw negated controller pitch** — *not* clamped to
  `±90` the way player `Pitch` is. Palms-down is readable.
- `*Pos` are the controller matrix translations in VR units; `vr_vunits_per_meter` defaults
  to `34.0` (`hw_vrmodes.cpp:94`).

### 2.4 Offhand-relative locomotion — EXISTS

`ThrustAngleOffset` is a per-actor yaw offset added inside `AActor::Thrust`
(`src/playsim/actor.h:1607-1612`). The VR layer sets it to the offhand controller's yaw minus
the player yaw when `vr_move_use_offhand` is on (`gl_openvr.cpp:2601-2610`;
`hw_openxrdevice.cpp:655-663`). That is a shipped "stick steers off the hand you hold it in"
feature, and because it is a plain `DAngle` on the actor it composes with anything a mod does
with `Thrust`.

### 2.5 Where a sprite-layer mod can inject — EXISTS

Tic order inside one `P_Ticker` pass:

1. `P_PlayerThink` (`src/p_tick.cpp:167`) → ZScript `PlayerThink` → `CheckPitch`,
   `HandleMovement` (`MovePlayer`, `CheckJump`, `CheckMoveUpDown`), `CalcHeight`, then
   **`TickPSprites`** (`player.zs:1785-1798`).
2. `localEventManager->WorldTick()` (`src/p_tick.cpp:171`).
3. `Thinkers.RunThinkers` (`src/p_tick.cpp:172`) → player mobj `Tick` →
   `P_XYMovement` / `P_ZMovement` (`p_mobj.cpp:4344,4372,4435`).

So **both** a weapon's psprite code and an `EventHandler.WorldTick` run *after* all player
input has been consumed and *before* physics integrates. Anything written to `Vel`,
`bNoGravity`, `bFly` or `Gravity` at either point takes effect on the same tic. This is the
whole foundation for sprite-layer flight and it is clean.

### 2.6 Pitch clamps — the exact numbers

- Applied in ZScript: `Pitch = clamp(Pitch - clook * (360./65536.), player.MinPitch, player.MaxPitch)`
  — `player.zs:1484`.
- Applied in C++ on every `SetPitch` / `A_SetPitch` for a player:
  `AActor::ClampPitch` — `src/playsim/p_mobj.cpp:3644-3661`, using `player->MinPitch` /
  `player->MaxPitch`, falling back to `±89°` for non-players (`p_mobj.cpp:3655-3656`).
- Applied again in the renderer:
  `viewPoint.Angles.Pitch = clamp<DAngle>(..., player->MinPitch, player->MaxPitch)` —
  `src/rendering/r_utility.cpp:636`.
- The limits themselves come over the net from `player_t::SendPitchLimits`
  (`src/playsim/p_user.cpp:670-690`), decoded at `src/d_net.cpp:2746-2747` as
  `MinPitch = -uppitch`, `MaxPitch = +downpitch`.
- **Hardware renderer (what VR runs): `uppitch = downpitch = (int)maxviewpitch`**
  (`p_user.cpp:682-684`). `maxviewpitch` defaults to `90.f` and is itself clamped to
  `±90` (`src/rendering/r_utility.cpp:1240-1243`). So **MinPitch = -90°, MaxPitch = +90°**.
- Software renderer only: `MAX_DN_ANGLE = min(56, maxviewpitch)`,
  `MAX_UP_ANGLE = min(32, maxviewpitch)` — `p_user.cpp:665-666`. The famous 32/56 numbers
  are **not** the VR limits.
- `MinPitch` and `MaxPitch` are exposed to ZScript as plain writable
  `native double` (`player.zs:3124-3125`, fields at `p_user.cpp:1931-1932`) — no `readonly`.
  A mod can widen or narrow the clamp at will; `SendPitchLimits()` is also callable
  (`player.zs:3169`).

### 2.7 Roll

- `Actor.Roll` and `A_SetRoll` exist and are settable; `AActor::SetRoll` —
  `p_mobj.cpp:3724+`.
- A separate view-only roll exists: `A_SetViewRoll` (`actor.zs:1277`,
  `src/playsim/p_actionfunctions.cpp:3046-3062`), feeding `player->ViewAngles.Roll`, applied
  at `src/rendering/r_utility.cpp:685`.
- **Neither reaches the VR image.** See §5.

---

## 3. Building blocks

Everything here is reachable from a psprite state action or an `EventHandler`.

**Read (per tic, 35 Hz):**
`AttackPos/Pitch/Angle/Roll`, `OffhandPos/Pitch/Angle/Roll`, `HmdPosition`,
`player.PlayInVR` (`player.zs:3136`), `pos`, `vel`, `floorz`, `ceilingz`, `waterlevel`,
`player.onground`, `player.cmd.*` (`player.zs:2-10`, all non-readonly), `Pitch`, `Angle`,
`player.MinPitch` / `MaxPitch`.

**Write:**
`Vel` (vector3, direct), `bNoGravity`, `bFly`, `bFloat`, `Gravity` (`actor.zs:176`),
`Speed` (`actor.zs:113`), `player.MinPitch` / `MaxPitch`, `player.keepmomentum`
(`src/playsim/d_player.h:414`), `A_SetAngle`, `A_SetPitch`, `player.cmd.*`,
`SetZ` (`actor.zs:846`), `SetOrigin` (`actor.zs:753`).

**Move / collide:**
`TryMove(vector2, dropoff, missilecheck, tm)` (`actor.zs:831`),
`CheckMove(vector2, flags, tm)` (`actor.zs:832`),
`Vel3DFromAngle(speed, angle, pitch)` — note this **replaces** `Vel`, it does not add
(`src/playsim/actor.h:1620-1626`), and its Z term is `speed * -sin(pitch)`.

**Velocity derivation:** no hand or head velocity field exists. Store last tic's
`AttackPos` / `OffhandPos` / `HmdPosition` and difference. At 35 Hz a one-second palm shove is
~35 samples; a flick is 2-4. See `_BASELINE_FACTS.md` §3.

### 3.1 Two traps you will hit

**`A_ChangeVelocity` is a no-op on the player by default.**
`wadsrc/static/zscript/actors/actions.zs:63-79`:

```
if (player != NULL && player.mo == ref)
{
    if (!vr_recoil) return;
    player.keepmomentum = true;
    ...
```

`vr_recoil` defaults to **false** (`src/playsim/p_actionfunctions.cpp:90`). Same early-out
in `A_Recoil` at `p_actionfunctions.cpp:1292-1295`. **Do not build flight thrust on
`A_ChangeVelocity`.** Write `player.mo.Vel` directly, and set
`player.keepmomentum = true` yourself.

**The VR momentum killer.** `MovePlayer` zeroes horizontal velocity every tic:

```
if (!vr_momentum && !player.keepmomentum && player.onground && friction == ORIG_FRICTION)
{
    vel.XY *= 0.0001;
    Speed = Default.Speed * 8;
}
```
— `player.zs:1373-1383`. `vr_momentum` defaults to **false**
(`src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:120`). It is gated on
`player.onground`, which is `pos.z <= floorz + 2 || bOnMobj || CF_NOCLIP2`
(`player.zs:1369`), so it does not fire in the air — but it fires hard on the launch tic and
whenever you skim within 2 units of the floor. `player.keepmomentum = true` is the documented
escape hatch (`d_player.h:414`: *"keep momentum until velocity reach 0, override vr_momentum"*),
cleared automatically once `|vel.x|` and `|vel.y|` fall under `vr_momentum_threshold`
(default `1.f`, `hw_vrmodes.cpp:121`) at `player.zs:1385-1388`.

**Also:** `vr_teleport` hard-zeroes stick locomotion —
`if (vrmode->IsVR() && vr_teleport) { side = forward = 0; }` (`src/g_game.cpp:844-847`).
Any stick-driven flight silently dies if the user plays with teleport locomotion. Direct
`Vel` writes are unaffected.

---

## 4. Mechanic designs

### 4.1 Jetpack — vertical thrust from a held button
**Player does:** holds a button (or the weapon-hand stick forward, already bound to
`JOYAXIS_Up`). Rises; releases and falls.
**Detect:** `player.cmd.buttons & BT_*` in the weapon's `Tick`, or read
`player.cmd.upmove` which the engine already fills.
**Resolve:** set `bNoGravity = false` (keep real gravity), and each tic
`player.mo.Vel.Z += thrust` with a cap, plus `player.keepmomentum = true`. Gravity subtracts
`grav` per tic in `FallAndSink` (`p_mobj.cpp:2957-2969`) so the thrust/gravity balance is the
whole feel. Alternatively set `Gravity` to a fraction (`actorinlines.h:86-90`) for a floatier
fall without touching the fall path.
**Knobs:** thrust per tic; `Vel.Z` cap; `Gravity` multiplier; fuel drain; whether to set
`bFly` (purely cosmetic here — it only gates bob, `player.zs:656`).
**Label:** BUILDABLE. **Cost: S.**

### 4.2 Hover / hold-altitude
**Player does:** nothing; it is a state the jetpack enters at zero input.
**Detect:** N/A.
**Resolve:** `bNoGravity = true; bFly = true`. The engine then damps `Vel.Z` by
`FRICTION_FLY = 0.918` per tic (`p_mobj.cpp:2640-2643`) and adds the `FlyBob` sine
(`p_mobj.cpp:2638`). For a hard altitude hold, PD-control `Vel.Z` toward
`(targetZ - pos.z) * k` in the weapon `Tick`.
**Knobs:** `FlyBob` (property, `player.zs:81`) and the user cvar `FViewBob` which gates the
bob entirely (`p_mobj.cpp:2634-2639`); the `k` gain; a dead band.
**Label:** EXISTS (as a flag pair) / BUILDABLE (as altitude hold). **Cost: S.**

### 4.3 Plane-style — "fly where you look", stick is throttle
**Player does:** looks where they want to go, pushes the stick forward.
**Detect:** nothing to detect. The engine does it: with `bNoGravity` set and
`Pitch != 0`, `ForwardThrust` splits forward move into
`Vel.Z -= move * sin(Pitch)` and `move *= cos(Pitch)` (`player.zs:1249-1256`).
**Resolve:** set `bNoGravity = true` from the psprite and stop. Because `Pitch` in VR is the
HMD pitch clamped to `±90` (§2.6), looking straight down and pushing forward dives.
**Knobs:** `Speed` (`actor.zs:113`) scales `fm`/`sm` at `player.zs:1414-1415`;
`level.aircontrol` is bypassed while `bNoGravity` so you get full authority;
`player.GetClassicFlight()` turns the whole thing off per-user (`player.zs:1249`) — read it
and fall back to §4.1 if set, do not fight it.
**Label:** EXISTS. **Cost: S.**
**Caveat:** `vr_teleport` zeroes `forwardmove` (`g_game.cpp:844-847`), so this specific
design has a user setting that silences it.

### 4.4 Iron Man palms — thrust along both hand vectors
**Player does:** points both palms down-and-back; rises. Points one palm sideways; yaws and
translates. Points both forward; flies forward.
**Detect:** in the weapon's `Tick`, build each hand's unit thrust vector from the readable
pose:
```
// main hand
double ma = AttackAngle, mp = AttackPitch;
vector3 mdir = (cos(ma)*cos(mp), sin(ma)*cos(mp), -sin(mp));
// offhand
double oa = OffhandAngle, op = OffhandPitch;
vector3 odir = (cos(oa)*cos(op), sin(oa)*cos(op), -sin(op));
```
(sign convention taken from `Vel3DFromAngle`, `actor.h:1620-1626`, and from
`A_ChangeVelocity`'s own use of the `AttackDir`/`OffhandDir` results,
`actions.zs:98-115`). Thrust is **opposite** the palm normal, so push `-mdir`, `-odir`.
Gate each hand on its own grip/trigger button — `CheckWeaponButtons(hand)` is per-hand
(`player.zs:588`, `:2031`) and `PSP_OFFHANDWEAPON` is a real separate layer with its own
state machine (`player.zs:411,563,1947-1998`), so a left-hand repulsor and a right-hand
repulsor are two ordinary weapons, not a hack.
**Resolve:** `bNoGravity = false; player.keepmomentum = true;`
`player.mo.Vel += (-mdir * mThrust - odir * oThrust) * perTic;` clamp `Vel.Length()`.
**Degenerate-case guard:** if `AttackPos ~== OffhandPos` or `!player.PlayInVR`, the pose
fields are the viewpoint, not hands (`hw_vrmodes.cpp:442-444`) — fall back to §4.3.
**Knobs:** per-hand thrust scalar; speed cap; a low-pass on the direction vectors (35 Hz
controller pitch is jittery); a minimum pitch before a hand counts as "down"; gravity
fraction via `Gravity` so hover is a balance, not a toggle.
**Label:** BUILDABLE. **Cost: M.**
**This is the design the engine most wants to run.** Unclamped per-hand pitch is the only
full-sphere direction script can read, and there are two of them.

### 4.5 Throttle in one hand, stick in the other
**Player does:** offhand stick steers, main-hand stick Y is the climb/dive throttle.
**Detect:** already wired. `ON_HAND_STICK_Y → JOYAXIS_Up` (`i_openVR.cpp:54-64`)
→ `g_game.cpp:827` → `cmd.upmove` → `CheckMoveUpDown` (`player.zs:1564-1595`). The offhand
stick drives `forwardmove`/`sidemove`, and `ThrustAngleOffset` makes those follow the offhand
controller's yaw when `vr_move_use_offhand` is on (`gl_openvr.cpp:2601-2610`,
`actor.h:1607-1612`).
**Resolve:** nothing to write; set `bNoGravity` once and the stock path flies.
**Knobs:** `Speed`; the `±0x300` upmove clamp is applied in ZScript (`player.zs:1577`) so a
`PlayerPawn` override could widen it, but under the sprite-layer rule you instead add your own
`Vel.Z` on top in the weapon `Tick` after reading `player.cmd.upmove`.
**Label:** EXISTS. **Cost: S.**
**Note on the brief's premise:** the asymmetry the brief worries about is real for
*locomotion bindings* (the vertical axis is on the main hand by default) but not for *pose* —
both hands' poses are readable (§2.3).

### 4.6 Hand-shove burst / thrust by swing
**Player does:** shoves a palm sharply.
**Detect:** store `AttackPos` / `OffhandPos` each tic, difference, compare magnitude against
a threshold. At 35 Hz a shove is 2-4 samples (`_BASELINE_FACTS.md` §3) — threshold the
single-tic delta, do not try to read the arc.
**Resolve:** one-shot `Vel += -dir * burst`.
**Knobs:** threshold; refractory tics; burst magnitude; whether the burst is along the hand
normal or along the measured motion.
**Label:** BUILDABLE. **Cost: M.**
**Risk:** no haptics to confirm the burst fired — `VR_HapticEvent` is an **empty function
body** on this PC build (`src/QzDoom/qzdoom_common.cpp:64-66`). Feedback must be audio/visual.

### 4.7 Zero-g Descent-style 6DOF
**Player does:** rolls, inverts, flies down a shaft upside-down.
**Resolve:** impossible as described. See §5.1 and §5.2. The *translational* half —
unconstrained movement along all three axes with no gravity — is §4.4 and works. The
*rotational* half — pitch past vertical, any roll, any sense of "down" other than world -Z —
does not exist.
**Label:** WALL (as the reference fantasy) / BUILDABLE (as 3-axis translation only).
**Cost: L and still wrong.**

### 4.8 Widening or narrowing the pitch clamp
**Resolve:** `player.MinPitch` / `player.MaxPitch` are writable doubles
(`player.zs:3124-3125`). Narrowing them works everywhere. **Widening past ±90 does not
help in VR**: the rendered pitch is the HMD's, not the actor's (§5.1), and
`r_utility.cpp:636` re-clamps the viewpoint anyway. Widening is also fought by
`SendPitchLimits` (`p_user.cpp:670-690`), which the engine re-sends whenever `maxviewpitch`
changes and which overwrites your values via `d_net.cpp:2746-2747`.
**Label:** HACK. **Cost: S**, and it buys you nothing in VR.

---

## 5. Walls

### 5.1 WALL — script cannot set the VR view pitch or roll
`R_SetupFrame` runs at `src/rendering/hwrenderer/hw_entrypoint.cpp:117`. `vrmode->SetUp()`
runs at `:143`, and calls `updateHmdPose`, which assigns:

```
vp.HWAngles.Roll  = FAngle::fromDeg(-hmdroll);
vp.HWAngles.Pitch = FAngle::fromDeg(-hmdpitch);
```
— `src/rendering/gl/stereo3d/gl_openvr.cpp:1467,1471`
(OpenXR path identical: `hw_openxrdevice.cpp:740-741`).

The view matrix is then built from `vp.HWAngles`
(`src/rendering/hwrenderer/scene/hw_drawinfo.cpp:443`), and the stereo eye offsets from the
same (`hw_openxrdevice.cpp:207`). Whatever `Angles.Pitch` / `Angles.Roll` the playsim
computed — including anything `A_SetPitch`, `A_SetViewRoll` or a direct write produced — is
overwritten before it is used. The engine author documented the intent at
`gl_openvr.cpp:1464`: *"Roll can be local, because it doesn't affect gameplay."*

**Missing piece:** there is no script-visible view-angle offset applied *after*
`updateHmdPose`. `player->ViewAngles` is applied in `r_utility.cpp:685`, which runs inside
`R_SetupFrame` — i.e. earlier, and into `Angles`, not `HWAngles`. The three tracking gates
are compile-time constants, not cvars: `doTrackHmdPitch`, `doTrackHmdRoll`,
`doLateScheduledRotationTracking`, all `static const bool ... = true`
(`gl_openvr.cpp:307-310`). An engine fix is one additive field —
a per-player `ViewRollOffset` / `ViewPitchOffset` added to `vp.HWAngles` at the end of
`updateHmdPose` — but it is an engine fix, and this survey is read-only.

### 5.2 WALL — no rotational 6DOF at all
Pitch is clamped to `±maxviewpitch` in three independent places
(`player.zs:1484`, `p_mobj.cpp:3644-3661`, `r_utility.cpp:636`); you cannot pitch past
vertical, and even if you could, §5.1 means the view would not follow. Roll never enters
collision: `P_XYMovement` / `P_ZMovement` are axis-aligned and the actor is an upright
cylinder. "Up" is `+Z` in world space, permanently. **Missing piece:** an orientation
quaternion on the actor and a collision volume that respects it — a different engine, not a
patch.

### 5.3 WALL — room-scale head motion is XY-only and vertically discarded
The VR layer applies positional tracking by temporarily overwriting the player's velocity and
calling `P_XYMovement` with zero scroll, then **explicitly restoring Z**:

```
bool wasOnGround = player->mo->Z() <= player->mo->floorz;
float oldZ = player->mo->Z();
P_XYMovement(player->mo, DVector2(0, 0));
if (player->mo->Z() >= oldZ && wasOnGround) player->mo->SetZ(player->mo->floorz);
else                                        player->mo->SetZ(oldZ);
```
— `gl_openvr.cpp:2613-2632` (OpenXR: `hw_openxrdevice.cpp:670-687`).

Head height reaches the renderer only as an eye offset
(`hw_openxrdevice.cpp:221-222`: `eyeOffset[2] += getHmdAdjustedHeightInMapUnit() - ...`) and
as crouch (`gl_openvr.cpp:2512-2517`, `player->crouchfactor = HmdHeight / defaultViewHeight`).
So **ducking or standing up never moves your collision volume vertically.** You cannot duck
under a flying obstacle; you cannot climb by standing. **Missing piece:** a Z component in
the positional-movement block.

### 5.4 WALL (under this survey's rule) — the movement functions are PlayerPawn virtuals
`MovePlayer`, `CheckPitch`, `CheckJump`, `CheckMoveUpDown`, `HandleMovement`,
`ApplyAirControl`, `CalcHeight`, `PlayerThink` are all `virtual` on `PlayerPawn`
(`player.zs:1340,1460,1514,1564,1612,1328,604,1734`), and `FallAndSink` is an overridable
`AActor` virtual (`p_mobj.cpp:3042-3051`). Overriding any of them is the clean way to build a
custom flight model — and it requires replacing the player class, which the sprite-layer rule
forbids. Everything in §4 is therefore built by *adding* velocity after those functions have
run, never by changing what they do. Practical cost: you inherit the `vr_momentum` kill
(§3.1), the `±0x300` upmove clamp, and `GetClassicFlight` behaviour whether you want them or
not.

### 5.5 WALL — no haptic confirmation
`VR_HapticEvent` has an empty body on this PC build
(`src/QzDoom/qzdoom_common.cpp:64-66`), and nothing in `wadsrc/static/zscript` calls
`VRMode::Vibrate`. Confirms `_BASELINE_FACTS.md` §4. Thrust feedback is sound and sprite
only.

### 5.6 Not a wall, but say it out loud — what "6DOF" costs here
The world stays a 2.5D sector world: upright cylinder collision, floor/ceiling bound
(`p_mobj.cpp:2769` sets `Z = floorz` and zeroes `Vel.Z` on contact; `:2796-2812` sets
`Z = ceilingz - Height` and zeroes upward `Vel.Z`), step-up by `MaxStepHeight`, and a view
that is always level about the world Z axis. What survives:

- **Survives:** free translation in X/Y/Z, hover, dive, climb, hand-directed thrust,
  momentum, inertia, speed caps, fuel. Everything positional.
- **Dies:** banking into a turn, barrel rolls, inverted flight, looking past vertical, any
  cockpit horizon that tilts, "which way is up" as a gameplay question, and every Descent
  reference built on those.

A VR jetpack and an Iron Man flight rig are honest fits. A space-sim is not.

---

## 6. Smallest prototype

One weapon, one `Tick` override, no class replacement, no EventHandler. Proves §4.4 and
§4.1 together and surfaces every trap in §3.1.

1. A `Weapon` with `+WEAPON.OFFHANDWEAPON` cleared (main hand) that overrides
   `Tick()`.
2. Each tic, read `AttackPos`, `AttackPitch`, `AttackAngle` from `owner`. Bail out to
   non-flight if `!owner.player.PlayInVR` or if `AttackPos` equals `OffhandPos` to within
   epsilon (the degenerate case of §2.3).
3. While the fire button is held (`player.cmd.buttons & BT_ATTACK`):
   - `owner.bNoGravity = false;`
   - `owner.player.keepmomentum = true;` (defeats `player.zs:1373-1383`)
   - build `mdir` as in §4.4, then `owner.Vel += -mdir * thrust;`
   - clamp `owner.Vel.Length()` to a cap.
4. On release, if `pos.z > floorz + 2`, set `bNoGravity = true; bFly = true;` for two seconds
   so the engine's `FRICTION_FLY` glide (`p_mobj.cpp:2640-2643`) is visible, then clear.
5. Log `AttackPitch`, `Vel.Z` and `pos.z` once per 5 tics to prove the path fired.

**Do not** use `A_ChangeVelocity` anywhere in this prototype (§3.1). Expect the first
version to appear dead on the ground tic and work in the air — that is the `vr_momentum`
kill, not your code.

**Gate it with the `gate-a-doom-mod` skill before claiming it works.** Absent telemetry is
not evidence.

---

## 7. Open questions

1. **Which VR backend actually ships?** `ENABLE_OPENVR` defaults to `NO`
   (`src/CMakeLists.txt:418`), and there is no OpenXR *input* code anywhere in this tree —
   no `xrGetActionStateFloat`, no `XrActionStateVector2f` hits under `src/` or `libraries/`,
   and `VR_HapticEvent` is a stub (`qzdoom_common.cpp:64`). `hw_openxrdevice.cpp` is the
   render half only. **If the OpenXR path is the one in use, I could not find what fills
   `weaponangles` / `offhandangles` / `hmdorientation`, and therefore could not verify that
   `AttackPos`/`OffhandPos` are populated at all on that path.** Every §2.3 citation is from
   `gl_openvr.cpp`. This needs answering before anyone builds §4.4 — ask the owner which
   backend the shipped build links.
2. Is `level.aircontrol` writable from ZScript at this tag? It is bypassed while
   `bNoGravity` (`player.zs:1404`) so it only matters for the jetpack-with-gravity model; I
   did not chase the `LevelLocals` field declaration.
3. Does `SendPitchLimits` get re-sent often enough to clobber a mod's `MinPitch`/`MaxPitch`
   writes in practice? It fires on `maxviewpitch` change (`r_utility.cpp:1244-1248`) and at
   `player.zs:217`; I did not enumerate every caller.
4. `_BASELINE_FACTS.md` §1a needs correcting — see §2.3. Recommend the baseline gain a row
   for `AttackPos`/`AttackPitch`/`AttackAngle`/`AttackRoll` with the degenerate-case caveat,
   since every two-handed design in this survey family was scoped against the wrong premise.
