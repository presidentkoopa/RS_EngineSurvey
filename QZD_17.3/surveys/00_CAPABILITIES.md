# 00 — Capability map

**Engine:** tag `qzd17.3` of `github.com/emawind84/gzdoom` — emawind84's QuestZDoom PC edition.
All citations are repo-relative paths at that tag.

**What this file is:** the shared foundation every topic report builds on. If you only read one
file before designing a mechanic, read this one. If a topic report contradicts it, the topic
report has to say so out loud and show its citation.

**What this engine is, in one line:** GZDoom with a real two-handed VR layer bolted into the
playsim — a second weapon, a second psprite layer, a second set of attack buttons, and exactly
one extra hand pose exposed to script.

---

## 0. Read this before you build anything — which VR backend fills the hand poses

Everything in this survey that reads a hand depends on four C++ globals: `weaponangles`,
`offhandangles`, `weaponoffset`, `offhandoffset`. In this source tree they are written in
**exactly one file**:

```
src/rendering/gl/stereo3d/gl_openvr.cpp:1331-1333   weaponangles
src/rendering/gl/stereo3d/gl_openvr.cpp:1341-1343   offhandangles
src/rendering/gl/stereo3d/gl_openvr.cpp:1733-1740   weaponoffset   (from dominantControllerPose)
src/rendering/gl/stereo3d/gl_openvr.cpp:1763-1770   offhandoffset  (from offhandControllerPose)
```

That is the **OpenVR / SteamVR** path, and on PC it is the *only* one:

1. **The OpenXR device file is not compiled on PC at all.** All of
   `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp` sits inside `#ifdef USE_OPENXR`
   (`:32` … `:792`), and `USE_OPENXR` is defined in **exactly one place in the whole tree**:
   `mobile/Android_src.mk:15` — the Android/Quest build. No CMake file defines it.
2. **There is no OpenXR input code in this tree either.** Zero hits for `xrSyncActions`,
   `xrLocateSpace`, `xrGetActionStatePose`, `xrGetActionStateFloat`. The OpenXR file only
   *reads* the hand globals (`extern` at `:116`, consumed at `:452-458`, `:586-611`); on Quest
   they are filled by the QzDoom native layer, which is **not in this repository**.
3. **`USE_OPENVR` is defined only under `option(ENABLE_OPENVR … NO)`** —
   `src/CMakeLists.txt:418-420`. It defaults to **off**.
4. **`VR_HapticEvent` is an empty function body** — `src/QzDoom/qzdoom_common.cpp:65-67`.
   (Haptics themselves *do* work on the OpenVR path — see §1.6 and §1.6a. It is this particular
   entry point that is a stub.)

**So there are exactly two shapes a working build can have:**

| Build | VR path | Hand poses filled by | Operative citations |
|---|---|---|---|
| **PC** | OpenVR / SteamVR, only if built `ENABLE_OPENVR=YES` | `gl_openvr.cpp:1331-1343`, `:1733-1770` | `src/rendering/gl/stereo3d/gl_openvr.cpp` |
| **Quest / Android** | OpenXR | QzDoom native layer, **absent from this repo** | `hw_openxrdevice.cpp`, unverifiable here |

**How to read the citations in this survey.** Several reports lead with
`hw_openxrdevice.cpp` line numbers because that file is the clearer read. Those are *parallel*
to the OpenVR ones and were checked against them — but **on a PC build the `gl_openvr.cpp`
citations are the operative ones**, and the OpenXR file is not even compiled. Where a report
gives only an OpenXR citation for something load-bearing, verify it against `gl_openvr.cpp`
before relying on it.

**What a team must confirm before committing to a hand-driven mechanic** — cheap, and it is the
first open question in several reports:

1. Which of `USE_OPENVR` / `USE_OPENXR` / neither the target binary was compiled with.
2. That `AttackPos` and `OffhandPos` actually move in the real build, and differ from each
   other. Print them from `DoEffect` for a few tics. If they are equal, you are on the non-VR
   fallback (`hw_vrmodes.cpp:440-444`) and every hand design in this survey reads the viewpoint.

This is not a reason to discard the survey — the engine capabilities documented here are real on
both paths. It is a reason to spend five minutes before spending a week.

---

## 1. VR data exposed to script

### 1.1 The complete list

There is nothing else. This is the whole surface.

| ZScript | Declared | Native | Thunk |
|---|---|---|---|
| `Actor.HmdPosition` — vector3, readonly | `wadsrc/static/zscript/actors/actor.zs:277` | `src/playsim/actor.h:1668` | `src/scripting/vmthunks_actors.cpp:2195` |
| `Actor.OffhandPos` — vector3, readonly | `actor.zs:272` | `actor.h:1661` | `vmthunks_actors.cpp:2191` |
| `Actor.OffhandPitch` — double, readonly | `actor.zs:273` | `actor.h:1662` | `vmthunks_actors.cpp:2192` |
| `Actor.OffhandRoll` — double, readonly | `actor.zs:274` | `actor.h:1664` | `vmthunks_actors.cpp:2193` |
| `Actor.OffhandAngle` — double, readonly | `actor.zs:275` | `actor.h:1663` | `vmthunks_actors.cpp:2194` |
| `Actor.OffhandDir(actor, angle, pitch)` — vector3 | `actor.zs:859` | `actor.h:1666` (function pointer) | — |
| `Actor.AttackPos` — vector3, readonly | `actor.zs:268` | `actor.h:1654` | `vmthunks_actors.cpp:2187` |
| `Actor.AttackPitch` — double, readonly | `actor.zs:269` | `actor.h:1655` | `vmthunks_actors.cpp:2188` |
| `Actor.AttackRoll` — double, readonly | `actor.zs:270` | `actor.h:1657` | `vmthunks_actors.cpp:2189` |
| `Actor.AttackAngle` — double, readonly | `actor.zs:271` | `actor.h:1656` | `vmthunks_actors.cpp:2190` |
| `Actor.OverrideAttackPosDir` — bool, readonly | `actor.zs:276` | `actor.h:1652` | `vmthunks_actors.cpp:2186` |
| `PlayerInfo.PlayInVR` — bool, readonly | `wadsrc/static/zscript/actors/player/player.zs:3136` | `src/playsim/d_player.h:502` | — |

