# 06 — The audit applied to RSQ_TacticalRig and RSQ_LaserSight

**What this is.** Two shipped VR mods checked line by line against the engine audit
(`surveys/00_CAPABILITIES.md`, `08_BOWS_OFFHAND.md`, `99_OVERLOOKED.md`). Proposals only —
nothing here was edited, and no engine change is proposed. ZScript and data only.

**Sources.** Engine: `scratchpad/qzd173`, tag `qzd17.3`. Mods: `scratchpad/ubqs_gh`, a fresh clone
of `github.com/presidentkoopa/RSQ_UBQSeries` at HEAD `6a5c98c` (2026-10-30). Every engine claim is
`path:line` against the engine tree; every mod claim is `path:line` against that clone.

> **Note on a first pass against a stale snapshot.** An earlier draft was written against a
> 2026-10-28 zip export. Re-verified: `body.zs`, `vest.zs`, `mirror.zs`, `cig.zs`, `fit.zs`,
> `mirrorbody.zs`, `beam.zs`, `hand.zs`, `learn.zs` and `TacticalRig/CVARINFO.txt` are **byte-identical**;
> `LaserSight/sight.zs` differs by one line (a cvar default) and all its citations hold unchanged;
> `TacticalRig/hooks.zs` gained a service entry. **`TacticalRig/rig.zs` changed materially and
> reverses this report's most important TacticalRig finding** — see §1.0 and §1.5. The clone also
> contains a mod the snapshot did not: **RSQ Wheel**, which earns Part 3.

**Headline.** Neither mod is behind the audit. Both were written by someone who had already found
most of what the audit found, usually with a comment naming the engine line. In **three** places
the mods are ahead of the audit and the audit is wrong — flagged **AUDIT CORRECTION**; those
matter more than the proposals. What remains is one live regression, one latent fault, and three
cheap wins.

Labels: **EXISTS** (already in the engine, cited) · **BUILDABLE** (script/data, named APIs) ·
**HACK** (works, abuses something) · **WALL** (needs an engine change — not proposed here).
Costs: **S** an afternoon · **M** a few days · **L** a week or more.

---
---

# PART 1 — RSQ_TacticalRig

## 1.0 Recommendation

**Detect `WF_TWOHANDSTABILIZED` and defer the off-hand raise.** On 10-29 the rig's guard against
the engine's two-hand stabilisation was **deliberately removed** at the owner's instruction, and
nothing replaced it. The rig now acts on a holster in exactly the pose that makes the engine kill
the off hand, so an off-hand draw can hand the player a weapon that is invisible and inert.
Details and the proposed fix in §1.5.

The unit-scale hardening (§1.1, formerly the lead recommendation) is second, and the model
distortion found by reading the Wheel alongside the rig (§1.7) is third.

## 1.1 THE UNIT TRAP — the pixelstretch answer: the rig already applies it, everywhere

Checked first, as instructed. It comes back clean. The audit's rule
(`00_CAPABILITIES.md` §1.2a: horizontal ÷ `vr_vunits_per_meter`; vertical ÷ `vr_vunits_per_meter`
**and** × `level.pixelstretch`) is implemented correctly in every place the rig converts a
vertical. `body.zs` is byte-identical between the snapshot and HEAD, so this stands as first read:

| Direction | Mod | Expression |
|---|---|---|
| metres → map, vertical | `body.zs:169` | `upMetres * UPM / PIXELSTRETCH` |
| metres → map, eye height | `body.zs:141` | `(metres + vr_height_adjust) * UPM / PIXELSTRETCH` |
| map → metres, distance | `body.zs:176` | `(d.x, d.y, d.z * PIXELSTRETCH).Length() / UPM` |
| map → metres, edit mode | `rig.zs:148` | `d.z * PIXELSTRETCH / UPM / k` |

Also correct at `rig.zs:1028` (shoulder icon drop) and `vest.zs:87` (helmet lift).

`body.zs:138-142` matches the engine's own `getHmdAdjustedHeightInMapUnit()`
(`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:151-154`) term for term, including
`vr_height_adjust`. The comment block at `body.zs:95-104` reconstructs the engine's height
convention from source and gets it right.

**The audit's nominated highest-value finding does not land. The ~17 % vertical error is not
present.** What is present is that both scale terms are compile-time constants —
`const UPM = 34.0` and `const PIXELSTRETCH = 1.2` (`body.zs:29-30`) — where the engine uses live
values:

- **`vr_vunits_per_meter` is an archived user cvar**, default 34.0
  (`hw_vrmodes.cpp:94`), and it scales the very hand positions the rig measures against
  (`hw_openxrdevice.cpp:592-595` main hand, `:606-608` offhand; OpenVR twin
  `src/rendering/gl/stereo3d/gl_openvr.cpp:2561-2568`). Change it and the hands move away from the
  pawn by the new scale while every region point and radius stays on 34. At
  `vr_vunits_per_meter 41` the hands sit **21 % further from the body** than the slots placed for
  them, against a hip radius of 0.14 m — **every holster silently stops being reachable.**
- **`level.pixelstretch` is per-map and MAPINFO-settable** — parsed at
  `src/gamedata/g_mapinfo.cpp:1566`, defaulted 1.2 at `:314`, copied to the live level at
  `src/g_level.cpp:1890`, script-readable at `wadsrc/static/zscript/doombase.zs:479`. The engine
  reads `level.info->pixelstretch` at every hand-transform site (`hw_openxrdevice.cpp:572`, `:616`;
  `gl_openvr.cpp:1275`, `:2510`, `:2561`). On a map that sets it, the rig's vertical conversion is
  wrong by that ratio while the engine's is right.

The horizontal term is the bigger exposure, which is the opposite of the audit's warning — and
note the audit itself cites 34 units/metre as a constant where the engine treats it as a cvar.

**BUILDABLE / S.** One accessor pair read once per tic inside the existing `Settings()` cache
(`rig.zs:92-106`). The rig already demonstrates the pattern for `vr_height_adjust`
(`body.zs:139-142`) and `vr_crouch_use_button` (`body.zs:133-135`), so there is a house style.

Family context: **LaserSight reads `vr_vunits_per_meter` live and hardcodes pixelstretch
(`hand.zs:111-114`, `:20`); TacticalRig hardcodes both.** Each mod has one half.

## 1.2 AUDIT CORRECTION — `HmdPosition` is NOT the holster anchor, and the rig is right not to use it

