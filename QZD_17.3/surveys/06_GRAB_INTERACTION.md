# 06 — Grab & Physical Interaction (gravity gloves, object manipulation)

**Engine surveyed:** tag `qzd17.3` of `github.com/emawind84/gzdoom` (QuestZDoom PC edition).
All `path:line` citations are repo-relative at that tag. Read `_BASELINE_FACTS.md` first.

---

## 0. CORRECTION TO THE BASELINE — read this before anything else

`_BASELINE_FACTS.md` §1a says there is **no main-hand pose field in ZScript**. That is wrong at
this tag, and the correction changes most of the designs below.

The main hand **is** readable from ZScript, as four `readonly` fields next to the offhand ones:

```
wadsrc/static/zscript/actors/actor.zs:268   native readonly vector3 AttackPos;
wadsrc/static/zscript/actors/actor.zs:269   native readonly double  AttackPitch;
wadsrc/static/zscript/actors/actor.zs:270   native readonly double  AttackRoll;
wadsrc/static/zscript/actors/actor.zs:271   native readonly double  AttackAngle;
wadsrc/static/zscript/actors/actor.zs:276   native readonly bool    OverrideAttackPosDir;
```

Native storage: `src/playsim/actor.h:1652-1657`. Thunks: `src/scripting/vmthunks_actors.cpp:2186-2190`.
Written per rendered frame from the controller, in world map units:
`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:592-596` (main hand),
`:606-610` (offhand); non-VR fallback `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:441-444`.

`OverrideAttackPosDir` (`hw_vrmodes.cpp:437`) is the "these fields are real controller data" gate —
check it before trusting `AttackPos`, exactly as the stock weapon code does
(`wadsrc/static/zscript/actors/attacks.zs:467`, `:482`).

Everything else in `_BASELINE_FACTS.md` holds: **no velocity fields** for either hand, and script
samples at 35 Hz. So: **two readable hands, zero readable velocities.**

---

## 1. Verdict

Grabbing, holding, throwing and human-shielding are all **BUILDABLE** on this engine with no C++
changes, and the two load-bearing primitives are already shipped: per-tic interpolation is
re-snapshotted for every actor at the top of the tic (`src/p_tick.cpp:151-162`), so **any**
position write during the tic renders smoothly; and `LineTrace` already traces from the real
controller via `TRF_USEWEAPON | TRF_ISOFFHAND` (`src/playsim/p_map.cpp:5185-5200`), which is a
finished gravity-glove ray. Human shields work for free because both the projectile path
(`src/playsim/p_map.cpp:1670-1717`) and the hitscan path (`src/playsim/p_map.cpp:4919-4920`,
trace actor-mask `MF_SHOOTABLE`) resolve against the first qualifying actor in the way, with no
"is this the intended target" test. Doors and switches are **not** physically pushable — the
engine only has an activation event (`P_ActivateLine`, `src/playsim/p_spec.cpp:155`), but that
event, and raw plane motion (`Sector.MoveCeiling`, `wadsrc/static/zscript/mapdata.zs:575`), are
both fully script-driven, so a hand-driven door is a mod-side animation over an engine-side mover.
The one real **WALL** is haptics: nothing in `wadsrc/static/zscript` can fire a rumble, so a grab
will never have a physical click.

---

## 2. What already exists

