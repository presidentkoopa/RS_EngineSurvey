# 10 — HAND OWNERSHIP, and the end of the DistanceGrab pass

**Part One** resolves the collision `07_WHEEL.md` §9.4 flagged: three mods answer "is this hand
claimed?" three private ways. **Part Two** takes `04_DISTANCEGRAB.md`'s findings to actionable
specifics.

**Sources, exactly.** Mod paths are inside the fresh clone of `github.com/presidentkoopa/RSQ_UBQSeries`,
HEAD `6a5c98c` (2026-10-30), at `.../scratchpad/ubqs_gh/<Mod>/`. Engine paths are relative to
`.../scratchpad/qzd173`, tag `qzd17.3`. Every citation below was read in those trees.

**Read first and not re-derived here:** `surveys/00_CAPABILITIES.md`, `apply/04_DISTANCEGRAB.md`,
`apply/07_WHEEL.md`, `apply/WHIP_HANDOFF.md`.

**Scope.** Recommendations only. Nothing was written to any mod's tree. No engine change is
proposed anywhere in this file. **No edit is proposed to any Grapple file** — the Grapple is being
rewritten as a tether/whip right now, so it is read here only to understand how it claims a hand,
and what it owes the protocol is stated as a requirement in §0, not as a change to make.

---

## 0. THE ANSWER, and the tether's requirement

### 0.1 The answer in one sentence

**One new service, `RSQ_HandsService`, published independently by every mod that touches a hand,
answering a per-hand claim with an owner and a priority; a mod consults the *other* publishers and
stands down against the highest priority it finds, so arbitration is distributed and no mod is
anybody's dependency.**

