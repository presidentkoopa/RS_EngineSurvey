# VR melee on stock QuestZDoom 17.3

**A reference for a ZScript programmer.** What the machine gives you, and what three shipping
systems already do with it.

*Assembled 2026-10-24. Nothing in this document proposes changing QuestZDoom — no mod here may, and none
does. Part One is a reading of the stock source to establish what ZScript can reach; Part Two is a
reading of three mods that reach it.*

---

## How to use this

- **Part One — What ZScript can reach.** The hand surface and its units, hooks and tic order, every
  hit-resolution mechanism with its costs and traps, swept geometry, damage and gore, haptics,
  sound, netplay, and the hard limits. Every claim cites `file:line` in stock QZD 17.3, which is
  the tree `670d19eba6^` in `E:\DOOMWork\QZD17-engine` — read with `git show 670d19eba6^:<path>`.
  **That checkout's HEAD is not stock** and its line numbers differ.
- **Part Two — Three systems compared.** UltraBadassQuestSaber, RS_ShieldSawQ and the BD22 Quest
  weapons, technique by technique rather than mod by mod, each ending in a verdict and what a
  fourth implementation should do. Its trap table — "this was tried and it failed because…" — is
  mined out of these mods' own comments and commit messages.

Both parts were written against the code as it stood on 2026-10-24. The saber was being edited
while Part Two was written, so it quotes code rather than citing line numbers; two defects it
reports in the saber (the muzzle origin, and `A_QuakeEx` truncating its intensity) were fixed
before this document was assembled, and a third it found — the body's turn counting as arm motion
— was fixed because of it.

---

# Part One — What ZScript can reach

## The engine half: what melee is possible in stock QuestZDoom 17.3, from ZScript

> Reference document for a ZScript programmer who has not worked in VR on this engine.
> Not a tutorial. Every claim cites `file:line` in **stock QZD 17.3**, which in the local
> checkout is the tree `670d19eba6^` (`E:\DOOMWork\QZD17-engine`). Read it with
> `git show 670d19eba6^:<path>`. Nothing in this tree was modified, built or run.
>
> **Line numbers matter.** The checkout's HEAD (`670d19eba6`, "pre-swarm snapshot") adds
> the owner's own fields and shifts line numbers in `actor.h`, `actor.zs`, `g_game.cpp`,
> `p_pspr.cpp`, `p_map.cpp`, `d_player.h` and `p_user.cpp`. Citations below are **stock**
> and differ from the earlier study's, which were taken at HEAD. Anything that exists only
> at HEAD is called out as **HEAD ONLY — unavailable**.
>
> Companion: `E:\DOOMWork\_reports\QZD173_MELEE_STUDY_2026-10-23.md` (the design-shaped
> version). This document corrects it where they disagree; §11 lists the corrections.

---

### 0. Index — find one answer fast

| You want | Go to |
|---|---|
| The exact fields, units, signs | §1 |
| What `AttackDir`/`OffhandDir` actually returns | §1.6 |
| Metres ↔ map units, and the vertical anisotropy | §1.5 |
| Hand velocity — is there any? | §1.8 |
| Which hook can see the pose / change the world | §2 |
| The precise order inside a tic | §2.1 |
| `LineTrace` | §3.1 |
| `LineAttack` — full signature, flags, traps | §3.2 |
| `AimLineAttack` | §3.3 |
| `BlockThingsIterator` — both constructors | §3.4 |
| `ThinkerIterator`, `ActorIterator`, `BlockLinesIterator` | §3.5 |
| `CheckSight` | §3.6 |
| `RailAttack` | §3.7 |
| `DamageMobj`, `Thrust`, `ApplyKickback`, `TriggerPainChance` | §3.8–§3.10 |
| Projectiles and hand-relative spawning | §3.11 |
| Cost table | §3.13 |
| Trap list (read this before writing anything) | §3.14 |
| Swept segment vs moving target | §4 |
| Damage types, puffs, decals, blood, pain, death, gore | §5 |
| Dropping and throwing; what you can make a monster do | §5.6–§5.7 |
| Haptics — every route, cost, hand, net safety | §6.1 |
| Sound placement | §6.2 |
| Screen and world effects from play scope | §6.3 |
| Netplay: what replicates, the exact recipe | §7 |
| Hard limits with proof | §8 |
| Where I am unsure | §10 |
| Corrections to the 2026-10-23 study | §11 |

**One-line answer.** The engine hands ZScript a per-hand *position and Euler orientation*
in map coordinates, written once per rendered frame by the renderer, for the local camera
player only, with no velocity of any kind. Everything else — sweeping, capsules,
classification, replication — you write. Every primitive you need to write it exists in a
pure pk3.

---

### 1. The hand, exactly

#### 1.1 The complete surface

Declared on `AActor` (`src/playsim/actor.h:1652-1668`):

```c
bool     OverrideAttackPosDir;          // :1652
DVector3 AttackPos;                     // :1654
DAngle   AttackPitch;                   // :1655
DAngle   AttackAngle;                   // :1656
DAngle   AttackRoll;                    // :1657
DVector3 (*AttackDir)(AActor*, DAngle yaw, DAngle pitch);   // :1659  raw C fn pointer
DVector3 OffhandPos;                    // :1661
DAngle   OffhandPitch;                  // :1662
DAngle   OffhandAngle;                  // :1663
DAngle   OffhandRoll;                   // :1664
DVector3 (*OffhandDir)(AActor*, DAngle yaw, DAngle pitch);  // :1666
DVector3 HmdPosition;                   // :1668
```

Exported `DEFINE_FIELD(AActor, …)` at `src/scripting/vmthunks_actors.cpp:2186-2195`, and
declared to ZScript at `wadsrc/static/zscript/actors/actor.zs:268-277`:

```zscript
native readonly vector3 AttackPos;        // :268
native readonly double  AttackPitch;      // :269
native readonly double  AttackRoll;       // :270
native readonly double  AttackAngle;      // :271
native readonly vector3 OffhandPos;       // :272
native readonly double  OffhandPitch;     // :273
native readonly double  OffhandRoll;      // :274
native readonly double  OffhandAngle;     // :275
native readonly bool    OverrideAttackPosDir;  // :276
native readonly vector3 HmdPosition;      // :277
```

and the two direction functions at `actor.zs:858-859`:

```zscript
native vector3 AttackDir(Actor actor, double angle, double pitch);    // :858
native vector3 OffhandDir(Actor actor, double angle, double pitch);   // :859
```

That is the entire VR pose surface. There is nothing else: no velocity, no angular rate, no
analog trigger or grip scalar, no finger or touch data, no previous-frame copy, no hand
matrix (`VRMode::GetHandTransform` / `GetWeaponTransform` are C++ only,
`src/common/rendering/hwrenderer/data/hw_vrmodes.h:89-90`), no bone transforms.

Supporting player-scope fields, `wadsrc/static/zscript/actors/player/player.zs`:

| ZScript | Decl | Notes |
|---|---|---|
| `player.PlayInVR` | `player.zs:3136` (`native readonly bool`) | written at `hw_vrmodes.cpp:437`; C++ `d_player.h:502` |
| `player.resetDoomYaw` | `player.zs:3060` (`native bool`, **writable**) | C++ `d_player.h:362`; set it after you change the pawn's yaw — stock does, `stateprovider.zs:406`, `weaponchainsaw.zs:167` |
| `player.cmd` | `player.zs:3134` (`native @UserCmd cmd`) | `struct UserCmd` at `player.zs:2-11` |
| `player.WeaponState` | `player.zs:3073` | holds `WF_TWOHANDSTABILIZED` |
| `player.OffhandWeapon` | `player.zs:3076` | C++ `d_player.h:405` |
| `player.crouchfactor` | `player.zs:3126` | multiplies `AttackOffset()` |
| `player.bonuscount` | `player.zs:3087` (`native uint`, **writable**) | see §6.1 — a haptic side door |

#### 1.2 Who writes them, and when

Two writers, both in the renderer, both for one player.

**Base / non-VR fallback** — `VRMode::SetUp()`, `hw_vrmodes.cpp:432-446`:

```c
player_t* player = r_viewpoint.camera ? r_viewpoint.camera->player : nullptr;   // :434
if (player && player->mo) {
    player->PlayInVR = IsVR();                                                  // :437
    player->mo->OverrideAttackPosDir = !puristmode && (IsVR() || vr_override_weap_pos);  // :438
    player->mo->AttackDir  = MapAttackDir;                                      // :439
    player->mo->OffhandDir = MapOffhandDir;                                     // :440
    double shootz = player->mo->Center() - player->mo->Floorclip + player->mo->AttackOffset();
    player->mo->AttackPos   = player->mo->OffhandPos   = player->mo->PosAtZ(shootz);     // :442
    player->mo->AttackAngle = player->mo->OffhandAngle = r_viewpoint.Angles.Yaw - 90°;   // :443
    player->mo->AttackPitch = player->mo->OffhandPitch = -r_viewpoint.Angles.Pitch;      // :444
    player->mo->HmdPosition = DVector3(hmdPosition[0..2]);                               // :445
}
```

**VR override** — `OpenXRDeviceMode::SetUp()`, `hw_openxrdevice.cpp:521`, which calls
`super::SetUp()` **first** (`:523`) and then, guarded by
`if (gamestate == GS_LEVEL && menuactive == MENU_Off)` (`:568`), overwrites the pose
(`:587-595` main hand, `:600-608` off hand):

```c
OffhandPitch = cinemamode ? -offhandangles[PITCH] - r_viewpoint.Angles.Pitch
                          : -offhandangles[PITCH];                        // :600-601
OffhandAngle = -90 + getViewpointYaw() + (offhandangles[YAW] - playerYaw);// :603
OffhandRoll  =  offhandangles[ROLL];                                      // :604
OffhandPos.X = mo->X() - offhandoffset[0] * vr_vunits_per_meter;          // :606
OffhandPos.Y = mo->Y() - offhandoffset[2] * vr_vunits_per_meter;          // :607
OffhandPos.Z = r_viewpoint.CenterEyePos.Z
             + ((hmdPosition[1] + offhandoffset[1] + vr_height_adjust) * vr_vunits_per_meter) / pixelstretch
             - getDoomPlayerHeightWithoutCrouch(player);                  // :608-609
```

Call chain: `D_Display()` → `RenderView` → `RenderViewpoint` → `vrmode->SetUp()`
(`src/rendering/hwrenderer/hw_entrypoint.cpp:142-143`), and `D_Display()` runs *after*
`TryRunTics()` in the main loop (`src/d_main.cpp:1314` then `:1317`). **The pose is written
at the end of a display frame and consumed by the next tic.**

`OverrideAttackPosDir` is true automatically in VR (`hw_vrmodes.cpp:438`): `puristmode`
defaults false (`src/g_game.cpp:132`), `vr_override_weap_pos` defaults false and is
PC-test-only (`hw_vrmodes.cpp:147`).

#### 1.3 Written for whom — one player, nobody else

Both writers open with `r_viewpoint.camera ? r_viewpoint.camera->player : nullptr`
(`hw_vrmodes.cpp:434`, `hw_openxrdevice.cpp:551`) and touch only that player's pawn. For
**every other player on this machine the hand fields hold class defaults (zero) and
`OverrideAttackPosDir` is false**. There is no code path by which a peer's hand pose
reaches you. A remote VR player's attacks resolve from their body along their view angles
on your machine (§7.1).

Also: because the writer keys on `r_viewpoint.camera`, the fields follow the **camera**, not
`consoleplayer`. Under `SetCamera` (`actor.zs:941`) or a chasecam onto another actor with a
player, that other player's pawn receives your controller pose.

#### 1.4 What the fields read when you are *not* in the hand branch

This is a real trap and the earlier study got it backwards. `super::SetUp()` at
`hw_openxrdevice.cpp:523` runs **unconditionally**; the hand block at `:568` is gated on
`gamestate == GS_LEVEL && menuactive == MENU_Off`. The level still renders behind an open
menu (`d_main.cpp:1063` gates on `gamestate`, not `menuactive`). Therefore:

- **Menu open, or gamestate ≠ GS_LEVEL:** the hand fields are *not* frozen at their last
  values. They collapse to the fallback — `AttackPos == OffhandPos == PosAtZ(shootz)` (the
  pawn's attack height), `AttackAngle == OffhandAngle == viewYaw − 90`,
  `AttackPitch == OffhandPitch == −viewPitch`, roll **untouched** (the base `SetUp` never
  writes roll, `hw_vrmodes.cpp:432-446`), i.e. roll keeps its last VR value.
- **Paused:** `paused` is *not* in the `:568` guard, so the pose keeps updating while
  paused. (`resetDoomYaw` is forced on by `paused` at `:563`, which is a different thing.)
- **Console down:** also not in the `:568` guard. The pose keeps updating.

So a gesture detector sampling in `ui` scope will see a **step discontinuity to the body
position** the instant a menu opens, and a step back when it closes. Latch on
`menuactive`/`gamestate` or on a jump threshold, or your first post-menu frame reads as a
one-frame 1-metre swing.

`cinemamode` (toggled by `CCMD(cinemamode)`, `hw_openxrdevice.cpp:776`) changes the pitch
formula (`:587`, `:600`) by subtracting the viewpoint pitch, and changes
`getViewpointYaw()` from `doomYaw` to `r_viewpoint.Angles.Yaw` (`:173-181`). Treat
cinemamode as a second coordinate convention, not a cosmetic switch.

#### 1.5 Units, and the vertical anisotropy — derived, not guessed

`vr_vunits_per_meter` default **34.0** (`hw_vrmodes.cpp:94`).
`level.info->pixelstretch` default **1.2** (`src/gamedata/g_mapinfo.cpp:314`; field
`src/g_levellocals.h:698`); `hw_openxrdevice.cpp:572` reads it with a 1.2 fallback.

XY uses `* vr_vunits_per_meter` with no pixelstretch (`:606-607`); Z uses
`* vr_vunits_per_meter / pixelstretch` (`:608`). That is **not a bug**: the renderer stretches
the vertical axis by `pixelstretch` when it builds the projection
(`hw_openxrdevice.cpp:281-282`, `new_projection.scale(1.0, pixelstretch, 1.0)`) and the
pitch (`src/rendering/r_utility.cpp:1222-1229`, `angy = sin(pitch) * pixelstretch`). Doom's
map space is itself vertically compressed relative to the room.

So, exactly:

```
metres_horizontal = Δxy_mapunits / 34
metres_vertical   = Δz_mapunits * 1.2 / 34          (= Δz / 28.333)
physical length of a map vector = sqrt(dx² + dy² + (1.2·dz)²) / 34   metres
```

Consequences you must act on:

- **A speed or distance threshold computed from `|ΔOffhandPos|` is direction-dependent by
  20 %.** Multiply the Z component by `pixelstretch` before taking a length, if you want a
  metric gate.
- **A physically spherical hit volume of radius R metres is an ellipsoid in map units:**
  `(dx² + dy²)/(34R)² + dz²/(28.333R)² ≤ 1`.
- Sanity check with stock numbers: the player is 56 units tall and 32 wide →
  56·1.2/34 = 1.98 m and 32/34 = 0.94 m. `DEFMELEERANGE = 64` (`constants.zs:966`) is 1.88 m
  of horizontal reach; `MeleeRange` default `64 − MELEEDELTA = 44` (`actor.zs:421`,
  `MELEEDELTA = 20` at `actor.zs:90`) is 1.29 m. Both are far longer than an arm.

**Z reference — the clean result.** `r_viewpoint.CenterEyePos = r_viewpoint.Pos`
(`r_utility.cpp:1236`, "Retain unshifted center eye pos"), i.e. the interpolated centre-eye
position, assigned at the end of `R_SetupFrame` after all view offsets. And
`getDoomPlayerHeightWithoutCrouch(player)` (`hw_openxrdevice.cpp:158-171`) is
`getHmdAdjustedHeightInMapUnit()` when `!vr_crouch_use_button`, else the cached
`DefaultViewHeight()`. Substituting into `:608`:

- **`vr_crouch_use_button = true` (the default, `hw_vrmodes.cpp:122`):**
  `OffhandPos.Z = CenterEyePos.Z + handHeightAboveRoomFloor_mapunits − DefaultViewHeight`.
  Since the HMD does not drive viewheight in this mode, `CenterEyePos.Z ≈ pawn feet +
  DefaultViewHeight`, so **`OffhandPos.Z − pawn.pos.Z` is the real hand height above the
  room floor, in map units** (metres × 28.333). Lowering your head does not change it.
- **`vr_crouch_use_button = false`:** the two terms cancel to
  `OffhandPos.Z = CenterEyePos.Z + offhandoffset[1] · 34 / 1.2`, i.e. **the hand height
  relative to the eye**. Lowering your head *does* change it.

These are two different frames. A gesture system that gates on hand height must know which
cvar the user has set, or must work only in differences.

#### 1.6 `AttackDir` / `OffhandDir` — it does **not** return a direction vector

The C++ function pointer returns a unit `DVector3`
(`MapWeaponDir`, `hw_vrmodes.cpp:390-415`). **The ZScript thunk does not.** At
`src/playsim/p_actionfunctions.cpp:5694-5712`:

```c
DVector3 dir = self->AttackDir(source, yaw, pitch);
ACTION_RETURN_VEC3(DVector3(dir.Angle().Degrees(), dir.Pitch().Degrees(), 0.));
```

**From ZScript, `AttackDir(...)` returns `(yawDegrees, pitchDegrees, 0)`.** Stock uses it
exactly that way — `attacks.zs:489-490`: `directionAngle = dir.x; directionPitch = dir.y;`.
Treating the result as a vector is a silent, plausible-looking bug.

**The argument convention — the single most important thing in this section.**
`MapWeaponDir` begins (`hw_vrmodes.cpp:401-402`):

```c
yaw   -= actor->Angles.Yaw;
pitch -= actor->Angles.Pitch;
```

so the `angle`/`pitch` you pass are **deviations from the pawn's own facing**, applied in the
hand's rotation frame. To fire straight out of the hand you pass the pawn's own angles:

```zscript
// straight down the hand's forward axis
let d = player.mo.OffhandDir(self, self.Angle, self.Pitch);   // d.x = world yaw, d.y = world pitch
// 5 degrees of spread to the hand's side
let s = player.mo.OffhandDir(self, self.Angle + 5, self.Pitch);
```

Stock does precisely this: `attacks.zs:465` (`directionAngle = self.Angle + angle`) then
`:476`/`:484` with the bare `pitch` (which, ZScript being case-insensitive, is `self.Pitch`).
Same for `A_Punch` (`weaponfist.zs:74-76`) and `A_Saw` (`weaponchainsaw.zs:72-73`), where the
spread terms are added to the pawn's yaw and to `AimLineAttack`'s return (which in VR *is*
the pawn's pitch — §3.3).

**Trap: the left hand's basis is mirrored.** `VRMode::GetWeaponTransform`
(`hw_vrmodes.cpp:455-473`) maps the logical hand to a physical one and mirrors:

```c
bool rightHanded = vr_control_scheme < 10;                          // :464
int  hand = (hand_weapon == VR_OFFHAND) ? 1 - rightHanded : rightHanded;  // :465
if (GetHandTransform(hand, out)) {
    if (!hand && autoReverse) out->scale(-1.0f, 1.0f, 1.0f);        // :468-469
```

`autoReverse` is true unless the weapon **in that hand** has `WIF_NO_AUTO_REVERSE`
(`:462`), which in ZScript is `+WEAPON.NOAUTOREVERSE`
(`wadsrc/static/zscript/actors/inventory/weapons.zs:101`; `WIF_NO_AUTO_REVERSE = 0x00800000`
at `src/gamedata/a_weapons.h:165`). An **empty** hand gets `autoReverse = true` as well
(`weap == nullptr` at `:462`).

`VSMatrix::scale(-1,1,1)` negates `mMatrix[0..3]` (`src/common/utility/matrix.cpp:173-178`),
and `LSMatrix44::operator[](i)` is row `i` (`src/rendering/gl/stereo3d/LSMatrix.h:119`).
`MapWeaponDir` uses `mat[0][*]` only against `local.y` (`hw_vrmodes.cpp:409-411`), and
`local.y = cos(pitchDev)·sin(yawDev)` (`:406`). Therefore:

> **The mirror negates the yaw deviation for whichever logical hand is the physical left
> hand.** Zero deviation is unaffected. With the default right-handed scheme that is the
> off hand; with `vr_control_scheme >= 10` it is the main hand. `+WEAPON.NOAUTOREVERSE` on
> the weapon in that hand disables it.

So identical spread parameters produce mirrored spread between hands, and a hand-relative
side offset points the wrong way on one side. Zero-deviation calls are safe.

**Trap: the direction's pitch is squashed by `pixelstretch`.** `GetHandTransform`
(`hw_openxrdevice.cpp:431-482`) applies `mat->scale(vr_vunits_per_meter, …)` (`:442`) then
`mat->scale(1, 1/pixelstretch, 1)` (`:448`/`:464`) before the rotations, so the basis is
anisotropic and the result is normalised afterwards (`hw_vrmodes.cpp:412`). A 45° real
controller pitch yields `atan(1/1.2) = 39.8°` of Doom pitch. This is the *same* 1.2 as §1.5
and it is consistent with the positions, so hand ray and hand position live in one frame —
but a hand ray is **not** the controller's physical ray.

`GetHandTransform` does **not** apply `vr_weaponRotate` (default −30°,
`hw_vrmodes.cpp:106`). That tilt is a render-only weapon-model offset. `AttackDir`'s forward
axis is the controller's own forward, not the gun bore.

**Trap: they are raw function pointers.** `AttackDir`/`OffhandDir` (`actor.h:1659`, `:1666`)
are written only at `hw_vrmodes.cpp:439-440`, for the camera player's pawn. On any other
actor they are whatever `AActor`'s zero-initialised storage holds. `MapWeaponDir` itself
degrades gracefully if `GetWeaponTransform` returns false (`hw_vrmodes.cpp:394-399`:
plain yaw/pitch direction), but the pointer being null is a different failure. Guard on
`OverrideAttackPosDir` — stock always does (`attacks.zs:467`, `p_map.cpp:4784`, `:5185`,
`:4597`, `:5649`, `p_mobj.cpp:7484`, `:7334`).

#### 1.7 The angle conventions, and the idiom that proves them

- **Yaw.** `AttackAngle`/`OffhandAngle` are the world yaw **minus 90°**. Add 90 to get a
  world yaw. The engine does so in every consumer:
  `p_map.cpp:6175`, `:6184`, `:6296`, `:6305`;
  `p_actionfunctions.cpp:1306`, `:1310` (`A_Recoil`);
  and in ZScript, `wadsrc/static/zscript/actors/heretic/weaponmace.zs:71`, `:77`.
  The base writer sets it to `viewYaw − 90` (`hw_vrmodes.cpp:443`), which fixes the
  convention beyond doubt.
- **Pitch.** `AttackPitch`/`OffhandPitch` are **negated** relative to Doom's pitch
  (positive-down). `hw_vrmodes.cpp:444` sets `AttackPitch = −r_viewpoint.Angles.Pitch`.
  Negate to get a Doom pitch. Stock idiom: `weaponmace.zs:72`, `:78`, `:167`, `:171`
  (`directionPitch = -player.mo.OffhandPitch`).
- **Roll.** Raw, in degrees, with no Doom equivalent. It is the only wrist channel. Stock
  negates it when transferring to an actor's `Roll`: `attacks.zs:475`, `:483`;
  `weaponphoenix.zs:184`, `:191`.

The canonical conversion, straight from `weaponmace.zs:69-79`:

```zscript
if (player.mo.OverrideAttackPosDir) {
    spawnpos       = hand ? player.mo.OffhandPos   : player.mo.AttackPos;
    directionAngle = (hand ? player.mo.OffhandAngle : player.mo.AttackAngle) + 90;
    directionPitch = -(hand ? player.mo.OffhandPitch : player.mo.AttackPitch);
}
```

Caveat: `AttackAngle` is built from `getViewpointYaw() + (weaponangles[YAW] − playerYaw)`
(`hw_openxrdevice.cpp:603`) while `GetHandTransform` rotates by
`doomYaw + (weaponangles[YAW] − hmdorientation[YAW])` (`:471`). `playerYaw` is assigned
`hmdorientation[YAW]` only when the overlay screen is **not** in use
(`src/QzDoom/qzdoom_common.cpp:87-90`), so **the Euler yaw field and the
`AttackDir` basis disagree while the overlay screen is showing** (menu, cinemamode,
`vr_overlayscreen_always`). Pick one source per subsystem and stay with it.

#### 1.8 Velocity — what the engine computes, and for whom

There is **no hand velocity, anywhere.** The controller globals are position and Euler
angles only:

```c
// src/QzDoom/VrCommon.h:37-41
extern vec3_t weaponangles;   extern vec3_t weaponoffset;
extern vec3_t offhandangles;  extern vec3_t offhandoffset;
```
defined at `src/QzDoom/qzdoom_common.cpp:10-13`. No `*vel`, no `*angvel`, no trigger/grip
scalar, no touch bitfield.

The **only** delta the engine computes is for the **headset**, and it is not exported:

```c
// src/QzDoom/qzdoom_common.cpp:93-104   void VR_SetHMDPosition(float x,y,z)
positionDeltaThisFrame[0] = (worldPosition[0] - x);
positionDeltaThisFrame[1] = (worldPosition[1] - y);
positionDeltaThisFrame[2] = (worldPosition[2] - z);
worldPosition[0..2] = x,y,z;
```

`positionDeltaThisFrame` is declared at `qzdoom_common.cpp:17` and is **not** in
`VrCommon.h`'s extern list, has no `DEFINE_FIELD`, and no ZScript name. It is consumed only
by the external Quest input layer. `HmdPosition` *is* exported (`actor.zs:277`) but is the
raw tracking-space position in **metres**, un-rotated and un-scaled
(`hw_vrmodes.cpp:445`) — you can difference it yourself, but it is the head, not a hand, and
it is in a different frame from `AttackPos`.

So: **a gesture system must difference positions itself**, at whatever rate its hook runs.
`vr_momentum` (`hw_vrmodes.cpp:120`) and `vr_momentum_threshold` (`:121`) are *head*
movement applied to the pawn's velocity in `player.zs`, not hand data.

#### 1.9 The derived quantities you should actually use

Everything below is well defined without knowing the external tracking-space sign
conventions (which are not in this tree — §10):

```zscript
// ui scope, inside RenderOverlay(RenderEvent e):
//   e.ViewPos is r_viewpoint.Pos == CenterEyePos (r_utility.cpp:1236), the SAME value
//   OffhandPos.Z was built from (hw_openxrdevice.cpp:608). The eye shift is applied to a
//   per-eye COPY inside HWDrawInfo (hw_entrypoint.cpp:162, :183), not to r_viewpoint, so
//   e.ViewPos carries no stereo shift.
Vector3 rel = pawn.OffhandPos - e.ViewPos;              // head -> hand, map units, same frame
double  fwd = rel.x * cos(pawn.Angle) + rel.y * sin(pawn.Angle);         // along pawn facing
double  rgt = rel.x * cos(pawn.Angle - 90) + rel.y * sin(pawn.Angle - 90);
double  up  = rel.z;                                     // map Z; × 1.2 / 34 for metres
double  metres = sqrt(rel.x*rel.x + rel.y*rel.y + (1.2*rel.z)*(1.2*rel.z)) / 34.0;
double  t = Level.time + e.FracTic;                      // continuous time, in tics
```

- `e.FracTic` is `r_viewpoint.TicFrac`, 0..1 within the current gametic
  (`src/events.cpp:2083`; field `src/events.h:370`; exported `events.cpp:1086`;
  ZScript `wadsrc/static/zscript/events.zs:65`). It is the only per-frame timestamp
  the pose comes with. `MSTimeF()` (`wadsrc/static/zscript/engine/base.zs:780`) is a wall
  clock — fine in `ui`, never in play scope.
- In play scope the nearest head reference is `(pawn.pos.xy, player.viewz)` — the
  un-interpolated tic value, which differs from `CenterEyePos` by the frame's interpolation.
- `RenderEvent` also carries `ViewAngle`, `ViewPitch`, `ViewRoll` and `Camera`
  (`events.zs:61-66`). `HudState` exists in C++ (`events.h:372`, set at `events.cpp:2109`)
  but is **not** exported (`events.cpp:1082-1087` lists six fields, not seven).

---

### 2. Where you may read, and where you may act

#### 2.1 The exact order inside a tic

From source, not from memory. `src/d_main.cpp` main loop (`:1278-1320`):

```
  TryRunTics();           // d_main.cpp:1314  -> G_Ticker() one or more times
  I_StartTic();           //           :1316
  D_ProcessEvents();      //           :1317  -> G_Responder -> InputProcess  (ui)
  D_Display();            //           :1318
```

`G_Ticker()` (`src/g_game.cpp:1243`):

```
  for each gameaction …                                     g_game.cpp:1272-1358
  for each player in game:                                  g_game.cpp:1372
      RunNetSpecs(i, buf)                                    :1381
        -> Net_DoCommand (d_net.cpp)
             DEM_NETEVENT  :2759  -> EventManager::Console   -> NetworkProcess        (play)
             DEM_ZSC_CMD   :2775  -> EventManager::NetCommand-> NetworkCommandProcess (play)
      G_WriteDemoTiccmd (if recording)                       :1385
      memcpy(cmd, newcmd)                                    :1397
      consistency check                                      :1407-1424
  localEventManager->UiTick()                                :1429          (ui)
  C_RunDelayedCommands()                                     :1430
  P_Ticker()                                                 :1436
  localEventManager->PostUiTick()                            :1470          (ui)
```

`P_Ticker()` (`src/p_tick.cpp:78`):

```
  interpolator.UpdateInterpolations()                        p_tick.cpp:84
  if (paused || P_CheckTickerPaused())                              :96
      -> run only MF8_RECREATELIGHTS pass, then RETURN              :99-113
  DPSprite::NewTick()                                               :116
  P_ResetSightCounters(false)                                       :145
  for each Level:
      for each actor: ClearInterpolation()   // Prev = Pos           :158-162
      P_ThinkParticles(Level)                                        :164
      for each player: P_PlayerThink()       // buttons, psprites,
                                             // weapon actions       :166-168
      localEventManager->WorldTick()                                 :171   (play)
      Level->Tick()                                                  :172
      Level->Thinkers.RunThinkers(Level)     // Actor.Tick            :173
      P_UpdateSpecials(Level)                                        :178
      Level->time++, maptime++, totaltime++                          :182-184
  StatusBar->CallTick()                                              :190
```

`D_Display()` (`src/d_main.cpp:949`):

```
  if ((gamestate == GS_LEVEL || GS_TITLELEVEL) && gametic != 0)      d_main.cpp:1063
      D_Render([]{ RenderView(&players[consoleplayer]); }, true)          :1069-1072
          -> RenderViewpoint -> vrmode->SetUp()     *** POSE WRITTEN ***
                                 hw_entrypoint.cpp:142-143
          -> per eye: vp.Pos += eye->GetViewShift(vp)  (on the DRAW INFO copy)
                                 hw_entrypoint.cpp:183
          -> V_DrawBlend(mainvp.sector)  per eye      hw_entrypoint.cpp:204
      StatusBar->DrawBottomStuff(state)                                   :1096/:1108/:1114
          -> RenderUnderlay()           src/g_statusbar/shared_sbar.cpp:1197   (ui)
      StatusBar->CallDraw(state, vp.TicFrac)                              :1102/:1109/:1115
      StatusBar->DrawTopStuff(state)                                      :1103/:1110/:1116
          -> RenderOverlay()            src/g_statusbar/shared_sbar.cpp:1234   (ui)
  NetUpdate()                                                             :1213
```

`TICRATE = 35` (`src/doomdef.h:61`; ZScript `base.zs:761`). **One tic = 28.571 ms.**

#### 2.2 The hook table

| Hook | Declared | Scope | Rate | Sees this frame's pose? | May change the world? |
|---|---|---|---|---|---|
| `StaticEventHandler.InputProcess(InputEvent)` | `events.zs:194` | `ui` | per input event, **before key bindings** (`g_game.cpp:1091` in `G_Responder`; dispatcher `events.cpp:953-963`) | previous frame's | no |
| `StaticEventHandler.UiProcess(UiEvent)` | `events.zs:193` | `ui` | per GUI event (`events.cpp:939-952`) | previous frame's | no |
| `StaticEventHandler.RenderUnderlay(RenderEvent)` | `events.zs:182` | `ui` | **per rendered frame**, before the HUD (`shared_sbar.cpp:1197`) | **yes** | no |
| `StaticEventHandler.RenderOverlay(RenderEvent)` | `events.zs:181` | `ui` | **per rendered frame**, after the HUD (`shared_sbar.cpp:1234`) | **yes** | no |
| `BaseStatusBar.Draw` / `Tick` | — | `ui` | frame / tic (`p_tick.cpp:190`) | yes / last frame's | no |
| `Weapon.ModifyBobLayer` / `ModifyBobLayer3D` | `weapons.zs` | `ui` | per frame per psprite layer | yes | no |
| `StaticEventHandler.UiTick` / `PostUiTick` | `events.zs:195-196` | `ui` | per tic (`g_game.cpp:1429`, `:1470`) | last frame's | no |
| `StaticEventHandler.NetworkProcess(ConsoleEvent)` | `events.zs:201` | **play** | per tic, **before** `P_PlayerThink` (`g_game.cpp:1381` → `d_net.cpp:2759` → `events.cpp:2324`) | last frame's | **yes** |
| `StaticEventHandler.NetworkCommandProcess(NetworkCommand)` | `events.zs:202` | **play** | same, `d_net.cpp:2775` → `events.cpp:2263-2275` | last frame's | **yes** |
| `P_PlayerThink` → weapon states / action functions | `p_tick.cpp:166-168` | play | per tic | last frame's | **yes** |
| `StaticEventHandler.WorldTick` | `events.zs:177` | play | per tic, after `P_PlayerThink` (`p_tick.cpp:171`) | last frame's | **yes** |
| `Actor.Tick` / `Thinker.Tick` | — | play | per tic, after `WorldTick` (`p_tick.cpp:173`) | last frame's | **yes** |
| `WorldThingDamaged` / `WorldThingDied` / `WorldThingSpawned` | `events.zs:162-167` | play | on event | yes-ish | **yes** |
| `WorldHitscanPreFired` | `events.zs:168` | play | inside `P_LineAttack` (`p_map.cpp:4718`) | yes | **veto only** (return true) |
| `WorldRailgunPreFired` | `events.zs:169` | play | inside `P_RailAttack` (`p_map.cpp:5623`) | yes | **veto only** |
| `WorldHitscanFired` / `WorldRailgunFired` | `events.zs:170-171` | play | after the trace (`p_map.cpp:5106`) | yes | **yes** |

**The gap, in numbers.** The pose is written in `D_Display` of frame *N*; the earliest play
hook that can act on it is `NetworkProcess` at the top of the next tic. On a 72 Hz display
and a 35 Hz tic, roughly two render frames land per tic. So:

- A `ui` detector sees ~2 samples per tic and can decide in the same frame it sampled.
- A play detector that differences `OffhandPos` across tics cannot *recognise* anything
  sooner than one tic (28.6 ms) after the motion started, and its sample is already
  0–14 ms stale when it reads it.
- Detect in `ui`, send one net command, act at the top of the next tic: **~29–43 ms** from
  threshold crossing to damage on the local machine, plus the session's tic buffering for
  peers.

#### 2.3 Scope rules you will hit

- `ui` may **read** play data (`players[]` is `native play @PlayerInfo players[MAXPLAYERS]`,
  `doombase.zs:22`, and reads from `ui` are legal) but may not write it, and may not call
  play functions.
- `AttackDir`/`OffhandDir` are `Actor` methods with no scope qualifier
  (`actor.zs:858-859`); `Actor : Thinker` is play, so **they cannot be called from `ui`.**
  A `ui` detector works from positions and the three Euler fields only — which is enough,
  given §1.9.
- `BlockThingsIterator` / `BlockLinesIterator` / `ThinkerIterator` carry no scope qualifier
  (`doombase.zs:201-238`) so the compiler will not stop you calling them from `ui`;
  `CreateFromPos` resolves the level through the C++ global `currentVMLevel`
  (`src/scripting/vmiterators.cpp:195`), which defaults to `&level`
  (`src/g_level.cpp:203`) and is only reassigned while ticking sublevels. It will *work*
  from `ui` on a single-level map, but anything you then do to the actors you find is a play
  write and is rejected. Treat iteration as play-scope work.
- `A_StartSound` is `action native clearscope` (`actor.zs:1199`) — callable from `ui`.
  Sound is presentational, so a `ui` sound is local and net-safe; see §6.2.
- `gametic` is `native readonly int` with no scope (`doombase.zs:14`), readable anywhere.
- Known ZScript hazard worth repeating here: `out Vector3` parameters compile and then abort
  the VM. Return vectors; `out` on a **struct** (`out FLineTraceData`, `out
  FTranslatedLineTarget`) is fine and is how the engine's own signatures work.

#### 2.4 Paused and frozen

`P_Ticker` returns early on `paused || P_CheckTickerPaused()` (`p_tick.cpp:96`) and
`P_CheckTickerPaused` is true for menus/console in single player (`p_tick.cpp:60-71`).
Crucially the comment at `p_tick.cpp:98` — "This must run even when the game is paused to
catch changes from netevents before the frame is rendered" — tells you `RunNetSpecs` has
already run by then: **net commands are delivered while paused, but no thinker runs.** A
melee hit resolved from `NetworkCommandProcess` while a menu is open will apply damage with
no thinker ticking afterwards. Gate on `Level.isFrozen()` / `paused` if that matters.

---

### 3. Hit resolution — every mechanism available

#### 3.1 `LineTrace` — read-only ray, the only hand-origin query with no side effects

```zscript
// actor.zs:805
native bool LineTrace(double angle, double distance, double pitch, int flags = 0,
                      double offsetz = 0., double offsetforward = 0., double offsetside = 0.,
                      out FLineTraceData data = null);
```

C++ `P_LineTrace` (`src/playsim/p_map.cpp:5165-5312`; prototype
`src/playsim/p_linetracedata.h:26-28`). Flags, `wadsrc/static/zscript/constants.zs:949-964`:

| Flag | Value | Effect (citation) |
|---|---|---|
| `TRF_ABSPOSITION` | 1 | `startpos = (offsetforward, offsetside, offsetz)` — the three "offsets" are absolute coordinates (`p_map.cpp:5207-5210`) |
| `TRF_ABSOFFSET` | 2 | offsets are map-absolute XY around the actor (`:5211-5214`) |
| `TRF_THRUSPECIES` | 4 | skip same-species actors (`:5172`, `:5158`) |
| `TRF_THRUACTORS` | 8 | skip all actors (`:5173`, `:5158`) |
| `TRF_THRUBLOCK` | 16 | clears `ML_BLOCKEVERYTHING` (`:5263`) |
| `TRF_THRUHITSCAN` | 32 | clears `ML_BLOCKHITSCAN` (`:5264`) |
| `TRF_NOSKY` | 64 | `TRACE_NoSky` (`:5266`) |
| `TRF_ALLACTORS` | 128 | actor mask `0xFFFFFFFF` instead of `MF_SHOOTABLE` (`:5249`) |
| `TRF_SOLIDACTORS` | 256 | adds `MF_SOLID` to the mask (`:5250`) |
| `TRF_BLOCKUSE` | 512 | adds `ML_BLOCKUSE` (`:5252`) |
| `TRF_BLOCKSELF` | 1024 | derives line flags from the tracer's own blocking (`:5253-5262`) |
| `TRF_ISOFFHAND` | 2048 | with `TRF_USEWEAPON`, use the off hand (`:5187`) |
| `TRF_USEWEAPON` | 4096 | origin `= OffhandPos`/`AttackPos`, direction `= OffhandDir`/`AttackDir` (`:5185-5201`) |

`FLineTraceData` (`src/playsim/p_linetracedata.h:9-24`, filled `p_map.cpp:5270-5310`):
`HitActor`, `HitLine`, `HitSector`, `Hit3DFloor`, `HitTexture`, `HitLocation` (= `trace.HitPos`),
`HitDir` (= `trace.HitVector`), `Distance`, `NumPortals`, `LineSide`, `LinePart`
(0 upper / 1 middle / 2 lower), `SectorPlane` (1 = ceiling), `HitType` (`ETraceResult`).
`memset` to zero first (`:5272`), so a false return leaves a clean struct.

Things only visible in the source:

- **`TRF_USEWEAPON` sets the direction independently of the position branch.** The direction
  override is at `:5185-5201`; the position is chosen separately at `:5207-5247`. So
  `TRF_USEWEAPON | TRF_ABSPOSITION` gives **your** origin with **the hand's** direction —
  useful, and nothing documents it.
- **Non-VR `LineTrace` starts at the actor's feet.** `startz = t1->Z() - Floorclip` (`:5182`),
  not `Center()`. `LineAttack` starts at `Center() + AttackOffset()` (`:4754`). Mixing the two
  silently shifts your melee origin by ~36 units when `OverrideAttackPosDir` is false.
- **The start sector is resolved from the start position**:
  `Trace(startpos, t1->Level->PointInSector(startpos), …)` (`:5269`). `LineAttack` does
  **not** (§3.2). This asymmetry matters for absolute-position traces.
- **`NumPortals` is `TData.NumPortals - 2`** (`:5308`) — two internal crossings are
  subtracted; it can read −2 on a non-portal trace in some paths. Do not trust it for
  "did this cross a portal" without testing.

Cost: one `Trace()`, i.e. a BSP/blockmap walk over `distance`. No allocation, no RNG, no
puff, no event, no world change. **This is the cheapest hand-origin query the engine has.**

#### 3.2 `LineAttack` — the full signature, and the uninitialised-origin bug

```zscript
// actor.zs:804
native Actor, int LineAttack(double angle, double distance, double pitch, int damage,
                             Name damageType, class<Actor> pufftype, int flags = 0,
                             out FTranslatedLineTarget victim = null,
                             double offsetz = 0., double offsetforward = 0., double offsetside = 0.);
```

C++: `AActor *P_LineAttack(AActor *t1, DAngle angle, double distance, DAngle pitch,
int damage, FName damageType, PClassActor *pufftype, int flags,
FTranslatedLineTarget *victim, int *actualdamage, double sz, double offsetforward,
double offsetside)` — `p_map.cpp:4714-5114`. (ZScript's `offsetz` is C++'s `sz`.)
An `FName` pufftype overload forwards at `:5116-5131`.

Flags, `constants.zs:936-947`:

| Flag | Value | Effect |
|---|---|---|
| `LAF_ISMELEEATTACK` | 1 | `puffFlags |= PF_MELEERANGE` (`p_map.cpp:4736`) |
| `LAF_NORANDOMPUFFZ` | 2 | `PF_NORANDOMZ` (`:4738-4739`) |
| `LAF_NOIMPACTDECAL` | 4 | suppresses the wall decal (`:4975`) |
| `LAF_NOINTERACT` | 8 | spawn and **return** the puff, no damage, no sound, no blood (`:4927-4944`, `:5041-5044`) |
| `LAF_TARGETISSOURCE` | 16 | damage source becomes `t1->target` (`:5073`) |
| `LAF_OVERRIDEZ` | 32 | `shootz = t1->Z()` instead of `Center()+AttackOffset()` (`:4780-4781`) |
| `LAF_ABSOFFSET` | 64 | offsets are map-absolute XY (`:4879-4882`) |
| `LAF_ABSPOSITION` | 128 | `tempos = (offsetforward, offsetside, sz)` — absolute coordinates (`:4875-4878`) |
| `LAF_ISOFFHAND` | 256 | off-hand origin/direction, off-hand weapon, off-hand haptic (`:4786`, `:4767`, `:4811`, `:4977`) |

Order of operations inside `P_LineAttack`:

1. `WorldHitscanPreFired` — returning true aborts and returns null (`:4718-4721`).
2. Default `direction` from the raw `angle`/`pitch`; `shootz = Center() − Floorclip +
   AttackOffset()` (`:4751-4754`). `AttackOffset()` for a player is
   `AttackZOffset * player->crouchfactor`, default `AttackZOffset 8`
   (`src/playsim/actorinlines.h:92-101`; `player.zs:119`).
3. **Haptics** for `t1->player && damage > 0`: 150 ms @ 0.8 on the attacking hand,
   plus 0.6 on the other hand if `weaponStabilised` (`:4758-4776`). This fires on hit *and*
   miss, before the trace. See §6.1.
4. `shootz += sz` (`:4782`), `LAF_OVERRIDEZ` applied first.
5. `OverrideAttackPosDir` branch: `fromPos = OffhandPos`/`AttackPos`, `direction` and the two
   offset bases from `OffhandDir`/`AttackDir` (`:4784-4800`). Otherwise
   `fromPos = t1->PosAtZ(shootz)` (`:4803`).
6. The weapon used for `MF2_THRUGHOST`, `MF5_PIERCEARMOR` and the decal generator is
   `(flags & LAF_ISOFFHAND) ? OffhandWeapon : ReadyWeapon` (`:4811`).
7. Puff species probe: a puff is **spawned and destroyed** to read `GetSpecies()` /
   `ThruBits` (`:4835-4844`). That is one `Spawn` + one `Destroy` per call, every call,
   whenever `pufftype != NULL`.
8. Origin selection (`:4872-4916`) — see the bug below.
9. `Trace(tempos, t1->Sector, direction, distance, MF_SHOOTABLE,
   ML_BLOCKEVERYTHING|ML_BLOCKHITSCAN, t1, trace, tflags, CheckForActor, &TData)` (`:4919`).
10. Miss: optional miss sound from the puff's `ActiveSound` on `CHAN_WEAPON` (`:4922-4925`);
    puff if `MF3_ALWAYSPUFF` or `LAF_NOINTERACT` (`:4930`).
    Wall: puff 4 units back along `HitVector` (`:4962-4966`), decal (`:4975-4999`), water
    splash (`:5000-5006`).
    Actor: blood or puff (`:5023-5032`), `victim` filled (`:5033-5040`), poison from the
    puff (`:5047-5050`), then `P_DamageMobj(trace.Actor, puff ? puff : t1, src, damage,
    damageType, DMG_INFLICTOR_IS_PUFF | DMG_PLAYERATTACK | DMG_USEANGLE,
    trace.SrcAngleFromTarget)` (`:5074`), then the `SpawnLineAttackBlood` virtual (`:5082-5086`)
    and `P_TraceBleed` (`:5090`).
11. `WorldHitscanFired(t1, tempos, puffpos, puff, flags)` (`:5106`).

**THE BUG — uninitialised attack origin.** `p_map.cpp:4872`:

```c
DVector3 tempos;                 // :4872  -- no initialiser
DVector3 puffpos;                // :4873
...
else if (t1->player != NULL && t1->player->mo->OverrideAttackPosDir)
{
    tempos += DVector3( … );     // :4890  -- accumulates onto garbage
```

`TVector3`'s default constructor is `= default` (`src/common/utility/vectors.h:351-357`), so
for an automatic variable this is uninitialised stack memory.
**Reachable exactly when: `OverrideAttackPosDir` is true (always, in VR), neither
`LAF_ABSPOSITION` nor `LAF_ABSOFFSET` is set, and `offsetforward != 0 || offsetside != 0`
(`:4883`).** The garbage origin is also what the `WorldHitscanFired` event reports as
`AttackPos` (`:5106`).

The identical bug is in `P_LineTrace`: `DVector3 startpos;` (`:5181`), `startpos +=`
(`:5221`), guarded by `:5215` and `:5219`.

Two safe forms:
- all three offsets zero → `tempos = fromPos` / `startpos = fromPos` (`:4883-4886`,
  `:5215-5218`);
- `LAF_ABSPOSITION` / `TRF_ABSPOSITION` → compute the point in script and pass it whole
  (`:4875-4878`, `:5207-5210`).

**Related: the offsets are half-disabled anyway.** `use_action_spawn_yzoffset` defaults
**false** (`hw_vrmodes.cpp:123`) and when false the side and vertical offsets are zeroed
inside the VR branch (`p_map.cpp:4896-4897`, `:5227-5228`, `:5668-5669`). It is a plain
`CVAR_ARCHIVE|CVAR_GLOBALCONFIG` user cvar read inside playsim — i.e. a stock desync source
as well as a surprise.

> **For melee: never pass the offset parameters. Use `LAF_ABSPOSITION` /
> `TRF_ABSPOSITION` with coordinates you computed.**

**A second trap specific to `LAF_ABSPOSITION`.** The trace's starting sector is
`t1->Sector` (`:4919`), the *attacker's* sector — not the sector containing `tempos`.
`P_LineTrace` uses `PointInSector(startpos)` (`:5269`); `P_LineAttack` does not. If your
absolute origin is across a line from the pawn (hand through a doorway, over a ledge), the
trace begins in the wrong sector. There is no flag to fix this; keep absolute origins inside
the pawn's own sector, or use `LineTrace` for the geometry and apply damage directly (§3.8).

#### 3.3 `AimLineAttack` — autoaim, and why its return value is not what you think

```zscript
// actor.zs:803
native double AimLineAttack(double angle, double distance,
                            out FTranslatedLineTarget pLineTarget = null,
                            double vrange = 0., int flags = 0,
                            Actor target = null, Actor friender = null);
```

C++ `P_AimLineAttack`, `p_map.cpp:4563-4652`. Flags `EAimFlags`, `constants.zs:923-934`:
`ALF_FORCENOSMART` 1, `ALF_CHECK3D` 2, `ALF_CHECKNONSHOOTABLE` 4, `ALF_CHECKCONVERSATION` 8,
`ALF_NOFRIENDS` 16, `ALF_PORTALRESTRICT` 32, `ALF_NOWEAPONCHECK` 64,
`ALF_IGNORENOAUTOAIM` 128, **`ALF_ISOFFHAND` 256**.

- `vrange == 0` picks a default: 35° without freelook, 0.5° if the hand's weapon has
  `WIF_NOAUTOAIM` (and that weapon is `(flags & ALF_ISOFFHAND) ? OffhandWeapon :
  ReadyWeapon`, `:4578`), else `clamp(userinfo.GetAimDist(), 0.5, 35)` (`:4569-4591`).
  **`userinfo.GetAimDist()` is a per-player userinfo cvar**, so autoaim width is
  replicated, but it is a user setting your mod does not control.
- With `OverrideAttackPosDir` and not `ALF_CHECKCONVERSATION`, `startPos` becomes the hand
  position and the aim yaw/pitch come from `OffhandDir`/`AttackDir` (`:4597-4613`).
- **But `shootz` stays on the body:** `shootz = t1->Center() − Floorclip + AttackOffset()`
  (`:4566`), and `aim.limitz = aim.shootz = shootz` (`:4623`) while `aim.startpos = startPos`
  (`:4621`). The traversal's pitch limits are therefore computed from the body's eye height
  and the ray's origin from the hand. This is inconsistent in stock and will skew autoaim
  when the hand is far from the chest.
- **The return value in VR is the pawn's pitch, always.** `:4646-4650`:
  ```c
  aimPitch = t1->Angles.Pitch;
  if (result->linetarget && (t1->player == NULL || !t1->player->mo->OverrideAttackPosDir))
      aimPitch = result->pitch;
  return aimPitch;
  ```
  So the classic idiom `pitch = AimLineAttack(...); LineAttack(angle, range, pitch, ...)`
  passes **zero pitch deviation** in VR — which, by §1.6, means "straight out of the hand".
  That is why `A_Punch` and `A_Saw` work in VR with no VR code. It also means
  **`AimLineAttack` cannot vertically autoaim a VR attack at all**; it only fills
  `pLineTarget`.

`FTranslatedLineTarget` gives `linetarget`, `angleFromSource`, `attackAngleFromSource`,
`unlinked`, plus a `TraceBleed` method (`constants.zs:912-921`). `attackAngleFromSource` is
the portal-correct angle (`p_map.cpp:5036-5038`); `angleFromSource` is the naive
`t1->AngleTo(actor)` unless the trace was unlinked.

Cost: a full `AimTraverse` over `distance`. Roughly the same order as a `Trace`, and for a
melee system it buys you almost nothing in VR. Skip it unless you want `linetarget`.

#### 3.4 `BlockThingsIterator` — the candidate-gathering primitive

```zscript
// doombase.zs:218-227
class BlockThingsIterator : Object native
{
    native Actor thing;        // :220
    native Vector3 position;   // :221
    native int portalflags;    // :222
    native static BlockThingsIterator Create(Actor origin, double checkradius = -1,
                                             bool ignorerestricted = false);           // :224
    native static BlockThingsIterator CreateFromPos(double checkx, double checky, double checkz,
                                                    double checkh, double checkradius,
                                                    bool ignorerestricted);            // :225
    native bool Next();        // :226
}
```

Thunks: `src/scripting/vmiterators.cpp:205-219` (`CreateBTI`, `CreateBTIFromPos`), `:233`
(`NextBTI`). Backed by `FMultiBlockThingsIterator`
(`src/playsim/p_maputl.cpp:1065-1073` and `:1075-1086`).

**What it actually does — read this before designing a hitbox.**

```c
// src/playsim/p_maputl.cpp:1075-1086  (the CreateFromPos constructor)
checkpoint.X = checkx; checkpoint.Y = checky; checkpoint.Z = checkz;
if (newsec == NULL) newsec = Level->PointInSector(checkx, checky);
basegroup = newsec->PortalGroup;
if (!check.inited) Level->CollectConnectedGroups(basegroup, checkpoint, checkz + checkh, checkradius, checklist);
checkpoint.Z = checkradius;        // <-- Z is OVERWRITTEN with the radius
Reset();
```
```c
// :1137-1144
void FMultiBlockThingsIterator::startIteratorForGroup(int group) {
    DVector2 offset = …getOffset(basegroup, group);
    offset.X += checkpoint.X; offset.Y += checkpoint.Y;
    bbox.setBox(offset.X, offset.Y, checkpoint.Z);   // half-width = radius
    blockIterator.init(bbox, false);
}
```

- **It is purely 2D.** `checkz` and `checkh` are used *only* to pick portal groups
  (`:1083`). There is no Z filtering whatsoever.
- **It is a square, not a circle.** `bbox.setBox(x, y, radius)` is an axis-aligned box of
  half-width `radius`, then every blockmap cell the box touches is walked
  (`FBlockThingsIterator::init`, `:913-921`).
- **Blockmap cells are 128 map units** (`src/gamedata/p_blockmap.h:46`,
  `MAPBLOCKUNITS = 128`). So the minimum candidate set is everything in one 128-unit cell;
  a radius of 1 and a radius of 60 usually cost the same.
- **Every distance, radius and height test is yours to write.** The iterator returns
  candidates, nothing more.
- **`position` is not the thing's position.** `item->Position = checkpoint + displacement`
  (`:1100`), i.e. your query point translated into the found actor's portal group — and its
  `.Z` is the radius you passed. To distance-test across portals, compare `it.thing.pos`
  against `it.position`, not against your own coordinates. `it.portalflags` is
  `FFCF_NOFLOOR`/`NOCEILING` for upper/lower portal groups (`:1114-1125`).
- **`ignorerestricted` does nothing.** It is accepted by both constructors (`:1065`,
  `:1075`) and referenced nowhere in the bodies; a tree-wide grep finds no other use.
- Duplicates across cells are suppressed by a hash (`:985-1010`), except for the fast path
  at `:990-993` for actors that do not span cells.

For a swung blade this is the right primitive: one iterator at the swept segment's midpoint
with `radius = halfLength + maxTargetRadius`, then your own capsule test per candidate
(§4). No trace, no puff, no engine quirk, no RNG, and fully deterministic given a
replicated segment.

`BlockLinesIterator` (`doombase.zs:229-238`) is the same shape for lines:
`CreateFromPos(Vector3 pos, double checkh, double checkradius, Sector sec = null)` — note it
takes a `Vector3`, unlike the things version. Use it to stop a swing at a wall without a
trace.

#### 3.5 `ThinkerIterator`, `ActorIterator`

```zscript
// doombase.zs:201-206
class ThinkerIterator : Object native
{
    native static ThinkerIterator Create(class<Object> type = "Actor",
                                         int statnum = Thinker.MAX_STATNUM+1);
    native Thinker Next(bool exact = false);
    native void Reinit();
}
// doombase.zs:208-216  ActorIterator  (TID-based; use Level.CreateActorIterator)
```

Cost: a linear walk of a statnum's thinker list, or of **all** statnums with the default
argument. For a melee hit test this is the wrong tool — `O(all thinkers)` per swing versus
`O(actors in 1–4 blockmap cells)`. Use it only for things with no spatial locality (e.g.
"every actor of class X on the map"), and pass a specific `statnum`
(`STAT_DEFAULT`, `MAX_STATNUM`) when you can.

#### 3.6 `CheckSight` — and the two reasons it is not a melee primitive

```zscript
// actor.zs:806
native bool CheckSight(Actor target, int flags = 0);
```
`P_CheckSight`, `src/playsim/p_sight.cpp:841-943`. Flags `ESightFlags`,
`constants.zs:971-977`: `SF_IGNOREVISIBILITY` 1, `SF_SEEPASTSHOOTABLELINES` 2,
`SF_SEEPASTBLOCKEVERYTHING` 4, `SF_IGNOREWATERBOUNDARY` 8.

1. **The eye height is fixed and is not yours.**
   `double lookheight = t1->Z() + t1->Height*0.75;` (`p_sight.cpp:910`). Not `viewz`, not
   the hand. For a 56-unit player that is Z+42. There is no parameter to move it.
2. **It consumes playsim RNG.** If the target is invisible and `SF_IGNOREVISIBILITY` is not
   set, `pr_checksight()` (or `pr_botchecksight()`) is rolled (`:874-884`) and the sight can
   succeed anyway. Calling `CheckSight` on a local-only code path therefore desyncs. Pass
   `SF_IGNOREVISIBILITY` to avoid the roll — then it is RNG-free.

Early rejections: `MF8_MVISBLOCKED` (`:852`), the reject matrix (`:862`), fake floor/ceiling
boundaries unless `SF_IGNOREWATERBOUNDARY` (`:888-905`). There is no sight budget — the
`sightcounts` array and `SightCycles` are statistics only (`P_ResetSightCounters`,
`:955-967`, called each tic at `p_tick.cpp:145`).

`CheckMeleeRange(double range = -1)` (`actor.zs:799`, `P_CheckMeleeRange`,
`src/playsim/p_enemy.cpp:267-307`) is the monster AI helper: it works on `actor->target`
only, uses `Distance2D` against `range + target->radius` (`:276-280`), rejects targets
wholly above `Top()` or below `Z()` unless `MF5_NOVERTICALMELEERANGE` (`:290-296`), and
rejects friends (`:299`). Body-based, target-based, 2D — useless for a hand.

#### 3.7 `RailAttack`

```zscript
// actor.zs:737
native void RailAttack(FRailParams p);
```
`struct FRailParams` at `doombase.zs:941-959`; C++ `P_RailAttack`,
`p_map.cpp:5600+`. Hand handling at `:5649-5688`, haptics at `:5690-5705`.

- **The flag name in ZScript is `RGF_ISOFFHAND = 64` (`constants.zs:252`), not
  `RAF_ISOFFHAND`.** The C++ enum is `RAF_ISOFFHAND = 64` (`src/playsim/p_local.h:394`) and
  **no `RAF_*` constant is exported to ZScript at all** (tree-wide grep over `wadsrc/`
  returns nothing). Use the `RGF_*` set at `constants.zs:246-252`.
- `A_RailAttack` (`stateprovider.zs:420-...`) sets `RGF_ISOFFHAND` itself from
  `weapon.bOffhandWeapon` (`:428-432`) — it is the one attack function that auto-detects the
  hand without you passing a flag.
- No uninitialised-origin bug here: `start` is assigned from the hand position (`:5655` /
  `:5662`) before the `+=` accumulation (`:5671`, `:5677`). The `use_action_spawn_yzoffset`
  zeroing is present (`:5668-5669`).
- `WorldRailgunPreFired` (`:5623`) can veto.
- `shootz` uses `source->FloatSpeed` and `RAF_CENTERZ` (`:5636-5641`) — a different formula
  again from `LineAttack`.
- Cost: a piercing `Trace` plus `P_DrawRailTrail` particles. Expensive, and a railgun trail
  is not what a blade looks like. Mentioned for completeness.

#### 3.8 Direct `DamageMobj` — the primitive a replicated melee system should use

```zscript
// actor.zs:801
native virtual int DamageMobj(Actor inflictor, Actor source, int damage, Name mod,
                              int flags = 0, double angle = 0);
```
`P_DamageMobj` → `static int DamageMobj(...)` at `src/playsim/p_interaction.cpp:1081`.
Flags `EDmgFlags`, `constants.zs:979-994`: `DMG_NO_ARMOR` 1, `DMG_INFLICTOR_IS_PUFF` 2,
`DMG_THRUSTLESS` 4, `DMG_FORCED` 8, `DMG_NO_FACTOR` 16, `DMG_PLAYERATTACK` 32,
`DMG_FOILINVUL` 64, `DMG_FOILBUDDHA` 128, `DMG_NO_PROTECT` 256, `DMG_USEANGLE` 512,
`DMG_NO_PAIN` 1024, `DMG_EXPLOSION` 2048, `DMG_NO_ENHANCE` 4096.

Returns the damage actually dealt, or −1 if the damage was cancelled
(`p_interaction.cpp:1079-1080`, `:1093`). Side effects you get for free: armour, invulnerability,
buddha, friendly fire scaling (`:1314-1328`), reflection (`:1290`), pain and wake-up via
`ReactToDamage` (`:886`), kickback via the `ApplyKickback` virtual (`:1307-1311`), death
states, obituaries, and the `WorldThingDamaged` / `WorldThingDied` events.

`DMG_USEANGLE` + `angle` makes the knockback use your angle instead of
`inflictor.AngleTo(target)` — which is exactly what a swing wants, since the inflictor for a
hand-based hit is either the pawn (wrong direction) or nothing at all. `P_LineAttack` passes
`DMG_USEANGLE` with `trace.SrcAngleFromTarget` (`p_map.cpp:5074`); `P_RailAttack` the same
(`:5795`).

**Local haptic side effect:** when the *target* is the console player, `P_DamageMobj` pulses
**both** controllers for 200 ms at `0.4 + 0.6·min(damage,100)/100`
(`p_interaction.cpp:1437-1444`). §6.1.

#### 3.9 `Thrust` and `ApplyKickback`

```zscript
// actor.zs:860
native void Thrust(double speed = 1e37, double angle = 1e37);
```
Defaults mean "use `Speed`" and "use `Angles.Yaw`". Adds to `Vel.XY`; it does not touch
`Vel.Z`. Deterministic, no RNG.

The damage path's knockback goes through a **ZScript virtual you can override**:

```zscript
// wadsrc/static/zscript/actors/interaction.zs:4
virtual void ApplyKickback(Actor inflictor, Actor source, int damage, double angle, Name mod, int flags)
```
called from `p_interaction.cpp:1307-1311`, gated at `:1300-1305` on: an inflictor that is not
the target, no `MF_NOCLIP`, no `MF2_NODMGTHRUST` on the inflictor, no `DMG_THRUSTLESS`, no
`MF7_DONTTHRUST` on the target, and no `MF2_NODMGTHRUST` on a player source.

The default body (`interaction.zs:4-77`):
```
kickback = inflictor.projectileKickback, else gameinfo.defKickback if no source/player/ReadyWeapon,
           else source.player.ReadyWeapon.Kickback                        // :10-15
kickback *= G_SkillPropertyFloat(SKILLP_KickbackFactor)                   // :17
ang      = angle if DMG_USEANGLE, else frandom[Kickback](0,360) if co-located,
           else origin.AngleTo(self)                                      // :22-36
thrust   = clamp(damage * 0.125 * kickback / Mass, 0, mod=='MDK' ? 10 : 32)  // :38-45
"fall forwards" flip: ang += 180, thrust *= 4, under random[Kickback](0,1)  // :46-59
Thrust(thrust, ang)                                                        // :72
```

**Stock limitation worth knowing: it reads `source.player.ReadyWeapon.Kickback`, never
`OffhandWeapon`** (`interaction.zs:15`). An off-hand melee weapon's `Kickback` property is
ignored; the main hand's is used. There is no flag; override `ApplyKickback` or do your own
`Thrust`. (`plrDontThrust` at `p_interaction.cpp:1086` is initialised false and never
reassigned — vestigial.)

Note the two `random[Kickback]` / `frandom[Kickback]` calls in the default body: the kickback
path consumes playsim RNG, so it must run identically everywhere.

#### 3.10 `TriggerPainChance` — the one RNG-free way to make a monster flinch

```zscript
// actor.zs:800
native bool TriggerPainChance(Name mod, bool forcedPain = false);
```
`p_interaction.cpp:975-1033`, thunk at `:1038-1044`. Returns `flinched` (did a Pain state
actually get set) when called from ZScript, versus `justhit` internally (`:1032`).

```c
if (forcedPain || (pr_damagemobj() < painchance))     // :991
```
`&&`/`||` short-circuit: **with `forcedPain = true` no RNG is rolled** for an ordinary
damage type. (`mod == 'Electric'` always rolls `pr_lightning()` at `:995`; `'PoisonCloud'`
rolls `pr_poison()` for monsters at `:1025`.) Per-damagetype `PainChances` are honoured
(`:982-989`); `MF5_NOPAIN` and `health < 1` reject (`:977`).

So `target.TriggerPainChance('Melee', true)` is a deterministic flinch with no RNG cost —
but it is still a world mutation, so it must be executed on every machine.

#### 3.11 Projectiles, and hand-relative spawning

- `SpawnPlayerMissile(type, angle, x, y, z, out pLineTarget, nofreeaim, noautoaim, aimflags,
  pitch)` — `actor.zs:772`; C++ `P_SpawnPlayerMissile`, `p_mobj.cpp:7470+`. Hand origin and
  direction at `:7484-7498`, under `OverrideAttackPosDir`, selected by `ALF_ISOFFHAND` in
  `aimflags`. Spawn sound goes to `CHAN_OFFWEAPON` for the off hand (`:7528`). Haptics at
  `:7545-7562` — but only if the missile does damage, has a damage type, or has a `SeeSound`
  (`:7547-7549`).
- `SpawnMissileAngleZSpeed(z, type, angle, vz, speed, owner, checkspawn, aimflags)` —
  `actor.zs:769`; hand handling at `p_mobj.cpp:7334-7343`.
- **`A_SpawnItemEx` with `SXF_RELATIVETOWEAPON`** (`1 << 29`, `constants.zs:130`) —
  `attacks.zs:412-558`. This is the only built-in that spawns *and* throws along the hand
  axis:
  - requires `player != null`, `flags & SXF_RELATIVETOWEAPON`, and
    `weapon && weapon == invoker && player.mo.OverrideAttackPosDir` (`:463-467`) — i.e. it
    only works from a weapon's own state, not from a thinker or an event handler;
  - `spawnpos = OffhandPos`/`AttackPos` (`:474`, `:482`);
  - `directionRoll = -OffhandRoll`/`-AttackRoll` (`:475`, `:483`);
  - direction from `OffhandDir(self, self.Angle + angle, self.Pitch)` (`:476`, `:484`), with
    `dir.x` = yaw and `dir.y` = pitch (`:489-490`);
  - `xofs` becomes forward along the hand, `yofs`/`zofs` side and up — **zeroed unless
    `use_action_spawn_yzoffset`** (`:492-493`);
  - `xvel` becomes speed along the hand, `yvel`/`zvel` side and up (`:513-530`);
  - `SXF_TRANSFERPITCH` / `SXF_TRANSFERROLL` copy the computed pitch/roll onto the spawned
    actor (`:548-555`).
  Note the asymmetry at `:465` vs `:476`: the yaw argument is `self.Angle + angle` but the
  pitch is the bare `self.Pitch` with no caller-supplied deviation — `A_SpawnItemEx` has no
  pitch parameter in this build (`:412`). I have not tested whether that was intended.
- `A_ChangeVelocity` with `CVF_RELATIVETOWEAPON` (`= 4`, `constants.zs:193`) — the same
  basis for an existing actor.
- `A_Recoil(xyvel)` — `p_actionfunctions.cpp:1285-1317`. **A no-op for players unless
  `vr_recoil`, which defaults false** (`:1294`; cvar at `:90`). When it does run it thrusts
  along `OffhandAngle + 90` / `AttackAngle + 90`, but only when called from a psprite
  (`:1299-1313`). Hand angles are local-only, so this desyncs. Don't use it.

#### 3.12 Also available, for completeness

`SpawnPuff(pufftype, pos, hitdir, particledir, updown, flags, victim)` (`actor.zs:787`),
`SpawnBlood(pos, dir, damage)` (`:788`), `BloodSplatter(pos, hitangle, axe)` (`:789`),
`TraceBleed(damage, missile)` / `TraceBleedAngle(damage, angle, pitch)` (`:795-796`),
`RadiusAttack(...)` (`:1263`), `LinePickActor` (C++ `p_map.cpp:5320`, exposed as
`actor.zs`'s `LinePickActor`), `Distance2D/3D[Squared]` (`:749-752`),
`Vec3To`/`Vec3Offset`/`Vec3Angle`/`Vec2OffsetZ` (`:847-853`), `AngleTo` (`:844`),
`AngleToVector` (`:483`), `SoundAlert` (`:727`), `LookForPlayers` (`:870`),
`GetDropItems` (`:865`), `A_DropInventory` (`:1228`).

#### 3.13 Cost, honestly

| Mechanism | Cost per call | Allocates | Playsim RNG | Changes the world |
|---|---|---|---|---|
| `LineTrace` | one BSP/blockmap `Trace` over `distance` | no | no | no |
| `BlockThingsIterator.CreateFromPos` + `Next` loop | the actors in ⌈2·radius/128⌉² cells, hash-deduped | one DObject | no | no |
| `BlockLinesIterator` | same, for lines | one DObject | no | no |
| `ThinkerIterator` | **all thinkers in the statnum(s)** | one DObject | no | no |
| `CheckSight` | a sight traverse + portal retries | no | **yes** unless `SF_IGNOREVISIBILITY` | no |
| `AimLineAttack` | an aim traverse | no | no | fills `pLineTarget` |
| `LineAttack` | one `Trace` **+ a puff Spawn/Destroy pair** (`p_map.cpp:4835-4844`) + possibly a second temporary puff (`:5069`) | 1–2 actors | no in itself; `DamageMobj`/pain/kickback do | yes |
| `RailAttack` | a piercing `Trace` + trail particles | puff + particles | the trail drift | yes |
| `DamageMobj` | the damage chain | no | pain + kickback | yes |
| `TriggerPainChance(mod, true)` | a state set | no | **no** | yes |
| `Thrust` | arithmetic | no | no | yes |
| `A_QuakeEx` | one `DEarthquake` thinker for `duration` tics | 1 thinker | only if `damrad > 0 && !QF_SHAKEONLY` | depends |

`LineAttack`'s puff probe is the hidden cost: a per-tic continuous slash through
`A_Saw`-style `LineAttack` spawns and destroys two actors per tic. A
`BlockThingsIterator` + `DamageMobj` path spawns none.

#### 3.14 Trap list

1. `AttackDir`/`OffhandDir` return `(yaw°, pitch°, 0)` from ZScript, **not** a vector
   (`p_actionfunctions.cpp:5700-5701`).
2. Their `angle`/`pitch` arguments are **deviations from the pawn's own angles**
   (`hw_vrmodes.cpp:401-402`). Pass `self.Angle, self.Pitch` for "straight out of the hand".
3. The **physical left hand's yaw deviation is mirrored** unless that hand's weapon has
   `+WEAPON.NOAUTOREVERSE` (`hw_vrmodes.cpp:464-469`, `weapons.zs:101`). An empty hand is
   mirrored too.
4. The hand ray's pitch is flattened by `pixelstretch` (`hw_openxrdevice.cpp:448`, `:464`).
5. **Never pass non-zero `offsetforward`/`offsetside` to `LineAttack` or `LineTrace` in VR** —
   uninitialised origin (`p_map.cpp:4872`/`:4890`, `:5181`/`:5221`).
6. Even if you fix that, `use_action_spawn_yzoffset` (default false,
   `hw_vrmodes.cpp:123`) zeroes the side and Z offsets (`p_map.cpp:4896-4897`) — and it is a
   local cvar read in playsim, so it desyncs.
7. `LineAttack` with `LAF_ABSPOSITION` still traces from **`t1->Sector`**
   (`p_map.cpp:4919`), not from the sector containing your origin. `LineTrace` resolves it
   properly (`:5269`).
8. Non-VR `LineTrace` starts at the **feet** (`:5182`); `LineAttack` at
   `Center() + AttackOffset()` (`:4754`).
9. `AimLineAttack` returns the **pawn's** pitch in VR (`:4646-4650`), and uses the body's
   `shootz` for its pitch limits while using the hand's position as origin (`:4621-4623`).
10. `BlockThingsIterator` does **no** Z test and **no** circle test; `position.Z` is the
    radius you passed; `ignorerestricted` is dead (`p_maputl.cpp:1075-1086`, `:1142`).
11. `CheckSight` rolls `pr_checksight` on invisible targets (`p_sight.cpp:879`) and looks
    from `Z + 0.75·Height` (`:910`).
12. Hand pose collapses to the body whenever a menu is open (`hw_openxrdevice.cpp:523` +
    `:568`) — a step discontinuity, not a freeze.
13. `cinemamode` changes the pitch formula and the yaw source (`:587`, `:600`, `:175-180`).
14. `SwitchWeaponHand` dereferences `nextweap.bTwoHanded` with no null check
    (`player.zs:2498`) after `PickNextWeapon` (`:2567`), which can return null. I did not
    reproduce the abort, but the check is plainly absent.
15. `ApplyKickback` and `SpawnLineAttackBlood` both read `player.ReadyWeapon` only
    (`interaction.zs:15`, `attacks.zs:874-876`) — off-hand weapon properties are ignored.
16. `A_QuakeEx` claims `CHAN_BODY` on its spot actor with a looping sound every tic
    (`a_quake.cpp:125-128`).

---

### 4. Swept tests — testing a moving segment against a moving target

Every melee system on this engine needs this, and **the engine gives you nothing swept.**
`Trace`/`P_LineTrace`/`P_LineAttack` are instantaneous rays against the world's current
state. `P_CheckPosition`/`P_TryMove` sweep an *actor's* cylinder, not an arbitrary segment,
and only as part of actually moving that actor.

#### 4.1 What the engine does give you

- **The attacker's segment, if you build it.** You have `OffhandPos` at frame *N−1* and
  frame *N* (you must keep the previous sample yourself). With `e.FracTic` you also have the
  elapsed time (§1.9).
- **The target's segment, for free.** `Actor.Prev` (`actor.zs:99`, `native vector3`, and
  writable) is set to `Pos()` by `AActor::ClearInterpolation()`
  (`src/playsim/actorinlines.h:29-35`), which runs for every actor at the top of `P_Ticker`
  **before any thinking** (`p_tick.cpp:158-162`). Therefore:
  - in `NetworkCommandProcess` (before `P_Ticker`), `thing.pos - thing.Prev` is the
    target's travel across the **previous** tic;
  - in `Actor.Tick` / `WorldTick` of tic *N*, `Prev` is the position at the start of tic *N*;
  - `PrevAngles` is maintained the same way (`actorinlines.h:32`) but is not exported under
    that name.
- **`Vel`** (`actor.zs:112`) — the target's per-tic velocity in map units, usable as a linear
  prediction when `Prev` is not what you want.
- **Candidate gathering** — `BlockThingsIterator.CreateFromPos` (§3.4).
- **`BlockLinesIterator`** — to clip the swing against geometry without a trace.

#### 4.2 What you must write

The test is a **capsule–capsule overlap in the anisotropic metric of §1.5.**

```
Attacker segment : A0 -> A1   (hand at frame N-1 and N, map units),  blade radius rb (metres)
Target  segment  : B0 -> B1   (thing.Prev -> thing.pos),             radius rt (map XY units),
                                                                     height h  (map Z units)

1. Work in a corrected space:  Ŝ = (S.x, S.y, 1.2 * S.z)
   so that Euclidean distance in Ŝ-space is physical, and 34 Ŝ-units = 1 metre.
   The target's cylinder becomes radius rt in XY and half-height 1.2*h/2 in Ẑ -- still a
   cylinder, not a sphere, so an exact test is capsule-vs-cylinder.

2. Cheap, adequate version: approximate the target as a capsule along its own vertical axis
   from (B.xy, B.z + rt) to (B.xy, B.z + h - rt) with radius rt, and test
       segmentDistance( Â0->Â1 , B̂0->B̂1 extended vertically )  <=  rt + 34*rb
   via the standard closest-point-between-two-segments routine (clamped s,t in [0,1]).

3. Relative-motion version (cheaper and usually better): subtract the target's motion.
       D  = (B1 - B0)                 // target travel this tic
       A0' = A0,  A1' = A1 - D        // attacker segment in the target's frame
   then test that single segment against the target's *stationary* cylinder at B0.
   One closest-point-to-segment call per candidate.
```

Neither the closest-point-between-segments routine nor a capsule test exists in ZScript; both
are a dozen lines of arithmetic with no natives involved, which is why this path is also the
cheapest and the most deterministic.

#### 4.3 Sampling, and what it costs you

At 72 Hz with a 35 Hz tic, a hand moving at 4 m/s covers 5.6 cm (1.9 map XY units) per render
frame and 11.4 cm (3.9 units) per tic. The segment approximation is a **chord** of the real
arc: for a 1 m-radius arm sweeping 4 m/s, the chord sags from the arc by
`r(1−cos(θ/2)) ≈ 0.4 mm` per frame and 1.6 mm per tic — negligible. The real error is not
geometric, it is temporal: the hand you test with is up to one render frame stale, and the
decision arrives one tic later (§2.2).

#### 4.4 The degenerate cases, and what each does if you ignore it

| Case | Symptom | Fix |
|---|---|---|
| `A0 == A1` (hand still, or two samples in the same frame) | zero-length segment; a naive `(A1-A0).Unit()` divides by zero → NaN propagates into `DamageMobj`'s angle and the actor's `Vel` | test `LengthSquared() < eps` first and fall back to a point-vs-cylinder test |
| `B0 == B1` (target still) | the relative-motion form (§4.2 step 3) is exact and cheapest | nothing |
| Both segments parallel | the closest-point solve's determinant is ~0 | clamp: fall back to testing the four endpoint pairs |
| Target radius 0 (`radius 0` actors, decorations) | capsule degenerates to a segment; `rt + 34rb` still positive, so it works | nothing, but `MF_SHOOTABLE` filtering matters more |
| Target taller than the swing and the swing horizontal | the 2D-only `BlockThingsIterator` returns it, your Z test rejects it correctly | remember the iterator did **no** Z test |
| Portals between hand and target | `BlockThingsIterator` crosses groups and gives you `it.position` translated into the target's group (`p_maputl.cpp:1100`); comparing against your own raw coordinates silently misses or double-hits | always distance-test against `it.position`, never your own XY |
| Target destroyed between gathering and damaging | `thing` is a dangling-safe TObjPtr in ZScript but `bDestroyed` may be set | check `thing && !thing.bDestroyed && thing.health > 0` inside the loop, not before it |
| Hand passing *through* a wall | there is no contact force and no collision on the hand; your capsule will happily hit a monster on the far side of a door | one `LineTrace` or `BlockLinesIterator` from the shoulder anchor to the hand, and clip the segment at the first solid |
| Same gesture sampled twice | two hits from one swing | latch per hand with an explicit cooldown or a direction-reversal test; nothing in the engine does this for you |
| Target already hit this swing | multi-hit on a wide sweep | keep a per-swing set of victims; `Actor` pointers are safe to store in a `play` array |
| `Prev` read in the wrong hook | in `Actor.Tick` after the target has already moved this tic, `pos - Prev` is this tic's *partial* travel | resolve hits in `NetworkCommandProcess` (before `P_Ticker`) or in `WorldTick` (after players, before thinkers) and know which you chose |
| A monster teleported | `pos - Prev` is a map-wide segment and sweeps everything | reject segments longer than some multiple of `Speed`, or check `bTeleport`-ish state |

---

### 5. Reaching back into the world

#### 5.1 Damage types

`DamageMobj(inflictor, source, damage, mod, flags, angle)`. `mod` is a free `Name`;
Pain/Death states can be specialised per damage type (`FindState(NAME_Pain, mod)`,
`p_interaction.cpp:998`, `:1017`). Stock melee uses `'Melee'`; `LineAttack` passes your `mod`
through unless the puff overrides it:

```c
// p_map.cpp:4854-4858
if ((damageType == NAME_None || damageType == NAME_Melee || damageType == NAME_Hitscan) &&
    puffDefaults != NULL && puffDefaults->DamageType != NAME_None)
        damageType = puffDefaults->DamageType;
```
So **a puff's `DamageType` silently overrides `'Melee'`** — but not an arbitrary custom name.
Choose a custom `mod` (e.g. `'Slash'`) if you want it to survive, and remember that
`p_interaction.cpp:1470-1475` substring-matches `"melee"`, `"strike"`, `"rip"`, `"tear"`,
`"slice"`, `"claw"` on the damage type for haptic-suit events (mobile builds only).

Useful flags for melee: `DMG_USEANGLE` (knockback along the swing), `DMG_THRUSTLESS`
(no knockback at all), `DMG_NO_PAIN`, `DMG_PLAYERATTACK` (obituaries; `P_LineAttack` sets it
automatically for players, `p_map.cpp:4762`), `DMG_FOILINVUL`, `DMG_NO_ARMOR`.

#### 5.2 Puffs and decals

`LAF_ISMELEEATTACK` gives the puff `PF_MELEERANGE` (`p_map.cpp:4736`), which is what makes
melee puffs behave like melee puffs. Other puff behaviour, all from the puff class's
defaults, read at `p_map.cpp:4807-4851`: `MF3_PUFFONACTORS`, `MF3_ALWAYSPUFF`,
`MF3_BLOODLESSIMPACT`, `MF3_SKYEXPLODE`, `MF2_THRUGHOST`, `MF2_THRUACTORS`,
`MF6_THRUSPECIES`, `MF6_MTHRUSPECIES`, `MF6_FORCEPAIN`, `MF6_NOTRIGGER`, `MF7_CAUSEPAIN`,
`MF7_NODECAL`, `MF7_FORCEDECAL`, `MF7_ALLOWTHRUFLAGS`, `MF8_ALLOWTHRUBITS` + `ThruBits`,
`MF4_SPECTRAL`, `PoisonDamage`/`PoisonDuration`/`PoisonPeriod`/`PoisonDamageType`
(`:5047-5050`), and `ActiveSound` as the miss sound (`:4922-4925`).

Decals: spawned only on `TRACE_HitWall`, non-`Line_Horizon`, non-visual-portal, without
`LAF_NOIMPACTDECAL` and without `MF7_NODECAL` (`:4975`). The generator is chosen from the
actor's `DecalGenerator`, the weapon's, or the puff's (`:4979-4998`), and
`SpawnShootDecal(…, hand)` is **hand-aware** (`:4977`, `:4984`, `:4986`, `:4992`, `:4997`).
Water splash on a floor hit (`:5005`), deep splash on crossing water (`:5095-5103`).

You can spawn all of this yourself without a trace: `SpawnPuff(pufftype, pos, hitdir,
particledir, updown, flags, victim)` (`actor.zs:787`), with `updown` 0 for a wall, 2 for an
actor (matching `p_map.cpp:4964`, `:5031`).

#### 5.3 Blood and the gore hook

`virtual void SpawnLineAttackBlood(Actor attacker, Vector3 bleedpos,
double SrcAngleFromTarget, int originaldamage, int actualdamage)` —
`wadsrc/static/zscript/actors/attacks.zs:873-898`, called **on the victim** from
`p_map.cpp:5082-5086`. Override it on your own monsters, or let stock run:
`SpawnBlood` normally, `BloodSplatter` if the attacker has `bBloodSplatter` or the weapon
has `bAxeBlood` — and that weapon is `player.ReadyWeapon` only (`attacks.zs:876`), so
`+WEAPON.AXEBLOOD` on an off-hand weapon does nothing. `random[LineAttack]()` at `:892`.

`P_TraceBleed` (`p_map.cpp:5090`; ZScript `TraceBleed`/`TraceBleedAngle`,
`actor.zs:795-796`) sticks blood on walls behind the victim.
`virtual class<Actor> GetBloodType(int type)` (`actor.zs:607`) chooses the blood class.
`virtual int GetGibHealth()` (`actor.zs:634`) controls extreme death.

#### 5.4 Pain and death

- Pain: `TriggerPainChance` (§3.10), or let `DamageMobj` → `ReactToDamage`
  (`p_interaction.cpp:886-968`) do it. `ReactToDamage` also handles wake-up
  (`:937-949`), `threshold`, `SeeState`, and `CallOkayToSwitchTarget`.
- Death: `virtual native void Die(Actor source, Actor inflictor, int dmgflags = 0,
  Name MeansOfDeath = 'none')` (`actor.zs:519`) — callable directly.
  `A_Die(name damagetype)` (`actor.zs:1077`). `virtual native int DoSpecialDamage(...)`
  (`:517`) and `TakeSpecialDamage(...)` (`:518`) are the per-class damage filters.
  `A_NoBlocking(bool drop = true)` (`:1176`) drops the item and clears `MF_SOLID`.
- `bJustAttacked` (set by `A_Saw` at `weaponchainsaw.zs:170` and `A_CustomPunch`'s
  `CPF_PULLIN` at `stateprovider.zs:409`) makes the monster's AI pull in toward the attacker.
- `DaggerAlert(Actor)` (`CPF_DAGGER`, `stateprovider.zs:410`) is the Hexen "quiet backstab"
  alert.
- `Howl()` (`actor.zs:876`), `A_Pain()` (`:1175`), `SoundAlert(target, splash, maxdist)`
  (`:727`).

#### 5.5 Events you can hang gore and reaction on

`WorldThingDamaged(WorldEvent e)` (`events.zs:166`) gives `Thing`, `Inflictor`, `Damage`,
`DamageSource`, `DamageType`, `DamageFlags`, `DamageAngle` and a **writable** `NewDamage`
(`events.zs:98`, `events.cpp:1108`). `WorldThingDied` (`:163`), `WorldThingGround` (`:164`),
`WorldThingRevived` (`:165`), `WorldThingDestroyed` (`:167`).

`WorldHitscanPreFired` (`:168`) returning true **cancels** someone else's hitscan before any
side effect (`p_map.cpp:4718-4721`) — the hook a shield or a parry wants. Its event fields
(`AttackPos`, `AttackAngle`, `AttackPitch`, `AttackDistance`, `AttackOffsetForward`,
`AttackOffsetSide`, `AttackZ`, `AttackPuffType`, `AttackLineFlags`) are declared
`native readonly` (`events.zs:100-110`), so you can inspect but not retarget. The handler
chain stops at the first `true` (`events.cpp:713-717`).

`WorldRailgunPreFired` (`:169`) receives `RailParams` (`events.zs:109`) and the C++ side
passes the **live** `FRailParams*` (`events.cpp:722-724`). Whether
`e.RailParams.damage = 0` compiles through a `readonly` struct field is worth testing — if it
does, it mutates the real shot. I did not test it.

#### 5.6 Dropping and throwing actors

- `A_DropInventory(class<Inventory> itemtype, int amount = -1)` (`actor.zs:1228`) and
  `DropWeapon(hand)` (`weapons.zs:326`) — the stock disarm paths.
- `A_NoBlocking(drop)` (`actor.zs:1176`) runs the actor's `DropItem` list.
- `GetDropItems()` (`actor.zs:865`) to inspect it.
- To **throw**: `Spawn(...)` then set `Vel` yourself, or `A_SpawnItemEx` with
  `SXF_RELATIVETOWEAPON` (§3.11) for a hand-axis throw, or
  `A_ChangeVelocity(..., CVF_RELATIVETOWEAPON)` for something already in flight.
  There is no "hand velocity" to inherit (§1.8) — a thrown object's speed must be a number
  your detector integrated.
- `Thrust(speed, angle)` (`actor.zs:860`) moves XY only; set `Vel.Z` directly for an arc.

#### 5.7 What a mod can and cannot make a monster do

**Can**, from play scope, with no engine change:
set any state (`SetState`, `SetStateLabel`, `actor.zs:836-839`), force pain (§3.10), damage
and kill (`DamageMobj`, `Die`), set `target`/`lastenemy`/`threshold`/`reactiontime`, call
`A_Chase`/`A_Wander`/`A_Face` (`:1179`, `:1193`, `:1154`), move it (`SetOrigin`,
`SetXYZ`, `Thrust`, `Vel`), flag it (`bFrightened`, `bDormant`, `bNoBlood`, `bInvulnerable`,
`bNoPain`, `MF7_DONTTHRUST`, …), alert it (`SoundAlert`, `DaggerAlert`), morph it, drop its
items, and override every damage/blood/kickback virtual listed above on **your own** classes.

**Cannot**:
- Override a virtual on a class you did not define without replacing the class — there is no
  mix-in or hook injection. For third-party monsters you are limited to the global
  `WorldThing*` events and to flags/fields.
- Make a monster flinch *without* a state: `TriggerPainChance` sets `Pain`; if the class has
  no `Pain` state, nothing visible happens and `flinched` is false
  (`p_interaction.cpp:998-1003`, `:1017-1022`).
- Interrupt a monster's attack mid-animation except by forcing a state — and forcing a state
  out of an attack sequence may skip an `A_*` call the class relied on.
- Produce a staggered/knocked-down state that stock does not have. There is no ragdoll, no
  physics body, no bone control (no bone natives in stock `actor.zs`; the model natives are
  `SetAnimation`/`SetAnimationFrameRate`/`SetModelFlag`, `actor.zs:1340-1348`).
- Feel contact. Hitting a wall and hitting air give identical data (§8).

---

### 6. Feedback

#### 6.1 Haptics — every route in stock

There is **no direct haptic call in ZScript.** `VRMode::Vibrate(duration, channel, intensity)`
is C++ only (`hw_vrmodes.h:94`), implemented as
`OpenXRDeviceMode::Vibrate → QzDoom_Vibrate` (`hw_openxrdevice.cpp:755-758`), and
`QzDoom_Vibrate`'s body is **not in this tree** (declared `hw_openxrdevice.cpp:140`, built
from `../../QzDoom/` per `mobile/Android_src.mk:558-565`). Channel 0 = left controller,
channel 1 = right controller (`r_utility.cpp:1090-1091` comments, consistent with
`p_map.cpp:4766-4768`). Duration is in milliseconds, intensity 0..1.

Hand mapping everywhere: `rightHanded = vr_control_scheme < 10`, then inverted by the
off-hand flag (`p_map.cpp:4766-4767`, `:5694-5695`, `p_mobj.cpp:7552-7553`).

Every `Vibrate` call site in stock:

| Route | Site | Duration / intensity | Hand(s) | Trigger | Net-safe? |
|---|---|---|---|---|---|
| **Damaging `P_LineAttack`** | `p_map.cpp:4768` | 150 ms @ 0.8 (+150 ms @ 0.6 on the other hand if `weaponStabilised`, `:4772`) | the attacking hand | `t1->player && damage > 0`, **before the trace**, so hit *and* miss | the `LineAttack` is; the pulse is local presentation |
| **Damaging `P_RailAttack`** | `p_map.cpp:5697` (+`:5701`) | same | attacking hand | `p->damage > 0` | same |
| **`P_SpawnPlayerMissile`** | `p_mobj.cpp:7555` (+`:7559`) | same | attacking hand | missile does damage, or has a damage type, or has a `SeeSound` (`:7547-7549`) | same |
| **Taking damage** | `p_interaction.cpp:1443-1444` | 200 ms @ `0.4 + 0.6·min(damage,100)/100` | **both** | `player == Level->GetConsolePlayer()` — console player only | local-only by construction |
| **Poison tick** | `p_interaction.cpp:1930-1931` | 500 ms @ level | both | poison damage to the console player | local |
| **Health gain** (`P_GiveBody`) | `p_mobj.cpp:943-944` | 100 ms @ `0.4 + 0.6·num/100` | both | `player == &players[consoleplayer]` | local |
| **Mugshot / pickup** | `src/g_statusbar/sbar_mugshot.cpp:348-349` | 80 ms @ `vr_pickup_haptic_level` (default 0.2, `hw_vrmodes.cpp:126`) | both | mugshot state change | local, render-thread |
| **Pickup flash** | `src/rendering/2d/v_blend.cpp:132-137` | 50 ms @ `vr_pickup_haptic_level` | both | `CPlayer->bonuscount != 0`, **every call to `V_AddPlayerBlend`** | local, render-thread |
| **Earthquake** | `r_utility.cpp:1090-1091` | 10 ms per rendered frame, intensity `QuakePower(vr_quake_haptic_level, jiggers.Intensity.X or .Y, …)` | **X → left, Y → right** | a live `DEarthquake` in `tremrad` of the camera, and `!paused` (`:1061`) | **the thinker is replicated; each client's own camera decides** |

Two of these are reachable from ZScript as *deliberate* feedback channels:

**(a) `A_QuakeEx` — the only per-hand, duration-controlled, net-safe route.**

```zscript
// actor.zs:1284
native void A_QuakeEx(double intensityX, double intensityY, double intensityZ, int duration,
                      double damrad, double tremrad, sound sfx = "world/quake", int flags = 0,
                      double mulWaveX = 1, double mulWaveY = 1, double mulWaveZ = 1,
                      double falloff = 0, int highpoint = 0, double rollIntensity = 0,
                      double rollWave = 0, double damageMultiplier = 1,
                      double thrustMultiplier = 0.5, int damage = 0);
```

- `intensityX` drives the **left** controller, `intensityY` the **right**
  (`r_utility.cpp:1086-1091` against `jiggers.Intensity.X/.Y`).
  **Only the non-relative intensities feed haptics** — `QF_RELATIVE` routes them to
  `RelIntensity` (`a_quake.cpp:377-388`) which the haptic block never reads. `rollIntensity`
  and `intensityZ` do not produce haptics either.
- **No view shake on a default install**: the view offsets use `r_quakeintensity`
  (`r_utility.cpp:1066`), which is `CUSTOM_CVAR(Float, r_quakeintensity, 0.0f, …)` with the
  comment "Defaulted to 0 for VR" (`r_utility.cpp:116`). The haptics use
  `vr_quake_haptic_level`, default **0.8** (`hw_vrmodes.cpp:127`). The user can raise
  `r_quakeintensity`, so do not *rely* on no shake.
- `damrad: 0` skips the whole damage/thrust block (`a_quake.cpp:130`). `QF_SHAKEONLY`
  (`1<<9`, `constants.zs:677`) only suppresses the damage roll (`a_quake.cpp:193`) — the
  **thrust still applies** (`:206-211`) while `damrad > 0`. Use `damrad: 0`.
- Intensities are clamped 0..9 (`a_quake.cpp:439-441`). `duration` is in tics, minimum 1
  (`--m_Countdown == 0 → Destroy`, `:162-169`), i.e. **28.6 ms granularity**.
- The haptic is re-issued **every rendered frame** at 10 ms, so it reads as continuous
  rumble for the quake's duration, not as a pulse.
- `DEarthquake` is an ordinary serialised thinker (`a_quake.cpp:88-99`), created through
  `Level->CreateThinker<DEarthquake>` (`:447`, `:458`), so it is **replicated,
  demo-recorded and savegame-safe** — the only haptic route that is.
- **`tremrad` gates on the camera's distance to the spot** (`a_quake.cpp:354-370`,
  2D by default or 3D with `QF_3D` at `:358`), so a quake on one co-op player's pawn rumbles
  any player standing close. Keep `tremrad` tight.
