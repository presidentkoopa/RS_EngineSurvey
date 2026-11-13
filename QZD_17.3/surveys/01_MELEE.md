# 01 — MELEE / FIGHTING / BLADES

Engine surveyed: tag `qzd17.3` of `github.com/emawind84/gzdoom` (QuestZDoom PC edition).
All citations are repo-relative at that tag. Base engine is GZDoom **4.14.3** (`src/version.h:44`).

---

## 0. GROUND RULES, FRAMES AND UNITS — read this before section 4

### 0.1 Two hands, symmetric, both readable

Independently confirmed here, and consistent with the revised `_BASELINE_FACTS.md` §1a.

| ZScript field | Declared | Native | Thunk |
|---|---|---|---|
| `Actor.AttackPos` (vector3, readonly) | `wadsrc/static/zscript/actors/actor.zs:268` | `src/playsim/actor.h:1655` | `src/scripting/vmthunks_actors.cpp:2187` |
| `Actor.AttackPitch` (double, readonly) | `actor.zs:269` | `actor.h:1656` | `vmthunks_actors.cpp:2188` |
| `Actor.AttackRoll` (double, readonly) | `actor.zs:270` | `actor.h:1658` | `vmthunks_actors.cpp:2189` |
| `Actor.AttackAngle` (double, readonly) | `actor.zs:271` | `actor.h:1657` | `vmthunks_actors.cpp:2190` |
| `Actor.OverrideAttackPosDir` (bool, readonly) | `actor.zs:276` | `actor.h:1651` | `vmthunks_actors.cpp:2186` |
| `Actor.AttackDir(actor, angle, pitch)` | `actor.zs:858` | `actor.h:1660` (fn ptr) | `src/playsim/p_actionfunctions.cpp:5694` |

`AttackPos` is written **per rendered frame from the dominant controller's tracked pose**, by exactly the
same code that writes `OffhandPos`, three lines above it:

- OpenVR path: `src/rendering/gl/stereo3d/gl_openvr.cpp:1732-1740` writes `weaponoffset[]`
  (controller minus HMD, in metres, then yaw-rotated into the Doom frame);
  `gl_openvr.cpp:1761-1769` does the identical thing for `offhandoffset[]`.
- Consumed into the actor fields at `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:588-596`
  (main hand) and `:601-609` (offhand).

**You get two full 6-DOF hand poses: position (absolute world map units) and yaw/pitch/roll (degrees, world frame), for both hands, every tic.** Punch recognition for the weapon hand is the same code as for the offhand with four field names changed. Nothing in this topic is gated on single-hand tracking.

**Haptics: the hardware path is real, but there is no ZScript binding.** `OpenVRMode::Vibrate` is a genuine implementation (`src/rendering/gl/stereo3d/gl_openvr.h:121-124` → `OpenVRHaptics::Vibrate` `gl_openvr.cpp:517-528` → `TriggerHapticPulse` `:546`, gated on `vr_enable_haptics`); only `VR_HapticEvent` is an empty stub on PC (`src/QzDoom/qzdoom_common.cpp:65-67`). Neither is bound to the VM. So the engine's own fire/damage/pickup buzzes fire, and a mod can *provoke* two of them (§3.7) — but **you cannot ask for a punch-landed thump of a chosen intensity and length.** For a boxing report that is a real loss, and it is the highest-value engine addition for this topic precisely because it is a *binding*, not a feature: see W2.

### 0.2 The five real asymmetries — design around these

Pose is symmetric. These are not, and they are what actually shapes the designs below.

1. **`OverrideAttackPosDir` gates everything.** When false (flatscreen, `puristmode`, `vr_override_weap_pos` off — `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:438`), `AttackPos` and `OffhandPos` are both overwritten with `player->mo->PosAtZ(shootz)` — i.e. **both hands collapse onto the player's centre and become identical** (`hw_vrmodes.cpp:442-443`). The engine itself branches on this flag throughout `src/playsim/p_map.cpp` (`:4597`, `:4647`, `:4784`, `:4888`, `:5185`) and `src/playsim/p_actionfunctions.cpp:1302`. Every mechanic in this document must early-out on `!OverrideAttackPosDir`.
2. **`VRMode::SetUp()` resets both hands to the body centre every frame before the device code overwrites them** (`hw_vrmodes.cpp:431-446`). The device code only runs inside `if (gamestate == GS_LEVEL && menuactive == MENU_Off && !paused)` (`gl_openvr.cpp:1704`). On any frame where that guard fails, **both hands teleport to the player's centre.** This is the #1 false-positive source. Mandatory spike filter in every sampler (§3.2).
3. **The player's `angle`/`pitch` follow the HEAD, not the weapon hand.** `VR_GetMove` returns `*yaw = hmdorientation[YAW] + snapTurn` (`src/QzDoom/qzdoom_common.cpp:119-121`). The weapon hand only overrides the *origin and direction inside* `P_LineAttack` / `P_AimLineAttack` / `P_LineTrace` (`src/playsim/p_map.cpp:4784-4800`, `:4597-4612`, `:5185-5200`). Autoaim still starts from the hand but the player's facing is the head's.
4. **`weaponStabilised` — the single worst trap in this topic. Read this twice.**

   When `vr_two_handed_weapons` is on and the player **presses the offhand grip button while the hands are within 0.50 m of each other**, the engine sets `weaponStabilised` (`gl_openvr.cpp:1716-1728`, distance test at `:1722`). It is **not gated on `+Weapon.TwoHanded`, not gated on what is in either hand, and not gated on anything a mod can set.** It is a raw grip-button-plus-proximity test.

   While it is set, **the offhand weapon goes completely dark**:
   - fires nothing — `player.zs:393-396` returns early out of the ready handler for `hand == 1`
   - checks no buttons — `src/playsim/p_pspr.cpp:755-758`
   - renders nothing — `src/rendering/hwrenderer/scene/hw_weapon.cpp:832-835` and `:917-920` both `continue` past any psprite whose caller is `player->OffhandWeapon`

   And `WF_TWOHANDSTABILIZED` is **re-asserted on every `P_SetPsprite`** (`p_pspr.cpp:512-515`), so script cannot durably clear it — clear the bit and the next state set puts it back.

   Why this is lethal *specifically* for melee: a two-handed sword grip, a boxing guard with both fists at the face, hands brought together to block, a clinch, a two-handed overhead — **every one of them puts the hands inside 0.50 m.** If the player happens to be holding the offhand grip at that moment, their left hand's weapon silently stops firing and stops drawing, and you will spend a day debugging your own perfectly correct code.

   It also hijacks the main hand's *angles* (not its position): `weaponangles[PITCH/YAW]` are overwritten with the hand-to-hand vector (`gl_openvr.cpp:1742-1754`), so `AttackAngle`/`AttackPitch` describe the two-hand line rather than the wrist. `AttackPos` is unaffected.

   Readable as `player.WeaponState & WF_TWOHANDSTABILIZED` (`wadsrc/static/zscript/constants.zs:1242`).

   **The design response taken in this report, stated once and applied throughout §4: the fists are not weapons.** Punch detection, guard, grabs and knees all run from a carried `Inventory` token's `DoEffect` (§3.1) reading `AttackPos`/`OffhandPos` directly, and resolve through `LineAttack` called on the player pawn. Nothing in §4.1–4.8 depends on an offhand `Weapon` existing, having a psprite, or receiving button events. Stabilisation can therefore flip on and off mid-guard and nothing breaks.

   The one mechanic that *does* need an offhand weapon is an **offhand blade** (§4.9/§4.10). There, the response is the opposite: detect `WF_TWOHANDSTABILIZED`, treat it as the deliberate two-handed grip, and **tell the player** (HUD glyph + a grip sound), so the dark offhand psprite reads as "you took both hands to the hilt" rather than as a bug. Never ship a dual-wield blade pair without that readout.
5. **Model mirroring on the offhand.** `VRMode::GetWeaponTransform` applies `scale(-1, 1, 1)` to the non-dominant hand's psprite model unless the weapon has `+Weapon.NoAutoReverse` (`hw_vrmodes.cpp:453-470`, flag at `wadsrc/static/zscript/actors/inventory/weapons.zs:101`, `src/gamedata/a_weapons.h:165`). The *angles* are not mirrored. A blade in the offhand draws with its edge flipped relative to your edge-vs-flat maths unless you set that flag.

### Build-configuration caveat you must verify on the target binary

`VR_OPENXR_MOBILE` mode is guarded by `#ifdef USE_OPENXR` (`hw_vrmodes.cpp:264-267`), and **`USE_OPENXR` is not defined anywhere in this tree** (only `USE_OPENVR`, at `src/CMakeLists.txt:420`, behind `option(ENABLE_OPENVR ... NO)`). So on a PC build of this tag, the only live VR mode is **OpenVR/SteamVR, OpenGL only**, and all hand data comes from `gl_openvr.cpp`. `hw_openxrdevice.cpp` compiles (`src/CMakeLists.txt:992`) but its mode is unreachable. `weaponoffset`/`offhandoffset`/`weaponangles`/`offhandangles` are written by **nothing else in the repository**. If your target binary is built with OpenXR, that backend lives outside this tree and must be re-surveyed.

### Units — the single most useful number in this document

`vr_vunits_per_meter` defaults to **34.0** (`hw_vrmodes.cpp:94`). The tic rate is 35 Hz.

> **1 map unit per tic ≈ 1.03 m/s — horizontally.** (`1 mu/tic × 35 tic/s ÷ 34 mu/m = 1.029 m/s`)

So every horizontal velocity threshold in this document can be read directly as metres per second. A 6 m/s jab is ~5.8 mu/tic. A sabre tip at 18 m/s is ~17.5 mu/tic. Player radius 16, height 56; imp radius 20, cacodemon radius 31.

**Z is on a different scale, and this will cost you your uppercut.** The hand transform divides the vertical component by `pixelstretch` (`gl_openvr.cpp:1282-1292`, and the same `/ pixelstretch` in the `AttackPos.Z` / `OffhandPos.Z` formulae at `hw_openxrdevice.cpp:595` and `:608`). With the default `pixelstretch = 1.2`:

```
X, Y:  1 m = 34.00 mu
Z:     1 m = 34 / 1.2 = 28.33 mu          <-- ~17% SHORT
```

So a vertical hand motion produces ~17 % fewer map units than the same motion horizontally. Any test that mixes axes — `D.Length()`, `u / D.Length()` in the uppercut classifier, the sagitta estimate for a vertical sabre arc — is wrong unless you normalise. `pixelstretch` is readable from ZScript (`wadsrc/static/zscript/doombase.zs:479`, `LevelLocals.pixelstretch`; also on `LevelInfo` at `:366`). **Normalise once, at the sampler, and never think about it again:**

```zscript
// In DoEffect, immediately after reading the raw poses:
double zfix = level.pixelstretch;                 // 1.2 by default
nl.Z *= zfix;  nr.Z *= zfix;                      // now Z is on the same mu/m as X and Y
```
Every threshold in §4 assumes this normalisation has already happened. Apply the inverse (`/ zfix`) only when you hand a Z back to the engine — i.e. when building the `offsetz` argument of a `LineTrace`/`LineAttack` (§3.5), which wants raw map-space Z.

`HmdPosition` is **raw metres in play space, un-yaw-rotated** (`hw_vrmodes.cpp:445`, from `VR_SetHMDPosition` at `src/QzDoom/qzdoom_common.cpp:93-95`). It is **not** in the same frame or unit as `AttackPos`/`OffhandPos`, which are absolute world map units. Do not subtract one from the other. See §3.3 for the correct head reference.

---

## 1. VERDICT

This engine carries this topic well. Two complete tracked hands, each with its own weapon, psprite layer, state machine and trigger button bit, is a shipped engine feature — not a hack. Hit resolution is covered by three good primitives (`LineTrace`, `LineAttack` with absolute positioning, `BlockThingsIterator.CreateFromPos`), and the swept-chord trace between tics is a genuine, cheap and complete fix for fast swings — six traces per tic makes a sabre correct. Per-actor hit-stop exists as a writable field (`Actor.freezetics`), independent per-hand shapeable rumble is reachable from script today through the quake channel, and physically ducking already shrinks the player's hitbox with no code at all.