Not a shared library (the series rule forbids one), not a hub in one mod (today's `rig.busy` is
that, and §2.5 shows why it fails), not N² pairwise lookups (today's shape, and §2 is the bill).

### 0.2 THE TETHER / WHIP REQUIREMENT — one section, actionable, no other reading needed

The tether is being written now, so this is the time-sensitive part. **Four requirements, in order
of how much breakage they prevent.** Nothing here asks the tether to depend on another mod: every
one degrades to today's behaviour when nothing else is loaded.

**R1 — Do not reuse `rsqdg_press` / `rsqdg_release` for the whip's Lock mode, and consider
defaulting `rsqgr_grab_dg` to 0.** `WHIP_HANDOFF.md` §4.5 already says Lock "needs a **fifth
bind**", and it is right. The reason, with the citation: `Grapple/zscript/rsqgr/input.zs:147-149`
listens for `rsqdg_press` / `rsqdg_release` and calls `core.Grab(...)`, gated on `rsqgr_grab_dg`,
which **defaults to 1** (`Grapple/KEYCONF.txt`, the `rsqgr_reset` alias). Those are DistanceGrab's
own KEYCONF aliases (`DistanceGrab/KEYCONF.txt`). This works today only by luck:
`DistanceGrab/zscript/rsqdg/grab.zs:127-130` **discards** its own press unless `rsqdg_button` is
set, and `rsqdg_button` defaults false. **Turn `rsqdg_button 1` for one desktop test and a single
key press locks a grab target and grabs a rope at once** — and with Lock bound there too it would
also lock a whip target. The borrow is a hard-coded string naming another mod's input channel,
which is the one thing the series rule exists to prevent.

**R2 — Publish `RSQ_HandsService` from the tether, per hand, with an owner and a priority.** The
request set is in §3.2. The tether's claims and the priority each should carry:

| Tether state | Source today | Claim |
|---|---|---|
| a hook in flight / latched / biting on hand *h* | `hook.phase` per hand, `service.zs:401-404` | `"grapple.hook"`, priority **70** |
| a hand holding a latched rope (hand over hand) | `h.gripHand`, `core.zs:567`, `:572-576` | `"grapple.climb"`, priority **80** |
| the whip's four-phase wind-up on hand *h* | `WHIP_HANDOFF.md` §4.1 | `"grapple.whip"`, priority **75** |
| LAID / at rest | — | **no claim** |

Priority 80 for the climb because letting go of a rope hand-over-hand at 40 m is not something
another mod's gesture may cause. The whip's wind-up above the hook because the wind-up is a long
motion another mod will otherwise read as its own gesture.

**R3 — Consult `RSQ_HandsService` before the wind-up arms, and make `HandFree` ask it.**
`Grapple/zscript/rsqgr/climb.zs:29-45` (`RSQGR_Climb.HandFree`) is the tether's existing
"is this hand free" predicate and it answers from the hand's **weapon class name** alone. It does
not know about the Wheel's open ring, DistanceGrab's carry, the TacticalRig's holsters, the
cigarette or the Zippo. The wind-up of §4.1 is a large, slow arm motion — exactly the shape of the
Wheel's ring-turning roll and DistanceGrab's palm-out reach — so it is the single most collision-
prone gesture the series will contain. **Ask before you arm, not after you fire.**

**R4 — Answer `"climbing"` per hand, or rename it.** `service.zs:406-415` documents `"climbing"`
as "1 **a hand** holds a latched rope" but the implementation loops both hooks and returns 1 if
**either** has `gripHand >= 0` (`:414`) — the `hand` intArg is never read. A consumer passing a
hand gets an answer about the player. Fix it in the rework, or rename it `"climbing.any"`; either
is fine, but a per-hand request that silently ignores its hand argument will be miscited by
everyone who reads the comment.

**What the tether does NOT owe.** Nothing about aim, nothing about `keepmomentum`, nothing about
the crack timing. R1–R4 are the whole bill, and R1 is the only one that is urgent.

---

## 1. HOW EACH MOD DECIDES A HAND IS BUSY TODAY

Three mods, three predicates, written independently, no two of which can agree. Quoted, not
paraphrased.

### 1.1 DistanceGrab — `HandBusy`, and a second one that is not it

**`RSQDG_Reach.HandBusy(p, hand)`** — `DistanceGrab/zscript/rsqdg/reach.zs:141-191`. Consumed by
`PalmOk` (`reach.zs:108`, the first line of the gesture test) and by the accessibility Reach
modifier (`grab.zs:356`). Five clauses, in this order:

```zscript
reach.zs:145   if (p && p.mo && RSQDG_A11y.Swapped(p.mo)) return hand == 0;
reach.zs:146   if (!p || !Flag("rsqdg_yield", p, true)) return false;
reach.zs:157-162   // TWO-HANDING
               if (p.mo && p.ReadyWeapon && !p.ReadyWeapon.bMELEEWEAPON)
               {
                   double steady = Num("rsqdg_steady_m", p, 0.45);
                   if (steady > 0 && Metres(p.mo.AttackPos - p.mo.OffhandPos) < steady) return true;
               }
reach.zs:163-166   let rit = ServiceIterator.Find("RSQTR_RigService");
               while (rs = rit.Next())
                   if (rs.GetInt("rig.busy", "", hand, 0, p.mo) == 1) return true;
reach.zs:167   Weapon w = (hand == 0) ? p.ReadyWeapon : p.OffhandWeapon;
reach.zs:168   if (!w) return false;
reach.zs:176-184   // ubqs_saber: ask UBQS_SaberService "saber.lit"; 0 -> free, no answer -> busy
reach.zs:188   if (n == "fist" || (w.bMELEEWEAPON && w.bCHEATNOTWEAPON)) return false;
reach.zs:189   if (w.bMELEEWEAPON) return true;
reach.zs:190   return n.IndexOf("ssq_") == 0 || n.IndexOf("ubqs_") == 0;
```

So: a hand is busy if a non-melee gun is held and the hands are within 0.45 m, or the TacticalRig
says so, or the hand holds a melee weapon that is not a bare fist, or the class name begins `ssq_`
/ `ubqs_`. **It asks exactly two services, both by a name specific to one mod**
(`RSQTR_RigService`, `UBQS_SaberService`) and otherwise decides from the weapon class name. This
is the best of the three — D26 (`docs/DECISIONS.md:426-432`) is the right instinct, stated
outright: *"The evidence that a saber is not being swung is its blade being off... the grab asks by
name; no answer means busy, the old behaviour."* The protocol in §3 is D26 generalised.

**And there is a second, different predicate in the same mod.** `near.zs` — the grip-up-close
system added 10-29 — does **not** call `HandBusy`. It has its own:

```zscript
near.zs:101-108   static bool RigBusy(PlayerPawn pmo, int h)
                  {
                      let rit = ServiceIterator.Find("RSQTR_RigService");
                      while (rs = rit.Next())
                          if (rs.GetInt("rig.region", "", h, 0, pmo) >= 0) return true;
                      return false;
                  }
```

consumed at `near.zs:179` (`if (RigBusy(pmo, h)) continue;   // holsters first`). It asks the same
service a **different request**, and that request is a strict subset:
`TacticalRig/zscript/rsqtr/hooks.zs:62` returns `rig.HoverRegion(h)`, while `rig.busy` at `:68-79`
returns `rig.HandBusy(h)` **OR** the cigarette **OR** the Zippo **OR** a DistanceGrab door. Since
`rig.HandBusy(h)` is `hoverR[h] >= 0` (`TacticalRig/zscript/rsqtr/rig.zs:76`, `:79`), the two
differ by exactly the cigarette, the Zippo and the door. So the walk-up grab, the hand swap and the
Headlock **fire while the hand holds a lit cigarette or the Zippo**. (The door is handled
separately at `near.zs:182` and `:212`, so only the smokes leak.)

### 1.2 The Wheel — `IsOpen` / `OpenHand` / `HasAnchor`, and the open/anchored split

**`wr_RigService`** — `Wheel/zscript/wheel/zscript.zs:158-222`:

```zscript
zscript.zs:173   if (request == "IsOpen")   return rig.IsOpen() ? 1 : 0;
zscript.zs:174   if (request == "OpenHand") return rig.OpenHand();
zscript.zs:185   if (request == "HasAnchor") return rig.HasAnchor() ? 1 : 0;
```
backed by `zscript.zs:585-598`:
```zscript
bool IsOpen() const   { return mOpen; }
int  OpenHand() const { return mRigHand; }   // 0 = main, 1 = off
bool HasAnchor() const { return mHaveAnchor; }
```

The `HasAnchor` / `IsOpen` split is **good service design and should be kept** — the comment at
`:175-184` gives the reason (the ring can report open one tic before `mAnchor` resolves, and a
caller that builds on the placeholder bakes itself at `(0,0,0)` forever, *"confirmed as the actual
cause of 'the card never shows up at all'"*). That is a *staleness* distinction, and §3.4 borrows
it.

**But as a hand claim it is the weakest of the three, in two specific ways.**

1. **`OpenHand` names one hand and the Wheel uses two.** `mRigHand` is the hand the ring is
   anchored to. The *poke* hand is `mPokeHand`, it moves across whenever the other hand is on the
   ring (`zscript.zs:7758-7766`), and the **two-hand compare** mode reads the other hand's own
   `AimBillboard` every tic (`zscript.zs:7979-7987`) and is **on by default**
   (`cvBool("wr_twohand_compare", true)`, `:7980`). A consumer standing down one hand on
   `OpenHand` stands down the wrong one half the time and never stands down both.
2. **There is no claim at all — it is a state report.** Nothing says "I am using this hand, do not
   use it". `IsOpen` is a fact about the Wheel; a consumer has to *know* that an open ring means
   both hands are in use. **And nothing anywhere in the series consumes it**: `grep -rn
   wr_RigService` over all 15 mods returns only the Wheel's own files and its own test probe
   (`Wheel/tools/rig/probe/zscript.txt:11`). It is a door nobody has walked through in either
   direction.

### 1.3 The Grapple — `HandFree`, the inverse of `HandBusy`, written from scratch

Read-only, for understanding. **`RSQGR_Climb.HandFree(pmo, hand)`** —
`Grapple/zscript/rsqgr/climb.zs:29-45`:

```zscript
static play bool HandFree(PlayerPawn pmo, int hand)
{
    if (!pmo || !pmo.player) return false;
    if (!RSQGR_Util.Flag("rsqgr_climb_free_only", true)) return true;
    let p = pmo.player;
    Weapon w = hand == 0 ? p.ReadyWeapon : p.OffhandWeapon;
    if (!w) return true;
    String n = w.GetClassName();
    n = n.MakeLower();
    if (n == "rsq_hands")
    {
        let psp = p.FindPSprite(FREEHAND_LAYER);
        return psp && psp.Caller == w && psp.CurState && psp.CurState == w.FindStateByString("AmmoH");
    }
    if (n == "fist" || n == "melee_attacks") return true;
    return w.bMELEEWEAPON && w.bCHEATNOTWEAPON;
}
```

Consumed at `core.zs:573` to drop a climbing grip. **This is the same algorithm as DistanceGrab's
`HandBusy`, independently written, reaching a different answer.** The shared shape: take the hand's
weapon, lower-case the class name, match a hardcoded list, fall back to `bMELEEWEAPON &&
bCHEATNOTWEAPON`. The differences:

| Hand holds | DG `HandBusy` says | GR `HandFree` says |
|---|---|---|
| nothing | **free** (`reach.zs:168`) | **free** (`climb.zs:35`) |
| `Fist` | free (`:188`) | free (`:43`) |
| `Melee_Attacks` (BD fists) | free *via* `bMELEEWEAPON && bCHEATNOTWEAPON` (`:188`) | free by name (`:43`) |
| `RSQ_Hands` (BD VR empty hand) | **free** — no clause matches, `:190` returns false | **free only while its psprite is in state `AmmoH`** (`:40-41`) |
| an `ssq_` / `ubqs_` class | **busy** (`:190`) | not free (falls to `:44`) |
| a lit saber vs a dark hilt | **asks `UBQS_SaberService`** (`:176-184`) | no distinction — not free either way |
| an ordinary gun | **free unless the hands are within 0.45 m** (`:157-162`) | **not free** (`:44`) |

Two rows matter. **`RSQ_Hands`:** DistanceGrab calls that hand unconditionally free; the Grapple
calls it free in one psprite state only. So on the same tic, with Brutal Doom VR's empty off hand
up, one mod will arm a gesture on a hand the other has already decided is occupied.
**An ordinary gun:** the Grapple will not climb with a hand holding a gun at all; DistanceGrab will
reach with it as long as the other hand is 45 cm away. Neither is wrong for its own mechanic —
which is precisely the point: *"busy" is not one question, and a boolean cannot carry two answers.*

Note also the `RSQ_Hands` psprite read at `climb.zs:40-41`: DistanceGrab tried the psprite-frame
route and recorded it as unworkable — `docs/DECISIONS.md:428-430`, *"The psprite frame does not
carry it (world saber on = frame 18 always, rig)."* Different layer, different weapon, so it is not
the same finding; but it is the same technique and the two mods disagree about whether it works.

### 1.4 What else is already answering a per-hand question — the precedent to copy

`RSQLS_SightService` (`LaserSight/zscript/rsqls/sight.zs:788-798`) is the shape the protocol
should take, and it already works:

```zscript
if (request == "laser.active") return (RSQLS_Sight.LiveHands() >> (intArg & 1)) & 1;
if (request == "laser.beam")   return (RSQLS_Sight.BeamHands() >> (intArg & 1)) & 1;
return -1;
```

Namespaced request, **per hand**, `-1` for "I do not answer that". DistanceGrab already consumes
it — `DistanceGrab/zscript/rsqdg/laser.zs:327-334`, for **drawing** only (D28,
`docs/DECISIONS.md:438-441`: one beam per hand). That is the precedent in both directions: the
publisher shape and the consumer idiom both exist and ship.

**Where each mod is already right, and should be left alone:**
- **DistanceGrab** asks by service name with the result checked, never `Service.Find` with a class
  literal — so an absent mod is a string that never matches, and the package builds alone (D26).
  Its `rsqdg_yield` master switch (`reach.zs:146`) means the whole yielding layer is one cvar.
- **The Wheel** publishes `HasAnchor` separately from `IsOpen` because open and placed are not the
  same tic. That distinction is rarer than it should be and §3.4 generalises it.
- **The Grapple** takes its input from its own KEYCONF aliases and has no `Weapon` subclass, so it
  cannot cause the engine's two-hand latch (`WHIP_HANDOFF.md` §1.5, verified: zero `Weapon`
  subclasses in `Grapple/zscript/`).
- **All three** use `ServiceIterator.Find(String)`, which is the correct idiom. The Wheel's comment
  at `zscript.zs:4838-4842` gives the reason: `Service.Find` takes a `class<Service>` resolved at
  compile time and would fail to build without the other mod loaded.

---

## 2. WHERE THEY COLLIDE, CONCRETELY

A collision counts here only if the player action can be described. Six are described. The
mechanism behind the first four is one engine fact, so it is stated once.

### 2.0 Why the Wheel's freeze does not stop DistanceGrab — the mechanism

The Wheel claims the player with `wr_VR.SuppressVRInput(true)` →
`p.cheats |= CF_TOTALLYFROZEN` (`Wheel/zscript/wheel/wr_q_shim.zs:727-731`). Engine side,
`PlayerPawn.CheckFrozen` (`wadsrc/static/zscript/actors/player/player.zs:1103-1134`) touches
**`cmd` and nothing else**: `cmd.buttons &= BT_USE` (`:1119`), `forwardmove`/`sidemove`/`upmove`
zeroed (`:1123-1125`), `turnticks = 0` (`:1126`). It does not touch `AttackPos`, `OffhandPos`,
`AttackRoll`, `OffhandRoll` or `AttackPitch`; it does not reach `InputProcess`; it does not reach
netevents.

**DistanceGrab has three input routes and the freeze reaches none of them:**

1. **The palm pose** — `PalmOk` reads `Roll()` and `Pitch()` off the hand transform
   (`reach.zs:112`, `:121`), consumed from `grab.zs:352` every tic. Not a button.
2. **The raw grip key scan** — `near.zs:61-86`, an `InputProcess` override. That is a `ui`-scope
   key hook that runs before any usercmd exists.
3. **The KEYCONF netevents** — `rsqdg_press` arriving at `grab.zs:118`. A netevent reaches
   `NetworkProcess` at the top of the tic, before `P_PlayerThink`.

So the Wheel's freeze stops locomotion, snap turn and weapon fire, and leaves every DistanceGrab
gesture live. This is not a bug in the freeze — `CF_TOTALLYFROZEN` is the right tool for what the
Wheel wants (`07_WHEEL.md` §5.1) — it is the gap a protocol has to fill.

### 2.1 **Turning the ring arms a distance grab, and dropping your hand fires it.** PROVEN.

The Wheel makes you roll your wrist on purpose: `handRollOf` (`zscript.zs:7574-7581`) reads
`AttackRoll` / `OffhandRoll` and feeds `wr_roll` (`:5951`) to turn the ring. DistanceGrab's trigger
is `|roll − rsqdg_palm_roll| <= rsqdg_palm_tol` — 90° ± 30° (`reach.zs:112-117`) — plus a pitch
floor of −30° (`reach.zs:119-122`), which a hand pointed at a ring in front of you clears easily.

Does `HandBusy` save it? No. The only clause that could is the two-handing gate
(`reach.zs:157-162`), which needs the hands within 0.45 m. One hand pokes the ring and the other
hangs at your side: well over 0.45 m. `rig.busy` answers 0 (no holster). The weapon is an ordinary
gun, so `:189-190` return false. **`HandBusy` returns false and the reach arms.**

**The player action.** Tap LThumb. The ring opens. Roll your wrist to spin it to the gun you want,
point, release the button. The ring folds. Your hand drops to your side — which is the **falling
edge of DistanceGrab's pose**, read at `grab.zs:382-392`, and a wrist tipping up as a hand drops is
the launch flick (`aim.zs:264-284`, fired at `grab.zs:269-287`). **A barrel across the room flies at
your face as the ring closes.** Nothing in either mod logged anything unusual.

Three further consequences of the same arming:
- `TryLock` ran (`grab.zs:415`), so the amber "could" / green "have" glow (D25) has been sitting on
  a monster behind the ring for the whole interaction.
- `laser.zs` drew a beam and cone from that hand. **The double-beam problem `07_WHEEL.md` §9.2
  names for RSQ_LaserSight applies to DistanceGrab too** — and DistanceGrab's own defence
  (`rsqdg_beam_yield`, `laser.zs:320`, `:327-334`) asks `RSQLS_SightService` only. It does not know
  the Wheel exists. Two beams, two dots, one controller, and the Wheel's is the one that means
  something at that moment.
- The Wheel's four commit confirmations all fire normally, so there is no cue that anything else
  happened.

### 2.2 **The grip press the ring needed is eaten by a Headlock.** PROVEN.

`near.zs:61-86` reads the grip as raw key scans and **consumes the press** when one of its three
acts takes it (`InputProcess` returns `eaten[h]`, `:78`). `ready[h]` is computed in `WorldTick`
(`:177-184`) and is true whenever `FindNear` finds anything in reach. The only stand-down is
`RigBusy` (`:179`) — the TacticalRig's hover region, nothing else.

**The player action.** The ring is open, the world slowed not stopped (`07_WHEEL.md` §5.2: *"a
wheel you can be shot at through is the point"*). An imp closes to arm's length. You squeeze the
off grip — for whatever you have bound there. `near.zs` sees a living monster within
`rsqdg_near_m` 0.30 m, `Mass` 100 ≤ `rsqdg_collar_mass` 400, starts a Headlock, and returns true
from `InputProcess`. **The press never reaches whatever you bound it to**, and a monster is now
frozen in front of your off hand for three seconds while you are trying to pick a gun. No mod was
asked, and the Wheel's ring is still up.

The same swallow applies to anything else the player bound to the grip: the ShieldSaw's draw, the
tether's Lock mode once R1 gives it a bind, a KEYCONF alias of their own.

### 2.3 **A braced rifle hands DistanceGrab a hijacked pitch.** PROVEN, and it is the engine's claim.

This is gotcha 10a interacting with `HandBusy`, and the arithmetic is exact.

`gl_openvr.cpp:1716-1728` latches `weaponStabilised` on the off-hand grip's **rising edge** with
the controllers inside 0.50 m. While latched, `:1742-1753` **overwrites `weaponangles[PITCH]` and
`[YAW]`** with the hand-to-hand line and **passes `weaponangles[ROLL]` through verbatim** (`:1752`,
the third argument to `VectorSet`). Release clears it with **no distance test** (`:1726-1727`).

DistanceGrab's `PalmOk` reads **roll** (unaffected) **and pitch** (`reach.zs:121-122`,
`Pitch(pmo, hand) < minPitch`). So while braced, `rsqdg_palm_pitch` is being tested against a
number that describes the line between the two controllers, not the hand's own pitch.

`HandBusy`'s two-handing gate uses **0.45 m** (`rsqdg_steady_m`, `reach.zs:159`); the engine latches
at **0.50 m** and then holds at any distance. **So:**

> Brace a rifle (grip pressed inside 0.50 m — the latch takes). Now spread your hands to 0.6 m
> while still holding the grip. The latch **stays** (`:1726-1727` only clears on release).
> `rsqdg_steady_m` 0.45 m is now satisfied, so `HandBusy` returns false and the reach **arms** —
> on a pitch value that is the hand-to-hand line. The gesture's one no-overlap discriminator, the
> pitch floor that `reach.zs:100-105` is explicit about (*"PITCH is what has no overlap"*), is
> reading the wrong quantity.

Raising `rsqdg_steady_m` to 0.50 closes the window partly and is a one-cvar mitigation worth doing
regardless, but it cannot close it: the latch survives the hands separating and a distance test
cannot see a latch. **Reading `player.WeaponState & WF_TWOHANDSTABILIZED` is the only correct
test** (`player.zs:393`, `constants.zs:1242` — both script-visible, confirmed by `07_WHEEL.md`
§4.4). `WHIP_HANDOFF.md` §1.5 reached the same conclusion from the other side and derives the whip
from **roll** for exactly this reason: *"the wrist is the one clean input channel during a
two-handed grip."* That finding is general and belongs in `00_CAPABILITIES` gotcha 10a.

### 2.4 **One key press locks a grab and grabs a rope.** LATENT, one cvar away. See §0.2 R1.

`Grapple/zscript/rsqgr/input.zs:147-149` acts on `rsqdg_press`; `rsqgr_grab_dg` defaults 1;
`grab.zs:127-130` discards the same event because `rsqdg_button` defaults false. Flip
`rsqdg_button` for a desktop test — which `grab.zs:124-126` says is exactly what it is for — and
the two fire together. This is the collision that is cheapest to prevent and the only one whose
prevention is urgent, because the tether is adding binds this week.

### 2.5 **The ring opens on a climbing hand.** PROVEN, no cue either way.

The Grapple claims a hand hand-over-hand: `core.zs:567` sets `h.gripHand = g`, re-validated at
`:572-576` through `HandFree`, and the whole climb is `climb.zs` + `core.zs:551-605`
(`WHIP_HANDOFF.md` §5: *"entirely built"*).

`openRig` (`zscript.zs:1048-1130`) asks nothing. It sets `mRigHand`, suppresses VR input, spawns
the ring and turns on the laser. The poke hand then moves to whichever hand is on the ring
(`:7758-7766`).

**The player action.** You are hand-over-hand up a rope, 20 m off the floor. You tap LThumb for a
weapon. `CF_TOTALLYFROZEN` zeroes your movement input but the rope's own velocity reconstruction
(`core.zs:423-431`) is not driven by `cmd`, so you keep moving. The ring anchors to the climbing
hand, the Wheel starts reading that hand's `AimBillboard` as a card selection, and
`RSQGR_Climb.HandFree` — which asks only what weapon the hand holds — still says the hand is free
to climb. Both mods are driving one hand, neither has a cue, and the failure mode is a fall.

### 2.6 **The walk-up grab fires on a hand holding a lit cigarette.** PROVEN, cosmetic.

§1.1: `near.zs:106` asks `rig.region`, not `rig.busy`, so it misses the cigarette and the Zippo
that `TacticalRig/zscript/rsqtr/hooks.zs:71-73` folds into `rig.busy`. Smoke in hand, lean toward a
clip on the floor, squeeze the grip: the clip is taken into the hand that is holding the cigarette.
**Smallest of the six, and the cheapest: one request string.** It is also the clearest evidence for
§3 — two predicates in *one mod* already disagree, so this is not an inter-mod problem that
discipline will solve.

### 2.7 The bill

| # | Collision | Status | Caused by |
|---|---|---|---|
| 2.1 | Ring-turn arms a reach; hand-drop launches an object | **proven** | Wheel's claim is not a claim; DG does not consult it |
| 2.2 | Headlock eats the grip the player bound elsewhere | **proven** | `near.zs` claims unilaterally and consumes the input |
| 2.3 | Braced rifle hands DG a hijacked pitch | **proven** | the engine's latch is an unarbitrated claim (§5) |
| 2.4 | One press = grab lock + rope grab | **latent**, `rsqdg_button 1` | a hardcoded string naming another mod's channel |
| 2.5 | Ring opens on a climbing hand | **proven** | neither mod asks |
| 2.6 | Walk-up grab on a cigarette hand | **proven** | two predicates in one mod, `rig.region` vs `rig.busy` |

**One more, not proven and labelled so.** DistanceGrab's **hand swap** needs both controllers
within `rsqdg_swap_m` 0.25 m with both wrists rolled ≥ 55° (`near.zs:110-116`), and the Wheel's
two-hand compare invites exactly that pose. Whether 0.25 m is actually reached depends on the live
ring geometry: `wr_radius` defaults **5.0** map units (`RingClearance`, `zscript.zs:656-662` — note
`RING_RADIUS 26` at `:227` is a dead constant nothing reads), so a ring is roughly 0.3 m across but
sits `wr_forward` 34 units ≈ 1 m in front of the hand, and the fingertip is *projected* to
`org + dir * mTouchReach` rather than placed there (`zscript.zs:7796-7800`, `:5878`). The hands
themselves may be nowhere near each other. **This one needs a headset and I will not guess it** —
but note that the engine's 0.50 m latch is twice as wide as `rsqdg_swap_m`, so §2.3 fires in this
pose whether or not the swap does.

---

## 3. THE RIGHT SHAPE — a protocol, designed

### 3.1 What it cannot be, from the series rule

Each mod works alone, they talk through services, never hard links, any combination must load. So:

- **Not a shared library.** A `#include`d common file is a hard link, and two mods shipping
  different copies of it is a class-name collision at load.
- **Not a new required dependency.** No mod may stop working because the publisher is absent.
- **Not a hub.** Today's de facto hub is `RSQTR_RigService.GetInt("rig.busy", ...)`, which already
  aggregates the TacticalRig's own holsters, the cigarette, the Zippo **and** DistanceGrab's doors
  by name (`hooks.zs:68-79`). It is the closest thing the series has to this protocol and it fails
  as one for three reasons: the arbiter is itself a claimant; every new claimant needs an edit to
  the TacticalRig; and with the TacticalRig absent **every** answer vanishes, including the ones
  about other mods.
- **Not N² pairwise lookups.** 15 mods, each naming the others, is the shape §2 is the bill for.

### 3.2 What a claim is

**`RSQ_HandsService`**, published by every mod that touches a hand, each answering only for itself.
Four requests. Hand in `intArg` (0 main, 1 off); pawn in `objectArg`; `-1` always means "I do not
answer that", which is what an absent or older publisher gives for free.

| Request | Returns |
|---|---|
| `GetInt("hand.claim", "", hand, 0, pawn)` | **priority 1–100** if this publisher is using that hand right now; **0** if not; `-1` if it does not implement the protocol |
| `GetString("hand.owner", "", hand, 0, pawn)` | a stable dotted id — `"wheel.ring"`, `"grab.carry"`, `"grapple.climb"` — or `""` |
| `GetInt("hand.exclusive", "", hand, 0, pawn)` | **1** if no other mod may use the hand at all; **0** if the claim is advisory (gesture-level only) |
| `GetInt("hand.until", "", hand, 0, pawn)` | `level.maptime` the claim self-expires at, or **0** for "until I say otherwise" |

**Why a priority and not a boolean.** §1.3's table is the argument: "busy" is not one question. A
carried barrel and a braced rifle are both "busy" and they are not comparable. A number lets a mod
decide *against what* it stands down without anyone agreeing on a taxonomy.

**Why an owner string as well.** Two things. It makes a log readable — the whole of §2 is invisible
today because nothing names who took a hand. And it lets a mod make one exception without
understanding the whole scheme: *"stand down for everything except `grab.carry`, because carrying a
barrel in the other hand does not stop me."*

**Why `exclusive`.** §2.1 is a gesture collision — the Wheel does not need DistanceGrab to stop
*drawing*, only to stop *arming*. §2.5 is the other kind: a hand on a rope 20 m up must not be
driven by anyone. One bit separates them, and without it every claim has to be the strict one and
the series gets stiff.

**Why `until`.** Self-healing, the same reason `freezetics` beats `tics = -1` (§8 below, and
`04_DISTANCEGRAB.md` §3): a claim with a deadline cannot outlive the mod that made it. `near.zs`
already has the shape — `collarEnd[n]` (`near.zs:326`, enforced `:349`) bounds the Headlock to 3 s.

**The priority band, proposed, so the numbers mean the same thing to everyone.** This is the one
piece that needs the owner's blessing, because a shared scale is a shared convention:

| Band | Meaning | Examples |
|---|---|---|
| 90–100 | the hand is physically committed; a wrong answer injures the player | `grapple.climb` (80, see R2), a vehicle's handlebars |
| 70–89 | a continuous mechanic is running on that hand | `wheel.ring`, `grapple.hook`, `grapple.whip`, `grab.carry` |
| 40–69 | a hand is inside a UI or a holster volume | `rig.holster`, `rig.radial`, `grab.door` |
| 10–39 | a gesture is armed but nothing is held | `grab.reach`, a saber's lit blade |
| 0 | free | an empty hand, a dark hilt, a holstered gun |

### 3.3 Who arbitrates, and on what rule

**Nobody. Each consumer arbitrates for itself, against the others, every time it asks.** That is
the only shape that survives the series rule, and it is one loop:

```zscript
// Highest claim on this hand from anyone who is NOT me. Returns 0 when nothing
// else is loaded, which is exactly today's behaviour.
static int ForeignClaim(PlayerPawn pmo, int hand, String mine, out String owner)
{
    int best = 0; owner = "";
    let it = ServiceIterator.Find("RSQ_HandsService");
    Service s;
    while (s = it.Next())
    {
        String who = s.GetString("hand.owner", "", hand, 0, pmo);
        if (who == "" || who == mine) continue;        // my own publisher, or nobody
        int pri = s.GetInt("hand.claim", "", hand, 0, pmo);
        if (pri > best) { best = pri; owner = who; }
    }
    return best;
}
```

`ServiceIterator.Find(String)` returns every service of that class name, so a mod finds every other
publisher and skips its own by owner string. No mod names another mod. **Adding the sixteenth mod
to the series requires no edit to any of the fifteen.**

The rule a consumer applies is its own, and it is two lines at the top of the gesture it already
has. For DistanceGrab's `HandBusy`:

```zscript
String owner;
int foreign = ForeignClaim(p.mo, hand, "grab", owner);
if (foreign >= 40) return true;                       // a holster, a ring, a rope: stand down
if (foreign >= 10 && Num("rsqdg_yield_pri", p, 10) > 0) return true;   // another gesture: cvar
```

**Ties.** Equal priority means neither yields, which would be a deadlock if claims were requests.
They are not — they are **reports**. A mod publishes what it is already doing and consults before
*starting*. So a tie can only arise if two mods start in the same tic, and the one that reads
second sees the first's claim because the first publishes before it acts. If the owner wants a
tie-break anyway, the owner string sorts lexicographically and that is deterministic and
documented; it is not needed for correctness.

### 3.4 Staleness — the Wheel's `HasAnchor` lesson, generalised

A claim must never be *newer* than the state it describes, for the reason
`Wheel/zscript/wheel/zscript.zs:175-184` gives about `IsOpen` vs `mAnchor`. Two rules:

1. **Publish the claim from the same place that sets the state**, not from a separate tick. Both
   the Wheel and DistanceGrab already read their state through a `Get()` on an `EventHandler`, so
   the service is a pure reader and this is free.
2. **A consumer that acts on hand *geometry* as well as the claim asks the publisher whether its
   geometry is resolved.** The Wheel's `HasAnchor` is exactly that question. Keep it; add nothing.
   A consumer that only needs "is it claimed" does not need to care.

### 3.5 A mod that has never heard of the protocol

**It behaves exactly as it does today, and this is the load-bearing property.**

- A non-publisher appears in no `ServiceIterator.Find("RSQ_HandsService")` listing, so
  `ForeignClaim` returns 0 and every consumer behaves as it does now. **With nothing else loaded,
  the loop runs zero iterations and the mod is unchanged.** That is the degradation path and it
  needs no code.
- A non-consumer keeps whatever private predicate it has. Everyone else still yields *to* it the
  moment it publishes — which is why **publishing is more valuable than consuming** and should
  come first in every mod (§4).
- An older publisher that answers `hand.claim` but not `hand.exclusive` or `hand.until` gets `-1`
  from `Service`'s default implementation. **Treat `-1` as the safe default**: `exclusive -1` → read
  as 0 (advisory), `until -1` → read as 0 (no expiry). D26's convention inverted —
  there, "no answer means busy"; here, no answer means the *weakest* claim, because the publisher is
  volunteering information rather than defending itself.
- **`-1` from `hand.claim` and `0` must be distinguished and both must be ignored.** `-1` is "I am
  not a hands publisher at all" (the request fell through to another service's `GetInt`); `0` is "I
  am, and I am not using that hand". The loop above treats both as no claim, which is correct, but
  the distinction matters for a debug print.

### 3.6 Release, and a claiming mod unloaded mid-game

**This is where a protocol usually rots, and the answer is that nothing is ever stored.**

A claim is **not state**. `hand.claim` is a function of the publisher's own live fields —
`mOpen`/`mPokeHand` for the Wheel, `held[h]`/`armed[h]` for DistanceGrab, `h.gripHand`/`h.phase`
for the tether. **There is nothing to release.** The mechanic ends, the field clears, the next
`GetInt` answers 0. No registry, no reference counting, no cleanup path to miss.

Consequently:

- **Mod unloaded mid-game** (a `-file` set changed, a save loaded without it): the service class is
  gone, `ServiceIterator.Find` does not list it, every claim it made evaporates in one tic. The
  failure mode is "a hand becomes free", which is the safe direction.
- **Mod still loaded but its handler is gone or confused** (a save taken mid-mechanic): the
  publisher answers from fields that *were* serialized, so it can over-report. `hand.until` is the
  defence, and §2.2's bug is the live example: `near.zs`'s `gripDown[]` is synced and serialized
  while `keyState`/`eaten` are `transient ui` (`near.zs:36-40`), so a save taken mid-Headlock comes
  back with `gripDown` true and **no key-up able to clear it** (the release at `:72` compares equal
  and sends nothing). `collarEnd` bounds it to 3 s (`:349`) — luck, as
  `04_DISTANCEGRAB.md` §3 says, but the right shape. **Every claim carries a deadline for that
  reason.**
- **The player dies.** Every publisher already has a death path (`hold.zs:446-450`,
  `near.zs:172`, `zscript.zs:3218`), so the claim clears with the mechanic.

**The rule to write into the protocol's own header:** *a claim is computed, never stored; if you
find yourself writing a release path you have built a registry and it will leak.*

### 3.7 One new service, or an extension?

**One new service class name, `RSQ_HandsService`, published separately by each mod — and nothing
existing is removed.**

Why not extend what exists:

- **Not `RSQTR_RigService`.** §3.1: the arbiter would be a claimant, every new claimant an edit to
  the TacticalRig, and with the rig absent all answers vanish. `rig.busy` should stay exactly as it
  is and keep its two consumers; it is a good answer to "is this hand in my rig".
- **Not `wr_RigService`.** `IsOpen`/`OpenHand`/`HasAnchor` are about the ring and should stay. The
  Wheel's hand claim is a different question with a different answer (both hands, not `OpenHand`).
- **Not a new request on each mod's existing service.** That is the N² shape: a consumer would have
  to know all fifteen service names. **The whole value of a shared class name is that
  `ServiceIterator.Find` enumerates publishers a mod has never heard of.**

Why a separate class rather than one shared implementation: two mods cannot both define
`RSQ_HandsService` — a duplicate class name is a load-time error. **So each mod defines its own
class with a distinct name and the protocol is the *request set*, not the class.** Concretely:
`RSQDG_HandsService`, `wr_HandsService`, `RSQGR_HandsService`, each `: Service`, and
`ServiceIterator.Find` keyed on the **class name**. That breaks the enumeration above.

**So the mechanism needs settling before anyone writes a line, and it is the one open question in
this design.** Two routes:

- **(a) One shared class name, one definition, guarded.** The series already ships `rsmenu` as a
  shared `#include`d piece (`DistanceGrab/zscript.txt:3`, *"the series' shared menu pieces"*), so
  there is precedent for a file carried identically by several mods. Whether two pk3s can carry the
  same ZScript class and have one win rather than erroring is **not established in this survey and
  must be tested before this route is chosen.**
- **(b) Enumerate by convention on the owner string, not the class name.** Each mod names its
  service `<prefix>_HandsService` and a consumer iterates *every* service — which
  `ServiceIterator` cannot do; it takes a name. So (b) needs a different enumeration and does not
  work as stated.

**Recommended: route (a), gated on a one-afternoon load test** — two pk3s, both carrying an
identical `RSQ_HandsService` whose only job is to delegate to `EventHandler.Find` in each mod,
loaded together, and check for a class-redefinition refusal in the log. If it refuses, the fallback
is **(c): a known small list of prefixes**, iterating `RSQDG_`, `wr_`, `RSQGR_`, `RSQTR_`,
`RSQLS_`, `SSQ_`, `UBQS_` + `HandsService`, published as part of the protocol document and grown as
mods are added. (c) is worse — adding a mod means every consumer's list is stale — but it is
correct today and costs nothing. **Settle this first; it changes nothing else in the design.**

### 3.8 Who publishes it first

**The Wheel.** Three reasons:

1. **It claims the most and consumes nothing.** It takes both hands and the whole player
   (`CF_TOTALLYFROZEN`), is the only mod with a genuinely exclusive claim, and asks nothing of
   anyone about hands. `07_WHEEL.md` §9.4 reached the same conclusion: *"the Wheel is the mod that
   claims the most and is therefore the one that would benefit most."*
2. **Publishing alone fixes §2.1 and §2.5 once DistanceGrab and the tether consume** — and
   publishing is strictly additive, default-off in effect, and cannot break the Wheel.
3. **It is the cheapest publisher in the series.** `mOpen`, `mRigHand`, `mPokeHand` and
   `eitherHand()` already exist and `wr_RigService` is already a pure reader; the claim is a
   function of fields it has.

**Then DistanceGrab**, which is the biggest consumer and already has the idiom four times over
(D26, D28, the two `rig.busy` sites). **Then the tether**, whose publish is R2 and whose consume is
R3 — and it is being written now, so its half costs nothing extra if it is designed in rather than
retrofitted.

---

## 4. COST, HONESTLY, PER MOD

| Mod | Publish | Consume | Label | Note |
|---|---|---|---|---|
| **Wheel** | **S** — one class, four requests, all from existing fields; both hands claimed `wheel.ring` priority 75 `exclusive 1` whenever `mOpen`, plus `mPokeHand` at 85 | **S**, and optional — the Wheel has nothing to ask about hands today | **BUILDABLE / S** | Do this first. Nothing in the Wheel changes behaviour. |
| **DistanceGrab** | **S** — `grab.carry` 75 when `hold.HandIsFull(h)`, `grab.reach` 20 when `armed[h]`, `grab.door` 50, `grab.collar` 75; all four are existing fields | **M** — two lines in `HandBusy`, two in `near.zs`'s `RigBusy`, **and reconciling the two predicates into one** (§2.6) | **BUILDABLE / M** | The M is the reconciliation, not the protocol. |
| **tether / whip** | **S** — §0.2 R2, from `h.phase` and `h.gripHand` | **S** — §0.2 R3, inside the existing `HandFree` | **BUILDABLE / S** | Cheapest of the three *if designed in now*; M if retrofitted. |
| **TacticalRig** | **S** — wrap the existing `rig.busy` as `rig.holster` priority 55; it already aggregates | **S** | **BUILDABLE / S** | Not required, but it is the mod every consumer already asks, so publishing retires two one-off strings. |
| **LaserSight** | **S** — `laser.beam` is already per-hand; priority 15, advisory | — | **BUILDABLE / S** | Retires DistanceGrab's `rsqdg_beam_yield` special case (D28). |
| the shared class-name test (§3.7) | — | — | **EXISTS / S** | **Gate on this before writing any of the above.** |

**Per-tic cost.** `ServiceIterator.Find` + up to N `GetInt`/`GetString` per hand per gesture test.
`07_WHEEL.md` §8.6 establishes the standard: *"`ServiceIterator.Find` is never in a per-tic path"*
in the Wheel, and it is careful about that. **DistanceGrab's `HandBusy` already does two
`ServiceIterator.Find` calls per hand per tic** (`reach.zs:163`, `:178`), from `PalmOk`
(`reach.zs:108`), from `grab.zs:352` — so the protocol adds one more Find and ~4 string-keyed
`GetInt`s to a path that already pays two. That is not free and it is not new. **Cache the
publisher list once per tic in the consumer's `WorldTick`** — the claim is cheap, the enumeration
is not — and the added cost is a handful of virtual calls. Measure it the way `07_WHEEL.md` §8.1
prescribes (`profilethinkers -t 20`), not by reasoning.

---

## 5. THE ENGINE'S OWN UNARBITRATED CLAIM, and how the protocol should treat it

`00_CAPABILITIES` gotcha 10a is itself a claim on the off hand, made by the engine, that no mod is
consulted about. Verified in this tree:

- `vr_two_handed_weapons` defaults **true** — `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:119`.
- The latch: off-hand grip **rising edge** with the controllers inside `0.50f` raw tracking metres —
  `src/rendering/gl/stereo3d/gl_openvr.cpp:1716-1728`, the distance computed at `:1709-1714`.
  Release clears it with **no distance test** — `:1725-1727`.
- While latched, the main hand's **pitch and yaw are overwritten** with the hand-to-hand line and
  **roll is passed through verbatim** — `:1742-1753` (the `VectorSet` at `:1751-1752` takes
  `weaponangles[ROLL]` unchanged as its third argument).
- The off-hand weapon goes dark: fires nothing (`player.zs:393-396`, `:426-429`), checks no buttons
  (`p_pspr.cpp:755-757`), renders nothing (`hw_weapon.cpp:832-835`, `:917-920`), re-asserted on
  every state set (`p_pspr.cpp:512-515`).

**Three rules, and the first is the one that matters.**

**(1) The protocol must publish it, and the publisher must be the first mod to load, not a special
case in every consumer.** It is a real claim on a real hand and every mod needs to see it. It should
appear in the enumeration like any other:

> owner `"engine.twohand"`, priority **95**, `exclusive 1`, `until 0`, claimed on **both** hands
> whenever `player.WeaponState & WF_TWOHANDSTABILIZED`.

Priority 95 because no mod can clear it (`p_pspr.cpp:514`) and nothing a mod does can win against
it. Both hands because it blanks the off hand's weapon *and* hijacks the main hand's pitch and yaw
(§2.3) — a claim on one hand would be a lie about the other, and that lie is exactly what §2.3 is.

**Who publishes it.** Any publisher can, from one bitwise test — `PlayerInfo.WeaponState` and
`WF_TWOHANDSTABILIZED` are both script-visible (`player.zs:393`, `constants.zs:1242`). So **every**
publisher should fold it into its own answer: `max(myClaim, engineClaim)`. That is a two-line
addition to each, it needs no coordination, and with any single hands-publisher loaded the engine's
claim is visible. **It is the one claim that must not depend on one named mod being present**, and
redundant publication is how that is guaranteed.

**(2) The claim is a report about the engine, so `hand.owner` must distinguish it.** A consumer
needs to tell "another mod is using this hand" (negotiable; the other mod may yield next tic) from
"the engine has it" (not negotiable; never try to clear it — `00_CAPABILITIES` 10a, verified at
`p_pspr.cpp:512-515`). The `engine.` prefix is the signal, and it should be reserved in the
protocol document.

**(3) A consumer that reads hand *angles* needs to know which ones survived.** This is the
`WHIP_HANDOFF.md` §1.5 finding and it is the most useful fact in this section: **under the latch,
roll is clean and pitch and yaw are not.** So the protocol should carry it, as a fifth optional
request rather than making every mod re-derive it:

> `GetInt("hand.posevalid", "", hand, 0, pawn)` — a bitmask of which components of that hand's pose
> are trustworthy this tic: `1` position, `2` yaw, `4` pitch, `8` roll. The engine's two-hand
> publisher answers `1|8` for the main hand under the latch and `1|2|4|8` otherwise. An absent
> publisher answers `-1`, which a consumer reads as "all valid" — today's behaviour.

That one request would have prevented §2.3 outright, and it is what lets the whip's roll-derived
flick be *provably* safe rather than safe by the author having read gotcha 10a.

**What the protocol must NOT do about it.** It must not try to clear the latch
(`p_pspr.cpp:514`), must not route around it by geometry (the latch survives the hands separating,
`:1725-1727`), and must not make `vr_two_handed_weapons` a mod's business. **The defence is to
know, and to stand down.**

---

# PART TWO — FINISHING THE DISTANCEGRAB PASS

`04_DISTANCEGRAB.md` established the baseline; these are the specifics, in its priority order.
Where a choice is already recorded in `docs/DECISIONS.md` it is cited and left alone.

**`near.zs` has no decision record** — `docs/DECISIONS.md` ends at **D30 (2026-10-27)** and
`near.zs` is dated the owner, 10-29. §11 lists what it needs documented.

---

## 6. The Headlock's `bTHRUACTORS` — the exact fix

### 6.1 The finding, verified in this tree

`near.zs:324` sets `a.bTHRUACTORS = true` on a headlocked monster. `PIT_CheckThing`:

```cpp
src/playsim/p_map.cpp:1405-1410
    // don't clip against self
    if (thing == tm.thing)
        return true;

    if ((thing->flags2 | tm.thing->flags2) & MF2_THRUACTORS)
        return true;
```

`return true` from `PIT_CheckThing` means "no collision, carry on". It is the **OR of both actors'**
flags, and it is the **second** test in the function — before the
`MF_SOLID|MF_SPECIAL|MF_SHOOTABLE` test (`:1414`), before the square-prism `blockdist` test
(`:1417-1419`), and 180 lines before the missile branch at `:1589` with its height gate, its
`CanAttackHurt` at `:1672` and its `P_DoMissileDamage` at `:1694`/`:1710`. **Every fireball,
revenant missile and cacodemon ball passes straight through the monster you are holding in front of
your face, on every map.**

Hitscans do stop at it, because the trace path never reads the victim's flag: the mask is
`MF_SHOOTABLE` (`p_map.cpp:4919`), the trace's own actor test is `thing->flags & ActorMask`
(`src/playsim/p_trace.cpp:866`), and the skip filter consults the **puff's** `ThruActors`
(`p_map.cpp:4829` → read at `:4690`). **That asymmetry is why this is silent:** in a room of
zombiemen the shield is perfect and the first imp looks like a different bug.

### 6.2 The fix

**Stop setting it. One line.**

```diff
  near.zs:321      cNoGrav[n] = a.bNOGRAVITY;
  near.zs:322      cThru[n]   = a.bTHRUACTORS;
  near.zs:323      a.bNOGRAVITY = true;
- near.zs:324      a.bTHRUACTORS = true;
+ near.zs:324      // NOT bTHRUACTORS. PIT_CheckThing returns "no collision" on the OR of both
+                  // actors' MF2_THRUACTORS (p_map.cpp:1409-1410) -- second test in the function,
+                  // before the height gate, before CanAttackHurt and before P_DoMissileDamage.
+                  // Setting it makes a human shield that stops bullets and stops no projectiles,
+                  // and the hitscan half works perfectly (p_trace.cpp:866 reads only ActorMask),
+                  // so the bug reads as "shields work" until the first imp.
+                  a.bFORCEINFIGHTING = true;    // see below: the second leak, invisible until this one is fixed
  near.zs:325      a.Vel = (0, 0, 0);
```

**Keep the save and the restore.** `cThru[n]` at `:322` and the restore at `:338` stay exactly as
they are — DistanceGrab's own discipline is *"whoever borrows is the single owner of the restore"*
(D7, `docs/DECISIONS.md:117-130`, which says the desktop tree states that rule three times because
it bit three times). Something else may have set the flag; the Headlock must hand back what it
found.

### 6.3 Is dropping it safe? The `TryMove` claim, checked

**Yes, and the claim in `04_DISTANCEGRAB.md` §9 is correct.** `near.zs:358`:

```zscript
if (!a.TryMove(at.xy, true)) { }
```

The empty body is deliberate and it is the right behaviour. `TryMove` is collision-tested and
interpolating; a blocked move leaves the monster where it was, which is what you want for a held
body pressed into a wall or into the player's own cylinder. The hold is then bounded two ways: the
tear-away test one line above (`near.zs:357`, `Metres(at - a.Pos) > 1.5` → `EndCollar`) and
`collarEnd` (`:326`, enforced `:349`). So with the flag dropped, the worst case is the monster
refuses to follow, falls 1.5 m behind the palm, and the hold ends — which is exactly what a human
shield torn out of your hands should do.

**The one case the flag was guarding, and it needs a headset.** The monster is placed at
`hp + fw * (a.Radius + 6.0)` with `at.z -= a.Height * 0.5` (`near.zs:355-356`). At point-blank,
with the off hand drawn back to the chest, that target sits inside the player's own 2r × 2r square
footprint (`p_map.cpp:1417-1419` — and gotcha 9: the footprint is a square, so diagonal reach is
√2 larger than you expect). `TryMove` will then refuse every tic and the monster will lag at arm's
length rather than following in. **That is arguably correct behaviour** — you cannot pull a demon
through your own chest — but it is a feel change and it should be looked at in the headset in the
same sitting. It is also the reason to raise the `+ 6.0` stand-off if it reads badly, rather than to
put the flag back.

### 6.4 The second leak, which (6.2) makes visible

`+FORCEINFIGHTING` is unset on the held monster, and it sits **behind** the `bTHRUACTORS` leak:

```cpp
src/playsim/p_map.cpp:1275-1282
static bool CanAttackHurt(AActor *victim, AActor *shooter)
{
    // players are never subject to infighting settings ...
    if (!victim->player && !shooter->player)
    {
        int infight = victim->Level->GetInfighting();
        if (infight < 0 && (victim->flags7 & MF7_FORCEINFIGHTING)) infight = 0;
```

Called at `p_map.cpp:1672` as `CanAttackHurt(thing, tm.thing->target)` — victim is the held
monster, shooter is the missile's **owner**. Neither is a player, so the infighting rules apply in
full, and on a `NoInfighting` map or skill `infight < 0` waves the missile through. The flag is read
off the **victim**, so it goes on the held monster, which is what the diff in §6.2 does.
`MF7_FORCEINFIGHTING` is a real flagdef — `src/scripting/thingdef_data.cpp:318`,
`DEFINE_FLAG(MF7, FORCEINFIGHTING, AActor, flags7)`, bit at `src/playsim/actor.h:398`.

**Two details worth having.** `CanAttackHurt` only runs `if (tm.thing->target != NULL)` (`:1665`),
so a missile whose owner is gone skips the gate anyway. And the flag needs the same borrowed-state
treatment as `bNOGRAVITY`: save at `StartCollar`, restore at `EndCollar`, beside `cNoGrav[n]`.

**Do (6.2) and (6.4) in one change**, or (6.2) will look like it did not work on the maps that set
the flag. **BUILDABLE / S**, ~6 lines together.

---

## 7. The swung-object hit test — the advertised feature that does not exist

### 7.1 Today

`hold.zs:179` sets `a.bTHRUACTORS = true` for the whole hold, so the carried object cannot collide
with a monster at all, and nothing tests a swing: damage exists only for a **thrown** object
(`throw.zs:472-535`). Walls still refuse it — `bTHRUACTORS` filters actors only
(`p_map.cpp:1409`) — which is how strain and pinning work (D22). So the hold is right about
geometry and silent about bodies, and the package's own description ("then carry them, throw them,
and hit things with them") is not true.

### 7.2 The flag can stay, and here is the citation that proves it

`04_DISTANCEGRAB.md` §7 decision 1 says `bTHRUACTORS` must stay — otherwise a held barrel shoves
monsters through the spring and the strain reads every body as a wall. **The reason that is
compatible with a `LineTrace` hit test is one line of engine, and it should be in the comment:**

```cpp
src/playsim/p_map.cpp:5173    TData.ThruActors = !!(flags & TRF_THRUACTORS);
```

`P_LineTrace`'s actor filter reads the **trace's own flag**, never the source actor's
`bTHRUACTORS`. The filter callback (`CheckLineTrace`, `p_map.cpp:5145-5161`) consults only
`TData->ThruActors` and `TData->ThruSpecies`. So tracing *from* a `bTHRUACTORS` object is not
blinded by its own flag, and tracing *into* a `bTHRUACTORS` victim still hits it. **The script hit
test and the engine pass-through coexist by construction, not by luck.**

### 7.3 The sweep, with the recipe corrected

`01_MELEE.md` §2's swept chord, with two corrections to the recipe as written in
`04_DISTANCEGRAB.md` §7, both verified:

```zscript
// In RSQDG_Hold.WorldTick, after strainM[h] is computed and before the spring write.
// p0 = lastObj[h] (last tic's position), p1 = a.Pos (where the engine actually got it to).
Vector3 ch = p1 - p0;
double L = ch.Length();
if (L > 0.01)
{
    double fatten = RSQDG_Throw.BodyRadius(a);
    FLineTraceData d;
    a.LineTrace(atan2(ch.Y, ch.X), L + fatten, -asin(ch.Z / L),
                TRF_ABSPOSITION,                 // see (a) and (b) below
                p0.Z, p0.X, p0.Y, d);
    ...
}
```

**(a) Drop `TRF_SOLIDACTORS` unless props are wanted.** `p_map.cpp:5248-5249`:
`aflags = MF_SHOOTABLE` by default and `TRF_SOLIDACTORS` adds `MF_SOLID`. Monsters are
`MF_SHOOTABLE`, so the default mask already finds them; adding `MF_SOLID` makes the swing stop on
non-shootable solid props and lose the monster behind. Keep it only if the owner wants a swung
barrel to clunk off a pillar actor.

**(b) Drop `TRF_BLOCKSELF`.** `p_map.cpp:5252-5261` shows it affects **line** flags only, never
actors, and it reads `t1->player` / `t1->flags` — so with the *held object* as the trace source it
adds `ML_BLOCKMONSTERS` (`:5258`, since `NotBlocked` is false for an ordinary barrel) and the swing
stops at a monster-blocking line. For a swing inside arm's reach that is wrong.

**(c) The origin arithmetic is right as written.** `TRF_ABSPOSITION` makes
`startpos = DVector3(offsetforward, offsetside, sz)` (`p_map.cpp:5206-5209`), so passing
`sz = p0.Z, offsetforward = p0.X, offsetside = p0.Y` yields `startpos = (p0.X, p0.Y, p0.Z)`.
Verified. And the start sector is `t1->Level->PointInSector(startpos)` (`:5269`), taken from the
absolute point, so there is no stale-sector trap.

**(d) The source actor is ignored, the player is not.** `Trace(..., t1, ...)` at `:5269` passes the
trace source as the ignore actor, so the held object cannot hit itself. **The player is not
ignored**, and the hold's target is a stand-off ahead of the palm (`hold.zs:399-417`), so a swing
across the body puts the chord through the player's own cylinder. **Use the mod's existing
predicate, do not take `d.HitActor` raw:** `RSQDG_Throw.CanHurt(t, a, by)` (`throw.zs:453-456`)
already excludes `t == a` and `t == by` and requires `bSHOOTABLE && Health > 0 && !bDORMANT`.

**(e) One new field.** The hold stores `lastPalm[h]` and `lastBody[h]` (`hold.zs:97-99`) but not the
object's own previous position. Add `lastObj[h]`, written beside them at `hold.zs:492-494`. **Use
the object's travelled chord, not the palm's** — it is the real, collision-tested travel, so sag
(`hold.zs:427-434`) and spring lag are honoured for free, which is also what gate (2) below wants.

**(f) Sub-chords.** `01_MELEE.md` §3.5's `SUBS = 3` covers the arc's sagitta (a 70 mu blade through
60° in one tic bulges 9.4 mu past its chord). For a carried object on a spring the arc is gentler
than a swung blade, so **start at `SUBS = 2` and raise it if the owner can swing past a lost soul**
— it halves the budget below for an effect that may not be visible.

### 7.4 Trace cost per tic

One `LineTrace` is one `FPathTraverse` — `01_MELEE.md` §3.4's table calls it "same order as a
hitscan bullet, with **no** side effects", and §7.2 above confirms the only work is the filter
callback and the actor mask.

**The budget, bounded by hands and not by content:**

| | SUBS = 2 | SUBS = 3 |
|---|---|---|
| per hand per tic | 2 | 3 |
| both hands, per tic | 4 | 6 |
| **per second (35 tics)** | **140** | **210** |

And it is **gated three ways before a trace is issued**: only a hand with `held[h] != null`
(`hold.zs:457`), only when `good` (the pose is valid, `:470`), and only past the speed gate below.
So the common case is **0 traces/s** and the worst case is 210 — two objects held and both being
swung. `01_MELEE.md` §3.4 calls that figure "nothing". It also replaces the point-sampled
thrown-pickup overlap test (`throw.zs:579-601`, pad 0) with the same code, so the §4 watch-item in
`04_DISTANCEGRAB.md` comes free.

### 7.5 How it reuses what the mod already owns — nothing new is invented

| Piece | Already exists | Used by the swing as |
|---|---|---|
| **palm history** | `aim.zs:213-235`, `PalmVelocity(hand)` — peak of 7 one-tic deltas, turned back into world by `frameYaw` (`:233-234`) | **not** the hit chord (§7.3e) — but the right input to the *speed gate*'s fallback when the object is pinned and the hand is not |
| **kilograms** | `RSQDG_Mass.Kg(a)`, `mass.zs:62`; guessed from class defaults never the live box (`mass.zs:175-180`), because the carry thins it | the mass term in the energy |
| **energy scorer** | `RSQDG_Throw.DamageFor(kg, speedMps, p)`, `throw.zs:442-451` — `½mv²/rsqdg_throw_j`, bounded before `int()` for Champions' `0x7FFFFFFF` Mass | **unchanged, called directly.** A thrown barrel and a swung barrel then cannot disagree about what a barrel weighs. |
| **the Doom-gravity term** | `throw.zs:427-450`, vertical scaled by `sqrt(9.81/g_doom)` (D23) | inherited by using `DamageFor` |
| **per-victim ledger** | `wHit[]` + `AlreadyHit(i, t)` + `MarkHit`, `throw.zs:65`, `:397-409` | the same array shape, one slot per hand, cleared every `rsqdg_swing_cool` tics |
| **the strike** | `RSQDG_Throw.Strike(i, t, speedMps, p)`, `throw.zs:472-486` — object as inflictor, player as source, so kill credit, infighting and kick direction stay right | **call it.** It also fires `RSQDG_Events.Tell("throw.hit", ...)` (`:485`) — the swing wants its own tell, `"swing.hit"`. |
| **collect-then-strike** | `throw.zs:497-512`, with the comment explaining why: damage inside an iterator can gib an actor and unlink blockmap nodes the iterator has already stepped to | a `LineTrace` returns one actor so the hazard is smaller, but with SUBS sub-chords the same rule applies: **collect, then strike** |
| **haptics** | `feel.zs:328-369` pulse route, `:446-480` level route; `OnImpact` is empty at `:433` by the decision at `:431` | **a swung object has not left the hand, so it should buzz.** `OnImpact`'s emptiness is right for a *throw* (D14/D21 vocabulary: a thrown thing is heard, not felt) and wrong for a swing. Pulse on the swinging hand, scaled by `Heft` (`mass.zs:210-215`) on the rumble route. |
| **hit-stop** | — | `victim.freezetics = 4` (~114 ms, `01_MELEE.md` §3.6). **On the victim, never the player** — hands are tracked outside the playsim, so a frozen player is a player whose hands stop registering while the world keeps rendering. |

**The three gates, which are design and need the owner:**

1. **`bTHRUACTORS` stays** — §7.2, and now with the engine citation that makes it safe.
2. **A speed gate, measured off the object.** `rsqdg_swing_mps`, 3 m/s to match `rsqdg_throw_min`'s
   default, from `Metres(ch)` per tic rather than from the hand — so carrying a barrel past an imp
   is not a hit and sag is honoured. `RSQDG_Reach.Metres` (`reach.zs:262-291`) is the right
   converter and already applies `PIXELSTRETCH` to Z.
3. **Hit-stop and cooldown** — `rsqdg_swing_cool` tics, one hit per victim per window, so a barrel
   resting against a monster does not tick damage at 35 Hz.

**Label: BUILDABLE. Cost: M** — the sweep is S; the gates, the ledger wiring, the haptic route and
headset tuning make it M.

---

## 8. `freezetics` for the Headlock

### 8.1 Verified, and it is the second thing in `Tick`

```cpp
src/playsim/p_mobj.cpp:3910   void AActor::Tick ()
                   :3930-3943   // unmorph check (only for a morphed non-player)
                   :3948-3952   if (freezetics > 0)
                                {
                                    freezetics--;
                                    return;
                                }
```

Nothing but the unmorph check runs before it — not states, not `MF5_NOINTERACTION`'s minimal path
(`:3964`), not velocity, not Z, not portals, not AI. Declared `native uint freezetics;` at
`wadsrc/static/zscript/actors/actor.zs:265`, **not readonly**, so it is writable from script. The
actor **stays in the blockmap**, which is the whole point of a shield.

### 8.2 Why it beats `near.zs:361`'s `a.tics = -1`

`04_DISTANCEGRAB.md` §3's three counts hold and one needs a correction:

1. **`tics = -1` does not stop `Tick`.** It stops the state counter from expiring; the whole
   movement, Z, portal, crush and water block still runs every tic. Nothing visibly breaks today
   only because `Vel` is zeroed (`near.zs:360`) and `bNOGRAVITY` is set (`:323`) — the mod is
   hand-doing, less completely, what one field does outright.
2. **It is a latch, so it needs a restore path that can be missed.** `near.zs:339`
   (`if (a.Health > 0 && a.tics < 0) a.tics = 1;`). `freezetics` is self-healing: stop writing it
   and the monster resumes within two tics, so **no exit path can leave a permanently frozen
   monster.** The mod states the principle itself at `hold.zs:541-542`: *"a flag set in one place
   and cleared in another is the bug that outlives the session."*
3. **The save path.** §3.6 above: `gripDown[]` is serialized, `keyState`/`eaten` are
   `transient ui` (`near.zs:36-40`), so a save taken mid-Headlock returns with `gripDown` true and
   no key-up able to clear it (the release at `:72` compares equal and sends nothing). Bounded to
   3 s by `collarEnd` (`:349`) — luck rather than design.

   > **Correction to `04_DISTANCEGRAB.md` §3.** It says `near.zs` has *"no `WorldLoaded`/`OnRegister`
   > handling at all"*. `WorldUnloaded` **does** exist, at `near.zs:373-376`, and it calls
   > `EndCollar` for every player. It is `WorldLoaded` that is missing, and that is the one the save
   > path needs.

### 8.3 The change

In `TickCollars`:

```diff
  near.zs:360      a.Vel = (0, 0, 0);
- near.zs:361      a.tics = -1;
+ near.zs:361      // freezetics, not tics = -1. freezetics early-returns out of AActor::Tick
+                  // (p_mobj.cpp:3948-3952) before states, velocity, Z and AI, while KEEPING the
+                  // blockmap link -- so a held monster is still shootable, which is the whole
+                  // point of a shield. Re-asserted each tic: WorldTick runs before RunThinkers
+                  // (p_tick.cpp:171-173) so the monster's own Tick consumes one and returns.
+                  // Self-healing: stop writing it and it resumes in two tics, so no exit path
+                  // can leave a monster frozen for ever (hold.zs:541-542's own rule).
+                  a.freezetics = 2;
```

and:

- **Drop the `tics` restore** at `near.zs:339`.
- **Add a `WorldLoaded` override** that calls `EndCollar(n)` for every player, mirroring
  `WorldUnloaded` at `:373-376` and `door.zs:59-68`, which explicitly notices a load and releases.
- **Keep the frozen-frame behaviour.** `near.zs:351-352` wants *"a punch's pain frame freezes too:
  it reads as stunned"* — `freezetics` stops the state advancing just as surely, so that is
  preserved.

**Why `2` and not `1`.** `WorldTick` runs at `src/p_tick.cpp:171`, `RunThinkers` at `:173`, so the
write lands before the monster's own `Tick` consumes it. `2` leaves one tic of slack if a handler
order ever changes; `1` would be exact and fragile.

**The same field, the other use:** the hit-stop in §7 (`victim.freezetics = 4`).

**Label: BUILDABLE. Cost: S.**

---

## 9. `FindNear`'s per-tic cost — and the fix is stronger than §9c said

### 9.1 The cost

```zscript
near.zs:124   double nearU = RSQDG_Reach.MetresToUnits(RSQDG_Reach.Num("rsqdg_near_m", p, 0.30));
near.zs:132   let it = BlockThingsIterator.CreateFromPos(hp.x, hp.y, hp.z, 128, 128, false);
```

`rsqdg_near_m` 0.30 m is **10.2 map units** horizontally. The iterator's `checkradius` is a
**half-extent**, not a diameter — `FMultiBlockThingsIterator`'s constructor stores it as
`checkpoint.Z = checkradius` (`src/playsim/p_maputl.cpp:1084`) and `startIteratorForGroup` calls
`bbox.setBox(offset.X, offset.Y, checkpoint.Z)` (`:1142`), so the box is **x ± 128, y ± 128: 256
units across**. Blockmap cells are 128 units (`src/gamedata/p_blockmap.h:46`,
`static constexpr int MAPBLOCKUNITS = 128`), so that is a **2×2 to 3×3 cell walk, per hand, per
tic, to find things within 10.2 units**.

Call sites: `WorldTick` → `:183` (per hand, and `||` short-circuits so it is skipped when swap,
`gripDown` or a door already made the hand ready), and again in `NetworkProcess` → `:214` on the
press. Worst case 2 calls/tic = 70/s. `06_GRAB.md` §5 names exactly this: *"The real cost is a
per-tic `BlockThingsIterator` if you scan for grab candidates continuously."*

### 9.2 The right half-extent is `nearU`, not `nearU + 64`

`04_DISTANCEGRAB.md` §9 recommends `nearU + 64` — *"enough for the biggest monster's radius"*. That
reasoning is unnecessary, and the padding is not needed, because **an actor is linked into every
blockmap cell its radius touches, not just its centre cell**:

```cpp
src/playsim/p_maputl.cpp:518-521
    int x1 = Level->blockmap.GetBlockX(pos.X - radius);
    int x2 = Level->blockmap.GetBlockX(pos.X + radius);
    int y1 = Level->blockmap.GetBlockY(pos.Y - radius);
    int y2 = Level->blockmap.GetBlockY(pos.Y + radius);
    ...
src/playsim/p_maputl.cpp:527-529
    { // [RH] Link into every block this actor touches, not just the center one
```

`FindNear`'s own test subtracts the radius (`near.zs:139`, `hz = max(0, dxy.Length() - a.Radius)`),
so an actor qualifies when its **cylinder** comes within `nearU` of the palm. Its cells span
`centre ± radius`, rounded outward to cell boundaries; the box's cells span `palm ± H`, likewise.
If the cylinders are within `nearU` then those intervals touch on both axes, so the two cell sets
share a cell — **so `H = nearU` finds every actor that could qualify, for any radius.** The
Euclidean test at `:139` is tighter than the per-axis bound, so qualification implies per-axis
overlap; the argument is sound.

**Recommendation: pass `nearU + 8`.** `nearU` alone is provably sufficient, and the `+ 8` is cheap
insurance against the rounding argument being tight in a diagonal corner case — it costs at most
one extra cell. That takes the half-extent from **128 to ~18**, which is a **1×1 or 2×2 cell walk
instead of 2×2 to 3×3** — a 4× to 9× reduction in cells examined, versus the ~1.5× the `+64` pad
would have given.

**Also check it against a very large corpse.** `rsqdg_corpses` is **on** by default since 10-29, so
a Spider Mastermind corpse (radius 128) is a candidate. With the old 128 it was found via its own
cell span; with `nearU + 8` it still is, by the same rule. Worth one rig check rather than taking it
on the arithmetic, because this is the one case where getting the bound wrong makes a grab silently
stop working.

### 9.3 The other three reductions

- **Skip the scan when the hand is full.** Already done at `near.zs:183`
  (`(!hold || !hold.HandIsFull(h)) && FindNear(...)`) — credit where it is due.
- **Cache `ready[]` across 3 tics.** `ready[]` is read only by `InputProcess` (`:76`), which fires
  on a key edge, not per tic. A 3-tic-stale `ready` is at most 86 ms of lag on a grip press, and the
  press re-derives the target anyway at `:214`. **Cheapest of the four.**
- **Hoist the cvar and `Get()` lookups out of `FindNear`.** Per call it does one `On(...)` plus
  three `RSQDG_Reach.Num(...)` (`:122`, `:124`, `:125`, `:126`) plus three `EventHandler.Find`
  round-trips (`RSQDG_Policy.Get()`, `RSQDG_Flight.Get()`, `RSQDG_Hold.Get()` at `:127-129`) —
  before a single candidate is examined. All seven are loop-invariant across both hands; resolve
  them once per `WorldTick`.

**Measure first.** `07_WHEEL.md` §8.1's gate applies here too: `profilethinkers -t 20` and
`stat think` with a hand near a crowd, in a Brutal Doom fight. Everything above is structurally
true; whether it is *felt* is a measurement, and the budget is 28.6 ms at 35 tics/s.

**Label: BUILDABLE. Cost: S.**

---

## 10. The `LineAttack` haptic route, and the inert puff

### 10.1 The replacement

`feel.zs:317-323` currently fires `pmo.SpawnPlayerMissile("RSQDG_Buzz", ...)` to provoke the
engine's own weapon buzz (D15 keeps `+MISSILE` off so Brutal22Q's recoil/flash hooks do not see a
shot). Replace with:

```zscript
private void Fire(int hand, PlayerPawn pmo)
{
    if (!pmo) return;
    pmo.LineAttack(pmo.angle, 0.1, 0, 1, 'None', "RSQDG_NullPuff",
                   LAF_NOINTERACT | ((hand == 1) ? LAF_ISOFFHAND : 0));
    busyUntil[hand] = MSTime() + BUSY_MS;
}
```

Verified against this tree:

```cpp
src/playsim/p_map.cpp:4758-4769
    if (t1->player != NULL)
    {
        pflag = DMG_PLAYERATTACK;
        if ( damage > 0) {
            long rightHanded = vr_control_scheme < 10;
            rightHanded = (flags & LAF_ISOFFHAND) ? 1 - rightHanded : rightHanded;
            vrmode->Vibrate(150, rightHanded ? 1 : 0, 0.8);
```

It tests the `damage` **parameter** and nothing else, *before* any tracing, and it is **not** gated
on `LAF_NOINTERACT` (read separately at `:4724`). Handedness comes from `vr_control_scheme` flipped
by `LAF_ISOFFHAND`, exactly as the `SpawnPlayerMissile` path's `ALF_ISOFFHAND` does. Gains: no actor
spawn, no `P_CheckMissileSpawn`, no `WorldThingSpawned` for any other mod to misread — **D15's whole
compatibility problem disappears** instead of being worked around by name. Same 150 ms, same 0.8,
same hand: it changes nothing about how the mod feels.