| Thing | Where | Note |
|---|---|---|
| Per-tic interpolation snapshot for every actor | `src/p_tick.cpp:151-162` | `ac->ClearInterpolation()` is called on **all** actors *before* any thinking. This is why moving an actor in script interpolates. |
| Interpolated render position | `src/playsim/actor.h:1488-1492` | `Prev + ticFrac * (Pos() - Prev)`; used at `src/rendering/hwrenderer/scene/hw_sprites.cpp:829`. |
| `Prev` is written **only** by `ClearInterpolation` | `src/playsim/actorinlines.h:29-35` | Plus portal/sector fixups (`p_map.cpp:2820`, `p_mobj.cpp:3871`). Nothing else per tic. |
| VR-aware `use` on **both hands** | `src/playsim/p_map.cpp:6168-6190` | `use_mode` cvar (`p_map.cpp:105`). `use_mode > 0` + `OverrideAttackPosDir` traces a use-line from `AttackPos`/`AttackAngle`, then from `OffhandPos`/`OffhandAngle`. |
| Controller-origin line trace | `src/playsim/p_map.cpp:5185-5200` | `TRF_USEWEAPON` (4096) + `TRF_ISOFFHAND` (2048), `wadsrc/static/zscript/constants.zs:961-962`. |
| Absolute-start line trace | `src/playsim/p_map.cpp:5207-5210` | `TRF_ABSPOSITION` puts the start at `(offsetforward, offsetside, offsetz)` in world coords. |
| Hitscan picks first `MF_SHOOTABLE` actor, no owner preference | `src/playsim/p_map.cpp:4919-4920` + filter `CheckForActor` at `:4673-4700` | Only skips on THRUACTORS/SPECTRAL/species/ghost/ThruBits/player-passthrough. |
| Projectile-vs-actor resolution | `src/playsim/p_map.cpp:1589-1717` | Height clip at `:1635-1641`, infighting gate `CanAttackHurt` at `:1672` (impl `:1275`). |
| Impact callback on a flying actor | `wadsrc/static/zscript/actors/actor.zs:520` `virtual native bool Slam(Actor victim)` | Fired from `PIT_CheckThing` at `src/playsim/p_map.cpp:1554-1559` when the mover has `MF_SKULLFLY`. |
| Pushable actors (vanilla, velocity-based) | `src/playsim/p_map.cpp:1723-1730` | `+PUSHABLE` / `+CANNOTPUSH` (`src/scripting/thingdef_data.cpp:156`, `:159`), `pushfactor` (`actor.zs:200`, default `0.25` at `actor.zs:436`). |
| Full actor freeze that keeps collision | `src/playsim/p_mobj.cpp:3948-3952`, field `src/playsim/actor.h:1284` | `freezetics` is a **writable** `native uint` (`actor.zs:265`). Returns out of `AActor::Tick` before any physics or AI; stays linked in the blockmap. |
| Destructible lines/sectors | `src/playsim/p_destructible.cpp:121-154`, `:156+` | `Line.GetHealth/SetHealth` (`mapdata.zs:256-257`), `Sector` equivalents (`mapdata.zs:661-662`). Map-authored health only. |
| Line activation from script | `wadsrc/static/zscript/mapdata.zs:239-240` → `src/playsim/p_spec.cpp:218-238` | `Line.Activate(actor, side, SPAC_*)`, `Line.RemoteActivate(..., pos)`. |
| Raw plane motion from script | `wadsrc/static/zscript/mapdata.zs:574-575` | `Sector.MoveFloor/MoveCeiling(speed, dest, crush, direction, hexencrush, instant)`. |
| Thinker ordering | `src/playsim/dthinker.cpp:145-149` | Statnums tick ascending. Players are `STAT_PLAYER` 33, actors `STAT_DEFAULT` 100, `STAT_BOT` 104 (`wadsrc/static/zscript/doombase.zs:161`, `:176`, `:179`). |
| Scale | `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:94` | `vr_vunits_per_meter` default **34.0**. 1 m/s hand speed ≈ 0.97 map units per tic. |

**Does not exist:** haptics from ZScript (baseline §4 — zero hits in `wadsrc/static/zscript`);
any hand/head velocity field; any engine concept of "held by hand".

---

## 3. Building blocks — exact APIs

**Move a held actor (the one that matters):**

| Call | Signature | Interpolation | Collision |
|---|---|---|---|
| `SetOrigin(pos, moving)` | `actor.zs:753` → `src/playsim/p_maputl.cpp:563-571` | `moving == true` → **smooth** (leaves `Prev` alone, renderer lerps). `moving == false` → calls `ClearInterpolation()` → **snaps**. | Unlink → `SetXYZ` → `LinkToWorld` → `P_FindFloorCeiling`. **No collision test at all.** Will place the actor inside geometry. |
| `SetXYZ(pos)` | `actor.zs:754` → `src/scripting/vmthunks_actors.cpp:573-578` | Smooth. | **Does not relink.** Blockmap/sector links go stale — the actor becomes un-hittable and un-findable at its new spot. Never use alone for a held object. |
| `TryMove(xy, dropoff, ...)` | `actor.zs:831` | Smooth. | Full `P_CheckPosition` + line/actor blocking. XY only; set Z separately. |
| `CheckMove(xy, flags, tm)` | `actor.zs:832` | — | Test only. Use to decide whether to follow the hand this tic. |
| `Warp(dest, ...)` | `actor.zs:942` → `src/playsim/p_things.cpp:703-823` | **Snaps by default** (`ClearInterpolation()` at `:805`) unless `WARPF_INTERPOLATE` / `WARPF_WARPINTERPOLATION` (`:791-804`). Internally calls `SetOrigin(..., true)` then `P_TryMove`. | Position-tests via `P_TestMobjLocation` and reverts on failure (`:765`, `:822`). |
| `UnlinkFromWorld` / `LinkToWorld` | `actor.zs:841-842` | — | The manual version of `SetOrigin`. Use when you must also change `radius`/`height`. |
| `A_SetSize(radius, height, testpos)` | `actor.zs:1314` | — | Shrink a grabbed corpse/barrel so it fits in front of the face. |

