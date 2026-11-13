# 07 — APPLY: RSQ Wheel against the engine audit

**Mod:** `github.com/presidentkoopa/RSQ_UBQSeries`, `Wheel/`, HEAD `6a5c98c`, 2026-10-30.
16,687 lines of ZScript; `zscript/wheel/zscript.zs` alone is 9,634.
**Engine:** tag `qzd17.3` of `emawind84/gzdoom`. All engine citations are repo-relative at that tag.
**Read first:** `surveys/00_CAPABILITIES.md`, `surveys/99_OVERLOOKED.md`, `surveys/10_TIME.md`,
`apply/05_SLOWMO.md` §8.1 (which found the Wheel interaction and is built on here, not re-derived).

**Scope:** recommendations only. Nothing in the Wheel's tree was written to. No engine change is
proposed anywhere in this file; every item is ZScript or a data lump.

---

## 0. Recommendation, up front — ranked, cheapest high-value first

| # | Recommendation | Label | Cost | Where |
|---|---|---|---|---|
| **1** | **Refuse / reroute an off-hand pick while `WF_TWOHANDSTABILIZED` is latched.** The ring cannot vanish, but the gun it delivers to the off hand can be invisible and dead, with every one of the Wheel's own confirmations firing normally. This is the shipped bug. | **BUILDABLE** | **S** | §4 |
| **2** | **Ask SlowMo whether it was already on before sending `rsm_off`.** Verified by the owner; closing the ring cancels a slow motion the player was holding. | **BUILDABLE** (bug fix) | **S** | §5.3 |
| **3** | **Two `Map`s to replace two linear scans in the per-tic path:** `grp()` over `mGroups` (called from inside `resolve()`), and `TransFor()`'s translation cache (called per part per tic from `Tint`). Mechanical, no behaviour change. | **BUILDABLE** | **S** | §8.2 |
| **4** | **Cache the resolved billboard basis once per tic.** `resolve()` is re-run from scratch in roughly nine O(N) sweeps a tic, each with 4–8 trig calls and a group scan. `place()` already resolves every record every tic; store the result and let the hit tests read it. | **BUILDABLE** | **S–M** | §8.3 |
| **5** | **Apply the `pixelstretch` term to the two tests written in real-world metres** — the wrist-card pose window and the gesture speeds — **before** they are tuned on a headset, so the tuning is not absorbing a 17% error. | **BUILDABLE** | **S** | §2 |
| **6** | **Fall back to the `LineAttack` pulse when `vr_quake_haptic_level` is 0.** Today the whole detent-and-commit feel silently disappears for any player who has zeroed that cvar. | **BUILDABLE** | **S** | §7.3 |
| **7** | **Get the ring's quads out of SlowMo's per-tic sweep** (rename `wr_*` → `rsqwr_*`, 3 MODELDEF lines + the class declarations). Cost leak only, not a correctness bug. | **BUILDABLE** | **S** | §5.4 |
| **8** | **Give the ring a time tool of its own for the no-SlowMo case:** `Level.SetFrozen` + `player.timefreezer`, replacing the `bt_activate` flip the mod's own comment already calls broken. Do **not** replace the SlowMo path with it — see §5. | **BUILDABLE** | **S** | §5.5 |
| **9** | **Consume `RSQLS_SightService`** so the ring's beam and RSQ_LaserSight's off-hand laser do not both come out of the same hand. | **BUILDABLE** | **S** | §9.2 |
| **10** | **Measure before anything else in §8 is spent on:** `profilethinkers -t 20` and `stat think`, ring open and closed, in a Brutal Doom fight. | **EXISTS** | **S** | §8.1 |

**Where the mod is already right, and should be left alone:** the pointing path is genuine 6DOF, not
yaw (§3); the wheel button is the thumbstick, never the grip, which is exactly the survey's
recommended defence against the two-hand trap (§4.2); drawing is world actors rather than psprites
or a canvas, and the reasoning is documented (§6); detent haptics already exist, weight-shaped, with
the `A_QuakeEx` safety arguments correct (§7); the time suppression is `CF_TOTALLYFROZEN`, which is
one bit and the right one (§5.1); the pickup-voxel sizing is not pixelstretch-exposed (§2.4); and
nine of the mod's per-card cvar reads are already hoisted out of the per-card loop (§8.4).

---

## 1. What the wheel is today — the baseline

### 1.1 How it opens

Three doors, all through one handler. `NetworkProcess` (`zscript/wheel/zscript.zs:702`) gates on
`e.Player != consoleplayer` (`:709`) and then on the master switch `wr_wheel_mode`
(`:729-745`, mirrored onto `wr_suppress_native_wheel` so the engine's own wheel is suppressed
unless the player asked for it):

- `wr_press` → `pressWheel(down)` (`:757`, `:931`) — **the default bind, and the designed one.**
  `KEYCONF:` `defaultbind LThumb +wr_wheel`. Hold: the ring is up while the button is down and the
  release picks whatever is under a hand (`:962-973`). Tap: the ring stays up (`:974`). Tap twice
  inside `wr_doubletap` (10 tics): `swapToPrevious` puts the last-picking hand back on the gun it
  had (`:940-947`, `:969`).
- `wr_toggle_main` / `wr_toggle_off` / `wr_toggle_any` → `toggle(hand)` (`:751-756`, `:1021`).
  Defaults are `[` and `]`, which take only on the VR profiles — KEYCONF documents the collision
  with `invprev`/`invnext` at length.
- `mWantAutoOpen` → `openRig(1)` on the first live tic (`:7589-7597`).

`openRig` (`:1048`) sets `mRigHand`/`mPokeHand`, clears the anchor, gathers weapons, spawns the
panels, card models, sheet and stars, claims the sticks via `wr_VR.SuppressVRInput(true)` (`:1109`),
turns the laser on, calls `bulletTime(true)` (`:1112`), lays out, and animates every group from 0 to
1 over `wr_growtics` (`:1122-1130`).

A same-hand repress inside `wr_recenter_window` (10 tics) clears `mHaveAnchor` and re-places the
ring instead of closing it (`:1024-1032`). An idle ring folds itself away after `wr_locktics`
(140 tics) with a haptic warning at `wr_warn_tics` (`:7679-7694`).

### 1.2 How a segment is selected

Resolved fresh every tic in `wr_Rig.WorldTick`, in a deliberate order (`:7755-7805`):

1. **Hand hand-off.** If the poke hand is not on the ring and the other hand is,
   `mPokeHand` moves across (`:7760-7765`) — `handOnRing` (`:8134`) is an `AimBillboard` plus a
   `TouchBillboard` from the fingertip.
2. **The ray.** `org = handPos(pmo, mPokeHand)`, `dir = handDir(pmo, mPokeHand)`, then
   `wr_BB.AimBillboard(org, dir, POINTER_RANGE)` (`:7768-7780`). Aim is one-to-one; the comment at
   `:7770-7774` records that an angular gain was removed because it multiplied tremor as much as
   intent.
3. **Stick selection** overrides the ray whenever the stick is deflected (`:7790-7793`) — fixed
   bearings, so a push maps straight to a slot. An accessibility path, not the primary one.
4. **Touch wins over beam** (`:7796-…`) — the fingertip, at `org + dir * mTouchReach`.

`mTouchReach` is frozen with the anchor at `clearForward - wr_touch * 1.2` (`:5878`) so a hand
standing still touches nothing and a few centimetres of real push carries it in.

The result goes through `updateHover(hit)` (`:8826`), which holds the dwell clock, grace-periods a
fan (`:8845-8854`), resets `mLockTics` every tic a hand stays on a card (`:8878`), opens a fan at
`DWELL_TO_EXPAND` (`:8897-8912`), and fires the landing click (`:8947-8961`).

The hit test itself is `wr_BB.AimBillboardT` (`wr_q_shim.zs:258-281`): a plane intersection per
billboard record, nearest wins, with the UV returned. `TouchBillboard` (`:287-307`) is the same
against a point.

### 1.3 How it draws

**World model actors, not psprites and not a canvas.** Every billboard is one or more
`wr_BBQuad` / `wr_BBStrip` actors (`wr_q_shim.zs:73`, `:91`) drawn from `MODELDEF.txt:4`, `:25`
(`models/wrq/quad.obj`, `strip4/8/12/24.obj`). Text is **one actor per line**, a strip whose cells
are skinned per surface with `CMDL_USESURFACESKIN` (`wr_q_shim.zs:518`, `:528`) — not one actor per
character.

