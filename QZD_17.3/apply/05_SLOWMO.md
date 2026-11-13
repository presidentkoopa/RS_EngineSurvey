# 05 — APPLY: RSQ_SlowMo against the engine audit

**Mod audited:** `RSQ_UBQSeries/SlowMo` (RSQ_SlowMo), read-only, from a fresh clone of
`github.com/presidentkoopa/RSQ_UBQSeries` at **HEAD `6a5c98cc` (Fri 30 Oct 2026)**. Mod
citations are `SlowMo/<path>:<line>`, and sibling-mod citations `<Mod>/<path>:<line>`, against
that clone.
**Engine:** tag `qzd17.3`, read-only. Engine citations are repo-relative at that tag.
**Reports used:** `surveys/00_CAPABILITIES.md` §4, §4.1; `surveys/10_TIME.md` (all of it).
**I changed nothing.** No engine edits proposed. ZScript and data only.

> **Source note.** The first pass of this report was written against a two-day-old snapshot
> unzipped from the local repo's `src/*.zip`. I re-ran it against the HEAD clone and **diffed
> the two trees**: `SlowMo/` in full, `Miniboss/` in full, `Score/zscript/rsscore/handler.zs`,
> `Grapple/zscript/rsqgr/util.zs` and `Accessibility/zscript/rsqa_core.zs` are **byte-identical
> modulo line endings** (CRLF in the zips, LF in the clone). `clock.zs` is 759 lines and
> `round.zs` is 226 in both. **Every mod citation and every line number below is verified
> against HEAD; none moved.** The only file that changed at all among the ones I read is
> `Score/zscript/rsscore/combo.zs`, which gained a "Headlock" combo for Distance Grab
> (`combo.zs:34`, `:73-75`) — no slow-motion reference anywhere in it, so it does not touch
> this report.
>
> The clone also carries a mod the snapshot lacked — `Wheel/` (the weapon wheel). It **is** a
> real interaction with a real hazard, and it is written up in §8.1.

---

## 0. Recommendation, up front

**This is a refinement, not a rework. Do not migrate to `freezetics`.**

The audit assumed the mod would be doing the naive thing and that `freezetics` would be the
upgrade. It is the other way round. `clock.zs` is a complete, deliberate implementation of
**10_TIME §5.2's "hero actor" tier — `Vel` + `Speed` + `tics` in lockstep — applied to every
actor in range**, precisely *because* `freezetics` strobes. The mod's header says so in its own
words at `SlowMo/zscript/rsqsm/clock.zs:4-8`: *"Freezing the world on most tics is easy, but at
quarter speed the world would update nine times a second, and that stutters in a headset."*
That is 10_TIME §5.2's 8.75 Hz judder finding, reached independently, before the audit was
written. **Both found it.** Recommending `freezetics` here would be recommending the bug the
mod was built to avoid.

The mod also already does the audit's single highest-value item — **`WorldHitscanPreFired`
returning true to convert an enemy hitscan into a dodgeable projectile**
(`SlowMo/zscript/rsqsm/round.zs:20-71`). The audit called that the mechanic the lobby is
actually about and the biggest win available. It is shipped, and its muzzle-position
reconstruction is **exactly right** — I checked it line by line against `P_LineAttack`.

So what is left is a short list of real defects and two cheap additions:

| # | Item | Label | Cost | Why |
|---|---|---|---|---|
| 1 | `Impulses()` corrupts a frozen actor's `Vel.Z` | BUG | **S** | one-line `isFrozen()` guard; see §2.3 |
| 2 | Friction prediction ignores `+SKULLFLY` / `+NOFRICTION` | BUG | **S** | two flags in two conditions; §2.4 |
| 3 | `Repitch` never touches channel 0 (`CHAN_AUTO`) | BUG | **S** | loop starts at 1; §6.2 |
| 4 | Walking monsters still strobe, at ~2.9 Hz | QUALITY | **M** | the one unsolved quality problem; §4 |
| 5 | No post-process grade at all | MISSING | **M** | the cheapest "sells it" win left; §6.1 |
| 6 | Per-tic cost unmeasured on the Quest CPU | RISK | **S** to measure | §7 |
| 7 | The Wheel cancels a player-held slow motion on close | BUG (in Wheel) | **S** | §8.1 — not SlowMo's file to fix |

Items 1-3 are an afternoon between them. Item 4 is the only thing in the mod that is
genuinely hard, and it is hard in the engine, not in the script.

---

## 1. What mechanism does `clock.zs` actually use? (Q1)

**It keeps its own per-actor clock, in `WorldTick`, by scaling `Vel`, `Gravity`, missile
`Speed` and state `tics` together. It does not use `freezetics`. It does not use
`Level.SetFrozen`. It does not touch `timefreezer`, `+NOTIMEFREEZE` or `ChangeStatNum`.**

Zero hits for `freezetics`, `SetFrozen`, `timefreezer`, `NoTimeFreeze` or `ChangeStatNum`
anywhere in `SlowMo/zscript/`. The only freeze-related call in the mod is a *read*:
`!a.isFrozen()` as a guard at `SlowMo/zscript/rsqsm/clock.zs:435`.

The mechanism, per slowed actor per tic, in `RSQSM_Clock.Process()`
(`SlowMo/zscript/rsqsm/clock.zs:411-489`):

| Channel | What the mod writes | Mod line | Engine field / consumer |
|---|---|---|---|
| Linear motion | `a.vel *= want/f` | `clock.zs:454` | `Actor.Vel`, `actor.zs:112` |
| Falling | `a.Gravity = baseGrav * want²` | `clock.zs:459` | `AActor::GetGravity()`, `actorinlines.h:86-90` |
| Missile re-aim / homing | `a.Speed = baseSpeed * want` (missiles only) | `clock.zs:456` | `Actor.Speed`, `actor.zs:113` |
| Animation + AI rate | state `tics` stretched by `1/f` on entry, fraction carried | `clock.zs:435-451` | `Actor.tics`, `actor.zs:134`; state machine `p_mobj.cpp:4498+` |
| Ground slide | `vel.xy *= friction^(want-1)` | `clock.zs:476-481` | `P_GetFriction`, applied at `p_mobj.cpp:2528-2532` |
| Scripted pushes | the *delta* from a predicted velocity is scaled by `f` | `clock.zs:499-521` | — |
| Sound | `A_SoundPitch(ch, 1-(1-want)·follow)` every 4th tic | `clock.zs:482`, `:538-543` | `actor.zs:1202` → `s_sound.cpp:1057-1068` |

Walking monsters are slowed **only** through `tics`. `Speed` is deliberately *not* scaled for
them (`clock.zs:456` gates on `a.bMISSILE`), with the reasoning spelled out at
`clock.zs:19-21`: `P_Move` reads `actor->Speed` once per `A_Chase`, and `A_Chase` runs once per
state, so stretching the state already divides the walk rate — scaling `Speed` too would give
f². **That reasoning is correct and verified:** `P_Move` takes `double speed = actor->Speed`
at `src/playsim/p_enemy.cpp:502`. The mod's own measurement agrees (handoff
`SlowMo/docs/HANDOFF_2026-10-26.md:47-49`: imp walk states 3 → 12 tics, ground speed ratio
0.244 against a nominal 0.25).