**One caveat worth knowing before reading a double pulse as a bug:** `p_map.cpp:4771-4773` buzzes
**both** hands at 0.6 when `weaponStabilised` is set. That is the engine, and it is §5's claim
showing up in the haptics.

### 10.2 The puff still spawns, so it must be inert

```cpp
src/playsim/p_map.cpp:4930-4937
    if (nointeract || (puffDefaults && puffDefaults->flags3 & MF3_ALWAYSPUFF))
    { // Spawn the puff anyway
        puffpos = trace.HitPos;
        puff = P_SpawnPuff(t1, pufftype, puffpos, ...);
        if (nointeract)
        {
            return puff;
        }
    }
```

`LAF_NOINTERACT` **guarantees** puff spawning — the comment at `:4927-4928` says so outright: *"[MC]
LAF_NOINTERACT guarantees puff spawning and returns it directly to the calling function."* (And the
same shape repeats at `:4968` and `:5041`.) So a visible or interacting puff class would leave a
mark in the world on every haptic tick. `RSQDG_NullPuff` must be a one-state inert actor — and the
mod already has the exact template twice over, `RSQDG_SoundPoint` (`feel.zs:157-176`) and
`RSQDG_RumbleSpot` (`feel.zs:180-200`):

```zscript
class RSQDG_NullPuff : Actor
{
    Default
    {
        +NOINTERACTION +NOBLOCKMAP +NOGRAVITY +NOTELEPORT +DONTSPLASH +NOTONAUTOMAP
        RenderStyle "None";
        Radius 1; Height 1;
    }
    States { Spawn: TNT1 A 1; Stop; }
}
```

