# 04 — RSQ_DistanceGrab, against the engine audit

**What this is:** the audit's findings (`surveys/00_CAPABILITIES.md`, `surveys/06_GRAB_INTERACTION.md`,
`surveys/01_MELEE.md` §3.4–3.5) held against the shipped mod, to find where the engine gives it
something it is not taking, and to record where it is already doing the right thing.

**Sources, exactly.** Mod paths are relative to the **fresh clone** of
`github.com/presidentkoopa/RSQ_UBQSeries`, HEAD `6a5c98c` (2026-10-30), at
`.../scratchpad/ubqs_gh/DistanceGrab`. Engine paths are relative to `.../scratchpad/qzd173`
(tag `qzd17.3`). Every mod citation below was read in that clone. No engine change is proposed
anywhere in this file; everything is ZScript and data.

> **The docs are two days behind the code, and it matters.** `docs/DECISIONS.md` ends at **D30
> (2026-10-27)**. A whole new system — `zscript/rsqdg/near.zs`, 377 lines, added to `zscript.txt`
> and dated by its own header and by `CVARINFO.txt:40-51` to **the owner, 10-29** — has no
> decision record, no HANDOFF entry and no mention in `PORT_SCOPE.md`. So where §3 and §9 below
> question a choice inside `near.zs`, there is no owner decision being re-litigated: there is no
> record of the choice at all. Everywhere a choice *is* explained in D1–D30 it is cited and left
> alone.

---

## 0. THE RECOMMENDATION, first

**Two changes, and the small one is the urgent one.**

**(a) The Headlock is a bullet shield but not a fireball shield, and one line causes it.
BUILDABLE / S.** `near.zs:324` sets `a.bTHRUACTORS = true` on a headlocked monster.
`PIT_CheckThing` tests the **OR of both actors'** `MF2_THRUACTORS` and returns "no collision"
at `src/playsim/p_map.cpp:1409-1410` — *before* the height gate, before the infighting gate and
before `P_DoMissileDamage`. So every imp fireball, revenant missile and cacodemon ball passes
straight through the monster you are holding in front of your face, on every map.

Hitscans, meanwhile, **do** stop at it: the trace mask is `MF_SHOOTABLE`
(`p_map.cpp:4919`), the trace's own actor test reads only `thing->flags & ActorMask`
(`src/playsim/p_trace.cpp:866`), and the skip filter consults the **puff's** `THRUACTORS`, not
the victim's (`p_map.cpp:4829` sets `TData.ThruActors` from `puffDefaults`, read at `:4690`).

That asymmetry is why this is silent: in a test room full of zombiemen the shield works
perfectly, and the first imp makes it look like a different bug. 06_GRAB §4.5 names
`NoInfighting` + `+FORCEINFIGHTING` (`p_map.cpp:1282`) as "the one leak"; here there are two,
and this one is in front of it and unconditional, which is why the narrower one has never shown.
Fix in §9.

**(b) A HELD object still has no hit test, and the mod advertises one. BUILDABLE / M.**
"Then carry them, throw them, and hit things with them" — it cannot. `hold.zs:179` makes the
carried object `bTHRUACTORS` for the whole hold, and nothing in the package tests a swing: a grep
for `swing|melee|bludgeon` over `zscript/` returns only the sag term (`hold.zs:432`), the
stand-down rule for a hand holding a melee weapon (`reach.zs:188`), and a service name
(`throw.zs:44`). Damage exists only for a **thrown** object (`throw.zs:472-535`). 01_MELEE §3.5
has the finished design and the mod already owns every input it needs. §7.

Third, much smaller: `feel.zs` is missing the third haptic route — `LineAttack` with
`LAF_NOINTERACT` — which is strictly cheaper than the `SpawnPlayerMissile` buzz it uses and makes
D15's whole compatibility problem disappear rather than working around it by name. **S.** §8.

Everything else the audit has to say about this mod, the mod already does — and in interpolation,
doors, units and haptics it does better than the audit's own recommendation.

---

## 1. What it does today

Seven systems, one `EventHandler` each, registered from `zscript.txt`, and the order is
load-bearing: `RSQDG_Aim` measures and publishes, `RSQDG_Grab` decides, the rest consume
(`aim.zs:1-15`, which records the desktop bug the ordering fixes — a catch test that only ever
saw last tic's position, ~23 units short of a window needing ≤ 1.0).

**The input and the lock.** The trigger is a pose, not a button: palm rolled out within
`rsqdg_palm_tol` of `rsqdg_palm_roll`, and not pointing at the floor (`reach.zs:106-124`). Read
fresh every tic, so it cannot stick — `grab.zs:28-36` names that as the real prize over the
netevent edge **D1** originally chose; the button path survives behind `rsqdg_button`, off by
default, with D1's lease as its backstop (`grab.zs:358-377`). A hand that is already somebody
else's gesture stands down **by evidence**: the rig's `rig.busy`, the saber's `saber.lit` per
**D26**, and both hands inside `rsqdg_steady_m` of each other because canting a steadied rifle
rolls both palms out (`reach.zs:141-190`). While armed, the cone's best candidate becomes the lock
and sticks (`grab.zs:406-416`), with **D27**'s optional dwell.

**The flick.** No velocity fields exist (00_CAPABILITIES §1.3), so the launch gesture is a **wrist
pitch rate** — `rsqdg_flick_deg` degrees per tic upward, peak over a 7-tic (~200 ms) window
(`aim.zs:264-284`), fired at `grab.zs:269-287`. **D3** chose wrist-only and dropped the
position-delta mode as frame-broken. `aim.zs:237-263` gives the reason it is pitch: stock writes
`AttackPitch` with no yaw term, so a snap turn cannot contaminate it — turn-immune at source
rather than by correction. The frame handling around it is the most careful thing in the package
(`aim.zs:62-143`): the tracking-space yaw is *inferred* from the one signature that separates a
head turn from a stick turn, and the sampler goes quiet for two tics after a head turn because the
engine reports both hands rotated the other way for exactly one tic.

**The pull.** Per **D9**/**D29**/**D30** there are three selectable flights; the default (mode 2)
is WorldHands' written path, with Steer (bounded acceleration toward the live palm, authority
= `PullShare`, `flight.zs:667-699`) and Lob (a ballistic solution re-solved onto the palm every
tic) alongside. Launch thins the radius to `rsqdg_flight_radius` (`flight.zs:237-238` — a pickup's
radius is a trigger, not a hull) and hops it a unit off the floor, because Doom's ~9 %/tic ground
friction beat a barrel's 0.14 share (`flight.zs:257-309`, **D18**). Timeout is per pull:
`max(menu, 2 × travel + 1 s)` (`flight.zs:274-282`).