Scope and selection:
- Who is slowed: every `Actor` within `rsqsm_radius` (3000 u) of any player
  (`clock.zs:304-312`) that is not real-time. **Real-time** = players, owned `Inventory`,
  dormant actors, and any class whose name starts `rsq`/`ubqs`/`ssq` except `rsqsm_round*`
  (`clock.zs:277-301`), decided once per class into a `Map<Name,int>`.
- How they are found: a `ThinkerIterator("Actor")` sweep on the first tic and every 17 tics
  (`clock.zs:261`, `:365-383`), plus `WorldThingSpawned` for anything born mid-effect
  (`clock.zs:385-388`).
- The clock itself: `scale` eased toward `target` by one step per tic
  (`clock.zs:243-253`), driven by a held toggle, or a `rsm_pulse` hold, or full speed.

**A fair summary of the design: the mod is a hand-written `i_timescale` that exempts the
player.** 10_TIME §2.6 names `i_timescale` a WALL for script and says per-actor scaling "gets
very close". This mod *is* that per-actor scaling, built out — and in one respect it beats the
audit's own recommendation, because it is **continuous, not quantised to 1/(N+1)**. 10_TIME
§2.6 lists the quantisation as the cost of per-actor scaling and shrugs it off as "1/4 and 1/8
are the only values anyone asks for". The mod needs arbitrary values, because it *eases*
(`rsm_ramp_in` 0.15 s, `rsm_ramp_out` 0.40 s) and a `freezetics` duty cycle cannot ease — it
can only jump 1 → 1/2 → 1/3. **That is a second, independent reason `freezetics` is wrong
here, and the audit missed it.**

---

## 2. Is it correct? (Q2)

### 2.1 The desync trap — HANDLED, and better than the audit's framing

10_TIME §5.2 warns: `Vel` alone gives a sprinting animation drifting slowly, `tics` alone gives
a crawling animation flying at full speed. The mod scales **all three channels the engine
actually reads**, not two:

- `Vel` for ballistic motion (`clock.zs:454`).
- `Gravity` by **f²** (`clock.zs:459`) — this is the term 10_TIME §5.2 does not mention and
  it is required. Velocity is per tic and gravity is per tic², so a slowed fall needs the
  square. The mod's header states it (`clock.zs:10-12`) and the measurement at
  `HANDOFF:47-49` is consistent.
- `Speed` for missiles only (`clock.zs:456`), because a missile's own re-aim code (`A_Tracer`,
  `A_SeekerMissile`) rebuilds `Vel` from `Speed`.
- `tics` for everything (`clock.zs:435-451`), with a **carried fraction** (`carry[i]`), so
  `a.tics = 3` at f = 0.4 does not round to a permanent 7 and drift.

The `tics` bookkeeping is the subtle part and it is right. It distinguishes three cases
(`clock.zs:440-443`) — *counting down* (same state, `tics` fell by exactly 1), *stalled* (same
state, same `tics`, `tics > 1` — an actor not ticking), *entered* (anything else) — and only
stretches on entry. Without the `stalled` test a non-ticking actor would be re-stretched every
tic and freeze permanently; the handoff records exactly that bug being found and fixed
(`HANDOFF:34-36`, commit `026b277`, "its tics grew 16x a tic until it froze").

A **factor change mid-state** rescales only what is left (`clock.zs:460-466`), which is the
correct behaviour during a ramp and is not something a duty cycle can express at all.

**Verdict: no desync trap. All three channels move together, plus the f² term the audit
omitted.**

### 2.2 The statnum-sweep leak — NOT APPLICABLE, and the mod is clean

00_CAPABILITIES §4.1 and 10_TIME §2.7/§4.3 warn that `FThinkerIterator` starts at
`STAT_FIRST_THINKING` (verified: `src/playsim/dthinker.cpp:936-944`), so an actor parked below
statnum 32 vanishes from your own unfreeze sweep and is stranded.

**The mod never calls `ChangeStatNum`.** There is nothing to strand. It also keeps its own
`Array<Actor> act` rather than relying on re-finding actors, which is the defence 10_TIME §4.3
prescribes anyway.

### 2.3 BUG — `Impulses()` corrupts a frozen actor's `Vel.Z`

`Impulses()` (`clock.zs:499-521`) predicts what the engine did to an actor's velocity since
last tic, treats any unexplained remainder as a scripted push, and scales that remainder by
`f`. The prediction subtracts one tic of gravity at `clock.zs:504`:

```
if (!a.bNOGRAVITY && !onFloor) expect.z -= level.gravity * a.CurSector.gravity * a.Gravity * 0.00125;
```

That expression is an **exact** transcription of `AActor::GetGravity()`
(`src/playsim/actorinlines.h:86-90`: `Level->gravity * Sector->gravity * Gravity * 0.00125`).
Good.

But if the actor is frozen — by another mod's `PowerTimeFreezer`, by `Level.SetFrozen`, or by
anything that sets `freezetics` — `AActor::Tick` early-returns at `src/playsim/p_mobj.cpp:3948-3952`
and **no gravity is applied**. The mod still predicts it. So `d.z = +1.0` (at default
`level.gravity` 800: 800 × 1 × 1 × 0.00125 = 1.0), which clears the `d.Length() <= 0.5` gate at
`clock.zs:517`, and if the actor is airborne (`touching` false) the mod writes
`vel.z = expect.z + d.z * f` — i.e. it **injects `-(1-f)` units/tic of fake downward velocity,
every tic, for as long as the actor stays frozen.** At f = 0.25 that is −0.75 u/tic
accumulating. The actor is motionless while frozen, so nothing is visible — and then it is
launched downward the moment it unfreezes.

The mod *does* guard the `tics` stretch with `!a.isFrozen()` (`clock.zs:435`) but not
`Impulses`, not the `Vel` rescale at `:452-468`, and not the friction compensation at `:476`.

**Fix, cost S:** early-return `Impulses` (and skip the friction compensation) when
`a.isFrozen()` — `actor.zs:513`, which short-circuits on `freezetics > 0` at
`actorinlines.h:175-176` and so covers every freeze route. One line each.

**How reachable is it?** The mod stands down for a rival *slow-motion* mod (`clock.zs:103-110`)
but a vanilla `PowerTimeFreezer` sphere is not in that list and should not be — it is a
different mechanic. So: pick up a time sphere while slow motion is easing out, and airborne
monsters take a downward kick when the sphere ends. Rare, real, trivially fixed.

### 2.4 BUG — friction prediction ignores `+SKULLFLY` and `+NOFRICTION`

`P_XYMovement` skips friction entirely for `MF_MISSILE | MF_SKULLFLY | MF8_NOFRICTION`
(`src/playsim/p_mobj.cpp:2441-2444`). The mod's two friction sites guard only `bMISSILE`:

- `clock.zs:476` — `if (a.pos.z <= a.floorz + 0.01 && !a.bNOGRAVITY && !a.bMISSILE && ...)`
- `clock.zs:505` — `if (onFloor && !a.bNOGRAVITY && !a.bMISSILE)`

So for a skull-flying actor (a Lost Soul charging, anything mid-`A_SkullAttack`, Hexen's
`+SKULLFLY` chargers) at floor level, the mod applies a friction factor the engine never
applied, and then `Impulses` sees the discrepancy `d = expect·(1-fr)` — at `ORIG_FRICTION`
0.90625 and a charge speed of 20 u/tic that is ~1.9 units, well past the 0.5 gate — and
"corrects" it. Net: a charging Lost Soul bleeds speed in slow motion. Same for any mod actor
with `+NOFRICTION` (both flags are script-visible: `src/scripting/thingdef_data.cpp:140`,
`:324`).

**Fix, cost S:** add `&& !a.bSKULLFLY && !a.bNOFRICTION` to both conditions.

### 2.5 Minor — the STOPSPEED cutoff is not modelled

`P_XYMovement` zeroes `Vel.XY` outright when both components fall below `STOPSPEED`
(0x1000/65536 = 0.0625) — `src/playsim/p_mobj.cpp:2497-2514`. In slow motion velocities are
f× smaller, so a sliding corpse crosses that threshold after f× less *velocity* and therefore
travels less far than it would at full speed. The mod's friction compensation at
`clock.zs:480` (`fr ** (want-1.0)`) corrects the decay *rate* but cannot restore distance lost
to a hard cutoff. Cosmetic; a corpse's slide is shorter in slow motion. Not worth fixing —
naming it so nobody chases it as a mystery.

### 2.6 Actors stranded slowed on level change — HANDLED

`WorldUnloaded` calls `Drop(i)` on every entry before the level goes away
(`clock.zs:151-153`), and `Drop` (`clock.zs:523-536`) restores `vel`, `Speed`, `Gravity` and
`tics` from the stored bases. The adrenaline meter is carried on the player in an
`RSQSM_AdrToken` inventory item (`clock.zs:154-161`, `:727-735`) because an `EventHandler` is
rebuilt per map. Correct, and the handoff records it as a review fix (`HANDOFF:80-82`).

The in-level restore path is also right, if indirectly: when the clock returns to 1.0,
`FactorFor` returns 1.0 for everyone (`clock.zs:396`), the rescale branch at `:452` restores
every channel, and `:487` then drops the entry. By the time `ClearAll()` runs at `clock.zs:263`
the array is already empty, so that call is a harmless belt-and-braces — **the comment on it
("every actor restored") describes what `Process` did, not what `ClearAll` does.** If anyone
later reorders those two calls, `ClearAll` before `Process` would strand every slowed actor
permanently. Worth a sharper comment.

### 2.7 Save / load — HANDLED, and it serialises correctly

`RSQSM_Clock` keeps `Array<Actor>`, `Array<double>`, `Array<int>` and — the one I doubted —
`Array<State> lastState` (`clock.zs:58`). **It serialises.** `PStatePointer::WriteValue` /
`ReadValue` go through `FSerializer::StatePointer`
(`src/common/scripting/core/types.cpp:1240-1255`, impl `src/serializer_doom.cpp:326`), and
`PDynArray::WriteValue` delegates to its element type per element
(`src/common/scripting/core/types.cpp`, `PDynArray::WriteValue`). So a save taken mid-slow-motion
restores the whole tracking table including the state pointers.

The transient/synced split is deliberate and right: everything that must match across machines
and across a save is a plain field; everything local is `private transient`
(`clock.zs:63-71`), and `OnRegister` — which also runs on load, *after* the synced fields are
read back — touches only transient state (`clock.zs:112-120`). `WorldLoaded` returns early on
`e.IsSaveGame` (`clock.zs:124`) so it does not stomp the restored clock. The handoff records
the earlier version wiping it (`HANDOFF:80`).

### 2.8 Actors that do work in a `Tick()` override before `Super.Tick()` — a different
### problem from the audit's, and partly better

10_TIME §4.1 names this as `freezetics`' blind spot: the check lives in the C++ `Super`, so a
ZScript override that works before `Super.Tick()` is not slowed and you cannot reach it from
outside.

The mod is **not** exposed to that failure, because it never relies on the `Tick` early return.
It rewrites fields from outside, which lands regardless of override order. What it is exposed
to instead:

- **Custom timers in a `Tick()` override are not slowed.** The mod stretches `tics` only. An
  actor counting its own `int fuse--` each tic keeps real time. Inherent to both mechanisms; no
  script fix exists.
