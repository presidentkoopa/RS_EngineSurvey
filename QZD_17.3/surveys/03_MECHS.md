# 03 — MECHS / GIANT ROBOTS

**Engine surveyed:** tag `qzd17.3` of `github.com/emawind84/gzdoom` (QuestZDoom PC edition).
All paths repo-relative at that tag. Builds on the **corrected** `_BASELINE_FACTS.md` (§1a rewrite:
both hands are readable). **§0 records where I disagree with an inference handed to me.**
**Constraint honoured:** sprite layer only — no map edits, no world-geometry changes. Spawned
actors are treated as "things", not geometry, and are flagged where used.

---

## 0. Corrections taken on board, and one disagreement

**Taken on board (verified, not re-derived):**

1. **Both hands are readable.** `Actor.AttackPos / AttackPitch / AttackRoll / AttackAngle`,
   `native readonly` (`wadsrc/static/zscript/actors/actor.zs:268-271`), thunked at
   `src/scripting/vmthunks_actors.cpp:2187-2190`, filled from the real main-hand controller at
   `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:586-597` (OpenVR:
   `src/rendering/gl/stereo3d/gl_openvr.cpp:2525-2537`). Gate on `OverrideAttackPosDir`
   (`actor.zs:276`, set `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:438`). I confirm
   `hw_vrmodes.cpp:437-444` is the **non-VR fallback** — it sets `AttackPos == OffhandPos` from the
   viewpoint, and the OpenXR block overwrites both with real controller transforms. **This changes
   §4.4: cockpit interaction is genuinely two-handed.**
2. **Script cannot change view pitch or roll in VR** — I reached this independently
   (`hw_openxrdevice.cpp:740-741`, `:750`; view matrix from `HWAngles` at
   `src/rendering/hwrenderer/scene/hw_drawinfo.cpp:443`); the flight agent's OpenVR citations
   (`gl_openvr.cpp:1464`, `:1467`, `:1471`) and the entrypoint ordering
   (`hw_entrypoint.cpp:117` then `:143`) agree. Screen shake / camera bob / stomp-kick are **WALLs**
   (W2, W10), not design choices. §4.7 is written that way.
3. **Pitch clamp:** `MinPitch`/`MaxPitch` = ±`maxviewpitch`, default 90° on the hardware renderer
   (`src/playsim/p_user.cpp:682-684`, `src/rendering/r_utility.cpp:1240`); the 32/56 figures are
   software-renderer only (`p_user.cpp:665-666`). Both are writable ZScript doubles
   (`player.zs:3124-3125`). Not load-bearing for a mech, but it means a mod *can* restrict the
   pilot's pitch — and must not, since in VR pitch comes from the neck.
4. **The persistent psprite band** at/above `PSP_TARGETCENTER` is real and is a genuinely useful
   find — see §2.6a, with one correction to its intended use.
5. **The two-hand stabilisation trap is real and it lands on my cockpit controls.** Verified:
   `vr_two_handed_weapons` (default **true**, `hw_vrmodes.cpp:119`) + an offhand **grip** edge +
   hands within **0.50 m** sets `weaponStabilised` (`gl_openvr.cpp:1716-1727`, distance test
   `:1722`). The offhand then fires nothing (`player.zs:393-396`), checks no buttons
   (`p_pspr.cpp:755-758`), renders nothing (`hw_weapon.cpp:832-835`, `:917-920`), and
   `WF_TWOHANDSTABILIZED` is re-asserted on every state set (`p_pspr.cpp:512-515`). Folded into
   §4.4 and W11. One addition to your note: it is **also** gated on the `vr_two_handed_weapons`
   cvar, which a mod cannot clear outside menu code (W8) — so "just turn it off" is not available
   to the mod either. Note too that `weaponStabilised` is written **only** on the OpenVR path
   (`gl_openvr.cpp:1723`, `:1726`; the global lives at `qzdoom_common.cpp:19` and the OpenXR device
   never touches it) — which, given item 7 below, is the live path anyway.
6. **Haptics — correction accepted in part; I hold one half.** Agreed: `VR_HapticEvent` is an empty
   stub (`src/QzDoom/qzdoom_common.cpp:65-67`) and **neither** entry point is ZScript-bound.
   `OpenVRHaptics::Vibrate` is genuinely implemented (`gl_openvr.cpp:517-528`, override
   `gl_openvr.h:121-124`, gated on `vr_enable_haptics`, default true, `hw_vrmodes.cpp:125`).
   **But "not reachable from script" is too strong for one specific route:** `A_QuakeEx` is
   ZScript, and the quake handler calls `vrmode->Vibrate(10, 0, left)` / `(10, 1, right)` directly
   at `src/rendering/r_utility.cpp:1084-1090`. On the OpenVR path that resolves to the real
   implementation. So a mod *can* cause controller rumble, indirectly, by raising a quake whose X
   and Y intensities it chooses — which is exactly the stomp design in §4.7. It is a HACK (one
   lever, no event naming, intensity laundered through quake maths), not a binding. **Your wider
   point stands and I have adopted it:** with view shake a hard WALL, a ZScript binding over the
   existing `VRMode::Vibrate` is the cheapest high-value engine addition for this topic — it is a
   binding over a working implementation, not a new feature. It is named in §7 and W9.
7. **Which VR path is even compiled — this re-weights every citation in this report.**
   `hw_openxrdevice.cpp` is entirely inside `#ifdef USE_OPENXR` (`:32`, `:792`), and **no CMake
   file in this tree defines `USE_OPENXR`** (grep finds it only in source guards). `USE_OPENVR` is
   defined, but only under `option(ENABLE_OPENVR … NO)` (`src/CMakeLists.txt:418-420`). So on this
   tree the live VR path is **OpenVR**, in a build that opted in. I wrote §2 leading with the
   OpenXR citations; the OpenVR equivalents are cited alongside them throughout
   (`gl_openvr.cpp:279-294`, `:786-795`, `:2515-2517`, `:1282-1292`) and say the same thing, so **no
   conclusion in this report changes** — but read the `gl_openvr.cpp` citations as the operative
   ones. It also means `QzDoom_Vibrate`, which the OpenXR `Vibrate` override calls
   (`hw_openxrdevice.cpp:757`), is **declared but defined nowhere in this repo**: that path is
   Quest-side glue, absent here.
8. **Units, recorded for cockpit geometry:** `vr_vunits_per_meter` = 34 (`hw_vrmodes.cpp:94`); Z
   additionally divides by `level.pixelstretch` (`gl_openvr.cpp:1282-1292`; same division at
   `hw_openxrdevice.cpp:151-155`), default 1.2. One real metre is **34 map units horizontally,
   ~28.3 vertically** — size controls with that asymmetry or a "square" console reads squashed.

**Where I disagree — and this is the hinge of the topic, so I am showing the citations.**

I was told: *"head height is a render-only eye offset plus crouchfactor … you cannot duck or stand
your collision volume … 'does VR view height follow a scaled player' — the answer looks like no."*

The first half is right. **The conclusion does not follow from it, and the source says the
opposite.** The question is not whether the head drives the body; it is whether the *body's*
script-owned height drives the *eye*. It does:

```
eye_Z = vp.Pos.Z                                   // = player->viewz, written in ZScript
      + ( hmdHeightInMapUnits − cachedDefaultViewHeight )   // the "render-only offset"
```

`vp.Pos.Z` is taken from `player->viewz` at `src/rendering/r_utility.cpp:1023`; `viewz` is written
in **virtual ZScript** at `wadsrc/static/zscript/actors/player/player.zs:707`; the HMD term is
*added* to it at `hw_openxrdevice.cpp:219` (the line cited to me as `:221`), applied at
`src/rendering/hwrenderer/hw_entrypoint.cpp:182`. "Render-only offset" is exactly the point — an
**offset is added to a base that script owns**. Raise `ViewHeight` and the eye rises by the same
amount, with real head motion still riding on top, unscaled. Nothing clamps, replaces or
normalises the base; the only clamps are to the sector (§W7).

Two supporting details, both from the same citations I was handed:
- The XY-only positional code **explicitly restores the pawn's Z** after room-scale movement
  (`hw_openxrdevice.cpp:669-686`, OpenVR `gl_openvr.cpp:2613-2632`). That is the engine taking care
  that the *head never writes the body's Z* — which is precisely why the body's Z and view height
  remain script's to set.
- "Plus crouchfactor" is itself a counterexample to "you cannot duck or stand your collision
  volume": with `vr_crouch_use_button false` the engine sets `crouchfactor = hmdHeight /
  cachedDefaultViewHeight` every frame (`hw_openxrdevice.cpp:574-579`) and `PlayerPawn.Tick` sets
  `Height = FullHeight * crouchfactor` (`player.zs:146-155`), with the 0.5–1.0 clamp skipped
  because `CrouchMove(0)` passes `direction == 0` (`player.zs:1177-1180`). In that branch your real
  stance **does** scale the collision volume, without limit. In the default branch
  (`vr_crouch_use_button true`, `hw_vrmodes.cpp:122`) it does not. The claim is true of the default
  branch only, and should be stated with that condition.

**So the decisive question resolves YES, and the topic is viable.** If I am wrong, the falsifier is
cheap and is step (a) of §6: morph to a pawn with `Player.ViewHeight 240` and see whether the eye
sits ~7 m up or snaps back to real head height.

---

## 1. Verdict

**A mech is viable, and the viability question has a clean source answer: the HMD does *not*
override the player's view height — it is added to it.** The VR eye is
`player.viewz + (realHmdHeightInMapUnits − cachedDefaultViewHeight)`
(`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:219`, applied at
`src/rendering/hwrenderer/hw_entrypoint.cpp:182`), and `viewz` is written in **virtual ZScript**
(`wadsrc/static/zscript/actors/player/player.zs:707`). Raise `ViewHeight` / `viewz` and you are
physically standing 3 m higher, with your real head motion still riding on top, unscaled.
Collision height follows through `FullHeight` (`player.zs:146-155`), and `A_SetSize`
(`p_actionfunctions.cpp:5062`) exists for the radius.

What you *cannot* have: **a second facing**. The playsim has exactly one yaw for the pawn, and in
VR the view yaw is overwritten from the HMD every frame (`hw_openxrdevice.cpp:750`), so
"torso twist independent of legs" must be inverted — the *legs* become the script-side second
angle, by overriding `MovePlayer` (`player.zs:1340`) and thrusting along a mod-owned `legsAngle`.
You also cannot rotate the VR camera from script at all (roll/pitch clobbered,
`hw_openxrdevice.cpp:740-741`): no screen-shake, no view-kick, no forced look — a **WALL**, and a
comfort blessing. Stomp feedback therefore has to come from the cockpit model, audio, and one
HACK-grade rumble route — `A_QuakeEx` reaches `VRMode::Vibrate` via
`src/rendering/r_utility.cpp:1084-1090`, which is implemented on the OpenVR path but unbound to
script and path-dependent (§0.6, §0.7); treat it as the thing to test first, not to design on.
Both hands being readable (§0.1) makes the cockpit genuinely two-handed: left hand on the throttle,
right on the stick, each tracked in world space.

The real cost centre is the cockpit. A psprite layer in VR is **hand-locked, not view-locked**
(`src/rendering/hwrenderer/scene/hw_weapon.cpp:256` picks the hand from the layer's *caller*), so
a hand-held psprite cockpit rides your wrist. The usable cockpit is spawned actors with MODELDEF
models, re-positioned every tic around the pawn. And the tall player breaks doorways hard
(`src/playsim/p_map.cpp:2413`) and gets its eye clamped to `ceilingz − 4` indoors
(`player.zs:714` and `src/rendering/r_utility.cpp:1127-1141`). A mech is an outdoor/arena mod on
this engine unless the maps are built for it — and we may not build maps.

---

## 2. What already exists

### 2.1 Player size is fully script-owned

| Thing | Where |
|---|---|
| `Player.ViewHeight` (plain writable double, default 41) | `wadsrc/static/zscript/actors/player/player.zs:38`, property `:71`, default `:123` |
| `FullHeight` (the pawn's real collision height, captured at spawn) | `player.zs:53`, set `:193` |
| `PlayerPawn.Tick` rewrites `Height = FullHeight * player.crouchfactor` **every tic** | `player.zs:146-155` |
| `CanCrouch()` is virtual — override it false and `Height = FullHeight` flat | `player.zs:1137-1140` |
| `A_SetSize(radius, height, testpos)` native | `wadsrc/static/zscript/actors/actor.zs:1314`; impl `src/playsim/p_actionfunctions.cpp:5062-5090` |
| `MaxStepHeight` / `MaxDropoffHeight` (default 24 each) | `actor.zs:233-234`, defaults `actor.zs:422-423` |
| `Player.UseRange` (default 64) drives the +use trace length | `player.zs:45`, used `src/playsim/p_map.cpp:6161` |
| `Actor.Scale` scales the rendered model directly | `src/r_data/models.cpp:81-83` |

### 2.2 The view height chain, end to end

1. `CalcHeight()` — **virtual ZScript** — computes `player.viewz = pos.Z + player.viewheight + bob`
   (`player.zs:604`, `:707`), with `defaultviewheight = ViewHeight + crouchviewdelta` (`:644`) and
   hard clamps `viewz ≤ ceilingz−4`, `≥ floorz+4` (`:714-721`).
2. `R_SetupFrame` takes `camPos.Z = player->viewz` (`src/rendering/r_utility.cpp:1023`), adds any
   `ViewPos` offset (`r_utility.cpp:648-672`), then **clamps the result into the sector**
   (`r_utility.cpp:1127-1141`), then stores `CenterEyePos = Pos` (`:1236`).
3. `hw_entrypoint.cpp:143` calls `vrmode->SetUp()`, then per eye `vp.Pos += eye->GetViewShift(vp)`
   (`:182`).
4. `GetViewShift` adds IPD sideways **and**
   `eyeOffset[2] += getHmdAdjustedHeightInMapUnit() − getDoomPlayerHeightWithoutCrouch(player)`
   (`hw_openxrdevice.cpp:219`).
   - `getHmdAdjustedHeightInMapUnit() = (hmdPosition[1] + vr_height_adjust) * vr_vunits_per_meter / pixelstretch`
     (`hw_openxrdevice.cpp:151-155`).
   - `getDoomPlayerHeightWithoutCrouch()` returns, with `vr_crouch_use_button` **true (the default,
     `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:122`)**, a `static float` cached from
     `player->DefaultViewHeight()` on the *first call in the process* (`hw_openxrdevice.cpp:158-171`;
     `DefaultViewHeight()` = `mo->FloatVar(NAME_ViewHeight)`, `src/playsim/d_player.h:465-468`).

   **So: eye_Z = pos.Z + player.viewheight + bob + hmdHeight − 41.** The HMD is an *additive term*.
   Script owns the base. This is the whole topic's load-bearing fact.
5. The OpenVR path is identical in structure (`src/rendering/gl/stereo3d/gl_openvr.cpp:279-294`,
   `:786-795`).

**The other branch.** With `vr_crouch_use_button false`, the engine instead forces, every frame,
`player->crouching = 10; player->crouchfactor = hmdHeight / defaultViewHeight;`
(`hw_openxrdevice.cpp:574-579`, `gl_openvr.cpp:2515-2517`), and the Z shift becomes zero (both
terms equal). Then `Height = FullHeight * crouchfactor` and `viewheight = ViewHeight * crouchfactor`
(`player.zs:152`, `player.zs:1181`), and because `CrouchMove(0)` is called with `direction == 0`
the `clamp(0.5, 1.0)` is skipped (`player.zs:1177-1180`) — crouchfactor is free to exceed 1.
In this mode your real standing height **multiplies** the mech's height: a 3× mech means a 3×
real crouch. That is the more "mech-like" mode and it is one cvar away, but the cvar is only
settable from menu code (§2.5).

### 2.3 The VR camera's rotation is not script's to touch

`updateHmdPose` runs each frame inside `vrmode->SetUp()` and writes:
`G_AddViewAngle(-hmdYawDelta)` (`hw_openxrdevice.cpp:716`), `vp.HWAngles.Roll = hmdRoll`,
`vp.HWAngles.Pitch = hmdPitch` (`:740-741`), `vp.Angles.Yaw = doomYaw` (`:750`).
`HWAngles` is what builds the view matrix (`src/rendering/hwrenderer/scene/hw_drawinfo.cpp:443`).
`A_SetViewAngle` / `A_SetViewPitch` / `A_SetViewRoll` (`actor.zs:1275-1277`) feed
`viewPoint.Angles` at `r_utility.cpp:676-692` — **before** `SetUp()` overwrites it. Dead in VR.
Positional offsets are not overwritten: `SetViewPos(offset, flags)` (`actor.zs:1278`) survives all
the way to the eye.

### 2.4 Quakes: the one script→haptics route

`A_Quake` / `A_QuakeEx` (`actor.zs:1283-1284`) land in `r_utility.cpp:1060-1095`:
- visual jiggle multiplied by `r_quakeintensity`, **default 0 "for VR"** (`r_utility.cpp:116`),
  applied as `iView->ViewOffset` / `RelativeViewOffset` (positional, survives into VR) and
  `AngleOffsets.Roll` (rotational, clobbered per §2.3);
- and unconditionally, when `vr_quake_haptic_level > 0` (**default 0.8**, `hw_vrmodes.cpp:127`):
  `vrmode->Vibrate(10, 0, left)`, `Vibrate(10, 1, right)` plus
  `VR_HapticEvent("rumble_front"/"rumble_back", …)` (`r_utility.cpp:1084-1094`), with `left`/`right`
  derived from the quake's X and Y intensity.

This **qualifies the caution in BASELINE §4** for one specific case: a mod *can* fire controller
rumble, by raising a quake whose intensity it chooses, left and right separately. It is a HACK
(the only lever is a quake) over a cited call path, and its liveness is **path-dependent**:
`VRMode::Vibrate` is a real implementation on OpenVR (`gl_openvr.cpp:517-528`, gated
`vr_enable_haptics`), while the OpenXR override calls `QzDoom_Vibrate`
(`hw_openxrdevice.cpp:757`), which has **no definition in this repo**. `VR_HapticEvent` — the
other call in that same block — is an **empty stub** (`qzdoom_common.cpp:65-67`) and does nothing
at all. Given §0.7 (OpenVR is the compiled path here), the rumble should work; verify before
shipping a mechanic that depends on it.

### 2.5 Engine cvars are menu-only for a mod

`CVar.SetFloat`/`SetInt` throw `"Attempt to change CVAR outside of menu code"` for any cvar lacking
`CVAR_MOD` (`src/common/scripting/interface/vmnatives.cpp:981-991`). So `vr_vunits_per_meter`
(default **34.0**, `hw_vrmodes.cpp:94`), `vr_crouch_use_button`, `vr_height_adjust`,
`vr_weaponScale` (`hw_vrmodes.cpp:107`), `vr_3dweaponOffset*`, `r_quakeintensity` are reachable
**only from a MENUDEF/menu class** the mod ships — not per tic, not per state.

### 2.6 psprites in VR ride the hand, not the head

- `DrawPlayerSprites` calls `vrmode->AdjustPlayerSprites(state, psp->GetCaller() == player->OffhandWeapon)`
  for **every non-model layer** (`hw_weapon.cpp:247-258`) — the hand is chosen by which item owns
  the layer, nothing else.
- `AdjustPlayerSprites` loads the hand matrix and scales by
  `0.000625 * vr_weaponScale * vr_2dweaponScale` (`hw_openxrdevice.cpp:405-417`).
- `GetHandTransform` builds that matrix from `CenterEyePos`, `vr_vunits_per_meter`, the controller
  offset and the controller's yaw/pitch/roll (`hw_openxrdevice.cpp:430-479`).
- Model layers go through `RenderHUDModel` (`src/r_data/models.cpp:207-285`): same hand transform
  (`:219-226`), MODELDEF `scale`/`offset`/`angleoffset` (`:243-271`), then **`psp->x`/`psp->y` enter
  as *rotations*, not translations** — `rotation + FVector3(mx/4, (my−WEAPONTOP)/−4, 0)`
  (`hw_weapon.cpp:108`).
- PSprite script surface: `x, y, scale, rotation, pivot, alpha, Sprite, Frame, Caller`
  (`player.zs:2932-2981`); layer ids `WEAPON=1`, `FLASH=1000`, `OFFHANDWEAPON=1000000`
  (`player.zs:2934-2943`); overlay verbs `A_Overlay*` (`actor.zs:1323-1334`).

There is **no script-side 3D translation of a psprite model** in this build: no per-actor model
offset exists either (`DActorModelData` carries models/skins/anims only, `src/playsim/actor.h:732-749`).

### 2.6a The persistent psprite band — useful, but not for a cockpit *frame*

Layers at or above `PSP_TARGETCENTER` (= `INT_MAX − 2`, `src/playsim/p_pspr.h:55-61`) are a
separate, persistent band:
- the weapon passes iterate only `psp->GetID() < PSP_TARGETCENTER`
  (`hw_weapon.cpp:830`, `:915`), so weapon switching and weapon-layer churn never touch them;
- they are collected in their own pass, `PrepareTargeterSprites` (`hw_weapon.cpp:1041`, loop at
  `:1068`), called from `PreparePlayerSprites` (`:1031`);
- `A_ClearOverlays` with `safety` **breaks out** before reaching them (`src/playsim/p_pspr.cpp:1273-1274`).
- **Trap (confirmed):** `GetPSprite` defaults the caller of any layer `>= PSP_TARGETCENTER` to the
  player's `PowerTargeter` item and the layer is culled without one (`p_pspr.cpp:310-318`) — pass
  `newcaller` explicitly.

**Correction to how it was offered to me.** It is *persistence* that this band buys, not
*view-locking*. `PrepareTargeterSprites` pushes into the same `hudsprites` array
(`hw_weapon.cpp:1072`, `:1076`), and `DrawPlayerSprites` applies the hand matrix to **every** non-model
entry in that array (`hw_weapon.cpp:252-258`) — choosing the hand by
`hudsprite.weapon->GetCaller() == player->OffhandWeapon`, which for a `PowerTargeter`-called layer
is false, i.e. the **main hand**. So a cockpit frame parked in this band is bolted to the firing
hand, not to the head. Same wall as W3, different layer id.

What the band *is* good for in a mech: **persistent hand-mounted instruments** — a wrist heat
gauge, an ammo readout, a damage-silhouette — that survive every weapon switch and every
`A_ClearOverlays(.., safety: true)` the host mod performs. Give it the offhand weapon as its
explicit caller and it rides the free hand, which is where a pilot's wrist display belongs.

### 2.7 Morph is a complete enter/exit-the-mech primitive

`PlayerPawn.MorphPlayer(...)` (`wadsrc/static/zscript/actors/player/player_morph.zs:98-190`) spawns
a different `PlayerPawn` class, transfers angle/pitch/inventory/target, stores
`PremorphWeapon` **and `PremorphWeaponOffhand`** (`player_morph.zs:181-183`), and — directly on
point — *"If the new view height is higher than the old one, start moving toward it"*
(`player_morph.zs:185-187`), i.e. the engine already ramps the view up into a taller pawn.
`Unmorph` / `UndoPlayerMorph` at `wadsrc/static/zscript/actors/morph.zs:139-142`.

### 2.8 Hand-origin +use already exists

With `use_mode > 0` (cvar default 0, `p_map.cpp:105`) `P_UseLines` re-traces from
`AttackPos`/`AttackAngle` and then `OffhandPos`/`OffhandAngle` with the per-weapon `UseRange`
(`p_map.cpp:6154-6188`); same for puzzle items (`p_map.cpp:6289-6310`). Line specials only — it
does not "use" actors — but it proves the hand pose is already wired into playsim interaction.

---

## 3. Building blocks

| Block | API | Citation |
|---|---|---|
| Make the pawn tall | `ViewHeight`, `FullHeight`, `A_SetSize`, override `CanCrouch()` | `player.zs:38,53,146-155,1137`; `actor.zs:1314` |
| Raise the VR eye | write `player.viewz` in an overridden `CalcHeight()` | `player.zs:604-721`; eye adds HMD on top, `hw_openxrdevice.cpp:219` |
| Nudge the eye without touching viewz | `SetViewPos(offset, flags)` per tic | `actor.zs:1278`; applied `r_utility.cpp:648-672` |
| Second (leg) facing | override `MovePlayer()`, thrust along a mod field | `player.zs:1340-1441` |
| Both hands in world space | `OffhandPos/Angle/Pitch/Roll` + `AttackPos/Angle/Pitch/Roll`, gated on `OverrideAttackPosDir` | `actor.zs:268-276`; filled `hw_openxrdevice.cpp:586-609` |
| Persistent hand-mounted instrument layer | psprite id ≥ `PSP_TARGETCENTER`, explicit `newcaller` | `p_pspr.h:55-61`; `hw_weapon.cpp:830`, `:915`, `:1068`; `p_pspr.cpp:310-318`, `:1273-1274` |
| Two weapon state machines | `OffhandWeapon` + `PSP_OFFHANDWEAPON` + `CheckWeaponButtons(hand)` | BASELINE §2; `player.zs:588` |
| Cockpit geometry | spawned actors + MODELDEF, repositioned per tic; `Actor.Scale` | `models.cpp:71-200`, `:842-932` |
| Swap a model at runtime (damage states) | `A_ChangeModel(modeldef, modelindex, …)` | `actor.zs:1232` |
| Rumble (HACK, verify first) | `A_QuakeEx` with chosen X/Y intensity → `VRMode::Vibrate` | `actor.zs:1284`; `r_utility.cpp:1084-1090`; impl `gl_openvr.cpp:517-528`; §0.6 |
| Screen tint (heat, alarms) | `A_SetBlend(color, alpha, tics, …)` | `actor.zs:1229` |
| Enter/exit | `MorphPlayer` / `UndoPlayerMorph` | `player_morph.zs:98`; `morph.zs:139` |
| Eject velocity / reposition | `Warp()`, `Vel`, `SetOrigin` | `actor.zs:942`, `:753` |

---

## 4. Mechanic designs

### 4.1 Becoming the mech (scale + view height)

**Player does:** steps into a hangar outline and presses the grip, or picks up a "mech key" actor.
**Script detects:** ordinary `+use` / inventory pickup; no gesture needed.
**Resolves:** `MorphPlayer` into `MechPawn : DoomPlayer` with `Player.ViewHeight 240`,
`Height 300`, `Radius 48`, `MaxStepHeight 64`, `Player.UseRange 192`, `Speed` raised,
`override bool CanCrouch() { return false; }` so `Tick` stops multiplying `Height`
(`player.zs:146-155`). The engine ramps the view up on its own (`player_morph.zs:185-187`).
Your real head motion stays 1:1 on top of the new base (`hw_openxrdevice.cpp:219`) — which reads
correctly as "a small pilot inside a big machine".
**Feel knobs:** `ViewHeight` (eye altitude), `Speed` + `ForwardMove1/2` (`player.zs:39`),
`MaxStepHeight` (does it step over cars or stop at them), `ViewBob`/`ViewBobSpeed`
(`player.zs:50-52`, consumed at `player.zs:707`) — set `ViewBob 0` for comfort and put the sway in
the cockpit model instead, `JumpZ`, friction via `vr_momentum` handling in `MovePlayer`
(`player.zs:1374-1388`).
**Label:** BUILDABLE — **cost S**.

**Trap, state it in the design doc:** the subtracted constant in `GetViewShift` is a
`static float` frozen at the *first* VR frame of the process (`hw_openxrdevice.cpp:158-171`). If
the player is somehow already the mech at that instant, the cached value is the mech's 240, not 41,
and the eye sits 199 units lower than the designed height for the rest of the session. Never spawn
directly as the mech pawn; always morph into it after the first rendered frame.

### 4.2 1:1 body scaling (the "real crouch scales too" mode)

**Player does:** ducks their real body; the mech ducks proportionally.
**Script detects:** nothing — the engine does it, when `vr_crouch_use_button false`:
`crouchfactor = hmdHeight / cachedDefaultViewHeight` (`hw_openxrdevice.cpp:574-579`), unclamped
because `CrouchMove(0)` skips the clamp (`player.zs:1177-1180`), and `viewheight = ViewHeight *
crouchfactor` (`player.zs:1181`).
**Resolves:** eye and collision height both scale with your real stance, multiplied by the mech's
`ViewHeight`. Note `CanCrouch()` must stay **true** for this path, which re-enables the per-tic
`Height = FullHeight * crouchfactor` — that is wanted here.
**Feel knobs:** `vr_height_adjust`, the mech's `ViewHeight` as the multiplier.
**Label:** HACK — **cost S for the code, M with the menu**. The cvar can only be set from a mod
menu (`vmnatives.cpp:981-991`), so the mod must ship a MENUDEF page and ask the player to flip it,
or detect it and degrade. It also changes crouching for every other mod the player runs.

### 4.3 Cockpit shell

**Player does:** looks around a cockpit that stays put as they move their head.
**Script detects:** n/a — it is rendering.
**Resolves — the viable route:** on morph, spawn N cockpit actors (`+NOINTERACTION`,
`+NOBLOCKMAP`, `+NOGRAVITY`), each with a MODELDEF model (`models.cpp:842-932`), and in the pawn's
`Tick` set each one's `SetOrigin(mechPos + rotate(localOffset, legsAngle), true)` plus
`Angle/Pitch/Roll` and `Scale` (`actor.zs:753`; model transform `models.cpp:143-200`). Hard rule
respected: these are things, not geometry. Cost is per-tic positioning jitter — use
`SetOrigin(..., true)` and `RF_INTERPOLATEANGLES` (`models.cpp:88-91`) so the renderer smooths
between tics, otherwise a head-locked frame at 35 Hz will be visibly steppy against a 72–90 Hz HMD.
**Resolves — the route that looks obvious and is wrong:** a psprite cockpit. Every non-model
psprite layer in VR is transformed by the *hand* matrix, chosen by the layer's caller
(`hw_weapon.cpp:256`, `hw_openxrdevice.cpp:405-417`), and HUD models likewise
(`models.cpp:219-226`). A psprite cockpit is nailed to your wrist. Dead end; say so once, loudly.
**Feel knobs:** cockpit radius vs. `vr_vunits_per_meter` 34 (one real metre = 34 map units,
`hw_vrmodes.cpp:94`) — a 2 m-wide cockpit is 68 units; frame struts' thickness for the
"heavy machine" read; a small per-tic positional lag (lerp the cockpit 1–2 tics behind the pawn)
to sell mass without ever rotating the camera.
**Label:** BUILDABLE (actors) / WALL (psprites) — **cost M**.

### 4.4 Levers, buttons and sticks — two hands

**Player does:** reaches out with either hand, closes their grip on a lever or stick,
pushes/pulls/twists, lets go. Both at once: left on the throttle, right on the stick.
**Script detects:** per tic, hand position vs. each control actor's position —
`OffhandPos` for the free hand and **`AttackPos` for the main hand**
(`actor.zs:268-271`, real controller data at `hw_openxrdevice.cpp:586-597`), both gated on
`OverrideAttackPosDir` (`actor.zs:276`) so the non-VR fallback — which sets
`AttackPos == OffhandPos` from the viewpoint (`hw_vrmodes.cpp:437-444`) — never reads as two hands
in the same place. Grip is each hand's fire button via `CheckWeaponButtons(hand)`
(`player.zs:588`, `:2031`). On grab, store `grabOffset = handPos − leverPivot`; each tic thereafter
derive the lever's angle from the current hand position, and drive the control's value from that
angle. Release on button-up or when the hand exceeds a break distance.
**Main-hand caveat:** the main hand is also the weapon hand — its orientation is the aim
(`AttackAngle`/`AttackPitch` feed firing). A control held in the main hand should therefore be a
control that *is* the aim (the gun stick), or the mech should holster while that hand is on a
lever. Do not put the throttle in the firing hand.
**Resolves:** the control actor is re-posed (`Angle`/`Pitch`, `models.cpp:165-167`), and its value
feeds the mech (throttle → `MovePlayer` scalar; stick → `legsAngle` rate; a toggle → a weapon
group). No native "use an actor with the hand" is needed, and none exists — `P_UseLines`' hand
traces (`p_map.cpp:6170-6187`) hit line specials only, which the hard rule puts out of reach.
**Feel knobs:** grab radius (BASELINE §3: 35 Hz sampling, so make the capture volume generous and
sticky — hysteresis between grab and break radius), lever travel in cm → value curve, a per-tic
`A_QuakeEx` tick-tick at detents for rumble feedback (§4.7), audio detents.
**The 0.50 m trap — this is a design constraint, not a footnote.** If the left-hand control is
driven by an **offhand weapon**, then the moment the player grips it with the hands closer than
**0.50 m** apart, `weaponStabilised` latches (`gl_openvr.cpp:1716-1727`) and that offhand weapon
goes silent: no fire (`player.zs:393-396`), no button checks (`p_pspr.cpp:755-758`), no psprite
(`hw_weapon.cpp:832-835`, `:917-920`), with the flag re-asserted on every state set
(`p_pspr.cpp:512-515`) so script cannot durably clear it. A twin-stick or yoke cockpit is exactly
the geometry that triggers it. Three mitigations, in order of preference:

1. **Drive the controls from hand positions, with no offhand weapon involved** — compare
   `OffhandPos` / `AttackPos` to control-actor positions in the pawn's `Tick` and read the grip as
   a `cmd.buttons` bit rather than a weapon fire state. Then nothing about the offhand weapon
   state machine is load-bearing and the trap is simply absent. **This is the recommended build**
   and is what §6 prototypes.
2. **Space the controls more than 0.50 m apart.** Entirely realistic for a mech — real heavy-plant
   and cockpit yokes put the grips 0.6–0.8 m apart, i.e. **20–27 map units** at 34 units/m (§0.8).
   Design the console that way and the trap never arms.
3. **Detect and degrade:** test `player.WeaponState & WF_TWOHANDSTABILIZED` and surface a cockpit
   warning rather than letting the left stick silently die.
Do **not** plan on clearing `vr_two_handed_weapons` from the mod — it is an engine cvar and
menu-only (W8).

**Feel knobs (cont.):** per-hand grab radius, detent spacing, which hand owns which control,
console width (≥ 0.5 m between grips).
**Label:** BUILDABLE — **cost M**. Two-handed is now genuinely available (§0.1); the 35 Hz sampling
and the absence of hand-velocity fields (BASELINE §1b, §3) still mean a *grab* must be a button
plus proximity, never a detected lunge.

### 4.5 Torso twist vs leg direction

**Player does:** turns their head/body to aim one way while the mech walks another.
**Script detects:** nothing new — the pawn's `Angle` *is* the head yaw in VR
(`hw_openxrdevice.cpp:716` feeds `G_AddViewAngle`, `:750` sets the view from `doomYaw`).
**Resolves — inverted design.** Keep `Angle` as the **torso/aim** (it must stay that, because the
weapon fire direction, autoaim and `P_UseLines` all read it) and make the **legs** the script-owned
second angle: a `double LegsAngle` on the mech pawn, turned toward the stick direction at a limited
rate; override `MovePlayer()` (`player.zs:1340`) and replace
`ForwardThrust(forwardmove, Angle)` / `Thrust(sidemove, Angle−90)` (`player.zs:1429-1438`) with
thrusts along `LegsAngle`. Clamp `(Angle − LegsAngle)` to a twist limit and bleed the legs toward
the torso when it saturates — that *is* the mech feel.
**What does not exist:** `ThrustAngleOffset` on `AActor` (`src/playsim/actor.h:1283`, applied in
`Thrust` at `:1609`) would be exactly the right field, but on the OpenXR path its only writer is
inside `#if 0` (`hw_openxrdevice.cpp:653-664`) and it has **no `DEFINE_FIELD`** anywhere in
`src/scripting` — ZScript cannot see it. A second *visible* facing for the legs needs the
leg model to be a spawned actor carrying `LegsAngle` (§4.3), because the pawn has one `Angle`.
**Feel knobs:** twist limit (±45–120°), leg turn rate (deg/tic), whether the legs snap to the
torso when you start walking, cockpit yaw-lag.
**Label:** BUILDABLE (as legs-follow, not torso-twist) — **cost M**.
Native per-actor "second facing": **WALL** — missing piece is a script-visible second yaw applied
in `AActor::Thrust` (i.e. a `DEFINE_FIELD` for `ThrustAngleOffset`, which the C++ already applies).