### 1.2 Both hands are fully tracked, plus the head

> **Correction, 2026-10-30.** The first draft of this file said there was no main-hand pose.
> That was wrong. Three topic surveys independently caught it and it was then verified by hand.
> The error came from reading `hw_vrmodes.cpp:442-444` — the **non-VR fallback path**, where
> `AttackPos` is simply assigned the same value as `OffhandPos` and the player position — and
> mistaking it for the live VR path. If you read an early copy of this survey, re-read this
> section; it is the most load-bearing fact in the document.

The live VR paths write a real, independent weapon-hand transform from the controller,
exactly symmetric with the offhand:

```
hw_openxrdevice.cpp:586-597   AttackPitch/Angle/Roll/Pos  from weaponangles[] / weaponoffset[]
hw_openxrdevice.cpp:600-611   OffhandPitch/Angle/Roll/Pos from offhandangles[] / offhandoffset[]
gl_openvr.cpp:2525-2537       the OpenVR path, same shape
```
(full paths: `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp`,
`src/rendering/gl/stereo3d/gl_openvr.cpp`)

So you have:

- **Main hand** — `AttackPos`, `AttackAngle`, `AttackPitch`, `AttackRoll`, plus an `AttackDir`
  basis helper (`actor.h:1659`). Full 6DOF.
- **Offhand** — `OffhandPos`, `OffhandAngle`, `OffhandPitch`, `OffhandRoll`, plus `OffhandDir`
  (`actor.zs:859`). Full 6DOF.
- **Head** — `HmdPosition`. Your reference frame for everything body-relative: guard height,
  duck, lean, holster slots, body shots vs head shots.

**Two independently tracked hands and a tracked head.** Boxing with both fists, two-hand
handlebars, a bow drawn between two hands, hand-over-hand climbing, sword-and-board — none of
these are blocked by the input layer.

**Gate every hand-reading path on `OverrideAttackPosDir`** (`actor.zs:276`). It is set at
`hw_vrmodes.cpp:438` to `!puristmode && (IsVR() || vr_override_weap_pos)`, and when it is false
you are not in VR and both hand poses collapse to the viewpoint. The engine branches on it
itself throughout `src/playsim/p_map.cpp` (`:4597`, `:4647`, `:4784`, `:4888`, `:5185`, `:5219`,
`:5649`) and `src/playsim/p_actionfunctions.cpp:1302`.

### 1.2a Units — get this wrong and every threshold is wrong

Converting a hand position to real-world metres is **not** one constant:

- **Horizontal (X, Y):** divide by `vr_vunits_per_meter` = 34.0 (`hw_vrmodes.cpp:94`).
- **Vertical (Z):** divide by `vr_vunits_per_meter` **and additionally multiply by
  `level.pixelstretch`**, because the hand transform divides the up axis by pixelstretch
  (`gl_openvr.cpp:1282-1292`, read into `AttackPos.Z` at `:2529`). `pixelstretch` defaults to
  **1.2** (`src/gamedata/g_mapinfo.cpp:314`) and is script-readable
  (`wadsrc/static/zscript/doombase.zs:479`).

**Forget the vertical term and every height reads ~17% short** — a duck threshold fires late or
never, an uppercut measures as a jab, a bow draw reads short. Horizontal lean and steering
angles are unaffected, which is why this hides so well: the signal you test first looks fine.

Timing: 35 tics/s, so 1 unit/tic ≈ 1.03 m/s ≈ 2.30 mph.

### 1.2b VR cvar defaults you must design against

| Cvar | Default | Where | Why it matters |
|---|---|---|---|
| `vr_two_handed_weapons` | **true** | `hw_vrmodes.cpp:119` | the two-hand trap is ON (gotcha 10a) |
| `vr_crouch_use_button` | **true** | `hw_vrmodes.cpp:122` | physical duck is view-only (gotcha 4) |
| `vr_momentum` | **false** | `hw_vrmodes.cpp:120` | velocity is wiped each grounded tic (gotcha 1) |
| `vr_enable_haptics` | **true** | `hw_vrmodes.cpp:125` | rumble is available (§1.6a) |
| `vr_vunits_per_meter` | 34.0 | `hw_vrmodes.cpp:94` | the scale constant above |
| `r_quakeintensity` | **0.0** | `r_utility.cpp:116` | quake view shake off in VR (§1.6a) |