The brief states "a holster is a body-relative volume, so `HmdPosition` is its anchor." That is
wrong, and `08_BOWS_OFFHAND.md` §4.6 already says so ("`HmdPosition` … is a different space from
`OffhandPos`, and the two cannot be subtracted"). The engine confirms §4.6:

- `HmdPosition` is assigned raw, unconverted **tracking-space metres**, Y up —
  `hw_vrmodes.cpp:445`: `player->mo->HmdPosition = DVector3(hmdPosition[0], hmdPosition[1], hmdPosition[2])`.
- `AttackPos` / `OffhandPos` are **map coordinates**, built from `r_viewpoint.CenterEyePos` and
  scaled by `vr_vunits_per_meter` — `hw_openxrdevice.cpp:592-596`.

Subtracting one from the other is meaningless. The rig anchors the body in map space —
`RSQTR_Body.Head()` returns `(pmo.pos.x, pmo.pos.y, player.viewz)` (`body.zs:84-93`) — and uses
`HmdPosition.y` **only as a scalar height**, in the two places that is legitimate: eye-height
calibration (`rig.zs:320-323`) and the seated-eye tracker (`body.zs:117-126`, `:144-147`). That is
what §4.6 prescribes.

Its floor reference is also right. The engine's hand-Z base is
`CenterEyePos.Z - getDoomPlayerHeightWithoutCrouch(player)` (`hw_openxrdevice.cpp:595-596`);
`CenterEyePos = viewPoint.Pos` (`src/rendering/r_utility.cpp:1236`); and
`getDoomPlayerHeightWithoutCrouch` returns `player->DefaultViewHeight()`
(`hw_openxrdevice.cpp:157-168`), which is `mo->ViewHeight`
(`src/playsim/d_player.h:465-468`). The rig computes `p.viewz - pmo.ViewHeight` (`body.zs:134`) —
the same quantity; its comment at `body.zs:103-104` correctly notes this keeps view bob in, as the
hands have it.

**One residual, note only.** `getDoomPlayerHeightWithoutCrouch` caches into a function-level
`static float` on first call (`hw_openxrdevice.cpp:162-166`) and never refreshes; the rig reads
`pmo.ViewHeight` live. A player class with a non-default `ViewHeight`, or a mid-session class
change, puts the two out of step and shifts the whole rig vertically. Not reproducible from the mod
side; no change proposed.

## 1.3 Room-scale Z (gotcha 4) — already handled, both modes

The audit asks whether the mod reads `vr_crouch_use_button`. It does, at `body.zs:133-135`, and it
branches as the engine does:

- button on (the default, `hw_vrmodes.cpp:122`) → floor = `viewz - ViewHeight`, so a physical duck
  does not move the rig. Matches `hw_openxrdevice.cpp:162-168`.
- button off → floor = `viewz - EyeUnits(HmdPosition.y)`, so the rig follows the head. Matches the
  engine driving `crouchfactor = getHmdAdjustedHeightInMapUnit() / defaultViewHeight`
  (`hw_openxrdevice.cpp:574-579`) and `PlayerPawn.Tick` setting `Height = FullHeight * crouchfactor`
  (`wadsrc/static/zscript/actors/player/player.zs:150-153`).

It goes further than the audit asks: `rsqtr_seated` (`body.zs:95-126`) is a third mode the audit
does not describe, for a player whose eyes and hands drop ~0.4 m while `viewz` does not.
**Already right. No change.**

## 1.4 The pickup trap — avoided by construction

The audit's point stands in the engine: `P_TouchSpecialThing` fires only when the **mover** carries
`MF_PICKUP` (`src/playsim/p_map.cpp:1737-1743`), and a tracked hand is not an actor.

It does not reach this mod. The rig never picks anything off the floor. Every path resolves through
`FindInventory` on weapons the player already owns — `Content()` (`rig.zs:402-408`), `Draw()`
(`rig.zs:564`), `DrawOff()` (`rig.zs:475`), `Owned()` (`rig.zs:169-183`), `OffCopy()`
(`rig.zs:584-593`) — and moves them with `bOffhandWeapon` + `PendingWeapon` (`rig.zs:525-526`,
`:568-569`), the engine's own route (`player.zs:2495`). `grep` over `TacticalRig/zscript` finds no
`TryPickup`, no `Touch`, no `MF_PICKUP`. **No exposure. No change.**

A hand-to-floor grab is RSQ_DistanceGrab's job and the rig cooperates rather than duplicating it
(`rig.zs:76-79`, `hooks.zs:11-25`).

## 1.5 THE TWO-HAND TRAP — the guard was removed on 10-29. This is now live and unmitigated.

**This is the finding the stale snapshot hid.** The snapshot carried a guard that refused a region
for the off hand when the off-hand grip was down, the hands were within `STEADY_RANGE = 0.50`, and
`vr_two_handed_weapons` was on. At HEAD that guard is **deleted**, replaced by a comment recording
why (`rig.zs:663-666`):

> *A HAND GRIPPING IN A HOLSTER COMES FIRST (the owner, 10-29: "when hands are gripping in holsters
> they should take precedent over everything"). Until 10-29 an off hand within half a metre of the
> gun with grip held was read as the engine's two-hand steadying and the holster was ignored; now
> the holster wins. Other mods yield through "rig.busy".*

`const STEADY_RANGE = 0.50` (`rig.zs:40`) is now dead — it is the only remaining reference in the
whole mod.

**The owner's instruction is about the rig's own arbitration, and the rig honours it correctly.**
The problem is that the engine is not a party to that arbitration. What the engine does, verified:

| Effect | Citation |
|---|---|
| `weaponStabilised` latches on the off-hand grip's **rising edge** when the controllers are within **0.50 m** and `vr_two_handed_weapons` is on (default **true**, `hw_vrmodes.cpp:119`) | `gl_openvr.cpp:1716-1727` |
| the off-hand weapon fires nothing | `player.zs:393-396` |
| its buttons are not checked | `src/playsim/p_pspr.cpp:755-757` |
| `A_WeaponReady` returns before setting `WF_OFFHANDREADY*` for hand 1 | `wadsrc/static/zscript/actors/inventory/weapons.zs:414-417` |
| every psprite layer whose **caller** is the off-hand weapon is skipped — sprite path and model path | `hw_weapon.cpp:832-835`, `:917-920` |
| `WF_TWOHANDSTABILIZED` is re-asserted on every psprite state set, so script cannot durably clear it | `p_pspr.cpp:512-515` |
| grip release clears it with **no distance test** | `gl_openvr.cpp:1724-1726` |

**The collision.** The rig's press edge *is* the stabilisation latch edge — both fire on the
off-hand grip going down. So whenever the two hands happen to be within 0.50 m at that moment, the
engine latches and the rig acts in the same tic. With the shipped layout
(`CVARINFO.txt:34-36`: hips at `l = ±0.18` m, `u = -0.70` m) the two hips are **0.36 m apart**, so
a hand resting at one hip while the other reaches the opposite hip is **inside** the window — not
an exotic pose but the ordinary cross-draw. Then:

- `DrawOff` (`rig.zs:469-481`) sets `w.bOffhandWeapon = true` and queues the raise through
  `QueueOff` → `TickPairs` → `PendingWeapon` (`rig.zs:603-628`);
- the weapon arrives in the off hand and **renders nothing** (`hw_weapon.cpp:917-920`), **fires
  nothing** (`player.zs:393-396`) and **answers no buttons** (`p_pspr.cpp:755-757`), until the
  player releases grip;
- and because release clears the latch with no distance test, the dead interval is however long
  the player holds the grip — which, for a player who holds grip to keep hold of a gun, can be the
  rest of the fight.

The player's experience: they grip at a holster, hear the rig's draw sound (`rig.zs:480`), feel the
confirming buzz (`rig.zs:844`), and get an empty hand.

**Proposed fix — BUILDABLE / S, and it keeps the owner's rule.** The holster still wins; only the
*raise* waits. `player.WeaponState` is a plain writable `native uint` (`player.zs:3073`) and
`WF_TWOHANDSTABILIZED` is a public constant (`wadsrc/static/zscript/constants.zs:1242`). The rig
already owns a deferral mechanism built for exactly this shape — `TickPairs` (`rig.zs:613-628`)
holds a queued off-hand weapon for up to 70 tics while `p.PendingWeapon != WP_NOCHANGE`. Extend
that wait condition:

> hold the queued raise while `p.WeaponState & WF_TWOHANDSTABILIZED`, and lengthen or remove
> `offWait` for that case so the weapon arrives on grip release rather than being dropped.

Optionally print once when the hold begins, per `08_BOWS_OFFHAND.md` W3's "detect and warn; never
try to clear it" — the rig already has a message channel (`rsqtr_messages`, `rig.zs:871`).

Do **not** restore the old 0.50 m geometric guard: the owner overruled it, and it was the wrong
shape anyway — it tested the live distance while the engine's latch, once set, ignores distance
entirely (`gl_openvr.cpp:1724-1726`), so the guard released before the engine did.

**Severity, stated honestly.** `weaponStabilised` is a single C++ global
(`src/QzDoom/qzdoom_common.cpp:19`) and its **only** writes in the tree are `gl_openvr.cpp:1723`
and `:1726`. The OpenXR path never sets it. So this is live on an **OpenVR/SteamVR** build and
dormant on a Quest/Android build — one patch from firing. The rig's own deleted guard is evidence
the family treated it as live until two days ago.

## 1.6 The mirror — AUDIT CORRECTION: `SetCameraToTexture` is the wrong tool here

`99_OVERLOOKED.md` §8 offers render-to-texture as "how you build a working mirror". The rig
explicitly rejected it — `mirror.zs:12-14`: *"Copies only (this machine's own actors, like the
rings): no camera texture, nothing rendered twice."* It instead spawns and mirror-maps duplicate
actors (`RSQTR_MirrorView.Copy`, `mirror.zs:40-73`; reflection at `:29-38`).

**That is the right call on this engine, for three reasons:**

1. **A camera texture is monoscopic.** Camera textures render once per frame, outside the per-eye
   loop — `src/rendering/hwrenderer/hw_entrypoint.cpp:392-403`, one `RenderViewpoint` per camtex
   with a single `FRenderViewpoint texvp`. In a headset that is a flat picture hanging in space,
   not a mirror. The actor-copy mirror is genuinely stereo because the copies are world geometry.
2. **`+ONLYVISIBLEINMIRRORS` does not work in a camera texture.** The flag is honoured only inside
   a real mirror **portal** — `isInMirror` is
   `di->mCurrentPortal && (MirrorFlag > 0 || PlaneMirrorFlag > 0)`
   (`src/rendering/hwrenderer/scene/hw_sprites.cpp:839`, used at `:844`). A camtex render sets no
   portal, so the helmet (`vest.zs:171`) would be **invisible in a camera-texture mirror** — the
   exact thing the owner asked for would not appear. The rig's workaround, clearing the flag on the
   copy (`mirror.zs:80`), is the only way to get it on screen, and it works because the copy is an
   ordinary world actor.
3. **There is no surface to put it on.** `SetCameraToTexture` resolves its target as
   `ETextureType::Wall` (`src/r_data/r_canvastexture.cpp:100`) and a cameratexture registers as
   Wall type (`src/gamedata/textures/animations.cpp:739`); an actor's `sprite` is
   `ETextureType::Sprite`. Showing one in mid-air with no map authoring means a model whose skin
   names it — a HACK on top of (1) and (2).

Also against it: `FCanvasTextureInfo::UpdateAll` renders any camtex whose `Texture->bNeedsUpdate`
is set (`r_canvastexture.cpp:120-127`), i.e. visible last frame — a full extra scene render per
frame while the mirror is up, against the rig's own cost discipline (`rig.zs:86-88` records it
already being the suite's costliest handler at 0.37-0.54 ms/tic).

**Verdict: WALL for a VR mirror.** The mod is right and the audit entry overstates the capability.
**No change.**

For contrast, the sibling API is used correctly elsewhere in the family for a different job:
LaserSight writes a **canvas** texture from `UiTick` for its range readout (`sight.zs:753-768`,
`TexMan.GetCanvas("RSLRANGE")`) — 2D drawing, no scene render. That is what render-to-texture is
cheap for here.

## 1.7 NEW — the rig's hung guns are distorted by the model pitch/pixelstretch trap

This emerged only from reading the Wheel beside the rig (Part 3) and is not in the audit.

**The engine.** For a model actor, the pixel-stretch compensation `scale(1, stretch, 1)` is applied
either **before** or **after** the actor's pitch/roll rotations, depending on one flag:

- with `MDL_CORRECTPIXELSTRETCH` → **before** the rotations, `src/r_data/models.cpp:154-158`;
- without it → **after** them, `models.cpp:195-200`.

The engine's own comment at `models.cpp:152-153` says why this matters: *"distortions might happen
depending on when the pixel stretch is compensated for so we make the 'undistorted' behavior
opt-in."* Scaling one axis after a rotation shears the mesh; the error grows with pitch. The flag
reads the live `actor->Level->info->pixelstretch` (`models.cpp:156`, `:197`), and it is reachable
from MODELDEF (`models.cpp:1111-1113`) and from ZScript —
`MDL_CORRECTPIXELSTRETCH = 1<<13` (`wadsrc/static/zscript/constants.zs:1553`) via
`SetModelFlag(int)` (`wadsrc/static/zscript/actors/actor.zs:1346`).

**The rig.** `RSQTR_Icon.Wear` sets `SetModelFlag(MDL_USEACTORPITCH | MDL_USEACTORROLL)` —
`rig.zs:1124` and `:1132` — and **not** `MDL_CORRECTPIXELSTRETCH`. Those icons are then hung at
steep angles by `Hang` (`rig.zs:1005-1017`): **pitch 80°** with roll ∓10° on the hips and saber
spots (`rig.zs:1011-1012`), **pitch 65°** on the shoulders (`rig.zs:1013-1014`). Its MODELDEF
models carry `USEACTORPITCH`/`USEACTORROLL` with no `CORRECTPIXELSTRETCH` either
(`TacticalRig/MODELDEF.txt:64`, `:168`).

So **every gun hanging on the rig is sheared**, worst where the pitch is steepest — which is the
hips, the most-used holsters. This is precisely the case the engine flag exists for.

**And the family already knows.** LaserSight's own quad ships the flag:
`CORRECTPIXELSTRETCH` in `LaserSight/MODELDEF.txt`, on the readout it pitches toward the eye
(`sight.zs:745-746`). The Wheel diagnosed the same distortion and hand-rolled a correction
(Part 3, §3.2). TacticalRig is the one of the three that neither sets the flag nor compensates.

**BUILDABLE / S.** One constant added at `rig.zs:1124` and `:1132`, and the keyword added to the
two MODELDEF blocks.

**Caveat, and it is the reason this is not the lead recommendation:** the flag changes the shape of
an already-shipped visual. Every placement number in `fit.zs`/`RSQTRFIT.txt` was measured against
the distorted draw, so the fix needs the owner's eye in the mirror before it ships — which is the
rig's own house method (`vest.zs:82-85`, `rig.zs:1019-1021`, both citing measurements taken in the
mirror).

