# The Whip — handoff

**For:** a co-designer with an engine session running.
**Subject:** reworking `RSQ_Grapple` into one tether engine, with the whip as its headline mode.
**Status:** designed and costed. **Nothing is built.** No file in any mod has been modified.
**Engine:** QuestZDoom 17.3, tag `qzd17.3` of `github.com/emawind84/gzdoom` (commit `9afc750`).
**Mod source read:** `github.com/presidentkoopa/RSQ_UBQSeries` HEAD `6a5c98c`, 2026-10-30.

Citations are `path:line`. Engine paths are at the tag; mod paths are inside `Grapple/`.
Where a claim came from a topic report rather than being re-verified by hand, it says so.

---

## 0. The short version

The whip already exists (`zscript/rsqgr/whip.zs`, ~10 KB, shipped, `rsqgr_whip` off by default).
It draws the right phenomenon and triggers on the wrong thing. Three faults, in order of how
much they cost the feel:

1. **It cracks when the loop is two thirds down the thong.** Provable by arithmetic, no testing.
2. **The crack is a fixed timer off a button press.** Your arm's speed contributes nothing but
   aim direction, so there is no technique and no skill expression.
3. **The post-crack droop is a constant with no physical meaning** — which is, ironically, the
   thing that made this project exist, because the owner saw it and thought "whip".

Fixing 1 and 2 is the same change. Fixing 3 means **keeping the droop and making it mean
something**, not deleting it.

Everything proposed here is ZScript and data. **No engine change is required.**

---

## 1. Five engine facts that constrain every decision

Read these before designing anything, because three of them are counter-intuitive and one of
them silently breaks mods today.

### 1.1 Both hands are fully tracked, and the head

`AttackPos` / `AttackAngle` / `AttackPitch` / `AttackRoll` and `OffhandPos` / `OffhandAngle` /
`OffhandPitch` / `OffhandRoll`, plus `HmdPosition` — `wadsrc/static/zscript/actors/actor.zs:268-277`.
Gate every read on `OverrideAttackPosDir` (`actor.zs:276`); when it is false you are not in VR
and both hands collapse onto the viewpoint (`hw_vrmodes.cpp:438-446`).

`RSQGR_Util` already does all of this correctly (`util.zs:51-79`). Do not re-plumb it.

### 1.2 There are no velocity fields

Nothing gives you hand speed. Every speed in this design is computed by storing the previous
tic's position and differencing. The mod already has a pose stream to hang this on
(`rsqgr_pose`); build on it rather than adding a second sampler.

### 1.3 You sample at 35 Hz, and the value is fresh

Hand poses are written by the **renderer**, per frame (`hw_vrmodes.cpp:442-445`,
`gl_openvr.cpp:2525-2537`), but script reads them once per tic. So values are current, and
motion between tics is gone. **Test the chord between the last two samples, never the point** —
a fast lash otherwise passes through a target without ever sampling inside it.

### 1.4 Units are not one constant

Horizontal: divide by `vr_vunits_per_meter` = 34 (`hw_vrmodes.cpp:94`).
**Vertical: divide by 34 AND multiply by `level.pixelstretch`** (default 1.2,
`g_mapinfo.cpp:314`, readable at `doombase.zs:479`), because the hand transform divides the up
axis by pixelstretch (`gl_openvr.cpp:1282-1292`).

Forget the vertical term and every height reads about 17% short. A whip is an overhead motion.
This matters here more than almost anywhere.

### 1.5 The two-hand brace — and the one channel it cannot touch

`vr_two_handed_weapons` defaults **true** (`hw_vrmodes.cpp:119`). Holding the off-hand **grip**
with the hands inside **0.50 m** sets `weaponStabilised` (`gl_openvr.cpp:1716-1727`), and the
offhand weapon then fires nothing (`player.zs:393`, `:426`), checks no buttons
(`p_pspr.cpp:755-757`), renders nothing (`hw_weapon.cpp:832-835`, `:917-920`), re-asserted on
every state set (`p_pspr.cpp:514`). It latches on the grip's **rising edge** and clears on
release **with no distance test** (`gl_openvr.cpp:1718-1726`).