### 4.6 Heat, per-limb damage, per-arm hardpoints

All three are script bookkeeping; what matters is what each binds to.

- **Heat.** Accumulate per shot in the weapon's fire state; bleed per tic in the pawn's `Tick`.
  Visible as: `A_SetBlend(0xFF4000, heat*0.25, 4)` (`actor.zs:1229`) for the red-out; an overlay
  psprite gauge on the **offhand** layer (`player.zs:2939`) so it sits on the free hand like a
  wrist readout rather than the aiming hand; a cockpit gauge actor whose `Frame` or model index is
  stepped with `A_ChangeModel` (`actor.zs:1232`); and rumble at redline via §4.7. Shutdown =
  refuse the fire state and play a vent sound.
  **BUILDABLE — cost S.**
- **Per-limb damage.** Give the pawn `int limbHP[N]`, and in `override int DamageMobj(...)` pick a
  limb from the damage direction (`AngleTo(inflictor)` vs. `Angle`, and the hit Z vs. `pos.Z +
  Height/2`). Visible as: `A_ChangeModel` swapping that limb's cockpit/exterior model for a damaged
  one (`actor.zs:1232`, read at `models.cpp:75`, `:212`); the arm's weapon disabled by clearing
  `OffhandWeapon` or forcing the weapon's state machine to a "jammed" state; a cockpit warning
  light actor. Arm loss has a real cost on the main hand — the main hand *is* the aim, so a dead
  main arm must become a reduced-accuracy state, not a dead controller.
  **BUILDABLE — cost M.**