What is genuinely missing is small and nameable: no 35-Hz-beating hand history (which costs you curved gestures and nothing else), no knockdown/prone state in Doom's monster model, no ZScript binding for `VRMode::Vibrate`, and no play-space yaw (costing the *direction* of a head slip but not its timing). Boxing, brawling and blades are all buildable to a shippable standard. Punch-Out-style patterned opponents need their own monster classes — telegraph windows cannot be retrofitted onto arbitrary third-party monsters.

Two things will cost a day each if the team does not read them first: `weaponStabilised` silently kills the offhand weapon whenever the hands come within 0.50 m with the grip held (§0.2(4)) — which is *every* guard, block and two-handed grip; and Doom collision is an unrotatable axis-aligned square, not a cylinder (§3.4), which rules out the hitbox-actor approach for punches outright.

Bluntly: the expensive part of this topic is **tuning**, not engine capability. Budget accordingly.

---

## 2. WHAT ALREADY EXISTS

| Thing | Where | Note |
|---|---|---|
| Two 6-DOF hand poses, readonly, per frame | `actor.zs:268-276` (main), `:272-275` (offhand) | §0 |
| Hand-space direction helpers | `actor.zs:858` `AttackDir`, `:859` `OffhandDir`; impl `src/playsim/p_actionfunctions.cpp:5694-5712`, maths at `hw_vrmodes.cpp:390-424` | Converts a (yaw,pitch) relative to the hand into a world unit vector |
| Per-hand weapon with its own psprite layer | `PSP_OFFHANDWEAPON` throughout `wadsrc/static/zscript/actors/player/player.zs:411,446,563,1947,1998` | |
| `+Weapon.OffhandWeapon` flag | `weapons.zs:98`, `src/gamedata/a_weapons.h:162` | Routes a weapon to the left hand |
| `+Weapon.TwoHanded` flag | `weapons.zs:100` | **Clears the other hand's weapon** — `player.zs:1980-1992`. Do not use for a two-fist setup. |
| `+Weapon.NoAutoReverse` | `weapons.zs:101` | Stops the offhand model mirror |
| `+Weapon.NoHandSwitch`, `SwitchWeaponHand(hand)` | `weapons.zs:99`, `player.zs:2488-2505` | Move a weapon between hands at runtime |
| **Dedicated offhand input bits** | `constants.zs:866-868`: `BT_OFFHANDATTACK = 1<<26`, `BT_OFFHANDALTATTACK = 1<<27`, `BT_OFFHANDRELOAD = 1<<28`; `BT_MAINHANDRELOAD = 1<<20` (`constants.zs:869`) | Read via `player.cmd.buttons`. A real per-hand trigger. |
| Per-hand ready/reload/user button routing | `src/playsim/p_pspr.cpp:100-116` (`ButtonChecks[]`), `:748-778` | `WRF_AllowUser1..4` give you four scripted per-hand buttons |
| Two-handed grip detection | `gl_openvr.cpp:1706-1728`; exposed as `player.WeaponState & WF_TWOHANDSTABILIZED` (`constants.zs:1242`) | Grip button + hands within 0.50 m |
| Melee hitscan from the hand | `Actor.LineAttack` `actor.zs:804`; hand-origin logic `src/playsim/p_map.cpp:4784-4800` | `LAF_ISMELEEATTACK`, `LAF_ISOFFHAND`, `LAF_ABSPOSITION` (`constants.zs:938-947`) |
| Non-damaging trace, hand-aware | `Actor.LineTrace` `actor.zs:805`; `src/playsim/p_map.cpp:5185-5215` | `TRF_USEWEAPON`, `TRF_ISOFFHAND`, `TRF_ABSPOSITION`, `TRF_SOLIDACTORS` (`constants.zs:948-962`) |
| Rich trace result | `struct FLineTraceData` `actor.zs:36-61` — `HitActor`, `HitLine`, `HitLocation`, `HitDir`, `Distance`, `HitType` | `HitLocation.Z` is how you decide head vs body |
| Scriptable trace with callback | `class LineTracer` `wadsrc/static/zscript/doombase.zs:307-330`, `struct TraceResults` `:279-305` | Subclass and override `TraceCallback()` for multi-hit / pass-through-corpses |
| Box-of-actors iterator | `BlockThingsIterator.CreateFromPos(x,y,z,h,radius,ignorerestricted)` `doombase.zs:226`, impl `src/scripting/vmiterators.cpp:219-229` | Ignores geometry |
| **Per-actor hit-stop** | `Actor.freezetics` (writable uint) `actor.zs:265`, thunk `vmthunks_actors.cpp:2130`, enforced `src/playsim/p_mobj.cpp:3948-3952`, `src/playsim/actorinlines.h:174-176` | Stops Tick entirely — movement *and* state advance |
| Global freeze | `Level.SetFrozen(bool)` — used at `wadsrc/static/zscript/actors/inventory/powerups.zs:1551` | Binary, excludes the player |
| Forced pain / stagger | `Actor.TriggerPainChance(Name mod, bool forcedPain = false)` `actor.zs:800`; `PainChance` field `actor.zs:236` | |
| Runtime state-machine surgery | `MeleeState` / `MissileState` as writable fields `actor.zs:245-246`; `SetStateLabel` `actor.zs:839`; `InStateSequence(state, base)` `actor.zs:819`; `A_SetTics` `actor.zs:1285` | `InStateSequence` is how you detect "is this monster mid-telegraph" |
| Hitbox resizing | `A_SetSize(radius, height, testpos)` `actor.zs:1314` | For downed/crouched monsters |
| Knockback / throws | `Thrust` `actor.zs:860`, `Vel3DFromAngle` `:857`, `VelIntercept` `:855`, `A_ChangeVelocity` `wadsrc/static/zscript/actors/actions.zs:63`, `A_Recoil` `actor.zs:1256` | |
| Teleport / grab positioning | `Warp(...)` `actor.zs:942` (flags `constants.zs:522-538`), `SetOrigin(vec3, moving)` `actor.zs:753`, `TryMove` `:831`, `CheckMove` `:832` | `CheckMove` first, then `SetOrigin` — `SetOrigin` does no collision test |
| Wall-contact readback | `BlockingMobj` `actor.zs:205`, `BlockingLine` `:206`, `MovementBlockingLine` `:207`, `BlockingFloor/Ceiling` `:209-210` | This is your wall-bounce trigger |
| Collision override | `virtual bool CanCollideWith(Actor other, bool passive)` `actor.zs:562` | Required for grabs |
| Missile reflection (passive) | `bReflective` + `A_SetReflective()` `actor.zs:962-965`; `P_ReflectOffActor` `src/playsim/p_map.cpp:3832-3872`; `MF7_AIMREFLECT`/`MF7_MIRRORREFLECT` branches at `:3845-3858` | Fires only when a missile's *move* is blocked by the actor |
| **Physical duck already shrinks the hitbox** | `player->crouchfactor = HmdHeight / defaultViewHeight` `gl_openvr.cpp:2512-2517`; `Height = FullHeight * crouchfactor` `player.zs:152`; `viewheight = ViewHeight * crouchfactor` `player.zs:1181` | And it is **unclamped** on the VR path: the `clamp(0.5, 1.)` at `player.zs:1177-1180` is inside `if (direction != 0)`, and VR calls `CrouchMove(0)` (`player.zs:1211-1214`). |
| **Physical lean already moves the player** | `positional_movementForward/Sideways` `gl_openvr.cpp:1779-1785`, consumed by `VR_GetMove` `src/QzDoom/qzdoom_common.cpp:116-117` | A real slip is a real world-space dodge for free. Note: only the OpenVR path writes these — the OpenXR device file never does. |
| Script-reachable rumble, path A | `A_QuakeEx(...)` `actor.zs:1284` → `vr_quake_haptic_level` (default **0.8**, `hw_vrmodes.cpp:127`) → `vrmode->Vibrate` per frame, **left channel from intensityX, right from intensityY** — `src/rendering/r_utility.cpp:1084-1096` | Camera shake is *off* by default in VR (`r_quakeintensity` default `0.0f`, `r_utility.cpp:116` — comment: "Defaulted to 0 for VR"), so this is a pure haptic channel |
| Script-reachable rumble, path B | Any `LineAttack`/`SpawnPlayerMissile`/railgun from a player with `damage > 0` fires a 150 ms, 0.8-intensity pulse on the **attacking hand**, picked by `LAF_ISOFFHAND` — `src/playsim/p_map.cpp:4762-4774`, `:5692-5704`, `src/playsim/p_mobj.cpp:7547-7562` | |
| Rumble backend | `OpenVRMode::Vibrate` `src/rendering/gl/stereo3d/gl_openvr.h:121-124` → `OpenVRHaptics::Vibrate` `gl_openvr.cpp:517-528` → `TriggerHapticPulse` `:546`; gated by `vr_enable_haptics` | |
| Visual effects without actors | `class VisualThinker` `wadsrc/static/zscript/visualthinker.zs:1-62` | Cheap trails, sparks, blood |
| Events | `WorldThingDamaged`, `WorldThingDied`, `WorldTick`, `PlayerSpawned`, `NetworkCommandProcess` — `wadsrc/static/zscript/events.zs:160-205` | |

### Haptics limitation, stated precisely

`OpenVRHaptics::Vibrate` **early-returns if a pulse is already running on that channel** (`gl_openvr.cpp:519-523`). Consequences:

- A 150 ms `LineAttack` impact pulse **blocks all other rumble on that hand for 150 ms.**
- `A_QuakeEx`'s per-frame 10 ms pulses are self-expiring and therefore chainable — this is the usable continuous channel.
- `QuakePower` returns a **random** value in `[-intensity, +intensity] × factor` (`r_utility.cpp:911-921`), so quake rumble is noise, not a waveform. Use `QF_WAVE` if you want it to breathe. Set `damage: 0` and `damrad: 0` on every cosmetic `A_QuakeEx` call.

---

## 3. BUILDING BLOCKS

### 3.1 Where the sampler lives

Three valid homes, in order of preference:

1. **`Inventory.DoEffect()`** on a token the player always carries (`wadsrc/static/zscript/actors/inventory/inventory.zs:982`). Runs once per tic for the owner, in play scope, with `owner` in hand. This is the right place.
2. **`PlayerPawn.Tick()`** override. Equivalent, but you are now in the critical path for everything.
3. **`StaticEventHandler.WorldTick()`** (`events.zs:177`) iterating `players[]`. Use only for cross-player / global bookkeeping.

Do **not** put sampling in a weapon's state machine — states skip tics.

### 3.2 The canonical hand sampler

```zscript
// RSQ_HandTrack : Inventory. Carried permanently. CHANGEABLE CONSTANTS AT TOP.
const RING = 8;              // 8 tics = 229 ms of history. Enough for every gesture in §4.
const SPIKE_REJECT = 40.0;   // mu/tic. Above this it is a menu frame or a teleport, not a hand.

Vector3 posL[RING], posR[RING];   // L = offhand, R = main hand
double  angL[RING], pitL[RING], rolL[RING];
double  angR[RING], pitR[RING], rolR[RING];
Vector3 headPos[RING];            // = owner.pos  (see 3.3)
int head;                         // ring write index
bool valid;                       // false while the buffer is not yet trustworthy
int  warmup;

override void DoEffect()
{
    Super.DoEffect();
    let p = PlayerPawn(Owner);
    if (!p || !p.player) return;

    if (!p.OverrideAttackPosDir) { valid = false; warmup = 0; return; }  // flatscreen / purist

    Vector3 nl = p.OffhandPos, nr = p.AttackPos;

    // SPIKE FILTER — mandatory. hw_vrmodes.cpp:442 collapses both hands onto the body
    // centre on any frame where the device update did not run (menu, pause, console).
    int prev = (head + RING - 1) % RING;
    if (warmup > 0)
    {
        if ((nl - posL[prev]).Length() > SPIKE_REJECT
         || (nr - posR[prev]).Length() > SPIKE_REJECT
         || (nl - nr).Length() < 0.01)          // both hands identical == collapsed
        { valid = false; warmup = 0; return; }
    }

    head = (head + 1) % RING;
    posL[head] = nl;  posR[head] = nr;  headPos[head] = p.pos;
    angL[head] = p.OffhandAngle; pitL[head] = p.OffhandPitch; rolL[head] = p.OffhandRoll;
    angR[head] = p.AttackAngle;  pitR[head] = p.AttackPitch;  rolR[head] = p.AttackRoll;

    if (warmup < RING) { warmup++; valid = false; return; }
    valid = true;
    // ... detectors run here, see §4
}

Vector3 VelL(int n = 1) { int a=head, b=(head+RING-n)%RING; return (posL[a]-posL[b]) / n; }  // mu/tic
Vector3 VelR(int n = 1) { int a=head, b=(head+RING-n)%RING; return (posR[a]-posR[b]) / n; }
```