- **It claims `CHAN_BODY` on the spot actor** and restarts a looping sound every tic:
  `if (!S_IsActorPlayingSomething(m_Spot, CHAN_BODY, m_QuakeSFX)) S_Sound(m_Spot, CHAN_BODY,
  CHANF_LOOP, m_QuakeSFX, 1, ATTN_NORM);` (`a_quake.cpp:125-128`). Pass `sfx: ""` and accept
  that `CHAN_BODY` on that actor is spoken for while the quake lives.
- `QF_GROUNDONLY` (`1<<7`) suppresses it when the victim is off the floor (`:354`).

**(b) `player.bonuscount` — a both-hand blip with a visible side effect.**
`player.bonuscount` is `native uint` and writable (`player.zs:3087`; C++
`d_player.h`, field exported `p_user.cpp:1894`). While it is non-zero,
`V_AddPlayerBlend` issues a 50 ms pulse on both controllers at `vr_pickup_haptic_level`
(`v_blend.cpp:131-137`) **and** tints the screen with `gameinfo.pickupcolor`
(`:147-149`). It decrements once per tic in `PlayerThink` (`player.zs:1801`). In stereo VR,
`V_DrawBlend` is called once per eye inside the eye loop (`hw_entrypoint.cpp:204`), so the
pulse is issued twice per frame. Local-only. A hack, but it is there, and it is the only
both-hand short blip that needs no thinker.

**The haptic-suit channel.** `VR_HapticEvent(const char* event, int position, int intensity,
float angle, float yHeight)` (`hw_vrmodes.h:97`) is a separate route for bHaptics-style
vests: a named event, a position code, an intensity 0..100 scaled by
`C_GetExternalHapticLevelValue(name)` (which looks up the cvar `ext_haptic_level_<name>`
times `ext_haptic_level_global_intensity`, `src/common/console/c_cvars.cpp:1442-1453`), and
an angle. Its body in this tree is an **empty stub**
(`src/QzDoom/qzdoom_common.cpp:65-67`), and several call sites are inside
`#if defined(__MOBILE__) && !defined(_WIN32)` (`p_interaction.cpp:1445-1505`). Event names
used by stock: `fire_weapon`, `pickup`, `pickup_weapon`, `doorclose`, `slime`, `fire`,
`poison`, `melee`, `fireball`, `shotgun`, `bullet`, `healstation`, `heartbeat`,
`rumble_front`, `rumble_back`. Not reachable from ZScript except through those sites.

**`vr_enable_haptics`** (`CVAR(Bool, vr_enable_haptics, true, …)`, `hw_vrmodes.cpp:125`) is
declared and **never read anywhere in this tree** — it is consumed in the external Android
layer. So the user can switch all haptics off and script has no way to detect it.

#### 6.2 Sound placement, and what spatialises

- `A_StartSound(sound, int slot = CHAN_BODY, int flags = 0, double volume = 1.0,
  double attenuation = ATTN_NORM, double pitch = 0.0, double startTime = 0.0)` —
  `actor.zs:1199`, `action native clearscope`. Attached to the calling actor, so it **follows
  it** and spatialises from its position.
- **`S_StartSoundAt(Vector3 pos, Sound, int channel, int flags, volume, attenuation, pitch,
  startTime)`** — `doombase.zs:136`, `native static`. A positional sound at an arbitrary
  point, with no actor. This is what a swing or an impact at the hand wants, since the hand
  is not an actor. `S_StartSound` (`:135`) is the non-positional form.
- Channels, `wadsrc/static/zscript/engine/base.zs:1-41`: `CHAN_AUTO` 0, `CHAN_WEAPON` 1,
  `CHAN_VOICE` 2, `CHAN_ITEM` 3, `CHAN_BODY` 4, `CHAN_5..7`, and
  **`CHAN_OFFWEAPON = 5`** (`base.zs:42`; C++ `src/common/audio/sound/s_soundinternal.h:156`)
  — a QZD addition so the off hand does not cut the main hand off. Flags: `CHANF_LOOP`,
  `CHANF_NOSTOP`, `CHANF_OVERLAP`, `CHANF_MAYBE_LOCAL`, `CHANF_LOCAL`, `CHANF_UI`
  (not saved), `CHANF_NOPAUSE`, `CHANF_SINGULAR`, `CHANF_FORCE`, `CHANF_TRANSIENT`.
  Attenuations: `ATTN_NONE` 0, `ATTN_NORM` 1, `ATTN_IDLE` 1.001, `ATTN_STATIC` 3
  (`base.zs:44-47`).
- **The engine auto-redirects `CHAN_WEAPON` to `CHAN_OFFWEAPON` only when the action runs
  from the off-hand psprite** — `ACTION_CALL_FROM_PSPRITE()` plus
  `pspr->GetID() == PSP_OFFHANDWEAPON` in `A_StopSound` (`vmthunks_actors.cpp:148`),
  `A_SoundPitch` (`:176`), `A_SoundVolume` (`:195`), `A_PlaySound` (`:220`),
  `A_StartSound` (`:244`), `A_StartSoundIfNotSame` (`:276`). From a thinker or an event
  handler **there is no redirect** — pass `CHAN_OFFWEAPON` yourself.
- Sound is pure presentation in GZDoom: it is not in the consistency sum, consumes no
  playsim RNG, and does not affect actor state. So a `ui`-scope sound is local and harmless,
  and a play-scope sound plays on every machine from the same data. `CHANF_UI` keeps it out
  of savegames.