Note the pattern: **the three defaults most likely to break a mod are the restrictive ones.**

### 1.3 There are no velocity fields

Nothing above is a velocity. No hand speed, no head speed, no acceleration. Every punch, swing,
flick, draw and throw in this survey is built the same way:

> store `OffhandPos` (and/or `HmdPosition`) each tic in a small ring buffer; difference it.

Four samples is a generous history at 35 Hz. Budget for that everywhere.

### 1.4 Update rate: written per frame, read per tic

The poses are written from the **renderer**, not the playsim:

- `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:442-445`
- `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:600-603` (OpenXR path)
- `src/rendering/gl/stereo3d/gl_openvr.cpp:2549-2552` (OpenVR path)

Two consequences that pull against each other:

1. **The value you read is fresh.** It is the most recent rendered frame's pose, not a stale
   35 Hz sample. Thresholds behave better than you would expect.
2. **You still only sample at 35 Hz.** Motion between tics is gone for good. A fast punch is two
   to four samples; a fast sword swing may put the blade on the far side of a target without ever
   sampling inside it.

**The standard remedy, used all over this survey:** don't test the hand's *position*, test the
*chord* between the last two samples — trace along it. Any mechanic that tests only "is the hand
inside the target right now" will miss fast motion, which is exactly the motion players use.

### 1.5 Buttons

`player.cmd.buttons` is readable from script — `player.zs:3048` (`native readonly uint buttons`),
with `oldbuttons` at `player.zs:3064` for edge detection.

The engine adds real VR buttons to the standard set (`wadsrc/static/zscript/constants.zs`):

| Constant | Bit | Line |
|---|---|---|
| `BT_OFFHANDATTACK` | 1<<26 | `constants.zs:866` |
| `BT_OFFHANDALTATTACK` | 1<<27 | `constants.zs:867` |
| `BT_OFFHANDRELOAD` | 1<<28 | `constants.zs:868` |
| `BT_MAINHANDRELOAD` | 1<<20 | `constants.zs:869` |
| `BT_USER1` … `BT_USER4` | 1<<21 … 1<<24 | `constants.zs:859-862` |

Note `BT_SHOWSCORES` was set to 0 and its bit reused for `BT_MAINHANDRELOAD` (`constants.zs:857`).

**So you have per-hand attack, alt-attack and reload, plus four free user buttons.** That is a
lot of input, and several mechanics in this survey that look like they need gesture recognition
are better served by a button plus a pose check.

Per-hand button dispatch already exists in the player code: `CheckWeaponButtons(hand)` at
`player.zs:588` and `:2031`, selecting `BT_OFFHANDATTACK`/`BT_OFFHANDALTATTACK` at
`player.zs:473-474`.

### 1.6 Haptics — present in C++, not reachable from ZScript

- `VRMode::Vibrate(duration, channel, intensity)` — `src/common/rendering/hwrenderer/data/hw_vrmodes.h:94`
- `VR_HapticEvent(event, position, intensity, angle, yHeight)` — `hw_vrmodes.h:97`
- Intensity cvars `ext_haptic_level_*` — `src/playsim/p_actionfunctions.cpp:92-101`
  (global, damage_projectile, pickup_weapon, pickup, fire_weapon, poison, healstation,
  heartbeat, rumble)

**Zero hits for haptic / rumble / vibrate anywhere in `wadsrc/static/zscript`.**

But be precise about what is and is not missing — an early draft of this file overstated it:

- **Haptics genuinely work on the OpenVR path.** `OpenVRHaptics::Vibrate` is a real
  implementation (`src/rendering/gl/stereo3d/gl_openvr.cpp:517-528`), reached through the
  `VRMode::Vibrate` override (`gl_openvr.h:121-124`), gated on `vr_enable_haptics`.
- **`VR_HapticEvent` specifically is an empty stub** (`src/QzDoom/qzdoom_common.cpp:65-67`).
- **Neither is bound to ZScript.**

So the engine's own hardcoded buzzes — firing a weapon, taking projectile damage, picking
something up, per the `ext_haptic_level_*` cvars — **do fire** on PC/OpenVR. What a mod cannot
do is call for a rumble of its own: no punch-landed thump, no rising bow-draw tension, no
engine-rev vibration. Since the buzz is tied to firing a weapon, a mod can provoke one
indirectly, which is a HACK, not a capability.

Several reports name a ZScript binding for `VRMode::Vibrate` as the **highest-value engine
addition available** — the implementation already exists, so it is a binding, not a feature.

### 1.6a But there is a back door, and it is good: `A_QuakeEx`

Earthquake intensity is routed into the controllers, **one hand per axis**:

```
src/rendering/r_utility.cpp:1085-1091
  if (vr_quake_haptic_level > 0.0) {
      left  = QuakePower(vr_quake_haptic_level, jiggers.Intensity.X, ...);
      right = QuakePower(vr_quake_haptic_level, jiggers.Intensity.Y, ...);
      vrmode->Vibrate(10, 0, left);    // left hand
      vrmode->Vibrate(10, 1, right);   // right hand
  }
```

and the camera shake that would normally accompany it is **off by default in VR** —
`r_quakeintensity` is `0.0f`, commented "Defaulted to 0 for VR" (`r_utility.cpp:116`).