**This has produced shipped bugs in four of the series' mods.** The Grapple is *not* one of
them — it has no `Weapon` subclass and takes its input from mod KEYCONF aliases, not the engine
grip (verified in the tether report). But it is exposed in one way that matters here:

**Stabilisation overwrites the main hand's pitch and yaw** with the hand-to-hand line
(`gl_openvr.cpp:1745-1753`), and `util.zs:74` / `:79` feed exactly those into every fire
direction. So in a two-handed pose, aim is hijacked.

**But roll is passed through verbatim** — `gl_openvr.cpp:1752-1753` recomputes pitch and yaw and
writes `weaponangles[ROLL]` unchanged. Verified by hand.

> **Design consequence, and it is the best single fact in this document:**
> **the wrist is the one clean input channel during a two-handed grip.** Derive the wind-up from
> hand **positions** and the flick from **roll**, and the whole design is immune to a condition
> that breaks four other mods. Do not derive anything from main-hand yaw or pitch.

Roll goes **stale, not zero**, on a collapsed frame, so a roll detector must still gate on the
position spike filter (from the tether report).

---

## 2. What is built today

| Piece | Where | State |
|---|---|---|
| Whip mode toggle | `rsqgr_whip`, CVARINFO.txt:51 | ships, **off by default** |
| Reach / damage / stun / snag | CVARINFO.txt:52-56 | flat cvar values |
| Trigger | `whip.zs` `NetworkProcess` | steals the claw core just fired, same netevent |
| Lash timing | `OUT_TICS = 4`, `BACK_TICS = 4` (`:26-27`) | fixed timer |
| Crack | `whip.zs:99` at `age == OUT_TICS` | fires early, see §3.1 |
| Direction | follows the wrist per tic (`:97-99`) | **correct, keep** |
| Snag → rope | `:168-196` → `core.Latch()` | **correct, keep** — the seam already exists |
| Drawn rope | 12 pieces, `NLINK = 12` (`:28`) | core's `RSQGR_Rope`, thinner |
| The loop | `:223` travelling wave | **correct phenomenon, wrong timing** |
| The bow | `:213`, applied at `:222` | constants, see §3.2 |
| Roll axis | `side` vector, `:210-212` | this is the real whip's "belly/spine" axis |
| Menu | MENUDEF.txt:75, :117-120 | five whip entries |
| Reset alias | KEYCONF.txt:25 | lists every whip cvar — **new cvars must be added here** |

The anatomy of a real whip maps onto this almost one-for-one: handle → `util.zs:63-69`,
thong → the 12 links, popper → the tip, belly/spine → the `side` vector, the loop → `:223`,
the taper → the width factor at `:216`. **The existing bow and wave are not decoration. They
are the correct two components, in the axis the real thing uses.**

---

## 3. The three faults

### 3.1 It cracks two tics early — arithmetic, not taste

The drawn wave is

```
whip.zs:223   pb += side * (len * 0.06 * u * sin(360.0 * (1.5 * u - phase)));
whip.zs:214   double phase = double(age) / OUT_TICS;
whip.zs:26    const OUT_TICS = 4;
whip.zs:99    if (wage[i] == OUT_TICS && !wdone[i]) Crack(...);
```

A point of constant phase sits where `1.5·u − phase` is constant, i.e. at `u = phase / 1.5`.
It reaches the popper (`u = 1`) when `phase = 1.5`.

The crack fires at `age == 4`, where `phase = 4/4 = 1.0`, so **`u = 0.667`**.

**The whip cracks when the loop is two thirds of the way down the thong**, in mid-air, every
time. To crack at the tip you need `phase = 1.5`, which is `age == 6`. Verified by hand against
all four lines.

### 3.2 The droop is constant, and applied downward

```
whip.zs:213   double bow = (age <= OUT_TICS) ? len * 0.25 * (1.0 - double(age)/OUT_TICS) : len * 0.08;
whip.zs:222   pb.z += bow * sin(180.0 * u) * (age <= OUT_TICS ? 1.0 : -1.0);
```

Before the crack the bow decays correctly to zero — so the lash **does** straighten as it
should. After the crack it jumps back to a **constant** 8% of length that never decays, and the
sign flips to **downward**, for the four tics of `BACK_TICS`. That is the sag.