- **A `Tick()` override that writes `Vel` every tic is *seen* as a push and scaled**
  (`clock.zs:511-520`). That is the right outcome and it is the one case where the mod beats
  `freezetics` outright: custom per-tic physics gets slowed, where `freezetics` would leave it
  at full rate. The sibling mods rely on this — `Grapple/zscript/rsqgr/util.zs:84-89`
  documents it explicitly ("a velocity GIVEN to a monster is a push, and RSQ_SlowMo's
  Impulses() scales a push by the factor itself — so a velocity is given at full-time size and
  never pre-scaled, or it would be scaled twice").
- **`Impulses` is a heuristic with a 0.5 u/tic deadband** (`clock.zs:517`). Pushes smaller
  than that are not slowed. At 35 Hz, 0.5 u/tic ≈ 0.5 m/s, so it only misses very gentle
  nudges. Acceptable, and a tighter gate would start mis-reading engine noise.

### 2.9 Where else the mod is already right (credit, not fixes)

- **Double-listing guard.** An actor spawned before a scan could be listed twice, each copy
  taking the other's stretch (`clock.zs:323-326`, `HANDOFF:34-36`). Found and fixed by the mod.
- **A script's own change to `Gravity`/`Speed` is adopted as the new base, not overwritten**
  (`clock.zs:429-430`). Brutal Doom's gibs set gravity in their first state; without this the
  mod would fight them forever.
- **Gravity is written regardless of `NOGRAVITY`** (`clock.zs:457-459`), because dying clears
  the flag and a corpse must fall at f². Subtle, and correct.
- **A push *up* off the floor is a push, not a landing** (`clock.zs:513-515`) — the handoff
  records the bug this fixed: a quarter-speed leap under f² gravity went 16× higher
  (`HANDOFF:35-36`).
- **Kills are credited from `WorldThingDamaged.DamageSource`, not `WorldThingDied.Inflictor`**
  (`clock.zs:582-605`), because a hitscan's inflictor is its puff and points at nobody unless
  `+PUFFGETSOWNER`. Measured: a pistol kill earned nothing before. This is a real engine
  subtlety the audit does not mention at all.
- **Gather is O(n) by construction, not O(n²).** The list is kept in thinker-iteration order so
  the sweep walks `act` alongside the iterator, one compare per listed actor
  (`clock.zs:351-383`), with a measured table (`HANDOFF:20-24`). A `Map` was measured and
  rejected because ZScript maps take only `int`/`String` keys and an `Actor` has no number
  (`HANDOFF:27-29`). This is better engineering than anything 10_TIME §6 proposes.

---

## 3. Would `freezetics` be better? (Q3)

**No. Straight answer: it would be a regression, and migrating would cost L for a worse
result.** Three independent reasons:

1. **It strobes.** 10_TIME §5.2 proves it structurally: `Prev` is re-snapshotted for every
   actor at the top of every tic (`src/p_tick.cpp:151-162` — verified, the
   `ClearInterpolation()` loop), so a `freezetics` actor moves its full one-tic distance on its
   one live tic and the renderer lerps that whole step across a single tic of frame time.
   ~8.75 Hz judder at factor 4. The mod exists in its current shape *because* of this, and says
   so at `clock.zs:4-8`. The audit's own §5.2 recommends the mod's approach for "the few actors
   the player is looking at"; in a VR mod where the player can lean in and stare at anything,
   that set is "everything in range".
2. **It cannot ease.** `freezetics` gives 1/(N+1) and nothing between (10_TIME §2.6). The mod's
   `rsm_depth` is a float, and `rsm_ramp_in` / `rsm_ramp_out` cross the whole range smoothly
   over 0.15 s / 0.40 s (`clock.zs:243-253`), with `RSQ_Accessibility` able to put a *floor*
   under both so the world never lurches (`clock.zs:94-97`). A duty cycle can only step
   1 → 1/2 → 1/3. **Replacing this mechanism would delete a shipped accessibility feature.**
   The audit does not raise this and should have.
3. **Its only real advantage does not apply.** `freezetics`' selling point in 10_TIME §4.1 is
   that "doing nothing restores it" — no stored originals, no leak. True, and the mod does pay
   that cost: nine parallel arrays and a `Drop` path. But the mod's restore is already proven
   correct across level change (§2.6), save/load (§2.7) and actor death
   (`clock.zs:419-423`), and it is the price of the two advantages above.

**Cost of migrating, honestly:** of the 759 lines in `clock.zs`, roughly 220 are the mechanism
(`Consider`, `Gather`, `Process`, `Impulses`, `Drop`, `FactorFor`) and would be replaced by
about 15. The other ~540 — the clock and ramps, the rival check, the service, adrenaline, the
reload detector, the gesture, haptics, sound follow — are mechanism-independent and would
survive. So it is not a 759-line rewrite; it is a ~220-line one, call it **cost M**. The
reason not to do it is not cost. **It is that the result would judder and could not ease.**

**The one place `freezetics` *is* worth having in this mod:** as a cheap tier for actors far
from the player, where strobe is invisible and the per-tic cost of the current mechanism is
pure waste. See §7.

---

## 4. The strobe (Q4)

**Verdict: ballistic motion is smooth. Walking monsters still strobe, at a LOWER frequency
than `freezetics` would — about 2.9 Hz at the default depth. This is the one genuine quality
problem left in the mod and it is unaddressed.**

**Why ballistics are fine.** A projectile's `Vel` is scaled (`clock.zs:454`), so it moves a
small distance **every** tic. `Prev` is snapshotted before thinking
(`src/p_tick.cpp:151-162`), so the renderer lerps each small step across a full tic of frame
time. That is 10_TIME §5.2's "per-tic smooth" case, exactly. Falls are smooth too (f² gravity).
Measured: `DoomImpBall` 10 → 2.5 units/tic (`HANDOFF:48`).

**Why walkers are not.** A monster does not move by `Vel`. `P_Move` performs one `TryMove` of
`actor->Speed` units (`src/playsim/p_enemy.cpp:502`), called once per state entry from
`A_Chase`. The mod stretches the state and leaves `Speed` alone. So at the default
`rsm_depth 0.25`, the measured imp (`HANDOFF:47`) goes from *0.78 units every 3 tics* to
**2.34 units every 12 tics**. Step, hold, hold, …, hold. That is a **~2.9 Hz** judder against
`freezetics`' 8.75 Hz — one third the frequency, four times the step. By 10_TIME §5.2's own
reasoning, **worse**.

**The cause is the same one §5.2 names,** and I can restate it exactly for this mechanism:
the engine applies monster locomotion in one discrete `TryMove` per `A_Chase`, and the mod's
only lever on `A_Chase`'s *rate* is state duration. It cannot make a monster take a smaller
step more often, because taking a step more often means running `A_Chase` more often, which
means un-stretching the state, which desynchronises the animation. **The two cheap options are
mutually exclusive and the mod picked the right one** (sync over smoothness).

**Fix tier, from 10_TIME §5.2: the fourth tier — the puppeted actor. BUILDABLE, cost M.**
Not the `Vel`+`Speed`+`tics` tier, which the mod is already at and which does not reach
`P_Move`. Concretely, for the handful of monsters nearest the player:

- Run a controller **after** the actors tick. `RunThinkers` walks statnums ascending
  (`src/playsim/dthinker.cpp:145-149`), so a `Thinker` at `STAT_BOT` (104) ticks after
  `STAT_DEFAULT` (100) — `wadsrc/static/zscript/doombase.zs:178-182`. `WorldTick` (which runs
  *before* `RunThinkers`, `src/p_tick.cpp:171` vs `:173` — verified) cannot see the jump.
- On the tic `A_Chase` fires, the controller observes the full-size step, `SetOrigin`s the
  monster back to a point `1/stretch` of the way along it, and pays out the remainder over the
  following tics. `SetOrigin(pos, true)` keeps interpolation (00_CAPABILITIES §7.12), so each
  small payout renders smoothly.
- Restrict it by count and distance: the nearest N (4? 8?) monsters inside, say, 512 units.
  Everything further away keeps today's behaviour, where 2.9 Hz at 20 m is not readable.

**Cost M, and it is fiddly** — you are second-guessing `TryMove`'s collision result, so a
monster that was blocked must not be pushed back into the wall. Budget a day and a half, and
treat it as optional polish, not a defect fix. The honest alternative is **to tune around it**:
raise `rsm_depth` toward 0.4 (a 7-tic state instead of 12 is ~5 Hz) or accept it. **The owner
has not played this on a headset yet** (`HANDOFF:91`), so the first move is to look at it at
0.25, 0.4 and 0.5 and decide whether it is worth cost M at all. 10_TIME §8 open question 2
says the same thing: the judder threshold is a measurement with a human in it.

---

## 5. Dodgeable bullets (Q5)

**The mod already does this, and does it correctly. It is shipped and on by default.**

`RSQSM_Shots.WorldHitscanPreFired` (`SlowMo/zscript/rsqsm/round.zs:20-71`) returns `true` and
spawns an `RSQSM_Round` on the same line. Verified against the engine: `P_LineAttack` calls
`WorldHitscanPreFired` as its **first statement** and returns `nullptr` on true —
`src/playsim/p_map.cpp:4718-4721`. 10_TIME §4.4 and the mod found the same door.

**The muzzle reconstruction is exact.** This is the part most likely to be subtly wrong and it
is not:

| Engine | Mod |
|---|---|
| `direction = { pc·cos(angle), pc·sin(angle), −sin(pitch) }` (`p_map.cpp:4752-4753`) | `dir = (cos(p)·cos(a), cos(p)·sin(a), −sin(p))` (`round.zs:36`) |
| `shootz = Center() − Floorclip + AttackOffset()`, non-player `AttackOffset` = `8 + offset` (`p_map.cpp:4754`, `actorinlines.h:92-100`), then `+= sz` (`:4853`) | `from.z = pos.z + height·0.5 − Floorclip + 8 + AttackZ` (`round.zs:40`) |
| `LAF_OVERRIDEZ`: `shootz = Z()`, then `+= sz` (`p_map.cpp:4850-4853`) | `from.z = pos.z + AttackZ` (`round.zs:39`) |
| non-player offsets: `Vec2OffsetZ(of·c + os·s, of·s − os·c, shootz)` (`p_map.cpp:4911-4915`) | `from.xy += (cos a, sin a)·of + (cos(a−90), sin(a−90))·os` (`round.zs:41`) — and `cos(a−90)=sin a`, `sin(a−90)=−cos a`, so **the same two components** |

The only divergence is portals: the engine uses `Vec2OffsetZ`, the mod adds raw. A shot whose
muzzle offset crosses a line portal would be placed wrong. Negligible, and worth one line of
comment rather than a fix.

**What else it gets right that 10_TIME only gestures at:**
- The audit's §4.4 hazard "returning true suppresses the puff, the decal and the sound — spawn
  them yourself" is handled: `Impact()` spawns the shot's own puff and a `BulletChip` decal
  (`round.zs:208-215`).
- **The puff's contribution to the hit is carried across** — damage type override plus
  `THRUGHOST`/`SPECTRAL`/`FOILINVUL`/`PAINLESS`/`FORCEPAIN` (`round.zs:52-63`), mirroring
  `P_LineAttack`'s own puff-defaults handling (`src/playsim/p_map.cpp:4806-4858`). The audit
  does not mention this at all and it is the difference between a round that behaves like the
  shot and one that does not.
- **The hit is dealt in `SpecialMissileHit`, deliberately bypassing the engine's projectile
  species rules** (`round.zs:152-166`), so a zombieman's round that strikes another zombieman
  wounds him and starts infighting — which is what a *hitscan* does and a projectile does not.
  The handoff records measuring this both ways (`HANDOFF:53-58`). **This is a correctness
  insight the audit missed entirely:** converting a hitscan to a projectile silently changes
  friendly-fire behaviour, and 10_TIME §4.4 does not warn about it.
- Test traces, melee and absolute-position attacks are excluded (`round.zs:27-30`); only
  shooters inside the slowed radius convert (`round.zs:32-33`).
- Fast small projectiles — Brutal Doom's bullet-as-projectile — are slowed *further*, to the
  same round speed, so they are dodgeable too (`clock.zs:399-404`). Measured: BD22
  `ZombieManTracer` speed 50 → 12.5 u/tic (`HANDOFF:59-60`).
- Near misses are detected by closest approach to each player's head, with pixelstretch applied
  to Z (`round.zs:180-203`) — which is 00_CAPABILITIES §1.2a's vertical-units rule, applied
  correctly. Whiz, crack, both-hand rumble, adrenaline credit.

**So Q5's premise is wrong in this mod's favour.** The headline promise is not partial; it is
real and measured (a 40-unit sidestep 3 tics after the shot missed in 3 of 3 runs,
`HANDOFF:51-53`). **Nothing to recommend here.** Both the audit and the mod found
`WorldHitscanPreFired`; the mod got there first and went further.

One small thing the mod does *not* do, which the player's shots half-cover: 10_TIME §4.4
suggests `+NOTIMEFREEZE` on the round. Irrelevant here — the mod never freezes the level, and
the round is excluded from being double-slowed by the `rsqsm_round` name exception at
`clock.zs:293` plus the `preFactor` handshake at `round.zs:66-69` / `clock.zs:330-331`, which
means **a round leaves the muzzle already slowed instead of flying one tic at full speed**.
That is a nicer solution than the audit's.

---

## 6. Selling it (Q6)

### 6.1 Post-process grade — ABSENT. This is the cheapest win left. BUILDABLE, cost M

**There is no GLDEFS lump in the pk3 at all.** The file list is `CVARINFO.txt`, `MAPINFO.txt`,
`MENUDEF.txt`, `MODELDEF.txt`, `KEYCONF.txt`, `SNDINFO.txt`, `zscript.txt`, `CREDITS.txt` —
no GLDEFS, no shaders directory. So the entirety of 10_TIME §4.5's visual grade is unbuilt.

Everything needed is verified present:
- GLDEFS parser accepts `HardwareShader PostProcess <target>` —
  `src/r_data/gldefs.cpp:1514-1529`, with `beforebloom` / `scene` / `screen` validated at
  `:1526-1529`.
- `PPShader.SetEnabled(name, bool)` and `SetUniform1f(name, uniform, value)` are
  `clearscope static` — `wadsrc/static/zscript/engine/ppshader.zs:3-4`.
- The clock already has the exact value to drive it: `scale`, eased, every tic
  (`clock.zs:46`), and already exposes it in permille through the service
  (`clock.zs:756`).

So it is: one `.fp` lump, one GLDEFS block, and two lines in `WorldTick`
(`SetEnabled` on the `Active()` edge, `SetUniform1f("...", "amount", 1.0 - scale)`).
**Honour 10_TIME §4.5's VR caution** — desaturate, a slight push, a mild contrast curve; no
chromatic aberration, no vignette, no radial warp, all of which break stereo fusion. And
10_TIME §8 open question 1 is still open: nobody has traced a custom post-process shader
through `OpenXRDeviceEyePose::submitFrame` to confirm per-eye application. **Test that before
building it**, because the mod's target is a Quest.

Cost **M**, mostly because a grade that looks right takes iteration, not because the plumbing
is hard.

### 6.2 Sound — the mod already does the audit's recommendation, with one bug

`Repitch` (`clock.zs:538-543`) calls `A_SoundPitch(ch, pitch)` on each slowed actor, every 4th
tic (`clock.zs:414`, `:482`), with the pitch tracking `want` — so it is not just a transition
sweep, it tracks the ramp continuously. This is 10_TIME §4.5's "the one that sells the
transition", and both found it.

**And it closes 10_TIME §8 open question 3.** That question asks whether `A_SoundPitch`, being
an `action` function (`actor.zs:1202`), can be called on a non-self actor from an
`EventHandler`. **It can.** When an action function is called on a different object, the
compiler passes self as `stateowner` and **`nullptr` as `stateinfo`** —
`src/common/scripting/backend/codegen.cpp:10136-10141`. The thunk's only use of `stateinfo` is
behind `ACTION_CALL_FROM_PSPRITE()`, which is
`(self->player && stateinfo != nullptr && ...)` — `src/common/scripting/vm/vm.h:772`. With
`stateinfo` null that branch is skipped and it falls straight through to
`S_ChangeActorSoundPitch(self, channel, pitch)` (`src/scripting/vmthunks_actors.cpp:165-181`).
**Safe, answered, no longer an open question.**

**BUG, cost S: the loop starts at channel 1** (`clock.zs:541`: `for (int ch = 1; ch <= 7; ch++)`),
so **nothing played on `CHAN_AUTO` is ever repitched.** `CHAN_AUTO` is 0
(`wadsrc/static/zscript/engine/base.zs:6`) and sounds started on it *stay on channel 0* — the
engine just sets `CHANF_OVERLAP` and lets them play there
(`src/common/audio/sound/s_sound.cpp:536-542`). `SoundEngine::ChangeSoundPitch` matches on
`chan->EntChannel == channel` (`src/common/audio/sound/s_sound.cpp:1057-1068`), so channel 0 is
reachable and would retune every overlapping sound on it. The mitigating fact is that
`A_StartSound`'s default slot is `CHAN_BODY` (4), not `CHAN_AUTO` —
`wadsrc/static/zscript/actors/actor.zs:1199` — so the common case is covered; but explicit
`CHAN_AUTO` calls are everywhere in stock content (e.g. `actor.zs:1459-1463` land/grunt,
`wadsrc/static/zscript/actors/shared/spark.zs:16`). **Fix: start at 0.** One character.

**No global pitch bus** — confirmed, 10_TIME §4.5 is right; the only music control is
`S_PauseSound` / `S_ResumeSound` (`doombase.zs:137-138`). The mod documents that as a limit
rather than faking it (`SlowMo/README.md:63`), which is the honest call.

### 6.3 Particles, casings, debris

The mod spawns particles for the missile wake (`clock.zs:546-560`) and the player's own hitscan
trail (`round.zs:75-101`), both via `level.SpawnParticle` with zero velocity and a lifetime —
so there is nothing to slow and nothing is wrong.

10_TIME §4.5's cheapest high-value shot — **`VisualThinker` casings without `SPF_NOTIMEFREEZE`
hang in the air at zero cost** — **does not apply to this mod**, and the audit should say so.
It depends on §4.2's global `Level.SetFrozen`, which this mod deliberately does not use. With
no level freeze, a `VisualThinker` is never frozen (`visualthinker.zs:34` →
`src/playsim/p_effect.cpp:1282-1290` tests the level freeze) and keeps full velocity.

**What *would* work here, and is the cheap add:** give the mod's own ejecta a `VisualThinker`
with a `Vel` the clock scales — `VisualThinker` has `Pos`/`Prev`/`Vel`
(`wadsrc/static/zperipheral/visualthinker.zs:1-45`) so it interpolates, and it is far cheaper
than an `Actor`. But note `freezetics` does not exist on non-`Actor` thinkers
(it is on `AActor` only, `src/playsim/actor.h:1284`), and neither does the mod's tracking — the
mod's `ThinkerIterator.Create("Actor")` will never see a `VisualThinker`. So a
`VisualThinker` tier needs its own small sweep or its own cooperation. **Cost S-to-M, and it is
optional.** The mod's honest README limit ("engine particles keep their speed",
`SlowMo/README.md:62`) is the right thing to say until someone builds it.