- **Hardpoints per arm.** This is the shipped offhand weapon system (BASELINE §2), not new work:
  `OffhandWeapon` + `PSP_OFFHANDWEAPON` + `bOffhandWeapon` + per-hand `CheckWeaponButtons`
  (`player.zs:588`, `:1977-1998`). Group weapons by arm; a "fire group" selector is a cockpit
  control (§4.4) that swaps which `Weapon` occupies each hand. `bTwoHanded` force-clears the
  offhand (`player.zs:1989-1992`) — use it for the shoulder cannon that needs both hands on the
  grips.
  **EXISTS — cost S** to configure.

### 4.7 Footstep stomps, screen shake, bob — and the VR comfort answer

**Be blunt: script cannot rotate the VR camera. This is a WALL, not a design choice.**
`vp.HWAngles.Roll/Pitch` are overwritten from the raw HMD every frame after `R_SetupFrame`
(`hw_entrypoint.cpp:117` then `:143` → `hw_openxrdevice.cpp:740-741`; OpenVR `gl_openvr.cpp:1467`,
`:1471`, with the engine's own comment at `gl_openvr.cpp:1464`) and `vp.Angles.Yaw` from `doomYaw`
(`hw_openxrdevice.cpp:750`), after any script contribution was folded in at
`r_utility.cpp:676-692`. The view matrix is then built from `HWAngles`
(`hw_drawinfo.cpp:443`). `A_SetViewRoll`, quake roll (`r_utility.cpp:1067`), weapon-kick view
rotation and camera-bob roll are all discarded in VR.

**The non-view alternatives, in the order they carry weight:**
1. **Haptics, via the quake route only** — alternating controller rumble per stride (below). The
   strongest substitute and the only one that lands on the body, but it is a HACK on a single
   lever and it is the one item here I would not ship unverified: there is **no ZScript binding**
   to `VRMode::Vibrate`, and the stub situation differs by path (§0.6, §0.7). If it proves dead in
   the owner's build, items 2–4 carry the mechanic on their own.
2. **World-space effects** — the cockpit actors (§4.3) drop and spring back; dust/debris actors
   spawned at the foot; a brief `A_SetBlend` flash for a hard landing (`actor.zs:1229`).
3. **psprite shake** — `psp.x/psp.y` jitter on the hand layers (`player.zs:2954-2955`), i.e. the
   *machine* shakes in your hands while your head stays still. In VR those offsets become model
   rotations for model layers (`hw_weapon.cpp:108`), so keep the amplitude small and expect rotary,
   not linear, motion.
4. **Audio** — the heaviest and cheapest cue; a sub-bass stomp sells more mass than any visual.
5. **Positional view nudge** — `SetViewPos` / quake positional offsets do survive into VR
   (`r_utility.cpp:1069-1082`, `actor.zs:1278`). Vertical-only, 2–4 units; see below.

Script *can* translate the VR view: `SetViewPos` (`actor.zs:1278`) and quake positional offsets
(`r_utility.cpp:1069-1082`) reach `vp.Pos` before the eye shift is added
(`hw_entrypoint.cpp:182`) — the same additive structure as §0's view-height finding. A
vertical-only stomp drop of 2–4 units is tolerable; anything lateral is the classic nausea source —
don't.

**Stomp design** (rumble half is HACK and conditional — see §0.6/§0.7; the visual/audio half stands
on its own). In `Tick`, track walked distance; every stride length, fire
`A_QuakeEx(intensityX, intensityY, intensityZ, duration, damrad, tremrad, "mech/step", …)`
(`actor.zs:1284`). The player feels it as **controller rumble**, because
`vr_quake_haptic_level` (default 0.8, `hw_vrmodes.cpp:127`) routes the quake's X and Y intensities
to `Vibrate(…, 0, left)` / `Vibrate(…, 1, right)` and `VR_HapticEvent("rumble_front"/"rumble_back")`
(`r_utility.cpp:1084-1094`) — and they see almost nothing, because `r_quakeintensity` defaults to
0 for VR (`r_utility.cpp:116`). Put the left foot's energy in X and the right foot's in Y and the
stomps **alternate between the two controllers**. Add the visible kick on the cockpit actors (§4.3)
instead of the camera: drop the whole cockpit 2 units and spring it back over 6 tics.
**Feel knobs:** stride length, X/Y split per foot, quake `duration`, cockpit drop amplitude and
spring rate, `ViewBob 0` on the mech pawn (`player.zs:51`, consumed `player.zs:707`) so the walk
cycle lives entirely in the cockpit and the rumble.
**Label:** stomps BUILDABLE — **cost S**; rumble HACK (quake is the only lever) — **cost S**;
camera shake / view-kick / camera-bob **WALL** — missing piece is a script-settable view-angle
offset applied *after* `updateHmdPose` (W2/W10), and in a mech we do not want it anyway.

### 4.8 Entering, exiting, ejecting

**Player does:** grabs an overhead handle and pulls (eject), or walks to a hangar bay and +uses.
**Script detects:** §4.4 grab on a dedicated eject handle actor, with a two-stage confirm (pull
past a long travel) so a stray hand does not eject you; or plain `+use`.
**Resolves:** `UndoPlayerMorph(player, MRF_UNDOALWAYS, true)` (`morph.zs:139-142`) returns you to
the pilot pawn, restoring `PremorphWeapon` and `PremorphWeaponOffhand` (`player_morph.zs:181-183`);
spawn a derelict mech actor at the position, and give the pilot an upward `Vel.Z` plus
`Warp`/`SetOrigin` for the eject arc (`actor.zs:942`, `:753`). Forced eject on destruction is the
same call from `DamageMobj`.
**Fit check:** the pilot pawn is *smaller*, so unmorphing always fits; morphing *into* the mech
must be refused where it does not fit — test it yourself with `A_SetSize(r, h, testpos=true)`
(`p_actionfunctions.cpp:5081`, which calls `P_TestMobjLocation`) before the morph, because
`MorphPlayer` itself does not check that the new pawn's `Height` fits
(`player_morph.zs:122-130` spawns and `MorphInto`s unconditionally).
**Feel knobs:** eject `Vel.Z`, the second or two of frozen input, a black `A_SetBlend` wipe.
**Label:** BUILDABLE — **cost S**.

**Rejected: a camera-actor cockpit.** `player.camera` (`player.zs:3110`) can be pointed at another
actor, and the view then sits at `mo->Z() + GetCameraHeight()` (`r_utility.cpp:1023`,
`actor.zs:879`). **Do not.** Every VR hand path dereferences `r_viewpoint.camera->player`:
`VRMode::SetUp` (`hw_vrmodes.cpp:437-440`), `OpenXRDeviceMode::SetUp` (`hw_openxrdevice.cpp:551`,
`:570`) and `GetHandTransform` (`hw_openxrdevice.cpp:432-437`). With a non-player camera the pose
writes stop: `OffhandPos`, `AttackPos`, `HmdPosition` all freeze and the weapon/HUD model transform
falls back. **WALL — missing piece: the VR pose writers keying on `players[consoleplayer]` instead
of `r_viewpoint.camera->player`.** (That is a two-line engine change in REMA; here it is a wall.)

---

## 5. Walls

| # | Wall | Exact missing piece | Citation |
|---|---|---|---|
| W1 | No second facing in the playsim | A script-visible field applied in `AActor::Thrust`. The field **exists in C++** (`ThrustAngleOffset`) and is applied, but has no `DEFINE_FIELD` and its only writer is `#if 0`-d out. | `src/playsim/actor.h:1283`, `:1609`; `hw_openxrdevice.cpp:653-664`; no hits in `src/scripting` |
| W2 | Script cannot rotate the VR view — **no screen shake, no view kick, no camera-bob roll** | A script-settable view-angle offset applied **after** `updateHmdPose` | `hw_entrypoint.cpp:117`, `:143`; `hw_openxrdevice.cpp:740-741`, `:750`; `gl_openvr.cpp:1464`, `:1467`, `:1471`; matrix built at `hw_drawinfo.cpp:443` |
| W3 | psprite layers are hand-locked in VR, so no psprite cockpit — **including the persistent `PSP_TARGETCENTER` band**, which lands on the *main* hand | A per-layer "view-locked" flag honoured by `AdjustPlayerSprites`/`RenderHUDModel` | `hw_weapon.cpp:252-258`, `:1068-1076`; `hw_openxrdevice.cpp:405-417`; `models.cpp:219-226` |
| W4 | No script 3D translation of a psprite model | `psp->x/y` are consumed as rotations; no per-actor model offset exists in `DActorModelData` | `hw_weapon.cpp:108`; `src/playsim/actor.h:732-749` |
| W5 | `player.camera` on a non-player actor kills VR hand tracking | VR pose writers keyed on `r_viewpoint.camera->player` | `hw_openxrdevice.cpp:551`, `:570`, `:432-437` |
| W6 | A tall pawn does not fit stock maps, and we may not edit maps | `tm.ceilingz − tm.floorz < thing->Height` → blocked, no exception | `src/playsim/p_map.cpp:1997`, `:2413` |
| W7 | The eye is clamped to `ceilingz − 4` indoors, twice | An opt-out on the view clamp for a tall pawn (`VPSF_ALLOWOUTOFBOUNDS` covers only `ViewPos` cameras, `r_utility.cpp:769`). The HMD term is added *after* the clamp (`hw_entrypoint.cpp:182`), so your real head can still poke through the ceiling | `player.zs:714-721`; `r_utility.cpp:1127-1141` |
| W8 | Engine cvars (`vr_vunits_per_meter`, `vr_crouch_use_button`, `r_quakeintensity`) are menu-only for a mod | `CVAR_MOD` check in `CVar.SetFloat` | `vmnatives.cpp:981-991` |
| W9 | Haptics have **no direct script entry point** — and with W2 making view shake impossible, this is the topic's most costly gap | **A ZScript binding over the existing `VRMode::Vibrate`** — an implementation already exists on the live path (`gl_openvr.cpp:517-528`, `gl_openvr.h:121-124`, gated `vr_enable_haptics`), so this is a binding, not a feature. `VR_HapticEvent` is a separate matter: an empty stub (`qzdoom_common.cpp:65-67`). The only script-reachable route today is a quake (`r_utility.cpp:1084-1090`), which is HACK-grade | BASELINE §4; §0.6 |
| W10 | No room-scale Z: your real duck/stand does not move the pawn's Z | The positional code is XY-only and restores Z explicitly; head height is an eye offset (+ `crouchfactor` in the non-default branch) | `hw_openxrdevice.cpp:669-686`, `:219`, `:574-579`; `gl_openvr.cpp:2613-2632`. **Not a blocker for this topic** — see §0: the eye still follows the script-set base height |
| W11 | Two-hand stabilisation silently kills an offhand-weapon-driven cockpit control when the hands come within 0.50 m | A gate on `bTwoHanded` (or any opt-out a mod can set), instead of unconditional `vr_two_handed_weapons` + grip + distance | `gl_openvr.cpp:1716-1727` (`:1722`); `player.zs:393-396`; `p_pspr.cpp:512-515`, `:755-758`; `hw_weapon.cpp:832-835`, `:917-920`. **Avoidable in design** — §4.4 mitigations |

### What breaks, concretely

- **Doorways.** Stock Doom doors are 56–128 units of clear opening; a 300-unit mech is blocked by
  every one of them (`p_map.cpp:2413`). The mech is an outdoor/arena machine, or it is parked.
  Mitigation inside the rules: keep the mech's `Height` modest (96–128) and sell scale with eye
  height, cockpit framing and *monster* smallness instead — visual scale is free, collision scale
  is not.
- **Map scale.** `vr_vunits_per_meter 34` (`hw_vrmodes.cpp:94`) means a 128-unit corridor is 3.8 m.
  Nothing in a stock map is mech-sized; the illusion survives only in open areas.
- **Monster sizes.** A Cacodemon's 31-unit radius is ankle height to a 300-unit mech; its melee and
  missiles still spawn at its own `Center()` and will pass under you, and its `A_Chase` will walk
  into your radius and stop. If the mod scales monsters up (`Scale` + `A_SetSize`, `actor.zs:1314`)
  to compensate, their collision grows too and *they* stop fitting the map — the same wall, moved.
- **Steps and ledges.** `MaxStepHeight 24` (`actor.zs:422-423`) makes a giant trip on a kerb; raise
  it with the mech pawn (64–96) and the mech reads as striding instead of stumbling.
- **Reach.** `Player.UseRange 64` (`player.zs:45`) means a 300-unit mech cannot press a switch its
  own hand is beside; raise `UseRange` with the pawn (`p_map.cpp:6161`).
- **Telefrag.** `DoomPlayer` has `+TELESTOMP` (`player.zs:117`) — a big radius plus teleport lines
  will stomp things the designer never intended.

---

## 6. Smallest prototype

Four files, one evening, and it settles the only question that matters:

1. `MechPawn : DoomPlayer` with `Player.ViewHeight 240`, `Height 160`, `Radius 32`,
   `MaxStepHeight 64`, `Player.UseRange 192`, `Player.ViewBob 0`, and
   `override bool CanCrouch() { return false; }`.
2. A `MechKey` inventory item whose `Use()` calls `MorphPlayer(player, "MechPawn", 0x7FFFFFFF,
   MRF_UNDOBYDEATHFORCED)`, preceded by an `A_SetSize(32, 160, true)` fit test.
3. In `MechPawn.Tick`: every 12 tics of walked distance, `A_QuakeEx(2, 0, 0, 8, 0, 512,
   "world/quake")` on the left stride and `A_QuakeEx(0, 2, 0, 8, 0, 512, …)` on the right.
4. Three cockpit actors (`+NOINTERACTION +NOBLOCKMAP +NOGRAVITY`, any MODELDEF model) repositioned
   each tic at fixed local offsets: a frame, a throttle at the offhand's rest position and a stick
   at the main hand's, **at least 20 map units apart** (§0.8, §4.4). Grab = proximity to
   `OffhandPos` / `AttackPos` (gated on `OverrideAttackPosDir`) **plus a `cmd.buttons` bit — no
   offhand weapon anywhere in the design**, so W11 cannot arm.

**What it proves, in order:** (a) the eye really is at mech height and the HMD adds to it, not over
it — stand up and confirm the view rises *from* 240, not *to* 1.64 m; (b) the stomps rumble the
correct controller and the view does not shake; (c) a tic-positioned cockpit actor is steady enough
at 35 Hz to live in front of your face; (d) both hands grab their own control, and `AttackPos`
really is somewhere other than `OffhandPos`.
(a) is the disputed hinge (§0) and the whole topic rests on it, so **test (a) first, alone** — and
if the eye snaps back to real head height, stop and report, because then §0's disagreement is
wrong and the mech is dead on this engine.

---

## 7. Open questions

1. **Which `vr_crouch_use_button` branch does the owner actually run?** Default is `true`
   (`hw_vrmodes.cpp:122`) but it is `CVAR_ARCHIVE|CVAR_GLOBALCONFIG` and may have been changed in
   the ini. The two branches give materially different mech feel (§2.2 vs §4.2). *Settles it:*
   read the value in the owner's config; or print `player.crouchfactor` in a tic and see whether it
   tracks a real crouch.
2. **Does the `static float` height cache ever get a mech value?** `getDoomPlayerHeightWithoutCrouch`
   caches on first call (`hw_openxrdevice.cpp:162-170`). I could not determine from source which
   pawn exists at that first call in every start-up path (title level, demo, load game).
   *Settles it:* morph to the mech, exit to title, start a new game as the pilot, and check whether
   the eye height is wrong by exactly the mech's `ViewHeight − 41`.
3. **How steady is a per-tic-positioned cockpit actor at 72–90 Hz?** `RF_INTERPOLATEANGLES`
   (`models.cpp:88-91`) covers angles; positional interpolation for a `SetOrigin(…, true)` actor I
   did not trace to a line. *Settles it:* prototype item 4 above and look at it.
4. **Does `A_QuakeEx` tremor radius behave when the quaking actor is the player itself?** The
   haptic block reads the quake's intensities, not distance, but `StaticGetQuakeIntensities`
   (`r_utility.cpp:1063`) does a per-camera lookup I did not read. *Settles it:* fire one and feel
   it.
5. **Is there any way to reach `VR_HapticEvent`'s named events other than quake/damage/pickup?**
   I found the quake caller (`r_utility.cpp:1091-1092`); I did not enumerate the rest of the call
   sites. Worth one grep pass if the mech wants distinct haptic textures per event.
6. **Monster scale policy** is a design decision, not an engine one, and it is outside this lane:
   scaling monsters up to match a mech re-creates W6 for them.
7. **The §0 disagreement needs one in-engine test to close**, not more reading. Step (a) of §6
   settles it in one launch. I am confident in the citations; I am not willing to call it proven
   without the headset, per the owner's standing rule.
8. **Does the owner's build compile `USE_OPENVR`, `USE_OPENXR`, or neither?** Both are build-time
   options and `ENABLE_OPENVR` defaults to NO (`src/CMakeLists.txt:418`); `USE_OPENXR` is defined
   by no CMake file in this tree (§0.7). Everything VR-specific in this report is conditional on
   that answer. *Settles it:* the build's compile definitions, or one `vr_` cvar's presence at the
   console.
9. **Does the quake→rumble route actually buzz the controllers in that build?** `VRMode::Vibrate`
   is implemented on OpenVR but the OpenXR override calls an undefined `QzDoom_Vibrate`, and
   `VR_HapticEvent` is a stub (§0.6). *Settles it:* `A_QuakeEx(2,0,0,8,0,512)` in a test state with
   `vr_enable_haptics 1` — feel the left controller. This is a five-minute test and the whole
   stomp-feedback design leans on it, so do it before §6 step (b).
10. **Does a psprite layer whose caller is cleared, or whose caller is neither weapon, still get a
   hand matrix?** `hw_weapon.cpp:256` evaluates `GetCaller() == OffhandWeapon`, so a null caller
   yields hand 0. I did not test whether such a layer renders at all after
   `p_pspr.cpp:310-318`'s culling. If some caller value produced a *view-locked* layer, W3 would
   fall and the psprite cockpit would be back on the table — worth 20 minutes.