**The catch.** Arrival parks the object on the palm and holds it `rsqdg_catch_hold` tics
(`flight.zs:617-645`) — the §11.1 dwell, because a state that lives zero tics cannot be observed.
An open palm takes it; an `Inventory` is offered to `CallTryPickup` first under
`rsqdg_catch_takes` (`flight.zs:368-397`).

**The hold.** A spring, not a teleport (**D22**): `vel = bodyVel + clamp(armVel × share + err × k)`
(`hold.zs:520-529`), target a stand-off ahead of the palm plus the object's half-size
(`hold.zs:399-417`), `k` and `vmax` scaled by mass share, the body's own motion fed forward in
full and only the arm's scaled — the fix for a barrel lagging 0.8 m while walking. Refusal is
measured as **strain** and ends the hold 18 tics past `rsqdg_hold_break` (`hold.zs:504-518`). A bad
pose freezes the object dead rather than chasing the collapsed hand field (`hold.zs:471-481`).
Held things turn with the wrist relative to the catch (**D30**, `hold.zs:243-274`), sag by weight
(`hold.zs:427-434`), and burden the player through `Inventory.GetSpeedFactor` — the only hook that
works in VR (**D24**, `hold.zs:543-566`).

**The throw.** Release decides by speed (**D4**): `arm/(arm+kg)` of the hand's peak velocity
(`hold.zs:320-343`), the thrower's own velocity added back only on a real throw. Landing is a
speed-drop watcher plus a per-tic overlap test for non-solid pickups (`throw.zs:537-622`), damage
`½mv²/rsqdg_throw_j` with the vertical term scaled by `sqrt(9.81/g_doom)` (`throw.zs:427-450`), a
MASSDEF `impact explode` row to set a barrel off, and a ten-second ledger for the scorer
(`throw.zs:69-86`). Everything released enters the air list: 0.27 g added back, drag, flutter
(`throw.zs:156-187`).

**The door lift.** Real continuous control — §6.

**NEW since the snapshot, and undocumented: the grip up close** (`near.zs`, "the owner, 10-29").
Three acts, all default **on**, all reading the grip as **raw key scans** through `InputProcess`
with the press eaten when one of them takes it (`near.zs:61-86`; `rsqdg_keys_main` "446",
`rsqdg_keys_off` "443 436", `CVARINFO.txt:50-51`), holsters first (`near.zs:179`):

- **Swap hands** — both controllers within `rsqdg_swap_m` and both wrists rolled past
  `rsqdg_swap_roll`: either grip swaps the two hands' weapons (`near.zs:110-116`, `:266-313`).
- **Walk-up grab** — the grip with the palm within `rsqdg_near_m` (0.30 m) of a thing takes it
  straight into the hand, through the same `RSQDG_Hold.Take` as a caught pull
  (`near.zs:214-230`); `grab.zs:388-389` now defers the release to the grip rather than the palm
  for anything the walk-up took.
- **Headlock** — the **off hand's** grip on a living monster no heavier than `rsqdg_collar_mass`
  (400, `Actor.Mass`: imp 100, demon 400, baron 1000) holds it frozen in front of the hand for
  `rsqdg_collar_secs` (3 s) so the main hand can punch it, and tells `RS_ScoreService`
  (`near.zs:316-371`). **This is a human shield**, and §3 and §9 are now about live code.