**The interpolation rule, stated once:** `Prev` is re-snapshotted for every actor at the top of
every tic (`src/p_tick.cpp:151-162`) and is written nowhere else per-tic
(`src/playsim/actorinlines.h:31`). Therefore **any** position change you make during the tic is
interpolated from where the actor was at tic start to where you put it — smooth, no smear.
The only thing that produces a *snap* is explicitly calling `ClearInterpolation()`, which is what
`SetOrigin(pos, false)` (`p_maputl.cpp:570`) and bare `Warp` (`p_things.cpp:805`) do.

**Use `SetOrigin(pos, true)` to track a hand. Never `SetOrigin(pos, false)`, never bare `Warp`.**

Cost of that choice: the object renders along the segment between last tic's pose and this tic's
pose, i.e. roughly one tic (~28 ms) behind the real controller. That is the floor; there is no
per-frame playsim hook to beat it.

**Sensing:**
- `LineTrace(angle, distance, pitch, flags, offsetz, offsetforward, offsetside, out data)` — `actor.zs:805`.
  Glove ray: `TRF_USEWEAPON|TRF_ISOFFHAND|TRF_SOLIDACTORS|TRF_BLOCKUSE`.
  Arbitrary ray: `TRF_ABSPOSITION` with the start in `offsetforward/offsetside/offsetz`.
- `BlockThingsIterator.CreateFromPos(x, y, z, height, radius, ignorerestricted)` — `wadsrc/static/zscript/doombase.zs:225`. The grab-sphere test.
- `CheckPosition(pos, actorsonly, tm)` — `actor.zs:763`.

**Damage / force:**
- `RadiusAttack(src, damage, dist, mod, flags, fulldamagedist, species)` — `actor.zs:1263`.
- `A_Explode(...)` — `wadsrc/static/zscript/actors/attacks.zs:702`.
- `DamageMobj`, `Vel` (writable, `actor.zs:112`), `Mass` (`actor.zs:165`, default `100` at `actor.zs:416`), `pushfactor` (`actor.zs:200`).

**Flags that decide grabbability:** `bSOLID`, `bSHOOTABLE`, `bNOGRAVITY`, `bNOCLIP`, `bTHRUACTORS`,
`bPUSHABLE`, `bCANNOTPUSH`, `bNONSHOOTABLE`, `bDONTBLAST`, `bNOINTERACTION`, `bCORPSE`
(`src/scripting/thingdef_data.cpp:119-253`).

---

## 4. Mechanic designs

> Read this section if you read nothing else.

### 4.1 Grab an inert object (barrel, item, corpse) and hold it

**Player does:** puts a hand near the object, squeezes grip.

**Script detects:** on grip-press, `BlockThingsIterator.CreateFromPos(hand.x, hand.y, hand.z, 16, 24, false)`
(`doombase.zs:225`) where `hand` is `OffhandPos` or `AttackPos`. Filter the candidates, pick the
nearest by `(it.thing.pos - hand).Length()`. Record `grabOffset = thing.pos - hand` and
`grabYaw = thing.angle - handAngle` so the object keeps the orientation it had when you took it.

**Resolves as:** a controller thinker at `STAT_BOT` (104) — above `STAT_DEFAULT` 100, so it ticks
*after* every actor has thought (`src/playsim/dthinker.cpp:145-149`) and your write is the last
word that tic. Each tic:

1. `held.freezetics = 2;` — stops its own gravity, AI and state advance while keeping it linked in
   the blockmap and still solid (`src/playsim/p_mobj.cpp:3948-3952`; contrast `+NOINTERACTION`,
   which forces `MF_NOBLOCKMAP` at `src/playsim/p_mobj.cpp:3988-3992` and makes the object
   un-hittable — do not use it).
2. `targetPos = handPos + rotate(grabOffset, handAngle);`
3. If you want the object to refuse to go through walls:
   `if (held.CheckMove(targetPos.xy)) held.SetOrigin(targetPos, true);`
   else slide: keep the last valid XY, apply the Z.
   If you want a rigid, "it is bolted to my hand" feel, skip `CheckMove` and call
   `SetOrigin(targetPos, true)` unconditionally — it does no collision test
   (`src/playsim/p_maputl.cpp:563-571`) so the object will clip into walls.
4. `held.angle = handAngle + grabYaw; held.pitch = handPitch; held.roll = handRoll;`

**Rigid vs lag:** rigid = step 3 with no filter. Lag = lerp `targetPos` toward the hand with a gain
`k` (`newPos = lerp(held.pos, targetPos, k)`), `k = 1.0` rigid, `k ≈ 0.35` for a heavy barrel.
Derive `k` from `Mass`: `k = clamp(100.0 / max(held.Mass, 1), 0.15, 1.0)` — `Mass` default 100
(`actor.zs:416`).

**Items do not auto-pick-up when you move them to yourself.** Pickup fires only when the *mover*
has `MF_PICKUP` and the *other* actor has `MF_SPECIAL` (`src/playsim/p_map.cpp:1737-1743`), and you
are moving the item, not the player. Call `Inventory.Touch(player)` (`wadsrc/static/zscript/actors/inventory/inventory.zs:828`)
or `CallTryPickup(player)` (`:647`) yourself when the item reaches the chest.