---

## 7. Cost (Q7)

**10_TIME §6's structural point is INVERTED for this mod, and this is the most important thing
in this report after §4.**

The audit says: *"Slowing actors makes the tic cheaper, not more expensive. The only new cost
is the iteration."* That is true **of `freezetics`**, because a frozen actor returns from
`AActor::Tick` at `src/playsim/p_mobj.cpp:3948-3952` before the inventory `DoEffect` loop,
`P_XYMovement`/`P_ZMovement`, the blockmap relink and the state machine.

**This mod skips none of that.** Every slowed actor runs a *full* `Tick` every tic — that is the
whole point, it is how motion stays smooth — **plus** the mod's own per-actor work in
`WorldTick`. So slow motion here is strictly **more** expensive than full speed, by:

| Per slowed actor, per tic | Where |
|---|---|
| `FactorFor` → `NearAPlayer`: a `MAXPLAYERS` loop with `Vec3Diff` + dot | `clock.zs:393-405`, `:304-312` |
| two tolerance comparisons against stored bases | `clock.zs:429-430` |
| `Impulses`: `GetFriction` (a native touching-sectorlist walk), two vector ops | `clock.zs:499-521` |
| the three-case `tics` bookkeeping | `clock.zs:435-451` |
| a second `GetFriction` for the ground-slide term | `clock.zs:476-481` |
| every 4th tic: `Repitch` — 7 × `IsActorPlayingSound` + up to 7 `A_SoundPitch` | `clock.zs:482`, `:538-543` |
| nine parallel-array accesses, and nine `Array.Delete` on drop | `clock.zs:523-536` |