## 1.8 The psprite band above `PSP_TARGETCENTER` — the rig does not need it

`99_OVERLOOKED.md` §1 offers the persistent band for "a torso/helmet/wrist piece that must not
vanish on weapon switch". Verified, including the traps:

- the band is iterated in its own pass from `FindPSprite(PSP_TARGETCENTER)` to the end
  (`hw_weapon.cpp:1068-1078`), while the weapon passes stop below it (`:830`, `:915`);
- `A_ClearOverlays(..., safety)` breaks before reaching it (`p_pspr.cpp:1274-1275`);
- `Caller` defaults to `PowerTargeter` (`p_pspr.cpp:310-318`), so `newcaller` must be passed;
- and the §1 correction holds — **hand-matrixed, not view-locked**: pushed into the same
  `hudsprites` array (`hw_weapon.cpp:1072`), hand chosen by caller (`hw_weapon.cpp:252-258`; HUD
  models likewise `models.cpp:219-226`), and `psp.x/y` become rotations (`hw_weapon.cpp:108`:
  `huds->mx / 4.`, `(huds->my - WEAPONTOP) / -4.`).

**The rig needs none of it.** Its torso, helmet, markers and icons are **world actors**
(`vest.zs:146-175`; `rig.zs:1072-1096`), placed in map space each tic and therefore already immune
to weapon switches and overlay clears. A psprite could not do this job: it would be stuck to a hand
instead of the body, and psprites do not appear in mirror portals — which is the helmet's whole
requirement (`vest.zs:74-77`). **No change.** Correctly unused.