**Corpses:** still linked and findable — `Die` clears `MF_SHOOTABLE|MF_FLOAT|MF_SKULLFLY` and sets
`MF_CORPSE` (`src/playsim/p_interaction.cpp:427-435`), and solidity is cleared by the death state's
`A_NoBlocking`, not by unlinking. So `BlockThingsIterator` finds corpses; test `bCORPSE` to grab them.

**Label: BUILDABLE. Cost: M.**

### 4.2 Grab a live enemy

Same as 4.1 plus:
- `freezetics = 2` each tic is what actually stops it fighting back — it returns out of
  `AActor::Tick` before AI and before `P_XYMovement` (`src/playsim/p_mobj.cpp:3948-3952`).
  Without it, the monster's own `A_Chase` → `P_Move` → `TryMove` runs at `STAT_DEFAULT` and fights
  your write; because your controller ticks at 104 you still win the frame, but the monster will
  keep triggering its own blocking/line specials.
- Big monsters: `A_SetSize(newradius, newheight, true)` (`actor.zs:1314`) while held, restore on release.
- Keep `bSHOOTABLE` set — that is the whole point (see 4.5).

**Label: BUILDABLE. Cost: M.** (L if you want per-monster struggle behaviour and grab-break.)

### 4.3 Gravity-glove pull: point, flick, catch

**Player does:** points the empty hand at a distant object, flicks the wrist/arm toward themself.

**Script detects the target:** this is shipped. From the player pawn,
`LineTrace(0, 1024, 0, TRF_USEWEAPON|TRF_ISOFFHAND|TRF_SOLIDACTORS, 0, 0, 0, data)` —
`src/playsim/p_map.cpp:5185-5200` substitutes `OffhandPos` for the start and `OffhandDir` for the
direction, so you get a ray out of the real controller without computing anything. Drop
`TRF_ISOFFHAND` for the weapon hand. `data.HitActor` is the candidate; highlight it.

**Script detects the flick:** there is no velocity field (baseline §1b) and you get 35 samples a
second, so keep a small ring buffer in the controller thinker:

```
vector3 hist[6];           // 6 tics ≈ 171 ms
hist[n] = OffhandPos;      // once per tic, in the controller's Tick
```

A flick is: over the last 3 samples, the component of displacement along the *hand's own
backward axis* exceeds a threshold, and the hand then decelerates.

```
vector3 d = hist[n] - hist[n-2];                 // 2 tics ≈ 57 ms
vector3 back = -(OffhandDir(self, 0, 0));        // actor.zs:859
double pull = d dot back;                        // map units over 2 tics
if (pull > FLICK_UNITS) -> fire
```

Calibration: `vr_vunits_per_meter` is 34 (`hw_vrmodes.cpp:94`), so a 2.5 m/s flick covers
`2.5 * 34 * (2/35) ≈ 4.9` map units in those two tics. `FLICK_UNITS ≈ 4.0` is a usable start.
Require `|d|` to also be the largest of the last four 2-tic deltas, so a steady arm sweep does not
trigger. **Two to four samples is the entire budget** — do not try to fit a curve.

**Resolves as:** set the target's `Vel` to an arc that arrives at the hand in a fixed time rather
than at a fixed speed, so the catch is predictable:

```
double t = CATCH_TICS;                       // 14 tics = 0.4 s reads well
vector3 to = (handPos + handForward*20) - targ.pos;
targ.Vel = to / t;
targ.Vel.Z += 0.5 * GRAV * t;                // GRAV = targ.gravity * level.gravity/800 per tic^2
targ.bNOGRAVITY = false;
```

Then run a "incoming" state: each tic, if `(targ.pos - handPos).Length() < CATCH_RADIUS` (24 units
reads as a hand) **and** the grip is held, transition straight into 4.1's held state. If the grip
is not held, let it fly past — that is the correct, legible failure.

**Feel knobs:** `FLICK_UNITS` (how hard a flick must be), `CATCH_TICS` (flight time; shorter is
snappier and harder to catch), `CATCH_RADIUS` (forgiveness), whether you zero `Vel.Z` for a flat
arc, and whether a missed catch leaves the object tumbling (`bNOGRAVITY = false` + spin `angle`).

**Label: BUILDABLE. Cost: M.** The ray is EXISTS; only the gesture and arc are yours.

### 4.4 Throwing

**Player does:** swings the hand, releases grip.

**Script detects:** the same ring buffer from 4.3, sampled while holding. On grip-release, take the
**best of the last three 2-tic deltas** (not the most recent — the release tic is often already
decelerating, and at 35 Hz you frequently have exactly one good sample):

```
vector3 best; double bestlen = 0;
for (i = 1; i <= 3; i++) { vector3 d = hist[n-i+1] - hist[n-i-1]; if (d.Length() > bestlen) {...} }
vector3 throwVel = best / 2.0;               // map units per tic
```