Cheaper than the `RSQDG_Buzz` it replaces (which carries `DamageType "RSQDGHaptic"` and a Speed, at
`feel.zs:140-151`).

**Keep `RSQDG_Buzz` as a class** so old saves load, for the reason `feel.zs:154-156` already gives
about keeping `RSQDG_SoundPoint`: *"a class missing from a save is a load failure."*

**Label: BUILDABLE. Cost: S.**

### 10.3 Which route for which, since §7 adds an event

`04_DISTANCEGRAB.md` §8's mapping stands and is the best-developed part of the mod. Two additions:

- **A swung object** (§7): `LineAttack`+`LAF_NOINTERACT` on the swinging hand; on the rumble route,
  scaled by `Heft` (`mass.zs:210-215`) so a swung barrel out-thumps a swung clip. `OnImpact`
  (`feel.zs:433`) stays empty for *throws* by the decision at `:431` — a thrown thing has left the
  hand — but a swung thing has not.
- **The Headlock** currently borrows the catch tick (`near.zs:329` → `feel.zs:405`). A 3-second hold
  on a struggling monster is the clearest case in the package for a continuous level: add it to
  `Level0` (`feel.zs:446`) beside the carry's strain, scaled by `Heft`, and it costs nothing new.

---

## 11. What `near.zs` needs documented — D31 onward