**Plus the sweep**, which is the cost the audit *did* anticipate and which the mod has already
optimised hard: `Gather` every 17 tics, measured at **2.35 ms at 3543 listed actors** and
**3.05 ms at the 4000 cap** after the O(n) rework (`HANDOFF:20-24`), down from 4.02 / 6.26 ms.
The floor with no membership test at all is 2.39 ms, so that work is essentially done.

**Against a 35 Hz tic budget of 28.6 ms, a 3.05 ms gather plus a `Process` pass over up to 4000
entries is the real risk, and it is the one thing nobody has measured.** The mod's own handoff
says so (`HANDOFF:93-94`): *"Cost on the Quest CPU with a big Brutal Doom fight is not
measured."*

**What to measure, concretely** (all verified present in this engine):
- `profilethinkers -t 20` — per-class total tick time, descending, top 20;
  `src/playsim/dthinker.cpp:833-870`. Run it on a Brutal Doom gore map with `rsm_off`, then
  `rsm_on`. **The number that decides it is whether `RSQSM_Clock` is in the top five.**
- `stat think` — total think time per tic as one figure; `src/playsim/dthinker.cpp:1054`. Watch
  it across the `rsm_on` / `rsm_off` transition. Unlike the `freezetics` design, **expect this
  to go UP**, and know by how much.
- `profilethinkers -#` — call counts.
- And 10_TIME §6's closing note still governs: in VR the figure that actually decides it is
  dropped frames at the headset refresh, not playsim ms. But here the playsim genuinely might
  be the problem, which is not true of the audit's design — so measure `stat think` properly
  rather than once to rule it out.

**The tell that an actor's `Tick()` override is doing work before `Super.Tick()`** — 10_TIME
§6's diagnostic — **does not transfer to this mod**, and this is worth stating plainly so
nobody runs the wrong test. The audit's tell is "*if a slowed actor's call count does not drop,
it has a `Tick()` override doing work before `Super.Tick()`*". That works because `freezetics`
is *supposed* to drop the call count. **Under this mod no call count ever drops** — every
slowed actor ticks every tic by design. The equivalent tell here is different: compare the
actor's **`tics` field against its state's defined duration**. If an actor's `tics` do not grow
by `1/f` on state entry while the clock is at `f`, its state machine is not being driven
through `AActor::Tick`'s normal path and the mod has no hold on it. The second tell is
behavioural: an actor that keeps acting at full rate while visibly animating slowly is running
its own counters in a `Tick()` override, and no amount of `tics` stretching will reach it
(§2.8).