**Do not delete it.** It is the only thing in the file that reads as weight, and it is what the
owner saw. Make it mean something (§4.4).

### 3.3 The gesture is a button

`NetworkProcess` fires on the bind; `Start()` sets a cvar-fixed reach; `Crack()` happens four
tics later. Arm speed contributes nothing. There is no wind-up, no technique, no failure mode,
and nothing to get good at.

---

## 4. The design

### 4.1 Four phases, from the real technique

A real crack is: let the whip **trail behind you**, **lift** the arm smoothly to about 90°,
**snap the forearm down** before the tip lands, then **flick the wrist** — which sharpens the
loop and sends it racing down the thong to crack at the tip.

| Phase | What it is | Signal |
|---|---|---|
| **LAID** | a *state*, not a motion — the whip is extended behind | hand low, behind `pmo.Angle`, still for ~6 tics |
| **LIFT** | smooth rise to ~90° | rising position, **second-difference jerk test**; more than one reversal zeroes the load |
| **SNAP** | forearm reversal — the energy input | sign flip, inside a timing window off the lift's peak |
| **FLICK** | the wrist sharpening the loop | **roll rate** — the channel §1.5 protects |

The lift **multiplies** the snap; the snap is the dominant term; the flick launches the loop.
One scalar comes out, and it drives wave speed, reach and damage together.

**LAID is a gameplay beat, not bookkeeping.** It stops crack-spam without a cooldown timer and
gives the weapon a rhythm. Draw it by putting the tip on the floor behind the player.

**The snap must be allowed to fail.** Snap too late and there is no clean loop — which the
player sees directly, because a failed crack simply does not produce one. That is the reference's
own feedback channel and it costs one comparison.

### 4.2 Crack on loop arrival

Replace `phase = age / OUT_TICS` with an accumulator:

```
phase += waveSpeed / thongLength     // per tic
crack when phase >= 1.5
```

`waveSpeed` comes from the energy of §4.1. Now arm speed sets **both** when the whip cracks and
**where the tip is** when it does — from one physical quantity, using the wave the mod already
draws. `OUT_TICS` and `BACK_TICS` cease to exist.

The tether report anchored the speed range against today's implied 48 units/tic, giving roughly
24–96 units/tic as the usable band. That is calibration, not invention — re-derive it if you
change the thong length.

### 4.3 The taper is why it cracks

Thong → fall → popper is a mass taper: the same energy driven into progressively less mass takes
the tip supersonic. Two consequences:

- **Taper the drawn width** across the 12 links. The knob already exists — `rsqgr_rope_width`
  with a 0.7 factor at `:216`.
- **Reach and damage come from the amplification**, not from flat cvars. A small input at the
  handle becomes a large effect at the tip. `rsqgr_whip_reach` and `rsqgr_whip_damage` become
  **ceilings**, not values — the same convention the tether rework uses throughout.

### 4.4 The slack is the charge gauge

At rest, trailing behind, the rope hangs with real droop. The lift and snap pull it taut. The
loop then races down a taut thong.

The player reads their charge **in the rope**. No meter, no HUD, no number. This is §3.2's
constant replaced by a mapping from the loaded state — same visual language, real physics.

### 4.5 Three aim modes, selectable in the menu

A `server int rsqgr_whip_aim` in CVARINFO.txt, an Option in MENUDEF.txt beside the existing whip
entries, **and an entry in the `rsqgr_reset` alias in KEYCONF.txt** — that alias lists every whip
cvar today, so anything new that is missing from it will survive a reset and confuse everyone.

| Mode | Behaviour | Feel |
|---|---|---|
| **0 — Free** | trace the chord, hit what it crosses | purest, hardest; what the mod does now |
| **1 — Snap to arc** *(default)* | on release, search a cone about the release direction and bend the tip to the best target | Indiana Jones: he doesn't aim, but he doesn't miss |
| **2 — Lock** | point, confirm, release goes there | most reliable, most likely to feel like a gun |

From the tether report: cone 28° (reasoned from a 35 Hz wrist turn), max bend 18° **scaled by
energy** so assist cannot mask bad technique, and "best" is **most aligned, tie-broken by
nearest — explicitly not lowest health**, which would make the whip jump to a dying enemy while
you swing at the healthy one in front of you. The target picker reuses `Throwable`
(`monster.zs:199-208`) and the existing trace predicate (`hook.zs:396-407`) rather than growing
a second targeting rule that drifts.