**Resolves as:**

```
held.freezetics = 0;
held.Vel = throwVel * THROW_GAIN * (100.0 / max(held.Mass,1));
held.bNOGRAVITY = false;
held.target = players[consoleplayer].mo;     // credit for the kill
held.bSKULLFLY = true;                       // <-- the important line
```

`+SKULLFLY` is the free impact callback. `PIT_CheckThing` calls `tm.thing->CallSlam(BlockingMobj)`
at `src/playsim/p_map.cpp:1554-1559` when a `MF_SKULLFLY` mover hits an actor, and `Slam` is a
ZScript virtual (`wadsrc/static/zscript/actors/actor.zs:520`). Override it on your thrown-object
mixin and scale damage by speed:

```
override bool Slam(Actor victim) {
    double spd = Vel.Length();                       // units per tic
    int dmg = int(spd * spd * Mass * IMPACT_SCALE);
    if (dmg > 0) victim.DamageMobj(self, target, dmg, 'Melee');
    bSKULLFLY = false; Vel *= -0.2;                  // bounce back a little
    return true;
}
```

Wall impacts: when a `MF_SKULLFLY` mover is stopped by geometry the engine clears the flag and
zeroes velocity itself (`src/playsim/p_mobj.cpp:2134-2143`), and bounces it off floor/ceiling
(`:2772`, `:2807`). Hook `A_Die`-style wall damage in your own `Tick` by watching for
`bSKULLFLY` having been cleared with `BlockingLine != null`.

**Throwing barrels:** an `ExplosiveBarrel` thrown this way is a grenade with no new code — have
`Slam` call `A_Explode()` (`wadsrc/static/zscript/actors/attacks.zs:702`) or
`RadiusAttack(self, 128, 128)` (`actor.zs:1263`) instead of `DamageMobj`. Set `target` to the
player first or `XF_HURTSOURCE` semantics will surprise you.

**Feel knobs:** `THROW_GAIN` (1.0 = real-arm speed, which feels weak in Doom units; 1.6–2.2 reads
better), the `Mass` divisor, `IMPACT_SCALE`, whether the object spins (`angle += spd*k` per tic),
and whether light objects get `+SKULLFLY` at all (a thrown health bonus should not kill).

**Label: BUILDABLE. Cost: M.**

### 4.5 Human shields

**Does a held monster actually block shots? Yes, and the citations are unambiguous.**

**Hitscan (zombieman, chaingunner, shotgunner):** `P_LineAttack` traces with the actor mask
`MF_SHOOTABLE` — `src/playsim/p_map.cpp:4919-4920`:

```
Trace(tempos, t1->Sector, direction, distance, MF_SHOOTABLE,
      ML_BLOCKEVERYTHING | ML_BLOCKHITSCAN, t1, trace, tflags, CheckForActor, &TData)
```

The trace stops at the **first** actor carrying `MF_SHOOTABLE`. `CheckForActor`
(`src/playsim/p_map.cpp:4673-4700`) skips only on `THRUACTORS`, `SPECTRAL`, species matches,
`GHOST`, `ThruBits`, and player-passthrough. **There is no check that the hit actor is the
shooter's intended target.** A live monster held between you and a shotgunner eats the pellets.

**Projectiles (imp fireball, revenant missile, cacodemon):** `PIT_CheckThing`
(`src/playsim/p_map.cpp:1589-1717`). The height gate is `:1635-1641` (over/under the target's
`Height`, or `projectilepassheight`), and damage lands at `P_DoMissileDamage(tm.thing, thing)`
(`:1708`). The one gate that can let a fireball **through** your shield is infighting:

```
src/playsim/p_map.cpp:1670-1676
    if (!CanAttackHurt(thing, tm.thing->target)) return false;
```

`CanAttackHurt` (`:1275-1300`) exempts players entirely — *"players are never subject to
infighting settings"* — but a monster's missile hitting **another monster** is gated by
`Level->GetInfighting()` (`src/g_level.cpp:2366-2371`, from `MAPINFO` flags or the skill's
`SKILLP_Infight`). On a map with `NoInfighting`, your shield is transparent to monster fireballs.
Force it per-shield by setting `+FORCEINFIGHTING` on the held monster — that flag overrides
`infight < 0` explicitly at `src/playsim/p_map.cpp:1282`.

**How to hold one in front of you:** 4.2, with `targetPos = HmdPosition-derived chest point +
forward * (held.radius + 12)`. Keep `bSHOOTABLE` and `bSOLID`. Do **not** set `+NOINTERACTION`
(`src/playsim/p_mobj.cpp:3988-3992` forces `MF_NOBLOCKMAP`, and an unlinked actor is hit by nothing).