## 1.9 Not gated on `OverrideAttackPosDir` — a latent desktop fault

The audit's standing instruction is to gate every hand-reading path on `OverrideAttackPosDir`
(`wadsrc/static/zscript/actors/actor.zs:276`, set at `hw_vrmodes.cpp:438` to
`!puristmode && (IsVR() || vr_override_weap_pos)`). `grep` over `TacticalRig/zscript` finds **no
read of it anywhere.**

When it is false, `hw_vrmodes.cpp:443` collapses both hands onto one point:
`AttackPos = OffhandPos = mo->PosAtZ(Center() - Floorclip + AttackOffset())` — the pawn's shoot
point, roughly chest height, permanently.

Does that fire a holster today? With the shipped defaults (`CVARINFO.txt:34-36`: hips `l = ±0.18`,
`u = -0.70`, `rad = 0.14`; shoulders `u = -0.10`) the shoot point is ~28 map units above the feet,
the hips ~21 and ±6.1 lateral, the shoulders ~38. Through `RoomMetres` that is ≈0.31 m to a hip and
≈0.35 m to a shoulder — **outside the 0.14 m radius.** So it does not misfire as shipped.

But the margin is ~0.17 m and the radius is user-scalable: `rad` is multiplied by
`rsqtr_reach * BodyScale` (`rig.zs:97`, `:104`), where `rsqtr_reach` has a menu slider to **2.0**
(`MENUDEF.txt:145`) and `BodyScale` reaches 1.3 (`rig.zs:125`). At the top of both the radius is
0.364 m and **all four gun holsters hover permanently**. That is not cosmetic: `HandBusy()`
(`rig.zs:79`) would report both hands busy forever, which by the mod's own note switches
RSQ_DistanceGrab off for good.

**BUILDABLE / S.** One early return in `WorldTick` beside the existing `rsqtr_on` check
(`rig.zs:643`) when `!pmo.OverrideAttackPosDir`, hiding the markers as `HideAll()` already does.

The Wheel does exactly this, everywhere, with graceful fallbacks (§3.1) — the in-family precedent
to copy.

## 1.10 Haptics — already on the audit's best route, and it got the trap right

`RSQTR_Buzzer` (`body.zs:208-248`) drives per-hand rumble through `A_QuakeEx`, X for the left
controller and Y for the right (`body.zs:244`), amplitude decaying, re-fired each tic — exactly
`99_OVERLOOKED.md` §1a. Engine side verified: `src/rendering/r_utility.cpp:1085-1091` routes
`jiggers.Intensity.X/Y` into `Vibrate(10, 0, left)` / `Vibrate(10, 1, right)`, with
`r_quakeintensity` defaulted to 0.0 for VR (`r_utility.cpp:116`).

Crucially the call passes **`damrad = 0`** (`body.zs:244`, fifth argument), which skips the whole
thrust-and-damage block at `src/playsim/a_quake.cpp:191-209` — the trap §1a warns about, where the
default `thrustMultiplier = 0.5` (`actor.zs:1284`) shoves the player in a random direction. No
`QF_GROUNDONLY` either. **Already right.**

Two things it inherits and cannot fix from script: it depends on the user's
`vr_quake_haptic_level` (default 0.8, `hw_vrmodes.cpp:127`), and a user who has raised
`r_quakeintensity` gets camera shake with every holster buzz. Both **WALL**; the real fix is a
ZScript binding for `VRMode::Vibrate` (`hw_vrmodes.h:94`, implemented at `gl_openvr.cpp:517-528`),
out of scope here.

The `LineAttack` route (`00_CAPABILITIES.md` §1.6a) is **not** an upgrade for the rig: it is a
fixed 150 ms / 0.8 literal, and the rig uses eight distinct strength/length pairs to distinguish
hover, holster, draw, empty, wrong-hand and edit-mode grab (`rig.zs:669`, `:689`, `:694`, `:799`,
`:810`, `:818`, `:844`, `:849`, `:865`). `A_QuakeEx` is the shapeable one and the right choice.
**No change.** (It *is* the right swap for LaserSight — §2.6.)

## 1.11 Where the rig is already right