Two traps baked into this and worth spelling out:

- **Warm-up.** After `PlayerSpawned`, after a level change, and after every spike rejection, the buffer is garbage. `valid` must gate every detector, and 8 tics of warm-up is 229 ms the player will not notice.
- **`(nl - nr).Length() < 0.01`.** This is the collapse signature from `hw_vrmodes.cpp:442-443` — both hands assigned the same `PosAtZ(shootz)`. It catches the case the spike filter misses on the *first* bad frame.

### 3.3 Head reference, and the correct frame

`HmdPosition` is metres in un-rotated play space and is useless for comparing against hand positions. The correct head reference is **`owner.pos`**, because both hand positions are literally derived as the player's position minus the hand's offset from the HMD:

```
AttackPos.X  = mo->X() - weaponoffset[0] * vr_vunits_per_meter      // gl_openvr.cpp via hw_openxrdevice.cpp:593
AttackPos.Y  = mo->Y() - weaponoffset[2] * vr_vunits_per_meter      // :594
OffhandPos.X = mo->X() - offhandoffset[0] * ...                     // :606
```
and `weaponoffset`/`offhandoffset` are *already yaw-rotated into the Doom frame* (`gl_openvr.cpp:1737-1740`, `:1766-1769`).

Therefore:

```zscript
Vector3 handRelHead = p.OffhandPos - p.pos;    // metres×34, in the DOOM frame. Exact.
double  fwd   = handRelHead.X * cos(p.angle) + handRelHead.Y * sin(p.angle);  // + = in front
double  right = handRelHead.X * sin(p.angle) - handRelHead.Y * cos(p.angle);
```

Calibrate the **signs of `fwd` and `right` once, empirically**, with a `Console.Printf` while standing still with one hand out. Do not trust the sign derivation above — the double negation through `weaponoffset[0] → -X` and the `v.Y / v.X` swap at `gl_openvr.cpp:1739-1740` is exactly the kind of thing that is off by a sign in one of the two backends.

For **vertical** reference use `player.viewz` (`player.zs:3053`), which on the VR path tracks the real HMD height through the unclamped `crouchfactor` (`gl_openvr.cpp:2517`, `player.zs:1181`). `handRelHead.Z` is also exact and frame-free; prefer it.

Head *world* motion (for slips) is `p.pos - headPos[prev]` — because physical lean is pushed into the player's movement (`gl_openvr.cpp:1779-1785`), head translation shows up in `pos`. Physical-only head motion is `HmdPosition` delta magnitude, which is frame-free and therefore safe to use as a scalar.

### 3.4 Hit testing — the three options, compared

| | `LineTrace` | `LineAttack` | `BlockThingsIterator.CreateFromPos` | short-lived hitbox actor |
|---|---|---|---|---|
| Signature | `actor.zs:805` `bool LineTrace(double angle, double distance, double pitch, int flags = 0, double offsetz = 0., double offsetforward = 0., double offsetside = 0., out FLineTraceData data = null)` | `actor.zs:804` `Actor, int LineAttack(double angle, double distance, double pitch, int damage, Name damageType, class<Actor> pufftype, int flags = 0, out FTranslatedLineTarget victim = null, double offsetz = 0., double offsetforward = 0., double offsetside = 0.)` | `doombase.zs:226` `static BlockThingsIterator CreateFromPos(double checkx, double checky, double checkz, double checkh, double checkradius, bool ignorerestricted)` | `Spawn()` + `A_SpawnItemEx` (`wadsrc/static/zscript/actors/attacks.zs:412`) |
| Respects walls/floors | **Yes** | Yes | **No** — pure blockmap box | Yes (via `P_TryMove`) |
| Side effects | **None** | Damage, puff, kickback, obituary, **haptic pulse** | None | Full actor: thinker, collision, missile rules |
| Hits | First only (or override `LineTracer.TraceCallback()`, `doombase.zs:313-328`) | **First `MF_SHOOTABLE` actor, with no intended-target test** — `p_map.cpp:4919-4920`, filter `:4673-4700`. A teammate or a barrel between your fist and the enemy eats the punch. | All in the box | First blocking |
| Arbitrary start point | `TRF_ABSPOSITION` → `startpos = (offsetforward, offsetside, sz)` — `p_map.cpp:5206-5208` | `LAF_ABSPOSITION` — `p_map.cpp:4876-4884` | Native | Native |
| Auto hand origin + direction | `TRF_USEWEAPON \| TRF_ISOFFHAND` — `p_map.cpp:5185-5200` | `LAF_ISOFFHAND` (always, when `OverrideAttackPosDir`) — `p_map.cpp:4784-4800` | — | — |
| Cost | One `FPathTraverse` — same order as a hitscan bullet | Same plus puff spawn + damage path | One blockmap walk; radius 96 ≈ a 2×2 or 3×3 cell sweep (cells are 128 mu) | Spawn + destroy + a full `P_XYMovement` per tic |
| Cost at 2 hands × 3 sub-samples × 35 Hz | ~210 traces/s. Nothing. | n/a (one per landed hit) | ~70 iterations/s. Nothing. | 70 actor spawn/destroy per second. Measurable, and it pollutes `ThinkerIterator` for every other mod. |

**Pick: `LineTrace` for detection, `DamageMobj` for resolution, one `LineAttack` per landed hit purely to get the puff + haptic.**

Why not the others:
- `BlockThingsIterator` is the right shape (a fat fist is a sphere, not a ray) but **it punches through walls**. You would have to follow every candidate with a `CheckSight` or a confirming `LineTrace`, at which point you have paid for both. Keep it for the one case where it wins: **projectile deflection** (§4.11), where you need *all* nearby missiles and the geometry does not matter.
- The hitbox actor loses on every axis: cost, control, and it inherits missile semantics you will spend a day fighting. The one thing it buys — "let the engine's own collision decide" — you actively do not want, for the reason below.

#### Doom collision is an axis-aligned SQUARE, and it does not rotate

This changes how you size every hitbox in this document, so it is stated once, here.

```
p_map.cpp:1418   double blockdist = thing->radius + tm.thing->radius;
p_map.cpp:1419   if (fabs(thing->X() - cres.Position.X) >= blockdist
                  || fabs(thing->Y() - cres.Position.Y) >= blockdist) return true;
```
and the same square test on the trace path:
```
p_trace.cpp:695  if (fabs(hit.X - in->d.thing->X()) > in->d.thing->radius ||
p_trace.cpp:696      fabs(hit.Y - in->d.thing->Y()) > in->d.thing->radius) return true;
```

An actor of "radius" `r` is a **2r × 2r axis-aligned box**, not a circle. Nothing in either test reads `Angles.Yaw` — **it is unrotatable.** Consequences you must design around:

- A hitbox actor of radius `r` reaches `r√2 ≈ 1.41 r` on the diagonal. A "12 mu fist" would connect at 17 mu to the north-east and 12 mu to the north. Punches would land from visibly different distances depending on which way the player happens to be facing. This alone disqualifies the hitbox-actor approach for boxing.
- The same applies to the *targets*: an imp (`radius 20`) is a 40 × 40 box. Your sweep's `fatten` is being added to a square, not a circle, so a trace that grazes a corner registers as a clean body shot. If that matters, confirm with a real `Distance3D` against `HitActor` after the trace and reject beyond `victim.radius + fatten`.
- A blade is long and thin; a box is not. Do **not** try to represent a blade as an actor. The segment traces in §3.5 and §4.9 are the only honest representation.

Related, so you do not repeat a commonly-quoted wrong number: **the 30 units/tic `MAXMOVE` clamp is conditional and normally does not apply.** `p_mobj.cpp:2096-2106` only clamps under `COMPATF_WALLRUN` (off by default — compatmode 0, `src/d_main.cpp:621`), in water, or while crouched below 0.75. The unconditional ceiling is 5000 u/tic (`p_mobj.cpp:2107-2111`). Irrelevant to a punch; relevant if you quote it at someone.

### 3.5 The swept-chord trace — the fix for fast swings

A fist travelling at 9 m/s moves ~8.7 mu per tic; a sabre *tip* at 18 m/s moves ~17.5 mu per tic. An imp is 40 mu across, a former human 40 mu. So a single-tic step does not usually jump clean over a target — but it does clip corners, and against a lost soul (radius 16) at full swing speed it absolutely tunnels. Point-sampling the hand position is wrong and will feel broken.

The fix is one trace along the chord between the last two samples:

```zscript
// Swept segment test. P0 = previous tic's point, P1 = this tic's point.
// For a fist, P = hand position. For a blade, P = the TIP (§4.9).
bool SweepHit(Actor src, Vector3 p0, Vector3 p1, double fatten, out FLineTraceData d)
{
    Vector3 ch = p1 - p0;
    double len = ch.Length();
    if (len < 0.01) return false;
    double ang = atan2(ch.Y, ch.X);
    double pit = -asin(ch.Z / len);          // p_map.cpp:5180: direction.Z = -sin(pitch)
    return src.LineTrace(ang, len + fatten, pit,
                         TRF_ABSPOSITION | TRF_SOLIDACTORS | TRF_BLOCKSELF,
                         p0.Z,        // offsetz        -> startpos.Z   (p_map.cpp:5207)
                         p0.X,        // offsetforward  -> startpos.X
                         p0.Y,        // offsetside     -> startpos.Y
                         d);
}
```

`fatten` is the effective radius of the striking surface: **8 mu for a fist (~24 cm diameter), 0 for a blade tip**, because the blade's own length already gives you the reach (§4.9).

**The residual failure mode, and its fix.** The chord is a straight line; a hook or a sabre arc *bows outward past the chord*. A 70 mu blade rotating 60° in one tic has a sagitta of `70 × (1 − cos 30°) ≈ 9.4 mu` — enough to miss a target sitting in the bulge. Fix: spherically sub-sample. Interpolate the hand position linearly and the hand *direction* by lerping (angle, pitch) across the tic, rebuild the tip at each sub-step, and trace each sub-chord:

```zscript
const SUBS = 3;                     // 3 is enough below ~25 mu/tic. Use 5 above.
Vector3 prevPt = TipAt(0.0);
for (int i = 1; i <= SUBS; i++)
{
    Vector3 pt = TipAt(double(i) / SUBS);   // lerp(pos0,pos1,t) + bladeLen * DirFrom(lerpAngles(t))
    if (SweepHit(src, prevPt, pt, 0, d)) { ...; break; }
    prevPt = pt;
}
```

Six traces per tic for a two-handed sword. Still nothing.

**Angle lerping gotcha:** `AttackAngle` / `OffhandAngle` wrap. Lerp the *shortest* arc: `a0 + Actor.deltaangle(a0, a1) * t`.

### 3.6 Hit-stop

```zscript
victim.freezetics = 4;          // ~114 ms. 3 is a tap, 6 is a heavy, 10 reads as a bug.
```
`freezetics` halts `AActor::Tick` outright (`p_mobj.cpp:3948-3952`) — the monster stops moving *and* stops advancing states, which is exactly the Streets-of-Rage freeze. Do **not** set it on the player: hands are tracked outside the playsim, so a frozen player is a player whose hands stop registering while the world keeps rendering. If you want the player's *weapon* to pause, pause the psprite with `A_SetTics` instead (`actor.zs:1285`).

For the big finisher, `Level.SetFrozen(true)` for 6-8 tics freezes everything except the player (`powerups.zs:1551`, `actorinlines.h:174-190`). Use sparingly; it stops ambient actors too.

### 3.7 Rumble, concretely