`near.zs` postdates D30 and has no record. Not a code change; it is the gap that made §2.6 and §8.2
possible.

| Proposed | Subject | Why it needs writing down |
|---|---|---|
| **D31** | **The walk-up grip** — the grip as a raw key scan through `InputProcess`, the press **eaten** when one of three acts takes it (`near.zs:61-86`), holsters first (`:179`) | The eat is the mechanism behind §2.2. A future session will read `eaten[h]` and not know it is deliberate. |
| **D32** | **Hand swap** — pose, mass/class gates, the 70-tic `offWait` queue (`near.zs:266-313`) | `Movable`'s exclusion list (`:235-244`) is a hardcoded set of other mods' prefixes; the reasoning must survive. |
| **D33** | **The Headlock is a human shield, and D6 does not cover it.** D6 (`docs/DECISIONS.md:100-116`) says *"The distance grab cannot hold a living monster at all"* and `policy.zs:360-362` enforces it **at range**. The near grip is a different act. | Without this, D6 reads as contradicted by shipped code. **Say explicitly that D6/D7 still hold for the distance grab and the near grip is a separate decision.** |
| **D34** | **`rsqdg_collar_mass` reads `Actor.Mass`, not `RSQDG_Mass.Kg`** (`near.zs:148`, documented in `CVARINFO.txt:48`) | `mass.zs:177` warns `Actor.Mass` is 100 on *every* non-monster, and the rest of the package decides weight through `Kg`. For monsters `Mass` is meaningful (imp 100, demon 400, baron 1000) so it works — but a mod that re-scales monster `Mass` (Champions gives a Bronze champion `0x7FFFFFFF`, the overflow `throw.zs:446-449` already had to bound) makes this gate arbitrary. **One comment saying Doom mass is deliberate here, or one line routing it through `Kg`.** |
| **D35** | **`RigBusy` asks `rig.region`, `HandBusy` asks `rig.busy`** (§1.1, §2.6) | Either reconcile them — which §3 does — or record why `near.zs` yields to a holster but not a cigarette. |