So a mod **can** fire independent per-hand rumble today, from the ordinary ZScript
`A_QuakeEx`, with no view motion: intensity X drives the left hand, Y the right.
It is a **HACK / S** — short 10 ms pulses, so re-fire per tic for sustained feel; depends on the
user's `vr_quake_haptic_level`; and a user who has raised `r_quakeintensity` gets shake with it.

> **Pass `damrad = 0`.** `A_QuakeEx`'s `thrustMultiplier` defaults to **0.5**
> (`actor.zs:1284`) and the thrust adds *randomised velocity to the player*
> (`a_quake.cpp:205-209`), inside a block gated on `dist < m_DamageRadius` (`:191`) that can
> also damage them (`:193-203`). With `damrad = 0` the whole block is skipped. Without it your
> haptic feedback shoves the player sideways. Also never pass `QF_GROUNDONLY` (`:354`) — it
> mutes airborne mechanics. And `QuakePower` is random (`r_utility.cpp:911-921`), so this is
> noise: fine for a thud, useless for a crisp click.

Full write-up, both routes compared, in `99_OVERLOOKED.md` §1a.

**And a second, simpler route — `LineAttack` (07_SHIELDS_PARRY found it).** `P_LineAttack`
buzzes the firing hand whenever a player deals damage:

```
src/playsim/p_map.cpp:4764-4769
  if (damage > 0) {
      long rightHanded = vr_control_scheme < 10;
      rightHanded = (flags & LAF_ISOFFHAND) ? 1 - rightHanded : rightHanded;
      vrmode->Vibrate(150, rightHanded ? 1 : 0, 0.8);
  }
```

It is **not** gated on `LAF_NOINTERACT`, so a trace that does nothing in the world still
buzzes. `LineAttack(angle, 0.1, pitch, 1, "None", "NullPuff", LAF_NOINTERACT|LAF_ISOFFHAND)`
gives a 150 ms / 0.8 pulse on the chosen hand and no other effect.

Compared with `A_QuakeEx`: this one **respects the user's handedness** (`vr_control_scheme`)
and needs no extra cvar, but duration and intensity are **literals** — every pulse feels the
same, so a light tap and a heavy block are indistinguishable. `A_QuakeEx` is shapeable but
cruder. Use `LineAttack` for a discrete thump, `A_QuakeEx` when you need an envelope.

Note also `p_map.cpp:4771-4774`: when `weaponStabilised` is set, the engine buzzes **both**
hands. Worth knowing before you read a double pulse as a bug.

### 1.7 VR cvars a mod can read

From `src/QzDoom/VrCommon.h:16-27`: `vr_control_scheme`, `vr_move_use_offhand`,
`vr_weaponRotate`, `vr_snapTurn`, `vr_ipd`, `vr_weaponScale`, `vr_teleport`, `vr_switch_sticks`,
`vr_secondary_button_mappings`, `vr_two_handed_weapons`, `vr_crouch_use_button`.

`vr_two_handed_weapons` and `vr_move_use_offhand` matter most — they are engine-level toggles
that change what the player's hands are already doing, so a mod should read them rather than
assume.

---

## 2. The sprite layer and hand-attached models

### 2.1 Two weapons, one per hand — this is shipped, not a hack

- `PlayerInfo.OffhandWeapon` — `src/playsim/d_player.h:384`
- Weapon flag `bOffhandWeapon` routes a weapon to that hand — `player.zs:1977-1998`
- Weapon flag `bTwoHanded` forcibly clears the offhand weapon — `player.zs:1989-1992`
- Ready / fire / down state selection is keyed on `hand` throughout:
  `player.zs:398`, `:431`, `:466`, `:2092`, `:2429`
- Morphing preserves it — `PremorphWeaponOffhand`, `d_player.h:410`

**Two independent weapon state machines, one per hand, each with its own psprite layer.**
This is the single most valuable feature in the engine for this project, and most topic reports
lean on it somewhere.

### 2.2 Psprite layers

From `wadsrc/static/zscript/constants.zs:798-801` (mirrored at `src/playsim/p_pspr.h:56-59`):

| Layer | Value |
|---|---|
| `PSP_WEAPON` | 1 |
| `PSP_FLASH` | 1000 |
| `PSP_OFFHANDWEAPON` | 1000000 |
| `PSP_TARGETCENTER` | `int.max - 2` |

Layers are integers, so you may create your own at any free index — the named ones are
conventions, not a fixed set. `GetPSprite(id)` / `SetPsprite(id, state)` take arbitrary ids.
The gap between 1 and 1000, and between 1000 and 1000000, is yours.

That gives each hand its own stack of independently positioned, independently animated overlay
layers. Per-layer position is script-writable each tic via the `PSprite` object.

### 2.3 Models — and who owns a layer decides everything

MODELDEF attaches models to states as normal, and the per-frame hand pose means a model on a
hand layer follows the real controller with no script work. But **the owner (`Caller`) of a
psprite layer, not its layer number, decides which hand it rides, whether it survives a weapon
switch, and whether it can be a model at all.** `11_HUD_MODELS.md` is the full report; these are
the rules.