Lock needs a **fifth bind** — reusing the grab bind would let DistanceGrab's grips silently lock
(`input.zs:149-150`). Its cue already exists as `RSQGR_A11y.Ready` (`a11y.zs:92-98`).

### 4.6 Disarm is already built and currently written up as a limitation

Reference step 6 is popping a gun out of a hand. `HANDOFF.md` correctly records that a Doom
monster's attack is a **state**, not an item, so nothing can be knocked loose — **but the whip's
forced Pain breaks off the wind-up**, which is the functional disarm. Present it as the feature
it is.

---

## 5. One tether, four presets

The whip is not a separate mod and should not become one. It is the hook with a short range and
a short life — `whip.zs:168-196` already hands a snag back to `core.Latch()`, so the swing, reel
and release are core's code, unchanged. Two mods would mean two rope renderers fighting over the
same binds.

The axes, from the tether report: **anchor type** (geometry / actor / none), **length policy**
(reel / free / hand-driven), **tension** (rigid / slack / elastic), **hand count**, and a fifth
the thesis missed — **who the constraint moves** (player / anchor / split).

| Preset | Anchor | Who moves | Built | Missing |
|---|---|---|---|---|
| **Indiana Jones** | geometry | player | **~90%** | §4 above |
| **Bulletstorm** (Leash) | actor | the *target* | **~80%** | hang time at the arc apex, and telling Score a juggle started |
| **Bionic Commando** | geometry | player | **~70%** | no-jump, tuned arcs, **and the momentum fix** |
| **Thief** (rope arrows) | geometry, persistent | player, climbing | **~60%** | material gate, planted-rope list |

Notes the engine session will care about:

- **Bulletstorm needs neither `freezetics` nor a SlowMo pulse.** `RSQGR_Flight.WorldTick`
  already writes `a.Vel = (0,0,0)` and re-pins `a.tics = -1` every tic (`monster.zs:419-421`)
  and places the actor itself — it is fully puppeted, so `freezetics` would suppress a Tick whose
  effects are already overridden *and* stop the placement. (Tether report.)
- **Thief's material gate** is real but the field is `TraceResults.HitTexture`
  (`doombase.zs:282`), **not** `FLineTraceData.HitTexture` — this mod traces with its own
  `LineTracer`, not `LineTrace`. Test with `TexMan.GetName` just before `core.zs:286`'s `Latch`.
  Hand-over-hand is **entirely built** (`climb.zs` + `core.zs:551-605`); only both-hands-at-once
  is missing. Save/load is free via the rebuild-from-`ThinkerIterator` at `core.zs:656-681`.
- **Bionic Commando's no-jump** is one assignment: `JumpZ` is a plain ZScript field on
  `PlayerPawn` (`player.zs:35`, property `:68`, default 8 at `:120`), so `pmo.JumpZ = 0` disables
  jumping per player with no dmflags and no engine change. `level.IsJumpingAllowed()` is native
  read-only and is *not* the route.
- **`core.zs:338-339`'s 52-tic no-progress guard drops a held monster**, and fights both the
  Bulletstorm preset and the lasso by definition. Same fix for both.

---

## 6. The blocker that outranks all of it

**`player.keepmomentum` does not appear anywhere in RSQ_UBQSeries** — zero hits across all 152
ZScript files. Verified by hand.

So `player.zs:1373-1378` multiplies `vel.XY` by 0.0001 **every grounded tic**. A measured
445-unit swing lands and stops dead. Three further consequences from the tether report:

- `bNOFRICTION` does **not** rescue it — the gate reads `GetFriction()` (`p_map.cpp:675-802`),
  which never consults `MF8_NOFRICTION`.
- It silently cancels core's own velocity reconstruction at `core.zs:423-431` on grounded tics.
- `rsqgr_max_speed 30` is folklore from `MAXMOVE`: that clamp (`p_mobj.cpp:2096-2097`) is
  conditional and **off by default**; the real ceiling is 5000 units/tic. A physical duck below
  `crouchfactor` 0.75 re-enables it mid-swing.