```zscript
// Impact rumble on the hand that landed the hit, 150 ms @ 0.8 — free, comes with the hit.
p.LineAttack(ang, range, pit, damage, 'Melee', "RSQ_FistPuff",
             LAF_ISMELEEATTACK | (isOffhand ? LAF_ISOFFHAND : 0) | LAF_ABSPOSITION,
             t, p0.Z, p0.X, p0.Y);
// p_map.cpp:4762-4774 fires vrmode->Vibrate(150, hand, 0.8) because damage > 0.

// Continuous / shaped rumble: A_QuakeEx. intensityX -> LEFT controller, intensityY -> RIGHT.
// r_utility.cpp:1084-1090. Camera shake is off in VR by default (r_quakeintensity = 0).
p.A_QuakeEx(leftIntensity, rightIntensity, 0,
            durationTics, 0 /*damrad*/, 512 /*tremrad*/, "",
            QF_SCALEDOWN | QF_WAVE, damage: 0);
```

Set `damrad: 0` and `damage: 0` or you will be damaging the player. `tremrad` must cover the player or `StaticGetQuakeIntensities` returns nothing (`r_utility.cpp:1063`). Remember the 150 ms channel lock (§2) — do not stack an impact pulse and a quake on the same hand in the same tic; the quake will be silently dropped.

#### The quake channel is a real, shapeable, per-hand rumble API — **label: HACK / S**

This is worth stating as its own mechanic because it changes what the topic can deliver. `A_QuakeEx`'s X intensity drives the **left** controller and its Y intensity drives the **right**, independently, with **no view motion at all**:

```
r_utility.cpp:1085   if (vr_quake_haptic_level > 0.0) {
r_utility.cpp:1086       left  = QuakePower(vr_quake_haptic_level, jiggers.Intensity.X, jiggers.Offset.X);
r_utility.cpp:1087       right = QuakePower(vr_quake_haptic_level, jiggers.Intensity.Y, jiggers.Offset.Y);
r_utility.cpp:1090       vrmode->Vibrate(10, 0, (float)left);     // LEFT
r_utility.cpp:1091       vrmode->Vibrate(10, 1, (float)right);    // RIGHT
```
and the camera shake that would normally ride along is off by default in VR — `r_quakeintensity` is `0.0f` with the literal comment *"Defaulted to 0 for VR"* (`r_utility.cpp:116`), read as `quakeFactor` at `:1066`.

So: intensity in X for the left fist, Y for the right, short duration, zero damage radius. **A punch now lands in the fist that threw it.** Same for a clash on the hand holding the blade (§4.10), and for a guard absorbing a hit (§4.4).

Because `Vibrate` here is hard-coded to a 10 ms pulse (`r_utility.cpp:1090-1091`), the pulses expire every frame and therefore **chain** — which is what makes the channel *shapeable*. Re-fire with a falling intensity each tic and you get an envelope:

```zscript
// Shaped per-hand rumble. Call once per tic from the token's DoEffect while rumbleTics > 0.
// Jab:        peak 0.55, 3 tics, sharp decay   -> a tap
// Heavy cross: peak 1.00, 7 tics, slow decay   -> a swell
double k = double(rumbleTics) / rumbleLen;
double amp = rumblePeak * (k ** rumbleShape);          // shape 2.5 = sharp, 0.7 = swell
p.A_QuakeEx(leftHand ? amp : 0, leftHand ? 0 : amp, 0,
            1 /*duration tics*/, 0 /*damrad*/, 512 /*tremrad*/, "",
            QF_SCALEDOWN, damage: 0);
rumbleTics--;
```

**Caveats, all of which must be in the code:**
- `vr_quake_haptic_level` is a **user cvar** (default 0.8, `hw_vrmodes.cpp:127`). Read it with `CVar.GetCVar` (`doombase.zs:82`), and if it is 0, degrade gracefully — fall back to sound and hit-stop, and do not pretend the feedback happened.
- A user who has **raised `r_quakeintensity`** gets camera shake along with every one of your punches. Keep `tremrad` tight (512) and intensities modest so that is survivable rather than emetic.
- `QuakePower` returns a **random** value in `[-intensity, +intensity]` (`r_utility.cpp:911-921`), so a single frame is noise; the *envelope* across tics is what the hand feels. Do not expect a clean waveform.
- This is the OpenVR backend (`OpenVRHaptics::Vibrate` `gl_openvr.cpp:517-528`), so it inherits the backend caveat in §0 and `00_CAPABILITIES.md` §0.
- Do not stack this with a `LineAttack` impact pulse on the same hand in the same tic — the 150 ms pulse holds the channel (`gl_openvr.cpp:519-523`) and silently swallows your envelope. **Pick one per hand per event.** For melee, prefer the quake envelope and pass `damage` through `DamageMobj` rather than `LineAttack` when you want full control.

**What you still cannot do:** set an explicit duration and intensity in one call. `VRMode::Vibrate(duration, channel, intensity)` has no VM thunk; there are zero hits for haptic/rumble/vibrate in all of `wadsrc/static/zscript`. The envelope above is a per-tic reconstruction of something the engine could expose directly, and it costs you a 28 ms quantisation floor. See W2.

### 3.8 Three playsim traps that will eat a day each

**(a) `A_ChangeVelocity` is a no-op on the player by default.** `wadsrc/static/zscript/actors/actions.zs:63-79`: when the target is the calling player's own pawn, it early-returns unless `vr_recoil` is set — and `vr_recoil` defaults to **false** (`src/playsim/p_actionfunctions.cpp:90`; the native side has the same gate at `:1294`). Every knockback-the-player mechanic (guard pushback, slip lunge, wall-slam recoil) must **write `Actor.Vel` directly** instead. Monsters are unaffected; `A_ChangeVelocity` and `Thrust` both work normally on them.

**(b) `MovePlayer` annihilates the player's horizontal velocity every grounded tic.** `wadsrc/static/zscript/actors/player/player.zs:1373-1383`:
```
if (!vr_momentum && !player.keepmomentum && player.onground && friction == ORIG_FRICTION)
    vel.XY *= 0.0001;
```
`vr_momentum` defaults to **false** (`hw_vrmodes.cpp:120`). So writing `Vel.XY` on the player and expecting it to carry is futile — it is erased next tic. The escape hatch is `player.keepmomentum`, which **is writable from ZScript** (`player.zs:3108`, native `src/playsim/d_player.h:414`):
```zscript
p.Vel.XY = knockDir * knockSpeed;
p.player.keepmomentum = true;          // survive player.zs:1373 this tic and after
```
It auto-clears once `|vel.x|` and `|vel.y|` both fall below `vr_momentum_threshold` (default **1.0**, `hw_vrmodes.cpp:121`) — `player.zs:1385-1388` — so you set it once and it releases itself. This is deliberate anti-nausea design, not a bug; respect it and keep player knockback short and small.

**(c) `+SKULLFLY` gives you a free impact callback — use it for throws.** `PIT_CheckThing` calls `CallSlam(BlockingMobj)` on any actor with `MF_SKULLFLY` the moment it collides (`src/playsim/p_map.cpp:1552-1557`), and `Slam` is a **ZScript virtual**: `virtual native bool Slam(Actor victim)` (`wadsrc/static/zscript/actors/actor.zs:520`). So a thrown body or a hurled improvised weapon wants `bSkullFly = true` and a `Slam` override, not `bMissile`. You get the victim, you decide the damage, and you avoid inheriting Doom's missile semantics (explode-on-wall, `Death` state entry, projectile damage typing) that you would otherwise spend the afternoon suppressing. See §4.8.

---

## 4. MECHANIC DESIGNS

Constants are given as named values so you can put them in a `RSQ_MeleeTune` singleton and expose sliders.

---

### 4.1 Punch detection — the state machine

**Label: BUILDABLE. Cost: M.**

One instance per hand. Four states.

```
IDLE  --(speed >= LAUNCH_SPEED and hand is retracted)-->  ARMED
ARMED --(accumulate; speed peaks)-->  EXTENDED
EXTENDED --(trace hits, or displacement cap, or speed collapse)--> RECOVER
RECOVER --(hand back within RETRACT_FRAC of peak extension, and REFRACTORY elapsed)--> IDLE
```

**What the player does:** throws a punch. Nothing else.

**Detection:**

| Constant | Value | Meaning |
|---|---|---|
| `LAUNCH_SPEED` | `3.0` mu/tic (~3.1 m/s) | Floor. Below this it is not a punch. |
| `FULL_SPEED` | `9.0` mu/tic (~9.3 m/s) | 100 % power. |
| `MIN_EXTEND` | `16.0` mu (~0.47 m) | Total displacement required before a hit counts. |
| `MAX_WINDOW` | `9` tics (~257 ms) | A punch longer than this is a shove. |
| `REFRACTORY` | `9` tics | Per hand. Human cap is ~5 punches/s. |
| `RETRACT_FRAC` | `0.65` | Hand must return to 65 % of peak extension before re-arming. |
| `CONE_DOT` | `0.5` | Every sample's velocity must stay within 60° of the launch velocity. |
| `FIST_RADIUS` | `8.0` mu | `fatten` for the sweep. |

Each tic, for each hand:

1. `v = Vel(1)`, `s = v.Length()`.
2. **IDLE → ARMED**: `s >= LAUNCH_SPEED` AND the hand is retracted (`extension <= RETRACT_FRAC * peakExtension` from the previous punch, or `peakExtension == 0`) AND `tics_since_last >= REFRACTORY`. Record `launchPos`, `launchVel = v.Unit()`, `launchTic`.
3. **ARMED/EXTENDED, each tic**:
   - `if (v.Unit() dot launchVel) < CONE_DOT` → **abort to RECOVER, no hit.** This is the waggle killer.
   - `peakSpeed = max(peakSpeed, s)`
   - `D = pos[head] - launchPos`; `extension = D.Length()`
   - If `extension >= MIN_EXTEND`, run `SweepHit(p, pos[prev], pos[head], FIST_RADIUS, d)`.
     On `d.HitType == FLineTraceData.TRACE_HitActor && d.HitActor.bShootable` → **resolve** (§4.5), go RECOVER.
     On `TRACE_HitWall/Floor/Ceiling` within `extension` → punched a wall; small self-damage or a thud sound, go RECOVER.
   - If `tic - launchTic > MAX_WINDOW` → RECOVER, no hit.
4. **RECOVER → IDLE** per the table.

**Why the cone test and not a speed threshold:** waggle has speed. Waggle does not have *consistent direction with net displacement*. Together, `CONE_DOT` + `MIN_EXTEND` + `RETRACT_FRAC` eliminate it; any one of them alone does not.

**Power:**
```
t = clamp((peakSpeed - LAUNCH_SPEED) / (FULL_SPEED - LAUNCH_SPEED), 0, 1);
damage = int(BASE_DAMAGE * (0.35 + 0.65 * t ** 1.5));
```
The `0.35` floor means a slow, deliberate, correctly-aimed punch still does something — important, because the alternative teaches players to flail for speed.

**Feel knobs:** `LAUNCH_SPEED` (how "hair-trigger" it feels), `FULL_SPEED` (how hard you must swing to cap), the `1.5` exponent (how much speed is rewarded), `MIN_EXTEND` (how committed a punch must be), `REFRACTORY` (how fast you can machine-gun).

---

### 4.2 Punch classification — jab / cross / hook / uppercut

**Label: BUILDABLE. Cost: M.**

Classify at the moment of resolution, from the window's accumulated data, in the player's facing frame (§3.3):

```
D     = contactPos - launchPos
f     = D · fwd        // forward
r     = D · right      // lateral
u     = D.Z            // vertical
yawSweep = |deltaangle(handAngle[launchTic], handAngle[contactTic])|
```