**A non-weapon owner works.** A plain `Inventory` item can own a HUD-model layer — nothing in
the model path requires a `Weapon` (`hw_weapon.cpp:922`, MODELDEF has no weapon gate at
`models.cpp:750-756`, and `Actor.DefaultStateUsage = SUF_ACTOR|SUF_OVERLAY` at `actor.zs:458`
makes an item's states legal overlay states).

**But the hand choice is a two-way ternary with no third case:**
```
src/r_data/models.cpp:221
int hand = psp->GetCaller() == playermo->player->OffhandWeapon ? 1 : 0;
```
So anything whose caller is not literally the offhand weapon draws on the **main hand**.
A non-weapon-owned model on the **off hand is a WALL**. The flip side is useful: such a layer
can never satisfy the `weaponStabilised` skip (`hw_weapon.cpp:832`, `:917`), so it is
**immune to the two-hand trap** in gotcha 10a.

**No layer at or above `PSP_TARGETCENTER` can draw a model at all** —
`PrepareTargeterSprites` hardcodes `hudsprite.mframe = nullptr` (`hw_weapon.cpp:1058`), and the
model call site is gated on `mframe` (`:101-108`). That band is sprites only.

**`GetPSprite(id)` from ZScript cannot set an owner** — the binding has no caller parameter
(`player.zs:3142`), and `player_t::GetPSprite` defaults an ordinary layer's caller to
`ReadyWeapon` (`p_pspr.cpp:331-332`). Use `SetPsprite(id, state, pending, newcaller)`
(`player.zs:3140`) or your MODELDEF, keyed on the item, will never match.

**Survival across a weapon switch comes from the caller, not the band.** The cull at
`player.zs:559-563` gates its "weapon not currently selected" clause on
`pspr.Caller is "Weapon"`; a plain `Inventory` caller is only tested for *ownership*. So an
item-owned layer persists across weapon switches **at any layer id**, and dies only when the
item leaves inventory — or to `A_ClearOverlays`, which tests no caller at all.

Two more gotchas on a fresh layer: `GetPSprite` sets `PSPF_ADDBOB|PSPF_ADDWEAPON`
unconditionally (`p_pspr.cpp:355-365`), and `huds->my` is fed as a **rotation**
`(my - WEAPONTOP)/-4` (`hw_weapon.cpp:108`, `WEAPONTOP = 32`), so a fresh layer at `y = 0` is
pitched about +8°. Clear the two flags and set `psp.y = 32`.

---

## 3. Tracing and collision

### 3.1 `LineTrace` is the workhorse of this entire survey

```
wadsrc/static/zscript/actors/actor.zs:805
native bool LineTrace(double angle, double distance, double pitch, int flags = 0,
                      double offsetz = 0., double offsetforward = 0., double offsetside = 0.,
                      out FLineTraceData data = null);
```

`FLineTraceData` — `actor.zs:36-61`:

| Field | Use |
|---|---|
| `HitActor` | what you hit, if an actor |
| `HitLine` | the wall line — **this is how you get a wall normal** (line delta rotated 90°, sided by `LineSide`) |
| `HitSector` | floor/ceiling sector |
| `Hit3DFloor` | 3D floor hit |
| `HitTexture` | the texture — material-specific reactions |
| `HitLocation` | the exact point |
| `HitDir` | the trace direction |
| `Distance` | how far |
| `LineSide` | which side of the line |
| `LinePart` | top / middle / bottom |
| `SectorPlane` | which plane |
| `HitType` | one of `TRACE_HitNone / HitFloor / HitCeiling / HitWall / HitActor / CrossingPortal / HasHitSky` (`actor.zs:38-47`) |

**`HitLine` plus `LineSide` is the single most important detail here.** It means a mod can
recover a wall's facing at runtime, which is what makes wall-running, rope anchoring, shield
alignment and surface-aware impacts possible *without touching a map*. Several topic reports
turn on this.

### 3.2 Collision is a SQUARE prism, not a cylinder

> **Correction.** An earlier draft of this file said "cylinders", repeating the usual Doom
> folklore. The shields survey checked it and it is wrong.

The actor-vs-actor test is a **Chebyshev / axis-aligned box** check:

```
src/playsim/p_map.cpp:1417-1419
double blockdist = thing->radius + tm.thing->radius;
if (fabs(thing->X() - cres.Position.X) >= blockdist || fabs(thing->Y() - cres.Position.Y) >= blockdist)
    return true;   // no hit
```

with the same shape on the trace path (`src/playsim/p_trace.cpp:695-696`). So an actor of
`radius` r occupies a **2r × 2r square footprint**, not a circle — and it is **unrotatable**:
nothing in that test reads `Angles.Yaw`.

Consequences:
- A "shield plate" blocks a square, and you cannot turn it to face anywhere.
- Diagonal reach is √2 larger than you expect.
- Exceptions worth knowing: `projectilepassheight` (missile-only Z window,
  `p_map.cpp:1620-1643`) and `+ACTLIKEBRIDGE`.

`07_SHIELDS_PARRY.md` §4 works through what to do about it, and concludes one box plus
`CanCollideWith` beats a grid of small actors — a 3×3 grid costs nine `P_FindFloorCeiling`
sweeps a tic to buy a shape that is still wrong in orientation.

---

## 4. Time

- `Actor.isFrozen()` — `wadsrc/static/zscript/actors/actor.zs:513`
- `PlayerInfo.timefreezer` — `player.zs:3079`, an int bitmask, not a boolean
- `PowerTimeFreezer` — `wadsrc/static/zscript/actors/inventory/powerups.zs:1502`, setting and
  clearing `timefreezer` bits at `:1529`, `:1537`, `:1604`
- Level clocks: `Level.time`, `Level.maptime`, `Level.totaltime`, `Level.starttime` —
  `wadsrc/static/zscript/doombase.zs:425-428`, all readonly

`timefreezer` is a **mask**, not a flag — the freeze is already expressed per-player in bits.

**`10_TIME.md` settled this topic, and the answer is better than expected.** Headlines, so other
reports can rely on them:

- **`Actor.freezetics`** (`actor.zs:265`, native `actor.h:1284`) is writable and checked at the
  *top* of `AActor::Tick` (`p_mobj.cpp:3948-3952`), before states, velocity, inventory and AI.
  Driven as a duty cycle (freeze N tics, run 1) it slows an actor by exactly 1/(N+1) with
  animation, velocity and AI **inherently in sync** — you never touch `tics` or `Vel`, so the
  classic desync trap never arises. A frozen actor **stays in the blockmap**, so it remains
  shootable and collidable.
- **Global freeze with exceptions is O(1) and already shipped:** `Level.SetFrozen()`
  (`doombase.zs:511`) plus the writable `player.timefreezer` bitmask (exemption at
  `actorinlines.h:186-190`) plus per-actor `+NOTIMEFREEZE` (`actor.h:315`). A mod does not need
  `PowerTimeFreezer` at all.
- **A global time scale exists but is a WALL for mods:** `i_timescale` (`d_main.cpp:217-234`,
  applied at `i_time.cpp:67-72`) cannot be set from script — `CVar.SetFloat` throws outside menu
  code (`vmnatives.cpp:981-990`) and there is no script `ConsoleCommand` in `src/`. Per-actor
  scaling gets you there anyway, quantised to 1/2, 1/3, 1/4…
- **Freezing the playsim does not stutter the VR view.** Head orientation is re-derived from the
  live pose every rendered frame and written into the viewpoint *after* interpolation
  (`hw_openxrdevice.cpp:736-751`). Player translation and slowed actors stutter; your head does not.

### 4.1 Where to inject per-tic logic

`WorldTick` runs after all player input processing and before `P_XYMovement` / `P_ZMovement`
(`src/p_tick.cpp:166-172` → `p_mobj.cpp:4344`, `:4372`), so `Vel` / `bFly` / `bNoGravity` /
`Gravity` writes land the same tic. `TickPSprites` (`player.zs:1796`) is the other clean hook.
Thinkers at `STAT_BOT` (104) tick after all actors (`src/playsim/dthinker.cpp:145-149`) if you
need a controller that runs last.

**Trap:** thinker iterators only visit statnums ≥ 32 (`dthinker.cpp:938-940`), so actors parked
at a low statnum vanish from your own sweep loops.

---

## 5. Camera and view

- `SetCamera(Actor cam, bool revert)` — `actor.zs:941` (action function)
- `PlayerInfo.camera` — `player.zs:3110`
- `Actor.CameraHeight` — `actor.zs:249`
- `SetCameraToTexture(viewpoint, texture, fov)` — `doombase.zs:30`, plus
  `SetCameraTextureAspectRatio` at `:31` — render-to-texture, which is how you build a working
  mirror, a rear-view mirror in a vehicle, or a cockpit display

**The VR caveat:** the HMD supplies view orientation. How much of a script-driven view change
actually survives that is the question the mech and flight reports have to settle, and it is
flagged in both as decisive rather than incidental.

---

## 6. Rendering a mod can use

- **Post-process shaders** are a supported mod feature —
  `src/common/rendering/hwrenderer/postprocessing/hw_postprocessshader.cpp`, with console
  control at `hw_postprocessshader_ccmds.cpp`. Declared from GLDEFS, uniforms settable from
  script. This is the route for bullet-time tinting, damage vignettes, heat haze.
- Dynamic lights, translucency and render styles are stock GZDoom and behave normally.
- `SetCameraToTexture` (above) for in-world screens and mirrors.

---

## 7. Gotchas that bite every topic

These are the ones that silently eat a prototype. Several were found the hard way by topic
surveys; each names the report that proved it.

1. **`MovePlayer` wipes `vel.XY` every grounded tic.** It multiplies by 0.0001 unless
   `player.keepmomentum` is set — `player.zs:1373-1383`; `keepmomentum` is writable
   (`d_player.h:414`) and the engine sets it itself at `actions.zs:78`; cvar `vr_momentum`
   defaults false (`hw_vrmodes.cpp:120`). **Every momentum mechanic in this survey dies without
   this** — slides, dashes, rope swings, vehicles. Set it every tic. (09_MOVEMENT, 05_GRAPPLE)
2. **`A_ChangeVelocity` on the player is a no-op by default.** Gated on `vr_recoil`, default
   false — `p_actionfunctions.cpp:90`, gate `actions.zs:63-79`. Write `Actor.Vel` directly
   (`actor.zs:112`). (05_GRAPPLE, 04_FLIGHT)
3. **Script cannot change view pitch or roll in VR.** `vp.HWAngles.Pitch/Roll` are overwritten
   from the raw HMD after `R_SetupFrame` — `hw_entrypoint.cpp:117` then `:143` →
   `gl_openvr.cpp:1467,1471`; OpenXR `hw_openxrdevice.cpp:740-741`; view matrix built at
   `hw_drawinfo.cpp:443`. Screen shake, camera bob, dash roll and wall-run tilt are **WALLs in
   VR**, not design choices. (04_FLIGHT, 03_MECHS, 09_MOVEMENT)
4. **Room-scale Z: it depends on one cvar, and the default is the restrictive one.**
   Positional tracking is XY-only and restores Z (`gl_openvr.cpp:2613-2632`); head height is a
   render-only eye offset (`hw_openxrdevice.cpp:221`).
   - At the **default** `vr_crouch_use_button true` (`hw_vrmodes.cpp:122`) a physical duck is
     **view-only** — you can detect it from `HmdPosition`, but the collision volume stays put.
   - At `vr_crouch_use_button 0`, the engine drives `crouchfactor` from head height
     (`hw_openxrdevice.cpp:574-579`) and `PlayerPawn.Tick` sets
     `Height = FullHeight * crouchfactor` (`player.zs:150-153`, gated on `CanCrouch()`),
     so the volume **does** follow.
   Design for both, and read the cvar. (04_FLIGHT, 03_MECHS, 09_MOVEMENT)
5. **The 30 units/tic cap is CONDITIONAL — it usually does not apply.** `MAXMOVE` is 30
   (`src/playsim/p_local.h:61`), but the clamp at `p_mobj.cpp:2096-2106` only runs when
   `(player && COMPATF_WALLRUN)` **or** `((waterlevel >= 1 || crouchfactor < 0.75) && !MF8_NOFRICTION)`.
   `compatmode` defaults to 0 (`d_main.cpp:621`, `:694`), so `COMPATF_WALLRUN` is **off** by
   default. Standing, dry, default compat → **no 30-unit cap**. The unconditional ceiling is
   `VELOCITY_THRESHOLD = 5000` units/tic (`p_mobj.cpp:2107-2111`).
   At `vr_vunits_per_meter` 34 and 35 tics/s, 1 u/tic ≈ 1.03 m/s ≈ 2.30 mph, so 30 u/tic ≈
   111 km/h — the cap would not have blocked a car chase even if it did apply, but know that it
   usually doesn't. Defend anyway: crouching below 0.75 re-enables it.
   Tunnelling is already handled — `P_XYMovement` substeps at `1 + speed/(radius-1)` with a full
   `TryMove` each (`p_mobj.cpp:2166-2192`). Real artefact: a blocked substep **aborts the rest of
   the move** (`:2283-2285`), so a glancing hit stops a fast actor dead instead of deflecting it;
   restore the tangential component yourself. Airborne friction is skipped
   (`p_mobj.cpp:2447-2463`), so air movement is lossless — but only at the default
   `sv_aircontrol`; raising it flips `airfriction` off 1.0 (`g_level.cpp:1928-1931`) and makes
   every jump lossy. Override `ApplyAirControl` (`player.zs:1325`) instead.
   (02_VEHICLES, which corrected this; 09_MOVEMENT)
6. **No velocities — you difference positions.** (§1.3)
7. **35 Hz sampling of a per-frame value.** Fresh, but sparse. Test chords, not points. (§1.4)
8. **Haptics are not scriptable.** `VR_HapticEvent` is an empty stub at
   `src/QzDoom/qzdoom_common.cpp:64`. Any design whose feel depends on rumble is on thin ice,
   and this is named across several reports as the highest-value engine addition. (§1.6)
9. **Collision is cylinders.** No shape you draw is the shape that blocks. (§3.2)
10. **`bTwoHanded` clears the offhand weapon** (`player.zs:1989-1992`) — a two-handed weapon
    and an offhand gadget cannot coexist without working around this. Its *only* effects are
    nulling the other hand's weapon (`player.zs:1977-1993`) and blocking paired hand-swap
    (`:2498`); it does not enable stabilisation and does not touch rendering. (08_BOWS)
10a. **THE TWO-HAND TRAP — read this before designing any two-handed mechanic. It is on by
    default.** `vr_two_handed_weapons` defaults **true** (`hw_vrmodes.cpp:119`) and its
    stabilisation is **not** gated on `bTwoHanded`. If the player presses the offhand **grip**
    button while the hands are within **0.50 m**, the engine sets `weaponStabilised`
    (`gl_openvr.cpp:1716-1727`) and the offhand weapon goes dark: it fires nothing
    (`player.zs:393-396`, `:426`), checks no buttons (`p_pspr.cpp:755-757`), and renders
    nothing (`hw_weapon.cpp:832-835`, `:917-920`). `WF_TWOHANDSTABILIZED` is **re-asserted on
    every state set** (`p_pspr.cpp:512-515`, the `|=` at `:514`), so script cannot durably
    clear it.
    **The script-visible flag is only as fresh as the last weapon state change.**
    `WF_TWOHANDSTABILIZED` is cleared and re-asserted in exactly one place —
    `DPSprite::SetState` (`p_pspr.cpp:500-515`) — and only when the layer is `PSP_WEAPON` or
    `PSP_OFFHANDWEAPON`. The C++ `weaponStabilised` updates per frame; the bit script reads
    updates only when a weapon layer changes state. Usually that is every tic or few, so a
    gate on it releases a few tics after the grip. But a weapon parked in a long or `-1`
    duration frame can hold the bit **stale**, so never build a timeout that assumes it will
    clear on its own. (Found while implementing the TacticalRig and LaserSight gates.)
    Three details that make it nastier, found by 02_VEHICLES:
    - It latches on the **rising edge** of the grip (`gl_openvr.cpp:1718-1720`), and releasing
      grip clears it with **no distance test** (`:1724-1726`). So moving your hands apart while
      still holding grip does **not** release it — the failure looks arbitrary.
    - The 0.50 m is **raw tracking-space metres** between controller poses
      (`gl_openvr.cpp:1709-1714`) — a true physical 50 cm, no pixelstretch involved, so you can
      check a design against real hardware dimensions.
    - Which means: **motorcycle bars (0.65–0.85 m) are safe by construction; a car steering
      wheel is not** — a 0.37 m wheel puts hands at 9-and-3 about 0.36 m apart, inside the trap.
    **It is OpenVR-only in this tree.** `weaponStabilised` is a single global
    (`qzdoom_common.cpp:19`) and its only writes anywhere are `gl_openvr.cpp:1723` and `:1726`;
    the OpenXR path never sets it. So on a Quest/OpenXR build it does not fire **today** — but
    it is one patch from doing so, and "silently broken under SteamVR" is still broken. Design
    as though it fires.
    **The robust defence is not geometry, it is to use no offhand weapon at all** — drive the
    mechanic from `AttackPos` / `OffhandPos`, which the renderer writes and `weaponStabilised`
    never touches, and take grab state from `BT_OFFHANDATTACK` (the trigger) rather than the
    grip. With no offhand weapon the latch has nothing to switch off.
    **The render skip tests the CALLER, not the layer id** — both skips are
    `if (weaponStabilised && psp->GetCaller() == player->OffhandWeapon) continue;`
    (`hw_weapon.cpp:832`, `:917`), inside loops that cover every layer below
    `PSP_TARGETCENTER` (`:830`, `:915`).
    **But "use an overlay layer and you are safe" is WRONG, and the distinction is where a
    real shipped bug lives.** `A_Overlay` creates its psprite with
    `P_CreatePsprite(player, stateowner, layer)` (`p_pspr.cpp:1241`), so an overlay fired from
    an offhand weapon's own state gets **that weapon as its caller** and is skipped exactly
    like the weapon's main layer — whatever layer number you chose. Only a psprite given an
    **explicit different `newcaller`** through `P_SetPsprite` / `GetPSprite(layer, newcaller)`
    escapes the skip.
    So the safe patterns are: a world actor, or a psprite whose caller is deliberately set to
    something that is not the offhand weapon. Not merely "a different layer".
    (Found by 03_SABER_AND_SAW, where a blade on `A_Overlay` layer 2000052 vanishes under the
    grip while still cutting; verified here.)
    Where you genuinely need an offhand weapon, detect
    `player.WeaponState & WF_TWOHANDSTABILIZED` and warn; never try to clear it.
    (08_BOWS found it; 02_VEHICLES §4.3a and 07_SHIELDS_PARRY §2.4 worked out the defences)
11. **`+NOINTERACTION` is a trap** — it forces `MF_NOBLOCKMAP` (`p_mobj.cpp:3988-3992`), taking
    the actor out of the world. To hold something still *and* keep it collidable, use
    `freezetics` (`actor.zs:265`), which early-returns out of `Tick` (`p_mobj.cpp:3948-3952`)
    but stays in the blockmap. (06_GRAB, 10_TIME)
12. **Interpolation cuts both ways — know which side you are on.** `Prev` is re-snapshotted for
    every actor at the top of every tic, before any thinking (`src/p_tick.cpp:151-162`;
    `ClearInterpolation()` is `Prev = Pos()`, `actorinlines.h:29-35`).
    - **If you write a position every tic** (a held object, a shield plate, a puppeted actor),
      this is your friend: the write renders smoothly. Use `SetOrigin(pos, true)`.
      `SetOrigin(pos, false)` (`p_maputl.cpp:570`) and bare `Warp` (`p_things.cpp:805`) call
      `ClearInterpolation` and **snap**. (06_GRAB, 07_SHIELDS)
    - **If you make an actor move intermittently** (a `freezetics` duty cycle for slow-motion),
      the same mechanism is why it *strobes*: it moves a full tic's distance on its one live
      tic, and the renderer lerps that whole step across one tic of frame time instead of
      spreading it. Step-hold-hold-hold, roughly 8.75 Hz judder at factor 4 — bad in a headset.
      The missing engine piece is per-actor interpolation stretch. (10_TIME §5.2, which gives
      the three-tier workaround)
13. **This is not REMA.** `E:\DOOMWork\UZDXREMA` has a far larger VR surface. Anything you
    remember from there and cannot find here is a WALL here.

---

## 8. How to read the rest of this survey

`README.md` is the index: a flat table of every mechanic across all ten reports, with its label,
its cost, and the file and section that holds the design. Go there first, then to the section.

Labels: **EXISTS** (already in the engine, cited) · **BUILDABLE** (script, with named APIs) ·
**HACK** (works, but abuses something — the report says what breaks) · **WALL** (needs engine
changes — the report names the missing piece).

Costs: **S** an afternoon · **M** a few days · **L** a week or more.