| Audit finding | Status in the mod | Citation |
|---|---|---|
| Both hands + head are tracked | used, symmetrically | `body.zs:180`, `:44` |
| The unit trap, vertical term | applied in all six conversions | `body.zs:141`, `:169`, `:176`; `rig.zs:148`, `:1028`; `vest.zs:87` |
| `HmdPosition` is tracking-space, not map | used only as a scalar height | `rig.zs:320`, `body.zs:120-125` |
| Engine hand-Z / floor convention | reconstructed correctly from source | `body.zs:95-104`, `:129-147` |
| Room-scale Z, both cvar modes | both branches implemented, plus a seated mode | `body.zs:133-135`, `:95-126` |
| Items do not auto-pick-up | never attempts it; inventory only | `rig.zs:402-408`, `:564` |
| Per-hand haptics via `A_QuakeEx`, `damrad = 0` | exactly the §1a recipe | `body.zs:244` |
| The psprite band | correctly unused — world actors instead | `vest.zs:146-175`, `rig.zs:1072` |
| `SetCameraToTexture` for a mirror | correctly rejected, with reasons | `mirror.zs:12-14` |
| `vr_control_scheme` handedness | read, and kept distinct from layout handedness | `body.zs:184-202` |
| Per-tic cost | settings cached once per tic after a measured regression | `rig.zs:86-106` |
| Cross-mod arbitration | yields regions by service rather than special-casing | `hooks.zs:26-41`, `rig.zs:772-777` |

## 1.12 Proposals for TacticalRig

| # | Change | Label | Cost | Why |
|---|---|---|---|---|
| **R1** | Hold the queued off-hand raise in `TickPairs` while `WF_TWOHANDSTABILIZED` is set; optionally say so once | BUILDABLE | **S** | §1.5. **Live regression since 10-29**: an off-hand draw can hand the player an invisible, inert weapon for as long as grip is held. Keeps the owner's "holsters take precedence" rule |
| R2 | Read `vr_vunits_per_meter` and `level.pixelstretch` live instead of `const UPM`/`PIXELSTRETCH` | BUILDABLE | S | §1.1. Every holster unreachable if the player moves the cvar; every height wrong on a map that sets pixelstretch |
| R3 | Add `MDL_CORRECTPIXELSTRETCH` to the icon models (needs a mirror re-measure) | BUILDABLE | S | §1.7. Guns hung at 65-80° pitch are sheared; LaserSight already ships the flag, the Wheel compensates by hand |
| R4 | Early-return `WorldTick` when `!pmo.OverrideAttackPosDir` | BUILDABLE | S | §1.9. Both hands collapse to one point off VR; at high `rsqtr_reach` it locks `HandBusy` and disables Distance Grab |
| R5 | Delete the now-unused `STEADY_RANGE` (or reuse it in R1) | — | S | `rig.zs:40`, dead since 10-29 |
| — | `SetCameraToTexture` mirror | **WALL** | — | §1.6. Monoscopic, no mirror-flag support, no surface. Audit corrected |
| — | Psprite band for torso/helmet | not needed | — | §1.8 |
| — | Pickup / `MF_PICKUP` | no exposure | — | §1.4 |

**Highest-value change: R1.**

---
---

# PART 2 — RSQ_LaserSight

All citations in this part were re-verified against HEAD. `beam.zs` and `hand.zs` are
byte-identical to the snapshot; `sight.zs` differs by one line — `rsqls_spread`'s default moved
from `true` to `false` (`sight.zs:665`, `CVARINFO.txt:73`, owner 10-29: no ring) — which changes
nothing below except that the spread ring is now off by default.

## 2.0 Recommendation

**Suppress the off-hand laser while the engine has the weapon two-hand stabilised.** This is a
shipped bug and the mod already contains the switch that would fix it, defaulted off:
`rsqls_ready_only` (`sight.zs:210-215`) hides a hand's laser unless `WF_OFFHANDREADY*` is set, and
`A_WeaponReady` returns before setting those flags for hand 1 under stabilisation
(`weapons.zs:414-417`) — but it defaults **false** (`CVARINFO.txt:10`). Details in §2.4.

## 2.1 AUDIT CORRECTION — `TRF_USEWEAPON` is unusable here, and the mod is right to avoid it

The brief says `TRF_USEWEAPON` "may replace that code". It cannot, and the mod says why at
`sight.zs:404-407`:

> *TRF_ABSPOSITION with coordinates computed here and zero offsets: P_LineTrace accumulates offsets
> onto an uninitialised vector exactly when OverrideAttackPosDir is true, which in VR is always.*

**That claim is correct.** In `P_LineTrace` (`src/playsim/p_map.cpp:5165`):

- `DVector3 startpos;` is declared at `p_map.cpp:5181` and **not initialised** — `TVector3`'s
  default constructor is `constexpr TVector3() = default;`
  (`src/common/utility/vectors.h:359`), so the doubles are garbage.
- With `TRF_USEWEAPON` and `OverrideAttackPosDir`, `fromPos` is set to `AttackPos`/`OffhandPos`
  (`p_map.cpp:5185-5200`), but `startpos` is assigned only in these branches
  (`p_map.cpp:5206-5234`): `TRF_ABSPOSITION` → assigned; `TRF_ABSOFFSET` → assigned; both offsets
  zero → `startpos = fromPos`; **otherwise** — the `TRF_USEWEAPON` branch at
  `p_map.cpp:5218-5234` — it does `startpos += …`, adding to the uninitialised value.

So `TRF_USEWEAPON` is correct **only with `offsetforward == offsetside == 0`**, i.e. only when
tracing from the controller origin itself. A laser sight must trace from the **muzzle**, a measured
offset from the grip (`sight.zs:376-382`). The required call is the one that corrupts `startpos`.

A second, independent reason the zero-offset form would still be wrong: `TRF_USEWEAPON` takes its
direction from `AttackDir`/`OffhandDir` (`p_map.cpp:5190`, `:5197`) — the **controller's** forward
axis. The mod needs the **drawn gun model's barrel** axis, carried out of the HUD frame by
`HudDir` (`hand.zs:126-132`), which differs from the controller axis by the gun's MODELDEF rotation
and by `vr_weaponRotate` (default **-30°**, `hw_vrmodes.cpp:106`, folded into `weaponangles` at
`gl_openvr.cpp:1330`). Tracing the controller axis would put the dot roughly 30° off the beam drawn
on the barrel.

**Verdict: `TRF_USEWEAPON` is a WALL for this mod.** The `TRF_ABSPOSITION` route with
hand-computed coordinates (`sight.zs:431-432`, `:445-446`) is the correct and only workable one.
**No change. Audit recommendation withdrawn.**

The surrounding frame maths is right and was derived from source: `hand.zs:7-12` and `:77-85`
reconstruct `Yaw = AttackAngle + 90`, `Pitch = AttackPitch` (positive up), the HUD-model
`0.01 * (p + (0,5,30))` transform and the left-controller mirror, matching
`hw_openxrdevice.cpp:433-465`, `models.cpp:219-226` and `hw_vrmodes.cpp` as read. Two regressions
are documented in place — a double-negated pitch (`hand.zs:23-26`) and a case-insensitivity
collision between a parameter `upM` and the constant `UPM` (`hand.zs:55-59`).