| Type | Test | Notes |
|---|---|---|
| **Uppercut** | `u > 0` AND `u / D.Length() > 0.45` AND `launchRelHead.Z < -26` AND `contactRelHead.Z > -14` | The height *transition* is the discriminator, not the velocity. `-26` mu ≈ 0.76 m below the head: roughly hip height. |
| **Hook** | `|r| / D.Length() > 0.45` OR `yawSweep > 45°` | `yawSweep` is what actually separates a hook from a wide cross — a hook rotates the forearm. Accept either, because a tight hook has little lateral displacement but lots of yaw. |
| **Overhand / hammer** | `u < 0` AND `|u| / D.Length() > 0.5` | Downward. Feeds §4.8 (stomps). |
| **Jab** | `f / D.Length() > 0.7` AND `D.Length() < 24` AND hand == **lead** hand (§4.3) | Short, straight, lead. |
| **Cross** | `f / D.Length() > 0.7` AND hand == **rear** hand | Straight, rear. Longer by nature; don't test length. |
| **(fallback)** | anything else | Treat as a straight. Do not drop the hit. |

**Jab vs cross is a stance question, not a path question.** The physical paths are nearly identical; what makes a cross a cross is that it comes from the rear hand and crosses the body. Classify by hand-vs-stance, and let the *displacement* feed power instead. This is the single most common mistake in VR boxing code.

**Head vs body shot — decide at resolution, not recognition.** `FLineTraceData.HitLocation` (`actor.zs:50`) gives the exact contact point:

```zscript
double frac = (d.HitLocation.Z - d.HitActor.pos.Z) / d.HitActor.Height;
bool headShot = frac > 0.72;
bool bodyShot = frac < 0.55;
```
Head shots: ×1.8 damage, higher `freezetics`, a "stun" token that opens a 20-tic combo window. Body shots: ×1.0 damage but they add to a per-monster `bodyMeter` that, at threshold, forces a guard-drop state — the Punch-Out body/head economy.

**Feel knobs:** the `0.45` ratios (how strict each classification is), `0.72` head line, the stun-window length. Classification strictness is where "it never registers my hooks" complaints come from — start loose.

---

### 4.3 Stance — orthodox vs southpaw

**Label: BUILDABLE (with a manual override). Cost: S.**

You cannot see the feet, the hips or the shoulders. Two usable signals:

1. **Lead-hand depth.** While `state == IDLE` for both hands, maintain a 35-tic rolling mean of `fwd` (§3.3) for each hand. The **lead hand sits consistently further forward**, typically 8-15 mu (0.25-0.45 m). Declare the leading hand after the mean difference exceeds `STANCE_MARGIN = 7.0` mu for `STANCE_HOLD = 35` tics (1 s), with hysteresis of `2 × STANCE_MARGIN` to flip back.
2. **Lead-hand height.** The lead hand usually rides slightly higher. Weak; use only as a tiebreak.

Orthodox (right-handed) → left hand leads. The *dominant* hand in the engine's sense is the main hand (`vr_control_scheme < 10` means right-handed, `hw_vrmodes.cpp:461`), so:

```zscript
bool southpaw = (leadHand == MAINHAND);   // main hand leading = southpaw for a right-hander
```

**Ship a manual override.** A cvar (`rsq_stance 0=auto, 1=orthodox, 2=southpaw`) plus a one-button "set stance" that snapshots the current pose. Auto-detection is a convenience, not a contract, and a player who guards with both hands level will flicker. Read cvars via `CVar.GetCVar(name, player)` (`doombase.zs:82`).

**Feel knobs:** `STANCE_MARGIN`, `STANCE_HOLD`, and whether stance switching mid-combo is allowed (it should be, with a 10-tic lockout).

---

### 4.4 Guard, slips, ducks, weaves

**Label: guard = BUILDABLE/S. Duck = EXISTS (free). Slip timing = BUILDABLE/S. Slip *direction* = HACK/S.**

**Guard.** Hands near the face for `GUARD_HOLD = 4` consecutive tics:
```
|handRelHead.XY| < 20 mu   AND   handRelHead.Z in [-14, +9] mu
```
for **both** hands (full guard) or one (half guard). Resolve by overriding `PlayerPawn.DamageMobj` (`actor.zs:801` is `native virtual`, so it is overridable):

```zscript
override int DamageMobj(Actor inflictor, Actor source, int damage, Name mod, int flags, double ang)
{
    if (guardTics >= GUARD_HOLD && source)
    {
        double rel = absangle(AngleTo(source), self.Angle);
        if (rel < GUARD_ARC)                     // GUARD_ARC = 55 degrees
        {
            if (damage >= GUARD_BREAK)           // GUARD_BREAK = 25
            { guardTics = 0; guardStun = 14; A_StartSound("rsq/guardbreak", CHAN_BODY); }
            else
            { damage = int(damage * GUARD_SOAK); // GUARD_SOAK = 0.25
              // Player knockback: NOT A_ChangeVelocity (no-op, vr_recoil=false) and NOT a bare
              // Vel write (erased next tic by player.zs:1373). See 3.8(a) and 3.8(b).
              double away = AngleTo(source) + 180;
              Vel.XY += (cos(away), sin(away)) * GUARD_PUSH;   // 2.0
              player.keepmomentum = true;
              A_QuakeEx(0.6, 0.6, 0, 6, 0, 512, "", QF_SCALEDOWN, damage: 0);  // both hands rumble
            }
        }
    }
    return Super.DamageMobj(inflictor, source, damage, mod, flags, ang);
}
```

**Duck — already free, and that is a problem too.** `crouchfactor` is driven straight from HMD height on the VR path (`gl_openvr.cpp:2512-2517`) and is **not clamped**, because the `clamp(0.5, 1.)` at `player.zs:1177-1180` only runs when `direction != 0` and VR calls `CrouchMove(0)` (`player.zs:1211-1214`). `Height = FullHeight * crouchfactor` (`player.zs:152`). So crouching physically shrinks the player's collision box, unclamped, with zero script. Enemy melee that uses `CheckMeleeRange` (`actor.zs:799`) or a pitched hitscan will miss a ducked player for real.

That also means **a player who crouches to the floor becomes nearly unhittable**. If that matters, clamp it yourself in your `Tick` override: `if (player.crouchfactor < 0.55) player.crouchfactor = 0.55;` and accept the view/hitbox mismatch, or leave it and design enemies with low sweeps.

**Slip timing.** Head world velocity is `p.pos - headPos[prev]` (§3.3) — physical lean is pushed into player movement at `gl_openvr.cpp:1779-1785`. A slip is:
```
headSpeed >= SLIP_SPEED (0.6 mu/tic ≈ 0.6 m/s) sustained over >= 2 tics
```
If an enemy melee resolves against you during a slip window, grant 1 tic of i-frames and open a `COUNTER_WINDOW = 14` tic counter flag (§4.6).

**Slip direction — the one real hole.** To say "you slipped *inside* the hook" you need the head's lateral motion in the player's facing frame. `p.pos` delta gives you exactly that, so direction is actually available *when the lean is translated into movement*. What you cannot get is **play-space yaw** (`playerYaw`, `src/QzDoom/qzdoom_common.cpp:27`, written at `:89`) — so you cannot separate "the player physically stepped left" from "the player pushed the stick left": both arrive in `pos`. Workaround (HACK): take the *magnitude* of the physical component from `HmdPosition` delta (frame-free, metres) and the *direction* from `p.pos` delta, and require both to be non-trivial:
```
isPhysicalSlip = (HmdPosition - prevHmd).Length() > 0.018   // metres/tic ≈ 0.63 m/s
slipDir        = normalised lateral component of (p.pos - headPos[prev]) in the facing frame
```
This is correct in practice and wrong if the player leans and pushes the stick in opposite directions in the same tic. Acceptable.

**Feel knobs:** `GUARD_ARC`, `GUARD_SOAK`, `GUARD_BREAK`, `SLIP_SPEED`, `COUNTER_WINDOW`.

---

### 4.5 Resolving a landed punch

**Label: BUILDABLE. Cost: S.**

```zscript
void ResolvePunch(PlayerPawn p, FLineTraceData d, int dmg, int punchType, bool offhand, Vector3 p0)
{
    Actor v = d.HitActor;

    // 1. The hit itself. LineAttack rather than DamageMobj so we get the puff, the
    //    obituary, the kickback AND the 150 ms hand rumble (p_map.cpp:4762-4774).
    Vector3 ch  = d.HitLocation - p0;
    double  len = ch.Length();
    p.LineAttack(atan2(ch.Y, ch.X), len + 4, -asin(ch.Z / len), dmg, 'Melee', "RSQ_FistPuff",
                 LAF_ISMELEEATTACK | LAF_ABSPOSITION | (offhand ? LAF_ISOFFHAND : 0),
                 null, p0.Z, p0.X, p0.Y);

    // 2. Hit-stop, scaled by power.
    v.freezetics = 3 + (dmg >= HEAVY_DAMAGE ? 3 : 0);

    // 3. Knockback. Along the swing, not along the player's facing.
    v.Thrust(KNOCK_BASE * dmg / BASE_DAMAGE, atan2(ch.Y, ch.X));
    if (punchType == PT_UPPERCUT) v.Vel.Z += UPPER_LIFT;        // 2.5 reads well

    // 4. Stagger.
    if (dmg >= STAGGER_DAMAGE) v.TriggerPainChance('Melee', true);   // actor.zs:800

    // 5. Wall-bounce arming (see 4.7).
    ArmWallBounce(v, dmg);
}
```

Use `LAF_ABSPOSITION` so the puff spawns where the fist actually is rather than at the engine's default shoot height — otherwise body shots spawn blood at eye level and it looks wrong immediately.

---

### 4.6 Combos, timing windows, counter-punches

**Label: BUILDABLE. Cost: M.**

**Combo buffer.** A ring of the last 6 resolved punches: `{hand, type, tic, damage, victim}`. A combo is a declarative table matched against the tail of that buffer:

```zscript
struct ComboDef { int n; int types[4]; int hands[4]; int maxGapTics; Name reward; }
// e.g. jab(lead) -> jab(lead) -> cross(rear), maxGap 16 tics  =>  "ThreePiece"
```
Match on each resolution: walk backwards, require `types` and `hands` to match and every consecutive `tic` gap `<= maxGapTics`, and that `victim` is the same actor (or not, for crowd combos). `COMBO_GAP = 16` tics (457 ms) is the number to start at — long enough to be forgiving, short enough that it does not fire across a pause.

**Rewards** should be concrete and visible: damage multiplier on the final blow, longer `freezetics`, a distinct `A_QuakeEx` rumble signature, a different puff, and a sound. Do not make combos unlock a different *move* — in VR, the move is the player's arm, and overriding it feels like losing control.

**Counter-punch during wind-up.** This is the Punch-Out heart and it is cheap:

```zscript
bool InTelegraph(Actor m)
{
    State s = m.FindState('Telegraph');
    return s && Actor.InStateSequence(m.CurState, s);     // actor.zs:819
}
```
On resolution, `if (InTelegraph(v)) { dmg *= COUNTER_MULT /*3*/; v.freezetics = 14; v.SetStateLabel('Countered'); }`.

For **third-party monsters with no `Telegraph` label**, fall back to: `v.CurState` is inside `v.MeleeState` or `v.MissileState` (`actor.zs:245-246`) AND the state entered within the last `TELEGRAPH_GRACE = 8` tics. Track entry by caching `(actor, CurState, tic)` in a small map updated from `WorldTick` for monsters within 256 mu of a player. That is a real, bounded cost: a `BlockThingsIterator.Create(playerPawn, 256)` once per tic.

**The counter window from a slip** (§4.4) is the mirror of this: the player slips, a flag opens for `COUNTER_WINDOW` tics, and the next landed punch in that window takes the `COUNTER_MULT`.

**Feel knobs:** `COMBO_GAP`, `COUNTER_MULT`, `COUNTER_WINDOW`, `TELEGRAPH_GRACE`.

---

### 4.7 Stagger, knockdown, wall bounce

**Label: stagger = BUILDABLE/S. Knockdown = BUILDABLE/M for your own monsters, HACK/L for arbitrary ones. Wall bounce = BUILDABLE/M.**

**Stagger** is `TriggerPainChance(mod, forcedPain: true)` (`actor.zs:800`) plus `freezetics`. Free.

**Knockdown.** Doom has no prone state. For **your own** monster classes, add a `Knockdown` / `Downed` / `GetUp` state block and a `downTics` field; on knockdown:
```zscript
v.SetStateLabel('Knockdown');
v.A_SetSize(v.radius, v.Height * 0.35, true);   // actor.zs:1314, testpos=true so it can't clip
v.bNoPain = true;                                // don't let chip damage reset the sequence
```
Restore size in `GetUp`. For **arbitrary third-party monsters**, you cannot inject states — you would have to `CheckReplacement` (`events.zs:205`) them into subclasses at spawn, which breaks mod compatibility in ways you will not enjoy. **Label that path HACK/L and do not ship it.**