**Also changed since the snapshot** (`policy.zs`, `CVARINFO.txt:23-39`, `MENUDEF.txt:104-120`):
every pickup family is its own switch, the `+1` bonuses went from a flat denial to a rule at the
lowest weight, and `rsqdg_corpses` and `rsqdg_powerups` are now **on** by default ("the owner,
10-29: expanded pickup options").

**Mass.** MASSDEF kilograms with a volume guess behind it (`mass.zs:42-62`),
`PullShare = k/(k+kg)` (`mass.zs:201-205`, **D17**), `Heft` on a log scale for rhythm and
amplitude (`mass.zs:210-215`). Guessed from the **class defaults**, never the live box, because
the flight and the carry both thin it (`mass.zs:175-180`).

---

## 2. Interpolation, the feel question. **Verdict: already right, and for a better reason than
the audit gives.**

06_GRAB §3 says "use `SetOrigin(pos, true)` to track a hand. Never `SetOrigin(pos, false)`, never
bare `Warp`." The mod does something stronger: **the carry writes no position at all.** It writes
`Vel` (`hold.zs:529`) and lets `P_XYMovement`/`P_ZMovement` move the object, which interpolates
for free and — the part the audit's own recommendation gives up — **collision-tests every step**.
`hold.zs:16-27` states the trade: "SetOrigin is a teleport: it puts the object through the wall you
hold it against, and there is nothing to push back." Pinning then costs no code, and the refusal
*becomes* the strain signal the rumble plays. That is the direct answer to 06_GRAB §3's own note
that `SetOrigin` does no collision test — verified at `src/playsim/p_maputl.cpp:563-571`: unlink →
`SetXYZ` → `LinkToWorld` → `P_FindFloorCeiling`, with `ClearInterpolation` only `if (!moving)` at
`:570`.

Every `SetOrigin` in the clone was checked. **There is no snap where a glide was wanted:**

| call | flag | correct? |
|---|---|---|
| `flight.zs:307` lift off the floor | `true` | yes |
| `flight.zs:626` park on the palm at arrival | `true` | yes |
| `door.zs:373` the door voice actor | `true` | yes (silent either way) |
| `laser.zs:229` / `:232` a laser piece | `false` when re-shown, `true` otherwise | **deliberate** — `laser.zs:222-224`: a piece that was hidden must teleport, "otherwise it streaks in from across the room" |
| `glow.zs:74` the glow twin following its host | `true` | yes |
| `glow.zs:89` a NEW host | `false` | **deliberate**, same reason (`glow.zs:86`) |
| `feel.zs:301` the rumble spot | `false` | moot: `RenderStyle "None"`, never drawn (`feel.zs:192`) |

`SetXYZ` and bare `Warp`: **zero hits.** `+NOINTERACTION` appears only on the mod's own inert or
un-rendered helper actors (`door.zs:414`, `feel.zs:161`, `:186`, `glow.zs:23`, `laser.zs:41`,
`:65`, `:94`) and twice as a *refusal* reason (`policy.zs:325`, `near.zs:148`) — which is exactly
the right use of the trap.

**One new position write, and it is also fine.** `near.zs:358-359` moves a headlocked monster with
`TryMove` (collision-tested, interpolating) then `SetZ`. `SetZ` does not relink
(`src/playsim/actor.h:1540-1543`, `__Pos.Z = newz;` and nothing else) — which 06_GRAB §3 warns
about for `SetXYZ`, but the blockmap is 2D, so a Z-only write leaves no stale link. The ignored
`TryMove` return (`if (!a.TryMove(at.xy, true)) { }`) is the right behaviour too: a blocked move
leaves the monster where it is, and `near.zs:357` then breaks the hold once it is torn more than
1.5 m away.

**The mod found something the audit did not.** **D20**, measured by `tools/rigtest/leadsim.py`:
the audit's "one tic behind" floor is beatable by placing the hand end at `palm + v × rsqdg_lead`,
so the renderer's own `Prev + (Pos−Prev)·ticFrac` cancels the lag — 4.0/6.7 cm → 1.1/2.8 cm on a
slow sweep. That belongs back in 06_GRAB §5's "held objects render ~1 tic behind the hand" row as
a partial mitigation.

**Label: EXISTS (nothing to do). Cost: none.**

The one optional refinement: the lead trick is applied to the *drawing* but not to the carry. Using
it on the spring's **target** (`hold.zs:401`, `palm + armVel × rsqdg_lead`) is a one-line headset
A/B — **BUILDABLE / S**, a feel experiment, not a correctness fix.

---

## 3. Holding things still — **`freezetics` is unused, and in `near.zs` that is now a real miss**

Verified: `freezetics` is a writable `native uint` (`wadsrc/static/zscript/actors/actor.zs:265`)
and early-returns out of `AActor::Tick` (`src/playsim/p_mobj.cpp:3948-3952`) **before** states,
velocity and AI, while keeping the blockmap link. `+NOINTERACTION` forces `MF_NOBLOCKMAP`
(`src/playsim/p_mobj.cpp:3987-3992`) and takes the actor out of the world.

`grep -rn freezetics zscript/` is **empty**, and `+NOINTERACTION` is never set on a gameplay
object. For the **carry**, that is correct by design and the audit's prescription lands on nothing:
gravity is suppressed by `bNOGRAVITY` (`hold.zs:178`, restored at `:228`), and there is no AI to
stop because a distance-pulled object is never alive (`policy.zs:360-362`, **D6**).

**But `near.zs` now holds a living monster, and freezes it the other way.** `near.zs:361` sets
`a.tics = -1` every tic, with `a.Vel = (0,0,0)` at `:360`. That is a *state* freeze, not a Tick
freeze, and it is the weaker of the two tools on three counts:

1. **It does not stop `AActor::Tick`.** `freezetics` returns out of the function;
   `tics = -1` only stops the state counter from expiring, so the whole movement, Z, portal,
   crush and water block still runs every tic for the held monster. Nothing visibly breaks today
   because `Vel` is zeroed and `bNOGRAVITY` is set — i.e. the mod is hand-doing, less completely,
   what one field does outright.
2. **It is a latch, so it needs a restore path that can be missed.** `near.zs:339` has to undo it
   (`if (a.Health > 0 && a.tics < 0) a.tics = 1;`). `freezetics` is self-healing: stop writing it
   and the monster resumes within two tics, so no exit path can leave a permanently frozen
   monster. The mod states this principle itself, about a different field, at `hold.zs:541-542`:
   *"a flag set in one place and cleared in another is the bug that outlives the session."*
3. **The save path is thin.** `RSQDG_Near` is a plain `EventHandler`, so `collar[]`, `gripDown[]`,
   `cNoGrav[]` and `cThru[]` are serialized with the level — but unlike `door.zs:59-68`, which
   explicitly notices a load and releases, `near.zs` has no `WorldLoaded`/`OnRegister` handling at
   all. A save taken mid-headlock comes back with `gripDown` true and no key-up able to clear it
   (`keyState`/`eaten` are `transient ui`, so the release at `near.zs:72` compares equal and sends
   nothing). It is bounded by `collarEnd` (`near.zs:349`), so the monster frees itself within
   `rsqdg_collar_secs` — a 3-second oddity rather than a leak, but it is luck rather than design,
   and `freezetics` plus a `WorldLoaded` release removes both.

**The change:** in `TickCollars`, replace `a.tics = -1` with `a.freezetics = 2` re-asserted each
tic, drop the `tics` restore at `near.zs:339`, and add a `WorldLoaded` that calls `EndCollar`.
The monster stays linked and shootable — which is the whole point of a shield (§9) — and the
"frozen in its pain frame reads as stunned" behaviour `near.zs:351-352` wants is preserved, because
`freezetics` stops the state advancing just as surely.

**Label: BUILDABLE. Cost: S.** This is the audit's finding #11/#3 landing on real code.

The other use worth having is the hit-stop in §7 (01_MELEE §3.6: `victim.freezetics = 4`, ~114 ms)
— on the victim, never the player, for the reason 01_MELEE gives: hands are tracked outside the
playsim, so a frozen player is a player whose hands stop registering while the world keeps
rendering.

---

## 4. Throwing — mostly right; one audit idea rejected on the record, one worth taking

**The unit trap is handled.** 00_CAPABILITIES §1.2a's "forget the vertical term and every height
reads ~17 % short" is the first thing `reach.zs:262-291` deals with, with the stock sanity check
written out (56 × 1.2 / 34 = 1.98 m). `Metres()` multiplies Z by `PIXELSTRETCH` before dividing by
34, and every throw, flick, carry and reach threshold goes through it (`aim.zs:230`,
`hold.zs:340`, `throw.zs:436`, `near.zs:143`). **Already right, and documented better than the
survey.**

**`+SKULLFLY` / `Slam` is rejected, deliberately, and the reasons beat the audit's.** The callback
is real — `PIT_CheckThing` calls `CallSlam` at `src/playsim/p_map.cpp:1552-1557`, and `Slam` is
`virtual native bool Slam(Actor victim)` at `actor.zs:520`. `throw.zs:8-14` declines it and gives
four costs the audit does not weigh: `P_ExplodeMissile` jumps the actor to Death on the first line
it touches, so a thrown barrel goes off on a doorframe whatever the throw; the whole
`P_XYMovement` branch changes; the thrower is passed through and monsters stop treating it as a
target; and an object that *survives* the throw is left permanently mutated — fatal for a mod whose
entire discipline is "whoever borrows is the single owner of the restore". **D23** records the same
rejection for `+MISSILE`. **Do not re-litigate this.** The tracker is the better trade here.

**Where the audit still has something.** 06_GRAB §4.4 recommends **best-of-three 2-tic deltas**.
The mod takes the **peak of seven 1-tic deltas** (`aim.zs:219-235`). The window is comparable; the
sample is not. At 35 Hz a 1-tic delta carries twice the quantisation and jitter of a 2-tic one, and
taking the **maximum** of a noisy set is the one estimator that turns noise straight into bias — it
over-reads, and the harder the throw the more samples it has to find a lucky one in. The ring
already holds per-tic deltas, so a 2-tic delta is the sum of two adjacent entries: a handful of
lines in `PalmVelocity`, with `rsqdg_throw_scale` retuned down for the bias that comes out.
**BUILDABLE / S**, and a feel A/B rather than a bug — `aim.zs:209-212` argues correctly for a peak
over a window, and the owner has tuned the present numbers in a headset.

**One thing to watch rather than change.** The non-solid overlap test (`throw.zs:579-601`) is a
**point sample** at the object's position each tic, pad 0 (`throw.zs:587`). A clip at 15 m/s covers
~15 map units a tic against a lost soul's 32-unit box, so it holds today — but `rsqdg_throw_scale`
and the Heavy Hitter perk (+25 %/level to `rsqdg_arm_kg`, `reach.zs:349`) both push that number up.
§7's swept chord is the same fix for both, so doing §7 gets this free.

**Label: EXISTS (sound); `+SKULLFLY` a documented WALL-by-choice; the 2-tic delta BUILDABLE / S.**

---

## 5. Pickup — **already right, no silent failure**

06_GRAB §4.1's warning is real and verified: a pickup fires only when the **mover** has
`MF_PICKUP` and the other actor `MF_SPECIAL` (`src/playsim/p_map.cpp:1737-1743`), and a mod that
moves the item to the player is the wrong way round.

The mod never relies on proximity. It calls `Inventory.CallTryPickup(pmo)` explicitly — at the
catch (`flight.zs:384`) and now at the walk-up grip too (`near.zs:223`) — and reads **both** return
values, with the trap named at `flight.zs:372-376`: the returned `Actor` may not be the one
offered. In the other direction it **suppresses** accidental pickup: `bSPECIAL = false` for the
whole flight and the whole carry (`hold.zs:180`), restored only once the object fits, with 10 tics
of uncollectability after a throw so walking after your own throw does not collect it
(`hold.zs:347`, `throw.zs:192-215`). The settle list is the single owner of that backup from the
moment of release (`hold.zs:216-240`).

Two further cases are handled that the audit does not raise: an item that acquires an `Owner`
mid-flight ends the pull and hands back gravity, pass-through and size but **not** `SPECIAL`,
because `Inventory.BecomeItem` set that itself (`flight.zs:514-540` — and the comment records that
an earlier version of itself was wrong, which is the right way to leave a file); and a held item
that acquires an `Owner` drops out of the hold (`hold.zs:463-469`).

**Label: EXISTS. Cost: none. Nothing to do.**

---

## 6. Doors — **real continuous control, and past what the audit asked for**

06_GRAB §4.7 concludes there is no force input to a plane and names the route: `Line.Activate` /
`RemoteActivate`, `Level.ExecuteSpecial`, and continuous `Sector.MoveCeiling` for "the door moves
as far as I pulled it", with the warning that vanilla door thinkers will fight you.

`door.zs` is that design, built, plus three things the audit does not mention:

- **Continuous, per tic, from hand height.** `target = startCeil + (handZ − grabZ) × rsqdg_door_gain`
  (`door.zs:217`), applied as `sec.MoveCeiling(sp, dest, -1, dir, false)` capped at
  `SpeedOf(line) × 4 × (64/width)`, so a flick cannot throw it open and a wide door is heavier
  (`door.zs:278-280`, `:347-351`).
- **It claims the sector from the vanilla mover.** `TakeOver` (`door.zs:290-300`) destroys a `Door`
  thinker that has only just started rising — because `BT_USE` reaches `P_PlayerThink` before this
  handler hears of the grab, so a Use-press on a door has *always* already started it opening — and
  stops the orphaned sound sequence. That is precisely the fight 06_GRAB §4.7 predicted, won, with
  the conservative failure case (anything further along is left alone) chosen rather than
  stumbled into.
- **The commit point.** Let go past `rsqdg_door_commit` and the door's **own** special finishes it
  (`l.Activate(pmo, 0, SPAC_Use)`, `door.zs:315`) so it waits and closes as that door always did
  and keys are honoured (`CheckKeys`, `door.zs:270`); let go short and `Door_Close` drops it
  (`door.zs:321`). Plus a frequency-modulated voice at the door (`door.zs:358-381`) and netevent
  sync so every machine moves the same ceiling (`door.zs:247-259`).

The audit's "the trace is 2D" caveat about `P_UseLines` does not apply: the mod does not use
`P_UseLines`. It runs its own `BlockLinesIterator.CreateFromPos(hand, 8, 14)` (`door.zs:120`) —
`checkh = 8` is a real height window (signature verified,
`wadsrc/static/zscript/doombase.zs:236`), front face only (`door.zs:127`), nearest by true segment
distance (`door.zs:129-134`).

Since the snapshot the grip route also accepts `near.zs`'s raw grip read
(`door.zs:156`), and `near.zs:182` yields the press to the door whenever a hand is on one — so the
two new systems were wired to each other rather than left to collide.

**Label: EXISTS / BUILDABLE-and-built. Cost: none.** The only things left are the audit's own open
question (polyobjects; `src/playsim/po_man.cpp` unread by audit and mod alike) and the mod's stated
v1 scope — manual doors, tag 0, front face, not already moving (`door.zs:24-25`). Swing doors and
lifts are a scope decision for the owner, not an audit finding.

---

## 7. Hitting things with held objects — **the gap, and the fix**

**Today: neither a point sample nor a sweep. There is no hit test.** `hold.zs:179` makes the held
object `bTHRUACTORS`, so it cannot collide with a monster at all, and no code tests a swing (§0).
Walls still refuse it, which is how strain and pinning work — `bTHRUACTORS` filters actors only
(`p_map.cpp:1409`) — so the hold is right about geometry and silent about bodies.

**Why a point test would not do even if one existed.** 01_MELEE §3.5: a fist at 9 m/s moves
~8.7 map units a tic, a swung tip at 18 m/s ~17.5, against an imp's 40-unit box and a lost soul's
32. A point sample clips corners always and tunnels against small targets at speed.
00_CAPABILITIES §1.4 states the rule: don't test the hand's *position*, test the *chord*.

**The design, already written, in 01_MELEE §3.5.** One `LineTrace` along the chord between the last
two samples:

```zscript
src.LineTrace(atan2(ch.Y, ch.X), ch.Length() + fatten, -asin(ch.Z / ch.Length()),
              TRF_ABSPOSITION | TRF_SOLIDACTORS | TRF_BLOCKSELF,
              p0.Z, p0.X, p0.Y, d);
```

`LineTrace` is `actor.zs:805`; `TRF_ABSPOSITION` = 1, `TRF_SOLIDACTORS` = 256,
`TRF_BLOCKSELF` = 1024 (`wadsrc/static/zscript/constants.zs:951`, `:959`, `:961`) — all verified.
For a carried object `p0`/`p1` are the object's own last two positions and
`fatten = RSQDG_Throw.BodyRadius(a)`, which the mod already has (`throw.zs:459-462`), already
capping a pickup's trigger radius at half its height. `SUBS = 3` sub-chords cover the arc's sagitta
(01_MELEE §3.5: a 70 mu blade through 60° in one tic bulges 9.4 mu past its chord).

**Cost, from 01_MELEE §3.4's table.** One `LineTrace` is one `FPathTraverse` — "same order as a
hitscan bullet", with **no** side effects. Two hands × 3 sub-chords × 35 Hz ≈ 210 traces/s, which
that table calls "nothing", and it is bounded by objects held, which is at most two. Resolve with
`DamageMobj` exactly as the thrown path already does (`throw.zs:482`: object as inflictor, player
as source) and kill credit, infighting and kick direction all stay correct.

**What this mod brings that 01_MELEE's prototype does not, so most numbers are already settled:**
kilograms (`mass.zs:62`), the impact-energy scorer with the Doom-gravity term
(`throw.zs:427-450`), the once-per-victim ledger (`throw.zs:397-409`), the event tell
(`throw.zs:485`) and a haptic route (`feel.zs:273-315`; `OnImpact` is currently empty at
`feel.zs:433`). A swing should go through the same `DamageFor`/`Strike` pair, so a thrown barrel
and a swung barrel cannot disagree about what a barrel weighs.

**Three decisions this needs from the owner**, because they are design, not engineering:

1. **`bTHRUACTORS` stays.** It must, or a held barrel shoves monsters around through the spring and
   the strain reads every body as a wall. So the sweep is a *script* hit test over a pass-through
   object — which is itself the reason it must be a sweep and not the engine's collision.
2. **A speed gate** — something like `rsqdg_swing_mps`, 3 m/s to match `rsqdg_throw_min`'s default,
   measured off the **object's** own travel rather than the hand's, so sag and spring lag are
   honoured and carrying a barrel past an imp is not a hit.
3. **Hit-stop and cooldown** — `victim.freezetics = 4` (01_MELEE §3.6), plus one hit per victim per
   `rsqdg_swing_cool` tics reusing the `wHit[]`/`AlreadyHit` shape (`throw.zs:397-409`), so a barrel
   resting against a monster does not tick damage at 35 Hz.

**Label: BUILDABLE. Cost: M** (the sweep is S; the gates, ledger wiring and headset tuning make it
M). It also fixes §4's point-sampled thrown-pickup overlap with the same code.

---

## 8. Haptics — the best-developed part of the mod, missing one route

`feel.zs` is 611 lines and 124 of them are a worked derivation of the engine's haptic paths from
source. Both routes the audit names are present, as a **route switch** rather than a layer, and the
reason is a finding the survey does not have: the Quest runtime refuses a vibrate while that
channel still has time left, and the quake writes **both** channels every frame, so a running
rumble swallows every 150 ms pulse — including other mods' weapon buzzes (`feel.zs:100-114`).

| route | audit says | mod does |
|---|---|---|
| `A_QuakeEx` | shapeable but crude and random; `damrad` **must** be 0 or it thrusts the player (§1.6a) | **the default** (`feel.zs:308-314`): `damrad 0`, `tremrad 64`, `QF_WAVE` with `mulWave 17.5` and duration 2, so the wave's phase is π·(1−ticFrac) and the sample is **never negative** — the fix for the randomness the audit warns about, measured in gdb per **D21**: 330 rumble writes, none negative |
| the engine's own buzz | §1.6: "a mod can provoke one indirectly, which is a HACK" | `SpawnPlayerMissile("RSQDG_Buzz")` (`feel.zs:317-323`), reaching the buzz at `src/playsim/p_mobj.cpp:7545-7557` (verified) — with **D15**'s compat fix: no `+MISSILE`, so Brutal22Q's recoil/flash hooks do not see a shot |
| `LineAttack` + `LAF_NOINTERACT` (§1.6a) | "150 ms, correct handedness, no world effect" | **not used** |

**The mapping is already the right one**, and matches what §1.6a implies: `A_QuakeEx` where an
envelope is wanted (the flight's 0.10→0.40 build scaled by heft, the lock's steady level, the
carry's strain — `feel.zs:446-480`), discrete pulses where a thump is wanted (lock, launch, catch,
release — `feel.zs:328-369`), and silence in the last stretch so the catch lands on a free channel
(`feel.zs:520-532`). A heavy catch is two hits (`feel.zs:405`). **D14** and **D21** carry the
vocabulary. It also handles a trap the audit does not: with `cl_capfps` on, `ticFrac` is pinned at
1.0 and the wave is silent, so it falls back to the pulse route (`feel.zs:261-269`).

**What to take.** Replace `Fire()`'s `SpawnPlayerMissile` with

```zscript
pmo.LineAttack(pmo.angle, 0.1, 0, 1, 'None', "RSQDG_NullPuff",
               LAF_NOINTERACT | ((hand == 1) ? LAF_ISOFFHAND : 0));
```

Verified against source: the buzz at `src/playsim/p_map.cpp:4762-4768` tests the `damage`
**parameter** and nothing else, *before* any tracing, and is **not** gated on `LAF_NOINTERACT`
(read separately at `:4724`); handedness comes from `vr_control_scheme` flipped by `LAF_ISOFFHAND`,
exactly as the missile path does. Gains: no actor spawn, no `P_CheckMissileSpawn`, no
`WorldThingSpawned` for any other mod to misread — **D15's whole compatibility problem disappears**
instead of being worked around by name. One caveat the audit omits: under `LAF_NOINTERACT` the puff
**is** still spawned (`:4930-4933`), so this needs a one-state inert `RSQDG_NullPuff`
(`+NOINTERACTION`, `RenderStyle "None"`) — cheaper than the `RSQDG_Buzz` it replaces. Keep
`RSQDG_Buzz` as a class so old saves load, for the reason `feel.zs:154-156` already keeps
`RSQDG_SoundPoint`.

**Label: BUILDABLE. Cost: S.** It changes nothing about how the mod feels — same 150 ms, same 0.8,
same hand.

**Which route for which, as asked:**

- **A grab landing in the hand** — a discrete thump. `LineAttack`+`LAF_NOINTERACT` on the pulse
  route; `Spike(hand, 1.00, 3)` on the rumble route, doubled for heft ≥ 0.55. Both shipped
  (`feel.zs:344`, `:405`).
- **A heavy object's weight** — needs an envelope, so `A_QuakeEx` only; the pulse route cannot
  express it. Shipped as strain × 0.6 × log-bulk (`feel.zs:465-477`), and correctly **silent** when
  the carry follows freely.
- **An impact** — `OnImpact` is empty by the decision at `feel.zs:431`: a thrown thing has left the
  hand, so it is heard and not felt. Right for a *throw*. A **swung** object has not left the hand,
  so §7 should buzz it: `LineAttack`+`LAF_NOINTERACT` on the swinging hand, scaled on the rumble
  route by `Heft` so a swung barrel out-thumps a swung clip.
- **The Headlock** currently borrows the catch tick (`near.zs:329`, `feel.zs:405`). A 3-second hold
  on a struggling monster is the clearest case in the whole package for a continuous level — add it
  to `Level0` (`feel.zs:446`) beside the carry's strain, scaled by `Heft`, and it costs nothing new.

---

## 9. Human shields — **they now EXIST in the mod, and both leaks are open**

06_GRAB §4.5 is right that the blocking is free. The citations verify in this tree:

- **Hitscan:** the trace mask is `MF_SHOOTABLE` (`p_map.cpp:4919`), the trace's actor test is
  `thing->flags & ActorMask` and nothing else (`p_trace.cpp:866`), and the skip filter consults the
  **puff's** flags, not the victim's (`p_map.cpp:4829` → `:4690`). There is no intended-target
  test anywhere. **A held monster eats the pellets today.**
- **Projectile:** `PIT_CheckThing` (`p_map.cpp:1589-1717`), height gate at `:1635-1641`, infight
  gate `CanAttackHurt` at `:1672` (impl `:1275-1300`), which exempts players entirely and is
  overridden per-actor by `+FORCEINFIGHTING` at `:1282` (verified verbatim).

**And the mod now does it.** `near.zs:316-371`: the off hand's grip holds a living monster — not a
corpse — frozen in front of the palm for 3 seconds, default **on** (`CVARINFO.txt:47`). It keeps
`bSHOOTABLE` (never touched), which is the whole shield property. **D6/D7** still hold for the
*distance* grab — `policy.zs:360-362` refuses anything living at range, and that is the owner's
decision with his reason on record — but the near grip is a different act, added 10-29, and D6 does
not cover it.

**Two leaks, both open, and the order matters:**

1. **`bTHRUACTORS` defeats the projectile shield outright.** `near.zs:324` sets it;
   `p_map.cpp:1409-1410` returns "no collision" on the **OR of both actors'** flag, before the
   height gate, the infight gate and `P_DoMissileDamage`. Every fireball passes through, on every
   map. **This is the biggest silent bug in the mod**, and it is silent because the hitscan half
   works perfectly (above), so the shield reads as working until the first imp.
   **The fix is to stop setting it.** It is there to keep the monster from fighting the player's own
   collision cylinder, but `near.zs:358` already uses a collision-tested `TryMove` whose failure is
   *ignored* — so a blocked move simply leaves the monster where it is, which is correct, and
   `near.zs:357` breaks the hold once it is torn 1.5 m away. Drop the flag, keep the restore
   (`near.zs:322`, `:338`) for compatibility with anything else that set it, and test in the
   headset for the one case it was guarding: a monster held at `Radius + 6` in front of the palm
   overlapping the player at point-blank. **BUILDABLE / S.**
2. **`+FORCEINFIGHTING` is not set**, so on a `NoInfighting` map or skill a monster's missile is
   still waved through by `CanAttackHurt` even after (1) is fixed. Set it while held, saved and
   restored through the same borrowed-state pattern `near.zs:321-322` already uses for
   `bNOGRAVITY`/`bTHRUACTORS`. **~4 lines, S.** Do it in the same change as (1), or (1) will look
   like it did not work on the maps that set the flag.

**Two smaller notes on the same code:**

- **The mass gate reads `Actor.Mass`, not the package's kilograms.** `near.zs:148`:
  `a.Mass <= maxMass`, default 400. For monsters `Actor.Mass` is meaningful (imp 100, demon 400,
  baron 1000) so this works, and `CVARINFO.txt:48` documents those numbers — but `mass.zs:177`
  warns that `Actor.Mass` is 100 on *every* non-monster, and the rest of the package decides weight
  through `RSQDG_Mass.Kg`. A mod that re-scales monster `Mass` (Champions gives a Bronze champion
  `0x7FFFFFFF` — the overflow the throw path already had to bound at `throw.zs:446-449`) will make
  this gate behave arbitrarily. One line to route it through `RSQDG_Mass.Kg` with a kilogram
  default, or one comment saying why Doom mass is deliberate here. **S.**
- **`FindNear` sweeps a 128-unit box every tic, both hands, to find things within 10 units.**
  `near.zs:132`: `BlockThingsIterator.CreateFromPos(hp.x, hp.y, hp.z, 128, 128, false)`, then every
  candidate is rejected by distance against `nearU`, which at the default `rsqdg_near_m` 0.30 m is
  **10.2 map units** (`near.zs:124`). Blockmap cells are 128 mu (01_MELEE §3.4), so that is a 2×2
  or 3×3 cell walk per hand per tic — and it runs unconditionally from `WorldTick` to compute
  `ready[]` (`near.zs:183`), then again inside `NetworkProcess` on the press (`:214`). 06_GRAB §5
  names exactly this: *"The real cost is a per-tic `BlockThingsIterator` if you scan for grab
  candidates continuously — scan only while the grip button is down, or every 3rd tic."* Pass
  `nearU + 64` (enough for the biggest monster's radius) instead of 128, cache `ready[]` across
  3 tics, and skip the scan entirely when the hand is full. **BUILDABLE / S**, and it is the only
  per-tic cost this mod adds that the audit flags by name.

**Label: the blocking is EXISTS; the Headlock is BUILDABLE-and-built with two open leaks, both
S.** D6/D7 are respected: nothing here proposes a living monster at *range*.

---

## 10. The piece table

| # | Piece | Label | Cost | Note |
|---|---|---|---|---|
| 1 | Flick detection (wrist pitch rate) | EXISTS | — | D3; turn-immune at source; the frame inference is the package's best work |
| 1a | 2-tic deltas for throw velocity | BUILDABLE | S | feel A/B; names a real estimator bias |
| 2 | Interpolation / the carry | EXISTS | — | velocity, not `SetOrigin`; better than the audit's advice |
| 2a | D20's lead applied to the spring target | BUILDABLE | S | A/B only |
| 3 | Holding still — the carry | EXISTS | — | neither `freezetics` nor `+NOINTERACTION` needed; both traps avoided |
| 3a | **Holding still — the Headlock** | **BUILDABLE** | **S** | `near.zs:361` uses `tics = -1`; `freezetics` is stronger and self-healing |
| 4 | Throw tracker; `+SKULLFLY` declined | EXISTS | — | four reasons the audit does not weigh; D23 |
| 4a | Unit trap (`pixelstretch` on Z) | EXISTS | — | `reach.zs:262-291` |
| 5 | Pickup via `CallTryPickup` | EXISTS | — | plus `bSPECIAL` suppression both ways |
| 6 | Door lift | EXISTS | — | continuous `MoveCeiling` + `TakeOver` + commit; past the audit |
| 7 | **Swung-object hit test** | **BUILDABLE** | **M** | **absent; the advertised feature** |
| 8 | Haptics vocabulary, both routes | EXISTS | — | best-developed part of the mod |
| 8a | `LineAttack`+`LAF_NOINTERACT` for the pulse | BUILDABLE | S | removes an actor spawn and D15's whole hazard |
| 9 | Human shield (Headlock) | BUILDABLE-and-built | — | the blocking is EXISTS |
| 9a | **`bTHRUACTORS` defeats the projectile shield** | **BUILDABLE** | **S** | `near.zs:324` vs `p_map.cpp:1409`; the biggest silent bug |
| 9b | `+FORCEINFIGHTING` on a held monster | BUILDABLE | S | the audit's named leak; invisible until 9a is fixed |
| 9c | `FindNear`'s 128-unit per-tic sweep | BUILDABLE | S | 06_GRAB §5's named cost, by name |
| 9d | Mass gate reads `Actor.Mass` | BUILDABLE | S | or one comment saying why |
| 10 | Polyobject doors | unknown | — | `po_man.cpp` unread by audit and mod alike |
| 11 | `near.zs` has no decision record | — | S | D31 for the walk-up grip, the swap and the Headlock |

---

## 11. Corrections this mod makes to the survey

Each was measured rather than reasoned, so each is worth folding back:

1. **The "~1 tic behind" floor is beatable** by leading the placement, measured in
   `tools/rigtest/leadsim.py` (D20). 06_GRAB §5 states it as a flat wall.
2. **The rumble's randomness is fixable.** 00_CAPABILITIES §1.6a says `A_QuakeEx` "is noise: fine
   for a thud, useless for a crisp click." With `QF_WAVE`, `mulWave 17.5` and duration 2 it is not
   — the sample is never negative, verified in gdb (D21, `feel.zs:76-87`).
3. **The two haptic routes cannot coexist.** The quake writes both channels every frame and the
   runtime drops a request to a busy channel (`feel.zs:100-114`), so a mod running a rumble mutes
   every other mod's buzz. §1.6a presents them as complementary; they are exclusive.
4. **`cl_capfps` silences the quake route entirely** — `ticFrac` is pinned at 1.0.
5. **A pickup's `radius` is a trigger, not a hull** — a clip is radius 20, the same as a shotgun
   guy. Both the flight and the carry must cap it, and every mechanic in the survey that sizes a
   grab sphere or a swept `fatten` off `radius` inherits the error (`flight.zs:228-238`,
   `hold.zs:183-193`, `throw.zs:459-462`).
6. **`Actor.AttackDir()` returns angles from ZScript, not a vector** — "a silent,
   plausible-looking bug" (`reach.zs:196-201`, citing `p_actionfunctions.cpp:5694-5712`).
   00_CAPABILITIES §1.2 lists `AttackDir` as "a basis helper" with no such caveat.
7. **Doom gravity is ~4.4 g** in impact terms, so a raw landing speed over-reads badly; the
   vertical term needs `sqrt(9.81/g_doom)` (D23, `throw.zs:411-438`). Nothing in the survey says
   so, and it blew a barrel up on a gentle set-down the first time.
8. **A held actor's own `+THRUACTORS` is asymmetric**, and 06_GRAB §4.5 should say so: it does
   **not** stop a hitscan (the trace reads only `ActorMask` at `p_trace.cpp:866`, and the skip
   filter reads the *puff's* flag at `p_map.cpp:4829`) but it **does** stop a projectile dead at
   `p_map.cpp:1409-1410`, before every other gate. Any shield design must name the flag.
