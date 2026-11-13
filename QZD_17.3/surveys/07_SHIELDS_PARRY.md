# 07 — Shields and parry

**Engine:** `qzd17.3` of `github.com/emawind84/gzdoom` (QuestZDoom PC edition), read-only.
All `path:line` citations are repo-relative at that tag.
Ground truth assumed from `_BASELINE_FACTS.md` **as corrected 10-30** (both hands are readable;
see §2.3). Every correction relayed to this lane was re-verified against source before use; where
this report adds to the baseline, it says so.

---

## 1. Verdict

A hand-held VR shield is **BUILDABLE on this engine without a single engine change**, and more of
it is already shipped than you would expect: the Hexen Centaur is a working, timed, directional
parry (`wadsrc/static/zscript/actors/hexen/centaur.zs:36-38`, `actor.zs:964`), and the flag it
uses — `+SHIELDREFLECT` — is a hardcoded ±45° frontal cone keyed on the blocker's own yaw
(`src/playsim/p_mobj.cpp:3468-3472`). **Both** hands are fully readable — `AttackPos`/`AttackAngle`/
`AttackPitch`/`AttackRoll` for the weapon hand and `Offhand*` for the shield hand
(`wadsrc/static/zscript/actors/actor.zs:268-275`) — so sword-and-board with real blade-on-shield
geometry is on the table, not just a shield in isolation (§4.12).