**Wall bounce.** Arm the victim and watch it:
```zscript
// On a heavy knockback, remember it.
void ArmWallBounce(Actor v, int dmg) { if (dmg >= BOUNCE_DAMAGE) bounceList.Push(v), bounceTic.Push(level.maptime + 10); }

// Each WorldTick, for each armed victim within its 10-tic window:
if (v.BlockingLine || v.MovementBlockingLine)      // actor.zs:206-207
{
    v.DamageMobj(null, puncher, BOUNCE_DAMAGE, 'Crush');
    v.Vel.XY = -v.Vel.XY * BOUNCE_RESTITUTION;     // 0.45
    v.freezetics = 5;
    v.TriggerPainChance('Crush', true);
    v.A_StartSound("rsq/wallslam", CHAN_BODY);
    // Spawn dust along the wall normal using the line's delta.
}
```
`BlockingLine` is set by the movement code when a move is refused; checking it the tic *after* the thrust is what catches the impact. Note it is also set by ordinary pathing bumps, so gate it on `v.Vel.XY.Length() > BOUNCE_MIN_SPEED` (3.0).

**Feel knobs:** `BOUNCE_DAMAGE`, `BOUNCE_RESTITUTION`, `BOUNCE_MIN_SPEED`, knockdown duration, whether a downed enemy is invulnerable while rising.

---

### 4.8 Grabs, throws, knees, stomps