**Feel knobs:** how much of the incoming damage leaks to the player (read it in the shield's
`DamageMobj` override and forward a fraction), how fast the shield dies, whether the corpse stays
grabbed after death (it does, see 4.1 — it just stops blocking, because `Die` clears
`MF_SHOOTABLE` at `src/playsim/p_interaction.cpp:427`).

**Label: BUILDABLE (the blocking itself is EXISTS). Cost: S** on top of 4.2.

### 4.6 Catching or redirecting projectiles in flight

**Player does:** swats at an incoming fireball.

**Script detects:** missiles are ordinary actors in the blockmap, so the same
`BlockThingsIterator.CreateFromPos` around the hand finds them; filter on `it.thing.bMISSILE`.
Because the missile moves in `P_XYMovement` at `STAT_DEFAULT` and you tick at 104, you see its
post-move position each tic — at imp-fireball speed (10 units/tic) a 40-unit grab sphere gives you
~4 tics of window, which is playable. Faster missiles (revenant tracer, 10–20) give 2.

**Resolves as:** one of three, all script-only:
- **Catch:** `m.Vel = (0,0,0); m.freezetics = 2;` and hold it per 4.1. Throwing it again is 4.4
  with `bSKULLFLY` left alone, since it is already `MF_MISSILE` and does its own damage.
- **Redirect:** `m.target = players[consoleplayer].mo; m.angle = handAngle; m.pitch = handPitch;
  m.Vel3DFromAngle(m.Speed, m.angle, m.pitch);` (`actor.zs:857`). Re-pointing `target` is what
  makes it hurt monsters: `CanAttackHurt` is called with `tm.thing->target`
  (`src/playsim/p_map.cpp:1672`), and a player source bypasses infighting entirely (`:1277-1279`).
- **Swat down:** `m.ExplodeMissile()` in place.

**Caveat:** this is the one mechanic the 35 Hz sample rate genuinely hurts. You cannot see the
missile between tics; a fast projectile can be 20 units past the hand before you look. Compensate
by sweeping the segment: build the iterator at the **midpoint** of the hand's last two samples with
a radius of `(handTravel + missileSpeed) / 2 + 24`, rather than at the current hand point.

**Label: BUILDABLE. Cost: M.** Feel knobs: grab radius, whether a catch requires grip-held or
is automatic, whether redirect preserves speed or boosts it.

### 4.7 Doors and switches — can you physically push or pull them?

**No. Find the activation path and you find a boolean event, not a force.**

- The use ray: `P_UseLines` (`src/playsim/p_map.cpp:6154-6190`). In VR with `use_mode > 0` it
  traces from `AttackPos`/`AttackAngle` and then `OffhandPos`/`OffhandAngle`, each over the
  weapon's `UseRange` (default 48; the pawn default is `USERANGE` 64, `src/playsim/p_local.h:64`).
  **The trace is 2D** — `start`/`end` are `DVector2` (`:6158-6160`); hand *height* does not select
  which linedef you use.
- The hit: `P_UseTraverse` (`src/playsim/p_map.cpp:5990-6105`). Actors first — `+USESPECIAL` or the
  `Used(Actor user)` ZScript virtual (`:6004-6011`, declared `wadsrc/static/zscript/actors/actor.zs:602`).
  Then lines, gated on `line->activation & (SPAC_Use|SPAC_UseThrough|SPAC_UseBack)` (`:6020`).
- The activation: `P_ActivateLine` (`src/playsim/p_spec.cpp:155-216`) → key check (`:175`) →
  `P_ExecuteSpecial` (`:180`) → `P_ChangeSwitchTexture` for `SPAC_Use|Impact|Push` (`:192-194`).
  It also fires `WorldLinePreActivated` / `WorldLineActivated` event-handler hooks (`:170`, `:183`).

**What script can drive, exactly:**

| Capability | API | Note |
|---|---|---|
| Fire a line's own special from a hand pose | `Line.Activate(actor, side, SPAC_Use)` — `wadsrc/static/zscript/mapdata.zs:239` → `src/playsim/p_spec.cpp:218-225` | Includes the key check and the switch-texture flip. This is the "my hand touched the switch" primitive. |
| Same, with a 3D contact point | `Line.RemoteActivate(actor, side, type, pos)` — `mapdata.zs:240` → `p_spec.cpp:227-238` | Pass the hand position; it reaches `P_TestActivateLine` as `optpos`. |
| Run any action special directly | `Level.ExecuteSpecial(special, activator, line, lineside, a1..a5)` — `wadsrc/static/zscript/doombase.zs:489` | `Door_Raise`, `Door_Open`, `Plat_*`, anything. |
| Drive a plane by hand, continuously | `Sector.MoveCeiling(speed, dest, crush, direction, hexencrush)` / `MoveFloor(..., instant)` — `wadsrc/static/zscript/mapdata.zs:574-575` | This is the only way to get "the door moves as far as I pulled it". |
| Veto / intercept an activation | event handler `WorldLinePreActivated` — `src/playsim/p_spec.cpp:170` | Lets you require a hand gesture before a normal use succeeds. |
| Make an actor usable by hand | override `Actor.Used(Actor user)` — `actor.zs:602`, called at `src/playsim/p_map.cpp:6004-6011` | For hand-cranked props you spawn yourself. |

