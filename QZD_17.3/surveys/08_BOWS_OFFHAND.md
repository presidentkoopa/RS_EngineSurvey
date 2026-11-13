# 08 — Bows, crossbows and two-hand interaction verbs

**Engine:** tag `qzd17.3` of `github.com/emawind84/gzdoom`. All paths repo-relative at that tag.
**Scope:** what a *mod* (ZScript + MODELDEF, no engine edit) can build.
**Reads against:** `_BASELINE_FACTS.md` (as corrected: both hands are fully tracked).
This report was written on the two-hand premise throughout; §2.0 is the independent
verification of that correction, with the native-side citations.

---

## 1. Verdict

A real VR bow is **BUILDABLE on this engine, cost M**. Both hands are fully tracked and live in
the **same world coordinate space**: the main hand as `Actor.AttackPos` / `AttackAngle` /
`AttackPitch` / `AttackRoll` (`wadsrc/static/zscript/actors/actor.zs:268-271`, thunked at
`src/scripting/vmthunks_actors.cpp:2187-2190`, native `src/playsim/actor.h:1654-1657`), the
offhand as `OffhandPos` and friends. Draw length is therefore a straight vector subtraction
`AttackPos - OffhandPos` — a genuine measurement, not an inference. Everything downstream
(draw power, arc, release, the pump/rack/crank strokes, throw velocity, holster grabs) follows
from differencing that pair plus a per-tic position history in `Inventory.DoEffect()`. Nothing
in this report is labelled WALL or HACK for want of a second hand.

The offhand *weapon* system (`PlayerInfo.OffhandWeapon`, `PSP_OFFHANDWEAPON`, per-hand
`WF_OFFHAND*` flags, `BT_OFFHAND*` buttons, `CheckWeaponButtons(hand)`) is a full second
weapon state machine and is the right home for the arrow/grenade/flare hand. It is shipped
engine, not a hack.

`bTwoHanded` **fights a bow design and should not be used for one.** Its only effect is to
destroy the other hand's weapon and psprite (`wadsrc/static/zscript/actors/player/player.zs:1977-1993`),
which is exactly the hand a bow needs. Use two ordinary one-hand weapons that cooperate by
script instead.

The real walls are: **no haptics API — which hurts a bow more than anything else here**, since
rising draw tension is exactly what rumble is for and the only rumble available is the release
buzz you get free from firing a sounding projectile; no runtime-deformable geometry (the drawn
bowstring cannot be drawn, only frame-stepped); and `vr_two_handed_weapons` stabilisation,
which silently kills the offhand — fires nothing, checks no buttons, renders nothing — the
moment the player presses offhand grip within 0.50 m of the main hand, which is precisely where
a bow draw puts them.

---

## 2. What already exists

### 2.0 Both hands are fully tracked — verification of the baseline §1a correction

The main hand's pose is readable. It is simply not called `MainhandPos`:

| ZScript field | Declared | Thunk | Native | What it is |
|---|---|---|---|---|
| `Actor.AttackPos` (vector3) | `wadsrc/static/zscript/actors/actor.zs:268` | `src/scripting/vmthunks_actors.cpp:2187` | `src/playsim/actor.h:1654` | weapon-hand controller position, **map coords** |
| `Actor.AttackPitch` | `actor.zs:269` | `vmthunks_actors.cpp:2188` | `actor.h:1655` | weapon-hand pitch |
| `Actor.AttackRoll` | `actor.zs:270` | `vmthunks_actors.cpp:2189` | `actor.h:1657` | weapon-hand roll |
| `Actor.AttackAngle` | `actor.zs:271` | `vmthunks_actors.cpp:2190` | `actor.h:1656` | weapon-hand yaw |
| `Actor.AttackDir(actor,yaw,pitch)` | `actor.zs:858` | — | `actor.h:1659` (fn ptr) | weapon-hand basis, mirrors `OffhandDir` |
| `Actor.OverrideAttackPosDir` (bool) | `actor.zs:276` | `vmthunks_actors.cpp:2186` | `actor.h:1652` | true when these are live VR values |

They are written every rendered frame next to the offhand values, symmetrically:
`src/rendering/gl/stereo3d/gl_openvr.cpp:2525-2537` (main hand) and `:2541-2553` (offhand),
both taken from `GetWeaponTransform()`'s translation column, i.e. the actual rendered hand
in map space. `OverrideAttackPosDir` is set at `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:438`
and is the gate: **if it is false you are not in VR, and `hw_vrmodes.cpp:442-444` has set
`AttackPos == OffhandPos ==` the player's own shoot point** — both poses degenerate, every
measurement below meaningless. Check it first, always.

Baseline §1b (no velocity fields) and §3 (35 Hz script sampling) stand unchanged.

**Backend caveat, important.** On the PC build, only the OpenVR/SteamVR path populates the
pose globals (`gl_openvr.cpp:1733-1771`, `:2400-2401`). `hw_openxrdevice.cpp` reads
`weaponoffset`/`offhandoffset`/`hmdPosition` (`:593-608`) but nothing in this tree assigns
them, and that file is behind `#ifdef USE_OPENXR` which is defined only in the Android build
(`mobile/Android_src.mk:15`; the PC list is `src/CMakeLists.txt:981,992`). So on desktop the
numbers come from SteamVR; on Quest they come from the Android launcher glue, which is not in
this repo. Anything below that depends on `weaponStabilised` is **PC/OpenVR only** (§2.3).