**One trap cleared.** `hand.zs:29` reads `pmo.AttackRoll` and feeds it into the muzzle offset
(`hand.zs:60-67`) and `HandToRoom` (`hand.zs:100`). The Wheel carries an upstream comment warning
that *"P_PlayerThink zeroes AttackRoll every tic before WorldTick runs"*
(`Wheel/zscript/wheel/zscript.zs:7580-7582`). On **this** engine that is not so: `AttackRoll` is
written per rendered frame (`hw_openxrdevice.cpp:591`, `gl_openvr.cpp:2537`) and nothing anywhere
in `src/` or `wadsrc/` zeroes it — the only other readers are
`wadsrc/static/zscript/actors/attacks.zs:483` and `heretic/weaponphoenix.zs:191`. `MainHandRoll`
does not exist in this tree. The Wheel's own inline RSQ note says as much, and it is right.
**LaserSight's roll term is safe. No change.**

## 2.2 AUDIT CORRECTION — the wall normal is already shipped, and `LineSide` is not to be trusted

`99_OVERLOOKED.md` §7 names "`HitLine` + `LineSide` recovers a wall NORMAL, which lets the dot lie
flat on the surface instead of facing the player" as "a cheap, visible quality win". **It is already
done, and done better than the audit describes** (`sight.zs:531-546`):

```
Vector2 d = t.HitLine.delta;
Vector2 n = (d.y, -d.x);                       // the line normal
if ((n dot (muzzle.xy - end.xy)) < 0) n = -n;  // turned to face the MUZZLE
wallS = true; sAngle = VectorAngle(n.x, n.y);
at += (n.x, n.y, 0) * 0.4;                     // a hair off the wall
```

The dot is `+WALLSPRITE` on walls and `+FLATSPRITE` on floors and ceilings
(`sight.zs:547-556`, `:562`), with the glow and ring following at staggered depth offsets
(`sight.zs:593-597`, `:676-679`). That is the flat-on-the-surface dot the audit asks for, in all
three orientations, not just walls.

And `sight.zs:533-537` records that **`LineSide`'s sense could not be trusted** — it put the spot
0.4 units on the far side of the wall, where the depth test hid it. Facing the muzzle is right by
construction whichever way the linedef was drawn. **The audit should carry that correction:** take
the normal's *axis* from `HitLine.delta` and its *sign* from a muzzle-facing dot product, not from
`LineSide`.

**Cost of the already-shipped win: paid. No change.** Fields verified at
`wadsrc/static/zscript/actors/actor.zs:48` (`HitLine`), `:58` (`LineSide`), `:41-46` (`HitType`).

## 2.3 The remaining §7 win — `HitTexture` for material-aware dots

`grep` over `LaserSight/zscript` finds **no use of `HitTexture`**. It is declared at `actor.zs:53`
and populated by the same trace the mod already runs. This is the one audit capability the mod does
not yet use.

Sky is already handled, by `HitType` rather than texture — `TRACE_HasHitSky` at `sight.zs:449`,
dot suppressed at `:525`. So "no dot on sky" is done.

What `HitTexture` adds at no extra trace cost: a different dot on glass, liquid, grating and metal
(scale, alpha, or a second sprite keyed off the texture name), and a way to tell a skybox from a
mid-texture the dot *should* paint on. A data-driven table fits the house style — the pack already
ships two large generated data lumps (`docs/MUZZLES.txt`, `zscript/rsqls/muzzles.zs`) and a
Service-based extension point for weapon packs (`sight.zs:686-715`).

**BUILDABLE / S** for the lookup and a small table; **M** if the table must cover more than a
handful of name patterns across the packs in the owner's rotation. The trace already returns the
field; the cost is authoring, not code.

**Caveat to state up front:** `HitTexture` is a `TextureID`, so matching means resolving names via
`TexMan.GetName()` per hit, per tic, per hand. Cache per `TextureID`, the way the mod already
caches the muzzle Service per weapon class (`sight.zs:695-715`) and the fade translations per key
(`beam.zs:120-145`) — this pack is explicit about per-tic cost.

## 2.4 THE TWO-HAND TRAP — the off-hand laser does not die when the player braces. Half of it survives.

Answering the brief directly: **the beam dies and the dot does not.**

The engine's behaviour under `weaponStabilised` is tabulated in §1.5. Against it:

`HandLive(p, 1)` (`sight.zs:193-217`) tests `rsqls_off`, that `p.OffhandWeapon` is non-null, the
melee/`CHEATNOTWEAPON` filter, and — only if `rsqls_ready_only` is set — the ready flags.
`p.OffhandWeapon` **stays set** under stabilisation; nothing nulls it. So with shipped defaults the
off-hand laser is drawn. Then:

- **the beam disappears**, because it is a psprite at layer `LAYER_OFF = 1000140` (`beam.zs:40`,
  `:165`) whose `Caller` is set to the off-hand weapon (`beam.zs:183`), and
  `1000140 < PSP_TARGETCENTER`, so `hw_weapon.cpp:917-920` skips it. The mod knows and says so at
  `beam.zs:23-25`.
- **the dot, glow, dynamic light, spread ring and range readout keep drawing.** They are world
  actors and an attached light — `RSQLS_Spot`/`RSQLS_Glow` (`sight.zs:30-46`), placed at
  `sight.zs:562-564`, lit at `:613`, ring at `:670-683`, readout at `:730-742`. Nothing in
  `hw_weapon.cpp` touches them.
- and the trace behind them still runs from `OffhandPos` along the **off-hand controller's** axis
  (`sight.zs:385-392`) — which during a brace points wherever the support hand points, not down the
  gun.

**What the player sees:** they bring the support hand to the rifle, the off-hand gun and its beam
vanish (engine), and a second laser dot plus a coloured dynamic light goes on wandering the room,
emitted by nothing visible, pointing nowhere they are aiming. It also keeps `liveMask` bit 1 set
(`sight.zs:350`), so `RSQLS_SightService` (`sight.zs:788-798`) reports a live off-hand laser to
RSQ_DistanceGrab, which then stands its grab beam down on a hand that has no laser.

**The fix — BUILDABLE / S.** Add one test to `HandLive` (`sight.zs:193-217`): for `h == 1`, return
false when `p.WeaponState & WF_TWOHANDSTABILIZED` (`player.zs:3073`, `constants.zs:1242`). That
hides beam, dot, glow, light, ring and readout together and clears the service bit, so
RSQ_DistanceGrab gets its hand back. One bitwise test per tic. Do **not** try to clear the flag —
`p_pspr.cpp:512-515` re-asserts it from the C++ global on every psprite state set.

The cheaper-looking alternative — defaulting `rsqls_ready_only` to true — is **not** equivalent and
should not be used for this: it also hides the laser during every reload and weapon switch on both
hands, a visible behaviour change nobody asked for. Keep the cvar and add the explicit test.

**Severity** is as in §1.5: live on an OpenVR/SteamVR build, dormant on Quest/OpenXR, one patch
from firing.

## 2.5 Collision is an axis-aligned square — real but marginal for the dot

Verified: the actor-vs-actor test is Chebyshev (`p_map.cpp:1417-1419`) and the trace path uses the
same shape — `src/playsim/p_trace.cpp:695-697` rejects a hit whose
`|hit.X - thing->X()| > radius || |hit.Y - thing->Y()| > radius`. Nothing reads `Angles.Yaw`, so
the footprint is a `2r × 2r` square and unrotatable.