**So a pull-open door is:** a thinker that, while the hand is gripped within reach of a door
linedef, computes hand displacement along the door's normal and each tic calls
`sec.MoveCeiling(speed, targetHeight, 0, dir, false)` with `targetHeight` derived from how far the
hand has moved. You are animating an engine mover from hand data; you are not pushing anything.
Vanilla door *thinkers* will fight you, so claim the sector (run `Door_Raise` never, or stop the
existing thinker) before driving it.

**Label: HACK (physical push/pull) / BUILDABLE (hand-driven activation + hand-driven plane motion).
Cost: M.**
**WALL:** there is no force applied to a sector plane and no "how hard are you pushing" input in
the engine. Resistance, slam-shut-on-release and weight must be faked by the mod's own curve.

### 4.8 Going through windows without map changes

Two separate questions, two separate answers.

**Can script make an existing linedef passable at runtime? Yes.** `Line.flags` is a writable
`native uint` (`wadsrc/static/zscript/mapdata.zs:213`), and the blocking test reads it live every
move: `PIT_CheckLine` at `src/playsim/p_map.cpp:936-948` tests `ld->flags & ProjectileBlocking`
and `P_IsBlockedByLine(tm.thing, ld)` on each `P_CheckPosition`. Clear `ML_BLOCKING` /
`ML_BLOCKEVERYTHING` on a two-sided line and bodies pass through from that tic onward. Textures
can go with it: `Side.SetTexture(which, tex)` (`mapdata.zs:122`).

**Can script make a *wall* passable? No.** `PIT_CheckLine` returns `false` for a one-sided line
(`!ld->backsector`, `src/playsim/p_map.cpp:880-931`) **before any flag is consulted**. There is no
back sector to open into, so there is nothing to pass through. No ZScript API creates a sector or
splits a linedef.

**What the vertical gap is:** `P_LineOpening` on the two-sided line decides whether you fit; a
line flagged `ML_RAILING` is handled specially (`src/playsim/p_map.cpp:943`) so you step over it
within `MaxStepHeight`. A chest-height window still needs the player raised — there is no vault.

**Destructible glass exists but is map-authored:** `P_DamageLinedef`
(`src/playsim/p_destructible.cpp:121-154`) only does anything if the line has `health` and
`SPAC_Damage`/`SPAC_Death` activation set in UDMF. Script can set it after the fact —
`Line.SetHealth(n)` (`mapdata.zs:257`) plus writing `line.activation` (`mapdata.zs:215`) — which
means a mod **can** retrofit breakable windows onto an unmodified map, if it can identify which
lines are windows (texture name match is the only handle).

**Label: BUILDABLE for two-sided lines (cost S). WALL for one-sided walls** — missing piece: any
runtime geometry-creation API (sector/linedef splitting). Nothing in `wadsrc/static/zscript`
creates map geometry.

---

## 5. Walls

| Wall | Missing piece | Citation |
|---|---|---|
| **No haptic feedback on grab, catch or impact** | No ZScript binding to `VRMode::Vibrate` / `VR_HapticEvent`. | `src/common/rendering/hwrenderer/data/hw_vrmodes.h:94`, `:97`; zero hits in `wadsrc/static/zscript` (baseline §4). A grab will be silent to the hands. |
| **No hand velocity; 35 Hz ceiling on every gesture** | No velocity field, and no per-frame playsim hook. Everything in §4.3/4.4/4.6 runs on 2–4 samples. | Baseline §1b, §3; write sites `hw_openxrdevice.cpp:585-610`. |
| **Held objects render ~1 tic behind the hand** | Interpolation lerps from last tic's position to this tic's; there is no way to write an actor's position per frame. | `src/p_tick.cpp:151-162`, `src/playsim/actor.h:1488-1492`. |
| **Cannot physically push a door or switch** | No force input to a sector plane; activation is a boolean event only. | `src/playsim/p_spec.cpp:155-216`. Mod must animate `Sector.MoveCeiling` itself. |
| **Cannot open a hole in a one-sided wall** | No runtime geometry creation; one-sided lines block before flags are read. | `src/playsim/p_map.cpp:880-931`. |
| **Two-handed grab on one object is asymmetric in practice** | Both hands *are* readable (§0), but the weapon hand's pose is also the player's aim — `AttackAngle` is what the weapon fires along (`src/playsim/p_map.cpp:5196-5199`), so using it to hold something fights the aiming model. | `hw_openxrdevice.cpp:587-596`. |
| **`+NOINTERACTION` is not a "held" flag** | It forces `MF_NOBLOCKMAP`, which makes the object un-hittable and invisible to iterators — fatal for shields and catchable objects. | `src/playsim/p_mobj.cpp:3988-3992`. Use `freezetics` instead. |
| **Monster-vs-monster shield leak** | `CanAttackHurt` can return false under `NoInfighting` maps/skills. | `src/playsim/p_map.cpp:1670-1676`, `:1275-1300`, `src/g_level.cpp:2366-2371`. Fix with `+FORCEINFIGHTING` (`p_map.cpp:1282`). |