**The cost-reducing change worth proposing (BUILDABLE, cost S):** a distance tier. Today every
actor inside `rsqsm_radius` (3000 u — about 88 m) gets the full treatment. Strobe is unreadable
past ~15 m, so actors beyond, say, 1000 units could be slowed with `freezetics` instead
(`actor.zs:265`, one field write, and the engine then *skips* their tick entirely —
`p_mobj.cpp:3948-3952`), reserving the expensive path for the near field. That is the audit's
mechanism used where it is actually better, inside the mod's own mechanism, and it turns the
far field from "more expensive than normal" into "much cheaper than normal". It needs a second
tracking array and a careful hand-off when an actor crosses the boundary — hence S, not trivial.
**This is the one place `freezetics` belongs in this mod.**

---

## 8. Interaction (Q8)

**The service is clean and already load-bearing across the series.** `RSQSM_Service`
(`clock.zs:744-759`) is a `Service`, found by name so nothing links:

| request | answer | mod line |
|---|---|---|
| `adrenaline.add` (intArg) | adds, returns the meter 0..100 | `clock.zs:750` |
| `adrenaline` | the meter | `clock.zs:751` |
| `slowmo.active` / `active` | 1 while slowed or easing and not stood down | `clock.zs:752`, `:755` |
| `scale` | the world's speed in permille | `clock.zs:756` |
| unknown | −1 | `clock.zs:757` |

Consumers I verified, all by name, none linked:
- `Miniboss/zscript/rsqmb/handler.zs:94`, `:140` — **the thrall kill.** It opens a window
  (`rsqmb_slow` s, chained by `rsqmb_slow_chain`, capped at `rsqmb_slow_max`) and re-sends
  `netevent rsm_pulse <ms> <depth>` **once per second while the window lasts**, relying on
  SlowMo keeping the longest hold. That contract holds: `Pulse` takes
  `pulseUntil = max(pulseUntil, holdUntil)` and `pulseDepth = min(pulseDepth, depth)`
  (`clock.zs:215-224`), clamps `ms` to 30..5000 and depth to 0.05..1.0 (`clock.zs:213-214`) so
  no caller can freeze the world for good, and lets a held toggle outrank a pulse
  (`clock.zs:198`, `:211`). **Correct, and the re-send pattern is the right way to use it.**
- `Score/zscript/rsscore/handler.zs:588` — a big-moment kill sends one `rsm_pulse 600 35`,
  only from the earning player's machine, with a cooldown. `Score/.../hook.zs:76-77` then
  refuses to pay Bullet Time for a kill inside the mod's *own* pulse window
  (`tr.slowmoUntil`), so the bonus cannot pay for slow motion it caused itself. That is a
  genuinely careful piece of wiring on Score's side and SlowMo's service supports it by
  answering honestly during the ramp.
- `Grapple/zscript/rsqgr/util.zs:84-100` — reads `scale` and **documents the `Impulses`
  contract**: a velocity given to a monster must be at full-time size and never pre-scaled, or
  the clock scales it twice; a position written directly must be scaled by hand. This is the
  interaction that would silently break a sibling mod, and it is written down.
- `Accessibility/zscript/rsqa_core.zs:155-157` — Sound Cues on the `slowmo.active` edge.
- `GunBonsaiOffhandVR/rsq/zscript/core.zs:117-121`, `up_melee.zs:258`, `up_suite.zs:234-250`.

**Does a slowed world break anything?** The guard is `RealTime()` (`clock.zs:277-301`): the
player, anything they own, and every class named `rsq*`/`ubqs*`/`ssq*` (except
`rsqsm_round*`) runs at full speed. So the grab's beam, the rig's props, the shield, the saber
and the grapple's own actors are exempt by construction. Two notes:

- **Melee at 35 Hz gets better, as the audit says** (10_TIME §4.8). A slowed target's position
  changes 4× less between hand samples, so a swept-segment test against it is nearly a test
  against a static point. The mod gets this for free and does not need to do anything.
- **The exemption is by class-name prefix, which is a convention, not a mechanism.** A future
  series mod whose classes are not named `RSQ*`/`UBQS*`/`SSQ*` will be slowed silently. That is
  a documentation risk, not a bug, and it is the one thing in the mod that would not pass the
  global rule "name a thing for what it does, not who asked for it" — but the alternative (a
  per-actor opt-out flag the engine does not have) is worse in a mod that must run on stock
  17.3. **Leave it; write it down where new series mods get started.** §8.1 is a shipped
  instance of it.
- Nothing in the series double-slows: the `preFactor` handshake (`round.zs:66-69` →
  `clock.zs:330-331`) and the `rsqsm_round` name exception stop the mod's own rounds being
  slowed twice; Score refuses to pay Bullet Time inside its own pulse window
  (`Score/zscript/rsscore/hook.zs:76`); and the rival check (`clock.zs:103-110`, `:122-147`)
  stands the whole mod down for Bullet Time X or RS_SlowMo, with a console explanation and an
  `rsm_yield 0` override.

### 8.1 The weapon wheel — a real hazard, and it is in the Wheel, not in SlowMo

`Wheel/zscript/wheel/zscript.zs:4045-4056` is `bulletTime(bool on)`. Opening the ring calls
`bulletTime(true)` (`:1115`), closing it calls `bulletTime(false)` (`:1222`, `:3059`), and when
`RSQSM_Service` is present it sends SlowMo's **absolute** events:

```
let sm = ServiceIterator.Find("RSQSM_Service");
if (sm.Next()) EventHandler.SendNetworkEvent(on ? "rsm_on" : "rsm_off");
else EventHandler.SendNetworkEvent("bt_activate");      // BulletTimeX's toggle
```

The long comment above it (`:4025-4044`) is about exactly this class of bug for BulletTimeX,
whose `bt_activate` is a *flip* and therefore cancels an already-running bullet time on open.
It states that the RSQ route avoids this because SlowMo's events "hold until told otherwise".
**Half right. The failure survives in the other direction.**

- `rsm_on` on open is harmless: `Aim(true, …)` just sets `on = true` (`clock.zs:197`), which is
  idempotent.
- `rsm_off` on close is **not**: `Aim(false, …)` sets `on = false` unconditionally. So **a
  player who toggles slow motion on himself, then opens and closes the wheel, loses his own
  slow motion** — and the guard `if (on == mBtOn) return;` (`:4047`) cannot catch it, because
  `mBtOn` records only what the Wheel did, not what SlowMo's state was beforehand. The Wheel
  flipped `mBtOn` to true on open (it had no idea slow motion was already running) and so
  believes the `off` on close is its own to send.