**Set `player.keepmomentum = true` every tic while swinging.** It is cost S and it gates the
feel of everything above.

---

## 7. Build order

1. **`keepmomentum`** (§6). Cost S. Nothing else is worth tuning until swings carry.
2. **Crack on loop arrival** (§4.2). Cost S. Deletes `OUT_TICS`/`BACK_TICS`, fixes §3.1.
3. **The slack gauge** (§4.4). Cost S. Replaces §3.2's constants; this is the visible payoff.
4. **The four phases** (§4.1). Cost M. The skill layer. Needs the per-frame hand ring.
5. **Swept chord** for the lash. Cost M. ~6 traces/tic per lashing hand, 12 for both — only
   while lashing. `RSQGR_Tracer` is already a `LineTracer` with arbitrary start and direction, so
   no new primitive is needed.
6. **The aim modes** (§4.5). Cost M, plus the twelve new cvars and the KEYCONF alias.
7. **Presets** (§5), cheapest first: Indiana Jones, Bulletstorm, Bionic Commando, Thief.

One prerequisite worth doing early: `PoseGood` (`util.zs:51-60`) does not reject the engine's
hand-collapse (`hw_vrmodes.cpp:438-446`), which is `HANDOFF.md`'s known "a press after a menu
closes fires from the body". Two lines, and §4.1's speed signal depends on it.

---

## 8. What the rig can settle, and what it cannot

The owner has a Linux QuestZDoom 17.3 build with the headset and both controllers posed from
cvars and left-eye frame dumps (`RSQ_WeaponDepot/tools/rig`).

- **Test A — does the gauge read?** Five frames: LAID, LIFT at half load, SNAP, a full-energy
  crack, a failed one. Pass = a stranger sorts them by loadedness on the first try **and** picks
  the failed crack out. Half a day.
- **Test B — does loop-arrival work?** Log `u_loop` at the crack tic; it must read 1.0, not
  0.667. Falsifies §4.2 cheaply.
- **Test C** — regression against `HANDOFF.md`'s measured baselines, plus a five-minute no-jump
  check.

**What the rig cannot tell you:** whether the snap window is *fun*. A cvar-posed hand proves
arithmetic and catches regressions. It cannot feel a whip. Budget a headset session for §4.1's
timing window and nothing else.

---

## 9. Decisions that belong to the owner

1. **One-handed or two?** Real whip technique is one-handed and the reference is explicit about
   it. He wants a two-handed wind-up anyway — Castlevania, not Indiana Jones. Both can exist;
   which is the default is his call.
2. **Does the mod get renamed** once it is the tether engine rather than a grappling hook?
3. **Bionic Commando's fixed 45° firing angle** — authentic, and overriding where a VR player
   points is an agency risk. Recommended **off** by default, offered as an option.
4. **`rsqgr_hop`** at 0 is authentic for Bionic Commando and carries a real stuck-on-a-step risk;
   2–3 is the safe fallback. State it in the preset, do not bury it.

---

## 10. Where everything lives

| | |
|---|---|
| This file | `apply/WHIP_HANDOFF.md` |
| Full whip design | `apply/08_WHIP_DESIGN.md` — plain language first, then implementation |
| The tether rework | `apply/01_GRAPPLE_TETHER.md` — sag diagnosis, mode matrix, migration |
| Engine capability map | `surveys/00_CAPABILITIES.md` — **§0 is a go/no-go, §7 is the gotchas** |
| Ropes topic report | `surveys/05_GRAPPLE_ROPES.md` |
| Fast-swing tracing | `surveys/01_MELEE.md` §2 and its hit-testing section |
| Cross-series findings | `apply/00_INDEX.md` |

Two corrections already filed against the survey by this work, so do not trust old copies:
`05_GRAPPLE_ROPES.md` §2.3 claims the 30 units/tic clamp applies unconditionally (it does not),
and its boilerplate suggests `bNOFRICTION` escapes the momentum wipe (it does not).

---

## 11. Not verified

No build, no compile, no headset, no rig run. Everything here is source-read and arithmetic.
The §3.1 crack-timing fault is proven from four lines and needs no test. The §4 design is
reasoned from the engine's capabilities and the mod's existing structure; its **feel** is
unproven and Test A exists precisely because of that.