### 2.1 The offhand weapon system

- `PlayerInfo.OffhandWeapon` — `wadsrc/static/zscript/actors/player/player.zs:3076`, native `src/playsim/d_player.h:384`
- `PSP_OFFHANDWEAPON = 1000000` — `wadsrc/static/zscript/constants.zs:800`, `player.zs:2939`
- `bOffhandWeapon` flagdef — `wadsrc/static/zscript/actors/inventory/weapons.zs:98`; **writable at runtime**, the engine itself writes it (`player.zs:2495`, `:2327`, `:2334`)
- `bTwoHanded` flagdef — `weapons.zs:100`; `bNoHandSwitch` — `weapons.zs:99`; `bNoAutoReverse` — `weapons.zs:101`
- Per-hand fire: `FireWeapon(state, hand)` `player.zs:389-414`, `FireWeaponAlt` `:422-448`, `CheckWeaponFire(hand)` `:465-491`
- Per-hand ready flags `WF_OFFHAND*` — `constants.zs:1230-1242`, `src/playsim/d_player.h:151-164`
- Per-hand button dispatch: `CheckWeaponButtons(hand)` — declared `player.zs:2861`, native `src/playsim/p_pspr.cpp:748-786`, table `p_pspr.cpp:100-116`
- Called every tic for both hands — `player.zs:582-588`
- Hand swap at runtime: `SwitchWeaponHand(hand)` — `player.zs:2488-2503`

### 2.2 Per-hand buttons

`src/d_event.h:41-78` / `wadsrc/static/zscript/constants.zs:832-870`:
`BT_OFFHANDATTACK` (1<<26), `BT_OFFHANDALTATTACK` (1<<27), `BT_OFFHANDRELOAD` (1<<28),
`BT_MAINHANDRELOAD` (1<<20, which replaced `BT_SHOWSCORES`), plus the shared
`BT_RELOAD`, `BT_ZOOM`, `BT_USER1..4`, `BT_USE`.
Bindable names: `+oh_attack`, `+oh_altatk`, `+oh_reload`, `+mh_reload` — `src/d_main.cpp:2726,2740,2747,2756`;
cmd assembly at `src/g_game.cpp:766-781`.
Script reads them from `player.cmd.buttons` (`player.zs:3134`, `UserCmd.buttons` `player.zs:3`)
with edge detection against `player.oldbuttons` (`player.zs:3064`); `player.ohattackdown`
(`player.zs:3062`) is the engine's own offhand-held latch.

`CheckWeaponButtons` reaches **only** the state labels `Zoom`, `Reload`, `User1`..`User4`
(`p_pspr.cpp:100-116`) and only if the matching `WRF_Allow*` was passed to `A_WeaponReady`
(`wadsrc/static/zscript/actors/inventory/weapons.zs:440-451, 456-465`). Six scriptable
"verbs" per hand, per weapon. That is the whole budget.

### 2.3 Two-handed stabilisation (PC/OpenVR only)

`weaponStabilised` is a C++ global (`src/QzDoom/qzdoom_common.cpp:19`), set when the player
presses offhand grip while the controllers are **within 0.50 m** of each other, with
`vr_two_handed_weapons` on — `src/rendering/gl/stereo3d/gl_openvr.cpp:1716-1727`.
While it is on:

1. **Main-hand aim is recomputed as the vector between the two controllers** —
   `gl_openvr.cpp:1742-1755`. Pitch `-atan(y/zxDist)`, yaw `-atan2(x,-z)`, roll preserved.
2. `WF_TWOHANDSTABILIZED` is forced into `player.WeaponState` on every psprite state set —
   `src/playsim/p_pspr.cpp:512-515`, flag at `d_player.h:164` / `constants.zs:1242`.
3. The offhand **cannot fire** (`player.zs:393-396`, `:426-429`), **cannot be made ready**
   (`weapons.zs:414-417`), and **its buttons are not checked** (`p_pspr.cpp:755-758`).
4. The offhand's psprite is **not rendered at all**, sprite path `src/rendering/hwrenderer/scene/hw_weapon.cpp:832-835`,
   model path `hw_weapon.cpp:917-920`.
5. Firing anything damaging buzzes **both** controllers — `src/playsim/p_map.cpp:4771-4775`,
   `:5700-5704`, `src/playsim/p_mobj.cpp:7557-7561`.

`player.WeaponState` is a plain writable `native uint` (`player.zs:3073`), so a mod *can* set
or clear `WF_TWOHANDSTABILIZED` itself — but `DPSprite::SetState` re-applies it from the C++
global on the next state change (`p_pspr.cpp:512`), so clearing it is not durable.

### 2.4 Per-hand spawn origin and direction, already wired through the action functions

Every player attack function already branches on which hand the invoking weapon is in and uses
that hand's origin and basis:

- `A_FireProjectile` — `wadsrc/static/zscript/actors/inventory/stateprovider.zs:239-307`
  (offhand flag at `:245-248`, direction fix-up at `:285-299`)