`wr_BB.WorldTick` (`wr_q_shim.zs:584`) runs an expire pass and then a build/place pass over
`mList`. `place()` (`:605`) resolves the record's centre and basis, applies the group scale, lifts
later records a hair toward the viewer (`LAYER_STEP * (order % 64)`, `:612`), tints, scales and
`SetOrigin(at, true)` — interpolated, which is the right call per 00_CAPABILITIES gotcha 12.

Everything is the WAD's own art through TEXTURES, by lump name: STBAR's health box for the plates
(`WRPLATE`), DoomFont cells for words (`WRGxxx`), the status bar's numerals for numbers (`WRDxxx`),
coloured by a run-time palette translation built from the palette's own luminance table
(`wr_q_shim.zs:440-465`, `wr_q_pallum.zs`). This is `WHEEL_WORK_QUEUE.md` item 3, done.

Pickup voxels stand on the cards when the gun has one: measured from its own VOXELDEF `Scale` and
KVX dimensions (`wr_q_vox.zs`) and fitted inside the card at
`min(panelW*0.70/vs.x, panelH*0.55/vs.y)` (`zscript.zs:8676`). No voxel, no guess — the card keeps
its flat icon (`:8681`).

### 1.4 How it commits a weapon switch

`commit` (`:9133`) → `commitAs` (`:9146`) → `equipInstantly` (`:8973`).

`commit` resolves the hand: with either-hand picking on, the gun goes to the hand that *pointed*
(`mPokeHand`), unless it is `bNoHandSwitch`, which pins it (`:9137-9143`).

`commitAs` records the outgoing weapon for the double-tap (`:9149-9156`), then handles picking the
gun the other hand already holds — a free `_2`…`_6` twin first, a two-hand swap second
(`:9183-9211`) — fires the confirm (`feedback(lock, 0.75, 90)`, or `nope, 0.35, 60` on a dry pick,
`:9221-9222`), bursts the card and arms its flip (`:9234-9266`), and finishes with
`equipInstantly` + `closeRig(true)`.

`equipInstantly` (`:8973-9010`) is deliberately not `MoveWeaponToHand`: that ends in `DropWeapon`
and the gun would arrive half a second late. It mirrors `bOffhandWeapon` onto the sister weapon,
honours `bNoHandSwitch`, sets `player.OffhandWeapon` or `ReadyWeapon`, and then
`player.SetPsprite(wantOffhand ? PSP_OFFHANDWEAPON : PSP_WEAPON, weap.GetReadyState())` (`:9002-9003`).

Two gestures also commit, both from `tickPhysical` (`:8034`): **pull a card out** (a hand inside a
card snapping back, `wr_pull_speed` 1.5 u/tic) and **throw a gun back** (`fileAway`, `:8093`, a
snap toward the ring on no card, `wr_throw_speed` 2.6).

### 1.5 What it does to time while open

Two separate things, and the distinction matters for §5:

1. **The player is frozen, the world is not.** `wr_VR.SuppressVRInput(true)` sets
   `p.cheats |= CF_TOTALLYFROZEN` (`wr_q_shim.zs:727-731`) and remembers that it was the one who
   set it (`h.mFroze`), so it never thaws a freeze it did not make. Engine side,
   `PlayerPawn.CheckFrozen` (`wadsrc/static/zscript/actors/player/player.zs:1103-1134`) reduces
   `cmd.buttons` to `BT_USE` and zeroes `forwardmove`/`sidemove`/`upmove`. Locomotion, snap turn
   and the weapons all stop; `+use` survives, which is exactly what the Wheel's grab needs
   (`zscript.zs:898`).
2. **The world is slowed, not stopped,** and only if SlowMo is loaded. `bulletTime(bool)`
   (`:4045-4056`) finds `RSQSM_Service` and sends `rsm_on` / `rsm_off`; with no SlowMo it sends
   BulletTimeX's `bt_activate`. The README states the design: *"The ring holds you still while it
   is up and, with RSQ SlowMo loaded, slows the world."*

### 1.6 What the Quest/VR seam actually reads

`wr_q_shim.zs` is three parts, named in its own header: `wr_BB` (the billboard system that replaces
REMA's engine billboards), `wr_Field` (REMA's field reflection — **every read returns false**,
`:655-668`, which is why the Lithium / DOOM Infinite / Pandemonia compat readers find nothing and
the sheet shows universal rows only), and `wr_VR`.

`wr_VR` reads and writes exactly this much:

| What | Where | Engine surface |
|---|---|---|
| handedness | `wr_q_shim.zs:697-701` | `vr_control_scheme >= 10` |
| haptics | `:707-721` → `wr_Buzz.Tick` `:676-688` | `A_QuakeEx` with `QF_WAVE`, X = left hand, Y = right |
| input suppression | `:727-742` | `player.cheats |= CF_TOTALLYFROZEN` |
| raw stick | `:747-753` | `p.original_cmd.forwardmove / 12800`, `.sidemove / 10240` |
| the beam | `:758-767`, `wr_VRState.WorldTick` `:784-826` | its own quad actor, plus `LineTrace(…, TRF_THRUACTORS \| TRF_ABSPOSITION, org.z, org.x, org.y)` |

The hand poses themselves come from `wr_Rig.handPos` / `handDir` / `handRollOf`
(`zscript.zs:7501`, `:7548`, `:7574`), all gated on `pmo.OverrideAttackPosDir`
(`wadsrc/static/zscript/actors/actor.zs:276`) with a desk-testable fallback that points where the
player looks. Three things worth noting as *correct*:

- The Z-sign trap is handled, and the comment at `:7530-7536` is right: the stored pitch convention
  is up-positive, so `handDir` returns `+sin(p)`, matching `hw_weapon.cpp`'s double negation rather
  than the engine's own wheel.
- `handAngle` adds the +90° the engine's own convention requires (`:7520-7525`), matching
  `AttackAngle = -90 + viewYaw + (weaponangles[YAW] - playerYaw)`
  (`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:590`,
  `src/rendering/gl/stereo3d/gl_openvr.cpp:2535`).
- Roll reads `AttackRoll`/`OffhandRoll` directly with a comment that QZD writes `AttackRoll` per
  frame (`:7580`). Verified: `hw_openxrdevice.cpp:593` and `gl_openvr.cpp:2536`.

`wr_FX` (`:1113-1133` of the shim) is REMA's shader dressing — volumetric beam, fog slab, sweep
rings, glow wave, shape slots — **all no-ops**, which the README states and marks "REMA only".

---

## 2. THE PIXELSTRETCH QUESTION

**Answer: the mod is mostly not exposed, and where it is exposed it is WRONG — in exactly two
places, both of which the README already flags as untuned. Neither is the voxel sizing.**

### 2.1 The engine fact, verified on both paths

Horizontal is `vr_vunits_per_meter` alone
(`src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:94`, `34.0f`). Vertical additionally divides
by `level.pixelstretch`:

```
hw_openxrdevice.cpp:597   AttackPos.Z  = CenterEyePos.Z + (((hmdPosition[1] + weaponoffset[1] + vr_height_adjust) * vr_vunits_per_meter) / pixelstretch) - heightWithoutCrouch
hw_openxrdevice.cpp:610   OffhandPos.Z = …same with offhandoffset[1]…
hw_openxrdevice.cpp:595-596  AttackPos.X/.Y = X()/Y() - (weaponoffset[0]/[2] * vr_vunits_per_meter)
```

The OpenVR path reaches the same place through the weapon matrix —
`gl_openvr.cpp:1287` translates the up axis by `… / pixelstretch` and `:1289` applies
`mat->scale(1, 1 / pixelstretch, 1)`, read into `AttackPos.Z` at `:2527`.

`pixelstretch` defaults to **1.2** (`src/gamedata/g_mapinfo.cpp:314`) and is script-readable as
`level.pixelstretch` (`wadsrc/static/zscript/doombase.zs:479`).

**So a hand's vertical offset from the eye reads 1/1.2 = 83% of its true size. ~17% short.**

### 2.2 The mod never reads it

`grep -rn pixelstretch` over the whole Wheel tree: **no hits.** `handPos` returns raw `AttackPos` /
`OffhandPos` (`zscript.zs:7501-7517`).

### 2.3 Where that bites — two places, both real-world-metre tests

**(a) The wrist-card pose window — `wr_q_watch.zs:34-50`.** This is the clear one.

```
Vector3 to = hp - eye;
double d = to.Length();
// within forearm's reach of the face: 15-50 cm (34 units a metre)
if (d < 5 || d > 17) return false;
…
if ((to / d) dot look < cos(35)) return false;
```