- `S_GetLength(sound)` (`doombase.zs:140`) if you need to time a swing to its whoosh.

#### 6.3 Screen and world effects from play scope

- `A_SetBlend(color1, alpha, tics, color2 = 0, alpha2 = 0)` — `actor.zs:1229`.
  Writes `player->BlendR/G/B/A`-equivalent state; applied at `v_blend.cpp:366-370` for the
  console player's camera.
- `A_QuakeEx` view shake — zero by default in VR (`r_quakeintensity`, `r_utility.cpp:116`).
  `rollIntensity`/`rollWave` roll the view (`r_utility.cpp:1067-1068`); in a headset that is
  a nausea source, not an effect.
- `A_SpawnParticle(...)` / `A_SpawnParticleEx(...)` — `actor.zs:1225-1226`. Particles are
  invisible to the simulation, consume no RNG of their own, are not saved, and are the
  cheapest swing trail. `P_ThinkParticles` runs them at `p_tick.cpp:164`.
- `VisualThinker` — `wadsrc/static/zscript/visualthinker.zs:1-89`. A serialised, play-scope,
  camera-facing textured quad with `Pos`, `Prev`, `Vel`, `Scale`, `Offset`, `Roll`, `Alpha`,
  `Texture`, `Translation`, `LightLevel`, `scolor`, a render style, and
  `Spawn(type, tex, pos, vel, alpha, flags, roll, scale, offset, style, trans, vtflags)`
  (`:39-40`). Heavier than a particle, savegame-safe, and interpolated via `Prev`.
  (Reminder from this project's own notes: a non-`Normal` RenderStyle model loses depth —
  prefer `STYLE_Normal` for anything opaque.)
- `SetCamera(Actor cam, bool revert)` — `actor.zs:941`. Note §1.3: this moves which pawn
  receives the controller pose.
- `player.extralight` — the muzzle-flash light hack `A_Saw` uses
  (`weaponchainsaw.zs:94`, `:105-113`).

---

### 7. Netplay

#### 7.1 What replicates and what does not

| Thing | Replicated? | Proof |
|---|---|---|
| `player.cmd.ucmd.buttons`, incl. `BT_OFFHANDATTACK` (`1<<26`), `BT_OFFHANDALTATTACK` (`1<<27`), `BT_OFFHANDRELOAD` (`1<<28`) | **yes** | `usercmd_t.buttons` is a packed field (`src/d_protocol.h:66`, `UCMDF_BUTTONS` `:79`); filled from `Button_OH_*` at `src/g_game.cpp:775-777`; readable in play scope for **every** player via `players[i].cmd.buttons` (`player.zs:3134`, `:2-11`) |
| `ucmd.pitch`, `ucmd.yaw` | **yes** | `d_protocol.h:67-68`, `UCMDF_PITCH`/`UCMDF_YAW` `:80-81`; written `g_game.cpp:849`, `:875`; VR HMD deltas enter through `G_AddViewAngle`/`G_AddViewPitch` from the render frame (`hw_openxrdevice.cpp:715`, `:731`) |
| `ucmd.roll` | **the field exists and is never written** | declared `d_protocol.h:69` with `UCMDF_ROLL` `:85`; no assignment to `ucmd.roll` anywhere in `g_game.cpp`. HMD roll reaches `vp.HWAngles.Roll` only (`hw_openxrdevice.cpp:740`) |
| `AttackPos`/`OffhandPos`/`*Angle`/`*Pitch`/`*Roll` | **no** | written by the renderer for `r_viewpoint.camera->player` only (`hw_vrmodes.cpp:434`, `hw_openxrdevice.cpp:551`); not in `usercmd_t` (`d_protocol.h:64-73`); not in the savegame |
| `OverrideAttackPosDir`, `player.PlayInVR` | **no** | same writer (`hw_vrmodes.cpp:437-438`) |
| `weaponStabilised` / `WF_TWOHANDSTABILIZED` | **no** | C++ global (`qzdoom_common.cpp:19`) set by the render/input thread, read from playsim at `p_pspr.cpp:512-515`; the bit is clear for remote players |
| `switchhand` | **no** | `CCMD(switchhand)` does a direct `VMCall` (`g_game.cpp:335-350`), not a `DEM_*` write — immediate, local, absent from demos |
| `vanilla_melee_attack`, `puristmode`, `use_action_spawn_yzoffset`, `vr_momentum`, `vr_recoil`, `vr_control_scheme`, `userinfo.GetAimDist()` | **no** (aimdist is userinfo, so it *is* known, but it is the user's) | `g_game.cpp:132-133`, `hw_vrmodes.cpp:96`, `:120`, `:123`, `p_actionfunctions.cpp:90` — all `CVAR_ARCHIVE|CVAR_GLOBALCONFIG`, **not** `CVAR_SERVERINFO`, and all branched on inside playsim |
| `SendNetworkEvent(name, a1, a2, a3)` | **yes** | `DEM_NETEVENT` (`events.cpp:381-395`), decoded `d_net.cpp:2759-2768` → `NetworkProcess` |
| `SendNetworkCommand(name, ...)` | **yes** | `DEM_ZSC_CMD` (`events.cpp:397-493`), decoded `d_net.cpp:2775-2791` → `NetworkCommandProcess` |
| `SendNetworkBuffer(name, NetworkBuffer)` | **yes** | `events.cpp:495-539`, same `DEM_ZSC_CMD` |
| `A_QuakeEx` | **yes** (the thinker) | `Level->CreateThinker<DEarthquake>`, serialised (`a_quake.cpp:88-99`, `:447`) |
| `LineAttack`/`LineTrace` from numbers everyone has | **yes** | pure playsim once origin and direction are replicated values |
| Particles, `A_SetBlend`, haptics | presentation only | not in the consistency sum |

**Stock VR aiming is already non-deterministic.** On the shooter's machine
`OverrideAttackPosDir` is true and the shot leaves the controller
(`p_map.cpp:4784-4800`); on every peer it is false and the shot leaves the body along the
view angles (`:4803`). Hits and damage diverge. The consistency check compares the RNG seed
sum plus `int((X+Y+Z)*257) + Yaw.BAMs() + Pitch.BAMs() ^ health`, `BACKUPTICS*ticdup` tics
late, and on mismatch only records `players[i].inconsistant`
(`g_game.cpp:1407-1424`). So a melee mod that replicates its own pose is **strictly better
than stock**, not worse.

#### 7.2 The recipe, with the exact calls

```zscript
// ---- ui, once per rendered frame -------------------------------------------------
class MyMeleeUI : StaticEventHandler
{
    // ui state: ring buffer of samples per hand
    override ui void RenderOverlay(RenderEvent e)
    {
        let p = players[consoleplayer];
        if (!p.mo || gamestate_is_not_level_or_menu_open()) { reset(); return; }  // §1.4
        Vector3 rel = p.mo.OffhandPos - e.ViewPos;        // §1.9, same frame, no eye shift
        double  t   = Level.time + e.FracTic;             // continuous tics
        push(rel, p.mo.OffhandAngle, p.mo.OffhandPitch, p.mo.OffhandRoll, t);
        if (gestureFired())
        {
            // send the SEGMENT in map coords, plus the verb. Nothing derived locally
            // may reach the simulation except through this packet.
            let b = new("NetworkBuffer");
            b.AddMapUnit(p0.x); b.AddMapUnit(p0.y); b.AddMapUnit(p0.z);   // 4 bytes each
            b.AddMapUnit(p1.x); b.AddMapUnit(p1.y); b.AddMapUnit(p1.z);
            b.AddAngle(worldYaw);  b.AddAngle(worldPitch); b.AddAngle(roll); // 4 bytes each
            b.AddInt16(int(speedMapUnitsPerTic * 64));
            b.AddInt8(hand | (verb << 1));
            EventHandler.SendNetworkBuffer('mymod_swing', b);
        }
    }
}

// ---- play, top of the next tic, on EVERY machine ---------------------------------
override void NetworkCommandProcess(NetworkCommand cmd)
{
    if (cmd.Command != 'mymod_swing') return;
    Vector3 p0 = (cmd.ReadMapUnit(), cmd.ReadMapUnit(), cmd.ReadMapUnit());
    Vector3 p1 = (cmd.ReadMapUnit(), cmd.ReadMapUnit(), cmd.ReadMapUnit());
    double yaw = cmd.ReadAngle(), pit = cmd.ReadAngle(), rol = cmd.ReadAngle();
    int    spd = cmd.ReadInt16();
    int    bits= cmd.ReadInt8();
    let pawn = players[cmd.Player].mo;            // cmd.Player is the originator
    // ... capsule sweep (§4) via BlockThingsIterator.CreateFromPos, then DamageMobj
    //     with DMG_USEANGLE and your own angle; or LineAttack with LAF_ABSPOSITION.
}
```

API, exact declarations:

- `EventHandler.SendNetworkEvent(String name, int arg1 = 0, int arg2 = 0, int arg3 = 0)` —
  `events.zs:225`; thunk `events.cpp:1660-1670`. Three int32s, period.
- `EventHandler.SendNetworkCommand(Name cmd, ...)` — `events.zs:226` (`vararg`,
  `version("4.12")`); thunk `events.cpp:1641-1649`. Tags from `ENetCmd`
  (`events.zs:1-9`): `NET_INT8` 1, `NET_INT16` 2, `NET_INT` 3, `NET_FLOAT` 4,
  `NET_DOUBLE` 5, `NET_STRING` 6.
- `EventHandler.SendNetworkBuffer(Name cmd, NetworkBuffer buffer)` — `events.zs:227`;
  thunk `events.cpp:1651-1658`. `class NetworkBuffer` at `events.zs:37-57` with
  `AddInt8/16/Int/Float/Double/String` and the wrappers `AddName`, **`AddMapUnit`** (double
  → 16.16 fixed int, 4 bytes), **`AddAngle`** (double → BAM int, 4 bytes), `AddVector2/3/4`,
  `AddQuat`, `AddIntArray`, `AddDoubleArray`, `AddStringArray`.
- `struct NetworkCommand` at `events.zs:11-35`: `readonly int Player`,
  `readonly Name Command`, `ReadInt8/16/Int/Float/Double/String`, `EndOfStream`, and the
  matching `ReadName`, **`ReadMapUnit`** (`events.cpp:1218-1222`, `× 1/65536`),
  **`ReadAngle`** (`events.cpp:1231`, `DAngle::fromBam`), `ReadVector2/3/4` (three/four
  **doubles**, `events.cpp:1253-1258`), `ReadQuat`, `ReadIntArray`, `ReadDoubleArray`,
  `ReadStringArray`.
- `ConsoleEvent` for `NetworkProcess`: `readonly int Player`, `readonly String Name`,
  `readonly int Args[3]`, `readonly bool IsManual` (`events.zs:121-131`).

**Why `AddMapUnit`/`AddAngle` and not `AddVector3`.** `AddVector3` is 24 bytes of IEEE
doubles (`events.cpp:1253-1258`); `AddMapUnit` is 4 bytes of 16.16 fixed point, resolution
1/65536 map unit over ±32768 — finer than any Doom map needs and **exactly** representable on
every platform. `AddAngle` is 4 bytes of BAM. A full pose packet is 3×4 + 3×4 + 2 + 1 = **27
bytes** plus the command name string and a 2-byte length (`events.cpp:442-444`).

**Budget, and the real hazard.** `Net_WriteInt8/16/32/Float/Double/String` append to the
`TicSpecial specials` buffer (`d_net.cpp:2090-2130`), which **grows without limit**
(`GetMoreSpace`, `:233-245`). The danger is at pack time: the per-tic specials stream is
`memcpy`'d into `netbuffer` (`d_net.cpp:1281-1282`) with **no size check**, and `netbuffer`
is `doomcom.data`, a fixed `uint8_t data[MAX_MSGLEN]` with `MAX_MSGLEN = 14000`
(`src/common/engine/i_net.h:23`, `:84`). A packet can carry several tics (`numtics`), so the
real per-tic budget is well under 14000/numtics. An oversized sustained stream is memory
corruption, not a clean error. **Keep the per-tic payload in the low hundreds of bytes.** The
demo buffer, by contrast, reallocs safely (`g_game.cpp:2742-2755`).

`Net_SkipCommand` handles `DEM_ZSC_CMD` by reading the big-endian length
(`d_net.cpp:2854-2857`; `WriteInt16` is big-endian, `src/d_protocol.cpp:128-133`), so demo
skipping over your commands is correct.

Both senders early-return unless `gamestate == GS_LEVEL || GS_TITLELEVEL`
(`events.cpp:383`, `:399`, `:497`) — a send from a loading screen is silently dropped.

#### 7.3 RNG discipline

The consistency sum includes `StaticSumSeeds()` (`g_game.cpp:1367`), so **any playsim RNG
consumed on only one machine desyncs.** Melee-relevant consumers:

- `random[Punch]`, `Random2[Punch]` (`weaponfist.zs:69`, `:74`)
- `random[Saw]`, `Random2[Saw]`, `Random[Saw]` (`weaponchainsaw.zs:64`, `:72`, `:73`, `:92`, `:102`)
- `random[cwpunch]`, `random2[cwpunch]` (`stateprovider.zs:333`, `:346`)
- `Random2[crailgun]` (`stateprovider.zs:442-443`)
- `pr_damagemobj` in `TriggerPainChance` (`p_interaction.cpp:991`) — **skipped** with
  `forcedPain = true`
- `pr_lightning`, `pr_poison` in `TriggerPainChance` for `'Electric'` / `'PoisonCloud'`
  (`:995`, `:1025`)
- `random[Kickback]`, `frandom[Kickback]` in `ApplyKickback` (`interaction.zs:31`, `:51`)
- `random[LineAttack]` in `SpawnLineAttackBlood` (`attacks.zs:892`)
- `pr_checksight` / `pr_botchecksight` in `P_CheckSight` (`p_sight.cpp:879`) — skipped with
  `SF_IGNOREVISIBILITY`
- `pr_quake` in `DoQuakeDamage` (`a_quake.cpp:193`, `:196`, `:208`) — skipped with
  `damrad: 0`

Rule: **everything after the `SendNetworkBuffer` must be identical on every machine, and
nothing before it may touch playsim state or playsim RNG.** Never branch play-scope code on
`consoleplayer`, on the hand fields, on `PlayInVR`, on `OverrideAttackPosDir`, or on a local
cvar. Key gameplay off `cmd.Player` / the actor's owner.

---

### 8. The hard limits, with the citation that proves each

1. **No hand velocity, angular velocity, analog trigger, grip, finger or touch data.**
   The controller globals are position and Euler angles only
   (`src/QzDoom/VrCommon.h:37-41`, defined `src/QzDoom/qzdoom_common.cpp:10-13`). The only
   delta the engine computes is for the headset and is not exported
   (`qzdoom_common.cpp:17`, `:97-99`). Joystick axes reach ZScript only as digital
   `Key_JoyAxisNPlus/Minus` key events (`wadsrc/static/zscript/engine/inputevents.zs:193+`)
   through `ui`-scope `InputProcess`.
2. **No direct haptic call from script.** `VRMode::Vibrate` is C++
   (`hw_vrmodes.h:94`); `VR_HapticEvent`'s body in this tree is empty
   (`qzdoom_common.cpp:65-67`); `QzDoom_Vibrate` is built from outside the repo
   (`mobile/Android_src.mk:558-565`). `A_QuakeEx` is the only per-hand, net-safe substitute
   (§6.1).
3. **No per-bone hand or model posing.** Stock `actor.zs` has
   `SetAnimation`/`SetAnimationFrameRate`/`SetModelFlag`/`ResetModelFlags` only
   (`actor.zs:1340-1348`). (`RegisterModelStateFrame`, `ClearModelStateFrames`,
   `SetModelSurfaceOffset`, `ClearModelSurfaceOffset`, `CountStateLabels`,
   `GetStateLabelAt` are **HEAD ONLY — unavailable** on stock.)
4. **A remote player's hand pose cannot be read.** Both writers key on
   `r_viewpoint.camera->player` (`hw_vrmodes.cpp:434`, `hw_openxrdevice.cpp:551`). Your mod
   must send it, and the peer must be running your mod.
5. **`weaponStabilised` cannot be set or cleared from script, and its consequences cannot be
   suppressed.** It is a C++ global (`qzdoom_common.cpp:19`) with no setter. While it is set:
   the off-hand psprite stops drawing in both the sprite and model loops
   (`src/rendering/hwrenderer/scene/hw_weapon.cpp:832`, `:917` — reading the global
   directly, so script cannot influence it); `WF_TWOHANDSTABILIZED` is forced on at every
   psprite state change (`p_pspr.cpp:512-515`); the off hand cannot become ready
   (`weapons.zs:415-418`), cannot fire (`player.zs:393-396`, `:426-429`), and its extra
   buttons are ignored (`p_pspr.cpp:755-758`). It does **not** touch the off-hand pose —
   `OffhandPos`/`OffhandAngle`/`OffhandPitch`/`OffhandRoll` are written unconditionally
   (`hw_openxrdevice.cpp:600-609`) — and it does not null `player.OffhandWeapon`.
6. **`switchhand 1` can always evict the off-hand slot**, whatever flags your weapon carries.
   `CCMD(switchhand)` (`g_game.cpp:335-350`) calls `SwitchWeaponHand(hand)` where `hand` is
   the **destination**, so `switchhand 1` moves the **main-hand** weapon across and tests
   `bNoHandSwitch` on **that** weapon (`player.zs:2490-2491`), then executes
   `player.OffhandWeapon = player.ReadyWeapon = null;` (`:2494`).
   `+WEAPON.NOHANDSWITCH` on your off-hand item blocks only `switchhand 0`.
   (`WIF_NOHANDSWITCH = 0x00200000`, `src/gamedata/a_weapons.h:163`; flagdef
   `weapons.zs:99`.)
7. **A `+WEAPON.TWOHANDED` weapon cannot coexist with anything in the other hand.**
   `BringUpWeapon` nulls the other slot and clears its psprite
   (`player.zs:1977-1994`).
8. **No contact force, no collision on the hand.** Hitting a wall and hitting air produce
   identical data: nothing in the pose pipeline consults the world
   (`hw_openxrdevice.cpp:587-609` is pure arithmetic on the controller offsets). You can
   trace for the wall and fake the stop, but the physical hand goes through it.
9. **`ucmd.roll` is never written**, so HMD/controller roll can never reach the simulation
   through the tic command (`d_protocol.h:69` declared; no assignment in `g_game.cpp`;
   `hw_openxrdevice.cpp:740` writes `vp.HWAngles.Roll` only).
10. **The hand pose is one render frame old by the time any play hook sees it, and the first
    play hook is one tic later.** `vrmode->SetUp()` runs in `D_Display`
    (`hw_entrypoint.cpp:143`), which runs after `TryRunTics()` (`d_main.cpp:1314`, `:1318`).
    Floor: ~29 ms detect-to-damage locally at `TICRATE = 35` (`doomdef.h:61`). A swing
    cannot parry a fast projectile; a continuously-maintained guard volume is a different
    mechanism and does not have this floor.
11. **The offset parameters of `LineAttack`/`LineTrace` are unusable in VR.** Uninitialised
    origin (`p_map.cpp:4872` + `:4890`; `:5181` + `:5221`), and half-disabled by a local
    cvar anyway (`hw_vrmodes.cpp:123`, applied `p_map.cpp:4896-4897`).
12. **`AimLineAttack` cannot vertically autoaim a VR attack.** It returns the pawn's pitch
    whenever `OverrideAttackPosDir` is set (`p_map.cpp:4646-4650`).
13. **`BlockThingsIterator` cannot filter by Z and is a square, not a circle**
    (`p_maputl.cpp:1083-1084`, `:1142`). `ignorerestricted` is dead.
14. **`CheckSight` cannot look from the hand.** Eye height is hard-coded to
    `Z + 0.75·Height` (`p_sight.cpp:910`).
15. **`ApplyKickback` and `SpawnLineAttackBlood` ignore the off-hand weapon**
    (`interaction.zs:12-15`, `attacks.zs:874-876`). An off-hand melee weapon's `Kickback`
    and `+WEAPON.AXEBLOOD` have no effect unless you override the virtuals.
16. **`vr_enable_haptics` has no reader in this tree** (`hw_vrmodes.cpp:125`), so a user who
    disables haptics is invisible to script.
17. **`DEarthquake` occupies `CHAN_BODY` on its spot actor** for its whole duration
    (`a_quake.cpp:125-128`).

---

### 9. Summary table — one line per primitive

| Primitive | Hand-aware? | Mutates world | RNG | Use it for |
|---|---|---|---|---|
| `LineTrace` + `TRF_USEWEAPON\|TRF_ISOFFHAND` | yes | no | no | hand-origin geometry queries; clipping a swing at a wall |
| `BlockThingsIterator.CreateFromPos` | n/a | no | no | **candidate gathering for a swept capsule** — the melee primitive |
| `BlockLinesIterator.CreateFromPos` | n/a | no | no | wall clipping without a trace |
| `LineAttack` + `LAF_ABSPOSITION\|LAF_ISOFFHAND\|LAF_ISMELEEATTACK` | yes | yes | via damage chain | thrusts and stabs, for free puffs/blood/obituaries/haptics |
| `AimLineAttack` + `ALF_ISOFFHAND` | yes | fills `linetarget` | no | finding a target; **not** for pitch in VR |
| `DamageMobj` + `DMG_USEANGLE` | n/a | yes | pain + kickback | arc hits, where you own the geometry |
| `TriggerPainChance(mod, true)` | n/a | yes | **no** | deterministic flinch |
| `Thrust` / `ApplyKickback` override | n/a | yes | default body rolls twice | knockback you control |
| `A_SpawnItemEx(SXF_RELATIVETOWEAPON)` | yes | yes | `random[spawnitemex]` if `failchance` | throwing along the hand axis |
| `A_QuakeEx(..., damrad: 0)` | per hand via X/Y | no (with damrad 0) | no | **net-safe per-hand haptics** |
| `S_StartSoundAt(pos, ...)` | n/a | no | no | positional impact/whoosh at the hand |
| `A_SpawnParticle[Ex]` / `VisualThinker` | n/a | no / thinker | no | swing trails |
| `SendNetworkBuffer` + `NetworkCommandProcess` | n/a | n/a | no | the only deterministic transport for a locally-made decision |
| `WorldHitscanPreFired` → true | n/a | **cancels** | no | parry / shield veto of someone else's hitscan |
| `players[i].cmd.buttons & BT_OFFHAND*` | n/a | no | no | a replicated trigger with no weapon slot involved |

---

### 10. What I could not establish

- **The external Quest input layer is not in this checkout.**
  `mobile/Android_src.mk:558-565` builds `../../QzDoom/TBXR_Common.cpp`,
  `QzDoom_OpenXR.cpp`, `OpenXrInput.cpp`, `VrInputCommon.cpp`, `VrInputDefault.cpp` from
  outside the repo. So: the exact formulas filling `weaponangles`/`weaponoffset`/
  `offhandangles`/`offhandoffset`, the Quest controller key codes, the Quest two-hand-grab
  trigger, `QzDoom_Vibrate`'s duration/intensity/overlap and queueing semantics, and
  whether `vr_enable_haptics` gates it, are **not verifiable here**.
  `src/QzDoom/` in this tree holds only the desktop fallback
  (`qzdoom_common.cpp`, 123 lines, `VR_HapticEvent` an empty stub at `:65-67`).
- **Therefore the absolute sign convention of `offhandoffset[0]`/`[2]` → `OffhandPos.X/.Y`
  cannot be derived here.** `src/rendering/gl/stereo3d/gl_openvr.cpp:1733-1771` shows the
  *PC OpenVR* construction (controller minus HMD in tracking metres, the XZ pair rotated by
  `-(getViewpointYaw() − hmdorientation[YAW])`, then **swapped**: `offhandoffset[0] = v.Y`,
  `offhandoffset[2] = v.X`), and `hw_openxrdevice.cpp:606-607` then **negates** both into
  Doom XY. But that file is `#ifdef USE_OPENVR` (`gl_openvr.cpp:30`) and is not compiled in
  the Quest build. The structure is certainly the same; the handedness is not something I
  would bet a hit test on. **Work in the pawn-frame projections of §1.9, which are
  convention-free, or measure in the headset.**
- **The grab trigger on Quest.** `gl_openvr.cpp:1717-1728` tests only the off-hand grip's
  *state change* and only at the press tests `distance < 0.50f` metres, behind
  `vr_two_handed_weapons` (default true, `hw_vrmodes.cpp:119`) — which means turning that
  cvar off while stabilised leaves `weaponStabilised` latched, since the release branch is
  behind the same test. The engine-side **consumption** is identical for both backends
  (both write the same global), but the Quest trigger itself is inferred, not verified.
- **Whether `e.RailParams.<field> = x` compiles** through a `readonly` struct field
  (`events.zs:109`). If it does, it mutates the live `FRailParams`
  (`events.cpp:722-724`). Untested.
- **Whether `SwitchWeaponHand` actually aborts** when `PickNextWeapon` returns null
  (`player.zs:2498`). The null check is absent; I did not reproduce the abort.
- **`A_SpawnItemEx`'s pitch handling** with `SXF_RELATIVETOWEAPON`: the yaw argument is
  `self.Angle + angle` (`attacks.zs:465`) but the pitch passed to `OffhandDir` is the bare
  `self.Pitch` (`:476`) with no caller deviation, because the function has no pitch
  parameter (`:412`). Whether that is intentional, I do not know.
- **`r_viewpoint.CenterEyePos` vs `RenderEvent.ViewPos` under a visual portal.**
  `src/rendering/hwrenderer/scene/hw_portal.cpp:571` does
  `vp.CenterEyePos += vp.Pos - StartPos` on the portal's own viewpoint copy; I traced the
  per-eye shift to the `HWDrawInfo` copy (`hw_entrypoint.cpp:162`, `:183`) and am confident
  `r_viewpoint.Pos` is unshifted at HUD time, but I did not exhaustively check every portal
  path for a write-back.
- **No headset, no netgame, nothing was run.** Everything above is source reading. In
  particular, every latency number is arithmetic from `TICRATE = 35` and the frame ordering,
  not a measurement.

---

### 11. Corrections to `QZD173_MELEE_STUDY_2026-10-23.md`

1. **Citations.** That study's line numbers for `actor.h` (`1665-1682`),
   `actor.zs` (`282-291`, `872-873`), `g_game.cpp` (`347-361`, `787-789`, `1393`, `1439`,
   `1363-1424`), `p_pspr.cpp` (`595-607`, `847-850`), `d_player.h` (`405`, `523`),
   `hw_openxrdevice.cpp` (`585-609`, `550`, `567`, `153`, `715`, `731`),
   `hw_vrmodes.cpp` (`432-447`, `434`, `436`, `439-440`, `445`, `94`, `127`),
   `p_map.cpp` (`4599-4612`, `4784-4800`, `5185-5201`, `4871-4910`, `5181-5241`, `6184`),
   `hw_entrypoint.cpp` (`143`), `r_utility.cpp` (`1084-1092`, `116`), `d_main.cpp`
   (`1314-1319`), `a_quake.cpp`, `shared_sbar.cpp` (`1234`) are **HEAD line numbers**, not
   stock. The stock equivalents are given throughout this document.
   Material differences: pose block `actor.h:1652-1668`; ZScript exports
   `actor.zs:268-277` and `858-859`; pose written at
   `hw_openxrdevice.cpp:587-595`/`600-609` under the guard at `:568`; `VRMode::SetUp`
   `hw_vrmodes.cpp:432-446`; `RunNetSpecs` `g_game.cpp:1381`; `UiTick` `:1429`;
   `PostUiTick` `:1470`; `switchhand` `g_game.cpp:335-350`;
   `BT_OFFHAND*` filled `g_game.cpp:775-777`; consistency check `g_game.cpp:1407-1424`;
   two-hand bit `p_pspr.cpp:501-515`; off-hand button gate `p_pspr.cpp:755-758`;
   `a_quake.cpp` lives at `src/playsim/mapthinkers/a_quake.cpp`.
2. **`AttackDir(actor, angle, pitch)` does not return a unit direction vector.** It returns
   `(yawDegrees, pitchDegrees, 0)` (`p_actionfunctions.cpp:5700-5701`; stock usage
   `attacks.zs:489-490`). The study's table says "unit direction in the main hand's frame" —
   that is the C++ function, not the ZScript one. §1.6.
3. **The `angle`/`pitch` arguments are deviations from the pawn's own angles**
   (`hw_vrmodes.cpp:401-402`), not absolute. The study does not mention this at all, and it
   is why `A_Punch`/`A_Saw`/`A_CustomPunch` work in VR unmodified. §1.6.
4. **The physical left hand's basis is mirrored** unless that hand's weapon has
   `+WEAPON.NOAUTOREVERSE` (`hw_vrmodes.cpp:464-469`, `weapons.zs:101`), which negates the
   yaw deviation. Not in the study. §1.6.
5. **The pose does not "freeze" when a menu is open — it collapses to the body.**
   `super::SetUp()` runs unconditionally (`hw_openxrdevice.cpp:523`) and the hand block is
   gated at `:568`. Also: `paused` and the console are **not** in that guard, so the pose
   keeps updating while paused, contrary to the study's §1 point 2. §1.4.
6. **`RAF_ISOFFHAND` is not a ZScript constant.** The exported name is `RGF_ISOFFHAND = 64`
   (`constants.zs:252`); no `RAF_*` constant exists in `wadsrc/` at all. `A_RailAttack`
   sets it itself (`stateprovider.zs:428-432`). §3.7.
7. **`player.cmd` is at `player.zs:3134`**, not `:3268`; `struct UserCmd` is at
   `player.zs:2-11` (the study cited both correctly-ish but attributed the field to `:3268`).
8. **The pixelstretch story is more precise than "the hand vector is not metric".** The whole
   map is in that anisotropic space (`r_utility.cpp:1222-1229`,
   `hw_openxrdevice.cpp:281-282`), so raw map arithmetic is *self-consistent*; it is only
   *metric* thresholds that need the 1.2. The conversion and the ellipsoid form are in §1.5.
   And the hand **direction** is squashed by the same factor
   (`hw_openxrdevice.cpp:448`, `:464`), which the study does not mention.
9. **The Z reference depends on `vr_crouch_use_button`.** The study suggests subtracting
   `(pawn.x, pawn.y, player.viewz)`. In the default configuration
   (`vr_crouch_use_button = true`) `OffhandPos.Z − pawn.pos.Z` is the hand's height above
   the **room floor**; with the cvar false it is the height above the **eye**. Two different
   frames. §1.5.
10. **`RenderEvent` carries `FracTic` and `ViewPos`** (`events.zs:61-66`;
    `events.cpp:2075-2086`), and `ViewPos` is exactly the `CenterEyePos` the hand Z was built
    from. The study's §7.1 proposes an 8-sample ring with "frameTime" but never names the
    timestamp the engine supplies or the interpolated head position. §1.9.
11. **`SendNetworkBuffer` / `NetworkBuffer` / `AddMapUnit` / `AddAngle` / `ReadMapUnit` /
    `ReadAngle` exist** (`events.zs:37-57`, `:226-227`, `:26-27`;
    `events.cpp:495-539`, `:1218-1231`) and make the study's §5.2 payload estimate
    (~61 bytes of doubles and floats) obsolete: a full pose packet is **27 bytes** of exact
    fixed point, with no float-determinism question. §7.2.
12. **The net-buffer limit is not "a fatal error".** `TicSpecial` grows unbounded
    (`d_net.cpp:233-245`); the pack loop `memcpy`s into the fixed 14000-byte `netbuffer`
    with no size check (`d_net.cpp:1281`, `i_net.h:23`, `:84`). It is a buffer overrun, not
    a clean failure. §7.2.
13. **`DEFMELEERANGE`/`MeleeRange` metre figures.** The study divides by 34 throughout
    (64 → 1.9 m, 44 → 1.3 m). Horizontally that is right; the study then applies the same 34
    to vertical reasoning elsewhere. Vertical is 28.333. §1.5.
14. **`WorldHitscanPreFired` runs before the free haptic pulse**, so a veto also suppresses
    the rumble (`p_map.cpp:4718` vs `:4764-4776`). The study says the haptic fires "before
    the trace even runs", which is true but leaves the veto ordering unstated.
15. **More haptic routes exist than the study's two.** Nine `Vibrate` call sites, two of
    them script-reachable on purpose (`A_QuakeEx`, `player.bonuscount`), plus the
    `VR_HapticEvent` suit channel with its `ext_haptic_level_*` cvar lookup
    (`c_cvars.cpp:1442-1453`). §6.1.
16. **`A_QuakeEx` haptics come only from `Intensity.X/.Y`**, so `QF_RELATIVE` silently
    disables them (`a_quake.cpp:377-388` vs `r_utility.cpp:1086-1087`); the quake also
    claims `CHAN_BODY` on its spot (`a_quake.cpp:125-128`); and `QF_SHAKEONLY` does **not**
    suppress the thrust (`:206-211`). The study's §4.4 and §7.4 miss all three.
17. **`A_Punch`'s VR turn test has a precedence quirk**:
    `(player != null && !player.PlayInVR || vanilla_melee_attack)` (`weaponfist.zs:81`)
    parses as `((player != null && !player.PlayInVR) || vanilla_melee_attack)`, unlike
    `A_CustomPunch`'s correct `(!player.PlayInVR || vanilla_melee_attack)`
    (`stateprovider.zs:402`). Not harmful, but not what the study's citation implies.
18. **`BlockThingsIterator` semantics.** The study offers it as the better melee primitive —
    correct — but does not say that it applies no Z filter, uses a square box, has a
    128-unit cell granularity, puts the **radius** in `position.Z`, reports a
    portal-translated query point rather than the actor's position, and ignores
    `ignorerestricted` entirely (`p_maputl.cpp:1075-1086`, `:1100`, `:1142`;
    `p_blockmap.h:46`). §3.4.
19. **`CheckSight` and `TriggerPainChance` RNG.** Neither is discussed. `CheckSight` rolls
    `pr_checksight` on invisible targets (`p_sight.cpp:879`) and `TriggerPainChance` rolls
    `pr_damagemobj` unless `forcedPain` (`p_interaction.cpp:991`). Both matter under the
    project's "never put playsim RNG on a local-only path" rule. §3.6, §3.10, §7.3.
20. **`ApplyKickback`/`SpawnLineAttackBlood` read `ReadyWeapon` only**
    (`interaction.zs:12-15`, `attacks.zs:874-876`) — an off-hand melee weapon's `Kickback`
    and `+WEAPON.AXEBLOOD` are dead. Not in the study. §3.9, §5.3.
21. **`LAF_ABSPOSITION` still traces from `t1->Sector`** (`p_map.cpp:4919`) while
    `TRF_ABSPOSITION` resolves the sector properly (`:5269`). The study recommends
    `LAF_ABSPOSITION` without this caveat. §3.2.
22. **`AimLineAttack`'s body/hand inconsistency** (`p_map.cpp:4621-4623`: hand origin, body
    `shootz` for the pitch limits) and the fact that it returns the pawn's pitch in VR
    (`:4646-4650`) are not in the study. §3.3.