---

## 12. Label and cost table

### Part One

| Item | Label | Cost | Where |
|---|---|---|---|
| **Tether: do not reuse `rsqdg_press` for Lock; review `rsqgr_grab_dg` default** | **BUILDABLE** (prevents a bug) | **S** | §0.2 R1 |
| **Tether: publish `RSQ_HandsService` per hand** | **BUILDABLE** | **S** | §0.2 R2 |
| **Tether: consult it before the wind-up arms** | **BUILDABLE** | **S** | §0.2 R3 |
| Tether: `"climbing"` ignores its `hand` argument | **BUILDABLE** (bug fix) | S | §0.2 R4 |
| **Wheel publishes `RSQ_HandsService`, both hands** | **BUILDABLE** | **S** | §3.8, §4 |
| **DistanceGrab consumes it in `HandBusy` and `RigBusy`** | **BUILDABLE** | **M** | §4 |
| **DistanceGrab: reconcile `HandBusy` and `near.zs`'s `RigBusy`** | **BUILDABLE** (bug fix) | **S** | §2.6, §11 D35 |
| Publish the engine's two-hand latch as `engine.twohand` priority 95, from every publisher | **BUILDABLE** | S | §5 |
| `hand.posevalid` bitmask (roll clean, pitch/yaw hijacked under the latch) | **BUILDABLE** | S | §5 (3) |
| Raise `rsqdg_steady_m` 0.45 → 0.50 to match the engine's latch distance | **BUILDABLE** (partial mitigation only) | S | §2.3 |
| Read `WF_TWOHANDSTABILIZED` in `HandBusy` | **BUILDABLE** (bug fix) | S | §2.3 |
| DistanceGrab's `rsqdg_beam_yield` should also yield to the Wheel's beam | **BUILDABLE** | S | §2.1 |
| TacticalRig / LaserSight publish | **BUILDABLE** | S each | §4 |
| **Can two pk3s carry one `RSQ_HandsService` class?** | **unknown — GATE** | **S** | §3.7 |
| Prefix-list fallback if the shared class refuses | **HACK** | S | §3.7 (c) |
| Replace `rig.busy`'s hub role with the protocol | **do not** — keep `rig.busy` as it is | — | §3.1, §3.7 |
| Try to clear `weaponStabilised` | **do not** — `p_pspr.cpp:512-515` | — | §5 |
| Any edit to a Grapple file | **do not** — it is moving | — | scope |