- `P_SpawnPlayerMissile` — `src/playsim/p_mobj.cpp:7480-7522` (`AttackPos`/`AttackDir` vs offhand)
- `P_LineAttack` — `src/playsim/p_map.cpp:4782-4800`
- `P_RailAttack` — `src/playsim/p_map.cpp:5670-5686`
- `A_SpawnItemEx` with `SXF_RELATIVETOWEAPON` — `wadsrc/static/zscript/actors/attacks.zs:467-500`, constant `constants.zs:130`
- `A_ThrowGrenade` — `attacks.zs:568-615`
- `A_CustomRailgun`-family with `CVF_RELATIVETOWEAPON` — `wadsrc/static/zscript/actors/actions.zs:79-96`, constant `constants.zs:193`
- `OffhandDir` / `AttackDir` — `actor.zs:858-859`, mapped at `hw_vrmodes.cpp:395-424`, installed `hw_vrmodes.cpp:439-440`
- `P_LineTrace` with `TRF_USEWEAPON` / `TRF_ISOFFHAND` — `src/playsim/p_map.cpp:5185-5200`

All of them are gated on `player.mo.OverrideAttackPosDir`.

### 2.5 Rendering, sound, haptics

- Overlay layers follow the hand of their **Caller**, not their layer number:
  `src/r_data/models.cpp:220-222` picks `hand = (psp->GetCaller() == OffhandWeapon)`, and
  `GetWeaponPosition3D` resolves the bob layer the same way (`hw_weapon.cpp:328-332`).
  So `A_Overlay` from the offhand weapon draws **on the offhand**.
- `CHAN_OFFWEAPON = 5` — `wadsrc/static/zscript/engine/base.zs:42`, `src/common/audio/sound/s_soundinternal.h:156`;
  auto-selected for offhand missile spawn sounds at `p_mobj.cpp:7528`.
- **Haptics are still not a ZScript API** (baseline §4 confirmed), and the two C++ haptic
  calls are *not* equivalent — this matters:
  - `VR_HapticEvent(...)` is an **empty stub** on this build — `src/QzDoom/qzdoom_common.cpp:65-67`.
    Every call site in `src/` does nothing. It is the Quest-side external-haptics bridge.
  - `VRMode::Vibrate(duration, channel, intensity)` is **real on PC/OpenVR** —
    `src/rendering/gl/stereo3d/gl_openvr.h:121-124` → `OpenVRHaptics::Vibrate`
    `gl_openvr.cpp:517-528`, pumped by `ProcessHaptics` `:530+`, gated on `vr_enable_haptics`.
    The base class is a no-op (`hw_vrmodes.h:94`).
  So the *only* mod-reachable rumble is the side effect of firing, and it works:
  `P_SpawnPlayerMissile` calls `Vibrate(150, firingHand, 0.8)` whenever the missile has damage,
  a damage type, or a SeeSound — `src/playsim/p_mobj.cpp:7545-7563`. Same for `P_LineAttack`
  (`p_map.cpp:4765-4776`) and `P_RailAttack` (`p_map.cpp:5692-5705`). All three also buzz the
  *second* hand when `weaponStabilised`.
- `LineTrace` with `TRF_USEWEAPON` substitutes the real controller origin and basis, and
  `TRF_ISOFFHAND` selects the other hand — `src/playsim/p_map.cpp:5185-5200`, constants
  `wadsrc/static/zscript/constants.zs:962-963`. Also gated on `OverrideAttackPosDir`. This is
  the honest way to ask "what is this hand pointing at / touching".
- `+SKULLFLY` gives a **free impact callback**: `PIT_CheckThing` calls `CallSlam` on the moving
  actor — `src/playsim/p_map.cpp:1553-1559`, ZScript virtual `Slam(Actor)` at
  `wadsrc/static/zscript/actors/actor.zs:520`. A thrown grenade, dynamite stick or arrow gets
  its hit hook for nothing.
- **Items do not auto-pick-up when moved to the player.** Pickup requires the *mover* to carry
  `MF_PICKUP` — `p_map.cpp:1737-1743`. A hand is not an actor and carries no flags, so a
  holster/grab design must call `Inventory.Touch` / `CallTryPickup` itself.
- Per-tic polling home: `Inventory.DoEffect()` — declared `wadsrc/static/zscript/actors/inventory/inventory.zs:982`,
  called once per tic per carried item from the owner's Tick, skipped under `CF_PREDICTING`
  — `src/playsim/p_mobj.cpp:4005-4018`.
- Scale constants a mod needs: `vr_vunits_per_meter` default **34.0** and `vr_height_adjust`
  default 0 — `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:94-95`; both readable via
  `CVar.FindCVar()` (`wadsrc/static/zscript/engine/base.zs:696, 700`). Vertical stretch is
  `level.pixelstretch` — `wadsrc/static/zscript/doombase.zs:479`.

### 2.6 What does not exist

- **No bow, nowhere.** The only draw-and-loose weapon in the tree is Heretic's
  `Crossbow` (`wadsrc/static/zscript/actors/heretic/weaponcrossbow.zs:3`), which is a plain
  hitscan-era weapon with no draw mechanic.