**Performance.** A held object costs one `BlockThingsIterator` sweep on grab and, per tic, one
`SetOrigin` (unlink + relink + `P_FindFloorCeiling`, `src/playsim/p_maputl.cpp:563-571`) plus
optionally one `CheckMove`. That is roughly one monster-move's worth of blockmap work per held
object per tic, and `freezetics` *removes* the object's own `Tick` cost
(`src/playsim/p_mobj.cpp:3948-3952`), so a held object is close to free. A dozen simultaneously
held objects is not a concern; a hundred would be, because each relink touches every blockmap cell
the object spans. The real cost is a per-tic `BlockThingsIterator` if you scan for grab candidates
continuously — scan only while the grip button is down, or every 3rd tic.

---

## 6. Smallest prototype

One `.pk3`, one ZScript file, no C++. Target: pick a barrel up with the offhand and throw it.

1. **`GrabController : Thinker`**, `ChangeStatNum(STAT_BOT)` in its ctor (`wadsrc/static/zscript/doombase.zs:179`, `:192`) so it ticks after all actors (`src/playsim/dthinker.cpp:145-149`). Created once per player from an `EventHandler.PlayerEntered`.
2. **Each `Tick`:** push `pawn.OffhandPos` into a 6-slot ring buffer. Bail out unless `pawn.OverrideAttackPosDir` (`actor.zs:276`).
3. **Grip press** (read whatever button your offhand weapon exposes — the per-hand button path already exists, `player.zs:588`): `BlockThingsIterator.CreateFromPos(hand.x, hand.y, hand.z, 24, 32, false)`, take the nearest `bSOLID && !bISMONSTER` thing. Store `held`, `grabOffset`, `grabYaw`.
4. **While held:** `held.freezetics = 2; held.SetOrigin(hand + rotate(grabOffset, OffhandAngle), true); held.angle = OffhandAngle + grabYaw;`
5. **Grip release:** best-of-3 2-tic delta → `held.Vel = delta/2 * 1.8;` `held.freezetics = 0;` `held.bSKULLFLY = true;` `held.target = pawn;` clear `held`.
6. **Prove it:** a `Slam` override on a `BarrelGrabbable : ExplosiveBarrel replaces ExplosiveBarrel` that calls `A_Explode()` and prints the impact speed.

If step 4 looks smooth and step 5 makes the barrel arc and explode on an imp, every other mechanic
in §4 is the same three primitives rearranged.

---

## 7. Open questions

1. **Which button reaches a grab?** This survey did not trace the VR grip button to a ZScript-readable input. `CheckWeaponButtons(hand)` (`player.zs:588`, `:2031`) and the offhand weapon state machine are the obvious route — a dummy `+OFFHANDWEAPON` "glove" weapon whose `Fire` state is the grab — but someone must confirm the grip maps to a weapon button under `vr_control_scheme` and is not swallowed. **Topic for whoever owns the input survey.**
2. **Is `AttackRoll` populated outside OpenXR?** `hw_openxrdevice.cpp:591` sets it; `hw_vrmodes.cpp:443` (the non-VR / fallback `SetUp`) sets `AttackPitch` and `AttackAngle` but **not** `AttackRoll`. A held object's roll may be stale on the OpenVR path (`src/rendering/gl/stereo3d/gl_openvr.cpp:2549-2552` not read in detail here).
3. **Does a held, frozen monster still get `WorldThingDamaged` events correctly?** `freezetics` returns out of `Tick` but `DamageMobj` is called from outside `Tick`, so it should; untested.
4. **Netplay.** Everything here writes playsim state from a per-player thinker driven by per-client controller poses. `OffhandPos`/`AttackPos` are written by the *renderer* on the local machine only. In multiplayer, a grab computed this way will desync. Not investigated; assume single-player only until someone cites the netcode path.
5. **`Z` is in stretched map space.** `AttackPos.Z` / `OffhandPos.Z` are divided by `pixelstretch` (`hw_openxrdevice.cpp:595`, `:609`). Grab radii compared in 3D will be anisotropic. Whether to correct for it is a feel decision, not a correctness one.
6. **Polyobjects** were not surveyed. A hand-pushed polyobject door may be a better fit than `Sector.MoveCeiling` for §4.7; `src/playsim/po_man.cpp` is unread.