23. **`Actor.Prev` is the swept-target primitive.** `ClearInterpolation()` sets
    `Prev = Pos()` for every actor at the top of `P_Ticker`
    (`actorinlines.h:29-35`, `p_tick.cpp:158-162`), so `pos - Prev` is the target's travel
    across a tic — exactly what the study's §7.3 capsule test needs and never names. §4.1.
24. Confirmed from the study without change: the uninitialised `tempos`/`startpos` bug; the
    `-90°` yaw and negated pitch; `use_action_spawn_yzoffset` defaulting false;
    `WF_TWOHANDSTABILIZED`'s five effects and what the grab does *not* do;
    `switchhand`'s destination-hand asymmetry and the `+WEAPON.NOHANDSWITCH` half-fix;
    the free 150 ms @ 0.8 `LineAttack` haptic; `r_quakeintensity` defaulting to 0;
    `ucmd.roll` never being written; stock VR aiming already being non-deterministic; and
    the `A_QuakeEx`-is-net-safe conclusion.


---

# Part Two — Three systems compared

## Three VR melee systems on stock QuestZDoom 17.3, compared technique by technique

*The practice half. Written for someone about to write or extend a melee system on this engine.*

---

### 0. What this is, and how to check it

Three shipping systems, all pure ZScript, all on **stock QuestZDoom 17.3**, none of which may
change the engine:

| | What it is | Where the code is | Size |
|---|---|---|---|
| **SABER** | `UltraBadassQuestSaber` — a lightsaber. Swing cutting, deflection, parry, lock-on painting, a throwable blade, blade locks, a world hilt. | `E:\DOOMWork\UltraBadassQuestSaber\zscript.txt`, one file | ~4,800 lines |
| **SHIELD** | `RS_ShieldSawQ` — a shield that is also a throwable saw. Guard plate, proactive missile sweep, hitscan stop at the muzzle, bash, grind. | `E:\DOOMWork\RS_ShieldSawQ\zscript\ssq\*.zs` (`pose`, `state`, `guard`, `shield`, `disc`, `props`) | ~2,570 lines |
| **BD22** | The owner's Brutal Doom 22 VR weapons. Axe and sword cuts, an axe throw, punches, a wall scrape. | `BD22_VR-Reloaded_1.92.pk3` → `zscript/rsvr/melee.zs` (758), `zscript/rsq/fists.zs` (137), `zscript/rsvr/vrhand.zs` (92) | ~990 lines of melee |

SABER was read at commit `828c070` with a clean tree, and **another session is editing it as you
read this**, so every citation below is quoted rather than line-numbered. Anything about its
*sound names* in particular may already have moved — the last three commits in its log are all
sound-set changes.

**The engine constraint frames everything.** Stock QZD 17.3 is `670d19eba6^` in
`E:\DOOMWork\QZD17-engine`; that checkout's HEAD carries additions the shipped engine does not
have. Everything asserted about the engine below was read with
`git show 670d19eba6^:<path>`, and the paths are given so you can repeat it. Where a mod's own
comment makes an engine claim I could not confirm, I say so.

Force powers are out of scope entirely, by instruction.

Units used throughout: QZD runs at **TICRATE 35**, and `vr_vunits_per_meter` defaults to **34**
map units per metre. So **1 map unit/tic ≈ 1.03 m/s**, and that conversion is what lets the three
mods' differently-expressed thresholds be compared at all.

---

### 1. Detecting a swing

This is the section that matters. All three mods shipped a broken swing detector, all three got
the same bug report from the same owner in different words, and the three fixes are not equally
good.

#### 1.1 What each one measures, and in what frame

**SABER** measures the **blade tip's travel in world space, minus the body's own step**, over one
tic:

```
Vector3 dTip = ((tip - lastTip) - bodyStep);
Vector3 hStep = h0 - h1;	// ZScript will not dereference .z off a parenthesised expression
dTip.z += hStep.z * (level.pixelstretch - 1.0);
double speed = lastOk ? dTip.Length() : 0.0;
```

`tip` is `BladeBase() + BladeDir() * BladeLen()` — derived from `AttackPos` and the three
controller angles, so it captures both the hand's translation *and* the blade's rotation about the
wrist. `bodyStep` is deliberately not the pawn's position step but the **eye's**:

```
static Vector3 BodyAt(Actor o) { return (o.pos.x, o.pos.y, o.player ? o.player.viewz : o.pos.z + 41.0); }
...
Vector3 bn = BodyAt(owner);
bodyStep = bn - bodyNow;
if (bodyStep.Length() > 48.0) bodyStep = (0, 0, 0);
```

with the reason given in the comment: *"QuestZDoom puts the hands relative to the EYE, and the eye
bobs as you walk — so the body's own motion is its feet's xy and its eye's height, as they
actually moved this tic (the owner, 10-01: 'walking around causes the saber to register swings')"*.
That is verifiable in the engine: `hw_openxrdevice.cpp` writes

```
player->mo->AttackPos.Z = r_viewpoint.CenterEyePos.Z + (((hmdPosition[1] + weaponoffset[1]
        + vr_height_adjust) * vr_vunits_per_meter) / pixelstretch) - getDoomPlayerHeightWithoutCrouch(player);
```

so yes — the hand's Z is hung off `CenterEyePos.Z`, and view bob rides in with it.

**BD22** measures the **blade's outer point in the body's own frame**:

```
// 1.11 (owner: "goes insane with attacks just when I move around"). The blade is measured
// in the BODY's own frame (from the pawn, turned by the pawn's angle), so walking,
// strafing and turning move nothing; only the arm swinging the blade counts. (World
// motion minus the pawn's motion was fooled by turns and by the controller and the pawn
// landing a tic apart.)
Vector3 rel = BodyFrame(mo, bout);
Vector3 vel = rel - prevRel;
prevRel = rel;
double speed = vel.Length();
```

```
// a world point in the body's own frame: forward, left, up (map units)
static Vector3 BodyFrame(PlayerPawn mo, Vector3 p)
{
	Vector3 d = p - mo.pos;
	double c = cos(mo.angle), s = sin(mo.angle);
	return (d.x * c + d.y * s, -d.x * s + d.y * c, d.z);
}
```

That parenthetical is the whole lesson: *world motion minus the pawn's motion* — SABER's method —
is defeated by a turn (the arm swings round the body, and no translation subtraction removes it)
and by the controller and the pawn landing a tic apart.

**SHIELD** does the same thing as BD22 and then does one more thing SABER and BD22 both skip:

```
Vector3 Rel(int k)
{
	Vector3 d = pos[k] - bod[k];
	double c = cos(byaw[k]), s = sin(byaw[k]);
	return (d.x * c + d.y * s, d.y * c - d.x * s, d.z);
}

Vector3 Turn(Vector3 v)
{
	double c = cos(byaw[0]), s = sin(byaw[0]);
	return (v.x * c - v.y * s, v.x * s + v.y * c, v.z);
}

// map units per tic, from the last two samples (zero until there are two)
Vector3 Vel()
{
	if (count < 2 || stamp[0] == stamp[1]) return (0, 0, 0);
	return Turn(Rel(0) - Rel(1)) / (stamp[0] - stamp[1]);
}
```

Three things in nine lines. It de-rotates **per sample** (`byaw[k]`, not a single current yaw), so
the body's own turn genuinely cancels. It then **re-rotates the answer into the frame the player
faces now**, because every caller compares it against a world direction (the plate normal for the
bash and the perfect block, the look vector for the throw). And it **divides by the sample gap in
tics** — which neither of the other two does.

Its comment carries the arithmetic that makes the bug concrete:

```
// Doom run speed is about 583 units/sec: eight times ssq_bash_speed, fourteen times
// ssq_perfect_speed and twenty-nine times ssq_throw_min. Walking with the plate ahead of you
// bashed everything in reach, every block came out perfect, and letting go of the lock button
// while moving threw the shield away. A snap turn did all three in one tic: the hand is 20 units
// off the body axis, so 45 degrees in a tic is 15 units, about 525 units/sec.
```

#### 1.2 The window

| | Window | History depth |
|---|---|---|
| SABER cut | **1 tic**, plus a decaying integral worth ~4 tics | 4 hand samples (`h0..h3`) |
| SABER throw | **best of the last 3 single-tic steps** (`HandVel`) | same 4 |
| BD22 cut | **1 tic** of the swept blade quad | 6 samples (`HIST = 6`, ~0.17 s) |
| BD22 throw | **best of up to 4 single-tic steps** | same 6 |
| BD22 punch | **1 tic**, plus a 6-tic "has the hand travelled `JAB_LEN` forward" test | 6 |
| SHIELD bash | **1 tic** along the face normal, plus a decaying integral worth ~4 tics | 8 samples |
| SHIELD perfect block | **1 tic** (`Vel()`) | 8 |
| SHIELD throw | **best 2-tic window of the last 6 samples** (`ThrowVel`) | 8 |

"Best of the last *n*" for a throw is a shared insight, arrived at independently, and SHIELD states
the reason best:

```
// THE THROW IS THE PEAK, NOT THE RELEASE. When the finger lets go the arm is already
// slowing, so the velocity on that tic reads low and a real throw looks like a drop. The
// fastest 2-tic window in the last six samples is the throw, and its direction is the
// throw's direction.
```

BD22 says the same thing in one line: *"fastest step of the axe's edge over the last few tics:
people slow down as they let go"*.

**The integral is the other shared insight, and it is the more interesting one.** Both SABER and
SHIELD found that a one-tic differential rewards only a fast *arc* and punishes every committed,
slow motion — a thrust, a draw-cut, a lean, a shove into a doorway. Both fixed it by summing as
well as differencing, with the same decay constant and the same shape:

SABER:
```
cutPath = cutPath * 0.75 + speed;
double eff = max(speed, cutPath * clamp(UF("ubqs_cut_sustain", 0.6), 0.0, 1.0));
```
SHIELD:
```
if (mo.Level.maptime != g.lastPushTic + 1) g.pushPath = 0;
g.lastPushTic = mo.Level.maptime;
g.pushPath = g.pushPath * 0.75 + max(shove, 0.0);
double eff = max(shove, g.pushPath * clamp(SSQ_Handler.SV('ssq_bash_sustain', 0.6), 0.0, 1.0));
```

SHIELD's version is the better of the two in one respect: `lastPushTic + 1` makes the sum
**self-healing** — any gap (stowed, thrown, dead, a level change) starts it clean, with no second
site obliged to remember to reset it. SABER resets `cutPath = 0` by hand in the `!Lit()` branch of
`DoEffect`, which works but is a reset that can be forgotten the next time a branch is added.
SHIELD also clamps the integrand at `max(shove, 0.0)` so pulling the plate *back* does not bank
credit toward a forward bash; SABER's `speed` is a magnitude so the question does not arise for it,
but it does mean a saber twirled back and forth banks path either way.

Critically, both of them keep the integral **out of** the things that should belong to a real arc:

```
// THE WHOOSH AND THE BLOCK WINDOW STILL BELONG TO A REAL ARC, and only the CUT uses the
// integral: lastFastTic buys a third of a second of hitscan immunity, and a slow lean must not
// be able to buy that.
if (eff >= cut)
{
	if (speed >= cut) lastFastTic = level.maptime;
	CutAlong(eff);
	if (speed >= cut && swingQuiet <= 0)
	...
```

#### 1.3 The thresholds, in comparable units