**Strictly better than the BulletTimeX path** (which breaks the whole wheel interaction rather
than the moment after it) and **not a stranding bug** — `rsm_off` ramps out through
`Process`, which restores every tracked actor before dropping it (§2.6), so nothing is left
slowed. It is a one-interaction annoyance with a cheap fix, **and the fix belongs in the
Wheel, not in `clock.zs`:** ask the service before acting. `RSQSM_Service` already answers
`active` and `scale` (`clock.zs:752-756`), so `bulletTime` can record "it was already on" on
open and skip the `off` on close. **Cost S, in `Wheel/zscript/wheel/zscript.zs`.** Worth noting
that SlowMo *could* instead expose a refcounted `rsm_hold` / `rsm_release` pair, which would fix
every caller at once rather than one — but that changes the published door RS_SlowMo defined
(`clock.zs:27-32`), so it is the owner's call, not mine.

**Second, smaller point: the Wheel's own actors are not exempt from being slowed.** SlowMo's
`RealTime()` test exempts classes whose name begins `rsq`/`ubqs`/`ssq` (`clock.zs:293`). The
Wheel's world actors are named `wr_*` — `wr_BBQuad`, `wr_BBStrip`, `wr_Beam`, `wr_Buzz`
(`Wheel/zscript/wheel/wr_q_shim.zs:73`, `:91`, `:763`, `:669`) — so **every billboard quad in
the ring is gathered and walked by the clock every tic while the wheel is up in slow motion.**
I checked whether that breaks anything and it does not: the quads are positioned purely by
`SetOrigin(at, true)` (`wr_q_shim.zs:636`, `:832`), never by `Vel`, and their state is
`WRBQ A -1` (`wr_q_shim.zs:89`) so `tics` is −1 and the mod's `a.tics > 0` guard
(`clock.zs:435`) skips them. Their `+NOTIMEFREEZE` (`wr_q_shim.zs:83`) is for the *level* freeze
and is irrelevant here. So: **a cost leak, not a correctness one** — dozens of inert actors in
the per-tic `Process` loop at the exact moment the player is also rendering a 3D ring. `wr_Buzz`
is the one to watch: it emits `A_QuakeEx` from a `Tick()` override, and a timed state on it
would be stretched 4× by the clock, lengthening a rumble. Cosmetic.

**Neither of these is a reason to change `clock.zs`.** The service it publishes is sufficient
to fix both from the Wheel's side, which is where the mistake is.

---

## 9. Label and cost table

| Piece | Label | Cost | Note |
|---|---|---|---|
| Per-actor `Vel`+`Gravity`+`Speed`+`tics` clock | **EXISTS (shipped, correct)** | — | `clock.zs:411-489`; 10_TIME §5.2's hero tier, applied broadly |
| Continuous easing of the factor | **EXISTS (shipped)** | — | `clock.zs:243-253`; `freezetics` cannot do this |
| Dodgeable rounds via `WorldHitscanPreFired` | **EXISTS (shipped, correct)** | — | `round.zs:20-71`; engine door `p_map.cpp:4718` |
| Puff flags / damage type carried to the hit | **EXISTS (shipped)** | — | `round.zs:52-63`; audit missed this |
| Hitscan-like infighting via `SpecialMissileHit` | **EXISTS (shipped)** | — | `round.zs:152-166`; audit missed this |
| `A_SoundPitch` follow on slowed actors | **EXISTS (shipped)** | — | `clock.zs:538-543`; closes 10_TIME §8 Q3 |
| Clean cross-mod service + `rsm_pulse` door | **EXISTS (shipped)** | — | `clock.zs:744-759`; 5+ consumers |
| O(n) gather | **EXISTS (shipped, measured)** | — | `clock.zs:351-383`, `HANDOFF:20-24` |
| `isFrozen()` guard in `Impulses` / friction | **BUILDABLE (bug fix)** | **S** | §2.3 |
| `+SKULLFLY` / `+NOFRICTION` in friction guards | **BUILDABLE (bug fix)** | **S** | §2.4 |
| `Repitch` from channel 0 | **BUILDABLE (bug fix)** | **S** | §6.2 |
| GLDEFS post-process grade driven by `scale` | **BUILDABLE** | **M** | §6.1; per-eye unverified (10_TIME §8 Q1) |
| `freezetics` tier for the far field | **BUILDABLE** | **S** | §7; the audit's mechanism, used where it wins |
| Smooth walking monsters (puppeted step payout) | **BUILDABLE** | **M** | §4; `STAT_BOT` controller + `SetOrigin(pos,true)` |
| `VisualThinker` ejecta that the clock slows | **BUILDABLE** | **S–M** | §6.3; needs its own sweep |
| Wheel's `rsm_off` on close cancelling a held slow-mo | **BUILDABLE (bug fix, in Wheel)** | **S** | §8.1; `Wheel/.../zscript.zs:4045-4056` |
| Doors, lifts, moving floors in slow motion | **WALL** | — | sector specials run outside the actor path; `HANDOFF:86` |
| Engine particles slowed | **WALL** | — | `README:62`; no per-particle time scale |
| Music pitch | **WALL** | — | `S_PauseSound` only, `doombase.zs:137-138` |
| Smooth motion without puppeting | **WALL** | — | per-actor interpolation stretch; 10_TIME §5.1/§5.2 names it |
| Scriptable haptics | **WALL** (worked around) | — | mod uses the `A_QuakeEx` back door, `clock.zs:685-723`; 00_CAPABILITIES §1.6a |
| Global time scale from script | **WALL** | — | `i_timescale` unsettable, `vmnatives.cpp:981-990` |

---

## 10. What I could not determine

Said plainly, because a guess here would be worse than the gap.

1. **Whether the walking strobe is objectionable in the headset.** Structural reasoning gives
   ~2.9 Hz at depth 0.25; whether that reads as broken is a measurement with a human in it, and
   the owner has not played it yet (`HANDOFF:91`). 10_TIME §8 Q2 says the same. **Look before
   spending cost M on §4.**
2. **Whether a custom post-process shader is applied per eye under OpenXR.** 10_TIME §8 Q1 is
   still open; I did not trace `submitFrame` either. Gates §6.1.
3. **Per-tic cost of `Process` on the Quest CPU.** The gather is measured; the per-actor pass
   is not, on any hardware. §7 names the two commands.
4. **`A_Tracer` homing under this mechanism.** 10_TIME §8 Q4 raises it for `freezetics`; it
   applies differently here. `A_Tracer` keys on world `maptime & 3` (`src/p_tick.cpp:118-120`),
   and under this mod the missile ticks every tic, so the homing *phase* is correct — but its
   turn rate per call is unchanged while its `Speed` is scaled, so a slowed Revenant missile
   should turn **more sharply per unit distance**. I could not prove by reading whether that
   looks wrong. Worth one look with a Revenant.
5. **Whether `Array<State>` restores correctly in practice.** The serialisation path is there
   (§2.7) and I traced it; I did not load a save taken mid-slow-motion.