**Label: grab/throw = BUILDABLE/M. Knee = BUILDABLE/S. Stomp = BUILDABLE/S (needs §4.7's downed state).**

**Grab.** Gate on the per-hand trigger — `player.cmd.buttons & BT_OFFHANDATTACK` (`constants.zs:866`) for the left, `BT_ATTACK` for the right — plus a proximity test from the hand:

```zscript
// Candidate search: a short LineTrace from the hand along the hand's own forward.
Vector3 hf = p.OffhandDir(p, 0, 0);           // actor.zs:859 - hand-forward unit vector
FLineTraceData d;
SweepHit(p, p.OffhandPos, p.OffhandPos + hf * GRAB_REACH /*22 mu*/, 10, d);
if (d.HitActor && d.HitActor.bShootable && d.HitActor.Mass <= GRAB_MAX_MASS) BeginGrab(d.HitActor);
```

**Holding.** Each tic while the trigger is held:
```zscript
grabbed.bNoGravity = true;
grabbed.bSolid     = false;                  // and override CanCollideWith, see below
Vector3 want = p.OffhandPos + hf * (grabbed.radius + 4) - (0, 0, grabbed.Height * 0.5);
if (grabbed.CheckMove(want.XY))              // actor.zs:832 — test BEFORE moving
    grabbed.SetOrigin(want, true);           // actor.zs:753, moving=true keeps interpolation
else
    ReleaseGrab();                           // shoved into geometry: drop it
grabbed.Angle = p.OffhandAngle + 180;
grabbed.freezetics = 2;                      // keep it from running its own AI while held
```
`SetOrigin` does **no** collision check — the `CheckMove` first is not optional, or you will post monsters into walls.

Override `CanCollideWith` (`actor.zs:562`) on your player pawn to return `false` against the currently grabbed actor, both passive and active.

**Throw.** On trigger release, hand the hand's own velocity to the actor:
```zscript
Vector3 hv = VelL(2);                        // 2-tic average — 1 tic is too noisy for a throw
grabbed.bNoGravity = false; grabbed.bSolid = true; grabbed.freezetics = 0;
grabbed.Vel = hv * THROW_GAIN * (THROW_REF_MASS / max(grabbed.Mass, 1));   // THROW_GAIN ≈ 1.6
grabbed.target = p;                          // credit
grabbed.bSkullFly = true;                    // see 3.8(c) — NOT bMissile
```
Mass scaling is what makes throws feel physical: an imp flies, a mancubus barely shifts. `THROW_REF_MASS = 100` (Doom's default `Mass`).

**Use `+SKULLFLY`, not `+MISSILE`.** `PIT_CheckThing` calls `CallSlam(BlockingMobj)` the instant a `MF_SKULLFLY` actor collides (`src/playsim/p_map.cpp:1552-1557`), and `Slam` is a ZScript virtual (`actor.zs:520`). That hands you the victim with full control and none of the missile baggage:
```zscript
override bool Slam(Actor victim)
{
    if (victim && victim.bShootable && victim != thrower)
    {
        int d = int(THROW_DAMAGE * Vel.Length() / THROW_REF_SPEED);
        victim.DamageMobj(self, thrower, d, 'Melee');
        victim.freezetics = 5;  self.freezetics = 5;
        DamageMobj(self, thrower, d / 2, 'Melee');     // the thrown body hurts too
    }
    bSkullFly = false;  Vel.XY = (0, 0);
    SetStateLabel('Knockdown');                        // 4.7
    return true;
}
```
`bSkullFly` must be cleared in `Slam` or the actor keeps slamming. If it hits a wall instead of an actor, the engine clears `MF_SKULLFLY` itself and sends it to `See`/`Spawn` — override `Tick` or watch `BlockingLine` (§4.7) to catch that and route it to `Knockdown` too.

**Knee.** Not a punch — it is a *head* gesture plus a hand gesture. Detect: both hands pulled in toward the body (`|handRelHead.XY| < 18` for both) AND the player's `viewz` drops by `>= 6` mu in 3 tics (the natural dip of a knee) AND an enemy within `KNEE_RANGE = 40` mu in a ±40° cone. Resolve as a punch with `punchType = PT_KNEE`, damage ×1.4, `freezetics = 6`. Cheap, and it reads surprisingly well because the dip is involuntary.

**Stomp.** Requires a downed enemy (§4.7). Detect: a downward overhand classification (§4.2) with `contactRelHead.Z < -40` (below hip, i.e. the player reaching down) OR simply a `BT_CROUCH`-free rapid `viewz` drop over a downed actor within `STOMP_RANGE = 36` mu. Resolve: `DamageMobj` with a large multiplier, `A_QuakeEx(1.0, 1.0, 0, 8, 0, 256, damage: 0)`, and `Level.SetFrozen` for 4 tics if it is the killing blow.

**Improvised weapons with durability.** Pure ZScript, no engine question:
- A `Weapon` subclass with `int durability` and `int maxDurability`.
- Every resolved hit: `durability -= 1 + (dmg >= HEAVY_DAMAGE ? 1 : 0)`.
- At 0: `A_StartSound("rsq/break")`, spawn a debris actor, `Owner.A_SelectWeapon('RSQ_Fists')` (or set `player.PendingWeapon`), `Destroy()`.
- Pick-up by grabbing a world actor (§4.8 grab) with a `RSQ_Improvised` token that names the `Weapon` class to give.
- Draw the wear: swap the model / `Translation` at durability quartiles.

**Feel knobs:** `GRAB_REACH`, `GRAB_MAX_MASS`, `THROW_GAIN`, `THROW_REF_MASS`, knee dip threshold, durability per hit.

---

### 4.9 Blades — tip derivation, arc tracing, thrust vs slash

**Label: BUILDABLE. Cost: M.**

**The blade is not the hand.** Derive the tip every tic:

```zscript
// hand = AttackPos or OffhandPos; dir from the matching Dir() helper.
Vector3 BladeTip(PlayerPawn p, bool offhand, double bladeLen, double gripOffset)
{
    Vector3 hand = offhand ? p.OffhandPos : p.AttackPos;
    Vector3 fwd  = offhand ? p.OffhandDir(p, 0, 0) : p.AttackDir(p, 0, 0);   // actor.zs:858-859
    return hand + fwd * (gripOffset + bladeLen);
}
Vector3 BladeBase(...) { return hand + fwd * gripOffset; }
```
`AttackDir` / `OffhandDir` take a yaw and pitch *relative to the hand* and return a world unit vector (`src/playsim/p_actionfunctions.cpp:5694-5712`, maths at `hw_vrmodes.cpp:390-424`). Passing `(0, 0)` gives you the hand's own forward. **Do not** build the direction yourself from `AttackAngle`/`AttackPitch` — those are world angles that have already been offset by `-90 + viewpointYaw` (`hw_openxrdevice.cpp:590`) and you will get it wrong.

**Fast swings skipping past targets — the direct answer.** A sabre tip at 18 m/s covers 17.5 mu per tic; at 25 m/s, 24 mu. Point-sampling the tip misses a lost soul (radius 16) outright and clips an imp. The fix, already given in §3.5, applied to the blade:

1. Each tic, sub-sample `SUBS = 3` (or 5 above 25 mu/tic) points along the arc, lerping **both** the hand position and the hand direction (shortest-arc angle lerp).
2. For each sub-step, trace **the whole blade segment**, not just the tip: `SweepHit(base[i], tip[i], 0)` *and* `SweepHit(tip[i-1], tip[i], 0)`. The first catches a target standing in the blade's path; the second catches a target the tip swept across.
3. Cost: 2 × SUBS = 6 traces per tic per blade. ~210 traces/s for a one-handed sword. Negligible against a hitscan weapon's budget.

That is 6 line traces to make a sabre feel correct. It is not a wall; it is a loop.

**Thrust vs slash.** From the tip's velocity and the blade's own direction:
```
tipVel = (tip[head] - tip[prev]);  bladeDir = (tip - base).Unit();
align  = tipVel.Unit() dot bladeDir;
align >  0.75  ->  THRUST   (pierce: ignore armour fraction, deeper, narrow, high freezetics)
align <  0.45  ->  SLASH    (cut: see edge test below, wide, bleeding)
in between     ->  SHOVE    (blunt, knockback, no cut)
```

**Edge vs flat.** Define the edge normal in model space (say `+Z` of the hand frame) and rotate it with the hand's roll:
```zscript
// edgeNormal in world space, from the hand's own frame via OffhandDir with a 90-degree pitch,
// then rolled.  Roll comes straight from OffhandRoll / AttackRoll (actor.zs:270, :274).
Vector3 up = offhand ? p.OffhandDir(p, 0, -90) : p.AttackDir(p, 0, -90);
Vector3 sd = offhand ? p.OffhandDir(p, -90, 0) : p.AttackDir(p, -90, 0);
double  rl = (offhand ? p.OffhandRoll : p.AttackRoll) * (pi / 180);
Vector3 edgeN = up * cos(rl) + sd * sin(rl);

double cut = abs(tipVel.Unit() dot edgeN);
cut > EDGE_DOT (0.62)  ->  clean cut: full damage, bleed, dismember
cut < FLAT_DOT (0.35)  ->  flat: damage * 0.3, converted to 'Melee' blunt, +stagger, clang sound
```

**Two gotchas, both real:**
- **The offhand model is mirrored** unless `+Weapon.NoAutoReverse` (`hw_vrmodes.cpp:460-470`, `weapons.zs:101`). The *angles* are not mirrored. Set the flag on every blade or your edge maths will be inverted in one hand only — which presents as "flat hits only work on the left", and will cost you a day if you do not know.
- **Roll sign.** `OffhandRoll` is raw `offhandangles[ROLL]` (`hw_openxrdevice.cpp:604`). Verify the sign with a print before trusting `cos/sin` above.

**Two-handed grip — free, and loaded.** `player.WeaponState & WF_TWOHANDSTABILIZED` (`constants.zs:1242`, set at `src/playsim/p_pspr.cpp:512-515`, source condition `gl_openvr.cpp:1716-1728`) tells you the player has both hands on the hilt, and the engine has already rewritten `weaponangles` to the hand-to-hand line (`gl_openvr.cpp:1742-1754`). A two-handed blade's direction is therefore *better* than a one-handed one — longer baseline, far less jitter. Reward it: `TWOHAND_DAMAGE_MULT = 1.4`, reduced flat-hit penalty, access to the heavy overhead.

**But read §0.2(4) before you build anything around it.** Stabilisation is a raw grip-button-plus-0.50 m test, not a weapon property, and while it is set the offhand weapon fires nothing (`player.zs:393-396`), checks no buttons (`p_pspr.cpp:755-758`) and **draws nothing** (`src/rendering/hwrenderer/scene/hw_weapon.cpp:832-835`, `:917-920`). The bit is re-asserted on every `P_SetPsprite` (`p_pspr.cpp:512-515`), so you cannot clear it and keep it clear.

Concretely, for blades:
- **Dual-wield is hazardous.** Two blades brought within 0.50 m with the grip held makes the left one vanish mid-fight. If you ship dual-wield, you must show a HUD readout of `WF_TWOHANDSTABILIZED` and play a distinct "both hands to the hilt" sound, so the disappearance reads as intentional.
- **Two-handed-only blades are the safe design.** Put the blade in the *main* hand, leave the offhand empty, and treat stabilisation purely as a read-only input. Nothing can go dark because nothing is there.
- **Never hang a mechanic on the offhand weapon's buttons** (`WRF_AllowUser1..4` etc.) if the hands will ever come together.
- The clash and deflect tests (§4.10, §4.11) are unaffected — they read `AttackPos`/`OffhandPos` directly and never touch a psprite.

**Cutting *through* enemies.** `LineTrace` returns the first hit. For a cleave, subclass `LineTracer` (`doombase.zs:307-330`) and return `TRACE_Skip` for actors you have already cut this swing (and `TRACE_Stop` on geometry), accumulating hits in the subclass. Keep a per-swing `Array<Actor> alreadyCut` so one swing cannot hit the same body twice.

**Feel knobs:** `bladeLen`, `gripOffset`, `SUBS`, `EDGE_DOT`, `FLAT_DOT`, thrust/slash alignment thresholds, `TWOHAND_DAMAGE_MULT`.

---

### 4.10 Blade-on-blade clash and parry

**Label: BUILDABLE. Cost: M.**

The enemy's blade must be an actor you control (a child actor warped to the monster's hand each tic, or just a pair of cached `Vector3` fields on your monster class updated in its `Tick`). Then clash is a **segment-segment distance test**, pure maths, no engine support needed:

```zscript
// Shortest distance between segment (a0,a1) and (b0,b1). Standard closest-point-on-two-segments.
// Clash if dist < CLASH_RADIUS (6.0 mu) and the two blades' velocities oppose:
//   (playerTipVel - enemyTipVel) dot (playerTip - enemyTip) < 0
```
On clash:
- Both blades stop: `p.A_QuakeEx(1.0, 1.0, 0, 5, 0, 300, damage: 0)` (rumble), enemy `freezetics = 8`.
- Spark `VisualThinker` at the contact point (`visualthinker.zs:38-62`).
- Whoever had the higher tip speed wins the bind: the loser's weapon goes to a `Recoil` state.
- **A parry is a clash that happens during the enemy's `Telegraph`/attack sequence** (`InStateSequence`, §4.6) — reuse the same test, grant the counter window.

A clean parry should open the enemy for `PARRY_OPEN = 20` tics with `PainChance = 255` and the `COUNTER_MULT` active. That is the Zelda/Condemned beat.

**Feel knobs:** `CLASH_RADIUS`, `PARRY_OPEN`, how long the bind holds, whether tip speed or weapon mass decides the winner.

---

### 4.11 Deflecting projectiles with a blade

**Label: BUILDABLE. Cost: M.**

This is the one place `BlockThingsIterator` wins, because you want *all* nearby missiles and geometry is irrelevant.

```zscript
// Once per tic while a blade is held.
let it = BlockThingsIterator.Create(p, DEFLECT_SCAN /*96 mu*/);    // doombase.zs:225
while (it.Next())
{
    Actor m = it.thing;
    if (!m || !m.bMissile || m.target == p) continue;
    if (m.Distance3D(p) > DEFLECT_SCAN) continue;                  // blockmap box is square

    // The missile's own swept segment this tic.
    Vector3 m1 = m.pos, m0 = m.pos - m.Vel;
    // Blade segment this tic (use the mid-sub-sample, SUBS is overkill here).
    if (SegSegDist(m0, m1, bladeBase, bladeTip) > DEFLECT_RADIUS /*7.0*/) continue;

    // Deflect.
    m.target = p;                               // credit, and stops it re-hitting the shooter's allies
    Actor newTarget = PickNearestHostile(p, DEFLECT_AIM_RANGE /*512*/);
    if (newTarget) m.VelIntercept(newTarget, m.Speed);              // actor.zs:855 — lead the target
    else { Vector3 n = BladeNormal(); m.Vel = (m.Vel - 2 * (m.Vel dot n) * n).Unit() * m.Speed; }
    m.Angle = atan2(m.Vel.Y, m.Vel.X);
    p.A_QuakeEx(offhand ? 1.0 : 0, offhand ? 0 : 1.0, 0, 4, 0, 300, damage: 0);  // one-hand rumble
    p.A_StartSound("rsq/deflect", CHAN_WEAPON);
}
```

`VelIntercept(targ, speed, aimpitch, oldvel, resetvel)` (`actor.zs:855`) is the aim-back; it is what makes a lightsaber deflection feel like a lightsaber deflection rather than a ricochet.

**Do not use `bReflective` on the player.** `P_ReflectOffActor` (`src/playsim/p_map.cpp:3832-3872`) only runs when a missile's *move is blocked by that actor* (`p_mobj.cpp` blocking path, `p_map.cpp:3727-3734`) — i.e. the missile must hit the player's collision box. That gives you "reflect everything that touches me", which is not a blade mechanic, and it reflects things you did not block. The blade-segment test above is the correct shape and costs one blockmap walk per tic.

**Feel knobs:** `DEFLECT_SCAN`, `DEFLECT_RADIUS`, `DEFLECT_AIM_RANGE`, whether a deflection requires the blade to be *moving* (it should — `bladeTipSpeed > 2.0` — or standing still becomes a shield).

---

### 4.12 Gesture-as-special-move at 35 Hz — what survives and what does not

**Label: BUILDABLE for the reliable tier, WALL for the rest. Cost: S per gesture.**

The budget is brutal and must be stated plainly: **a 300 ms gesture is 10 samples. A 150 ms flick is 5. A sabre flourish is 3.**

**RELIABLE — ship these.**
- **Zone-to-zone strokes.** Partition the space around the head into ~8 zones (above head, chest, hip, behind, across-body, outside shoulder, extended, tucked). A gesture = `start zone → end zone`, duration ≥ 4 tics, with a dominant axis. Two-sample confirmation on each zone.
- **Dwell poses.** Hand held in a zone for ≥ 5 tics. Rock solid.
- **Hand separation.** `(AttackPos - OffhandPos).Length()` crossing a threshold. One number, no noise.
- **Both-hands-together / both-hands-apart.** The most reliable gesture in VR.
- **Orientation quadrants.** `OffhandRoll` bucketed to 90° sectors with ±25° hysteresis (palm up / palm down / edge in / edge out). Held for ≥ 3 tics.
- **Single-stroke direction.** Net displacement ≥ 20 mu with all samples within a 60° cone. This is the punch detector (§4.1), reused.

**MARGINAL — prototype before committing.**
- **One direction reversal** (an "L" or a "V"). Needs ≥ 7 tics total (200 ms) to have 3 samples per leg. Players naturally gesture faster than that, so you must *teach* the slow version.
- **Hand + head combination** (e.g. hand up while head ducks). Works, but doubles the false-negative rate.

**NOISE — do not build these. This is the WALL.**
- **Circles, spirals, figure-eights.** Curvature estimation needs ≥ 8 well-spaced samples; a natural circle is 400-600 ms, which gives you 14-21 samples at 35 Hz but with the *corners* — the part that carries the shape — at 2-3 samples each. Detection rates in practice are not shippable.
- **Any gesture with more than one reversal.** Symbol drawing, runes, letters.
- **Anything above ~14 mu/tic.** You get 2 samples; you cannot distinguish a shape from a line.
- **Anything shorter than 4 tics (115 ms).** Below the noise floor of the tracking-to-playsim path.

**The rule to hand the team:** *a gesture ships only if it can be fully described by (start zone, end zone, dominant axis, duration ≥ 5 tics, optional roll quadrant). If you need a sixth clause, it is not a gesture, it is a wish.*

**Missing piece, named:** there is no sub-tic hand history. The poses are written per rendered frame (`hw_vrmodes.cpp:442-445`) but only the **latest** is readable; there is no buffer, no timestamp and no interpolation hook in ZScript. A 90 Hz ring buffer in C++ exposed as `GetHandSample(n)` would move "circles" from NOISE to RELIABLE. That is the exact engine addition this topic would ask for.

---

### 4.13 Punch-Out-style enemies on Doom monster AI

**Label: BUILDABLE/M for your own classes. HACK/L to retrofit arbitrary monsters.**

Doom's AI is grid-chase (`A_Chase`, `actor.zs:1179`): close distance, enter `MeleeState` when `CheckMeleeRange` passes (`actor.zs:799`), otherwise `MissileState`. That is not a boxing opponent. What you keep and what you replace:

**Keep:** the state machine, `MeleeState`/`MissileState` as *writable* fields (`actor.zs:245-246`), `PainChance` (`actor.zs:236`), `TriggerPainChance` (`:800`), `A_FaceTarget` (`:1156`), `A_StartSound`, the Death states, `freezetics`.

**Replace:** `A_Chase`. Write a `Tick`/`A_RSQChase` that paces the opponent on a ring around the player instead of pathfinding to it:

```zscript
// Ring footwork. Keeps a fixed radius, strafes, closes only to attack.
double want = RING_RADIUS;                       // 72 mu reads as "in range but not touching"
double have = Distance3D(target);
A_FaceTarget();
Vector3 toP = (target.pos - pos); toP.Z = 0;
double radial = (have - want) * RING_CLOSE;      // 0.25
double tangent = ringDir * RING_STRAFE;          // 1.2, ringDir flips on a timer
Vector2 step = toP.XY.Unit() * radial + (-toP.Y, toP.X).Unit() * tangent;
if (!TryMove(pos.XY + step, true)) ringDir = -ringDir;     // actor.zs:831
```

**Telegraphs.** A dedicated state block before each attack:
```
Telegraph:
    BOSS EF 7 A_FaceTarget;              // wind-up, distinct frame + bright
    BOSS G 5 A_StartSound("rsq/tell_hook", CHAN_VOICE);
    BOSS G 4 { PainChance = 255; vulnerable = true; }   // THE WEAK WINDOW
    Goto AttackHook;
AttackHook:
    BOSS H 4 A_RSQMelee(12, 'Hook');
    BOSS IJ 6;
Recover:
    BOSS K 10 { PainChance = 255; }      // second weak window
    BOSS L 4 { PainChance = Default.PainChance; vulnerable = false; }
    Goto See;
Countered:
    BOSS M 18 A_Pain;                    // actor.zs:1175
    Goto See;
```
`A_FaceTarget` with a `max_turn` (`actor.zs:1156`) makes the telegraph *commit* to a direction, which is what lets the player slip it.

**Patterns.** A per-monster `int patternStep` and a static table of state labels. In your chase replacement, when `have < ATTACK_RANGE` and the attack cooldown is up:
```zscript
Name nxt = PATTERN[patternIdx][patternStep];
patternStep = (patternStep + 1) % PATTERN_LEN[patternIdx];
SetStateLabel(nxt);                               // actor.zs:839
```
Use a **deterministic** table, not `A_Jump`. Punch-Out is about learnability; randomness destroys it. Vary by switching `patternIdx` on health thresholds (phase 2 at 60 %, phase 3 at 25 %).

**Weak windows.** Two per attack: during the telegraph (counter — `COUNTER_MULT`, §4.6) and during recovery (free hits — `PainChance = 255`, so every landed punch staggers). Detect from the player side with `InStateSequence(m.CurState, m.FindState('Telegraph'))` (`actor.zs:819`).

**Tells the player can actually read in VR.** Frame changes alone are not enough at VR scale and framerate. Use, in order of readability: a distinct *sound* 6 tics before the strike, a bright frame, a dynamic light, and a `VisualThinker` glint at the striking limb. Audio is the primary tell in a headset.

**Why retrofitting arbitrary monsters is a HACK/L:** adding a `Telegraph` label to a class you did not author requires replacing the class at spawn (`CheckReplacement`, `events.zs:205`) with a subclass that re-declares the whole state machine. That breaks every other mod touching the same monster and is not worth it. Build a small roster of your own opponents instead.

**Feel knobs:** `RING_RADIUS`, `RING_STRAFE`, telegraph length (the single most important number — 10-14 tics is readable, 6 is unfair, 20 is boring), recovery length, `PainChance` during windows, pattern length.

---

## 5. WALLS

| # | Wall | The exact missing piece | Severity |
|---|---|---|---|
| W1 | **No sub-tic hand history.** Script sees only the newest pose. Fast gestures, curvature, flicks and true swing arcs are lost between tics. | A C++-side ring buffer of per-frame `AttackPos/OffhandPos/angles` with a ZScript accessor (`GetHandSample(int tics_ago)` / `GetHandSampleCount()`). The data already exists at render rate — it is simply discarded each frame at `hw_vrmodes.cpp:442-445`. | High for gestures, **low for everything else** — the swept-chord trace (§3.5) already recovers correct hit testing. |
| W2 | **No direct haptic call from ZScript — still the highest-value engine addition for this topic.** `VRMode::Vibrate` (`hw_vrmodes.h:94`) has **no VM thunk** and there are zero hits for haptic/rumble/vibrate anywhere in `wadsrc/static/zscript`, even though the hardware path is real and working (`gl_openvr.h:121-124` → `gl_openvr.cpp:517-528` → `TriggerHapticPulse` `:546`). | A single `DEFINE_ACTION_FUNCTION` exposing `Vibrate(duration, hand, intensity)`. **This is a binding, not a feature** — the implementation already exists and already runs. | **Low–medium, not blocking.** The `A_QuakeEx` channel (§3.7) gives independent per-hand, per-tic, shapeable rumble today, so a jab *can* be told from a cross by feel. What the binding would buy: no dependence on a user cvar, no risk of camera shake for users who raised `r_quakeintensity`, sub-tic pulse lengths instead of a 28 ms floor, and no abuse of the earthquake system. |
| W2b | **`weaponStabilised` cannot be suppressed.** Grip + 0.50 m silently kills the offhand weapon's firing, buttons and rendering, and `WF_TWOHANDSTABILIZED` is re-asserted on every `P_SetPsprite` (`p_pspr.cpp:512-515`). | Either a `+Weapon.NoStabilise` flag, or gating the whole mechanism on `bTwoHanded`, or making the bit script-writable. | **High** if you want dual-wield blades or any offhand weapon used at guard distance. Zero if the fists-are-not-weapons design (§0.2(4)) is followed. |
| W3 | **No play-space yaw in script.** `playerYaw` (`src/QzDoom/qzdoom_common.cpp:27`, written `:89`) is not exposed. | A readonly `Actor.PlayspaceYaw` double. | Low. Costs you the clean separation of physical lean from stick input (§4.4). Workaround given. |
| W4 | **No knockdown / prone model in Doom actors.** No state, no flag, no size hook that survives a `SetState`. | Nothing in the engine; this is a content-model gap. | Medium. Solvable for your own classes (§4.7); not retrofittable. |
| W5 | **`+Weapon.TwoHanded` clears the other hand's weapon** (`player.zs:1980-1992`). You cannot have "two hands on one sword" expressed as two weapons. | A flag that means "occupies both hands visually but keeps both psprite layers". | Low — `WF_TWOHANDSTABILIZED` (`constants.zs:1242`) already gives you two-handed *grip detection* without the flag. Just don't use `+TwoHanded`. |
| W5b | **No player knockback without opt-in.** `A_ChangeVelocity` is a no-op on the player (`actions.zs:63-79`, `vr_recoil` default false) and `MovePlayer` erases `vel.XY` every grounded tic (`player.zs:1373-1383`, `vr_momentum` default false). | Nothing — this is deliberate anti-nausea design. | None. `Vel` + `player.keepmomentum = true` (§3.8) is the sanctioned route and works. |
| W6 | **Both hands collapse to the body centre on non-gameplay frames** (`hw_vrmodes.cpp:442-443` + the `gl_openvr.cpp:1704` guard). | A validity flag, or leaving the last-good pose in place. | Low — the spike filter in §3.2 handles it completely, but **every** consumer must have it. |
| W7 | **`r_quakeintensity` defaults to 0 in VR** (`r_utility.cpp:116`). Camera shake for hit feel is off. | Nothing — this is deliberate and correct (camera shake in VR is nausea). | None. Treat `A_QuakeEx` as a haptics API, not a camera API. Use psprite/model motion for visual impact instead. |
| W8 | **Could not find** any ZScript access to raw controller button state beyond the `BT_*` bitmask — no grip-button bit, no trigger *analogue* value, no thumbstick axes. The OpenVR grip button is consumed entirely inside `gl_openvr.cpp:1716-1722`. | Exposing the controller state struct, or at minimum an analogue trigger float per hand. | Medium for "squeeze to grip harder"; low otherwise, since `BT_OFFHANDATTACK`/`BT_USER1..4` cover binary needs. |
| W9 | **Could not find** any skeletal-hand / finger-tracking data anywhere in the tree. | OpenXR hand tracking extension. Not present. | None for this topic — fists and hilts do not need fingers. |

---

## 6. SMALLEST PROTOTYPE

**One pk3. One map. One afternoon. It proves the three things everything else rests on.**

### `RSQ_PunchProbe` — what it is

1. **A permanent token** (`RSQ_HandTrack : Inventory`, §3.2) given in `PlayerSpawned` (`events.zs:186`), carrying the 8-sample ring buffer, the spike filter and the warm-up gate for **both** hands.
2. **A HUD overlay** (`StaticEventHandler.RenderOverlay`) printing, per hand, live: `OverrideAttackPosDir`, `|vel| mu/tic`, `fwd/right/up` in the facing frame, the detector state (IDLE/ARMED/EXTENDED/RECOVER), and the last classification. **This is the deliverable** — the numbers it prints are what every constant in §4 gets tuned against.
3. **The §4.1 punch detector on both hands**, resolving via the §3.5 swept-chord `LineTrace` and §4.5 `LineAttack`.
4. **One test monster**: a Zombieman subclass with a `Telegraph` block, `freezetics` hit-stop, and `Speed 0` so it stands still.

### What it proves, in order

| Claim | How the probe proves it |
|---|---|
| **`AttackPos` is a real main-hand pose** (the §0 correction) | The HUD shows `AttackPos` and `OffhandPos` diverging and tracking independently. If they move together, §0 is wrong and everything above must be revised. **Check this first.** |
| The spike/collapse filter is necessary and sufficient | Open the menu, pause, drop the console, die and respawn. The HUD must show `valid=false` and **no** phantom punch. |
| Both hands punch | Jab with each hand; both register, with the correct `LAF_ISOFFHAND` so the rumble lands on the right controller. |
| Fast swings do not tunnel | Swing as hard as you can past the stationary zombieman at every height. **Zero misses.** Then replace `SweepHit` with a point test and watch it miss — keep that comparison, it is the argument for the chord trace. |
| Hit-stop reads | `freezetics = 4` on contact. It should feel like the punch landed, not like a frame drop. |
| Per-hand rumble is reachable **and shapeable** | Fire `A_QuakeEx(1, 0, 0, 6, 0, 512, damage: 0)` and confirm the **left** controller buzzes and the right does not. Then run the §3.7 envelope for a jab (peak 0.55, 3 tics, shape 2.5) and a cross (peak 1.0, 7 tics, shape 0.7) and confirm they feel different. Confirm **no camera shake** at default `r_quakeintensity`. |
| The two-hand trap is understood | Hold the offhand grip and bring the hands to the face. Confirm `WF_TWOHANDSTABILIZED` lights on the HUD — and confirm the punch detector keeps working anyway, because the fists are not weapons (§0.2(4)). |
| Z scaling is handled | Throw an uppercut and a straight of the same real length. With `pixelstretch` normalisation on, `D.Length()` should match within ~5 %. With it off, the uppercut reads ~17 % short. |
| Head/body split works | Punch high and low; the HUD prints `frac` from `HitLocation.Z`. |
| 35 Hz is or is not enough | The HUD's `|vel|` readout during a real hard punch tells you immediately whether `FULL_SPEED = 9.0` is in the right place for this player. |

**Explicitly out of scope for the prototype:** blades, grabs, combos, stance, patterns. None of them can be tuned until the sampler and the sweep are trusted.

**Gate it headlessly before you ever put the headset on** (`main_boottest.ps1` / the `gate-a-doom-mod` recipe): confirm the pk3 parses, the token is given, and `DoEffect` is reached. `OverrideAttackPosDir` will be false headless, so the detector will correctly sit at `valid=false` — **that is the pass condition**, not a failure, and the log line proving `DoEffect` ran is the thing to grep for.

---

## 7. OPEN QUESTIONS

1. **Which VR backend does the target binary actually use?** `USE_OPENXR` is undefined in this tree (`hw_vrmodes.cpp:264-267`, no definition anywhere), so only OpenVR/SteamVR is live here. If the shipping binary is OpenXR, the hand-pose writer lives outside this repository and §0's citations describe a code path that is compiled but unreachable. **Answer this before writing a line of code** — it decides whether `positional_movementForward/Sideways` (and therefore free physical dodging, §4.4) exists at all, since only `gl_openvr.cpp:1779-1785` writes them.
2. **Sign conventions for `fwd` / `right` / `roll`.** Derived in §3.3 and §4.9 but *not verified at runtime*. The double negation through `weaponoffset[0] → -X` plus the `v.Y`/`v.X` swap (`gl_openvr.cpp:1737-1740`) is exactly where a sign flips. One `Console.Printf` session settles it; do not ship on the derivation.
3. **Does `AttackRoll` carry usable wrist roll on this hardware,** or is it noisy/clamped? It is passed through raw (`hw_openxrdevice.cpp:591`), but the edge-vs-flat mechanic (§4.9) lives or dies on it. Measure the noise floor while holding still.
4. **How often does a real boxer accidentally trip `weaponStabilised`?** The trigger is the offhand *grip* button plus 0.50 m (`gl_openvr.cpp:1716-1728`). Guarding puts the hands close together; whether the player's grip finger is also closed depends entirely on their controller and their habit. If it happens often, `AttackAngle`/`AttackPitch` become the hand-to-hand line (`gl_openvr.cpp:1742-1754`) and `yawSweep` in §4.2 goes nonsense during guard-break counters. Measure it in the probe (§6), and if it is common, gate hook classification on `!(WeaponState & WF_TWOHANDSTABILIZED)` and fall back to lateral displacement alone.
   **Related, and worth asking the owner:** is `vr_two_handed_weapons` on in the target config? Turning it off removes the whole trap at the cost of two-handed aiming for guns.
5. **Real peak hand speeds for this player, in mu/tic.** Every constant in §4.1 is an estimate from the 1 mu/tic ≈ 1.03 m/s identity. The probe's HUD (§6) answers it in five minutes and the answer may move `FULL_SPEED` by 30 %.
6. **Does unclamped `crouchfactor` break anything else?** `Height = FullHeight * crouchfactor` with no lower bound (`player.zs:152` + `:1177-1180` clamp bypassed) means a player who sits down has a very short collision box. Check whether it ever goes `<= 0` (which would be a crash or a stuck-in-floor), and decide whether to floor it.
7. **Multiplayer.** None of this is netplay-safe as written: `AttackPos`/`OffhandPos` are per-client render state, so any detector running in playsim scope on a non-local player reads garbage. Gate every detector on `player == players[consoleplayer]` and route outcomes through `NetworkCommandProcess` (`events.zs:202`) if MP is ever in scope.
8. **Could not find** any existing GZDoom-side melee-arc or swept-hitbox helper to build on — `A_CustomPunch` (`wadsrc/static/zscript/actors/inventory/stateprovider.zs:320`) and `A_Saw` (`wadsrc/static/zscript/actors/doom/weaponchainsaw.zs:45`) are both single-ray and both aim from `self.Angle` with random spread. The sweep in §3.5 is new code, not a wrapper.