**For the dot:** `t.HitLocation` on an actor lies on that square's side plane, not on the sprite's
visual plane. At an oblique approach the corner stands up to `r(√2 − 1) ≈ 0.41 r` proud of the
sprite — about 8 units ≈ 24 cm for a 20-unit-radius imp. The mod pulls the dot *toward* the muzzle
by `2.0 / len` of the ray (`sight.zs:559-561`), the right direction but a fixed 2 units, so on an
oblique hit the dot can read as floating slightly off the monster.

**BUILDABLE / S**, and honestly **low priority**: on a billboard dot at combat range this is at the
edge of visible, and a radius-aware pull-back would be guesswork about sprite width that the audit
cannot ground. Recorded, not worth doing ahead of §2.3 and §2.4. On walls, floors and ceilings —
where the geometry *is* exact — the handling is already correct (§2.2).

## 2.6 Haptics — the audit's `LineAttack` route is a real simplification here

`Buzz` (`sight.zs:220-225`) spawns `RSQLS_Buzz` via `SpawnPlayerMissile` with `ALF_ISOFFHAND` to
provoke the engine's firing rumble. Verified: `P_SpawnPlayerMissile` calls
`Vibrate(150, hand, 0.8)` when the spawned actor has a damage function, a `DamageType`, or a
`SeeSound` (`src/playsim/p_mobj.cpp:7545-7563`), and `RSQLS_Buzz` carries
`DamageType "RSQLSHaptic"` (`sight.zs:79`) to satisfy it. The class comment (`sight.zs:68-72`)
explains the care taken to avoid tripping BD22Q's recoil hooks.

`00_CAPABILITIES.md` §1.6a's `LineAttack` route is strictly simpler:
`p_map.cpp:4759-4776` buzzes the firing hand on `damage > 0`, flipped by `LAF_ISOFFHAND` at
`:4767`, **before** any `LAF_NOINTERACT` handling, so

```
LineAttack(angle, 0.1, pitch, 1, "None", "NullPuff", LAF_NOINTERACT | LAF_ISOFFHAND)
```

gives a 150 ms / 0.8 pulse on the chosen hand with no world effect. Both routes produce the
**identical literal** pulse — the same `Vibrate(150, hand, 0.8)` — so there is nothing to tune
differently, and the toggle tick needs no envelope.

What it removes: a spawned actor per buzz; `WorldThingSpawned` reaching every listener in the load
order; the `P_CheckMissileSpawn` success requirement (`p_mobj.cpp:7545`), which can fail when the
hand is inside geometry and silently eat the buzz; and the invented `DamageType`. It respects the
user's handedness through `vr_control_scheme` the same way.

**EXISTS / S.** Genuine simplification, no behaviour change. Lower value than §2.3 and §2.4.

(Contrast §1.10: the rig needs shaped envelopes and eight distinct strengths, so `A_QuakeEx` is
right there; LaserSight needs one identical tick, so `LineAttack` is right here.)

## 2.7 `OverrideAttackPosDir` — mentioned, never read; degrades acceptably

`grep` finds it only in comments (`hand.zs:12`, `sight.zs:373`, `:407`). Off VR,
`hw_vrmodes.cpp:443-444` sets both hands to the pawn's shoot point with
`AttackAngle = viewyaw − 90` and `AttackPitch = −viewpitch`, so `RSQLS_Hand.Grip`/`Dir`
(`hand.zs:30`, `:53`) yield the shoot point and the view direction — a laser down the crosshair,
which is a reasonable thing to draw. Dot, glow and light are world actors and behave.

The beam does not: psprite hand-matrixing is gated on `vrmode->IsVR()`
(`hw_weapon.cpp:252-253`), so off VR the rod draws as a flat 2D overlay at `psp.x/y` instead of
riding the barrel. These are QuestZDoom VR packs and the mod is explicit about that (`hand.zs:1`,
`sight.zs:3-6`), so this is a **note**, not a proposal. If a desktop build ever matters, the gate
belongs in `HandLive` (`sight.zs:193`) against `rsqls_beam` only.

## 2.8 Where LaserSight is already right

| Audit finding | Status in the mod | Citation |
|---|---|---|
| Psprite ids are arbitrary ints with huge gaps (§2) | used, and the whole load order audited for collisions | `beam.zs:27-31`, `:39-40` |
| The hand is chosen by the layer's **Caller**, not its id | the beam's `Caller` is the gun, so switchhand carries it | `beam.zs:8-10`, `:183`; engine `hw_weapon.cpp:256`, `models.cpp:219-226` |
| `HitLine` gives a wall normal; the dot lies flat | shipped — walls **and** floors **and** ceilings | `sight.zs:531-562` |
| `LineSide`'s sense | found unreliable; muzzle-facing dot product instead | `sight.zs:533-542` — **corrects the audit** |
| Sky must not take a dot | `TRACE_HasHitSky` suppresses it | `sight.zs:449`, `:525` |
| `TRF_USEWEAPON` substitutes the hand origin | correctly rejected; `startpos` is uninitialised with offsets | `sight.zs:404-407` — **corrects the audit** |
| The unit trap, horizontal term | reads the live `vr_vunits_per_meter` | `hand.zs:111-114`, `sight.zs:453-456` |
| Model pitch/pixelstretch distortion | `CORRECTPIXELSTRETCH` on the pitched readout quad | `LaserSight/MODELDEF.txt`; engine `models.cpp:152-158` |
| Interpolation: write every tic, `SetOrigin(pos, true)` | done, with an explicit `ClearInterpolation()` on un-hide so a piece does not streak | `sight.zs:158-163` |
| 35 Hz sampling of a per-frame value | the beam moved onto a psprite precisely because a world actor trailed the gun | `beam.zs:3-6` |
| Render-to-texture (the 2D half) | canvas texture for the range readout, written in `UiTick` | `sight.zs:753-768` |
| Services, found by name, no hard link | muzzle, colour and status all extension points | `sight.zs:273-279`, `:695-715`, `:788-798` |

## 2.9 Proposals for LaserSight

| # | Change | Label | Cost | Why |
|---|---|---|---|---|
| **L1** | In `HandLive`, for `h == 1`, return false on `p.WeaponState & WF_TWOHANDSTABILIZED` | BUILDABLE | **S** | §2.4. Shipped bug: the beam dies but dot, light, ring and readout keep drawing from an invisible gun, and the status service lies to Distance Grab |
| L2 | Material-aware dot from `HitTexture`, with a cached name table | BUILDABLE | S–M | §2.3. The one audit capability unused; same trace, authoring cost only |
| L3 | Replace the `RSQLS_Buzz` spawn with `LineAttack(..., LAF_NOINTERACT \| LAF_ISOFFHAND)` | EXISTS | S | §2.6. Identical pulse, no spawned actor, no `WorldThingSpawned`, no spawn-failure hole |
| L4 | Radius-aware pull-back for the dot on an actor | BUILDABLE | S | §2.5. Real but marginal |
| — | `TRF_USEWEAPON` / `TRF_ISOFFHAND` | **WALL** | — | §2.1. `startpos` uninitialised with nonzero offsets (`p_map.cpp:5218-5234`, `vectors.h:359`). Audit recommendation withdrawn |
| — | Wall normal from `HitLine` | already shipped | — | §2.2 |
| — | Psprite band above `PSP_TARGETCENTER` | not wanted | — | It *would* survive stabilisation (`hw_weapon.cpp:1068` has no skip), but a beam riding a gun the engine has stopped drawing is wrong. L1 is the fix |