- **No holster, pouch, body-slot or grab system.** `grep -ri holster` over `src/` and
  `wadsrc/` returns nothing.
- No hand velocity, no finger/trigger analog value, no grip-button state exposed to script
  (the grip is consumed in C++ at `gl_openvr.cpp:1718-1726`).

---

## 3. Building blocks

These five expressions are the whole toolkit. Everything in §4 is built from them.

**B1 — Hand separation, in metres.** Both hands are in map space; X/Y scale by
`vr_vunits_per_meter`, Z additionally by `1/pixelstretch` (`gl_openvr.cpp:1282-1292`).

```
double vpm = CVar.FindCVar('vr_vunits_per_meter').GetFloat();   // 34.0 default
double ps  = level.pixelstretch;                                // 1.2 typical
Vector3 d  = mo.AttackPos - mo.OffhandPos;
double  mx = d.X / vpm, my = d.Y / vpm, mz = d.Z * ps / vpm;
double  sepMetres = sqrt(mx*mx + my*my + mz*mz);
```
Guard every use with `if (!mo.OverrideAttackPosDir) return;` (`actor.zs:276`).

**B2 — Hand-to-hand direction, for firing.** Do **not** metric-convert here; the arrow flies
in map space and is rendered with the same stretch the hand transform used.

```
Vector3 shot = mo.OffhandPos - mo.AttackPos;      // bow hand minus draw hand
double ang = atan2(shot.Y, shot.X);
double pit = -atan2(shot.Z, shot.XY.Length());    // Vel3DFromAngle: Vel.Z = -sin(pitch)
arrow.Vel3DFromAngle(speed, ang, pit);            // actor.zs:861, src/playsim/actor.h:1620-1626
```

**B3 — Stroke along the weapon axis.** For pump/rack/crank, project the offhand onto the main
hand's forward axis. `AttackAngle`/`AttackPitch` (`actor.zs:269,271`) give that axis directly:

```
Vector3 fwd = (cos(mo.AttackAngle)*cos(mo.AttackPitch),
               sin(mo.AttackAngle)*cos(mo.AttackPitch),
              -sin(mo.AttackPitch));
double stroke = (mo.OffhandPos - mo.AttackPos) dot fwd;   // map units along the barrel
```

**B4 — Velocity by differencing, 35 Hz.** No velocity fields exist (baseline §1b). Keep a ring
of the last 3-4 samples of `OffhandPos` in `DoEffect()` and difference; a release is 2-4
samples, so use the **peak** of the ring, never the last delta alone (baseline §3).

**B5 — Body-relative hand position, for holsters.** The hand transform is built from
`CenterEyePos` minus the player's standing height (`gl_openvr.cpp:1282`), i.e. from a point at
the player's feet. So `OffhandPos - mo.Pos` is hand-relative-to-body, in map units, and
`(OffhandPos.Z - mo.Pos.Z) * ps / vpm` is hand height above the floor in metres. Rotate the
horizontal part by `-mo.Angle` to get left/right/forward in body frame. The error term is the
roomscale decoupling between `CenterEyePos.XY` and `mo.Pos.XY`; it is small and does not drift.

**B6 — "What is this hand touching?"** `LineTrace(angle, dist, pitch, TRF_USEWEAPON | TRF_ISOFFHAND, ...)`
substitutes the real controller origin and basis for that hand (`src/playsim/p_map.cpp:5185-5200`,
constants `wadsrc/static/zscript/constants.zs:962-963`). A 0.15 m trace is a contact test; a
long one is a pointing test. This is the only surface query a hand has, and like everything
else it is gated on `OverrideAttackPosDir`.

---

## 4. Mechanic designs

### 4.1 Bow — nock, draw, release — **BUILDABLE, cost M**

**Layout, and this ordering matters.** Bow = **offhand weapon** (`bOffhandWeapon`, held out
front in the non-dominant hand). Draw hand = **main hand**, carrying a one-state "Arrow"
weapon whose psprite is the nocked shaft. This is the right way round for two reasons: the
arrow flies from the bow toward the target, which is `OffhandPos - AttackPos` (B2, positive
sense); and if the player ever does grip-stabilise, the engine's own recomputed aim
(`gl_openvr.cpp:1746-1753`, dominant → offhand) is already the correct shot vector.

**Nock.** Player brings the hands together and presses main-hand trigger (`BT_ATTACK`).
Script detects `sepMetres < 0.25` (B1) plus a fresh `BT_ATTACK` edge against `oldbuttons`.
On success the Arrow weapon enters a `Nocked` state loop and consumes one from an `Arrow`
ammo class.

**Draw.** In the Arrow weapon's `Nocked` loop (one tic per frame, `A_WeaponReady(WRF_NOFIRE|WRF_NOBOB)`),
sample B1 each tic. `drawFrac = clamp((sep - nockSep) / (fullSep - nockSep), 0, 1)` with
`nockSep ≈ 0.20 m`, `fullSep ≈ 0.70 m` (the owner is 5'9", so ~0.65-0.72 m is a real anchor
draw; see `user_owner-height-floor-1016`). Feed `drawFrac` to `A_OverlayOffset`
(`actor.zs:1330`) on the bow's own overlay layer to slide the string/limb frames, and to
`A_OverlayScale`/`A_OverlayRotate` if the model needs it.