The central problem is real but **smaller than it sounds**. `vr_vunits_per_meter` is 34
(`src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:94`), so one tic of hand travel at a
realistic blocking speed of 2 m/s is **1.94 map units — about 9% of the width of a 60 cm shield**.
And the plate does not judder: `Prev` is re-snapshotted for **every** actor at the top of every
tic, before any thinker runs (`src/p_tick.cpp:151-162` → `src/playsim/actorinlines.h:29-35`), so a
plate that *teleports* with `SetOrigin(pos, true)` still renders as a smooth tween. It only breaks
down at parry/bash speeds (5-8 m/s → 4.9-7.8 units, 24-38% of the shield's width per tic), where
the plate steps straight past a projectile because it sweeps nothing. That is the one place the
design must not rely on the plate.

**One structural decision has to be made before any code: do not host the shield as an offhand
`Weapon`.** Offhand grip with the hands inside 0.50 m sets `weaponStabilised`
(`src/rendering/gl/stereo3d/gl_openvr.cpp:1717-1728`) with no `bTwoHanded` test, and the offhand
weapon then fires nothing, reads no buttons and **renders nothing** — at exactly the pose a player
uses to brace a shield. A plate actor plus a plain psprite overlay sidesteps it entirely, because
the render suppression is keyed on the layer's *caller* being `OffhandWeapon`
(`hw_weapon.cpp:832`), and the plate has to be a world actor anyway. Full working in §2.4.

The hard limits are shape and sight. Actor collision is **not a cylinder** — it is an
axis-aligned box with a square footprint, `fabs(dx) < r+r && fabs(dy) < r+r` and a vertical
`[Z, Z+Height]` slab (`src/playsim/p_map.cpp:1417-1419`, `:1633-1643`,
`src/playsim/p_trace.cpp:695-696`), and **it cannot be rotated** — nothing in the collision path
reads `Angles.Yaw`. A flat, angled plate therefore has no native representation. The escape is
`CanCollideWith` (`src/playsim/p_map.cpp:1514` → `wadsrc/static/zscript/actors/actor.zs:562`),
a ZScript virtual called on *both* actors before any blocking decision, which lets you carve the
box down to any shape you can write — but it is **not** consulted on the hitscan path, so bullets
see the raw box. And `P_CheckSight` contains no actor iteration at all (`src/playsim/p_sight.cpp`,
967 lines, no blockmap/actor test), so **no shield ever blocks line-of-sight, splash damage or a
melee swing** — those three must be resolved in script on the player's own `DamageMobj`.

---

## 2. What already exists

### 2.1 A complete, shipped, timed directional parry

```
Pain:
    CENT G 6 A_Pain;
    CENT G 6 A_SetReflectiveInvulnerable;
    CENT EEE 15 A_CentaurDefend;
    CENT E 1 A_UnsetReflectiveInvulnerable;
```
`wadsrc/static/zscript/actors/hexen/centaur.zs:34-39`

- `A_SetReflectiveInvulnerable()` is one line of ZScript: `bInvulnerable = true; bReflective = true;`
  — `wadsrc/static/zscript/actors/actor.zs:964`. Both are plain scriptable bools, so a mod opens
  and closes the window whenever it likes.
- The Centaur's `+SHIELDREFLECT` (`centaur.zs:15`) is what makes it *directional*.
- The window above is 45 tics ≈ 1.29 s. A VR parry window is the same code with a smaller number.

### 2.2 Reflection, as a built-in, with four behaviours

`AActor::AdjustReflectionAngle` — `src/playsim/p_mobj.cpp:3463-3505`:

| Flag on the blocker | Behaviour | Line |
|---|---|---|
| `+REFLECTIVE` alone | required for any reflection at all | `p_map.cpp:3834` |
| `+SHIELDREFLECT` | **±45° frontal cone off `Angles.Yaw`; outside the cone the missile explodes** | `p_mobj.cpp:3468-3477` |
| `+DEFLECT` | always ±45°, no cone test | `p_mobj.cpp:3479-3486` |
| neither | random ±8° scatter | `p_mobj.cpp:3488-3490` |
| `+AIMREFLECT` (additive) | re-aims the missile at the shooter and re-speeds it | `p_mobj.cpp:3491-3502`, `p_map.cpp:3845-3850` |
| `+MIRRORREFLECT` | straight back along its own path at half speed | `p_map.cpp:3851-3855` |
| `+THRUREFLECT` on the blocker | missile passes through unchanged | `p_map.cpp:3838`, `p_map.cpp:1712` |
| `+DONTREFLECT` on the **missile** | opts that missile out | `p_mobj.cpp:3465` |
| `+NOSHIELDREFLECT` on the **missile** | opts out of shield cones only | `p_mobj.cpp:3471` |

The call site is `P_ReflectOffActor` (`src/playsim/p_map.cpp:3832-3879`), reached from
`P_XYMovement` at `src/playsim/p_mobj.cpp:2359` — **after** the missile has been blocked and
**before** `P_ExplodeMissile` at `:2395`. So reflection is simply "what happens instead of
exploding."

`P_ReflectOffActor` is **exposed to ZScript**: `src/scripting/vmthunks_actors.cpp:1929-1935`,
declared `native bool ReflectOffActor(Actor blocking);` at
`wadsrc/static/zscript/actors/actor.zs:887`. `P_BounceActor` likewise —
`vmthunks_actors.cpp:1886-1893`, `actor.zs:883`.

### 2.3 The two hands — **both** are readable

Baseline correction, re-verified here. `wadsrc/static/zscript/actors/actor.zs:268-276`:
```
native readonly vector3 AttackPos;     native readonly double AttackPitch;
native readonly double  AttackRoll;    native readonly double AttackAngle;
native readonly vector3 OffhandPos;    native readonly double OffhandPitch;
native readonly double  OffhandRoll;   native readonly double OffhandAngle;
native readonly bool    OverrideAttackPosDir;
```
Thunks `src/scripting/vmthunks_actors.cpp:2187-2194`; natives `src/playsim/actor.h:1654-1664`.

- **Weapon hand:** `AttackPos` / `AttackAngle` / `AttackPitch` / `AttackRoll`, filled from the real
  controller transform at `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:585-597`
  (`weaponoffset[]`, `weaponangles[]`) and `src/rendering/gl/stereo3d/gl_openvr.cpp:2525-2537`.
- **Shield hand:** `OffhandPos` etc., the identical block at `hw_openxrdevice.cpp:599-610`
  (`offhandoffset[]`, `offhandangles[]`).
- Both are **world positions**: `AttackPos.X = player->mo->X() - (weaponoffset[0] * vr_vunits_per_meter)`
  (`hw_openxrdevice.cpp:593`), `OffhandPos.X` likewise at `:605`. The Z lines (`:595`, `:610`)
  divide by `pixelstretch` (default 1.2, `:572`), so the **vertical** scale is 34/1.2 = 28.33
  units/m while XY is 34 units/m.
- **The gate is `OverrideAttackPosDir`** (`actor.zs:276`), set at `hw_vrmodes.cpp:438` to
  `!puristmode && (IsVR() || vr_override_weap_pos)`. When it is false the values are **not hand
  poses**: `VRMode::SetUp` collapses `AttackPos = OffhandPos = PosAtZ(shootz)` and both angle sets
  to the view angles (`hw_vrmodes.cpp:440-444`). That block is the non-VR fallback; in VR the
  device code above overwrites it every frame. Gate every shield behaviour on
  `OverrideAttackPosDir` (and/or `player.PlayInVR`), not on the presence of the fields.
- A shield has a facing, and `OffhandAngle`/`Pitch`/`Roll` give you all three of its angles for
  free. `OffhandAngle` is the input `+SHIELDREFLECT`'s cone wants.
- `LineTrace(..., TRF_USEWEAPON|TRF_ISOFFHAND, ...)` casts a ray **from the shield hand along the
  shield hand's facing** — `src/playsim/p_map.cpp:5185-5190`, flags at
  `wadsrc/static/zscript/constants.zs:962`, `:4096`.
- `LineAttack(..., LAF_ISOFFHAND, ...)` fires a hitscan **from the shield hand** —
  `src/playsim/p_map.cpp:4786-4792`, flag at `wadsrc/static/zscript/constants.zs:946`. This is
  the shield bash, already wired.
- The shield *could* be an **offhand `Weapon`** (`+OffhandWeapon`, `PSP_OFFHANDWEAPON`) — see
  `_BASELINE_FACTS.md` §2 — and HUD **models** render on psprite layers in VR at the render-frame
  hand transform (`src/rendering/hwrenderer/scene/hw_weapon.cpp:107-108`, `:139-141`;
  `VRMode::GetWeaponTransform(out, VR_OFFHAND)` at `hw_vrmodes.cpp:455-470`).
  **But do not host it that way — see §2.4.**

### 2.4 THE TWO-HAND TRAP — read this before choosing how to host the shield

**A shield is the archetypal offhand weapon, and bringing it across to guard the sword arm is
exactly the pose that disables it.** Found by the bows survey, re-verified here line by line.

`src/rendering/gl/stereo3d/gl_openvr.cpp:1717-1728`:
```
if (vr_two_handed_weapons &&
    (pOffTrackedRemoteNew->ulButtonPressed & ButtonMaskFromId(k_EButton_Grip)) !=
    (pOffTrackedRemoteOld->ulButtonPressed & ButtonMaskFromId(k_EButton_Grip)))
{
    if (pOffTrackedRemoteNew->ulButtonPressed & ButtonMaskFromId(k_EButton_Grip)) {
        if (distance < 0.50f) { weaponStabilised = true; }
    } else { weaponStabilised = false; }
}
```
`distance` is the raw 3D controller separation (`:1710-1715`). **There is no `bTwoHanded` test, no
test that an offhand weapon exists, and no test of what that weapon is.** Offhand grip + hands
inside half a metre, and the offhand goes dark:

| Consequence | Line |
|---|---|
| Offhand weapon fires nothing | `wadsrc/static/zscript/actors/player/player.zs:393-396`, also `:426`, `weapons.zs:415` |
| Offhand weapon checks no buttons | `src/playsim/p_pspr.cpp:755-758` |
| Offhand psprite renders nothing | `src/rendering/hwrenderer/scene/hw_weapon.cpp:832-835`, `:917-920` |
| `WF_TWOHANDSTABILIZED` re-asserted on **every** state set, so script cannot durably clear it | `src/playsim/p_pspr.cpp:507-515` — the flag is cleared at `:509` and immediately re-OR'd at `:512-515` |

Flag at `src/playsim/d_player.h:164`, mirrored `wadsrc/static/zscript/constants.zs:1242`.

**Two corrections to how this reached me, both load-bearing:**

1. **Only the OpenVR path sets it.** `weaponStabilised` is one global bool
   (`src/QzDoom/qzdoom_common.cpp:19`, declared `hw_vrmodes.h:100`) and the *only* writes in the
   entire tree are `gl_openvr.cpp:1723` and `:1726`. `hw_openxrdevice.cpp` never writes it. So on
   an **OpenXR runtime — VDXR, which is this owner's setup — the trap does not fire today.**
   Do not build on that: it is one global, one patch away from being set on both paths, and a mod
   that silently breaks under SteamVR is still broken.
2. **The escape hatch is visible in the suppression itself.** Both render skips are keyed on
   `psp->GetCaller() == player->OffhandWeapon` (`hw_weapon.cpp:832`, `:917`) — **not** on the
   layer ID. A psprite layer whose caller is not the offhand `Weapon` object keeps rendering, and
   the fire/button gates (`player.zs:393`, `p_pspr.cpp:755`) only guard `hand == 1` *weapon*
   logic, which a non-weapon layer never enters.

**Therefore: do not host the shield as an offhand `Weapon`.** Host it as a plate actor plus a
plain psprite overlay layer whose caller is not `OffhandWeapon`. This costs nothing — the plate
has to be a world actor regardless (§3.2), and the shield's state is a handful of fields on that
actor rather than a `Weapon` state machine. The shield then survives the grip-brace pose, survives
`vr_two_handed_weapons` being on, and leaves the offhand weapon slot free for an actual weapon
(§4.12). This revises §4's preamble and §6 step 2 accordingly.

If you *must* ship it as an offhand `Weapon`: keep the guard pose beyond 0.50 m, and read
`player.WeaponState & WF_TWOHANDSTABILIZED` (`constants.zs:1242`) each tic so it fails loudly
rather than silently.

### 2.5 Pass-through pairing (`ThruBits`)

`if ((thing->ThruBits & tm.thing->ThruBits) && ((thing->flags8 | tm.thing->flags8) & MF8_ALLOWTHRUBITS)) return true;`
— `src/playsim/p_map.cpp:1412`, and the same at `:534` and `:2111`.
`ThruBits` is a plain scriptable `uint` (`wadsrc/static/zscript/actors/actor.zs:100`, property at
`:379`); `+ALLOWTHRUBITS` at `src/scripting/thingdef_data.cpp:332`.
This is the clean, intended way to let **your own player walk through your own shield plate**
while the plate still blocks everything else — it is tested first, before any other condition.

Hitscans honour `ThruBits` too, but only via the **puff's** bits
(`src/playsim/p_map.cpp:4841-4842`, `:4696`), which a shield mod does not control. See §5.

---

## 3. Building blocks

| Primitive | Where | Note |
|---|---|---|
| `SetOrigin(vector3, bool moving)` | `actor.zs:753`; impl `src/playsim/p_maputl.cpp:563-571` | Unlink / `SetXYZ` / relink / `P_FindFloorCeiling`. **No collision test.** `moving=true` keeps interpolation. |
| `SetXYZ(vector3)` | `actor.zs:754` | Writes the position **without relinking the blockmap**. Collision then uses stale links. Never use it to move a collider. |
| `Warp(...)` | `actor.zs:942`; impl `src/playsim/p_things.cpp:~700-820` | Calls `SetOrigin`, then `P_TestMobjLocation`, **reverts on failure** (`p_things.cpp:819`), and ends with `P_TryMove` (`p_things.cpp:816`). |
| `A_SetSize(r, h, testpos)` | `actor.zs:1314`; impl `src/playsim/p_actionfunctions.cpp:5062-5094` | Unlink, write `radius`/`Height`, relink; `testpos` reverts if the new size doesn't fit. |
| `CanCollideWith(other, passive)` | `actor.zs:562`; impl `src/playsim/p_map.cpp:153-185`, called `:1514` | Asked of **both** actors. Only runs for `SOLID`/`TOUCHY`/`BUMPSPECIAL` vs `SOLID`/`MISSILE`/`BLASTED`/… |
| `SpecialMissileHit(victim)` | `actor.zs:588`; impl `src/playsim/p_mobj.cpp:3434-3446`, called `p_map.cpp:1655` | **On the missile, not the shield.** Useless for foreign projectiles. |
| `DamageMobj(...)` | `actor.zs:801`; missile path `P_DoMissileDamage`, `p_map.cpp:1352-1384` | The shield's own hook, if it is `+SHOOTABLE`. Durability lives here. |
| `BlockThingsIterator.CreateFromPos(x,y,z,h,r,ign)` | `wadsrc/static/zscript/doombase.zs:225` | Script-side sweep around the hand. The "software shield". |
| `LineTrace(...)` | `actor.zs:805`; impl `p_map.cpp:5165-5300` | `TRF_ALLACTORS` (`constants.zs:958`) makes it see non-shootable actors — including missiles. |
| `ReflectOffActor(blocking)` / `BounceActor(blocking, onTop)` | `actor.zs:887` / `:883` | Run the native reflection yourself, on a missile you found. |
| `bReflective`, `bShieldReflect`, `bAimReflect`, `bSolid`, `bShootable`, `bThruActors`, `bInvulnerable` | `src/scripting/thingdef_data.cpp:176, 225, 302, 166, 290, …` | All plain runtime bools. Flip per tic. |
| `InterpolatedPosition(ticFrac)` = `Prev + ticFrac*(Pos()-Prev)` | `src/playsim/actor.h:1488-1492` | How a **world** actor is drawn. `+DONTINTERPOLATE` (`thingdef_data.cpp:378`) disables it. |

### 3.1 The numbers — quantifying the mismatch

`CVAR(Float, vr_vunits_per_meter, 34.0f, ...)` — `hw_vrmodes.cpp:94`.
Tic = 1/35 s = 28.57 ms (`_BASELINE_FACTS.md` §3).

- 1 map unit horizontally = **2.94 cm**; vertically = **3.53 cm**. The vertical difference is
  `level.pixelstretch` (default 1.2), applied on the Z lines of both hand poses
  (`hw_openxrdevice.cpp:595`, `:610`) and in the OpenVR transform (`gl_openvr.cpp:1282-1292`).
  **Forget it and every vertical distance you compute reads ~17% short** (1/1.2 = 0.83) — which,
  on a shield whose job is to be in front of a face, is the difference between covering the head
  and covering the chin.
- A 60 cm shield is **20.4 map units** across (radius ~10) and, at 90 cm tall, **25.5** units high.

| Hand speed | per tic (cm) | per tic (map units, XY) | as a fraction of a 60 cm shield |
|---|---|---|---|
| 0.5 m/s — holding, micro-corrections | 1.4 | 0.49 | 2% |
| 1 m/s — tracking an approach | 2.9 | 0.97 | 5% |
| 2 m/s — a deliberate block | 5.7 | 1.94 | **9%** |
| 3 m/s — a brisk re-position | 8.6 | 2.91 | 14% |
| 5 m/s — a parry swat | 14.3 | 4.86 | **24%** |
| 8 m/s — a shield bash | 22.9 | 7.77 | **38%** |
| 12 m/s — a hard throw release | 34.3 | 11.66 | 57% |

**What this means for "solid".** The plate is placed once per tic at the *latest rendered frame's*
`OffhandPos` (baseline §3 — the value is fresh, not stale). Over the tic the drawn shield drifts
away from it by at most one row of that table. For blocking — the shield held up, moving at
0.5-2 m/s — the plate is inside the thickness of the shield model and the player will never
perceive the gap. **It is solid.**

The failure mode is different from a lag: because `SetOrigin` **teleports**, the plate sweeps
nothing. A projectile that passes through the volume the shield crossed *between* two tics is not
blocked. At 5 m/s the plate skips a 14 cm band every tic. A deliberate swat at a fireball is
therefore the one motion a plate-based shield will drop.

**The missile side does not have this problem.** `P_XYMovement` substeps a missile at
`maxmove = mo->radius - 1` (`src/playsim/p_mobj.cpp:2166-2192`), and the blocking test uses
`thing->radius + tm.thing->radius` (`p_map.cpp:1417`), so a fast rocket cannot tunnel through a
stationary plate. **Exception worth naming:** a projectile with `radius <= 1` falls back to
`maxmove = MAXMOVE` = 30 (`p_mobj.cpp:2168-2171`, `src/playsim/p_local.h:61`), so a zero-radius
projectile above ~1000 units/s *can* step clean over a small plate.

### 3.2 Which move call — `SetOrigin` vs `Warp` vs direct write

**Use `SetOrigin(pos, true)`. Every tic. Nothing else is correct.**

**First, the mechanism that makes this work at all** (settled by the grab survey, re-verified
here). `Prev` is not maintained by the move calls. It is re-snapshotted for **every actor on every
level at the top of every tic, before any thinker runs**:
```
// Reset all actor interpolations on all levels before the current thinking turn so that
// indirect actor movement gets properly interpolated.
while ((ac = it.Next())) { ac->ClearInterpolation(); ac->ClearFOVInterpolation(); }
```
`src/p_tick.cpp:151-162`, and `ClearInterpolation()` is just `Prev = Pos(); PrevAngles = Angles; …`
(`src/playsim/actorinlines.h:29-35`). It is written nowhere else during the tic.

**Consequence, and it is the single thing that decides whether a VR shield feels solid:** *any*
position write made during the tic — including a teleport — is rendered as a smooth tween from
where the actor was at the start of the tic to where it is now, via
`InterpolatedPosition(ticFrac) = Prev + ticFrac*(Pos()-Prev)` (`src/playsim/actor.h:1488-1492`).
The plate does **not** judder. It glides.

What it *does* do is trail by exactly one tic: at `ticFrac = 0` the plate is drawn where the hand
was last tic, and it only arrives at this tic's hand position at `ticFrac = 1`. That is up to
28.6 ms of positional lag — §3.1's table, read as "how far behind the drawn plate is". This is why
the **model goes on the offhand psprite and only the invisible collider is a world actor**: the
psprite is transformed at the render frame (`GetWeaponTransform(VR_OFFHAND)`,
`hw_vrmodes.cpp:455-470`; HUD models at `src/rendering/hwrenderer/scene/hw_weapon.cpp:107-108`)
and has no lag at all.

- `SetOrigin` — `src/playsim/p_maputl.cpp:563-571`:
  ```
  UnlinkFromWorld(&ctx); SetXYZ(x,y,z); LinkToWorld(&ctx);
  P_FindFloorCeiling(this, FFCF_ONLYSPAWNPOS);
  if (!moving) ClearInterpolation();
  ```
  No `P_CheckPosition`, no `P_TryMove`, no revert. It will happily place the plate inside a wall
  or inside a monster — which is what you want, because the *hand* decides where the shield is,
  not the geometry. `moving=true` leaves `Prev` alone so the renderer tweens
  (`actor.h:1488-1492`); `moving=false` snaps `Prev` to `Pos` and the actor pops.
- `Warp` — `src/playsim/p_things.cpp:~700-820`. Wrong tool twice over: it **reverts the move** if
  `P_TestMobjLocation` fails (`:765`, `:819`), so the plate would refuse to follow your hand into
  a doorway or against a monster; and it finishes with `P_TryMove(caller, caller->Pos().XY(), false)`
  (`:816`), a full collision-resolving move that can slide the plate off the hand. Its
  interpolation is a *choice* — `WARPF_INTERPOLATE` / `WARPF_WARPINTERPOLATION` /
  `WARPF_COPYINTERPOLATION` (`p_things.cpp:789-805`) — and the default without any of them is
  `ClearInterpolation()`, i.e. a visible pop every tic.
- **Direct `SetXYZ` / writing `pos`** — `actor.zs:754`. This moves the actor *visually* but leaves
  it linked into the old blockmap cells and the old sector list (`LinkToWorld`,
  `p_maputl.cpp:505-557`, links into **every** block the radius touches). Collision then happens
  where the shield used to be. This is the bug that will look like "the shield blocks nothing" or
  "the shield blocks at the wrong place"; it is not an engine fault.

**Draw on the psprite; block with an invisible plate.** That split is the whole answer to the
central problem: zero-lag visuals from the render-frame hand transform, and a collider that is
smooth (the `Prev` snapshot above) and one tic behind — which per §3.1 is a hand's thickness at
blocking speeds.

### 3.3 Sizing the plate — collision is a box, not a cylinder

The claim "GZDoom collision is cylinders" is **half wrong and the half that is wrong matters here.**

Actor-vs-actor, XY (`src/playsim/p_map.cpp:1417-1419`):
```
double blockdist = thing->radius + tm.thing->radius;
if (fabs(thing->X() - cres.Position.X) >= blockdist ||
    fabs(thing->Y() - cres.Position.Y) >= blockdist) return true;
```
That is a **Chebyshev test — an axis-aligned square footprint of side 2·radius**, not a circle.
Hitscan agrees (`src/playsim/p_trace.cpp:695-696`, `:710-711`):
`if (fabs(hit.X - thing->X()) > radius || fabs(hit.Y - thing->Y()) > radius) return true;`

Vertically it is a slab `[Z, Z+Height]`, with one documented exception: against **missiles only**,
`projectilepassheight` replaces `Height` (`p_map.cpp:1620-1643`;
`wadsrc/static/zscript/actors/actor.zs:133`, property at `:331`). So you can give the plate a
**taller missile profile than its walk-on profile** for free.

The other exceptions to "simple slab": `+ACTLIKEBRIDGE` makes the top walkable
(`p_map.cpp:1437-1456`, `thingdef_data.cpp:211`), and 3D floors / polyobjects are *geometry*, not
actors, so they are not available to a runtime-spawned shield (no ZScript polyobject API exists —
`Polyobj_MoveTo` is special 88, `src/playsim/actionspecials.h:89`, reachable only via
`Level.ExecuteSpecial` (`wadsrc/static/zscript/doombase.zs:489`), speed-based rather than
instantaneous, and polyobjects must be authored into the map).

**Nothing in the collision path reads `Angles.Yaw`.** The box cannot be rotated. A shield angled
45° to the world axes presents the same square as one facing north.

#### One fat box vs several small ones

| | One box, radius 20, Height 34 | 3×3 grid of radius-5, Height-9 boxes | 1 box + `CanCollideWith` |
|---|---|---|---|
| Covered volume | 118×118×120 cm — a **cube**, over-covering a 60 cm shield by ~4× in footprint | ~90×90×95 cm in nine chunks, still axis-aligned, still not a plate | exactly the shape you write |
| Per-tic cost | 1 × (`UnlinkFromWorld` + `LinkToWorld` + `P_FindFloorCeiling`) | **9 ×** the same. `P_FindFloorCeiling` is the expensive member — it is a full sector/line sweep, not a pointer swap | 1 × |
| Blockmap load | links into ⌈40/128⌉+1 = 2×2 cells (`p_maputl.cpp:520-537`) | 9 actors × 1-2 cells = 9-18 block entries; **every mover near the player tests all nine** in `PIT_CheckThing` | 2×2 cells |
| Per-collision cost | 0 extra | 0 extra | 1 VM call per candidate, bounded by `p_map.cpp:1508-1514` to SOLID-vs-(SOLID\|MISSILE) pairs only |
| Fidelity | terrible — blocks shots that visibly miss by half a metre | mediocre — still a staircase of boxes, still unrotatable | good |
| Hitscan fidelity | box | staircase of boxes | **box** — `CanCollideWith` is not called from `p_trace.cpp` |

**Recommendation:** one box sized to the *inscribed* square of the shield (radius ≈ half the
shield's narrow dimension, so it under-covers rather than over-covers), plus `CanCollideWith` to
reject anything outside the real plate. The nine-actor approach costs nine floor/ceiling sweeps
per tic to buy a shape that is still wrong in the one way that matters (orientation), and it is
not worth it. Reach for a second actor only when the shape is genuinely disjoint — e.g. a shield
*and* a separate bash hitbox.

### 3.4 Does a non-player actor actually intercept projectiles aimed at the player?

**Yes.** `PIT_CheckThing`, `src/playsim/p_map.cpp:1677-1680`:
```
if (!(thing->flags & MF_SHOOTABLE))
{ // Didn't do any damage
    return !(thing->flags & MF_SOLID);
}
```
Returning `false` makes `P_TryMove` fail with `BlockingMobj` set, and `P_XYMovement` then runs
bounce → reflect → `P_ExplodeMissile` (`src/playsim/p_mobj.cpp:2335-2395`).

So **`+SOLID` is the single flag that makes an actor block a missile.** The gates it must clear,
in order (`p_map.cpp:1398-1680`):

1. `thing != tm.thing` (`:1405`)
2. neither has `+THRUACTORS` (`:1409`)
3. no matching `ThruBits` pair (`:1412`)
4. has one of `SOLID|SPECIAL|SHOOTABLE|TOUCHY` (`:1415`)
5. XY box overlap (`:1417`)
6. **not `+NONSHOOTABLE`** (`:1595-1598`) — this flag makes missiles pass straight through; a
   shield must never have it
7. not a `+GHOST` vs `+THRUGHOST` pair (`:1600`), not `MTHRUSPECIES` (`:1605`)
8. Z overlap against `projectilepassheight`/`Height` (`:1620-1643`)
9. `SpecialMissileHit` on the **missile** may veto (`:1655-1660`)
10. not the missile's own `target` (`:1666`); `CanAttackHurt` (`:1671`)
11. → `MF_SHOOTABLE`? damage + block. Not shootable? **block, no damage.**

**The infighting leak, and how to close it.** Step 10 above is `CanAttackHurt(thing, tm.thing->target)`
(`p_map.cpp:1670-1676`, impl `:1275-1339`). On a map with `NoInfighting` set, a monster missile
whose shooter is non-player can be waved through the damage branch. Two things make this a
non-issue in practice and one line makes it a non-issue always: players are exempt outright
(`:1277-1279`), and `+FORCEINFIGHTING` on the plate flips `infight` from -1 to 0 and forces the
check (`:1282`). Put `+FORCEINFIGHTING` on the plate and stop thinking about it. Note also that
`CanAttackHurt` returning false still returns `false` from `PIT_CheckThing` — the missile is
**blocked** either way; only the damage is skipped. For a `-SHOOTABLE` plate this gate is never
reached at all (step 11 short-circuits at `:1677`).

**Hitscans are a real block too, and this is now established rather than inferred.**
`P_LineAttack` traces with `MF_SHOOTABLE` as its actor mask (`src/playsim/p_map.cpp:4919-4920`)
and the filter `CheckForActor` (`:4673-4700`) returns `TRACE_Stop` on the **first** shootable
actor with **no test of who the shooter intended to hit**. A non-player actor standing between a
monster and the player genuinely eats the bullet. The rail attack is the same (`:5745`). The
other side of that coin: a `+SOLID -SHOOTABLE` plate is **invisible to every bullet in the game.**
To stop
hitscans the plate must be `+SHOOTABLE`, which also means it takes damage (good — that is your
durability system) and becomes a legitimate autoaim candidate (`p_map.cpp:4378`) — give it
`+NOTAUTOAIMED` (`thingdef_data.cpp:286`) and `+NEVERTARGET` (`:290`).

`P_LineTrace` is the script-side exception: `TRF_ALLACTORS` sets the mask to `0xFFFFFFFF` and
`TRF_SOLIDACTORS` adds `MF_SOLID` (`p_map.cpp:5249-5250`), so your own detection traces can see
the plate and see missiles, whatever their flags.

### 3.5 Haptics — the impact thump, and how close you can get

The baseline (§4) calls haptics unreachable. That is too pessimistic in one direction and too
optimistic in another; here is what is actually in the tree.

**What is implemented:**
- `VRMode::Vibrate(duration, channel, intensity)` is a virtual with an empty base
  (`src/common/rendering/hwrenderer/data/hw_vrmodes.h:93`).
- **OpenVR overrides it for real** — `gl_openvr.h:121-124` → `OpenVRHaptics::Vibrate`
  (`src/rendering/gl/stereo3d/gl_openvr.cpp:517-528`), queued and flushed by `ProcessHaptics`,
  gated on `vr_enable_haptics` (`hw_vrmodes.cpp:125`, checked `gl_openvr.cpp:532`).
- **OpenXR overrides it too** — `hw_openxrdevice.h:83`, impl `hw_openxrdevice.cpp:755-757`, which
  calls `QzDoom_Vibrate`. **But `QzDoom_Vibrate` has no definition anywhere in this repository**
  — only the forward declaration at `hw_openxrdevice.cpp:140` and that one call site. It must be
  supplied by the Quest-side native layer, which the PC edition does not build. So on the OpenXR
  path, treat haptics as **unproven** until someone cites the definition.
- `VR_HapticEvent(...)` is an **empty stub** — `src/QzDoom/qzdoom_common.cpp:65-67`.
- **Neither is bound to ZScript.** Zero hits for vibrate/haptic/rumble in `wadsrc/static/zscript`.

**But `Vibrate` is called from playsim, on a path a mod controls.** `P_LineAttack`,
`src/playsim/p_map.cpp:4762-4775`:
```
if ( damage > 0) {
    long rightHanded = vr_control_scheme < 10;
    rightHanded = (flags & LAF_ISOFFHAND) ? 1 - rightHanded : rightHanded;
    vrmode->Vibrate(150, rightHanded ? 1 : 0, 0.8);
    ...
```
and the same block in `P_SpawnPlayerMissile` (`src/playsim/p_mobj.cpp:7549-7562`). The gate is
`damage > 0` **and nothing else** — in particular it is *not* gated on `LAF_NOINTERACT`, which
sits further down (`p_map.cpp:4772` is reached before the `nointeract` branches).

**So the shield can fire an impact thump on the shield hand:**
```
LineAttack(angle, 0.1, pitch, 1 /*damage>0, required*/, "None", "RSQ_NullPuff",
           LAF_NOINTERACT | LAF_ISOFFHAND);
```
`LAF_NOINTERACT` (`constants.zs:941`) means no damage dealt, no sound, no blood, no decal — it
spawns a puff and returns it (`p_map.cpp:4930-4972`), so give it a puff class with no sprite and
`-1`-tic Stop. `LAF_ISOFFHAND` flips the channel to the shield controller (`p_map.cpp:4767`).
Result: a 150 ms, 0.8-intensity pulse on the correct hand, and nothing else happens in the world.

**Label: HACK. Cost: S.** It is a hardcoded fire event provoked for its side effect — exactly the
case the baseline §4 says to call HACK rather than BUILDABLE. Duration, channel and intensity are
the literals `150` / `0.8` and cannot be varied from script; a "heavy block" and a "light tick"
are the same pulse. On OpenXR it may do nothing at all (see above). Budget sound and a psprite
kick as the primary feedback and treat this as a bonus.

---

## 4. Mechanic designs

Throughout, **"the shield"** is a **plate actor plus a plain psprite overlay layer** — *not* an
offhand `Weapon`, for the reason in §2.4. The plate is an invisible world actor that owns the
shield's state, moves itself in its own `Tick`, and is destroyed when the shield is stowed; the
overlay layer carries the MODELDEF model and is drawn at the render-frame hand transform. The
layer's caller must not be `player.OffhandWeapon`, or the suppression at `hw_weapon.cpp:832`
applies to it.

Plate defaults, unless a mechanic says otherwise:
```
+SOLID +NOGRAVITY +NOTAUTOAIMED +NEVERTARGET +NOBLOOD
+FORCEINFIGHTING // closes the CanAttackHurt leak on NoInfighting maps (§3.4)
+ALLOWTHRUBITS   // ThruBits shared with the owner, so you don't block yourself
RenderStyle "None";
Radius <half the shield's narrow dimension>;  Height <shield height / 3.53>;
```
and in `Tick()`: `SetOrigin(owner.OffhandPos + facingOffset, true);` then
`angle = owner.OffhandAngle; pitch = owner.OffhandPitch; roll = owner.OffhandRoll;`
(the angles are free and `+SHIELDREFLECT` reads `Angles.Yaw`).

Two flags **not** in that list, deliberately:
- **`+DONTINTERPOLATE`** (`src/scripting/thingdef_data.cpp:378`) — leave it off. Per §3.2 the
  per-tic `Prev` snapshot already makes a teleporting plate render smoothly; disabling
  interpolation would make it snap 35 times a second instead. Only set it if you ever draw a
  model on the plate itself and want it pinned to the collider rather than tweened.
- **`+NOINTERACTION`** (`thingdef_data.cpp:253`) — a trap. Its fast path in `AActor::Tick`
  **forces `MF_NOBLOCKMAP`** (`src/playsim/p_mobj.cpp:3988-3992`: `flags |= MF_NOBLOCKMAP`), which
  takes the plate out of the blockmap entirely and makes it block nothing. If you want a plate
  that costs no thinking but still collides, use `freezetics` instead — it early-returns from
  `Tick` at `p_mobj.cpp:3948-3952`, *above* the `NOINTERACTION` block, and leaves the actor linked.
  `freezetics` is a writable `native uint` (`wadsrc/static/zscript/actors/actor.zs:265`).

---

### 4.1 Block — projectiles

**Player:** raises the offhand into the line of fire. No button.
**Detection:** none needed. The plate is simply there.
**Resolution:** `PIT_CheckThing` → `!SHOOTABLE && SOLID` → `return false` (`p_map.cpp:1677-1680`)
→ `P_ExplodeMissile` (`p_mobj.cpp:2395`). The fireball dies on the shield.
**Refinement:** override `CanCollideWith(other, passive)` (`actor.zs:562`) on the plate and return
`false` unless the incoming actor is within the shield's real frontal cone:
`absangle(AngleTo(other), angle) < coneDeg && abs(DeltaZ) < halfHeight`. This is what turns the
square box back into a plate. It is called for missiles — `p_map.cpp:1508-1514` includes
`tm.thing->flags & MF_MISSILE` — and only for SOLID pairs, so it is not a per-actor-per-tic cost.
Projectile resolution in full is `PIT_CheckThing` `p_map.cpp:1589-1717`.
**Feel knobs:** plate radius vs drawn shield size (under-cover ≈ "you must aim the shield",
over-cover ≈ "forgiving"); cone half-angle in `CanCollideWith`; whether the plate leads the hand
by its own half-depth along `OffhandAngle`; an impact sound, the §3.5 haptic thump, and a psprite
recoil offset on
`DamageMobj`.
**Label: EXISTS (a `+SOLID` actor blocking a missile is engine behaviour, `p_map.cpp:1677-1680`) +
BUILDABLE (the plate). Cost: S.**

### 4.2 Block — hitscans

**Player:** the same motion.
**Detection:** none.
**Resolution:** the plate must be `+SHOOTABLE` (`p_map.cpp:4919-4920` masks the trace on it).
`CheckForActor` then returns `TRACE_Stop` on it as the first shootable actor along the ray, with
**no test of whom the shot was aimed at** (`p_map.cpp:4673-4700`). It takes the damage; override
its `DamageMobj` to spend durability and return 0 so nothing else happens.
**Trap — your own gun.** The trace stops on the plate *before* any damage logic
(`CheckForActor`, `p_map.cpp:4680-4700` → `TRACE_Stop`), so a `+SHOOTABLE` plate **eats your own
bullets** if you fire through your own shield. `ThruBits` would fix it but only via the *puff's*
bits (`p_map.cpp:4841-4842`), which a shield mod does not own. Practical fixes, all script-side:
clear `bShootable` on the tic the owner's weapon fires; or accept it as correct behaviour (you did
shoot your shield) and play a ricochet.
**Feel knobs:** `+SHOOTABLE` permanently vs only while a "brace" button is held; damage factors per
`DamageType` on the plate (`DamageFactor "Fire", 0.2`) so a shield resists some things and not
others.
**Label: EXISTS (the interception is engine behaviour, not inference — `p_map.cpp:4673-4700`) +
BUILDABLE (the plate that uses it). Cost: S.** (The self-shot trap is the only real work.)

### 4.3 Block — melee and splash

**WALL for both, with a clean script fallback.**
`P_CheckMeleeRange` tests distance, Z overlap and `P_CheckSight` only (`src/playsim/p_enemy.cpp`,
`P_CheckMeleeRange`), and `P_RadiusAttack` gates on `P_CheckSight` too
(`src/playsim/p_map.cpp:6585`, `:6448`). `src/playsim/p_sight.cpp` is 967 lines with no actor
iteration anywhere — sight is purely sectors and lines. **No actor has ever blocked either.**

**Fallback:** override `DamageMobj` on the *player*, and when `inflictor`/`source` lies inside the
shield's frontal cone and within the shield's reach, scale the damage down and spawn the clang.
This is fully in a mod's control and is how every GZDoom shield mod has always done melee.
**Feel knobs:** cone half-angle; reduction factor; whether a blocked melee also staggers (§4.6).
**Label: HACK (script substitute for a physics behaviour the engine does not have). Cost: S.**
**Missing piece for a true WALL fix:** an actor test inside `p_sight.cpp` / `P_CheckMeleeRange`.
Not available on a frozen engine.

### 4.4 Parry — timed reflect back at the shooter

**Player:** presses the offhand trigger (or thrusts the shield forward past a speed threshold) as
a projectile arrives.
**Detection:**
- *Button:* since the shield is not a `Weapon` (§2.4), read the button directly rather than through
  `CheckWeaponButtons(hand)` (`player.zs:588`, `:2031`, which is offhand-*weapon* machinery and is
  gated off by `WF_TWOHANDSTABILIZED` at `p_pspr.cpp:755-758` anyway): poll `player.cmd.buttons`
  against the `BT_*` mask in the plate's `Tick` (`UserCmd.buttons`, `player.zs:4`). This is also
  immune to the §2.4 trap.
- *Gesture:* in the shield's `Tick`, keep `prevOffhandPos`; `speed = (OffhandPos - prev).Length() * 35`
  map units/s. 5 m/s = 170 u/s. Baseline §1b: you must difference it yourself; §3: you get 2-4
  samples in a fast swat. Require **two consecutive** tics over threshold or you will fire on
  tracking jitter.

**Resolution — use the engine, do not reimplement it.** For the window length, set on the plate:
```
bReflective = true;  bShieldReflect = true;  bAimReflect = true;   // open
...
bReflective = false; bShieldReflect = false; bAimReflect = false;  // close
```
This is literally `A_SetReflectiveInvulnerable` (`actor.zs:964`) plus two flags. While open, a
missile arriving within ±45° of the plate's `Angles.Yaw` is re-aimed at its own shooter at full
`Speed` (`p_mobj.cpp:3468-3477` for the cone; `p_map.cpp:3845-3850` for the aim) instead of
exploding. Outside the cone it explodes — i.e. the shield still *blocks*, it just doesn't *parry*.
That cone/blocked split is exactly the feel you want and it is already written.
**Kill credit:** `P_ReflectOffActor` ends with `mo->target = blocking` (`p_map.cpp:3875`). Set the
plate's `target`/`master` to the player, or re-point `mo->target` in the plate's `Tick`.

**The parry's real enemy is §3.1.** At 5 m/s the plate skips 4.9 units per tic, so a *swatting*
parry can miss the projectile it was aimed at. If the parry is gesture-driven, do not rely on the
plate for the detection — on the tic the gesture fires, run a `BlockThingsIterator.CreateFromPos`
(`doombase.zs:225`) around the hand with a generous radius, filter to `bMissile`, and call
`missile.ReflectOffActor(plate)` (`actor.zs:887`) directly. That covers the swept gap because
*you* chose the radius; the plate alone cannot.
**Feel knobs:** window length in tics (the Centaur ships 45 ≈ 1.29 s; a tight VR parry is 4-8 tics
≈ 114-229 ms); `+MIRRORREFLECT` (straight back, half speed, `p_map.cpp:3851-3855`) vs
`+AIMREFLECT` (at the shooter, full speed) as a "perfect parry" upgrade; cooldown tics; whether
`bInvulnerable` rides along with the window as the Centaur does.
**Label: EXISTS (the mechanic) + BUILDABLE (the VR trigger). Cost: S for a button parry, M for a
gesture parry.**
**Named limit:** the ±45° cone is a literal in `p_mobj.cpp:3471` with no property behind it. If
you want a 20° cone, drop `+SHIELDREFLECT`, keep `+REFLECTIVE`, and do the cone yourself in
`CanCollideWith`.

### 4.5 Parry — reflecting *hitscans*

**WALL.** `P_ReflectOffActor` operates on an `AActor` with a velocity (`p_map.cpp:3832-3879`); a
hitscan has no actor to redirect, and `CheckForActor` offers only Stop/Skip (`p_map.cpp:4680-4700`).
**Missing piece:** any hook that converts a trace hit into a new trace from the hit point.
**Script substitute:** on the plate's `DamageMobj`, if the window is open and the damage is a
bullet, spawn a *new* `LineAttack` from the plate back along `AngleTo(source)` with
`LAF_ISOFFHAND` (`constants.zs:946`, `p_map.cpp:4786`). It is a different thing happening at the
same moment, and it reads correctly in play.
**Label: WALL (true reflection) / HACK (convincing substitute). Cost: S for the substitute.**

### 4.6 Parry — staggering melee enemies

**Player:** parries as a melee attacker commits.
**Detection:** the player's `DamageMobj` override from §4.3, or a `BlockThingsIterator` sweep in
front of the shield on the parry tic.
**Resolution:** on the attacker, `SetStateLabel("Pain")` (`actor.zs:839`), then `tics = staggerTics`
to hold it, `Vel.XY = ...` or `Thrust(speed, angle)` (`actor.zs:860`) to shove it back, and
`reactiontime` (`actor.zs:322`) to delay its next attack. All of these are plain writes on another
actor from `play` scope.
**Feel knobs:** stagger tics; knockback scaled by the shield's own measured speed and by the
attacker's `Mass`; whether a parry refunds the shield's durability.
**Label: BUILDABLE. Cost: S.**

### 4.7 Shield bash

**Player:** thrusts the shield forward hard.
**Detection:** the §4.4 speed differencing, with a direction test: the motion must be along
`OffhandDir` (`actor.zs:859`), not across it. Threshold ~5-6 m/s → 170-204 u/s → ≥4.9 u/tic.
Two consecutive tics over threshold; refractory period of ~15 tics.
**Resolution:** `LineAttack(angle, range, pitch, damage, "ShieldBash", "BashPuff", LAF_ISOFFHAND|LAF_ISMELEEATTACK)`
— the `LAF_ISOFFHAND` branch at `p_map.cpp:4786-4792` makes it originate at `OffhandPos` along
`OffhandDir`, i.e. genuinely from the shield. Then stagger as §4.6.
`LAF_ISOFFHAND` works **with no offhand weapon equipped** — it reads `OffhandPos`/`OffhandDir`
(`p_map.cpp:4786-4792`), and the only `OffhandWeapon` dereference on that path is null-tested
(`:4811-4816`). So the bash is unaffected by the §2.4 hosting decision.
**Feel knobs:** speed threshold; range (shield half-depth + 20-40 units); damage scaled by measured
speed; `+FORCEPAIN` on the puff for a guaranteed stagger; `A_QuakeEx` for impact; the §3.5 haptic
pulse rides along for free here, since a real bash already has `damage > 0`.
**Label: BUILDABLE. Cost: S.** (The offhand attack path is already in the engine; only the gesture
is yours.)

### 4.8 Stationary deployable shield — area denial

**Player:** holds the shield and presses the altfire, or plants it with a downward motion.
**Detection:** button, or `OffhandPos.Z` below a threshold with low speed.
**Resolution:** spawn a persistent `+SOLID +SHOOTABLE +NOGRAVITY` actor at the plant point and
**stop moving it**. Size it with `A_SetSize` (`actor.zs:1314`). Monsters are `+SOLID` and will be
blocked by it in `PIT_CheckThing` (`p_map.cpp:1753-1757`), their missiles will explode on it
(§3.4), their bullets will stop on it if it is `+SHOOTABLE`.
**The honest caveat:** it does **not** break line of sight (`p_sight.cpp` has no actor test), so
monsters still see, target and shoot at the player behind it — the shots just land on the barrier.
That reads fine in play. It also does **not** stop splash (`p_map.cpp:6585`).
**Make it cost nothing.** A planted barrier does not need to think. Set `freezetics` to a large
value (`actor.zs:265`): `AActor::Tick` decrements it and returns at `p_mobj.cpp:3948-3952`, which
is **above** everything else in `Tick`, while the actor stays linked into the blockmap and keeps
colliding exactly as before. Re-arm it each time it is hit if you want a hit reaction. Do **not**
reach for `+NOINTERACTION` to do this — it forces `MF_NOBLOCKMAP` (`p_mobj.cpp:3988-3992`) and the
barrier silently stops blocking.
**Feel knobs:** health; `Radius`/`Height`; whether `ThruBits` lets allies through; a decay timer;
`+ACTLIKEBRIDGE` (`thingdef_data.cpp:211`) if you want to stand on it.
**Label: BUILDABLE. Cost: S.**

### 4.9 Throwable returning shield (Captain America)

**Player:** a throwing motion with the offhand, then it comes back.
**Detection:** speed over ~8 m/s (≥7.8 u/tic) sustained two tics, plus the release of the grip.
Compute the throw vector from the last two `OffhandPos` samples — baseline §1b and §3 say this is
2-4 samples and that is all you get. Fit the direction on those, not on a curve.
**Resolution:**
- Outbound: spawn an actor with `+MISSILE`, `Vel` = measured hand velocity × a gain, and
  `+BOUNCEONWALLS +ALLOWBOUNCEONACTORS` (`thingdef_data.cpp:396`, `:399`) — the bounce path is
  `P_BounceActor` (`p_map.cpp:3727-3827`), entered at `p_mobj.cpp:2343`, and is also directly
  callable from script as `BounceActor(blocking, onTop)` (`actor.zs:883`). `bouncecount`
  (`p_map.cpp:3745-3752`) caps the ricochets.
- Return: when `bouncecount` runs out or a timer fires, clear `bMissile`, set `tracer = owner` and
  run `A_SeekerMissile(threshold, turnmax, flags, chance, distance)` (`actor.zs:1210`) — or simply
  steer `Vel` toward `owner.OffhandPos` each tic, which is more controllable.
- Catch: when `Distance3D(owner) < catchRadius` **and** the offhand is nearby, destroy it and
  re-ready the offhand weapon. Give it a grace state so a missed catch bounces off the player
  rather than hitting them.
**The free impact callback — use `+SKULLFLY`, not `+MISSILE`.** `PIT_CheckThing` tests the
*moving* actor for `MF_SKULLFLY` **before** any of the missile logic and hands you a virtual:
```
if (tm.thing->flags & MF_SKULLFLY)
{ bool res = tm.thing->CallSlam(tm.thing->BlockingMobj); tm.thing->BlockingMobj = NULL; return res; }
```
`src/playsim/p_map.cpp:1552-1557`, virtual declared `virtual native bool Slam(Actor victim);` at
`wadsrc/static/zscript/actors/actor.zs:520`. Override `Slam` on the thrown shield and you get
"the shield hit *this* actor, decide what happens" in one place — damage it, ricochet to the next
target, count the bounce, start the return — with the return value deciding whether the shield
keeps going. That is a far better fit for a returning shield than wiring up `+MISSILE` plus
bounce flags plus `SpecialMissileHit`, and it sidesteps the fact that `SpecialMissileHit` is on
the missile (§5.5) by *making the shield the missile*.
**Feel knobs:** throw gain; bounce count; return speed curve; catch radius (be generous — one tic
at return speed is several units); whether `Slam` damages or just ricochets.
**Label: BUILDABLE. Cost: M.** (Three states, a catch window, and a throw-gesture detector that
has to behave on 2-4 samples.)

### 4.10 Dynamic resizing — grow, shrink, charge

**Resolution:** `A_SetSize(newradius, newheight, testpos)` — `p_actionfunctions.cpp:5062-5094`.
It unlinks, writes `radius`/`Height`, relinks, and with `testpos=true` reverts if the new size
doesn't fit (`:5081-5088`). Call it every tic if you like; it is an unlink/relink, the same cost
`SetOrigin` already pays. Pair with the actor's `Scale` vector for the visual — nothing in the
collision path reads `Scale`.
A "charging" shield is: a counter, `A_SetSize(baseR + charge*k, baseH + charge*k2, false)`, and a
matching `Scale`. A "damaged" shield shrinks the same way as its health drops.
**Trap:** `testpos=false` lets the plate grow into a wall. For a hand-held plate that is correct
(the hand decides). For a *deployed* barrier use `testpos=true`.
**Feel knobs:** growth rate; whether growth is clamped by `testpos`; whether the drawn model scales
on the psprite in lockstep (it must, or the mismatch of §3.3 becomes visible).
**Label: EXISTS. Cost: S.**

### 4.11 Durability and breaking

**Resolution:** the plate is `+SHOOTABLE` with `Health <durability>`. Every missile that hits it
runs `P_DoMissileDamage` → `P_DamageMobj` (`p_map.cpp:1352-1384`, call site `:1707`) → the plate's
ZScript `DamageMobj` override (`actor.zs:801`), which is your one hook for "the shield was hit":
decrement, play a sound, kick the psprite, scale damage by `DamageType` via `DamageFactor`, and
return 0 so nothing standard happens. When `health <= 0`, move the shield weapon to a `Break` state
and destroy the plate.
**Note:** `+SHOOTABLE` is also what makes it block hitscans (§4.2), so durability and bullet
blocking arrive together — you cannot have one without the other, and that is fine.
**Feel knobs:** health; per-`DamageType` factors; whether a *parried* hit costs nothing (return
early from `DamageMobj` while the window is open — this is the strongest reason to parry);
regeneration when not hit for N tics; a cracked-model swap at health thresholds.
**Label: BUILDABLE. Cost: S.**

### 4.12 Sword-and-board — blade-on-shield, and bash-while-aiming

Now available because the **weapon hand is readable too** (§2.3), which the original baseline said
it was not.

**Player:** holds a weapon in the main hand and a shield in the offhand, and can use both at once.
**Detection:** both poses, every tic, gated on `OverrideAttackPosDir` (`actor.zs:276`):
`AttackPos`/`AttackAngle`/`AttackPitch`/`AttackRoll` and `OffhandPos`/`OffhandAngle`/`OffhandPitch`/
`OffhandRoll` (`actor.zs:268-275`).
**What it unlocks:**
- **Real blade-on-shield geometry.** `(AttackPos - OffhandPos)` and the angle between
  `AttackAngle` and `OffhandAngle` tell you whether the sword is resting on the shield, crossed
  behind it, or clear of it — so a two-handed brace stance, a sword-over-shield guard, or a
  "you just clipped your own shield" clang are all measurable, not guessed.
- **Bash while aiming.** The shield bash (§4.7) uses `LAF_ISOFFHAND` and the gun uses the main
  hand; they are separate origins in `P_LineAttack` (`p_map.cpp:4784-4799`) and separate psprite
  layers (`PSP_OFFHANDWEAPON`, baseline §2). Nothing couples them. Bash and fire in the same tic
  works.
- **Two shields.** Nothing in §4.1-§4.11 is offhand-specific except which pose field it reads.
  A second plate on `AttackPos` with `AttackAngle` is the same code with four identifiers changed.
**Feel knobs:** the hand-separation distance that counts as "braced"; whether bracing widens the
plate (§4.10) or lengthens the parry window (§4.4); whether a main-hand swing that intersects the
shield plate is suppressed.
**Label: BUILDABLE. Cost: M.** (The primitives are all S; the cost is the stance state machine.)
**Caveat 1:** `AttackPos`/`AttackAngle` are also what the engine aims the player's gun with
(`p_map.cpp:4794-4798`). Reading them is free; nothing here writes them, and they are
`native readonly` anyway (`actor.zs:268-271`).
**Caveat 2 — this mechanic is where §2.4 bites hardest.** "Bring the shield across to guard the
sword arm" *is* hands-inside-0.50 m, and if the player's thumb is on the offhand grip that sets
`weaponStabilised` (`gl_openvr.cpp:1717-1728`). Hosted as an offhand `Weapon`, the shield
disappears at the exact instant the stance is adopted. Hosted as a plate + overlay layer it does
not. If the main-hand weapon is a real `Weapon` and the player *wants* two-handed stabilisation
for it, that still works and is unaffected — the suppression only ever targets the offhand weapon.

---

## 5. Walls

1. **The collision box cannot be rotated.** Nothing in `PIT_CheckThing` (`p_map.cpp:1417-1419`) or
   `FTraceInfo::ThingCheck` (`p_trace.cpp:695-696`) reads `Angles.Yaw`. *Missing piece:* an
   oriented-box or plane collider on `AActor`. *Substitute:* `CanCollideWith` for everything except
   hitscans; accept the box for hitscans.
2. **`CanCollideWith` is not called on the hitscan path.** `p_trace.cpp` / `CheckForActor`
   (`p_map.cpp:4680-4700`) never invoke it. So the shape-carving trick works for projectiles and
   bodies but not for bullets. *Missing piece:* a script hook in `CheckForActor`.
3. **No actor blocks line-of-sight.** `src/playsim/p_sight.cpp` (967 lines) contains no actor
   iteration. Consequence: shields do not break aggro, do not stop `P_CheckMeleeRange`
   (`p_enemy.cpp`, `P_CheckMeleeRange`), and do not stop splash (`p_map.cpp:6585`, `:6448`).
   *Missing piece:* an actor pass in `p_sight.cpp`.
4. **Hitscans cannot be reflected.** There is no actor to redirect and `CheckForActor` offers only
   Stop/Skip. *Missing piece:* a continue-the-trace-from-here callback.
5. **`SpecialMissileHit` is on the missile, not the shield** (`p_mobj.cpp:3434`, call site
   `p_map.cpp:1655`). A shield mod cannot use it against projectiles it does not own. The
   shield-side hooks are `CanCollideWith` and `DamageMobj`; plan around those two.
6. **The `+SHIELDREFLECT` cone is a hardcoded 45°** (`p_mobj.cpp:3471`). No property, no cvar.
   *Substitute:* plain `+REFLECTIVE` plus your own cone in `CanCollideWith`.
7. **No haptic call bound to ZScript** — but see §3.5, because the picture is better than the
   baseline says and the plain statement is: *you cannot fire a haptic pulse directly, you can
   only provoke one.* A shield impact is the single most deserving haptic event in this survey.
   **The highest-value engine addition for this topic, by a wide margin, is a ZScript binding for
   `VRMode::Vibrate(duration, channel, intensity)`** (`hw_vrmodes.h:93`) — one thunk, no new
   state, usable by every mod in the family, not just shields.
8. **No hand velocity field** and 35 Hz sampling (`_BASELINE_FACTS.md` §1b, §3). Every gesture in
   §4.4, §4.7, §4.9 and §4.12 is built on two stored samples. *Not* a wall — just the whole reason
   the thresholds have to be generous and double-confirmed. This one survived the 10-30 baseline
   correction unchanged: **both** hands are readable, but neither reports a velocity.
9. **The plate does not sweep.** `SetOrigin` teleports (`p_maputl.cpp:563-571`); there is no
   swept-volume move anywhere in the actor API. At parry speeds (§3.1) the plate steps over a
   14-23 cm band each tic. *Missing piece:* a continuous-collision move. *Substitute:* do fast
   interactions with a `BlockThingsIterator` sweep in script rather than through the plate.
10. **Polyobjects are not reachable as a runtime collider.** No ZScript API; `Polyobj_MoveTo`
    (`src/playsim/actionspecials.h:89`) via `Level.ExecuteSpecial` (`doombase.zs:489`) is
    speed-based and needs a map-authored polyobject. Dead end for hand tracking; worth knowing
    only so nobody spends a day on it.
11. **An offhand `Weapon` cannot be kept alive through two-handed stabilisation.**
    `WF_TWOHANDSTABILIZED` is re-asserted on every state set while `weaponStabilised` is true
    (`p_pspr.cpp:507-515`), so clearing it from script does not stick, and the trigger has no
    `bTwoHanded` gate (`gl_openvr.cpp:1717-1728`). *Missing piece:* either a `bTwoHanded` /
    opt-out test at `gl_openvr.cpp:1721`, or a weapon flag that exempts a weapon from
    stabilisation suppression. *Substitute, and it is a good one:* don't be a weapon — §2.4.
12. **Haptic duration, channel and intensity are literals.** `Vibrate(150, …, 0.8)` at
    `p_map.cpp:4768` and `p_mobj.cpp:7555`; nothing varies them. *Missing piece:* a ZScript
    binding for `VRMode::Vibrate` (`hw_vrmodes.h:93`) — **the single highest-value engine addition
    for this topic**, one thunk, no new state, immediately useful to every mod in the family.
    Also: `VR_HapticEvent` is an empty stub (`qzdoom_common.cpp:65-67`) and `QzDoom_Vibrate` has
    no in-tree definition, so the OpenXR haptic path is unproven here.

**Not a wall, recorded so nobody re-derives it:** the plate's one-tic positional lag is inherent
(the `Prev` snapshot at `src/p_tick.cpp:151-162` is per-tic by design) but it is **smooth**, not
juddery, and §3.1 prices it at a hand's thickness during a block. The answer is the psprite/plate
split of §3.2, not an engine change.

---

## 6. Smallest prototype

One plate actor + one psprite overlay layer (**not** an offhand `Weapon` — §2.4). Should be ~120
lines of ZScript and no new art beyond a cube.

1. `class RSQ_ShieldPlate : Actor` — `+SOLID +NOGRAVITY +NOTAUTOAIMED +NEVERTARGET +FORCEINFIGHTING`,
   `RenderStyle "None"`, `Radius 10`, `Height 20`. **No `+DONTINTERPOLATE`, no `+NOINTERACTION`**
   (§4 preamble). In `Tick()`: `if (owner) SetOrigin(owner.OffhandPos, true);` plus copy
   `OffhandAngle` into `angle`.
2. **Not** a `Weapon` (§2.4). An inventory item or an event handler spawns the plate and gives the
   player a psprite overlay layer for the model whose caller is **not** `player.OffhandWeapon`.
   Gate the whole thing on `owner.OverrideAttackPosDir` (`actor.zs:276`) — that is the flag that
   says the pose fields are real hand poses, not the non-VR fallback of `hw_vrmodes.cpp:440-444`.
2a. **Prove §2.4 in the prototype, before building on it.** With `vr_two_handed_weapons 1`, hold
   the offhand grip with the hands together and confirm the shield still draws and still blocks.
   On an OpenXR runtime it will pass trivially (nothing sets `weaponStabilised` there); set
   `weaponStabilised` by hand under OpenVR, or just confirm the layer's caller is not the offhand
   weapon, which is what `hw_weapon.cpp:832` actually tests.
3. **Verify blocking first, before anything else:** stand in front of an imp, hold the offhand in
   the fireball's path, confirm the fireball dies on nothing visible. If it does not, the bug is
   almost certainly `SetXYZ` instead of `SetOrigin`, or `+NONSHOOTABLE` inherited from a parent
   class (`p_map.cpp:1595`).
4. **Then verify the mismatch number:** log `(OffhandPos - prev).Length() * 35` each tic while
   blocking normally. If it sits under ~70 units/s you are in the 2 m/s row of §3.1 and the plate
   is solid; if it routinely exceeds 170 you need the §4.4 iterator path, not the plate.
5. Add the parry: `bReflective = bShieldReflect = bAimReflect = true` for 6 tics on the offhand
   trigger, off after. The Centaur (`centaur.zs:34-39`) is the reference implementation; copy its
   shape.
6. Add `+SHOOTABLE` + `Health` + a `DamageMobj` override last, and only then deal with the
   self-shot trap of §4.2.

Sword-and-board (§4.12) is step 7 and changes nothing above it: it is the same plate class reading
`AttackPos`/`AttackAngle` instead of the offhand pair.

Steps 1-3 are the whole central problem. If they land, everything in §4 is additive.

---

## 7. Open questions

1. **Is `OffhandPos` portal-correct?** It is computed as `player->mo->X() - offset*34`
   (`hw_openxrdevice.cpp:605-607`), a flat subtraction with no portal-group handling, while
   `LinkToWorld` does `CollectConnectedGroups` (`p_maputl.cpp:509`). A shield reached through a
   line portal is untested here. I did not find any portal adjustment on the offhand path.
2. **What is the real hand-speed distribution for this owner?** Every threshold in §4.4/§4.7/§4.9
   is derived from the unit conversion, not from measurement. The §6 step-4 log should set them.
3. **Does the plate disturb `P_CheckPosition` for the player in tight corridors?** `ThruBits`
   skips the pair at `p_map.cpp:1412`, which should settle it, but the plate also enters
   `touching_sectorlist` (`p_maputl.cpp:496`) and triggers sector specials. Untested.
4. **Does the offhand psprite model and the plate agree about where the shield's *centre* is?**
   `GetWeaponTransform(VR_OFFHAND)` applies `scale(-1,1,1)` for one hand
   (`hw_vrmodes.cpp:467-470`), and MODELDEF offsets are authored in the model's own frame. The
   plate offset along `OffhandAngle` has to be measured against the drawn model, not guessed.
5. **Can a parried missile's `target` be re-pointed cleanly for kill credit?**
   `P_ReflectOffActor` sets `mo->target = blocking` (`p_map.cpp:3875`); whether overwriting it in
   the plate's `Tick` the following tic gives the player the frag is untested.
6. **Is `A_SetSize` safe every tic for an actor that is also being `SetOrigin`'d every tic?** Both
   unlink/relink (`p_actionfunctions.cpp:5074-5078`, `p_maputl.cpp:565-567`); doing both in one
   tic is two full relinks. No correctness problem found, but the cost was not measured.
7. **`+ALLOWTHRUBITS` on the player side.** Setting `ThruBits` on `PlayerPawn` is a mod touching
   the player class, which may conflict with other mods in the family. Is a shared `ThruBits`
   allocation needed across RSQ mods?
8. **Does the weapon-hand plate (§4.12) fight the aim?** `AttackPos` is the origin of every
   `P_LineAttack` the player makes (`p_map.cpp:4794-4798`). A second plate parked at `AttackPos`
   would sit exactly on the muzzle. Offsetting it along `AttackAngle` is the obvious fix, by how
   much is a measurement, not a guess.
9. **Is `weaponStabilised` reachable on the OpenXR path by any route I did not find?** I grepped
   the whole tree: the only writes are `gl_openvr.cpp:1723` and `:1726`. If the Quest-side native
   layer (outside this repo) also sets it, the §2.4 trap is live on VDXR today and the
   plate+overlay hosting goes from "right answer" to "mandatory".
10. **Is `QzDoom_Vibrate` defined anywhere outside this repository?** It is declared at
   `hw_openxrdevice.cpp:140` and called at `:757` with no definition in-tree. Until someone cites
   it, the §3.5 haptic thump is OpenVR-only and may be silent on the owner's actual setup.
11. **Does `LAF_NOINTERACT` with `damage = 1` really reach `p_map.cpp:4772` with no side effects?**
   The code path reads that way and the `nointeract` branches are all downstream, but it spawns a
   puff (`:4930-4972`) and I did not trace every `ALWAYSPUFF`/decal interaction. Verify in the
   engine before relying on it.
12. **Does `freezetics` interact badly with a `+SHOOTABLE` barrier?** The early return at
   `p_mobj.cpp:3948-3952` skips state advancement, so a frozen barrier cannot run a Pain or Death
   state until it is unfrozen. Clearing `freezetics` from `DamageMobj` should cover it; untested.