### Part Two

| # | Item | Label | Cost | Where |
|---|---|---|---|---|
| 9a | **`bTHRUACTORS` defeats the projectile shield** — drop the set, keep the restore | **BUILDABLE** | **S** | §6.2 |
| 9b | **`+FORCEINFIGHTING` on a held monster** — same change, or 9a looks broken on NoInfighting maps | **BUILDABLE** | **S** | §6.4 |
| 9a′ | Headset check: a monster held at `Radius + 6` overlapping the player at point-blank | **EXISTS** (a test) | S | §6.3 |
| 7 | **Swung-object hit test** — swept chord, 140–210 traces/s worst case, 0 at rest | **BUILDABLE** | **M** | §7 |
| 7a | Drop `TRF_SOLIDACTORS` and `TRF_BLOCKSELF` from `01_MELEE`'s recipe for this caller | **BUILDABLE** | S | §7.3 (a)(b) |
| 7b | `lastObj[h]`: trace the object's travelled chord, not the palm's | **BUILDABLE** | S | §7.3 (e) |
| 7c | Filter `d.HitActor` through the existing `CanHurt` — the player is not ignored by the trace | **BUILDABLE** (bug prevention) | S | §7.3 (d) |
| 3a | **`freezetics = 2` instead of `tics = -1`**, drop the restore, add `WorldLoaded` | **BUILDABLE** | **S** | §8 |
| 9c | **`FindNear`: half-extent 128 → `nearU + 8`** (provably sufficient; 4–9× fewer cells) | **BUILDABLE** | **S** | §9.2 |
| 9c′ | Cache `ready[]` across 3 tics; hoist 7 loop-invariant lookups out of `FindNear` | **BUILDABLE** | S | §9.3 |
| 8a | **`LineAttack`+`LAF_NOINTERACT`** for the pulse route; retires D15's hazard | **BUILDABLE** | **S** | §10.1 |
| 8b | `RSQDG_NullPuff` — inert, because `LAF_NOINTERACT` **guarantees** the puff | **BUILDABLE** | S | §10.2 |
| 8c | Buzz a swung object; a continuous level for the Headlock | **BUILDABLE** | S | §10.3 |
| 11 | **D31–D35 for `near.zs`** | — | S | §11 |
| — | `+SKULLFLY` / `Slam` for the throw | **do not** — D23 and `throw.zs:8-14`'s four reasons | — | `04_DISTANCEGRAB.md` §4 |
| — | A living monster at *range* | **do not** — D6/D7 | — | §11 D33 |
| — | `SetOrigin` for the carry | **do not** — D22; the carry writes `Vel` and is better than the audit's advice | — | `04_DISTANCEGRAB.md` §2 |

---

## 13. Corrections this file makes to earlier reports

1. **`04_DISTANCEGRAB.md` §3: `near.zs` does have `WorldUnloaded`** (`near.zs:373-376`, calling
   `EndCollar` for every player). It is `WorldLoaded` that is missing, and that is the one the save
   path needs. §8.2.
2. **`04_DISTANCEGRAB.md` §9c: `nearU + 64` is more padding than needed.** `nearU` alone is
   provably sufficient, because an actor is linked into every blockmap cell its radius touches
   (`p_maputl.cpp:518-529`) and `FindNear` subtracts the radius (`near.zs:139`). The recommendation
   should be `nearU + 8`, a 4–9× cell reduction rather than ~1.5×. §9.2.
3. **`04_DISTANCEGRAB.md` §7 / `01_MELEE.md` §3.5: the swept-chord recipe should drop
   `TRF_SOLIDACTORS` and `TRF_BLOCKSELF`** for a held-object caller. `TRF_SOLIDACTORS` widens the
   mask past `MF_SHOOTABLE` to solid props that can shadow a monster (`p_map.cpp:5248-5249`), and
   `TRF_BLOCKSELF` affects **line** flags only and adds `ML_BLOCKMONSTERS` when the source is not a
   player (`:5252-5258`). §7.3.
4. **`04_DISTANCEGRAB.md` §7: the reason `bTHRUACTORS` and a script hit test coexist has a
   citation**, and it should be in the comment: `P_LineTrace`'s actor filter reads the **trace's**
   `TRF_THRUACTORS`, never the source actor's `bTHRUACTORS` (`p_map.cpp:5173`, filter at
   `:5145-5161`). Not luck. §7.2.
5. **`07_WHEEL.md` §9.4 named the architecture note; this file finds the mechanism and it is one
   engine fact.** `CF_TOTALLYFROZEN` reaches only `cmd` (`player.zs:1119-1126`), and DistanceGrab's
   three input routes — the palm pose, `InputProcess` raw key scans, and netevents — are none of
   them `cmd`. The Wheel's freeze cannot reach any of them. §2.0.
6. **`00_CAPABILITIES` gotcha 10a should carry `WHIP_HANDOFF.md` §1.5's finding, because it is
   general:** under the latch, the main hand's **pitch and yaw are overwritten** with the
   hand-to-hand line and **roll is passed through verbatim** (`gl_openvr.cpp:1742-1753`, the
   `weaponangles[ROLL]` argument at `:1752`). Any mod reading hand pitch is affected; any mod
   reading roll is not. §5.
7. **`00_CAPABILITIES` §1.2b / gotcha 10a: the latch distance and `rsqdg_steady_m` do not match,
   and the latch survives the hands separating.** 0.50 m latch
   (`gl_openvr.cpp:1716-1722`), cleared only on grip release with no distance test (`:1725-1727`),
   versus a 0.45 m geometric stand-down. That gap is a window, and only
   `WF_TWOHANDSTABILIZED` closes it. §2.3.
8. **`p_tick.cpp` line drift.** `04_DISTANCEGRAB.md` and `07_WHEEL.md` cite `p_tick.cpp:172` for
   the `WorldTick` call. In this tree it is **`:171`**, with `RunThinkers` at `:173` and
   `P_UpdateSpecials` gated on `isFrozen()` at `:176-179`. The conclusion is unchanged.

---

## 14. What I could not determine

- **Whether two pk3s may both carry a class named `RSQ_HandsService`.** §3.7 — this is the gate on
  the whole protocol and it is a one-afternoon load test, not a reasoning problem. If it refuses,
  the fallback is the prefix list, which is worse but correct.
- **Whether DistanceGrab's hand-swap pose is actually reachable with the ring open.** §2.7 — it
  depends on the live ring geometry (`wr_radius` 5.0, `wr_forward` 34, a projected fingertip) and on
  where a player's hands physically are. Needs a headset. The engine's 0.50 m latch is twice as
  wide and fires regardless, so §2.3 covers the pose either way.
- **Whether the shipping binary is `USE_OPENVR`, `USE_OPENXR` or neither.**
  `00_CAPABILITIES` §0's first open question. It decides the *severity* of §2.3 and §5 —
  `weaponStabilised` is a single global (`src/QzDoom/qzdoom_common.cpp:19`) whose only writes in
  this tree are `gl_openvr.cpp:1723` and `:1726`, so on a Quest/OpenXR build the latch is dormant
  **today**. It is one patch from firing, and "silently broken under SteamVR" is still broken.
  The five-minute check is the one §0 prescribes.
- **Whether `FindNear` with half-extent `nearU + 8` still finds a Spider Mastermind corpse**
  (radius 128, and `rsqdg_corpses` is on by default since 10-29). The blockmap-link rule says yes
  (§9.2) and the arithmetic is sound, but this is the one case where a wrong bound makes a grab
  silently stop working, so check it on the rig rather than taking it on the proof.
- **Whether any of §9's costs are felt.** Structurally true, unmeasured. `profilethinkers -t 20`
  and `stat think` are the gate, per `07_WHEEL.md` §8.1.
- **The protocol's priority numbers.** §3.2's band table is a proposal and a shared scale is a
  shared convention. It is the one piece that needs the owner's word before anything is written.
- **Nothing was built, compiled or run.** Every claim above is source-read in the two named trees.