**Release.** `BT_ATTACK` released (edge on `oldbuttons`) → jump to `Fire`. Spawn the arrow
**yourself**, not via `A_FireProjectile`: that function's direction comes from
`AttackDir`/`OffhandDir` (`stateprovider.zs:285-299`), which is the *hand's* forward axis, not
the hand-to-hand line. Use `Spawn()` at `mo.AttackPos` then B2 with
`speed = minSpeed + drawFrac*(maxSpeed-minSpeed)`.

**Arc and gravity.** The arrow is an ordinary `+MISSILE` actor **without** `+NOGRAVITY`, with
`Gravity` tuned (`actor.zs:176`). That gives a real ballistic arc with no extra code. Face the
model to flight with `A_SetPitch(PitchFromVel())` in the arrow's Spawn loop (`actor.zs:860`).
Flat-ish at full draw, visibly lobbed at half draw — the arc *is* the draw-power feedback.

**Haptic.** Give the arrow a `SeeSound` or non-zero damage and the release buzzes the firing
hand automatically (`p_mobj.cpp:7545-7563`). That is the only rumble available.

**Feel knobs:** `nockSep`, `fullSep`, `minSpeed`/`maxSpeed`, `Gravity`, a draw-rate cap (clamp
`drawFrac` delta per tic so a yank doesn't instantly full-draw), creak-sound pitch bound to
`drawFrac` via `A_StartSound(..., pitch:)`, a hold-fatigue timer that bleeds `drawFrac` after
N tics, and a dry-fire threshold below which release does nothing.

**Residual risk:** `OverrideAttackPosDir` false (desktop/purist) must fall back to a
button-timed draw.

### 4.2 Crossbow — crank/lever reload, bolt types — **BUILDABLE, cost S-M**

**Physically.** Crossbow in the main hand. To reload, the offhand grabs the lever/windlass at
the front of the stock and pulls back along the weapon axis.

**Detection.** B3. The crank stroke is `stroke` going from near 0 (offhand at the muzzle end)
to a negative threshold (offhand pulled back toward the shoulder) while offhand trigger
(`BT_OFFHANDATTACK`) is held. Count a stroke when `stroke` crosses `-0.35 m` worth of map
units having been above `-0.10 m`, with hysteresis. A windlass needs 2-3 strokes; a goat's-foot
lever one. Each completed stroke advances a counter on the weapon and plays a ratchet sound on
`CHAN_OFFWEAPON`.

**Resolution.** The crossbow sits in a `Reload` state loop entered through `BT_MAINHANDRELOAD`
(`p_pspr.cpp:108`, requires `WRF_AllowReload` on the `A_WeaponReady`). When the stroke counter
hits the required number, `A_TakeInventory`/`A_GiveInventory` the bolt and return to `Ready`.

**Bolt types.** Ordinary ammo classes, one per bolt. Selection = a `User1` state on the
crossbow (`BT_USER1` → `WF_USER1OK` → `NAME_User1`, `p_pspr.cpp:104`) cycling the chambered
type, or an offhand "quiver" weapon whose `AltFire` cycles it.

**Feel knobs:** stroke length, stroke count, per-stroke resistance sound pitch, a partial-draw
abort (release offhand trigger mid-stroke → counter decays), bolt-specific `Speed`/`Gravity`.

### 4.3 Offhand interaction verbs

| Verb | Player motion | What script can actually detect | Label | Cost |
|---|---|---|---|---|
| Pull a grenade pin | Offhand comes to the grenade (main hand), trigger, yank away | Separation gate B1 (`sep < 0.20 m`) + `BT_OFFHANDATTACK` edge + then `sep` rising past 0.30 m within N tics. The "yank" is a separation *rate*, from B4. | BUILDABLE | S |
| Light dynamite with a lighter | Offhand holds lighter, brings flame to the fuse in the main hand, holds it there | B1 proximity (`sep < 0.15 m`) sustained for K tics while `BT_OFFHANDATTACK` held. **Dwell, not gesture** — dwell survives 35 Hz; a flick does not. | BUILDABLE | S |
| Strike a flare / match | Scrape the offhand across a surface | **Offhand speed** (B4 peak over the ring) over a threshold, **plus a real surface test**: `LineTrace` with `TRF_USEWEAPON\|TRF_ISOFFHAND` (`p_map.cpp:5185-5200`, `constants.zs:962-963`) traces from the true offhand origin along the true offhand direction, so "is there a wall within 0.15 m of my hand" is answerable. Speed test alone would be a HACK; speed + short trace is a genuine strike-against-something. | BUILDABLE | S |
| Manual reload (mag in) | Offhand brings a magazine up to the gun's well | B3 + B1: offhand within `0.15 m` of a well point offset along `fwd` from `AttackPos`, with `BT_OFFHANDRELOAD` or a sustained dwell. This is the same shape as the existing well/grip work. | BUILDABLE | M |
| Pump a shotgun | Offhand slides fore-end back and forward along the barrel | B3 exactly. Back stroke past `-0.18 m`, forward stroke back past `-0.05 m` → one cycle. Both edges required, so a single jab does not fire it. | BUILDABLE | S |
| Rack a slide | Offhand pinches the slide and pulls straight back | B3 with a shorter throw (`-0.06 m`) and a tighter lateral tolerance: reject if the perpendicular component of `(OffhandPos - AttackPos)` exceeds ~`0.08 m`, so only an on-axis pull counts. At 35 Hz a fast rack is 2-3 samples — detect by **extremum reached**, never by seeing the whole path. | BUILDABLE | S |

Every one of these resolves the same way: an `Inventory.DoEffect()` ticker
(`inventory.zs:982`, `p_mobj.cpp:4005-4018`) owns the sampling and sets a flag; the weapon's
state machine reads the flag from a `Ready`/`Reload` loop and jumps. Do **not** try to sample
from inside a one-tic weapon state — you will miss frames whenever the state has `Tics > 1`.

**Common wall for all six:** none of them can be *felt*. No haptics API (baseline §4,
confirmed §2.5). The best available substitute is firing a zero-damage, `SeeSound`-bearing
dummy missile from the acting hand to provoke `p_mobj.cpp:7545-7563`. That is a HACK and it
costs a spawned actor per buzz.

### 4.4 `bTwoHanded` versus a bow — **it fights it. Do not use it.** — WALL (design-level)

`bTwoHanded`'s *entire* behaviour in this engine is two things:

1. Readying it nulls the other hand's weapon and psprite — `player.zs:1977-1993`
   (offhand case `:1980-1984`, main-hand case `:1989-1993`).
2. It blocks the paired hand-swap in `SwitchWeaponHand` — `player.zs:2498`.

It does **not** enable stabilisation, does not change rendering, does not change aim, and is
not read by any C++ VR path (`grep` over `src/` finds it only in `d_player.h`'s flag list and
`a_weapons.h:165`'s neighbour).

A bow needs **both** hands to carry a live, independently-stated psprite: the bow limb on one,
the nocked arrow on the other. `bTwoHanded` is a tool for the opposite case — a rifle that
wants the other hand emptied. Putting it on a bow deletes the arrow hand on the first
`BringUpWeapon`. **Plainly: `bTwoHanded` is a liability here; a bow should be two ordinary
one-hand weapons that cooperate through a shared `Inventory` ticker.**

Separately, and more dangerously: `vr_two_handed_weapons` stabilisation (§2.3) is *not* gated
on `bTwoHanded` at all. Any weapon, bow included, loses its offhand the instant the player
presses offhand grip within 0.50 m of the main hand — offhand fires nothing
(`player.zs:393-396`), checks no buttons (`p_pspr.cpp:755-758`) and renders nothing
(`hw_weapon.cpp:832-835, 917-920`). Drawing a bow puts the hands *exactly* in that window. A
bow mod must therefore either (a) ask the player to set `vr_two_handed_weapons 0`, or (b)
detect `player.WeaponState & WF_TWOHANDSTABILIZED` (`constants.zs:1242`) each tic and abort the
draw with an explicit message rather than silently doing nothing. **(b) is mandatory either way** —
a mod that fails here looks broken and gives no clue why.

### 4.5 Grenades: fuse, cooking, throwing with real arm motion — **BUILDABLE, cost M**

**Fuse.** A counter on the grenade *item* ticked in `DoEffect()` (`inventory.zs:982`), or
`ReactionTime` + `A_CountdownArg` (`actor.zs:155`, `:1252`) once it has left the hand. Keep the
pre-throw fuse on the inventory side so it survives weapon state changes.

**Cooking.** Pin pulled (§4.3) starts the counter while the grenade is still a psprite in the
offhand. Audio pip per second on `CHAN_OFFWEAPON` (`base.zs:42`). If the counter expires in
hand, detonate at `OffhandPos`.

**Throwing with real arm motion.** B4 on `OffhandPos`. On `BT_OFFHANDATTACK` release:
- direction = the normalised peak-velocity delta from the ring, **not** `OffhandDir` — a thrown
  object goes where the hand was moving, not where the hand is pointing;
- speed = `|peak delta| * tunable`, clamped. Use **best-of-three 2-tic deltas** measured at
  release — the convention the grab survey settled on, and the right one here: at 35 Hz one tic
  is 28.6 ms, so a brisk throw is 3-4 usable samples and the *last* single delta always
  under-reads (the hand is already decelerating into the release). A 2-tic baseline also halves
  the sampling jitter.
- Spawn with `Spawn()` at `OffhandPos` and set `Vel` directly. `A_ThrowGrenade`
  (`attacks.zs:568-615`) is *not* the right tool: it does honour the offhand origin
  (`:595-597`) but it derives direction from `OffhandDir` and the player's own pitch
  (`:596`), which is the pointing direction, not the swing.
- Gravity: ordinary, `+MISSILE` without `+NOGRAVITY`, plus a bounce flag.
- **Impact hook for free:** give the grenade `+SKULLFLY` and `PIT_CheckThing` calls your
  `Slam(Actor)` override on contact — `src/playsim/p_map.cpp:1553-1559`, virtual at
  `wadsrc/static/zscript/actors/actor.zs:520`. Detonate-on-contact, stick-on-contact and
  bounce-off-monster all hang off that one callback with no extra machinery.

**Feel knobs:** velocity scale, clamp ceiling, ring length (3 vs 4 samples trades latency for
noise), a minimum speed below which it is a drop not a throw, spin rate bound to speed.

### 4.6 Holsters and body slots — **BUILDABLE, cost M**

Nothing like this exists in the engine (§2.6) — it is entirely a mod construct, and B5 is all
it needs.

**Where items live.** Define each slot as a point in body frame: `(right, forward, up)` in
metres, converted to map units by `vpm` horizontally and `vpm/ps` vertically, with the origin
at `mo.Pos` (feet) and the horizontal axes rotated by `mo.Angle`. For the owner at 5'9"
(eyes ~1.64 m, `user_owner-height-floor-1016`): right hip ≈ `(+0.20, -0.05, 0.95)`, left hip ≈
`(-0.20, -0.05, 0.95)`, chest rig ≈ `(0, +0.15, 1.30)`, over-shoulder quiver ≈ `(-0.18, -0.18, 1.45)`.
Anchor these to the **floor**, not to `HmdPosition`: `HmdPosition` (`actor.zs:277`,
`hw_vrmodes.cpp:445`) is raw tracking-space metres, a different space from `OffhandPos`, and
the two cannot be subtracted. Use it only for the player's standing height
(`HmdPosition.Y`) when scaling slot heights to the player.

**Grab detection.** In `DoEffect()`: for each slot, `distance(OffhandPos, slotPoint) < radius`
(0.18 m is forgiving without overlapping neighbours) **and** a `BT_OFFHANDATTACK` press edge.
On match, set `weap.bOffhandWeapon = true` (writable — the engine does it at `player.zs:2495`),
`player.PendingWeapon = weap`, `player.mo.BringUpWeapon()` (`player.zs:1926`). Main hand:
same with `BT_ATTACK` and `AttackPos`, `bOffhandWeapon = false`.

**Holstering** is the same test with the hand already full: nearest slot within radius +
button edge → `player.PendingWeapon` = the empty-hand placeholder weapon.

**Feel knobs:** per-slot radius, a re-grab cooldown (so one press doesn't fire twice),
a proximity click on `CHAN_OFFWEAPON` when a hand enters a slot (the only feedback available,
since no haptics), and whether slots are body-locked or view-locked.

**Picking a world item off the floor with a hand.** The same proximity test works, but the
pickup will **not** happen by itself: `P_TouchSpecialThing` fires only when the *moving* actor
carries `MF_PICKUP` (`src/playsim/p_map.cpp:1737-1743`), and a hand is not an actor at all. So
a grab must find the item (`BlockThingsIterator` around `OffhandPos`) and call
`Inventory.CallTryPickup` / `Touch` on it directly. Do not build a design that assumes
"move an invisible grabber actor onto the item and it gets picked up".

**Caveat to flag up front:** `SwitchWeaponHand` (`player.zs:2488-2503`) already does a
two-weapon swap, and it refuses `bNoHandSwitch` weapons and bails if either weapon is
`bTwoHanded` (`:2498`). A holster system should call it where it fits rather than reimplement it.

---

## 5. Walls

| # | Wall | Exact missing piece |
|---|---|---|
| W1 | **No haptics from script — and a bow is the worst case for it.** | `VRMode::Vibrate` (`hw_vrmodes.h:94`, really implemented at `gl_openvr.cpp:517-528`) has no `DEFINE_ACTION_FUNCTION`; `VR_HapticEvent` (`hw_vrmodes.h:97`) is an empty stub anyway (`qzdoom_common.cpp:65-67`). Zero hits for haptic/rumble/vibrate in `wadsrc/static/zscript`. Reachable **only** as a side effect of a damaging/sounding attack (`p_mobj.cpp:7545-7563`, `p_map.cpp:4765-4776`, `:5692-5705`) — a release buzz is free, but **rising draw tension cannot be felt at all**, and that is most of what sells a bow. Missing piece: one thunk exposing `Vibrate(duration, hand, intensity)`. Mod-side substitute is audio only: a creak loop whose pitch tracks `drawFrac`. |
| W2 | **The bowstring cannot be drawn.** | There is no runtime vertex/bone write from ZScript. `A_OverlayVertexOffset` (`p_pspr.cpp:817-836`) moves the four corners of a *2D* psprite quad only. A 3D string that stretches between the hands needs per-frame geometry, which does not exist. Workaround: a frame-stepped bow model (N draw frames), which is what every shipping VR bow does anyway. |
| W3 | **Stabilisation silently disables the offhand, and a mod cannot opt out.** | `weaponStabilised` is a C++ global (`qzdoom_common.cpp:19`) driven only by `gl_openvr.cpp:1716-1727`; `WF_TWOHANDSTABILIZED` is re-asserted on every psprite state set (`p_pspr.cpp:512-515`), so script cannot durably clear it. Missing piece: a per-weapon "never stabilise" flag, or exposing `weaponStabilised` as a settable bool. Mod-side mitigation only: detect and warn. |
| W4 | **Six verbs per hand, hard cap.** | `ButtonChecks` (`p_pspr.cpp:100-116`) is a fixed table: Zoom, Reload, User1-4. No seventh. A rich bow (nock / draw / aim-assist / quiver-cycle / unnock / stow) runs out. Workaround: dispatch within one `User1` state from script-read `cmd.buttons` and gesture state. |
| W5 | **No grip-button state in script.** | The offhand grip is consumed entirely in C++ (`gl_openvr.cpp:1718-1726`) and never reaches `cmd.buttons` (`g_game.cpp:766-781` has no grip entry). A "squeeze to hold the string" design is not available; use trigger. |
| W6 | **OpenXR path does not populate hand poses in this tree.** | `hw_openxrdevice.cpp:593-608` reads `weaponoffset`/`offhandoffset`/`hmdPosition`, which are defined at `qzdoom_common.cpp:11-16` and assigned **only** by `gl_openvr.cpp:1733-1771`. The file is behind `#ifdef USE_OPENXR`, defined only in `mobile/Android_src.mk:15`. On Quest the values come from launcher glue outside this repo — unverifiable here. Anything in §2.3 is PC/OpenVR only. |
| W7 | **35 Hz, and nothing between samples.** | Baseline §3. Not fixable in a mod. Every gesture above is written as a threshold-and-extremum test for this reason; no design in this report depends on seeing a path. |

---

## 6. Smallest prototype

One pk3, one evening, proves or kills the whole topic. **Do not build the bow first.**

**Step 1 — prove the measurement.** A single `Inventory` subclass, given to the player, whose
`DoEffect()` prints once every 7 tics:
`OverrideAttackPosDir`, `AttackPos`, `OffhandPos`, `sepMetres` (B1), `stroke` (B3), and
`WeaponState & WF_TWOHANDSTABILIZED`. Launch headless through `main_boottest.ps1` per the
standing rule, read the log in `Documents/My Games/DoomXR/logs`.
**Two kill conditions: if `OverrideAttackPosDir` is false, or if `AttackPos == OffhandPos`
(the non-VR fallback at `hw_vrmodes.cpp:442-444`), stop — everything in §4 is void and the
answer is W6.** Printing both positions rather than only the separation is what distinguishes
"the hands are not tracked" from "the hands are tracked and the player is standing still".

**Step 2 — prove one verb.** Add the shotgun pump (§4.3): a `Ready` loop that counts B3 strokes
and prints a line per completed cycle. Cheapest honest gesture in the report; if it works, the
crank, the rack and the mag-in are the same code with different constants.

**Step 3 — prove the bow skeleton.** Two one-hand weapons, bow on `bOffhandWeapon`, arrow in
the main hand, nock by proximity, draw printed as `drawFrac`, release spawning a gravity
arrow with B2. Sprite placeholders, no model work. At this point draw power, arc and release
are all observable in one shot.

**Nothing in steps 1-3 touches `bTwoHanded`, and nothing needs an engine change.**

---

## 7. Open questions

1. **Does the Quest/Android build populate `AttackPos`/`OffhandPos` at all?** The writers are
   not in this repo (W6). This has to be answered by instrumenting on-device, not by reading.
   Every mechanic in §4 depends on it.
2. **Is `vr_two_handed_weapons` on in the owner's config?** If it is, §4.1 must ship the W3
   mitigation from day one, and the bow's nock distance has to sit outside the 0.50 m grip
   window or the player will stabilise by accident mid-draw.
3. **What is the owner's actual full-draw separation?** `fullSep` is the single most
   feel-critical constant in §4.1 and it is a measurement, not a guess. Step 1 of §6 yields it
   directly — have him stand at anchor and read the log.
4. **Is the "arrow hand" allowed to be a weapon?** It occupies `ReadyWeapon`, so while the bow
   is out the player cannot hold a gun. That is almost certainly the intent, but it is a
   product decision, not an engine one.
5. **Does `vr_control_scheme >= 10` (left-handed) invert anything here?** `GetWeaponTransform`
   swaps which physical controller is "main" (`hw_vrmodes.cpp:463-465`), so `AttackPos` follows
   the dominant hand and B2's sense should hold — but this is reasoned, not tested.
6. **Is a zero-damage `SeeSound` missile really enough to trigger the release buzz?** The
   condition at `p_mobj.cpp:7545-7547` reads as yes (`SeeSound != NO_SOUND` alone suffices) and
   `VRMode::Vibrate` is genuinely implemented on PC (`gl_openvr.cpp:517-528`), but the buzz also
   requires `P_CheckMissileSpawn` to succeed at the hand position, which may fail when the hand
   is inside geometry, and `vr_enable_haptics` must be on. Worth a bench test before any design
   leans on it.
7. **Is W1 worth escalating to the owner as an engine ask?** A single `DEFINE_ACTION_FUNCTION`
   over `VRMode::Vibrate` would make draw tension, crank resistance and slide detent all
   feelable, and would serve every mod in the family, not just a bow. It is the one engine
   addition this topic genuinely wants — but UZDXREMA is a different tree, and on *this* frozen
   engine it stays a wall. Flagging it, not proposing it.