**Highest-value change: L1.**

---
---

# PART 3 — RSQ Wheel (short, and it earns it)

Not in the brief's original scope; read because it is in the clone and because the coordinator
asked whether it duplicates holster logic or gets the vertical maths right where the rig does not.

## 3.1 The hypothesis is half wrong, and the half that is right is a different trap

**Wrong half: the Wheel is not a body-relative surface and does not touch `HmdPosition`.** It is
**hand-anchored** — a ring planted in front of whichever hand opened it, at `wr_forward` map units
(default 34.0, `zscript.zs:3280`, `:5787`, guarded against 0 and negatives at `:5795`), with the
anchor frozen once per ring-open and wall-checked by a `LineTracer` from the hand's own sector
(`zscript.zs:5800-5830`). `grep` over `Wheel/zscript` returns **zero** hits for `HmdPosition`,
`pixelstretch` and `vr_vunits_per_meter`. It works in map units throughout and so never enters the
metres↔units conversion the audit's §1.2a is about. **The measurement form of the pixelstretch trap
does not reach it.**

**It does not duplicate holster logic either — it consumes the rig's.** `zscript.zs:1527-1540`
calls `RSQTR_RigService`'s `"rig.holster"` to badge a card with where that gun hangs, and
`zscript.zs:9083` treats a sibling sitting in a holster as not-free. That service entry is one of
the two additions `hooks.zs` gained since the snapshot (`hooks.zs:21-23`, implemented `:80-87`,
with `"rig.regionname"` at `:92`). The two mods are already composed, by name, with no hard link.
**Nothing to report as duplication.**

**Right half, and it is the finding:** both are exposed to the pixelstretch trap, but in its
**render** form rather than its measurement form — and that is §1.7.

## 3.2 The model distortion: three mods, three different answers

`models.cpp:152-200` applies the pixel-stretch compensation `scale(1, stretch, 1)` **before** the
actor's pitch/roll rotations when `MDL_CORRECTPIXELSTRETCH` is set (`:154-158`) and **after** them
when it is not (`:195-200`) — the engine's own comment calling the undistorted behaviour "opt-in".
After a rotation, a one-axis scale shears the mesh, worse the steeper the pitch.

| Mod | Pitched models | `CORRECTPIXELSTRETCH`? | What it does |
|---|---|---|---|
| **LaserSight** | readout quad, pitched at the eye (`sight.zs:745-746`) | **yes** (`LaserSight/MODELDEF.txt`) | correct, by the engine flag |
| **Wheel** | every card, label and bar — `wr_BBQuad`/`wr_BBStrip`, `USEACTORPITCH`/`USEACTORROLL` (`Wheel/MODELDEF.txt`) | **no** | compensates in script: `const CARD_STRETCH = 1.2` (`zscript.zs:9384`) and `fitIcon` (`zscript.zs:4711-4725`), with the diagnosis written out correctly at `:4716-4718`, `:3699`, `:5364-5365`, `:6257-6258` |
| **TacticalRig** | stored guns hung at **pitch 80°** (hips) and **65°** (shoulders), `SetModelFlag(MDL_USEACTORPITCH \| MDL_USEACTORROLL)` (`rig.zs:1124`, `:1132`, `:1011-1014`) | **no** | nothing |

So the Wheel found the trap and paid for it by hand; LaserSight uses the flag; **TacticalRig is
the one that neither notices nor corrects, on the models it pitches hardest.** That is R3 in §1.12,
and it would not have been visible from either mod alone.

Two notes on the Wheel's own version: its `1.2` is hardcoded where the engine reads the live
`level.info->pixelstretch` (`models.cpp:156`, `:197`), so it is wrong on a map that sets
pixelstretch; and one MODELDEF keyword on `wr_BBQuad`/`wr_BBStrip` would replace the whole
script-side correction against the live value. Both are **BUILDABLE / S** — but the Wheel's card
layout constants were tuned against the hand-corrected draw, so this is the same
re-measure-before-shipping caveat as R3, and it is the Wheel lane's call, not this report's.

## 3.3 Where the Wheel is the in-family example to copy

**It gates every hand read on `OverrideAttackPosDir`, with real fallbacks** — the thing TacticalRig
does nowhere (§1.9): `handPos` (`zscript.zs:7501-7515`) synthesises a hand 22 units forward,
11 to the side and 8 down from the view when tracking is absent, with the reason written out at
`zscript.zs:6566-6571` (*"Without it they hold the actor's ORIGIN — its feet — so panels anchored
to them sit on the floor at ankle height and read as broken rather than absent"*); and
`handAngle` (`:7521-7525`), `handAim` (`:7538-7548`), `handDir` (`:7550-7563`), `handPitchOf`
(`:7566-7570`) and `handRollOf` (`:7574-7582`) each branch on it too, as does
`wr_q_watch.zs:36`. It also documents the `AttackPitch` sign convention and names the engine's own
wheel as getting it wrong (`zscript.zs:7527-7536`).

That is the pattern R4 proposes for TacticalRig, already written, in the same codebase.

**No proposals for the Wheel** beyond the optional MODELDEF keyword in §3.2. It was read as a
comparison, not audited.

---
---

## Open questions (not answerable from source)

1. **Which VR backend the owner's binary was built with.** `00_CAPABILITIES.md` §0 stands: on PC
   `USE_OPENXR` is defined only in `mobile/Android_src.mk:15` and `USE_OPENVR` defaults off
   (`src/CMakeLists.txt:418-420`). This decides whether R1 and L1 are live bugs or dormant ones.
   All three mods assume live hand poses and all three work, which is itself evidence the poses are
   populated.
2. **Whether `vr_two_handed_weapons` is on in the owner's config** (engine default true). With the
   rig's guard gone (§1.5) this is now the single variable that decides whether R1 is urgent.
3. **Whether the owner has ever moved `vr_vunits_per_meter` off 34.** Decides whether R2 is a fix
   or a hardening.
4. **Whether any map in the rotation sets `pixelstretch` in MAPINFO.** Decides how much of R2's
   vertical half, and of §3.2's hardcoded 1.2, is theoretical.
5. **Whether R3 is wanted at all**, given every `RSQTRFIT` placement number was measured against
   the distorted draw. This is an owner's-eye call in the mirror, not a code question.