The comment does the conversion explicitly, at 34 units a metre and nothing else. The watch gesture
is *the forearm brought up across the chest, the wrist in front of the face* — a pose whose
hand-to-eye vector is dominated by its vertical and across-body components, which is the worst case
for this error. A hand genuinely 17 cm below the eyeline contributes 5.8 units of true Z and reads
4.8. `d` under-reads, so the `d < 5` floor rejects poses the comment says it should accept, and the
35° cone is skewed too: compressing Z tips `to` toward the horizontal, so the cone is effectively
wider below the eyeline than above it. **Verdict: wrong.** The README is already honest about it
(*"Not yet on a headset — gesture speeds and the wrist pose are first guesses"*), which is why this
is recommendation 5 and not recommendation 1.

**(b) The gesture speeds — `zscript.zs:8041-8086`.** Subtler, and the subtlety is the point.

```
Vector3 v = mHavePrev[h] ? hp - mPrevHand[h] : (0, 0, 0);
…
if (v.Length() > 12.0) continue;            // "a hand cannot move 12 units (35 cm) in one tic"
Vector3 toRing = mAnchor - hp;
double toward = v dot (toRing / dist);
```

`mAnchor = handP + ahead * clearForward` (`:5862`) where `ahead` is a full 3D unit vector built from
the hand's own yaw and pitch (`:5774`), so the anchor lives in the same compressed space as the hand
and the *position* error partly cancels. The *speed* does not. `v.Length()` and the `v · toRing`
dot product both mix a true-scale XY with a 0.83× Z, so **`wr_pull_speed` and `wr_throw_speed` mean
a different real hand speed depending on how vertical the gesture is.** A pull straight back toward
the chest is roughly horizontal and reads true; a pull down-and-back reads slow and needs a harder
snap. The header comment at `:8020-8027` states the units as "34 units a metre: 1.5 is about
1.5 m/s", which is true of a horizontal pull and false of a vertical one.

**The fix, both places (BUILDABLE / S).** One helper beside `handPos`, and route the two
magnitude tests through it:

```
// A hand delta in TRUE map-unit proportions. The engine divides the hand's vertical
// offset from the eye by level.pixelstretch (hw_openxrdevice.cpp:597, gl_openvr.cpp:1287-1289),
// so a raw Z delta is ~17% short and any 3D magnitude built from one is skewed.
static Vector3 trueDelta(Vector3 d) { return (d.x, d.y, d.z * level.pixelstretch); }
```

Then `d = trueDelta(hp - eye).Length()` in `watching()`, and `trueDelta(v)` /
`trueDelta(toRing).Unit()` in `tickPhysical`. Do this **before** the headset tuning pass, not after:
tuning first means the 17% gets baked into `wr_watch_dwell`, `wr_pull_speed` and `wr_throw_speed` as
a correction nobody will remember is there, and it will then be wrong on any map that sets a
non-default `pixelstretch`.

### 2.4 The pickup voxels are NOT exposed — the latest commit is clean

The question was specific, so the answer should be. `wr_VoxSize.Size(frame)` returns
`(max(xsiz, ysiz) * scale, zsiz * scale)` (`wr_q_vox.zs:64-67`) — pure KVX dimensions times the
VOXELDEF `Scale`, both already in map units. It is consumed at `zscript.zs:8676` as
`min(panelWNow() * 0.70 / vs.x, panelHNow() * 0.55 / vs.y)`, against card dimensions that are also
map units in the same world space. **No hand pose, no eye-relative Z, no metre conversion anywhere
in the path. Correct as written, and nothing to change.**

### 2.5 Also not exposed, and worth saying so

- **The ring layout itself.** `wr_forward` 34, `wr_radius` 5, `wr_rise` 2, `wr_tilt` 12 are placed in
  map units relative to `handPos` and drawn as world actors in the same space. The ring is a
  body-relative *surface*, but it is built in map units throughout and never compares a vertical
  distance against a metre figure.
- **Segment layout is by bearing.** `bearingForIndex` (`:4483`) and `fanSpread` (`:4499`) are
  angular. There is no up/down segment and no vertical reach threshold in the selection path.
- **The hit tests** (`AimBillboardT`, `TouchBillboard`) work in the same compressed map space as both
  the hand and the quads, so a card drawn at a position derived from `handPos` is hit at that
  position. Self-consistent.

---

## 3. Pointing at a segment

**The mod already does 6DOF, correctly, and there is nothing to recommend here except leaving it
alone.**

`AimBillboard(handPos(pmo, hand), handDir(pmo, hand), POINTER_RANGE)` (`zscript.zs:7780`) takes a
real controller **origin** and a real controller **direction**:

- origin: `AttackPos` / `OffhandPos` (`:7501-7506`)
- direction: `AttackAngle + 90` / `OffhandAngle + 90` with `AttackPitch` / `OffhandPitch`
  (`:7548-7563`), built as `(cos p · cos y, cos p · sin y, sin p)`

Both gated on `OverrideAttackPosDir` (`actor.zs:276`), with a look-direction fallback for the desk.
Both hands are read independently and either can own the beam (`:7760-7765`, `:7984` for the
two-hand compare), which is what 00_CAPABILITIES §1.2 says the engine supports.