| | Measured point | Threshold as declared | In units/tic | ≈ m/s |
|---|---|---|---|---|
| BD22 axe | axe cutting edge, model `(55,0,16)` | `AXE_SWING = 3.0` ("m/s") × `vu/35` | 2.91 | 3.0 |
| BD22 sword | Dragonslayer point, model `(90,0,159)` | `SWORD_SWING = 3.5` | 3.40 | 3.5 |
| BD22 punch | the hand itself | `PUNCH = 2.5` | 2.43 | 2.5 |
| BD22 throw | axe edge | `THROW_MIN = 2.5` | 2.43 | 2.5 |
| SABER cut | blade tip | `ubqs_swing_cut = 8.0` units/tic | 8.0 | 8.2 |
| SABER parry | blade tip | `ubqs_parry_speed = 6.0` units/tic | 6.0 | 6.2 |
| SABER throw | the hand | `ubqs_throw_min = 1.5` units/tic | 1.5 | 1.5 |
| SHIELD bash | plate centre, projected on the face normal | `ssq_bash_speed = 70.0` units/**sec** | 2.00 | 2.06 |
| SHIELD perfect block | plate centre, on the incoming line | `ssq_perfect_speed = 40.0` units/sec | 1.14 | 1.18 |
| SHIELD throw | the hand | `ssq_throw_min = 20.0` units/sec | 0.57 | 0.59 |

Three different unit conventions in three mods: BD22 writes a constant in metres per second and
converts (`thr = AXE_SWING * vu / 35.`), SABER writes map units per tic straight into the cvar,
SHIELD writes map units per second and multiplies by `TICRATE` at the use site
(`pose.Vel() * TICRATE`). All three are internally consistent; none of them is readable against
another without doing this table. **Pick one and document it on the cvar.** Units per second is the
most honest of the three, because it survives a TICRATE change and a player can be told what it
means.

An estimate worth having, flagged as an inference rather than a measurement: SABER's threshold
looks 2.7× BD22's, but it is measuring a point 2.5× further from the wrist. Following BD22's
`Place()` by hand, its axe edge lands ≈ 0.31 m from the controller
(`f = 0.01*(1.02*55 − 30) = 0.261`, `u = 0.01*(1.02*16) = 0.163`, magnitude ×34 ≈ 10.4 units);
SABER's blade is `FullLen() = 26.0 * Size` ≈ 26 units ≈ 0.76 m at size 1. 8.0/2.91 = 2.75 against a
lever-arm ratio of ~2.5. **Two mods tuned independently by feel converged on roughly the same
angular wrist speed.** I did not resolve `EmitAlong()` or the shipped `Size`, so treat the ratio as
approximate — but it is close enough that a fourth implementation should set its threshold as an
*angular* rate and multiply by the lever arm, rather than asking the player to re-tune a linear
number every time the blade length changes.

#### 1.4 The direction gate: a backswing is not a strike

BD22 is the only one of the three with a real one, and it got there over three owner reports:

```
// THE BACKSWING IS NOT A STRIKE. The test used to be speed alone, so raising the axe to
// strike was itself a hit -- it landed before the player had swung, and read as the axe
// being wildly oversensitive (owner, 2026-10-13). Motion that is mostly upward is the
// player winding up, so it does not cut, and it falls through to the calm branch below
// so the downswing that follows is a fresh swing with a cleared strike list.
double rise = speed > 0 ? vel.z / speed : 0;
// 2026-10-16 (owner): NO hits on the upswing. Only a level or downward blade cuts
// (rsvr_melee_up, default 0.15: about 9 degrees up at most; it was 0.55, about 33).
// 1.11 (owner): "swinging an axe DOWNWARD meaningfully". A blow must now point down by at
// least rsvr_melee_down of its speed (default 0.25, about 15 degrees below level); a
// level or rising blade never cuts.
bool backswing = rise > -clamp(Cv(mo, 'rsvr_melee_down', 0.25), 0.0, 0.9);
```

Three successive tightenings, each with a date and the owner's words: 0.55 → 0.15 → *must* be 0.25
*down*. Note the surviving dead constant: `RISE = 0.15` is still declared with the comment *"of the
blade's speed that may be upward before it is a backswing"*, but nothing reads it any more — the
live number is the `rsvr_melee_down` cvar. The stale constant is a small trap for the next reader.

BD22's punch has the equivalent gate on the forward axis, and three more conditions besides:

```
//   fast         over PUNCH m/s in the body frame;
//   forward      mostly along the body's forward axis;
//   a real jab   the hand has travelled JAB_LEN forward within the last few tics;
//   out in front the hand ends at least FRONT ahead of the body.
// After a blow the hand must come back (RECOCK towards the body) or go still before the next.
```
```
if (speed < thr || v.x / speed < FORWARD || lf < FRONT * vu) return;
double back = lf;
for (int i = 0; i < count; i++)
	back = min(back, fx[i]);
if (lf - back < JAB_LEN * vu) return;
```

The "travelled `JAB_LEN` forward within the window" test is the strongest single idea in the three
mods' swing detectors. It is **a displacement test layered on a velocity test**, and it is immune
to exactly the thing a velocity test cannot see: a twitch, a tracker glitch, or one noisy sample.
SABER and SHIELD both have only velocity (plus SABER's integral, which is the nearest thing but is
a magnitude, not a displacement along an axis).

SABER has no direction gate on the cut at all: a blade swept in any direction at `ubqs_swing_cut`
cuts. That is defensible for a lightsaber — a saber has no edge and no backswing — but it means
the raise is a cut, and it is the only one of the three for which that is true. SHIELD's bash gate
*is* a direction gate by construction (`shove = v dot n`, the component along the plate's own face
normal), which is the most elegant of the three: no extra test, the geometry supplies it.

#### 1.5 What fools each one

| | Walking | Strafing | Smooth turn | Snap turn | Teleport | Frame hitch |
|---|---|---|---|---|---|---|
| **SABER** | handled (`bodyStep` off the eye) | handled | **NOT handled below 8 deg/tic**; above it, swings are *disabled* | `snapQuiet = 2`, history flushed | `(owner.pos − lastBodyPos).Length() > 48` → same flush | `lastOk` guards, but **no dt**: the tic after a dropped tic reads a double step |
| **BD22 cut** | handled (body frame) | handled | handled exactly | `settle = 2` on >12 deg/tic, plus the frame is already exact | `(mo.pos − prevPos).Length() > 64` → `settle = 2` | `havePrev` guards; **no dt** |
| **BD22 throw** | **NOT handled** — see below | **NOT handled** | partly (the `settle` flush) | `settle` flush | `settle` flush | no dt |
| **SHIELD** | handled (body frame, per sample) | handled | handled exactly | handled exactly, **no threshold needed** | handled by construction | **dt-normalised** (`/ (stamp[0] − stamp[1])`) |

Four of these deserve naming individually.

**SABER's smooth-turn band is a real, live defect, and the file says so.** The guard is:

```
// ubqs_snap_turn_deg: 8 degrees in ONE TIC is 280 deg/s, so a smooth-turn player above that rate
// trips this every tic -- which sets snapQuiet, flattens the hand history and zeroes the
// predictor, killing prediction, swings and throws for the whole turn. A plausible single cause of
// "the glow lags when I turn". The knob is a workaround: the real fix is telling a snap from a
// fast smooth turn, and that belongs on the backlog, not in a slider.
if (turned >= UF("ubqs_snap_turn_deg", 8.0) || jumped)
{
	h1 = h0; h2 = h0; h3 = h0;
	lastOk = false;
	snapQuiet = 2;
}
```

So there are two bands. **Below 8 deg/tic** the turn is not subtracted at all, and a tip 26 units
out gains `26 × 7° in radians = 3.18` units/tic of spurious speed — **40% of the 8-unit cut
threshold, free, on every turn**, with no arm motion whatever. **Above 8 deg/tic** the saber cannot
swing, deflect-lead, throw or predict its own glow for the duration of the turn. Neither band
exists for BD22 or SHIELD, and the fix is not a better threshold: it is BD22's `BodyFrame`, which
removes the knob along with the bug.

**BD22's throw is still on the method its cut was fixed away from.** `Throw()` never calls
`BodyFrame`:

```
Vector3 t0 = (tx[j], ty[j], tz[j]);
Vector3 p0 = (px[j], py[j], pz[j]);
Vector3 v = t1 - t0;
double rel = (v - (p1 - p0)).Length();
if (rel > best)
{
	best = rel;
	vel = v;
}
```

The *magnitude* test (`rel`) is body-corrected by subtraction, so the throw does not trigger from
walking. But the launched velocity is `vel = v` — **the raw world delta, body travel included** —
and `dir = vel.Unit()` is what the axe flies along. Throwing while running forward mostly points
the same way, so it is benign; throwing while strafing or circling skews the heading by the
strafe's share of 16.7 units/tic. `AimAssist` within 8° of a visible monster papers over it when
there is a target. SABER has the same class of defect in a milder form: `h0..h3` are
`HandPos() − bodyNow`, so its throw is translation-corrected but **not de-rotated**, and a turn
below the 8-deg flush skews the heading. SHIELD's `ThrowVel` is the only one that is right, because
`Turn(Rel(0) − Rel(k))` de-rotates per sample and then re-rotates once into the current facing.

**Only SHIELD is dt-correct.** `Vel()` and `VelAt()` divide by `stamp[0] − stamp[k]`. SABER and
BD22 both treat consecutive samples as exactly one tic apart. On a dropped tic — which a Quest
does — the next real sample is a double-size step and reads as a swing twice as hard, or as a swing
where there was none. SHIELD gets this for free because its samples arrive as net events with
arrival stamps and it therefore *had* to handle an uneven gap; SABER and BD22 read fields in-tic
and never had to. The right lesson is the general one: **stamp your samples and divide, even when
you believe the gap is fixed.**

**The vertical is not metric, and all three found it independently.** The engine multiplies the
hand's X and Y by `vr_vunits_per_meter` and divides Z by `pixelstretch` as well
(`hw_openxrdevice.cpp`, verified above). So a vertical motion arrives at 1/1.2 = 0.833 of its real
size, and you multiply by 1.2 (+20%) to recover it. All three fix it, in three different places:

- SABER, on the hand term of the tip only, because the blade-direction part is already metric:
  `dTip.z += hStep.z * (level.pixelstretch - 1.0);` — and separately in `HandVel()`:
  `a.z *= ps; b.z *= ps; c.z *= ps;`
- SHIELD, in the **magnitude only**, leaving the heading alone:
  ```
  double zt = v.z * ps;
  double l = sqrt(v.x * v.x + v.y * v.y + zt * zt);
  ...
  if (bl > 0 && dl > 0) best = best * (bl / dl);
  ```
- BD22, by pulling `ps` back through `Place()` for the whole chain: `w.z /= ps;` placing the model
  point, and `HandDir` builds its vectors with `-sin(ap.y) * ps`.

SABER's commit describes the symptom precisely: *"an overhead chop needed a fifth more real swing
than a flat slash to reach `ubqs_swing_cut`, and a throw with any vertical component under-read the
same way and flew shorter."* SHIELD's choice — correct the magnitude, keep the heading — is subtly
the most defensible, because the heading in Doom space is *already* what the player will see the
object fly along; "correcting" it would bend the throw away from the arm.

#### 1.6 A fourth answer, from the same pack

`BD22/zscript/rsq/flick.zs` is not melee, but it solves the same problem a fourth way and the idea
transfers:

```
// The flick is read from the barrel's elevation, which turning and walking do not change:
//   fast    the barrel drops FLICK_DEG within FLICK_TICS;
//   down    it ends at least DOWN_DEG below level;
//   once    it re-arms only after coming back above REARM_DEG.
```

**For any gesture expressible as an orientation change, there is no body-motion problem at all** —
the controller's pitch is already in a frame the pawn does not move. `rsvr_axe_keys`-style position
work is only needed when the gesture is genuinely a translation. A wrist-flick ignition, a "point
down to holster", a "turn the plate edge-on", a blade-roll parry: all of them are angle problems,
and an angle problem needs no body frame, no snap-turn flush and no teleport test.

#### 1.7 Verdict — swing detection

**Best answer today: SHIELD's `SSQ_Pose`, with BD22's gates bolted on.** `Rel()`/`Turn()`/`Vel()`
is 15 lines, exact against walking, strafing, turning, teleporting and an uneven sample gap, and it
*deletes* a tuning knob (`ubqs_snap_turn_deg`) instead of adding one. BD22's `BodyFrame` is the same
idea with the per-sample yaw and the dt left out. SABER is a tic behind both, and its own comments
say where.

**A fourth implementation should:**

1. Keep a per-player ring of samples carrying, for each: the measured point, the pawn position,
   the pawn yaw, and a tic stamp. 8 samples is plenty.
2. Difference **in the body frame, per sample**, and re-rotate into the current facing exactly once
   at the end. Divide by the stamp gap.
3. Correct the vertical by `pixelstretch` in the **magnitude**, not the heading.
4. Express the threshold as an **angular rate at the wrist** and multiply by the measured lever arm
   of whatever point you are tracking, so a longer blade does not silently need a new number.
5. Gate on **three** things, not one: velocity, direction (project onto the axis the attack is
   *about* — BD22's `v.x / speed < FORWARD`, SHIELD's `v dot n`), and a **displacement** over the
   window (BD22's `JAB_LEN`). The displacement test is what kills noise.
6. Integrate as well as differentiate, with the integral self-healing on a tic gap (SHIELD's
   `lastPushTic + 1`), so a thrust and a lean work — and keep the integral out of anything that
   grants immunity or plays a whoosh.
7. Require a re-arm: BD22's `ready`/`RECOCK`, or SABER's `calmTics > 2`.
8. Needing no snap-turn knob and no teleport test is the sign you got step 2 right.

---

### 2. The geometry of a hit

#### 2.1 How the blade or plate is derived from the hand

All three follow the HUD-model transform chain by hand so that the thing they test sits where the
thing you see is drawn. They do it at three different levels of rigour.

**BD22 re-implements `RenderHUDModel` and places arbitrary model points:**

```
// a point of the held model -> world. Same chain as RenderHUDModel: hand,
// scale .01, translate (0, 5, 30), model Scale -1 1 1 with ZOffset -5,
// the jDoom 90 degree turn, vr_weaponScale; local x right, y up, -z forward.
Vector3 Place(Vector3 m)
{
	double f = 0.01 * (ws * m.x - 30 - oz);
	double r = 0.01 * (-ws * m.y + ox);
	double u = 0.01 * (ws * m.z + oy);
	Vector3 w = fF * f + fR * r + fU * u;
	w.z /= ps;
	return fH + w * vu;
}
```

with the blade's endpoints as measured model coordinates, per weapon:

```
static Vector3 BladeIn(int k)
{
	if (k == AXE)
		return (44, 0, 26);		// top of the axe head
	return (38, 0, 16);			// Dragonslayer: guard
}

static Vector3 BladeOut(int k)
{
	if (k == AXE)
		return (55, 0, 16);		// the axe's cutting edge
	return (90, 0, 159);		// Dragonslayer: point
}
```

This is the most general of the three: `Place()` takes *any* model point, and it reads the four live
cvars (`vr_weaponScale`, `vr_3dweaponOffsetX/Y/Z`) that the player can move, so the hitbox follows
the player's fit.

**SHIELD reduces the chain to three numbers and documents the reduction**, which is the most
checkable:

```
// models_md3.cpp BuildVertexBuffer   render vertex = (mx, mz, my)
// models.cpp RenderHUDModel          M = Hand . S(0.01) . T(0,5,30) . S(xs,zs,ys)
//                                        . T(xo/xs, zo/zs, yo/ys) . R(90,Y)
// hw_vrmodes.cpp MapWeaponDir        F = -Zlocal, L = -Xlocal, U = +Ylocal
//
// which reduces, in the off controller's own frame and in cm, to
//
//   F = mx - 6      L = my (right-handed scheme) / -my (left)      U = mz - 5
```
```
shield_rim     centre F -15.36  L -8.37  U -4.93   radius 28.46 / 29.27
shield_back    centre F -15.40  L -6.60  U -5.00   radius 29.04
...
shield_handle  a bar at F -1.3, L -4.5, spanning U -13.8 .. +4.4: the grip
```

Then the plate is a disc at measured offsets, every one of them a live cvar:

```
return pose.pos[0]
	+ pose.F * (SSQ_Handler.CM(SSQ_Handler.SV('ssq_plate_f', -15.4)) * k)
	+ pose.L * (SSQ_Handler.CM(SSQ_Handler.SV('ssq_plate_l', -7.5)) * s * k)
	+ pose.U * (SSQ_Handler.CM(SSQ_Handler.SV('ssq_plate_u', -5.0)) * k);
```

Two things here are worth stealing outright. The **sign of the lateral axis is derived from the
handedness**, not guessed (`Side(p)` → `-1` for left-handed, matching the mirrored MODELDEF block),
and v0.6's inverted sign is on record as having produced both *"the shield guards the wrong side"*
and *"the plate is not on the geometry"* — one sign error, two apparently unrelated bug reports.
And the **mesh origin is the grip**, which is why the stowed shield hangs correctly with no fudge
offset and why `ssq_plate_yaw`/`ssq_plate_pitch` could be deleted: once the frame is right there is
nothing left to tilt.

**SABER derives the blade from `BeamAlong(mx, mz)`**, a projection of a model point onto the blade
axis, and uses it for the emitter, the pommel and the tip:

```
double FullLen() { return 26.0 * UBQS_Data.Size(SizeIdx()); }
double BladeLen() { return max(0.0, BeamAlong(37.940, 45.216) - BeamAlong(5.111, 6.091)) * max(UBQS_Data.Ext(frameNow), 0.0); }
Vector3 BladeBase() { return UBQS_Data.BladeBaseOf(HandPos(), BladeDir(), SizeIdx()); }
Vector3 Pommel()    { return HandPos() + BladeDir() * BeamAlong(-5.111, -6.091); }
```

with the reason on the record: *"THE BEAM STARTS AT THE EMITTER, NOT IN YOUR HAND (the owner,
10-01: 'your dots still sit in the handle')"*. `UBQS_Data.Ext(frameNow)` scales the length by the
*animation frame*, so an igniting or dousing blade is geometrically short — which is why `Lit()`
tests `Ext(frameNow) > 0.5` and why the ignition-counter bug (below) was felt as *"a 3 unit stub
below `Lit()`'s threshold: no hum, no light, no cut and no block"*.

All three ship a **visual debug of the actual test geometry**, and all three authors evidently used
it: SABER's `ubqs_diag 1` draws dots along the blade line, SHIELD's `ssq_debug_ring` draws the rim,
a green spoke along the face normal, and the hand. Build this on day one.

#### 2.2 Swept, or frozen

| | Offence | Defence |
|---|---|---|
| **BD22** | **swept** — a bilinear patch from last tic's blade segment to this tic's | n/a |
| **SABER** | **frozen** — this tic's blade segment only | **swept** — 3 samples of the blade against the missile's path |
| **SHIELD** | **frozen** — this tic's plate only | **swept** — 3 plate positions along its own travel |

BD22 is the only one that sweeps the *offensive* volume, and it is a genuine swept patch, adaptively
sampled:

```
// the blade's path this tic (a0-b0 -> a1-b1) against every body near it;
// each body is hit once per swing, at the first point the blade reaches it
void Sweep(PlayerPawn mo, int k, Vector3 a0, Vector3 b0, Vector3 a1, Vector3 b1, double power)
{
	double pad = (k == AXE) ? 3 : 5;
	Vector3 c = (a0 + b0 + a1 + b1) / 4;
	double r = max(max((a0 - c).Length(), (b0 - c).Length()), max((a1 - c).Length(), (b1 - c).Length())) + pad;
	double travel = max((b1 - b0).Length(), (a1 - a0).Length());
	int steps = clamp(int(travel / 4) + 1, 1, 16);
	int segs = (k == AXE) ? 2 : 8;
```
```
	for (int i = 1; i <= steps && !hit; i++)
	{
		double f = double(i) / steps;
		for (int j = segs; j >= 0 && !hit; j--)
		{
			double s = double(j) / segs;
			Vector3 p0 = a0 + (b0 - a0) * s;
			Vector3 p1 = a1 + (b1 - a1) * s;
			Vector3 q = p0 + (p1 - p0) * f;
			if (Touches(t, q, pad))
			{
				hit = true;
				at = q;
				dir = p1 - p0;
			}
		}
	}
```

Three details do a lot of work. `steps` is derived from the *actual travel* at 4 units a sample, so
a slow swing costs 1 step and a fast one up to 16 — the cost tracks the risk of tunnelling.
`segs` is 2 for the axe (3 points along a short head) and 8 for the Dragonslayer (9 points along a
long blade) — the sampling density tracks the blade's length. And the inner loop runs **`j` from
`segs` down to 0**, i.e. from the blade's *outer* point inward, so the first contact found is the
one nearest the tip; combined with `i` ascending, the recorded `at` is the earliest-in-time,
furthest-out contact. That is the right contact point for a cut, and it is free.

The bounding sphere is the four corners' max radius plus pad, and the iterator is built from it:

```
let it = BlockThingsIterator.CreateFromPos(c.x, c.y, c.z - r, 2 * r, r, false);
```

SABER's cut, by contrast, is a single closest-point-on-segment test against this tic's blade:

```
Vector3 c = level.Vec3Diff(base, (mo.pos.xy, mo.pos.z + mo.height * 0.5));
Vector3 blade = dir * len;
double t = clamp((c dot blade) / (len * len), 0.0, 1.0);
Vector3 gap = c - blade * t;
if ((gap.xy).Length() > mo.radius + pad || abs(gap.z) > mo.height * 0.5 + pad) continue;
```

That is cheap and reads well, but a blade moving 8+ units a tic (the cut threshold) with a victim
radius of 20 is on the edge of tunnelling, and at 24 units a tic (the mod's own idea of "a hard
swing", where the whoosh saturates) it will pass clean through a 20-radius imp between samples
roughly as often as not. `ubqs_cut_pad` (3 units) is the only mitigation. **SABER already has the
swept machinery** — `Deflect` interpolates `pb → base` and `pt → tip` at `t = 0, 0.5, 1` — it
simply never applied it to the cut. That is a two-line change and it is the single highest-value
geometric improvement available to that mod.

Note the asymmetry is in the *same direction* for both SABER and SHIELD: both sweep where a fast
*projectile* could slip through, neither sweeps where a fast *arm* could. Which is backwards — the
arm is the thing the player is swinging on purpose.

SHIELD's defensive sweep is the one to copy for a moving surface, because it sweeps the **surface**
rather than only the threat:

```
Vector3 PlateTravel()
{
	...
	return pose.pos[1] - pose.pos[0];	// where it came FROM, relative to where it is
}
```
```
// three places the plate was during this tic: where it came from, halfway, here
bool got = false;
for (int s = 0; s <= 2 && !got; s++)
{
	Vector3 Cs = C + travel * (1.0 - s * 0.5);
	if (Crosses(A, B, Cs, n, R + sh.radius)) got = true;
}
```

Its commit message names the bug exactly: *"a shield being swung was checked at a stale position
while the shot was checked at a live one — wrong exactly when you are parrying, which is the moment
you care."* And it is honest about the residual: *"The facing is this tic's: the history keeps
positions, not past axes, and across one tic the translation is the part that matters."* So a plate
that **rotates** fast without translating is still tested at one orientation. For a ~20-unit-diameter
disc that is a real but small error; for a long blade it would not be, which is why SABER's
`Deflect` interpolates both endpoints instead.

#### 2.3 The test volume itself

**SHIELD's thick disc** is the best-argued primitive of the three:

```
// does the segment A->B pass through the THICK disc (centre C, face n, radius R, half
// thickness T)? P is where. A thick disc, not a plane: the plane test misses a shot that
// skims the face edge-on, and it needs the face's sign to be right; the thick disc blocks
// anything whose path comes within the rim and within T of the face.
```

and it reports the contact point **on the face**, not at the closest approach, because the slab is
thick enough that the two differ visibly:

```
// THE REPORTED POINT GOES ON THE FACE. Where the segment actually pierces the
// face plane if it does -- the slab is thick, so the closest point can be 10 cm
// off the face and the spark lands in mid air.
double da = (A - C) dot n, db = (B - C) dot n;
if ((da > 0) != (db > 0))
{
	double den = da - db;
	if (den != 0 && den == den)
	{
		Vector3 x = A + d * (da / den);
		...
```

In map units: `CM(cm) = cm * 34/100`, so the plate is radius 29.2 cm = **9.9 units**, half-thickness
25 cm = **8.5 units** (a 17-unit-thick slab), centred 15.4 cm = 5.2 units back along the forearm,
plus `ssq_sweep_pad` 18 cm = 6.1 units on the sweep and muzzle paths for an effective radius of
~16 units — about one Doom player radius. SABER's equivalent is a 26-unit segment with
`ubqs_blade_guard` 6 units of slop (a strip ~26 × 12) and `ubqs_cut_pad` 3 units on the cut.
Comparable areas, different shapes, both defensible.

Note the NaN discipline in `Crosses`, which is where SHIELD's handoff note about zero-divides lands:

```
if (dd <= 0 || dd != dd) return false;
...
if (along != along) return false;
```

`x != x` as a NaN test, because — per the handoff — `abs(x) < 1e-6` *"did not stop a true zero
divide; use `x == 0 || x != x || (x < 1e-6 && x > -1e-6)`"*. I did not reproduce that on the VM, so
treat the exact form as the mod's finding rather than a verified engine fact; but the NaN guards
themselves are cheap and SABER's log records a NaN poisoning its thrown saber's position
permanently (§5.4), so take them.

#### 2.4 The tic boundary

All three guard the first tic after a discontinuity, and all three had to learn it.

- **BD22**: `if (!havePrev) { prevIn = bin; prevOut = bout; prevPos = mo.pos; prevRel = BodyFrame(mo, bout); havePrev = true; return; }` — it stores and returns without testing. Clean.
- **SABER**: a single `lastOk` flag, set false on a snap, a teleport, an unlit blade, a twirl frame, or the saber leaving the hand. `speed = lastOk ? dTip.Length() : 0.0`.
- **SHIELD**: `pose.Fresh(now)` (`now - stamp[0] <= 3`) plus `count < 2` in `Vel()`.

SABER's twirl case is the instructive one, because it is a discontinuity created by the *mod's own
animation* rather than by the engine:

```
// A FREE CUT OUT OF EVERY TWIRL. A twirl frame's blade points along Up() 90 and
// frame 0's along Up() 50, so the tic the twirl ended measured a ~19 unit tip jump
// out of a 40 degree frame change -- full damage to everything along the blade for
// no swing at all. The swing history is forgotten across the twirl now.
```

**Any model frame that moves the blade relative to the hand is a swing as far as a differencer is
concerned.** SABER's answer is to enumerate the frames that are safe and prove the rest cannot
occur:

```
// IT IS SAFE TO FEED IT THE FRAME'S OWN BLADE DIRECTION: of the 31 md3 frames the states only ever
// ask for 0, 1, 2, 13-18 and the twirl, and UBQS_Data.Yaw() is 0 and Up() is 50 on every one of
// those nine -- so across every frame this history can span, BladeDir() is a pure function of the
// controller pose and there is no frame step for the predictor to mistake for a swing.
```

That is the right analysis and the wrong mechanism — it is a proof in a comment, which the next
state-table edit silently invalidates. The mechanical version is: hash the frame into the sample,
and flush the history whenever it changes in a way that moves the measured point.

#### 2.5 Verdict — hit geometry

**Best answer today: BD22's swept, adaptively-sampled blade patch for offence; SHIELD's
travel-swept thick disc for defence; SHIELD's documented mesh reduction for deriving either from
the hand.**

**A fourth implementation should:**

1. Derive the test shape from the **mesh**, follow the HUD chain once, and write the reduction into
   the file as SHIELD does — `F = mx − 6, L = ±my, U = mz − 5` is checkable by the next reader;
   "I measured it" is not. Read the live fit cvars (`vr_weaponScale`, the three offsets) like BD22
   does, so the hitbox follows the player's fit.
2. Derive the lateral sign from handedness, never hard-code it. One sign error produced two
   unrelated-looking bug reports in SHIELD.
3. **Sweep the offensive volume**, with the step count derived from actual travel (BD22's
   `clamp(int(travel / 4) + 1, 1, 16)`) and the along-blade density derived from blade length.
   Iterate from the tip inward so the first contact found is the right one.
4. Sweep the *surface* as well as the threat on the defensive path; if the surface is long, sweep
   both its endpoints, not just its centre.
5. Prefer a thick slab or capsule to a plane. Report the contact point on the surface, not at the
   closest approach, or your sparks land in mid-air.
6. Guard NaN with `x != x` and guard the zero denominator explicitly.
7. Put the frame index in the sample and flush on a change that moves the measured point; do not
   rely on a proof in a comment that the state table will not change.
8. Ship the debug draw of the actual test shape before you ship the feature.

---

### 3. Deciding what was hit, and what happens to it

#### 3.1 The iterator and the filter

All three use `BlockThingsIterator`, and in all three cases that was a *fix* rather than the
original design — SHIELD's log: *"EVERY ACTOR IN THE LEVEL was being walked, every tic, to find
things within 1200 units of one hand. A blockmap iterator around the hand looks at the few sectors
that can possibly hold a candidate instead — the owner's lightsaber does the same, and on a Quest
the difference is not academic."* BD22's `AimAssist` is the one place a `ThinkerIterator` survives,
and it runs once per throw rather than per tic, so it is defensible.

The filters are near-identical and worth setting side by side, because the differences are the
interesting part:

```
// BD22
if (!t || t == mo || t.player || !t.bSHOOTABLE || t.bFRIENDLY || t.bDORMANT || t.health <= 0) continue;

// SABER
if (!mo || mo == owner || !mo.bShootable || mo.health <= 0) continue;
if (mo.bFriendly && owner.player) continue;
if (mo.player && !deathmatch) continue;

// SHIELD
if (!t || t == mo || !t.bShootable || t.health <= 0) continue;
if (t.player && !deathmatch) continue;
```

BD22 excludes players unconditionally — it has no deathmatch path. SABER and SHIELD both gate on
`!deathmatch`, which is the right shape. BD22 is the only one that checks `bDORMANT`; SABER and
SHIELD will both cut a dormant monster. SHIELD's bash is the only one that omits `bFriendly`
entirely, so a bash shoves a friendly — arguably correct for a shove, but it is not stated as a
decision.

**Read every cvar above the iterator.** All three say so, in the same words, because all three got
it wrong first:

```
// EVERY SETTING IS READ HERE, ABOVE THE ITERATOR. UBQS_Data.F is an uncached CVar.GetCVar by string
// name and this loop runs over every actor in the blade's box, so a read at a use site inside it is
// unbounded.
```

#### 3.2 Line of sight — and this is where they genuinely differ

| | Test | From | Cost |
|---|---|---|---|
| **BD22** | `LineTrace` with `TRF_ABSPOSITION \| TRF_THRUACTORS`, accepting `HitNone` or `HitActor` | **the hand**, to the contact point | one trace per candidate |
| **SABER** | `owner.CheckSight(mo, SF_IGNOREVISIBILITY)` | the player | one sight check per candidate, last test |
| **SHIELD** | `mo.CheckSight(t, SF_IGNOREVISIBILITY)` | the player | same |

```
// BD22: nothing solid between the hand and that point
static bool Clear(Actor mo, Vector3 from, Vector3 to)
{
	Vector3 d = to - from;
	double len = d.Length();
	if (len < 1)
		return true;
	FLineTraceData tr;
	mo.LineTrace(VectorAngle(d.x, d.y), len, -asin(clamp(d.z / len, -1, 1)),
		TRF_ABSPOSITION | TRF_THRUACTORS, from.z, from.x, from.y, tr);
	return tr.HitType == TRACE_HitNone || tr.HitType == TRACE_HitActor;
}
```

BD22's is the correct test for a melee weapon and the other two are approximations. The case that
separates them is exactly the one both SABER and SHIELD's commits describe: *"a swing near a thin
wall or a closed door killed what was behind it."* `CheckSight` runs from the player's eye to the
actor's centre — so a monster the *player* can see but the *blade* cannot reach round a corner is
still cut, and a monster the blade is genuinely inside but which is out of the player's line (above,
below, behind a railing) is not. BD22 asks the question the geometry actually poses: is there
anything solid between **the hand** and **the point the blade touched**.

Both SABER and SHIELD put the sight check **last**, deliberately, and both say why in the same
words: *"Last of the per-victim tests, so it costs a sight check only for something the blade
geometrically touched."* That ordering is right regardless of which test you use.

Note the `TRF_THRUACTORS` and the `TRACE_HitActor` acceptance in BD22: the trace must pass *through*
intervening monsters (a blade does) while being stopped by walls and flats. Getting that pair wrong
gives you a blade that cannot cut the second imp in a queue.

#### 3.3 Per-victim cooldowns

Three different scopes, and the scope is the design decision:

**BD22 — per swing.** A `struck` array, one entry per victim, cleared when the hand goes calm:

```
if (power >= 1 && !backswing)
{
	calmTics = 0;
	swingTics++;
	...
	Sweep(mo, k, prevIn, prevOut, bin, bout, power);
}
else if (++calmTics > 2)
{
	swingTics = 0;
	struck.Clear();
}
```

**SABER — per victim, timed.** Parallel arrays with a tic stamp, and the comment names it as the
real balance lever:

```
// ubqs_cut_rate: how soon the SAME enemy can be cut again -- this, not ubqs_damage, is the
// real cap on the saber's rate of killing, and the reason holding the blade inside a monster does
// not delete it.
int i = cutWho.Find(mo);
if (i < cutWho.Size() && level.maptime - cutWhen[i] < rate) continue;
```
with pruning of the dead on every call:
```
for (int k = cutWho.Size() - 1; k >= 0; k--)
	if (!cutWho[k] || cutWho[k].health <= 0) { cutWho.Delete(k); cutWhen.Delete(k); }
```

**SHIELD — global, timed.** `bashCool = 10` tics for the whole bash, and the grind on a modulo of
map time (`(maptime % ssq_grind_every) == 0`), so all victims share one clock. Its disc has a
single-slot memory: `if (victim == lastHit && Level.maptime - lastHitTic < 6) return 1;`.

Per-swing is the most honest — a swing is one event and should land once — but it needs a reliable
"the swing ended" signal, and BD22 only has one because it has a direction gate (the backswing
*is* the signal, which is why the fix comment says the downswing *"falls through to the calm branch
below so the downswing that follows is a fresh swing with a cleared strike list"*). SABER's timed
per-victim is the most robust and the easiest to balance. SHIELD's global clock is the weakest: a
bash into a crowd damages the first thing the iterator returns and then goes on cooldown for 10
tics, so which monster gets hit in a crowd is blockmap order.

Note also that **SABER's twirl shares `ubqs_cut_rate` with the swing deliberately** — *"the SAME
cvar the swing uses, so the two cooldowns cannot drift apart"* — which is the right instinct and is
worth generalising: one cooldown cvar per weapon, not per attack mode.

#### 3.4 Damage, and who owns the result

This is the biggest architectural difference in the three mods, and BD22's answer is the one worth
copying.

**BD22 owns no damage numbers at all.** It spawns the host mod's own attack actor, at the contact
point, at the height the blade met the body, pointed the way the blade was travelling, and lets BD22
decide everything else:

```
void Strike(PlayerPawn mo, int k, Actor t, Vector3 at, Vector3 dir, double power)
{
	// start BD22's attack just inside the body, at the height the blade met it
	double rx = at.x - t.pos.x;
	double ry = at.y - t.pos.y;
	double rl = sqrt(rx * rx + ry * ry);
	double lim = max(t.radius - 2, 1);
	if (rl > lim)
	{
		rx *= lim / rl;
		ry *= lim / rl;
	}
	Vector3 p = (t.pos.x + rx, t.pos.y + ry, clamp(at.z, t.pos.z + 1, t.pos.z + max(t.height - 1, 1)));
	...
	if (k == AXE)
	{
		Launch(mo, 'AxeAttack', p, dir);
		if (power >= 1.6)
			Launch(mo, 'AxeAttackNoDecal', p, dir);
		if (power >= 2.4)
			Launch(mo, 'AxeAttack', p, dir);
	}
	else
	{
		Launch(mo, 'DSAttack', p, dir);
		Launch(mo, 'DSAttack2', p, dir);
	}
	Buzz(mo);
}
```
```
static void Launch(PlayerPawn mo, Name cname, Vector3 p, Vector3 dir)
{
	class<Actor> cls = cname;
	if (!cls)
		return;
	let a = Actor.Spawn(cls, p, ALLOW_REPLACE);
	if (!a)
		return;
	a.target = mo;
	a.angle = VectorAngle(dir.x, dir.y);
	a.pitch = -asin(clamp(dir.z, -1, 1));
	a.vel = dir * a.speed;
	a.CheckMissileSpawn(0);
}
```

Six consequences fall out for free, all named in the file header: *"BD22's gore rules decide the
result: its blood actors mark a head or leg hit from the impact height, and the Saw / Cut deaths take
heads, legs, arms or cut the body in half from there."* Swing power scales the result by **stacking
more of the host's attacks** (1 / 2 / 3 at `power` ≥ 1, 1.6, 2.4) rather than by multiplying a
number — so a hard swing produces the host mod's own double-hit gore, not a bigger integer. The
`class<Actor> cls = cname;` through a `Name` means an absent host mod is a run-time null, not a
compile error. And because the attack is a positioned projectile, **the engine's own thrust works
normally** — which is the problem the other two spend code on.

**SABER passes the Weapon item as inflictor and therefore cannot thrust:**

```
mo.DamageMobj(self, owner, dmg, 'Saber', DMG_THRUSTLESS);
// ...AND IT MOVES WHAT IT CUT. The engine's own thrust cannot be used: the inflictor is this
// Weapon item, which has no position of its own, which is what DMG_THRUSTLESS is for. So the
// shove is ours, along the arc, scaled by the victim's MASS (an imp is 100, a Cyberdemon 1000)
// and by the same swing-speed multiplier the damage uses. The lift is what makes it read as a
// stagger instead of a slide: a Doom monster off the floor cannot change direction.
if (mo && !mo.bDestroyed && shove > 0.01 && !mo.bDontThrust && !mo.bNoDamageThrust)
{
	double kk = shove * 5.0 * clamp(100.0 / max(mo.mass, 1.0), 0.08, 2.0) * mult;
	mo.vel += sw * kk;
	mo.vel.z += kk * lift;
}
```

The shove direction `sw` is **the arc the blade actually travelled**, taken before the history moves
on and with the body's own motion already out of it:

```
// THE ARC THE BLADE ACTUALLY TRAVELLED, which is the direction a cut throws a body. Taken here,
// before the history moves on, so it is the same step `speed` was measured from; the body's own
// motion is already out of it, so walking into a monster does not shove it.
Vector3 sw = lastOk ? (((hand + dir * len) - lastTip) - bodyStep) : (0, 0, 0);
if (sw.Length() < 0.01) sw = dir;
sw = sw.Unit();
```

That is the right direction and it is a much better answer than "away from the player", which is
what engine thrust would have given. The damage scale:

```
double mult = 1.0 + (clamp(speed / 20.0, 0.5, 2.0) - 1.0) * clamp(UF("ubqs_swing_scale", 1.0), 0.0, 1.0);
```
with an explicit accessibility reason for the knob: *"0 = not at all, which is what anyone seated or
in a small room needs — they live on the 0.5 floor of this curve for good otherwise. Its slider
stops at 1 because above 1 the floor drives the product toward zero, and a cut that draws blood and
plays a hit for no damage is worse than none."* That last clause is a real design rule: **never let
a tuning knob reach a state where the feedback fires and the damage does not.**

**SHIELD passes the pawn as inflictor** — `t.DamageMobj(mo, mo, dmg, 'Melee')` — so engine thrust
*would* work, and it still rolls its own, for the reason the engine's cannot address:

```
// AND SCALED BY WHAT YOU HIT. Thrust() adds velocity outright and ignores Mass, so a
// Cyberdemon (1000) left the plate exactly as fast as an imp (100) and the shove read
// as a scripted launch instead of as weight. The lift takes the same factor, so a
// heavy thing leans where a light one is thrown.
double mk = clamp(100.0 / max(t.mass, 1.0), 0.08, 2.0);
double knock = clamp(eff / 20.0, 4, 16) * SSQ_Handler.SV('ssq_bash_knock', 1.35) * mk;
t.Thrust(knock, atan2(n.y, n.x));
if (!t.bNoGravity && !t.bDontThrust)
	t.vel.z += SSQ_Handler.SV('ssq_bash_lift', 2.5) * mk * (t.player ? 0.5 : 1.0);
```

SABER and SHIELD arrived at the **identical mass formula**, `clamp(100.0 / max(mass, 1.0), 0.08, 2.0)`,
and the identical lift-under-the-shove reasoning, independently. Both also halve or omit the effect
for players (`t.player ? 0.5 : 1.0` — *"so a co-op partner is shoved, not launched"*).

#### 3.5 Gore and tiers

**SABER** had its own bug worth recording, because it is a bug that looks like working code:

```
// He was right, and for two reasons that both look like working code. The entry flash was UXHF at
// scale 0.12 -- on a 96-pixel sprite that is nine pixels, invisible beside the blade's own glow. And
// UBQS_Data.Sparks sat in the `else` for bNoBlood actors, which no monster is: the CombatFX spark
// burst was already shipped in this pack and no kill had ever fired it. Smoke was simply absent.
// A cut does not bleed, it cauterizes.
```

A feature in the `else` of a branch no monster takes, and a sprite scaled to nine pixels. Both pass
review; neither is visible. The fix is worth reading for its *scale-with-the-victim* idea
(`double bs = clamp(mo.radius / 20.0, 0.6, 2.2);` — radius 20 is an imp, 40 a Cyberdemon).

**SHIELD's disc** is the only one with tiered outcomes by `SpawnHealth()`, and it is the only place
in the three mods where a mod deliberately forces a gib:

```
if (tiers && !victim.bBoss && sh <= fodder)
{
	// bisect: past the gib threshold every time
	int gib = max(victim.health + abs(victim.GibHealth) + 1, base);
	bool ne = victim.bNoExtremeDeath;
	victim.bNoExtremeDeath = false;
	victim.DamageMobj(self, thrower, gib, 'Saw', DMG_FORCED);
	if (victim && victim.health > 0) victim.bNoExtremeDeath = ne;
```

Temporarily clearing `bNoExtremeDeath` and restoring it **only if the victim survived** is the
correct shape (restoring it on a dead actor is harmless but restoring it on a destroyed one is a
null deref waiting to happen). `health + abs(GibHealth) + 1` is the right arithmetic for "past the
gib threshold whatever the monster's own numbers".

#### 3.6 Strike-after-search

BD22 is the only one that separates the search from the effect:

```
// strike after the search: the hits spawn gore and kill things
for (int i = 0; i < victims.Size(); i++)
{
	Actor t = victims[i];
	if (!t || t.health <= 0)
		continue;
	struck.Push(t);
	Strike(mo, k, t, (hx[i], hy[i], hz[i]), (dx[i], dy[i], dz[i]), power);
}
```

This matters because `Strike` spawns actors and kills things, and **mutating the world inside a
`BlockThingsIterator` walk is undefined in practice** — a gib spawns actors into the blockmap, a
death can remove a link, and the iterator may revisit or skip. SABER and SHIELD both damage inside
the walk. Neither reports a bug from it, so it evidently survives in practice on this engine, but
BD22's two-phase shape is the one to copy, and it costs six parallel arrays only because
**`Array<Vector3>` is not allowed on this VM** — which is why BD22 carries `hx/hy/hz/dx/dy/dz` and
SHIELD carries `crumbX/crumbY/crumbZ`:

```
Array<double> crumbX, crumbY, crumbZ;	// Array<Vector3> is not allowed
```

#### 3.7 Verdict — resolution and result

**Best answer today: BD22, clearly.** Hand-to-contact-point `LineTrace` instead of player-eye
`CheckSight`; a two-phase search-then-strike; per-swing victim scoping; and above all **delegating
the result to the host mod's own attack actor at the measured contact point**, which buys
height-dependent gore, correct engine thrust, mod-agnostic balance and a null-safe optional
dependency for about twelve lines.

**A fourth implementation should:**

1. `BlockThingsIterator` from the swept volume's bounding sphere, every cvar read above it.
2. Filter in cost order, cheapest first, with the reachability test last.
3. Use a **hand-to-contact `LineTrace`** with `TRF_ABSPOSITION | TRF_THRUACTORS`, accepting
   `HitNone`/`HitActor`. `CheckSight` is the wrong question for a melee weapon.
4. Collect, then strike. Carry positions as parallel `Array<double>`, because `Array<Vector3>` is
   not available.
5. Scope the cooldown to **the swing**, with a short per-victim timer underneath it as a floor, and
   one shared cvar across all of the weapon's attack modes.
6. If a host mod exists, **spawn its attack actor at the contact point and height** and let it own
   the result. Scale power by stacking attacks, not by multiplying a number. Resolve the class
   through a `Name` so an absent host is a null.
7. If you must damage directly, pass **the pawn** as inflictor (not the Weapon item), and still roll
   your own thrust: `clamp(100.0 / max(mass,1), 0.08, 2.0)` along the arc the blade travelled, with
   a lift under it, halved for players.
8. Never let a feedback path fire on a configuration where the damage is zero.

---

### 4. Defence

**BD22 has none.** No `AbsorbDamage`, no `ModifyDamage`, no `WorldHitscanPreFired`, no deflection,
no reflection flag — I grepped its whole `zscript/` and `actors/` tree. It is an offence-only melee
system. Everything in this section is SABER against SHIELD.

#### 4.1 The three-layer architecture, invented twice

SABER and SHIELD independently converged on the *same three layers*, in the same order, for the same
reasons. That convergence is the strongest signal in this document that the architecture is right.

| Layer | When | SABER | SHIELD |
|---|---|---|---|
| **1. Pre-trace cancel** | before the bullet exists | `StopsHitscan` via `WorldHitscanPreFired` | `StopsHitscan` via `WorldHitscanPreFired` |
| **2. Proactive sweep** | every tic, before contact | `Deflect` — missiles near the blade | `DoEffect` sweep — missiles near the plate |
| **3. Damage-time backstop** | after it has hit | `ModifyDamage(passive)` | `AbsorbDamage` |

The reasoning is stated almost identically in both. SHIELD:

```
// AbsorbDamage only ever runs when something is already damaging you, so the old guard
// could only block what was going to hit anyway, at the exact instant it hit. That is why
// the window felt small: a shot is tested against one line, drawn to your centre, on one
// tic. The owner's lightsaber does the opposite and it is the better idea -- every tic it
// looks for missiles NEAR THE BLADE and bats them away [...]
```

SABER, on the hitscan half:

```
// THE MUZZLE. The hitscan block lived in ModifyDamage, which the engine calls only once the shot has
// already reached you and is applying damage -- and it tested a GUESSED line, the attacker's middle
// to yours with a few units of slack. So it blocked shots that would have missed, missed shots that
// really were coming, and could never stop anything the blade stood in front of that was not aimed
// at your centre.
```

#### 4.2 Layer 1 — the pre-trace cancel

`WorldHitscanPreFired` **does exist on stock** — verified:
`670d19eba6^:wadsrc/static/zscript/events.zs` declares
`virtual bool WorldHitscanPreFired(WorldEvent e) { return false; }`, and
`670d19eba6^:src/playsim/p_map.cpp:4718` is the very first statement of `P_LineAttack`:

```
if (t1->Level->localEventManager->WorldHitscanPreFired(t1, angle, distance, pitch, damage,
        damageType, pufftype, flags, sz, offsetforward, offsetside))
{
	return nullptr;
}
```

Returning true cancels the shot whole — no trace, no puff, no damage, for anybody.

**The event does *not* carry the muzzle.** `events.cpp` fills only
`Thing, AttackAngle, AttackPitch, AttackDistance, Damage, DamageType, AttackPuffType,
AttackOffsetForward, AttackOffsetSide, AttackZ, AttackLineFlags`. The `WorldEvent` struct *has* an
`AttackPos` field, but it is populated in `WorldHitscanFired` (after the trace) and left stale here.
So both mods are right to compute the origin themselves — and **this is where they differ, and where
SABER is wrong as of `828c070`.**

Engine truth (`p_map.cpp`):
```
shootz = t1->Center() - t1->Floorclip + t1->AttackOffset();
...
if (flags & LAF_OVERRIDEZ)
	shootz = t1->Z();
shootz += sz;
```
and `actorinlines.h`: `AttackOffset(offset)` returns **`8 + offset`** for a non-player.

SHIELD implements that formula exactly:
```
// WHERE THE SHOT REALLY STARTS, by P_LineAttack's own formula (p_map.cpp on stock QZD
// 17.3): shootz = Center() - Floorclip + AttackOffset() + sz, and AttackOffset() is a flat
// 8 for anything that is not a player. sz is the attack's own z offset, which the event
// hands us as AttackZ and this never read. height * 0.75 was a guess sitting 6 units high
// on an imp and 14 on a Cyberdemon [...]
// LAF_OVERRIDEZ measures from the actor's feet instead.
double mz = ((e.AttackLineFlags & LAF_OVERRIDEZ) ? 0.0 : (shooter.height * 0.5 - shooter.Floorclip + 8.0)) + e.AttackZ;
Vector3 A = shooter.pos + (0, 0, mz);
```

SABER does not:
```
// the muzzle, by P_LineAttack's own formula: the shooter's centre plus the attack's own z
Vector3 from = (sh.pos.xy, sh.pos.z + sh.height * 0.5 + e.AttackZ);
```

Missing `AttackOffset()`'s flat **+8**, missing `Floorclip`, and not handling `LAF_OVERRIDEZ` (where
the engine measures from the feet, so SABER would be `height/2 + 8` units high). Against a 26-unit
blade with a 6-unit guard width at twenty metres, 8 units of origin error is most of the guard — this
is the same defect SHIELD's `2d10397` commit was written to fix, in a smaller dose. The comment even
claims the formula it does not implement. **Flagged as of `828c070`; the session editing that file
may already have fixed it.**

Both then require the shot to be stopped *in front*, not caught on its way elsewhere:

```
// SABER: and it has to be stopped IN FRONT of us, not caught on its way to something behind us
if ((ray * t).Length() > toMe.Length() + owner.radius * 2.0) return false;
```
```
// SHIELD: it has to be coming at US, not merely passing the shield on its way elsewhere: the
// crossing point must be nearer the shooter than we are, give or take our own width.
if ((hitP - A).Length() > (centre - A).Length() + mo.radius * 2) return false;
```

Identical idea, identical slack (`radius * 2`), written twice.

**A player's own shots are never tested**, in both — which matters more than it looks, because
`P_LineAttack` calls this hook *before* its own haptic block, so SABER's `Buzz()` trick (a 1-unit
damage-1 `LineAttack`, §6) would otherwise be eaten by its own shield.

**One interaction neither mod documents:** `EventManager::WorldHitscanPreFired` loops handlers with
`ret == false` as the continue condition, so **the first handler to return true short-circuits the
rest**, and handler order is registration order, i.e. load order. With both pk3s loaded, a bullet the
saber eats never reaches the shield's test. That is benign for gameplay (one block, one set of FX)
but it means *which mod's geometry decides* depends on pk3 order — the exact class of problem
SHIELD's `dbef2ec` commit solved for the off-hand slot and did not consider here.

#### 4.3 Layer 2 — the proactive sweep

SABER sweeps the blade against the missile's own path, from a tic behind to `ubqs_deflect_lead` tics
ahead:

```
Vector3 pb = lastOk ? lastBase : base;
Vector3 pt = lastOk ? lastTip : base + dir * len;
// ubqs_deflect_lead: how far along a shot's OWN path the blade is tested, in tics. 3 is what fixed his
// report that one blade held across did not turn Wardust's fast bolts -- and it is also why shots are
// turned two tics early, including ones that would have missed. Only he can say whether that now feels
// psychic.
double dlead = clamp(UF("ubqs_deflect_lead", 3.0), 1.0, 6.0);
...
for (int k = 0; k <= 2; k++)
{
	double t = k * 0.5;
	Vector3 b = pb + (base - pb) * t;
	Vector3 e = pt + ((base + dir * len) - pt) * t;
	near = min(near, UBQS_Data.SegSeg(level.Vec3Diff(b, mo.pos - mo.vel), mo.vel * dlead, e - b));
}
if (near > mo.radius + 5.0) continue;
BatBack(mo, dir);
```

SHIELD sweeps the plate against the same window, `ssq_sweep_lead` also defaulting to 3:

```
Vector3 A = sh.pos - v;				// a tic behind
Vector3 B = sh.pos + v * lead;		// and a few ahead
```

Both honestly flag the cost of the lead: a shot that *would have missed* is turned, two tics early.
SABER: *"Only he can say whether that now feels psychic."* SHIELD: *"including shots that would have
missed."* That is the right thing to write down rather than hide.

**SHIELD's sweep handles the turn-not-replace distinction and SABER's does not care, correctly:**

```
// A SWEPT SHOT IS TURNED, NOT REPLACED. The block path spawns a fresh missile because the
// original is already mid-impact by then; out here it has not hit anything yet, so it just
// gets a new heading -- no spawn, no second sound, and a seeker keeps seeking.
```

Both share the critical two lines in the turn:
```
sh.target = mo;						// ours now: it hurts them, never us
if (sh.bSeekerMissile) sh.tracer = shooter;
```
Re-targeting is what stops a turned shot from immediately turning round, and `tracer = shooter` is
what makes a *seeker* chase the thing that fired it. Both mods have it; it is easy to miss.

Both also exclude their own thrown weapon from their own sweep (`mo is "UBQS_Thrown"` /
`sh is 'SSQ_Disc'`) and anything already theirs (`mo.target == owner` / `sh.target == mo`).

**Why the sweep exists at all in SHIELD is an engine fact worth carrying forward:**

```
// WHAT CROSSES THE FACE IS STOPPED. Not by sweeping the missile list: the plate sits inside
// the player's own collision box, and the engine kills a missile at the box edge before any
// sweep sees it reach the plate (the 2026-10-17 audit).
```

A guard surface that sits **inside** the player's own radius can never be reached by a missile,
because the missile dies on the player's box first. I did not re-derive this from the engine, so it
is SHIELD's audit rather than my verification — but it is the reason the sweep must test a *lead*
rather than a contact, and it generalises to any shield, plate or bracer held close to the body.

#### 4.4 Layer 3 — the backstop, and the hook choice

**SABER uses `ModifyDamage`; SHIELD uses `AbsorbDamage`. SABER's hook is the better one**, and the
engine says so in its own comment at `670d19eba6^:wadsrc/static/zscript/actors/inventory/inventory.zs`:

```
// It may seem that this is redundant and AbsorbDamage is the same. However,
// AbsorbDamage is called only for players and also depends on other settings
// which are undesirable for a protection artifact.
```

The gates, from `p_interaction.cpp`:

| Hook | Call site | Skipped when |
|---|---|---|
| `GetModifiedDamage(..., passive=true)` | line 1231 | `flags & DMG_NO_PROTECT` |
| `AbsorbDamage` (player) | line 1363 | `flags & DMG_NO_ARMOR` — which the engine **sets automatically** for any inflictor with `+PIERCEARMOR` (line 1151) |

So **any `+PIERCEARMOR` projectile walks straight through SHIELD's backstop**, and `AbsorbDamage`
also sits behind the invulnerability/godmode checks and after the passive `ModifyDamage` chain has
already run. In practice SHIELD's exposure is small because layers 1 and 2 catch most of what layer 3
would, but the hook choice is a free improvement.

**SHIELD's backstop is better in a different way: it classifies the attack.** Four kinds, four
different test segments:

```
int kind;	// 0 missile, 1 hitscan, 2 melee, 3 blast
if (inflictor && inflictor.bMissile && !(flags & DMG_EXPLOSION))
{
	kind = 0;
	Vector3 v = inflictor.vel;
	double sp = v.Length();
	Vector3 u = sp > 0 ? v / sp : (centre - inflictor.pos).Unit();
	A = inflictor.pos - u * (sp * 2 + 8);
	B = inflictor.pos + u * (mo.radius * 2 + 24);
}
else if (flags & DMG_EXPLOSION) { kind = 3; if (!inflictor) return; A = inflictor.pos; B = centre; }
else if (source && inflictor == source && !source.player) { kind = 2; A = source.pos + (0, 0, source.height * 0.6); B = centre; }
else
{
	kind = 1;
	if (!source) return;
	A = (source.player && source.OverrideAttackPosDir) ? source.AttackPos : source.pos + (0, 0, source.height * 0.75);
	B = centre;
}
```

A missile is tested along **its own velocity**, extended back `sp*2+8` and forward
`radius*2+24` — the right test. A blast is attacker-to-you. Melee is identified by
`inflictor == source && !source.player` and tested from 0.6 of the attacker's height. Only the
hitscan fallback still guesses (`height * 0.75`), and that path is mostly pre-empted by layer 1.
SABER's backstop has one test for everything (`BladeInPath(source, w)`: attacker-centre to
your-centre against the blade segment), which is coarser.

SABER's backstop carries a legacy path worth reading as a cautionary tale, because it is the
*un-geometric* version of the same feature and the mod had to decide not to layer them:

```
// IT REPLACES THE ROLL, it does not stack on it. The roll below is 70 per cent of everything inside
// 150 degrees of the blade's aim, which swamps any amount of precision -- a geometric guard layered
// over it would be unfeelable. ubqs_blade_guard 0 is exactly the old behaviour, roll and all.
// The roll also bailed out inside 96 units, so a claw was never blocked by anything but the cross
// guard. The geometric path has no distance floor: a blade across you stops a claw, which is right.
```

and the geometric claim that makes the feature work at all:

```
// NOTHING HERE ASKS WHETHER THE BLADE IS HORIZONTAL. Held across your chest it crosses the line and
// blocks; held pointing forward it is edge-on to the line and blocks nothing. That falls out of the
// geometry, which is the whole reason it will feel real instead of feeling like a dice roll.
```

That is the single best sentence about VR defence in either codebase. **Do not add a cone, a dice
roll or a facing arc; put a shape in space and test the line against it.**

SABER's cross-guard is the one place it kept an arc, and the file calls it out:

```
// ubqs_guard_arc: 100 either side is 200 degrees of TOTAL immunity, measured from the BODY's
// yaw and not from the blades. The most abusable number in the mod, with no way to tighten it.
// ubqs_guard_arc is the outer gate: nothing from behind you is guarded, whatever the blades
// are doing. ubqs_guard_width is the wall itself -- ONE of the two crossed blades has to be
// in the attack's own path, exactly as the single-blade block is tested, just wider [...]
// This is what makes WHERE you cross them matter.
```

So the arc survived as an outer gate with a geometric test inside it. That is the right compromise
when the shape (two crossed blades held in a trembling grip) is genuinely bigger and fuzzier than a
single segment.

#### 4.5 The parry — reactive versus proactive, and the thing neither latches

Both mods independently added "you *met* the attack" as distinct from "you *held* the surface in
the way", both measured it as the defending surface's step **into the incoming direction**, and both
refused to latch it.

SABER:
```
// ---- THE PARRY: the blade going INTO the shot, not merely standing in it --------------------
// The blade's own step this tic (the body's motion out of it, as everywhere else) against the
// direction the shot was travelling. 1.0 is a block; above it is a parry. Nothing is latched and
// nothing is remembered: it is the step that is happening when the shot arrives, which is what the
// player felt themselves do. ubqs_parry_speed is in the SAME units as ubqs_swing_cut.
double ParryGain(Vector3 shotDir)
{
	if (!lastOk) return 1.0;
	double need = UF("ubqs_parry_speed", 6.0);
	if (need <= 0.01 || shotDir.Length() < 0.001) return 1.0;
	Vector3 step = ((BladeBase() + BladeDir() * BladeLen()) - lastTip) - bodyStep;
	double sp = step.Length();
	if (sp < need) return 1.0;
	Vector3 su = step / sp;
	Vector3 sd = shotDir.Unit();
	double into = -(su dot sd);
	if (into < 0.3) return 1.0;
	return 1.0 + (clamp(UF("ubqs_parry_power", 2.0), 1.0, 4.0) - 1.0) * clamp(into, 0.0, 1.0);
}
```

SHIELD:
```
// PERFECT BLOCK: you did not hold the shield in the way, you MET the shot with it. The
// plate has to have been moving into the attack at ssq_perfect_speed (units/sec along
// the line it came in on) at the moment it landed -- a push, a parry, a swat, not a
// stance.
bool perfect = false;
if (held != 0 && SSQ_Handler.SVI('ssq_perfect', 1) != 0)
{
	let h = SSQ_Handler.Get();
	let pose = h ? h.PoseOf(mo.PlayerNumber()) : null;
	if (pose) perfect = (pose.Vel() * TICRATE) dot outward >= SSQ_Handler.SV('ssq_perfect_speed', 40.0);
}
```

Two differences. SABER's is **continuous** — `ParryGain` returns a multiplier from 1.0 up to
`ubqs_parry_power`, scaled by how squarely the blade met the shot (`into`), with a 0.3 floor on the
alignment — and it feeds *everything*: the return bolt's speed, the spark count, the flash radius,
the sound pitch and the rumble length all come off the same `pg`. SHIELD's is **binary** but it
requires the surface to be held (`held != 0`, not stowed) and it reads off the **synced** pose, so
it is the same answer on every machine.

And SABER replaced a latch with this, which is the real lesson:

```
// THE PARRY. A block was pass/fail: the only reward for a good one was a blanket third of a second
// of immunity after any fast swing, which a twirling player never leaves.
```

That latch (`ubqs_block_swung`, 12 tics of total immunity after any fast swing) is still in the file
as the legacy path, with the twirl noted as setting `lastFastTic` *every single tic* so a twirling
player is permanently immune. **A time-window reward for a gesture is an exploit whenever the
gesture can be held.** The replacement — read the step that is happening at the moment the attack
arrives, latch nothing — is correct, and it is cheaper.

#### 4.6 Where a turned shot goes

Both mods implement the same three aim modes and both route the block path and the sweep path through
**one function** so they cannot disagree. SHIELD states the rule:

```
// WHERE A TURNED SHOT GOES, for both the sweep and the block, so they never disagree.
// ssq_deflect_aim: 0 straight back at whoever fired it, 1 where you are LOOKING (helped
// onto the nearest enemy within ssq_deflect_assist of your gaze), 2 mirrored off the face.
// 1 is the default because it is the one that makes a shield feel like a weapon: you catch
// a fireball on the plate and send it wherever you turn your head.
```

SABER's mirror is the better-written of the two mirrors, because it reflects about the plane
containing the blade rather than about a single normal:

```
Vector3 v = mo.vel.Unit();
Vector3 n = (blade cross (v cross blade));
if (n.Length() > 0.001) { n = n.Unit(); d = v - n * (2.0 * (v dot n)); }
```

That is the right construction for a *line* (a blade has no unique normal); SHIELD's
`d = u - n * (2 * (u dot n))` is the right one for a *surface*. Both then fall back through the same
ladder — gaze target, look direction, shooter, reverse velocity, an arbitrary unit vector — so the
direction is never zero.

`GazeTarget` is near-identical in both, scoring on angle and distance together
(SABER: `off * 40.0 + dist * 0.05`; SHIELD: `ang + dist / range * 4.0`), with `CheckSight` required.
SABER's carries a useful warning about a coincidence that looks like a link:

```
// ubqs_deflect_assist: the aim help that puts a deflected shot onto an enemy. It happens to equal
// ubqs_lock_cone's default, which makes it LOOK linked to it -- it is not, and turning
// ubqs_lock_cone does nothing here.
```

#### 4.7 Verdict — defence

**Best answer today: SHIELD's layering and classification, on SABER's hook, with SABER's parry
curve.** Specifically: SHIELD's muzzle formula, SHIELD's attack-kind classification, SHIELD's
turn-not-replace on the sweep path, SABER's `ModifyDamage` instead of `AbsorbDamage`, SABER's
continuous `ParryGain` driving every feedback channel, and SABER's refusal to use a cone or a dice
roll anywhere the geometry can answer.

**A fourth implementation should:**

1. Build all three layers. They are not alternatives: the pre-trace cancel is the *only* way to stop
   a bullet before it hits, the sweep is the only way to turn a fast missile early, and the
   damage-time hook is the only thing that catches blasts and claws.
2. Compute the muzzle with `Center() − Floorclip + AttackOffset() + sz`, `AttackOffset() == 8` for
   non-players, and handle `LAF_OVERRIDEZ`. Do not use the event's `AttackPos` — it is not filled on
   the pre-fire. Do not guess `height * 0.75`.
3. Use `ModifyDamage(passive)`, not `AbsorbDamage`, so `+PIERCEARMOR` does not walk through you.
   Classify the attack into missile/hitscan/melee/blast and build the right test segment for each.
4. Remember that the first handler to return true from `WorldHitscanPreFired` wins and order is load
   order. If two of your mods guard, decide which, explicitly.
5. Re-target a turned projectile (`target = you`) and re-tracer a seeker (`tracer = shooter`).
6. Turn, don't replace, where you caught it before it hit; replace only where it is already
   mid-impact.
7. Make the parry **continuous and unlatched**: the surface's step into the incoming direction at
   the moment of arrival, feeding every feedback channel off one gain. Never grant a timed immunity
   window for a gesture that can be held.
8. One `DeflectDir` for every path that turns anything.
9. Resist the cone. If you want a shape to be bigger because it genuinely is bigger (crossed
   blades), widen the geometric test, don't add an arc — and if you must keep an arc, make it an
   outer gate with a geometric test inside it.

---

### 5. Throwing and returning

#### 5.1 Launch velocity and heading

| | Speed | Heading | Blend |
|---|---|---|---|
| **BD22** | `clamp(vel.Length() * 3, 14, 42)` | raw world delta, body travel included (see §1.5) | none |
| **SABER** | `clamp(16.0 + sp * 4.0, 22.0, 44.0)`, × `ubqs_throw_speed` | `hv.Unit() * bl + Aim() * (1 − bl)` | `ubqs_throw_blend` 0.6 |
| **SHIELD** | `hv.Length() * keep * ssq_throw_speed * (1 + revFrac * bonus)`, floored at `ssq_speed_floor` | `hv.Unit() * bl + pose.F * (1 − bl)` | `ssq_throw_blend` 0.7 |

SABER and SHIELD both carry the same slider with the same framing, written twice:

```
// SABER: ubqs_throw_blend: how much of the throw is the direction the arm actually MOVED and how much
// is where the hand points. 1 = all arm (honest), 0 = all aim (accurate) -- the classic VR slider.
```
```
// SHIELD: HONEST VERSUS ACCURATE. The arm's own direction is what you actually did; where the
// hand points is what you meant. ssq_throw_blend slides between them -- 1 all arm,
// 0 all aim -- which is the classic VR throw slider and what the owner's saber carries.
```

SHIELD adds a mass term SABER does not have, which is a nice touch for a heavy object:

```
double keep = 30.0 / (30.0 + SSQ_Handler.SV('ssq_mass_kg', 5.0));
```

and a charge term (`revFrac`, 0..1 over `ssq_rev_tics` 21 tics of holding the button) that scales
both speed and damage, so a revved throw is a stronger throw.

#### 5.2 The release gate

All three require more than speed, and all three learned it from the same complaint.

**SABER** — speed, plus "forward relative to the **head**":
```
// ubqs_throw_straight: ubqs_throw_min says how HARD the throw must be, this says how FORWARD, and only
// the first was exposed. It is measured against the HEAD (owner.pitch / owner.angle), so a seated or
// swivel-chair player throwing across his body gets nothing. Exposed as the dot it is, not converted
// to degrees -- 0.55 is not the cosine of any round number.
return (hv / sp) dot fwd >= UF("ubqs_throw_straight", 0.55);
```
**SHIELD** — the same, at a looser cosine, with the same reason:
```
// A THROW IS A FORWARD SNAP, NOT ANY FAST HAND. Letting go at the end of a block, a
// backhand or a chop used to launch the shield because only raw speed was tested. The
// hand has to be going out the way you are looking -- ssq_throw_forward is the cosine
// of how far off that it may be (0.35 is about 70 degrees). 0 takes any direction back.
```
**BD22** has no direction gate on the throw — only `THROW_MIN` — but its throw is on an explicit
*hold-and-release* input, so the gesture is already bracketed.

SABER's note about measuring against the **head** is the one to keep, and it names the cost:
a seated player throwing across their body gets nothing. SHIELD's 0.35 (≈70°) is the more forgiving
number and is probably the better default for that reason.

SABER also solved the opposite problem — a release that *should* throw and was demanding a gesture
that changed nothing:

```
// A PAINTED RELEASE IS ALREADY A THROW. Throw() does not read the arm at all when there is a
// route -- it flies the ringed targets in order -- so demanding a forward snap on top of the
// marks asked for a gesture that changed nothing, and threw the marks away when it was missed.
// With nothing marked the snap is still the whole decision, and nothing below this changed.
if (invoker.locks.Size() > 0 && invoker.UInt("ubqs_throw_on_lock", 1)) invoker.Throw();
else if (invoker.ThrowRelease(thr)) invoker.Throw();
else invoker.keepMarksUntil = level.maptime + 35;
```

And both SABER and SHIELD keep the marks alive across a brief release, so re-gripping does not cost
you your targets (`keepMarksUntil = maptime + 35` / `ssq_keep_marks = 35`).

#### 5.3 Aim assist

**BD22 has the only ballistic one, and it is the best single idea in its throw:**

```
// a throw within 8 degrees of a visible monster goes to it, arced for the drop
static Vector3 AimAssist(PlayerPawn mo, Vector3 from, Vector3 dir, double spd)
{
	...
	Vector3 to = best.pos + (0, 0, best.height * 0.5) - from;
	double tics = to.Length() / spd;
	to.z += 0.5 * mo.GetGravity() * tics * tics;
	return to.Unit();
}
```

It leads the target for gravity over the flight time. SABER's and SHIELD's thrown objects are
`+NOGRAVITY` and steer, so they do not need it — but any *ballistic* thrown melee weapon does, and
this is three lines.

**SABER and SHIELD both steer instead, and both hit the same wall:** a steering gain that
renormalises cannot catch a target the throw did not point at.

SHIELD's version of the fix:
```
// LEAVE TOWARD THE FIRST RING. The steering blends only 22% of the wanted heading per
// tic and renormalises to the launch speed, so it can never catch up with a target the
// swing did not point at -- it loops instead, which is most of what reads as a weird
// slow throw. If a lock is already ringed and is not behind the swing, the disc leaves
// along it and the steering only has to correct for the target moving.
```
and the companion fix, that renormalising also pinned the speed:
```
// IT CAN BUILD SPEED. The steering renormalised to the launch speed every tic,
// so a disc that left slowly stayed slow all the way round the room however
// many targets it chained. Now it winds up toward ssq_cruise_max while it has
// somewhere to be, at ssq_cruise_gain a tic [...]
double gain = SSQ_Handler.SV('ssq_cruise_gain', 0.05);
double top = SSQ_Handler.SV('ssq_cruise_max', 34.0);
if (gain > 0 && cruise < top) cruise = min(top, cruise * (1.0 + gain));
Vector3 want = d / dist * cruise;
vel = vel * 0.78 + want * 0.22;
double vl = vel.Length();
if (vl > 0) vel = vel / vl * cruise;
```

#### 5.4 The return, and the two ways it fails

**SHIELD returns along breadcrumbs it dropped on the way out** — `crumbX/crumbY/crumbZ` pushed every
`ssq_crumb_every` (4) tics, walked backwards on the way home, with three escape hatches:

```
// IT COMES BACK. EVERY TIME. A disc walking crumbs through a doorway it no longer
// fits through used to bounce until the 12-second age cap dropped it on the floor.
// Past ssq_home_patience the geometry stops being consulted: it flies straight to
// the hand. Nothing else in the mod depends on it being solid on the way in.
if (!bNOCLIP && homeTics > SSQ_Handler.SVI('ssq_home_patience', 105))
{
	bNOCLIP = true;
	crumbAt = -1;
}
```
plus a bounce handler that drops the crumb it is fighting:
```
SSQF A 0 { if (invoker.phase == PH_OUT) invoker.Home(); else if (invoker.phase == PH_HOME && invoker.crumbAt >= 0) invoker.crumbAt--; }
```
plus a shortcut: `if (crumbAt > 0 && thrower && CheckSight(thrower)) crumbAt = -1;`

That is three independent guarantees that the return terminates, and it is the right level of
paranoia for an object the player cannot get back any other way.

**SABER goes straight home through walls** — *"the Force does not path-find"* — with a 140-tic
timeout to a fall, and it recorded the two failures that matter:

```
// NEVER DIVIDE BY IT. Once the step is min(hs, d) == d the saber lands exactly on the hand
// point, so d really does reach 0 whenever two tics read the same pose (a hitch) -- and to / d
// then poisons dir, step and the actor's own position with NaN, with no PH_PULL timeout to
// recover from it: the hilt is gone for the rest of the level. PH_ROUTE already guards this.
double d = max(to.Length(), 0.001);
```
```
// IT HAS TO HAVE LEFT BEFORE IT CAN COME BACK. Catch() fires inside 26 units and the saber is
// SPAWNED at the hand, so a throw that ends at once is already inside its own catch radius:
// catch, destroy, return to hand, thrown again next release -- a hilt flickering at the hand.
// ONLY THE CATCH WAITS. An earlier build put this test above the movement, which stopped the
// saber flying at all for its first 17 tics and left it hanging in the air.
if (d < 26.0 && age >= 17) { Catch(); return; }
```

The second is worth reading twice: the fix for the first attempt at the fix was in the *wrong place*
and produced a different bug, and the comment records both. **An object spawned at the hand is
inside its own catch radius; age-gate the catch, not the movement.**

**BD22 does not return at all** — `ThrowedAxe` is BD22's own actor and behaves as BD22 made it. That
is the same delegation principle as §3.4 and it is a legitimate answer.

#### 5.5 Catching

**SHIELD** — the hand within `ssq_catch_radius` (24 units) of the disc, off the synced pose, with a
grace period so the catch does not immediately read as a holster press:

```
if (h && pnum >= 0 && SSQ_Handler.SVI('ssq_catch', 1) != 0 && h.Holding(pnum))
{
	let pose = h.PoseOf(pnum);
	if (pose.Fresh(Level.maptime) && (pose.pos[0] - pos).Length() <= SSQ_Handler.SV('ssq_catch_radius', 24.0))
	{ Done(true); return; }
}
```
```
if (caught)
{
	Draw(i);
	mCatchGrace[i] = 8;
	...
```
A miss lands on the mount (`MountPos`) rather than on the floor, which is the right failure mode for
a holstered item — *"a missed catch lands on the mount"*, and it is on the decided-do-not-reopen
list.

**SABER** — `Catch()` inside 26 units once `age >= 17`, plus a `PH_REST` walk-over pickup:
```
case PH_REST:
	// lying flat and dark; walk over it to take it back
	if (Distance2D(master) < master.radius + 24 && abs(master.pos.z - pos.z) < 48) { Catch(); return; }
```
Two recovery routes, which is right for something that can be left on the floor.

#### 5.6 Verdict — throwing and returning

**Best answer today: SHIELD for the velocity measurement and the return; BD22 for the aim; SABER for
the release semantics.**

**A fourth implementation should:**

1. Take the **peak** windowed velocity over the last few samples, not the velocity at release.
   People decelerate before they let go.
2. Measure it in the body frame and re-rotate once, so the *heading* is clean and not just the
   magnitude. BD22's throw and SABER's throw both skew the heading; SHIELD's does not.
3. Correct the vertical in the magnitude only.
4. Expose the honest/accurate blend. 0.6–0.7 toward the arm is where both mods landed.
5. Gate the release on speed **and** a forward cosine measured against the head, and say in the cvar
   comment what a seated player loses. ≈70° is kinder than ≈57°.
6. If the object is ballistic, lead for gravity over the flight time — three lines, from BD22.
7. If it steers, launch it **along** the first target rather than relying on the steering to find it,
   and let the cruise speed build rather than renormalising to the launch speed.
8. Make the return terminate three different ways: drop the obstacle you are fighting, shortcut when
   you can see the owner, and go `NOCLIP` straight home past a patience timeout.
9. `max(d, 0.001)` on every normalisation in a homing path, and age-gate the catch, not the
   movement.
10. A missed catch should land somewhere the player can get it, and preferably back on the holster.

---

### 6. Feedback

#### 6.1 Haptics — and this is where SABER found something the other two had given up on

There are two ways to buzz a Quest controller from ZScript on stock QZD 17.3, and all three mods use
one of them. They are not equivalent.

**Route A — fire a damaging attack and let the engine's own fire-weapon haptic fire.** Verified in
`p_map.cpp` (`P_LineAttack`) and `p_mobj.cpp` (`P_SpawnPlayerMissile`):

```
if ( damage > 0) {
	//Haptics
	long rightHanded = vr_control_scheme < 10;
	rightHanded = (flags & LAF_ISOFFHAND) ? 1 - rightHanded : rightHanded;
	vrmode->Vibrate(150, rightHanded ? 1 : 0, 0.8);
```

SABER's original `Buzz()` uses the `LineAttack` form, and its own comment explains the trick:

```
// Haptic tick on the saber hand. QZD has no script haptics; the engine buzzes the firing hand on any
// player hitscan with damage > 0. A 1-unit trace pointed straight up reaches nothing: no puff, no hit, no noise.
...
owner.LineAttack(owner.angle, 1, -90, 1, 'UBQS_Haptic', 'UBQS_HapticPuff', bOffhandWeapon ? 256 : 0);
```

`256` is `LAF_ISOFFHAND` (`p_local.h: LAF_ISOFFHAND = 1 << 8`), confirmed — so the hand selection is
correct.

BD22 uses the missile form, which fires off `DamageType` being non-`None` (verified: the engine
checks `DamageFunc != nullptr || DamageType.GetIndex() != 0 || SeeSound != NO_SOUND`):

```
// controller rumble: the engine buzzes the weapon hand when the player
// fires a projectile (p_mobj.cpp P_SpawnPlayerMissile)
static void Buzz(PlayerPawn mo)
{
	mo.SpawnPlayerMissile('RSVR_Buzz', mo.angle, noautoaim: true);
}
```
```
class RSVR_Buzz : Actor
{
	Default
	{
		Projectile;
		+NOCLIP
		Radius 1;
		Height 1;
		Speed 1;
		DamageType "RSVRHaptic";
		RenderStyle "None";
	}
```

Route A's limits, all three verifiable: **the duration and amplitude are fixed** (150 ms, 0.8),
there is **no strength parameter**, and — because `OpenXRDeviceMode::Vibrate` is a bare pass-through
to `QzDoom_Vibrate` with **no check of whose player fired** — in co-op **every player's shot buzzes
every player's controllers**. BD22's `SpawnPlayerMissile` also passes no `ALF_ISOFFHAND`, so an
off-hand event buzzes the weapon hand.

**Route B — `A_QuakeEx` with `damrad 0`.** SABER's `15f6332` commit is the one to read:

```
// THE COMMENT ABOVE IS WRONG ABOUT THIS ENGINE, and it cost the mod its best channel: A_QuakeEx IS
// the hook. r_utility.cpp drives vrmode->Vibrate from the quake's intensity -- X is the LEFT
// controller and Y the RIGHT -- scaled by vr_quake_haptic_level (0.8 by default), while the VIEW
// shake rides r_quakeintensity, which is 0 for VR. With damrad 0 the whole damage and thrust block
// is skipped, so this is a pure feedback channel, and a quake is a serialised thinker: the only
// haptic route on this engine that is net-safe and demo-safe. The spot is an actor following the
// pawn with a tremor radius of 8, so in co-op only its owner feels it, and walking does not walk
// you out of your own rumble.
```

Every clause of that checks out:
- `r_utility.cpp:1085-1092`: `QuakePower(vr_quake_haptic_level, jiggers.Intensity.X, ...)` → left; `.Y` → right.
- `r_quakeintensity` is `CUSTOM_CVAR(Float, r_quakeintensity, 0.0f, ...) // Defaulted to 0 for VR`.
- `DEarthquake::StaticGetQuakeIntensities(viewPoint.TicFrac, viewPoint.camera, jiggers)` — the **camera's** actor, so a tremor radius of 8 around one pawn is felt only by that pawn's own view.

So Route B gives you **a strength and a length**, it is **per-player in co-op**, and it is
deterministic. Route A gives you neither. The mod shipped believing Route B was impossible — *"QZD
gives script no hook at all"* in its own `Buzz()` comment — and **because of that belief the two
haptics that matter most, the blade cutting flesh and blades crossing, both defaulted OFF**:

```
void Rumble(double strength, int tics)
{
	if (!owner || !owner.player || !UInt("ubqs_haptics", 1)) return;
	double k = strength * clamp(UF("ubqs_haptic_power", 1.0), 0.0, 3.0);
	if (k <= 0.01) return;
	int lh = UInt("ubqs_lefty", -1);
	bool lefty = (lh < 0) ? UBQS_Data.WorldMirror(false) : (lh != 0);
	bool left = bOffhandWeapon != lefty;
	let q = UBQS_Buzzer(Actor.Spawn("UBQS_Buzzer", owner.pos));
	if (!q) return;
	q.feels = owner;
	q.A_QuakeEx(left ? int(k) : 0, left ? 0 : int(k), 0, clamp(tics, 1, 16), 0, 8, "", QF_SCALEDOWN);
}
```

SHIELD had been using Route B all along (`SSQ_FX.Buzz`, tremor radius 4), and SABER's commit says
so: *"The sibling shield mod has been using it all along."* So the actual history is: one mod
discovered the better channel, the other believed a comment in its own file, and two of its best
feedback events shipped off by default for it.

**The two operational notes are worth carrying:**

```
// strength is 0..3 before ubqs_haptic_power. Never fire this in the same tic as Buzz() on the same
// hand: the Quest drops a haptic request while one is running, so a short weak pulse eats a long one.
```
and `vr_quake_haptic_level > 0` is required — it defaults to 0.8, but a player who zeroed it gets
nothing and will report the mod as broken.

Note also that `A_QuakeEx` takes an **int** intensity, which is why SABER casts (`int(k)`) and why
its strengths are 0.9 and 1.2 — the cast truncates, so 0.9 × a power of 1.0 is **0**, i.e. no cut
rumble at default power. SHIELD passes `strength` as a double into the same int parameter
(`SSQ_FX.Buzz(mo, 0.7, 5)`), which truncates to 0 likewise. **I could not confirm how the ZScript
binding coerces a double into that parameter**, and the mods' own strengths (0.3, 0.55, 0.6, 0.7,
0.8, 0.9, 1.0, 1.2) only straddle 1.0 — so either the cast is doing something kinder than truncation
or several of these pulses are silent. This is the single most worthwhile thing to measure next in
either mod.

Handedness is `userinfo` in both, deliberately:
```
// WHICH CONTROLLER. ubqs_lefty is a user cvar, so it is userinfo: every machine reads the same
// owner's answer and the thinker spawned is identical everywhere. -1 falls back to the engine's
// own scheme, read in the one place this mod has ever read it.
```

#### 6.2 Sound placement — SABER's answer is in a different class

SABER's last four commits are all about one realisation:

```
// A sound played on the player is a sound at the player's feet. In a headset the whoosh of a
// metre-long blade arrived from inside your chest, dead centre, the same whatever you did with your
// arm -- which is the one place a lightsaber never is.
```

So every saber sound moved onto a **silent emitter actor carried with the thing that makes it**:
`UBQS_Hummer` at the blade's midpoint for the hum, `UBQS_Swish` on the tip for the swing,
`UBQS_Ping` at the contact point for clashes, cuts and scorches:

```
Actor HumSrc(bool make, Vector3 mid)
{
	if (!owner) return null;
	if (UInt("ubqs_swing_spatial", 1) == 0) return owner;
	if (!hummer)
	{
		hummer = UBQS_Hummer(Actor.Spawn('UBQS_Hummer', mid));
		if (hummer) { hummer.ear = owner; hummer.at = mid; }
	}
	return hummer ? Actor(hummer) : owner;
}
```
```
if (hummer) { hummer.at = hand + dir * (len * 0.5); hummer.base = hpitch; }
else if (owner) owner.A_SoundPitch(ch, hpitch);
```

And then the Doppler, which is the part no other mod here has:

```
// AND IT BENDS. Each tic the emitter measures how fast the tip is closing on your
// head along the line between them -- the radial component, the only part of a
// velocity that shifts a pitch -- and rides A_SoundPitch with it: rising while the
// blade comes at you, falling as it leaves. On a blade swung at arm's length that
// is a real interval, not a subtlety; a fast pass is about a semitone each way.
```

Four engineering details make it work and are the transferable part:

- **One emitter, reused** — *"so a flurry of swings does not leave a dozen of them chasing the tip."*
- **`life = 24` tics, longer than any sample, with the tic loop ending it** — so the emitter outlives
  the sound rather than cutting it.
- **Torn down on every exit** — *"when the blade douses, when the saber leaves the hand and when the
  item is destroyed: nothing else would clean it up, and a hum left running in mid air is a bug you
  can hear from across the map."*
- **One pair of helpers owns start/volume/pitch/stop** — *"so the blade route and the old
  on-the-player route cannot drift apart."* With `ubqs_swing_spatial 0` restoring the old behaviour
  exactly.

The contact sounds are the cheapest win of the lot, because the position was already computed:

```
// A clash is at the crossing point, a cut is at the body, a scorch is on the wall -- and all three
// are a position this code had already worked out to put its sparks at, then threw away to play the
// sound at your feet.
```

**If you take one thing from this document into a new melee system, take this.** It is a handful of
lines, the position is already in hand, and in a headset it is most of the difference between a
weapon and a sound effect.

Two traps recorded alongside it, both ZScript-level and both fatal:

```
// (And the helper is called At, not Play: `play` is a scope keyword in ZScript and
// a method of that name is a fatal "Unexpected 'play'" at every call site.)
```
```
// 0 MUST BE HANDLED HERE, not just passed through: the engine refuses a zero-volume sound outright
// (SoundEngine::StartSound: `volume <= 0` returns NULL), so starting the loop at 0 and setting
// humming = true anyway would leave no channel for the per-tic re-assert below to act on -- silence
// that the slider could never undo until the blade was doused.
```

And the pattern that makes a live volume slider work at all:

```
// THE LINE THAT ACTUALLY WINS: this re-assert runs every tic, so a hum volume set only where the loop
// starts would appear to work for one tic and then silently revert. Both sites read HumVol().
```

SHIELD and BD22 both play on the pawn (`mo.A_StartSound(...)`) throughout. For SHIELD's plate that
is nearly right (the plate is 5 units from the hand, the hand is near the body); for BD22's axe edge
0.3 m out and swinging it is the exact case SABER's commits describe.

#### 6.3 Visual tells

**SABER's trail is deliberately keyed 2 units below the cut threshold**, off the same cvar, so the
tell cannot drift from the thing it tells you about:

```
// The TRAIL stays 2 units below it, off the same cvar, so the deliberate "you see the arc just
// before it bites" tell cannot drift apart from the cut.
...
if (speed >= cut - 2.0 && lastOk && UInt("ubqs_trail", 1)) Trail(hand, tip, speed, cut);
```

and the trail's density ramps from `cut`, not from a literal, with the identity property stated:
*"at the shipped cap of 2 this is bit-identical to the old `clamp(int(speed / 8.0), 1, 2)` at every
speed."* That is the right way to make a refactor reviewable.

**SHIELD's perfect block announces itself three ways at once**, which is the right answer to "how
does the player know":

```
// HOW YOU KNOW IT WAS PERFECT, without reading a message: it rings a tone higher,
// the hand buzz is twice as long, and the plate blooms white at the contact point.
// A normal block is a spark and a thud; this is unmistakable.
```
```
mo.A_StartSound("ssq/block", CHAN_5, 0, perfect ? 1.0 : 0.8, ATTN_NORM, perfect ? 1.35 : 1.0);
if (held != 0) SSQ_FX.Buzz(mo, perfect ? 1.0 : 0.7, perfect ? 12 : 5);
```

SABER does the same off its continuous `pg`:
```
UBQS_Ping.At(owner, at, UBQS_Font.Clash(), UF("ubqs_vol_clash", 1.0), pg > 1.01 ? 1.25 : 1.0);
Rumble(0.8 * pg, pg > 1.01 ? 5 : 3);
...
UBQS_Data.Sparks(at, back, RgbNow(), int(10 * pg));
FlashAt(at, 64 * pg, 4);
```

**Both learned that FX need to be scaled to be seen in a headset**, and both reported the same
failure in the same shape. SABER: a 96-pixel sprite at scale 0.12 is nine pixels, invisible beside
the blade's own glow. SHIELD: *"The marks were 1.6 / 1.1 / 0.8 scale at 60% alpha over a monster's
head, which in a headset at ten metres is nothing"* — and then the *opposite* error one commit
later, when the mark became a sprite and kept the model's scales: *"which drew the saber's own icon
about eight times larger than the saber draws it"*. The fix is the right one:

```
// the size the art was drawn for: it is a 48x48 picture, and the saber draws it at
// 0.35. The rank multipliers in LocksTick are around THIS, not around 1.
Scale 0.35;
```

**Both deduplicate per-tic feedback**, for the same reason — a shotgun blast is many pellets in one
tic:

```
// SABER: `loud` is false for the second and later pellets of one shotgun blast in a tic.
BlockFx(at, -dir, sh, damage, level.maptime != lastBlockTic);
lastBlockTic = level.maptime;
```
```
// SHIELD
int now = Level.maptime;
if (now != lastBlockTic)
{
	lastBlockTic = now;
	...
```

And SABER buzzes **once per swing, not once per victim**:
```
// THE BLADE GOING THROUGH A BODY is the single event a VR player most expects to feel, and it did
// nothing [...] ONCE per swing, not once per victim -- a sweep through four imps is one pulse a
// controller could tell apart
if (buzzc && cutAny) Rumble(0.9, 4);
```

#### 6.4 Verdict — feedback

**Best answer today: SABER, by a distance, on sound; SABER on the haptic channel; SHIELD on the
"how does the player know it was a good block" question.**

**A fourth implementation should:**

1. Use `A_QuakeEx` with `damrad 0` for haptics, on a small tremor radius so it is per-player in
   co-op. Carry a strength and a length. The `LineAttack`/`SpawnPlayerMissile` tricks give you
   neither and buzz everybody.
2. Verify what the int intensity parameter does with a sub-1.0 strength before relying on it.
3. Never fire two haptics on the same hand in the same tic; the device drops the second.
4. Make handedness `userinfo` so every machine agrees.
5. Put every sound on a **carried emitter at the thing that makes it** — the blade midpoint, the
   tip, the contact point. The contact point is already computed for your sparks.
6. One emitter per channel, reused; `life` longer than any sample; torn down on douse, on leaving the
   hand, and in `OnDestroy`.
7. Route start/volume/pitch/stop through one pair of helpers, re-assert volume every tic, and handle
   0 explicitly — the engine refuses a zero-volume sound and you lose the channel.
8. Key the visual tell off the **same cvar** as the gameplay threshold, offset by a constant.
9. Deduplicate feedback per tic (shotgun pellets) and per swing (a sweep through four monsters is one
   pulse).
10. Scale FX to the victim and check them at ten metres in a headset. Both mods shipped invisible FX
    and one then shipped a comically oversized one.

---

### 7. Netplay

This is the one technique where the three give genuinely different answers, and **the gap is wider
here than anywhere else in this document.**

#### 7.1 The underlying engine fact

Everything turns on one verified fact. `VRMode::SetUp()` in
`670d19eba6^:src/common/rendering/hwrenderer/data/hw_vrmodes.cpp`:

```
void VRMode::SetUp() const
{
	player_t* player = r_viewpoint.camera ? r_viewpoint.camera->player : nullptr;
	if (player && player->mo)
	{
		player->PlayInVR = IsVR();
		player->mo->OverrideAttackPosDir = !puristmode && (IsVR() || vr_override_weap_pos);
		player->mo->AttackDir = MapAttackDir;
		player->mo->OffhandDir = MapOffhandDir;
		...
```

**`r_viewpoint.camera->player` and nobody else.** So `AttackPos`, `OffhandPos`, the six hand angles,
`PlayInVR` and `OverrideAttackPosDir` are written by the **renderer** for the **local camera's
player only**, and they are never replicated. On your machine, every other player's hand pose is
whatever it was last set to — which, for a player who was never the local camera, is zero.

A melee decision read off those fields is therefore **a local-only decision made inside the
playsim**, which is the definition of a desync.

#### 7.2 What each one does about it

**BD22: nothing, and it does not notice.** `RSVR_MeleeHandler.WorldTick` loops all players and calls
`hands[i].Tick(mo)` for each, and `Frame()` gates on:

```
static bool InVR(PlayerPawn mo)
{
	return mo && mo.player && mo.player.PlayInVR && mo.OverrideAttackPosDir;
}
```

For a remote player on your machine both of those are false, so the swing simply never happens
locally. Meanwhile on *their* machine it does, and it spawns `AxeAttack` actors that damage and gib.
**The playsim diverges on the first axe swing in a co-op game.** BD22 has one piece of genuine net
hygiene, and it is in exactly the place you would least expect:

```
// rsvr_axe_keys: the weapon hand's grip arrives as a KEY, not a usercmd button, so its codes
// become a synchronised event the same way rsr_grip_keys does in vreload.zs. Every machine
// sees the same down and up, so readying a throw is never a local-only decision.
```
```
if (down == keyIsDown)
	return false;			// key repeat: only the edges are worth a net event
keyIsDown = down;
EventHandler.SendNetworkEvent("rsvr_axe_grip", down ? 1 : 0, e.KeyScan);
return false;				// never eaten: the grip keeps whatever else it does
```

So BD22 correctly identified that an `InputEvent` key is UI-side and local, and correctly promoted
it to a net event with edge-only sending — and then read the hand *pose* straight out of a
renderer-written field three functions later. The input is synced; the geometry it feeds is not.

**SABER: it noticed, and its fix stops the worst symptom without fixing the sim.**

```
// THE HAND POSE IS LOCAL-ONLY, so a saber whose owner's pose this machine does not have is not a
// blade. VRMode::SetUp and OpenXRDeviceMode::SetUp both write r_viewpoint.camera->player and nobody
// else; for every OTHER player here AttackPos is zero, HandPosOf falls back to the chest and
// BladeDir() is a fixed world direction. Left to run, that phantom blade deflects, burns, clashes
// and blocks on THIS machine and not on theirs. The same four reads WorldUp() already makes.
bool HandLive() { return owner && (bOffhandWeapon ? owner.OffhandPos : owner.AttackPos) != (0, 0, 0); }
bool Lit() { return !thrown && UBQS_Data.Ext(frameNow) > 0.5 && InHand() && HandLive(); }
```

That is a genuine and important fix: without it, a co-op partner carries a blade at a *fixed world
heading* out of their chest that cuts, deflects, burns, clashes and blocks **on your machine and
nowhere else**. But what it buys is *silence* rather than *agreement* — the remote player's saber is
simply not lit locally, so their cuts happen on their machine and not on yours. Same divergence as
BD22, with the phantom removed.

SABER's other net hygiene is real and worth copying:

```
// EVERY random[]/frandom[] CALL BELOW STAYS IN THE SHIPPED ORDER, drawn before the guard that might
// skip the thing it feeds. The gating cvars are userinfo read off the owner, so every machine already
// computes the same answer, but keeping the stream byte-identical costs nothing and removes the
// question entirely.
```
```
// THE TRAINING REMOTE drew its aim scatter and shot timing from the same named RNG as every
// cosmetic spark, and those spawn off the local-only pose. Its own stream now.
```

The second is the deep one: **a cosmetic effect that spawns off a local-only pose and draws from a
shared named RNG stream desynchronises the stream for everything else that uses it.** Giving the
local-only consumer its own named stream is the fix, and the whole class of bug is easy to miss.

**SHIELD: it syncs the pose, and it is the only one of the three whose melee decisions are
deterministic.**

```
// Stock QuestZDoom writes OffhandPos / OffhandAngle / OffhandPitch / OffhandRoll on the
// console player's pawn only, from the renderer, and never sends them. Anything the sim
// decides from the hand (a block, a throw, a bash, a catch) has to come from a pose every
// machine has, or the game desyncs the moment somebody raises a shield.
//
// So: the console player's UiTick reads its own fields and sends "ssq-pose" (three ints) once
// a tic while the pose matters; NetworkProcess, which runs on every peer, unpacks it into the
// per-player history below. The sim reads ONLY the history -- on the sender's machine too.
```

That last clause is the whole architecture in nine words. The sender does **not** shortcut to its own
live fields; it reads the same delayed history every peer reads, so there is nothing to diverge.

The pose is packed into the three ints a `ConsoleEvent` carries — 11 bits each for X and Y in
quarter-units, 10 for Z, then 10+10+10+1 for yaw, pitch, roll and the grip bit:

```
// arg1: x, y relative to the pawn in quarter units (11 bits signed each), z in quarter
//       units (10 bits signed)
// arg2: yaw of F (10 bits, 0..360), pitch of F (10 bits, -90..90), roll of L about F
//       (10 bits, -180..180), grip (1 bit)
```
```
static clearscope int PackPos(Vector3 rel)
{
	int x = ClampI(int(round(rel.x * 4)), -1024, 1023) & 0x7FF;
	int y = ClampI(int(round(rel.y * 4)), -1024, 1023) & 0x7FF;
	int z = ClampI(int(round(rel.z * 4)), -512, 511) & 0x3FF;
	return x | (y << 11) | (z << 22);
}
```

So position quantises to 0.25 units (≈7 mm) over ±256 units, yaw/roll to ~0.35°, pitch to ~0.18°.
That is comfortably finer than the geometry it feeds — a 10-unit-radius plate — and it is a good
default budget for any hand-pose sync.

Three more details that make it work:

- **It sends only when the pose matters** — `bool want = st != SS_STOWED || grip || nearMount;` — so a stowed shield costs nothing.
- **It reconstructs the full basis from three angles** rather than sending three vectors, because three `Vector3`s do not fit and, separately, because the VM will not return them (§8).
- **The roll is measured against the horizontal left so the receiver rebuilds the same axes**:
  ```
  static clearscope double RollOf(Vector3 F, Vector3 Lreal)
  {
      double yaw = atan2(F.y, F.x);
      Vector3 L0 = (cos(yaw + 90), sin(yaw + 90), 0);
      Vector3 U0 = (F.y * L0.z - F.z * L0.y, F.z * L0.x - F.x * L0.z, F.x * L0.y - F.y * L0.x);
      return atan2(Lreal dot U0, Lreal dot L0);
  }
  ```
- **Handedness rides along** as a bit, because the lateral sign of the plate depends on it and the receiver must not read its own `vr_control_scheme`.
- **`SendPose` lives in `WorldTick`, not `UiTick`**, with the reason given: *"OffhandDir is play-scope, so this cannot live in UiTick."*

**What it costs.** One net event per tic per player holding a shield, which is 35/s of three ints —
small, but not nothing, and it grows linearly with players. And the decision is made from a pose
that is as old as the round trip: `Fresh(now)` allows `now - stamp[0] <= 3`, so three tics (86 ms) of
tolerance before the shield simply stops working. In a laggy game the block, the bash and the throw
all get the latency, on the sender's machine too — **deliberately**, since that is what makes it
deterministic. That is the honest trade, and it is the right one for a co-op target.

#### 7.3 Everything else

**Both SABER and SHIELD route *input* decisions through replicated channels.** SABER reads
`owner.player.cmd.buttons` (in the replicated `ticcmd`) with per-hand press latches, and routes its
keys through `netevent`:

```
// KEYS, NOT CVARS, so nothing here can go dead in his ini -- and NO KEYCONF alias either: a MENUDEF
// Control row binds whatever command string it is given, and `netevent` is a plain CCMD [...]
```

SHIELD reads `p.cmd.buttons` exclusively, which is also what makes it immune to the two-hand grab:

```
// The two-hand grab needs no handling here at all. It never nulls OffhandWeapon and never
// touches the hand pose; it only disables the off-hand weapon's ready/fire/button
// machinery [...] and the shield does not use any of it. Every input it has is read
// straight off p.cmd.buttons, which is in the replicated ticcmd and which the grab cannot gate.
```

**Both make gating cvars `user` (userinfo) deliberately**, so every machine computes the same answer
from the same player's settings rather than from its own:

```
// It is a USER cvar, so it is userinfo: every machine reads the same owner's answer and netplay
// cannot diverge.
```

That is the single cheapest netplay discipline in either mod and it applies to every cvar that gates
a sim decision.

One residual worth flagging in SHIELD: `SSQ_FX.Sparks` and `SSQ_Disc.SpecialMissileHit` use
**unnamed** `random()` / `frandom()`, i.e. the default RNG stream, from paths gated on
`UVI(mo.player, 'ssq_feedback', 1)`. Since `ssq_feedback` is `user`/userinfo the gate is replicated
and every machine takes the same branch, so this is consistent today — but it is one cvar-scope
change away from being a desync, and SABER's own `[UBQSFx]` named-stream discipline is the safer
pattern.

#### 7.4 Verdict — netplay

**Best answer today: SHIELD, uncontested, and it is not close.** SHIELD is the only one of the three
in which a melee decision made from the hand is the same decision on every machine. SABER knows the
problem exists and removes the phantom; BD22 does not appear to know.

**A fourth implementation should:**

1. Treat `AttackPos`, `OffhandPos`, the six hand angles, `PlayInVR` and `OverrideAttackPosDir` as
   **renderer output for the local camera's player only**. Verified above. Never make a playsim
   decision from them directly.
2. Send the pose as a `netevent` once a tic while it matters, and **read only the synced history,
   including on the sender's machine**. That one rule is what makes it deterministic.
3. Pack it: quarter-units for position over ±256, three 10-bit angles, plus a handedness bit and
   whatever button bits you need. 0.25 units is 7 mm and is finer than any melee geometry.
4. Reconstruct the basis from the angles on the receiver; do not try to send vectors.
5. Stamp every sample and have a freshness window, so a dropped connection fails safe rather than
   acting on a stale pose.
6. Accept the latency deliberately — the throw, the block and the bash all get the round trip, and
   that is the price of agreement.
7. Make every cvar that gates a sim decision `user` (userinfo), not `server` and not a local cvar.
8. Give every local-only or cosmetic consumer its **own named RNG stream**, and keep the
   `random[]`/`frandom[]` call order stable across guard changes.
9. Promote any `InputEvent`-sourced input to a net event, edges only, and never eat the key (BD22
   gets this exactly right).

---

### 8. The traps, collected

Everything below is from the codebases' own comments or commit logs, i.e. it was paid for. Engine
claims are marked **[verified]** where I checked them against `670d19eba6^`, **[mod's claim]** where
I did not.

#### ZScript and the VM

| Trap | Source |
|---|---|
| **Multiple `Vector3` returns and `out Vector3` params are dropped** — silently zero, or a JIT abort. SHIELD's hand orientation arrived as all zeros because of this; it now builds axes in place and writes members. **[mod's claim, found 2026-10-22]** | SHIELD `pose.zs`, HANDOFF §4 |
| **`Array<Vector3>` is not allowed.** Use three `Array<double>`. | SHIELD `disc.zs`, BD22 `melee.zs` |
| **`abs(x) < 1e-6` did not stop a true zero divide.** Use `x == 0 \|\| x != x \|\| (x < 1e-6 && x > -1e-6)`. **[mod's claim]** | SHIELD HANDOFF §4, `guard.zs` |
| **`play` is a scope keyword**; a method named `Play()` is a fatal "Unexpected 'play'" at every call site. Hence `UBQS_Ping.At`, not `.Play`. | SABER `25c7358` |
| **ZScript will not dereference `.z` off a parenthesised expression** — hence `Vector3 hStep = h0 - h1;` on its own line. | SABER `DoEffect` |
| **`LineAttack` takes 11 arguments**, not 12: `(angle, distance, pitch, damage, type, puff, flags, victim, offsetz, forward, side)`. **[verified]** — `actor.zs:804`. | SHIELD HANDOFF §4 |
| **`LAF_ABSPOSITION` makes the three offsets literal coordinates**: `tempos = DVector3(offsetforward, offsetside, sz)`. Passing `(P.z, 0, 0)` started a trace at the map origin. **[verified]** — `p_map.cpp:4875`. | SHIELD `8b8c963` |
| **`A_QuakeEx` intensity is an int.** X = left controller, Y = right. **[verified]** — `r_utility.cpp:1085`. Several shipped sub-1.0 strengths may therefore be silent (unresolved). | SABER `Rumble`, SHIELD `SSQ_FX.Buzz` |
| **The engine refuses a zero-volume sound** (`SoundEngine::StartSound: volume <= 0` returns NULL), so starting a loop at 0 leaves no channel for a later re-assert to act on. **[mod's claim]** | SABER `DoEffect` |
| **A sound's definition is fixed at load and ZScript cannot build a `Sound` from a `String`.** A run-time font switch is a `switch` over literal names. | SABER `1dd72a4` |
| **`MODELDEF` only knows a sprite that some *state* names**; a sprite written to a layer at run time must appear in a dummy state or startup dies with "Unknown sprite" — in a dialog, not on stdout. **[mod's claim]** | SHIELD HANDOFF §4 |
| **`ModifyBobLayer3D` returns a translation and a rotation. There is no scale.** **[verified]** — `weapons.zs:46`. Hence SHIELD's five baked meshes and SABER's baked blade lengths. | SHIELD §3, `65300bd` |
| **`PSprite.NoDraw` does not exist on this engine; `psp.alpha` does.** Using the former killed a build. | SABER `HideWhileAway` |
| **`+WEAPON.NOAUTOFIRE` is not usable here** — this engine's `CheckWeaponFire` lost the latch that would make it mean anything, so a press latch has to live in the mod, per hand. | SABER `TickFireLatch` |
| **Assigning a null `PendingWeapon` is not `WP_NOCHANGE`** and stalls the engine's weapon handling. | SABER `WorldTick` |

#### The engine's VR layer

| Trap | Source |
|---|---|
| **The hand's Z is divided by `pixelstretch`; X and Y are not.** So vertical hand travel arrives at 1/1.2 ≈ 0.833 of real, and you multiply by 1.2 to recover it. **[verified]** — `hw_openxrdevice.cpp:595/608`. | all three |
| **The hands are positioned relative to `CenterEyePos.Z`**, so view bob rides into the hand's vertical. **[verified]** — same lines. | SABER `BodyAt` |
| **`VRMode::SetUp` writes the hand fields for `r_viewpoint.camera->player` and nobody else**, and they are not replicated. **[verified]** — `hw_vrmodes.cpp:432-446`. | §7 |
| **`e.AttackPos` is NOT filled on `WorldHitscanPreFired`** (only on `WorldHitscanFired`). Compute the muzzle yourself. **[verified]** — `events.cpp:1829-1840`. | §4.2 |
| **The muzzle is `Center() − Floorclip + AttackOffset() + sz`, `AttackOffset() == 8` for a non-player, and `LAF_OVERRIDEZ` replaces it with `Z()`.** **[verified]** — `p_map.cpp`, `actorinlines.h:92`. | SHIELD `2d10397` |
| **The first handler to return true from `WorldHitscanPreFired` short-circuits the rest**, and handler order is load order. **[verified]** — `events.cpp:713-717`. | my note, undocumented in both mods |
| **`AbsorbDamage` is skipped for `DMG_NO_ARMOR`, which the engine sets for any `+PIERCEARMOR` inflictor**; `ModifyDamage(passive)` is skipped only for `DMG_NO_PROTECT`. **[verified]** — `p_interaction.cpp:1151, 1231, 1363`. | my note |
| **`r_quakeintensity` defaults to 0 for VR, so the view does not shake; `vr_quake_haptic_level` (0.8) drives the haptic separately** and must be > 0. **[verified]** — `r_utility.cpp:116, 1085`. | SABER `15f6332` |
| **The engine's fire-weapon haptic is fixed at 150 ms / 0.8 and has no player check**, so in co-op every shot buzzes everyone. **[verified]** — `p_map.cpp`, `p_mobj.cpp`, `OpenXRDeviceMode::Vibrate`. | my note |
| **`LAF_ISOFFHAND` is `1 << 8` = 256.** **[verified]** — `p_local.h:330`. | SABER `Buzz` |
| **`P_PlayerThink` runs before `WorldTick` in a tic**, and `P_Ticker` has already done `ClearInterpolation` on every actor before the player thinks — so anything a player's own tick writes interpolates from the end-of-last-tic pose for free. **[mod's claim, p_tick.cpp:151-173]** | SHIELD HANDOFF §4, SABER `PoseTick` |
| **A guard surface inside the player's own collision box is unreachable by a missile** — the engine kills the missile at the box edge first. **[mod's claim, 2026-10-17 audit]** | SHIELD `guard.zs` |
| **`switchhand` is a console command, not a net command, and its argument is the *destination* hand** — so it moves the weapon in the *other* one. `+WEAPON.NOHANDSWITCH` blocks `switchhand 0` only. **[mod's claim, g_game.cpp:347-361 → player.zs:2488-2502]** | SHIELD `dc99a85` |
| **The two-hand grab never touches `OffhandWeapon` or the hand pose** — it only disables the off-hand weapon's ready/fire/button machinery and hides its psprite in C++. **[mod's claim]** | SHIELD §4 |
| **QZD's stock bind puts `+oh_altatk` behind a grip shift**, and that grip is what arms the two-hand grab — so on a Quest the off-hand alt alone was unreachable in practice. | SHIELD `c5ae2a3` |
| **QZD has no usercmd bit for the *weapon* hand's grip.** It arrives as a key, which is UI-side. | BD22 `melee.zs`, CVARINFO.rsvr |

#### Design traps — "this was tried and it failed because…"

| Tried | Failed because | Who |
|---|---|---|
| World hand motion minus the pawn's position | a turn swings the arm round the body with no translation to subtract, and the controller and the pawn land a tic apart | BD22 1.4→1.5 and 1.11; SABER still |
| Speed alone as the swing test | raising the axe *was* a hit — it landed before the player had swung, and read as wild oversensitivity | BD22 2026-10-13 |
| `rsvr_melee_up` at 0.55 (≈33° up still cuts) | still hit on the upswing; tightened to 0.15, then replaced by a *must point down* test at 0.25 | BD22 2026-10-16, 1.11 |
| One tic of tip travel as the cut test | a thrust moves the tip along its own axis, the slowest motion it makes; a stab, a draw-cut and a lean all did nothing, for as long as you held them | SABER `e825d10` |
| One tic of plate travel as the bash test | with the body's motion correctly removed, a lean or a walk-into-a-doorway did nothing at all | SHIELD `2d10397` |
| An 8-deg/tic snap-turn flush | a smooth-turn player above 280°/s trips it every tic, killing swings, throws and glow prediction for the whole turn — and below it, the turn is not subtracted at all | SABER, by its own comment |
| A cut iterator with no reachability test | a swing near a thin wall or a closed door killed what was behind it | SABER `e825d10`; SHIELD `2d10397` (bash) |
| `DMG_THRUSTLESS` on every cut path | a lit blade swept through four imps left four imps standing exactly where they were | SABER `e825d10` |
| `Thrust()` for the shove | it adds velocity outright and ignores `Mass`, so a Cyberdemon left the plate as fast as an imp | SHIELD `2d10397` |
| A shove with no lift | a body slides a few units and walks straight back in; off the floor, a Doom monster cannot change direction | SABER, SHIELD |
| A 200° guard arc off the body's yaw | "the most abusable number in the mod, with no way to tighten it" — fixed by requiring a blade in the attack's own path *inside* the arc | SABER `e825d10` |
| A 12-tic immunity window after any fast swing | the twirl sets the window every single tic, so a twirling player never leaves it | SABER `70da633` |
| A dice-roll block (70% inside 150°) | swamps any amount of precision — a geometric guard layered on top would be unfeelable, so it had to *replace* it | SABER `70da633` |
| `ModifyDamage`/`AbsorbDamage` as the *only* guard | runs after the shot has already hit, against one guessed line to your centre: blocks shots that would have missed, misses shots that were really coming, cannot stop anything not aimed at your centre | SABER `70da633`; SHIELD `a7663d1` |
| Guessing the shooter's muzzle at `height * 0.75` | 6 units high on an imp, 14 on a Cyberdemon — against a 29 cm plate at 20 m, most of the guard | SHIELD `2d10397`; **SABER still off by 8** |
| Testing a moving plate at its tic-start position | the shield is checked stale while the shot is checked live — wrong exactly when you are parrying | SHIELD `33e5d44` |
| Steering that renormalises to the launch speed | cannot catch a target the swing did not point at; it loops instead, and a gentle throw stays gentle all the way round the room | SHIELD `dc99a85`, `33e5d44` |
| A homing return that divides by the distance | the object lands exactly on the hand point, so `d` really reaches 0 on a repeated pose, and NaN poisons the position permanently — the hilt gone for the level | SABER `3f5fdec` |
| A catch radius with no age gate | the object is spawned at the hand, so it is already inside its own catch radius: catch, return, re-throw, flicker | SABER |
| …then moving that gate above the movement | stopped the object flying at all for its first 17 tics, leaving it hanging in the air | SABER |
| A bounce handler that does nothing on the way home | a crumb path through a corner leaves the disc grinding on a wall until the age cap drops it on the floor | SHIELD `dc99a85` |
| Demanding a forward snap on top of a painted route | asked for a gesture that changed nothing (the route ignores the arm) and threw the marks away when it was missed | SABER `e825d10` |
| Rebuilding the lock list from a cone every tic | a target falls out the moment the hand drifts, and the visit order churns | SHIELD `dc99a85` |
| Taking every target in the cone at once | lock-on becomes a scan: point at a crowd, five rings. One every few tics makes it aiming | SHIELD `a7663d1` |
| Playing sounds on the player | a metre-long blade's whoosh arrives from inside your chest, dead centre, the same whatever you do with your arm | SABER `9020291`, `76e86d5`, `25c7358` |
| Setting a looping sound's volume only where the loop starts | appears to work for one tic and then silently reverts | SABER |
| An entry flash at scale 0.12 on a 96-pixel sprite | nine pixels, invisible beside the blade's own glow | SABER `e825d10` |
| A spark burst in the `else` branch for `bNoBlood` actors | no monster is `bNoBlood`, so the shipped effect had never once fired | SABER `e825d10` |
| Keeping a model mark's scales after swapping in a sprite | drew a 48×48 icon about eight times larger than intended | SHIELD `e60b82a` |
| A cosmetic effect spawned off a local-only pose, drawing from a shared named RNG | desynchronises the stream for everything else that uses it | SABER `3f5fdec` |
| A re-raise latch kept per level | `PlayerPawn` nulls both hands on respawn and re-raises only the main one, so one death leaves you one-handed for the map | SABER `3f5fdec` |
| An ignition counter ticked only inside `ReadyLoop` | holding fire during the ignition freezes the blade on a 3-unit stub: no hum, no light, no cut, no block, for the whole hold | SABER `3f5fdec` |
| A press latch on fire but not on alt | a held alt strobed the second blade on and off with both its sounds every 8 tics | SABER `3f5fdec` |
| A dead lock still counting against the lock cap | killing what you painted mid-hold stopped you painting anything else | SABER `3f5fdec` |
| `ClearAll(null)` reading null as a wildcard | wiped every player's marks | SABER `3f5fdec` |
| Pinning `bOffhandWeapon` without fixing the slots | the engine leaves the main weapon in the off slot and you get two sabers reading one controller — "a mace windu" | SABER `3f5fdec`, `WorldTick` |
| Writing only `PendingWeapon` to claim the off hand | event handlers tick in load order, so the last writer in the tic wins and correctness depended on pk3 order | SHIELD `dbef2ec` |
| Reading eviction from the off hand as an error to undo | the weapon pack re-gives its hands weapon to an empty off hand every tic, so "yield" took the shield out of the hand immediately | SHIELD `1b99ce4` |
| Pinning both hands every tic forever | fought the engine's own `switchhand`, whose menu says it works | SHIELD `dc99a85` |
| Drawing nothing when stowed but still blocking | "an invisible thing that stops bullets is not a shield, it is a secret" | SHIELD `fdc2607` |
| Calling the motion predictor twice in a tic (hilt and emitter) | the second call pushes the first's extrapolation in as a fresh sample, so the measured velocity and the applied lead both compound and the object runs away on fast swings | SABER `PoseTick` |
| Reading the predictor's tunables at their use site | read a call too late, and never at all on the paths that return early — the hilt then steps at 35 Hz with no lead | SABER `3f5fdec` |
| Importing twelve sound fonts wholesale and hanging a "boot" on the draw | the fonts ship character dialogue and music stings beside their sounds, so the saber recited film lines when you turned it on | SABER `7a8bae7` |
| Two cvar defaults for the same name in one file | the declared value won silently and the second was dead | SHIELD `2d10397` (`ssq_sweep_pad`) |
| A feature body after an unconditional `return;` | survived a grep for readers; its cvars looked live and were not | SABER `3f5fdec` (`FlickCheck`) |
| A feature with no bindable key on a clean profile | the staff twirl ships at Fast with ten md3 frames and forty baked sprite sets and could never happen | SABER `e825d10` |

---

### 9. What a fourth implementation should be

Collapsing the nine verdicts into one shape.

**The spine.** A per-player `Pose` object carrying 8 samples of `{point, pawn pos, pawn yaw, frame
index, tic stamp}`, fed by a once-a-tic `netevent` from the console player and read **only** from
the history, by everyone, including the sender. Three packed ints: position in quarter-units
relative to the pawn, three 10-bit angles, a handedness bit, the button bits. Everything downstream
reads this and nothing reads `AttackPos`/`OffhandPos` directly. This is SHIELD's `SSQ_Pose`, and it
is the single highest-leverage thing in the three codebases: it makes the mod net-correct, it makes
it dt-correct, and it deletes the snap-turn and teleport special cases rather than tuning them.

**The gesture layer.** Difference in the body frame per sample, divide by the stamp gap, re-rotate
once into the current facing, correct the vertical by `pixelstretch` in the magnitude only. Gate on
velocity **and** direction (project onto the axis the attack is about) **and** displacement over the
window. Integrate as well as differentiate, self-healing on a tic gap, and keep the integral out of
anything that grants immunity or plays a whoosh. Express the threshold as an angular rate at the
wrist times the measured lever arm. Where the gesture is an orientation change, use the angle and
skip all of this.

**The geometry layer.** Derive the shape from the mesh, write the transform reduction into the file,
read the player's live fit cvars, and derive the lateral sign from handedness. Sweep the offensive
volume with the step count from actual travel and the along-blade density from blade length,
iterating from the tip inward. Sweep the defensive surface along its own travel too. Thick slabs,
not planes. Contact points on the surface. NaN guards.

**The resolution layer.** Blockmap iterator from the swept bound, cvars above it, filters cheapest
first, **hand-to-contact `LineTrace`** last. Collect then strike. Cooldown scoped to the swing with
a per-victim floor under it. If a host mod exists, spawn its attack actor at the contact point and
height and let it own the result; otherwise pass the pawn as inflictor and roll a mass-scaled shove
along the arc the blade travelled, with a lift, halved for players.

**The defence layer.** All three of: `WorldHitscanPreFired` with the muzzle computed by the engine's
own formula; a per-tic sweep of threats against the surface with a 3-tic lead, turning rather than
replacing; and `ModifyDamage(passive)` — not `AbsorbDamage` — classifying the attack into
missile/hitscan/melee/blast with a different test segment for each. One `DeflectDir` for everything
that turns anything. A continuous, unlatched parry gain from the surface's step into the incoming
direction, driving every feedback channel. No cones, no dice rolls, no timed immunity windows.

**The feedback layer.** `A_QuakeEx` with `damrad 0` and a small tremor radius, with a strength and a
length, handedness from userinfo, never two on one hand in one tic. Every sound on a carried
emitter at the thing that makes it, one per channel, reused, torn down on every exit, volume
re-asserted every tic with 0 handled explicitly. Visual tells keyed off the same cvar as the
gameplay threshold. Dedupe per tic and per swing. Scale to the victim and look at it at ten metres.

**And the discipline that produced the best parts of all three:** the owner's complaint goes into
the code as a dated quote next to the fix, the failed approach stays described, and the cvar comment
says what the number means and what turning it off costs. Three mods' worth of that is why this
document could be written at all.

---

### 10. What I could not establish

- **Whether `A_QuakeEx`'s int intensity parameter truncates a sub-1.0 ZScript double.** If it does,
  SABER's cut rumble (0.9) and SHIELD's grind (0.3), sweep (0.55) and block (0.6/0.7) buzzes are
  silent at default power. This is cheap to measure and would change the feedback section's verdict.
- **The three NaN/zero-divide claims in SHIELD's handoff** (`abs(x) < 1e-6` insufficient; multiple
  `Vector3` returns zeroed; `Array<Vector3>` rejected) — I took these as the mods' findings. The
  `Array<Vector3>` one is corroborated by both mods independently working around it.
- **SHIELD's "the engine kills a missile at the player's box edge before a sweep sees the plate"**
  audit. I did not re-derive it. It is the stated reason the sweep must test a lead rather than a
  contact, so it matters.
- **Whether SABER's `CutLine` actually tunnels in play.** The geometry says a 24-unit/tic blade
  against a 20-radius victim with 3 units of pad should sometimes pass through; I did not run it.
- **SABER's `Ext(frameNow)` and `EmitAlong(sizeIdx)` numeric values**, so the lever-arm ratio in §1.3
  is an estimate from `FullLen() = 26.0 * Size` rather than a measurement.
- **Whether SHIELD's thresholds were retuned after the body-frame fix.** Its own `2d10397` commit says
  *"The defaults will want lowering after a headset pass: they were being met by the body, and the arm
  alone is a much smaller number"* — and `CVARINFO.txt` still shows `ssq_bash_speed = 70.0`,
  `ssq_perfect_speed = 40.0`, `ssq_throw_min = 20.0`. So either the pass happened and found them
  acceptable, or it has not happened. I cannot tell which from the repository.
- **SABER's muzzle offset**, flagged in §4.2, is as of `828c070`. The session editing that file may
  have fixed it since.