**On `TRF_USEWEAPON` / `TRF_ISOFFHAND`.** Verified at `src/playsim/p_map.cpp:5185-5200`: with
`TRF_USEWEAPON` and `OverrideAttackPosDir` set, the trace substitutes `AttackPos`/`OffhandPos` as
the origin and `AttackDir`/`OffhandDir` as the direction. The Wheel **does not need it for card
selection**, because cards are its own `wr_BBRec` records, not world geometry — a `LineTrace` would
not see them at all (the shim's header says so). Where the Wheel does trace the world it passes
`TRF_ABSPOSITION` with the hand origin spelled out explicitly (`:7495-7496`,
`wr_q_shim.zs:808-809`), which reaches the same place and is more legible. **No change.**

One small note: `handDir` ignores roll. That is right for a ray — roll about the pointing axis does
not move it — and roll is read separately where it is wanted, for turning the ring (`handRollOf`,
`:7574-7581`, consumed as `wr_roll` at `:5951`).

**Where yaw-only *is* used, deliberately:** `stickPick` (`:7790`) maps a stick deflection to a fixed
bearing. That is an accessibility alternative for a player who would rather not aim — the comment at
`:7783-7789` says exactly that — and it overrides the ray only while the stick is deflected. Keep
it.

---

## 4. THE TWO-HAND TRAP

### 4.1 The ring itself cannot vanish

**Determined: the wheel is drawn from world actors, not an off-hand `Weapon` and not a psprite
overlay.** Every card, label, bar, sheet row, star, beam and burst is a `wr_BBQuad` / `wr_BBStrip` /
`wr_Beam` actor (`wr_q_shim.zs:73`, `:91`, `:763`) placed by `SetOrigin` each tic
(`wr_q_shim.zs:636`). The mod's only psprite touch in the whole tree is
`player.SetPsprite(layer, weap.GetReadyState())` in `equipInstantly` (`zscript.zs:9002-9003`).

The render skips are `psp->GetCaller() == player->OffhandWeapon`
(`src/rendering/hwrenderer/scene/hw_weapon.cpp:832`, `:917`) and they iterate `player->psprites`.
**Nothing in that loop can reach a world actor.** So stabilisation cannot blank the ring, and the
survey's "a plain overlay layer survives" escape hatch is not even needed here — the mod went
further than that and the ring is immune by construction.

### 4.2 And the Wheel's own button is not the grip

`KEYCONF`: `defaultbind LThumb +wr_wheel`. The confirm is `+attack` / `+oh_attack` (the triggers) or
`+use` (`zscript.zs:875-899`). **The Wheel never reads the grip.** That is precisely the defence
00_CAPABILITIES gotcha 10a recommends ("take grab state from `BT_OFFHANDATTACK` (the trigger) rather
than the grip"), and it means the Wheel cannot *cause* the latch.

### 4.3 But the exposure is real, and it is the shipped bug — it has just moved

**The Wheel delivers weapons into the off hand, and a latched off hand is dead.** Verified, every
link:

| What goes dark | Engine |
|---|---|
| fires nothing | `player.zs:393-396` (`FireWeapon`), `:426-429` (`FireWeaponAlt`) |
| checks no buttons | `src/playsim/p_pspr.cpp:755-757` |
| renders nothing (sprite **and** model passes) | `hw_weapon.cpp:832-835`, `:917-920` |
| cannot be cleared from script | `p_pspr.cpp:505-515` — `SetPsprite(PSP_OFFHANDWEAPON, …)` clears `WF_TWOHANDSTABILIZED` at `:508` and **immediately re-asserts it at `:514`** from the C++ global |
| latch condition | `gl_openvr.cpp:1716-1727` — off-hand **grip** rising edge with the controllers within `0.50f` raw tracking metres (`:1709-1714`); release clears it with no distance test (`:1724-1726`) |
| default | `vr_two_handed_weapons` **true**, `hw_vrmodes.cpp:119` |

So `equipInstantly`'s `SetPsprite(PSP_OFFHANDWEAPON, …)` at `zscript.zs:9003` lands on
`p_pspr.cpp:505-515` and the flag is re-stamped on the very call that was supposed to arm the hand.

**The realistic path to it is not "reaching across your body" — it is bracing a rifle.** The latch
needs a grip *press* while the hands are close. A player on BD22 VR-Reloaded with a two-handed
weapon braced is holding the off-hand grip already. He taps LThumb, the ring opens, he points the
off hand at a card, releases, and:

- `commit` (`:9133`) sends the gun to `mPokeHand` = 1 (`:9137-9139`)
- `commitAs` fires the full confirm — `feedback(Sound("wristrig/lock"), 0.75, 90)` (`:9222`), the
  card bursts (`:9241`), the flash ring fires (`:9250`), the card flips (`:9258`), the ring folds
- `equipInstantly` sets `player.OffhandWeapon = weap` (`:8997`) and the psprite (`:9003`)
- **and the hand is empty-looking and inert.** No sprite, no model, no fire, no reload.

Every one of the Wheel's four confirmations — sound, haptic, sparks, flip — says the pick landed. It
did land, in the sense that `OffhandWeapon` is set; it just cannot be seen or used until the player
releases the grip, and nothing anywhere tells him that.

**Three other call sites have the same hole:**

- `fileAway` (`:8093`) — the throw-back gesture equips the bare hand into `h` (`:8186`), including
  `h == 1`.
- `commitAs`'s hand-swap branch (`:9203-9207`) — moves the outgoing weapon into the other hand, which
  may be the latched one.
- `swapToPrevious` (`:969`) — the double-tap, `equipInstantly(pmo, w, hand)` at `:975`.

**Severity, stated honestly, as 00_CAPABILITIES does.** `weaponStabilised` is a single global
(`src/QzDoom/qzdoom_common.cpp:19`) and its only writes in this tree are `gl_openvr.cpp:1723` and
`:1726`. The OpenXR path never sets it, so on a Quest build this is **dormant today** — and the
Wheel ships for QuestZDoom. But the mod also runs on the desktop QZD rig the README names, and
"silently broken under SteamVR" is still broken. Design as though it fires.

### 4.4 The fix — BUILDABLE / S

`PlayerInfo.WeaponState` and `WF_TWOHANDSTABILIZED` are both script-visible
(`player.zs:393`, `constants.zs:1242`), so this is one bitwise test. Three touches:

1. **In `equipInstantly` (`:8973`), before the psprite set:**
   ```
   // The engine's two-hand brace blanks the off-hand weapon entirely -- it fires nothing
   // (player.zs:393), checks no buttons (p_pspr.cpp:755) and renders nothing
   // (hw_weapon.cpp:832, :917) -- and SetPsprite re-asserts the flag from the C++ global
   // on every call (p_pspr.cpp:514), so handing a gun to a braced off hand hands over a
   // gun that cannot be seen or used. Refuse, and let the caller say so.
   if (wantOffhand && (player.WeaponState & WF_TWOHANDSTABILIZED)) return false;
   ```
   (make it return a bool; it already early-returns on `bNoHandSwitch` at `:8987`, so the shape is
   established).
2. **In `commit` (`:9137-9143`), reroute rather than refuse where the gun allows it.** The hand
   resolution already has the `bNoHandSwitch` pin; add: if the resolved hand is 1 and the brace is
   latched, and the weapon is not off-hand-only, send it to the main hand instead. The player gets
   the gun he asked for, in the hand that can hold it.
3. **Where neither works, say so in the vocabulary the mod already has.** `commitAs` has a complete
   dry-pick dialect — a different sound (`wristrig/nope`), a weaker pulse, fewer sparks (`:9221-9222`,
   `:9241`). Use it: `feedback(Sound("wristrig/nope"), 0.35, 60)` and no equip. The comment at
   `:9216-9220` ("you find out now, from the wrist, rather than at the trigger with something walking
   at you") is the exact argument for doing it here too.

**Cheap addition worth having (BUILDABLE / S):** dim the cards while the off hand is the poke hand
and braced, the way `warnFrac` already fades the whole ring (`:7690`). The machinery exists.

**Do not try to clear the flag.** `p_pspr.cpp:512-515` re-asserts it from `weaponStabilised` on
every state set; this is stated in 00_CAPABILITIES and verified above.

---

## 5. Time while the wheel is open

### 5.1 The player freeze is already the right tool, and it is one bit

`CF_TOTALLYFROZEN` (`constants.zs:1192`, `d_player.h:122`) consumed in `CheckFrozen`
(`player.zs:1103-1134`). One `|=` on open, one `&= ~` on close, with `h.mFroze` so the Wheel never
thaws someone else's freeze (`wr_q_shim.zs:727-741`), and a belt-and-braces clear in
`WorldUnloaded` (`:849`) and `PlayerDied` (`zscript.zs:3218`). **Nothing to change.** It is cheaper
than `Level.SetFrozen` for this purpose, because it is per-player and leaves the world alone, which
is what the design wants.

Two details the mod gets right that are easy to get wrong:
- It reads the stick from `p.original_cmd` (`wr_q_shim.zs:750-751`), **because `cmd` has already been
  zeroed** by `CheckFrozen`. Correct.
- `BT_USE` survives the mask (`player.zs:1119`), and the `+use` grab is the one input the Wheel needs
  to still arrive. Correct, and almost certainly not an accident.

### 5.2 The verdict on switching to `SetFrozen` — **No, for the SlowMo path**

The question assumes the Wheel drives SlowMo's continuous clock "just to pause for a menu". **It does
not, and the premise should be retired.** The ring deliberately does not pause the world:

- The player is frozen; the world keeps running (§1.5, §5.1).
- The README states the design intent: *"The ring holds you still while it is up and, with RSQ
  SlowMo loaded, slows the world."* Slows. Not stops.
- SlowMo's clock eases in over `rsm_ramp_in` 0.15 s and out over `rsm_ramp_out` 0.40 s
  (`SlowMo/zscript/rsqsm/clock.zs:243-253` per 05_SLOWMO §1), with `RSQ_Accessibility` able to put a
  floor under both. A `freezetics` duty cycle cannot ease — it can only step 1 → 1/2 → 1/3
  (05_SLOWMO §3) — and `Level.SetFrozen` cannot ease at all. Replacing one with the other is a
  **design change** (the world lurches to a stop and lurches back), not a cost optimisation.

05_SLOWMO §7 is right that SlowMo's clock is strictly more expensive per tic than full speed, and
that `SetFrozen` is one int. But the comparison only matters if the two are interchangeable, and they
are not: a wheel you can be shot at through is the point. **Leave the SlowMo path as it is.**

### 5.3 The bug that is real — recommendation 2

Already verified by the owner and documented at 05_SLOWMO §8.1. Restated here only so this file is
self-contained:

`bulletTime` (`zscript.zs:4045-4056`) sends **absolute** `rsm_on` / `rsm_off`. `rsm_on` is
idempotent; `rsm_off` is not. A player holding his own slow motion, who opens and closes the ring,
loses it. The guard `if (on == mBtOn) return;` (`:4047`) cannot catch it, because `mBtOn` records only
what the Wheel did.

**The fix is in the Wheel, and `RSQSM_Service` already answers `active`** (`clock.zs:752-756`):

```
private bool mBtWasOn;   // slow motion was already running when we opened; not ours to end

private void bulletTime(bool on)
{
    if (on && !cvBool("wr_bullettime", true)) return;
    if (on == mBtOn) return;

    let sm = ServiceIterator.Find("RSQSM_Service");
    let svc = sm.Next();
    if (svc)
    {
        if (on)
        {
            // Ask before acting: rsm_off is absolute, so an off we send on close would
            // cancel a slow motion the player was holding before the ring ever opened.
            mBtWasOn = svc.GetInt("active") != 0;
            if (!mBtWasOn) EventHandler.SendNetworkEvent("rsm_on");
        }
        else
        {
            if (!mBtWasOn) EventHandler.SendNetworkEvent("rsm_off");
            mBtWasOn = false;
        }
    }
    else EventHandler.SendNetworkEvent("bt_activate");
    mBtOn = on;
}
```

**Cost S.** The long comment above the function (`:4025-4044`) already argues this exact class of bug
for BulletTimeX; this makes its own route honest about it. `mBtWasOn` wants clearing in
`OnRegister`/`WorldUnloaded` alongside `mBtOn`.

**Note:** the `GetInt("active")` request name is taken from 05_SLOWMO's reading of `clock.zs:752-756`
and was **not** verified against SlowMo's source here — the clone under audit contains only `Wheel/`.
Confirm the request name before writing the line.

### 5.4 The cost leak — recommendation 7

SlowMo's real-time exemption is a class-name prefix test, `rsq` / `ubqs` / `ssq`
(`clock.zs:293` per 05_SLOWMO §8.1). The Wheel's actors are `wr_BBQuad`, `wr_BBStrip`, `wr_Beam`,
`wr_Buzz` (`wr_q_shim.zs:73`, `:91`, `:763`, `:669`) — **no match**, so every billboard quad in the
ring is gathered and walked by SlowMo's per-tic `Process` loop at the exact moment the player is also
rendering a 3D ring in slow motion.

05_SLOWMO §8.1 checked that it breaks nothing and that is confirmed from this side: the quads are
positioned only by `SetOrigin(at, true)` (`wr_q_shim.zs:636`), never by `Vel`, and their state is
`WRBQ A -1` (`:89`) so `tics` is −1 and SlowMo's `a.tics > 0` guard skips them. Their `+NOTIMEFREEZE`
(`:83`) is for a level freeze and irrelevant to SlowMo.

**The Wheel-side fix (BUILDABLE / S):** rename the drawing primitives `rsqwr_*`. The blast radius is
genuinely small — three `Model` headers in `MODELDEF.txt:4`, `:14`, `:25`, the class declarations in
`wr_q_shim.zs`, and about a dozen `Actor.Spawn("…")` strings. `wr_BBStrip` already inherits from
`wr_BBQuad`, and nothing outside the Wheel references any of them (checked: `MODELDEF.txt` is the
only non-ZScript file that names them).

**Say the dependency out loud, though:** this fix works by matching a prefix test that lives in
another mod's file and is not a published contract. It works today. The durable answer — a real
real-time marker (an inventory token, or a flag the exempt actor sets) published by SlowMo — is the
owner's call, as 05_SLOWMO §8.1 already says. Recommend the rename as the cheap move and name the
fragility.

`wr_Buzz` is the one to watch: it emits `A_QuakeEx` from a `Tick()` override
(`wr_q_shim.zs:676-688`) and its `TNT1 A 40` state would be stretched by the clock, lengthening a
rumble. Cosmetic, and the rename fixes it with the rest.

### 5.5 Where `SetFrozen` **does** belong — recommendation 8

**The no-SlowMo fallback.** With no `RSQSM_Service`, `bulletTime` sends `bt_activate`
(`zscript.zs:4054`) — BulletTimeX's **toggle**. The mod's own comment (`:4025-4044`) already
documents that a flip cancels an already-running bullet time on open, and that the RSQ route exists
to avoid it. So the ring's fallback is the broken case, and it depends on a third-party mod nobody in
the series controls.

Give the ring a correct stop of its own, for when neither SlowMo nor BulletTimeX is loaded. 10_TIME
settled the mechanism and it is three script-visible pieces:

- `Level.SetFrozen(bool)` — `wadsrc/static/zscript/doombase.zs:511`
- `player.timefreezer` — `player.zs:3079`, an int bitmask, with the exemption read at
  `src/playsim/actorinlines.h:180-193`: a player whose `timefreezer` is non-zero is **not** frozen by
  the global `state & 1`
- `+NOTIMEFREEZE` per actor — `src/playsim/actor.h:315`, and `wr_BBQuad` **already has it**
  (`wr_q_shim.zs:83`), as does `wr_Buzz` (`:674`)

So: `Level.SetFrozen(true)` plus `players[consoleplayer].timefreezer = 1` on open, both reverted on
close, and the ring keeps drawing, keeps hit-testing and keeps buzzing while the world stops. Two
engine facts verified for this:

- **`WorldTick` still runs.** `P_Ticker` calls `Level->localEventManager->WorldTick()` unconditionally
  at `src/p_tick.cpp:172`, before `RunThinkers`. Only `P_UpdateSpecials` is gated on `isFrozen()`
  (`:175-178`). So `wr_BB.WorldTick` and `wr_Rig.WorldTick` are untouched.
- **The haptics survive.** `DEarthquake::Tick` (`src/playsim/mapthinkers/a_quake.cpp:115`) has no
  freeze check, and `RunThinkers` is not gated, so `wr_Buzz`'s `A_QuakeEx` quakes keep reaching
  `r_utility.cpp:1085-1091`. The detent clicks and the commit thump do not go silent in a frozen
  world. (This is worth knowing before anyone reaches for `SetFrozen` elsewhere in the series.)

**Cost S**, and it also lets `wr_bullettime` mean something without a dependency. Keep the ordering
honest: with SlowMo present, SlowMo wins; with BulletTimeX only, keep today's behaviour if the owner
wants it; with neither, the ring's own freeze.

---

## 6. Drawing it

**Determined: world actors. The choice is right, it is documented, and the mod sidestepped a trap it
had every reason to fall into.**

### 6.1 Why it is right

A ring you *point at* has to exist in the world, because the selection is a ray-plane intersection
against the cards' own geometry (`wr_q_shim.zs:258-281`). A psprite cannot be that, and
99_OVERLOOKED §1 says why: in VR **every non-model psprite layer is transformed by a hand matrix**
with the hand chosen by the layer's caller (`hw_weapon.cpp:252-258`), and `psp.x` / `psp.y` become
**rotations, not translations** (`hw_weapon.cpp:108`). A ring built on psprites would be glued to a
hand and unaimable.

**The `PSP_TARGETCENTER` band is specifically the wrong tool here, and the mod did not use it.** It
is tempting — it survives `A_ClearOverlays(safety)` (`p_pspr.cpp:1273-1274`) and weapon switches,
and layer ids are free integers with a million-wide gap (`constants.zs:798-801`). But
`PrepareTargeterSprites` pushes into the same `hudsprites` array (`hw_weapon.cpp:1072`) and gets the
same hand matrix, and a layer created at or above it defaults its `Caller` to the player's
`PowerTargeter` and gets culled without one (`p_pspr.cpp:310-318`). So the "survives a weapon switch
mid-open" property that makes it attractive comes attached to a hand-locked frame and a null-caller
trap. **World actors survive a weapon switch for free and have neither problem.** The Wheel is
already on the better side of this.

### 6.2 And the canvas question is already answered, in writing

`wr_q_shim.zs:14-21` and the README both state it: QZD 17.3's `Draw2D` skips its viewport and its 2D
projection whenever a VR mode is on, so a canvas drawn in a headset comes out black — *measured on
the 17.3 rig: canvas black, camera texture fine*. `wr_canvas` is therefore a no-op and the faces are
composed from billboards instead. The data sheet's own comment (`zscript.zs:1241-1254`) adds the
other reason: a canvas paints in 2D pixel coordinates whose Y axis runs the opposite way, and *"the
last panel this project tried was a canvas and drew upside down three separate times."* **Cited and
moved past.**

### 6.3 The one cost of the choice, and it is already paid for

World actors are depth-tested against world geometry, so a ring planted through a wall renders
behind it. That is the entire reason for the anchor's wall-clamp work at `:5787-5925`: a forward
`LineTracer` from the hand's **own** sector (`:5806`, with a good comment on why not the player's),
`ignoreAllActors` so a monster is not mistaken for a wall (`:5810-5826`), `TRACE_PortalRestrict` so a
portal does not report a wall in another room's frame (`:5828-5839`), plus two side traces from the
anchor (`:5896-5920`) because the ring is a wide grid and a forward ray says nothing about a corner.
Frozen once per open, not re-checked, with the reasoning given. This is the right shape and it should
not be made live.

### 6.4 One ceiling worth knowing before anyone grows a strip

Text is one actor per line, a strip model whose cells are skinned per surface
(`wr_q_shim.zs:518`, `:528`, `CMDL_USESURFACESKIN`). **Surfaces are capped at 32:**
`MD3_MAX_SURFACES` is `32` (`src/common/models/model.h:30`), and `models.cpp:505-525` resizes the
per-actor surface-skin array to exactly that and loops to it. `strip24.obj` fits with room to spare;
a `strip32` would be exactly at the ceiling and a `strip48` would **silently lose its last 16
cells**. Worth a comment in `tools/gen_assets.py` next to the strip generator. (Note for context:
UZDXREMA raised a related constant in commit `c3740759b7`; stock qzd17.3 is 32, so the Wheel's 24 is
safe on the shipping target.)

---

## 7. Detent haptics

**The mod already has them, they are shaped by the gun's weight, and the `A_QuakeEx` safety arguments
are correct. There is one real gap.**

### 7.1 What exists

One route, `wr_VR.VRHaptic(hand, intensity, ms)` (`wr_q_shim.zs:707-721`) spawning a `wr_Buzz` that
fires `A_QuakeEx(left ? a : 0, left ? 0 : a, 0, 2, 0, 8, "", QF_WAVE, 17.5, 17.5, 1)` per tic with a
linear fade (`:676-688`). Left hand in intensity X, right in Y, handedness resolved from
`vr_control_scheme >= 10` (`:697-701`). This is 99_OVERLOOKED §1a's route, and the two mandatory
safety points are both honoured:

- **`damrad = 0`** — the 5th argument is `0`, so the block at `src/playsim/mapthinkers/a_quake.cpp:191`
  is skipped entirely and neither the random thrust (`:205-209`) nor the damage (`:193-203`) can fire.
  Verified against the signature at `wadsrc/static/zscript/actors/actor.zs:1284`.
- **No `QF_GROUNDONLY`** — flags are `QF_WAVE` only, so the quake is not dropped when the player is
  off the floor (`a_quake.cpp:354`).

The event vocabulary is complete and well judged:

| Event | Call | Where |
|---|---|---|
| crossing onto a card | `wristrig/move`, amp `0.14 + 0.40·k·wf`, `18 + 50·k·wf` ms, where `k` is the weapon's slot normalised 1..9 | `zscript.zs:8947-8961` |
| commit | `wristrig/lock`, **0.75 / 90 ms** — "twice the hover pulse and three times as long" | `:9222` |
| dry commit | `wristrig/nope`, 0.35 / 60 ms | `:9221` |
| open / close | 0.40 / 55, 0.18 / 35 | `:1132`, `:1193` |
| fan opened | `wristrig/tick`, 0.30 / 45 | `:8909` |
| idle-close warning, once | 0.3 / 40 | `:7693` |
| gun filed away | 0.6 / 70 | `:8188` |
| wrist card appears | 0.18 / 25 | `wr_q_watch.zs:91` |
| compare opens | 0.25 / 30 | `:8010` |

Haptics go to `mPokeHand` — "the hand that did the thing" — with the reasoning at
`zscript.zs:8158-8162`. Correct: buzzing the hand wearing the cards when the other one selected
would arrive at the wrong wrist.

### 7.2 Which route for which event — the answer is "keep `A_QuakeEx`"

`LineAttack(angle, 0.1, pitch, 1, "None", "NullPuff", LAF_NOINTERACT|LAF_ISOFFHAND)` gives a
**150 ms / 0.8** pulse on the chosen hand with correct handedness and no world effect
(`src/playsim/p_map.cpp:4764-4769`, verified; the channel flip is at `:4767` and nothing gates it on
`LAF_NOINTERACT`). But duration and intensity are **literals in C++**. The Wheel's whole feel design
is that a pistol and a BFG tick differently — `amp` and `ms` both scale with the gun's slot — and
`LineAttack` cannot express that. **`A_QuakeEx` is the right choice and the mod chose it for the
right reason.**

### 7.3 The gap — recommendation 6, BUILDABLE / S

`A_QuakeEx`'s route to the controllers is gated on `vr_quake_haptic_level > 0`
(`src/rendering/r_utility.cpp:1085`). It defaults to **0.8** (`hw_vrmodes.cpp:127`), so it works out
of the box — but it is an archived user cvar, and **any player who has set it to 0 loses the Wheel's
entire tactile layer with no indication.** Every one of the nine events above goes silent. The mod
checks `wr_haptics` and relies on the engine still checking `vr_enable_haptics` (`:8176-8180`), but
it never checks the cvar its own route actually depends on.

`LineAttack`'s route is gated only on `vr_enable_haptics` inside `OpenVRHaptics::Vibrate`
(`gl_openvr.cpp:532`) — **not** on `vr_quake_haptic_level`. So:

```
static void VRHaptic(int hand, double intensity, double durationMs)
{
    let mo = players[consoleplayer].mo;
    if (!mo || intensity <= 0) return;

    // The quake route reaches the controllers only through r_utility.cpp:1085, which is
    // gated on vr_quake_haptic_level -- a user cvar. Zeroed, every tick, thump and warning
    // in this mod goes silent. P_LineAttack's own buzz (p_map.cpp:4764-4769) is gated only
    // on vr_enable_haptics, so it still lands: unshaped (a flat 150 ms / 0.8 literal), but
    // present, which beats nothing.
    let qc = CVar.FindCVar('vr_quake_haptic_level');
    if (!qc || qc.GetFloat() <= 0.0)
    {
        bool offhand = (hand == 1) != LeftHanded();   // LAF_ISOFFHAND flips the channel
        mo.LineAttack(mo.angle, 0.1, mo.pitch, 1, "None", "NullPuff",
                      LAF_NOINTERACT | (offhand ? LAF_ISOFFHAND : 0));
        return;
    }
    … existing wr_Buzz path …
}
```

Work out the handedness term against `vr_control_scheme` on the rig before shipping it — the engine
does `rightHanded = vr_control_scheme < 10; rightHanded = (flags & LAF_ISOFFHAND) ? 1 - rightHanded
: rightHanded` (`p_map.cpp:4765-4767`), so the mapping from "Wheel hand 0/1" to `LAF_ISOFFHAND` is
one XOR and easy to get backwards. Also note `p_map.cpp:4771-4774`: when `weaponStabilised` is set
the engine buzzes **both** hands, so a double pulse under a brace is the engine, not a bug.

Two smaller notes:

- A sustained pulse costs one spawned actor per event (`wr_Buzz`, destroyed after
  `clamp(ms·35/1000, 1, 16)` tics). At nine event types and a card crossing per few tics while
  sweeping the ring, that is a handful of short-lived actors. Acceptable, but if §8's profiling shows
  spawn churn, a single persistent buzzer with a queue is the cheaper shape — RSQ ShieldSaw's
  `SSQ_Buzzer`, which this is modelled on, is presumably already that.
- `QuakePower` is random in `[-intensity, +intensity]` (`r_utility.cpp:911-921`), so the detent is a
  *thud*, not a *click*. That is a known limitation of the route, not a mod bug, and the owner should
  hear it on hardware before anything is changed — a weight-shaped thud may be exactly right for a
  wheel of guns.

---

## 8. Cost

### 8.1 What to measure — recommendation 10, EXISTS / S

Both verified present in this engine:

- **`profilethinkers -t 20`** — `src/playsim/dthinker.cpp:833`. Per-class total tick time,
  descending, top 20. Run it in a Brutal Doom fight with the ring **closed**, then with the ring
  **held open**, then open with SlowMo running. **The numbers that decide it are whether `wr_BB` and
  `wr_BBQuad` appear in the top five, and by how much `RSQSM_Clock` moves between runs 2 and 3.**
- **`stat think`** — `dthinker.cpp:1054`. Total think time per tic as one figure, plus the thinker
  count. Watch it across the open/close transition. The budget is 28.6 ms at 35 tics/s.
- **`profilethinkers -#`** for call counts.

And 10_TIME §6's closing note governs: in VR the figure that actually decides it is dropped frames at
the headset refresh, not playsim ms. But the Wheel runs its heaviest work at the exact moment the
renderer is also drawing 100+ model actors, so both matter and they are not independent.

**Nothing in §8.2–§8.4 should be spent on before this is run.** Everything below is structurally
true; whether any of it is *felt* is a measurement.

### 8.2 Two linear scans in the per-tic path — recommendation 3, BUILDABLE / S

**(a) `grp()` is a linear scan over `mGroups`, and it is called from inside `resolve()`.**

```
wr_q_shim.zs:131-136   private wr_BBGroup grp(int gid)
                       { for (int i = 0; i < mGroups.Size(); i++) if (mGroups[i].id == gid) return mGroups[i]; }
wr_q_shim.zs:375       resolve():  let g = grp(r.group);
wr_q_shim.zs:352       groupScale(): let g = grp(gid);
wr_q_shim.zs:344       hittable():   if (r.group && groupScale(r.group) <= 0) return false;
wr_q_shim.zs:608       place():      double s = r.group ? groupScale(r.group) : 1.0;
```

So one `place()` call on one record walks `mGroups` **three times** (`hittable` is not in `place`,
but `groupScale` at `:608` and `resolve`'s own `grp` + `groupScale` are), and `mGroups` holds one
entry per card plus the sheet, the stars, the fan and the compare group — call it 12–20. The record
map `mRecs` is already a `Map<int, wr_BBRec>` (`:117`); `mGroups` should be a `Map<int, wr_BBGroup>`
beside it, with `mList`-style ordered iteration only where the animate-all loops need it
(`zscript.zs:1126`, `:1196`). Mechanical, no behaviour change.

**(b) `TransFor()`'s cache is a linear scan, and `Tint` calls it per part per tic.**

```
wr_q_shim.zs:441   for (int i = 0; i < mTransKey.Size(); i++) if (mTransKey[i] == key) return mTransId[i];
wr_q_shim.zs:463   void Tint(Actor a, color c, double alpha, bool recolour)
                   { a.A_SetRenderStyle(alpha, STYLE_Translucent);
                     if (recolour) a.Translation = TransFor(wr_Look.Col(c)); }
wr_q_shim.zs:623   place():  Tint(a, pc, r.alpha, recolour);
```

The distinct-colour count is the palette's roles plus the tier colours plus the nine slot colours —
20–40 entries. So the inner comparison count per tic is roughly
*records × parts × distinct colours*, which at 150 records is several thousand integer compares for
nothing. `Map<int, int>` keyed on the packed `(r<<16)|(g<<8)|b`, same shape as `mRecs`.

**Free second half of the same fix:** `Tint` re-applies the render style *and* the translation on
every part on every tic, whether or not anything changed. The record already has a `lookDirty` flag
(`:62`, set by `UpdateBillboard`/`SetBillboardText`/`SetBillboardGradient`). Alpha and group scale
genuinely change per tic during the grow-in and the fade, so `A_SetRenderStyle` has to stay; the
`Translation` assignment does not. Cache the resolved `TranslationID` on the record and write it only
when the colour changed.

### 8.3 The resolve is done nine times a tic — recommendation 4, BUILDABLE / S–M

`resolve()` (`wr_q_shim.zs:371-401`) does: a group lookup, a group-scale evaluation, up to two
`atan2` for a camera-facing record, four `cos`/`sin`, a cross product, and three vector multiplies —
**per record**. And it is re-run from scratch in every one of these O(`mList`) sweeps, each tic, while
the ring is open:

| # | Sweep | Where |
|---|---|---|
| 1 | expire pass | `wr_q_shim.zs:590-594` |
| 2 | build + place pass (this one *must* resolve) | `:596-601` |
| 3–4 | `handOnRing(mPokeHand)` → `AimBillboard` + `TouchBillboard` | `zscript.zs:8136-8145` via `:7761` |
| 5–6 | `handOnRing(other)`, when the poke hand is off-ring | `:7761` |
| 7 | the selection ray | `:7780` |
| 8–9 | `tickPhysical`'s `TouchBillboard`, once per hand | `:8047` |
| 10 | `tickTwoHand`'s other-hand `AimBillboard` | `:7984` |
| 11 | `wr_VRState`'s idle-beam `AimBillboardT` | `wr_q_shim.zs:800` |

Not all fire every tic, but the common case with two hands near the ring is **six to nine full
sweeps**, each re-deriving a basis that sweep #2 has already computed this tic.

**The fix:** cache it. `place()` already resolves every record every tic, and `resolve` already
stashes the basis in the handler's `mR`/`mU`/`mN` scratch fields (`:393`). Move that onto the record
— `cC`, `cHW`, `cHH`, `cR`, `cU`, `cN`, `cTic` — written in sweep #2, and have `AimBillboardT` /
`TouchBillboard` / `hittable` read the cache when `cTic == level.maptime`. That removes the trig and
the group scans from four to seven sweeps outright.

**One honest caveat, which is why this is S–M and not S:** `resolve(r, eye)` takes the viewpoint and
a `BBF_CAMERA`/`BBF_CAMERAYAW` record turns to face it (`:385-390`). The hit tests pass the **hand
position**, not the eye (`TouchBillboard(point)` resolves against `point`; `AimBillboardT(start, …)`
resolves against `start`). So a camera-facing billboard currently presents a slightly different plane
to the ray than it does to the renderer. Caching the eye-resolved basis would make the hit test agree
with what is drawn — which is arguably **more** correct, since a player aims at what he sees — but it
*is* a behaviour change on camera-facing records and wants a rig pass. Note also that the cards
themselves are mostly not camera-facing (`wr_facing` gates a separate per-card yaw at
`zscript.zs:5968`), so the affected set is the sheet, the compare card and the wrist card.

### 8.4 Cvar string lookups — real but minor, and mostly already handled

`cv(name, fallback)` and `cvBool` do a fresh `CVar.FindCVar(name)` on every call
(`zscript.zs:3299-3313`). There are **155** `cv("` call sites and 12 `cvBool("` in `zscript.zs`, plus
31 `FindCVar` across the tree.

**Credit where it is due: `layout()` already hoists.** Twenty reads are pulled above the per-card
loop (`:5945-5967`) and only four remain inside it — `wr_pop`, `wr_facing`, `wr_pulse`,
`wr_model_spin`. So the live per-tic count is roughly 40–60 lookups, not 155. That is modest.

If §8.1 says it matters, the fix is a resolved-cvar cache: `CVar` objects are stable for the session,
so fetch each one once in `OnRegister` into a named field and read `.GetFloat()` per tic. The four
in-loop reads are the cheapest to hoist and the only ones worth doing unconditionally.

### 8.5 The always-on cost, which is the one that runs in a fight

Everything above only runs while the ring is up. **`wr_StatTracker` runs every tic regardless**
(`wr_stattracker.zs:264-283`), and its per-call path is not free:

```
wr_stattracker.zs:264   WorldTick():  trackFire(pawn, ReadyWeapon, false); trackFire(pawn, OffhandWeapon, true);
wr_stattracker.zs:165   StatsFor():   pawn.FindInventory("wr_StatLedger")          // string -> class, then an inventory chain walk
wr_stattracker.zs:173-176             for (i = mStats.Size()-1; i >= 0; --i) if (wpn == null) mStats.Delete(i);   // full prune
wr_stattracker.zs:178-181             for (i = 0; i < mStats.Size(); ++i) if (mStats[i].wpn == w) return …        // linear scan
```

Two calls a tic, each a string-keyed `FindInventory` plus a full prune pass plus a linear scan.
`mStats` is one entry per weapon instance the player has fired — small on a vanilla set, not small on
a 60-gun VR pack. **Cheap fix (BUILDABLE / S):** cache the `wr_StatLedger` reference on the handler
and prune on a 35-tic timer rather than on every lookup. The event handlers are already careful —
`WorldThingDamaged` bails on `e.DamageSource.player != players[consoleplayer]` in three lines
(`:456-462`) before doing anything — so this is the only always-on cost worth naming.

### 8.6 What is *not* structurally expensive, so nobody hunts for it

- **No `ThinkerIterator` and no `AllActorClasses` anywhere in the tree.** `gatherWeapons` walks
  `WeaponSlots` (`:3355`), not the class list, with a comment saying why.
- **`ServiceIterator.Find` is never in a per-tic path.** `bulletTime` is open/close only (`:4052`);
  `RSQTR_RigService` and `RSQ_MeleeUpgradeService` are read in `seriesRows` (`:1533`, `:1539`), which
  runs from the sheet build, gated on a weapon change or a `SHEET_TICK` interval; `RS_TierColorService`
  is in `rsTierLookup` (`:4843`), per card at build time; `RS_WeaponArchetypeService` in
  `wr_compat_modelswapper.zs:73`, on the models page.
- **Text is one actor per line, not per character** (strip models, `wr_q_shim.zs:518`), and the
  per-surface skinning only runs on `lookDirty` (`:598`), not per tic.
- **The sheet rebuilds on change, not per tic.** `refreshSheet` returns immediately in the common
  case (`:1255-1262`); `tickTwoHand` gates on `SHEET_TICK` (`:8001`).
- **The compat files are pure readers.** No `override void Tick` or `WorldTick` in any
  `wr_compat_*.zs` (checked, zero), and only `wr_compat_modelswapper.zs` touches hands at all — via a
  service and a bridge cvar. **No recommendation in this file touches any of them.**

---

## 9. Series fit

The owner's rule — each mod works alone, they talk through services, never hard links — is followed,
and followed carefully.

### 9.1 What it consumes today, and it degrades cleanly in every case

| Service | Mod | Where | What it adds | Absent → |
|---|---|---|---|---|
| `RSQSM_Service` | RSQ SlowMo | `zscript.zs:4052` | slows the world while the ring is up | falls to `bt_activate`, then nothing |
| `RSQTR_RigService` | RSQ TacticalRig | `:1533-1537` | a HOLSTER row on the sheet | row omitted |
| `RSQ_MeleeUpgradeService` | RSQ Gun Bonsai VR | `:1539`, `wr_q_watch.zs:132` | a BONSAI level row on the sheet and the wrist card | rows omitted |
| `RS_WeaponArchetypeService` | ModelSwapper | `wr_compat_modelswapper.zs:73` | names the model a gun wears; the Models page | page count drops to 1 (`:784-786`) |
| `RS_TierColorService` | RS_Main | `:4843` | tier colours on cards | falls back to slot colours (`:4779-4782`) |

Every one is `ServiceIterator.Find` with the result checked, **not** `Service.Find` — and the comment
at `:4838-4842` gives the reason: `Service.Find` takes a `class<Service>`, which resolves at compile
time and would fail to build without RS_Main loaded. That is the correct idiom and it is used
consistently. The README's "each by name, each only when loaded" is accurate.

It also **publishes** one, `wr_RigService` (`:158-222`): `IsOpen`, `OpenHand`, `HasAnchor`,
`AnchorX/Y/Z`, `AnchorYaw`, `RingClearance`, `HoveredWeapon`. `HasAnchor` is separate from `IsOpen`
for a reason the comment spells out (`:176-187`): the ring can report open before `mAnchor` is
resolved, and a caller building on the placeholder bakes itself at `(0,0,0)` forever. That is good
service design.

### 9.2 What it should consume and does not — recommendation 9, BUILDABLE / S

**`RSQLS_SightService`** (`RSQ_LaserSight`, `sight.zs:788-798` per 06_RIG_AND_LASER §2.4). The Wheel
draws its own beam from the hand (`wr_VRState.drawBeam`, `wr_q_shim.zs:811-837`) and turns it on for
the pointing hand plus a dimmer one on the other (`zscript.zs:7965`). RSQ_LaserSight draws an
off-hand laser — beam, dot, glow, dynamic light, spread ring and range readout — from the same hand.
**With both loaded and the ring open, two beams and two dots come out of one controller**, and the
Wheel's is the one that means something at that moment.

The service already reports a live per-hand laser (`liveMask`, `sight.zs:350`). Read it on open and,
when a hand's laser is live, either suppress the Wheel's dimmer second beam on that hand or accept a
single-beam mode. **S**, and it is a visible-in-the-headset fix rather than a theoretical one.

**The reciprocal, which is not the Wheel's to write:** `RSQ_DistanceGrab` reads hand poses and
hand-to-hand distance for its grip gestures — a hand swap when both controllers are within
`rsqdg_swap_m` with both wrists rolled (`near.zs:110-116`), and a walk-up grab at `rsqdg_near_m`
0.30 m (`near.zs:214-230`), per 04_DISTANCEGRAB §1. **Two hands reaching into the Wheel's ring is
exactly both of those poses**, and the Wheel's two-hand compare mode (`:7977`) actively encourages
it. The Wheel's side needs nothing: `wr_RigService.IsOpen` is already the door, and it is already
published. **Name it to whoever owns DistanceGrab** rather than building anything here; a mod that
reads hands should stand down while another mod has claimed them.

### 9.3 What it should *not* consume

- **`RS_ScoreService`** (`RSQ_Score`, per 04_DISTANCEGRAB §1). The Wheel has its own tracker,
  `wr_StatTracker` (`wr_stattracker.zs`), which is what feeds the sheet's SHOTS / HITS / KILLS / ACC
  rows (`statRows`, `:2321`) and the wrist card. That tracker exists because the sheet needs
  per-weapon attribution — which hand fired, through a shared ammo pool, with hitscan damage landing
  inside the same `P_PlayerThink` (`wr_stattracker.zs:276-291`, a genuinely hard problem solved
  carefully). A score service answers a different question and would not replace it. **Leave it.**
- **A Grapple or ShieldSaw service.** No service name for either appears anywhere in this survey's
  files, and the Wheel has nothing to ask them. If they publish one later, the interesting request is
  the same as DistanceGrab's — "is a hand busy" — and `wr_RigService.IsOpen` is the Wheel's half of
  that conversation already.

### 9.4 One series-level observation

Three mods now want to know "is a hand claimed right now": the Wheel (which claims both while open),
LaserSight (which reports per-hand), and DistanceGrab (which consumes LaserSight's report). That is a
shared question being answered three private ways. **Not a recommendation for this mod** — it is an
architecture note for the owner, and the right shape is probably one small `RSQ_HandsService` that
every hand-reading mod publishes into and reads, rather than N² pairwise service lookups. Noted here
because the Wheel is the mod that claims the most and is therefore the one that would benefit most.

---

## 10. Label and cost table

| Item | Label | Cost | Section |
|---|---|---|---|
| Off-hand pick while `WF_TWOHANDSTABILIZED` is latched | **BUILDABLE** (bug fix) | **S** | §4.4 |
| `rsm_off` on close cancelling a held slow motion | **BUILDABLE** (bug fix) | **S** | §5.3 |
| `mGroups` → `Map`; `TransFor` cache → `Map`; skip the per-tic `Translation` write | **BUILDABLE** | **S** | §8.2 |
| Cache the resolved basis per record per tic | **BUILDABLE** | **S–M** | §8.3 |
| `pixelstretch` on the wrist-pose window and the gesture speeds | **BUILDABLE** (bug fix) | **S** | §2.3 |
| `LineAttack` haptic fallback when `vr_quake_haptic_level` is 0 | **BUILDABLE** | **S** | §7.3 |
| Rename `wr_*` actors out of SlowMo's sweep | **BUILDABLE** | **S** | §5.4 |
| `Level.SetFrozen` + `timefreezer` as the no-SlowMo fallback | **BUILDABLE** | **S** | §5.5 |
| Consume `RSQLS_SightService` (double beam) | **BUILDABLE** | **S** | §9.2 |
| `profilethinkers -t 20` / `stat think` with the ring open | **EXISTS** | **S** | §8.1 |
| Cache the `wr_StatLedger` reference; prune on a timer | **BUILDABLE** | **S** | §8.5 |
| Dim the cards while the off hand is braced | **BUILDABLE** | **S** | §4.4 |
| Resolved-cvar cache (only if §8.1 says so) | **BUILDABLE** | **S–M** | §8.4 |
| Hoist the four in-loop `cv()` reads in `layout()` | **BUILDABLE** | **S** | §8.4 |
| `strip > 32` cells silently truncates — comment it | **EXISTS** (ceiling) | **S** | §6.4 |
| Replace SlowMo's clock with `SetFrozen` for the ring | **do not** | — | §5.2 |
| Move card selection to `LineTrace` with `TRF_USEWEAPON` | **do not** | — | §3 |
| Move the ring to psprites or the `PSP_TARGETCENTER` band | **do not** | — | §6.1 |
| Replace `A_QuakeEx` with `LineAttack` for the detents | **do not** | — | §7.2 |
| A real-time *marker* in SlowMo instead of a name prefix | **owner's call** | M | §5.4 |
| A shared `RSQ_HandsService` for the series | **owner's call** | M | §9.4 |
| Reciprocal stand-down in `RSQ_DistanceGrab` | not this mod | S | §9.2 |

---

## 11. What I could not determine

- **Whether `RSQSM_Service` answers `GetInt("active")` under that exact request name.** The recipe in
  §5.3 depends on it. 05_SLOWMO §8.1 reads it from `clock.zs:752-756`; the clone audited here contains
  only `Wheel/`, so it is cited at second hand. **Confirm the request name before writing the line.**
- **Whether the shipping binary is `USE_OPENVR`, `USE_OPENXR` or neither** —
  00_CAPABILITIES §0's first open question, and it decides the *severity* of §4 (live on OpenVR,
  dormant on OpenXR) and nothing else in this file. The five-minute check is still the one §0
  prescribes: print `AttackPos` and `OffhandPos` from `DoEffect` for a few tics and confirm they move
  and differ.
- **Whether the §8 costs are felt.** Everything in §8.2–§8.5 is structurally true and none of it is
  measured. §8.1 is the gate.
- **Whether the wrist-pose window is wrong by enough to matter after the `pixelstretch` fix.** The
  error is ~17% on the vertical term; whether that is the difference between the gesture working and
  not working is a headset question. The README already says the pose is a first guess, so the fix
  and the tuning want to happen in that order and in one sitting.
- **The `LAF_ISOFFHAND` handedness mapping for §7.3.** The engine's own flip
  (`p_map.cpp:4765-4767`) composes with `vr_control_scheme`, and the Wheel's `wr_VR.LeftHanded()`
  already does its own composition for the quake route (`wr_q_shim.zs:714-716`). Getting both right at
  once wants one rig check with a known hand, not reasoning.
