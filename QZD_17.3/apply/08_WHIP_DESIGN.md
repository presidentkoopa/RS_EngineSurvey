# 08 APPLY — THE WHIP, AND THE FOUR PRESETS

**Mod:** `Grapple/` in `github.com/presidentkoopa/RSQ_UBQSeries`, HEAD `6a5c98c`, 2026-10-30.
**Engine:** `qzd173` (tag `qzd17.3`, emawind84/gzdoom). All `file:line` resolve in those two trees.
**Companion:** `apply/01_GRAPPLE_TETHER.md` — the tether engine and its five-axis matrix.
**This document supersedes `01` §1's "delete the sag" recommendation.** The diagnosis was right; the
prescription was wrong. It also supersedes the circle-to-build-momentum wind-up of this file's own
first draft — real whip technique is not a circle, and the mod already draws the right phenomenon.
**Scope:** ZScript and data only. No engine change anywhere. Nothing is written into the mod's tree.

> **The owner, verbatim:** *"that 'sag' is what made me think it could be a whip in the first place…
> i saw what could be a whip and i had visions of two handing a whip to build momentum, somehow
> targeting an enemy and letting loose… indiana jones X castlevania"* — plus all three targeting
> options selectable from the menu, a whip-anatomy reference, and four reference games:
> **Bulletstorm, Indiana Jones, Thief: The Dark Project, Bionic Commando (NES)**.

---

# PART ONE — WHAT THE PLAYER DOES

*(No code in this half. Part Two is the implementation.)*

## 1. The four games are four different answers, not four skins

The grapple already contains one tether engine. Each of the four games sets its dials differently, and
once you see them side by side it is obvious they are the same machine:

- **Bulletstorm — the Leash.** You whip out an enemy, he comes off his feet, goes floaty and **hangs
  there** while you shoot him. The rope moves *him*, not you. It is not traversal at all; it is a
  setup move.
- **Indiana Jones.** You crack the whip at a beam above you and **you** swing. The rope moves you.
  And you can pop a weapon out of a hand — more on that below, because it already works.
- **Thief — rope arrows.** You plant a rope high up and it **stays there**, and you climb it hand
  over hand. The rule that made it a puzzle instead of a cheat: **it only bites wood.** Stone refuses.
- **Bionic Commando.** **You cannot jump.** The arm is your entire movement system. Every bit of
  traversal is an arc you commit to, and momentum is the whole game.

Those become four named presets in the menu, over one engine. Nothing is duplicated.

**Which are cheap, honestly:** Indiana Jones and Bulletstorm are mostly already built. Thief needs
two new things (the wood rule, and ropes that stay put). Bionic Commando is really a change to how
*moving* works, and it lives or dies on one missing line that is also the single biggest win available
anywhere in this mod — see §4.

## 2. The thing you saw in the rope was the right thing

The droop is the only part of the current whip that has any **weight**. Everything else is a
stopwatch: press, and exactly four tics later — 114 ms, identical every time — the lash reaches
exactly as far as a slider says and does exactly as much damage as another slider says. How you moved
your arm changes nothing but the direction.

So: **keep the droop, delete the stopwatch.**

## 3. How a whip actually cracks — and the mod already draws it

A whip has named parts, handle outward: **handle** (with the butt knot) → **thong**, the braided
length → **fall**, a replaceable lace → **cracker / popper**, the little string at the very tip that
makes the noise. The curl has an axis too: the inside of the curl is the **belly**, the outside the
**spine**, and the whip wants to roll along that axis.

And the crack is one specific thing: **a loop travels down the thong, and it cracks when the loop
reaches the popper.** Because the whip tapers — thick thong, thinner fall, tiny popper — the same
energy driven into less and less mass makes the tip go faster and faster, until the very end goes
supersonic. That is the bang. The taper is why it cracks at all.

**Here is the good news.** The current whip already draws a loop travelling down the rope — that is
what the wave in the drawing code is. It draws the right phenomenon, in the right plane (the side
vector it uses *is* the belly–spine axis, so the existing bow and wave are not arbitrary decoration,
they are the correct two components) — **and then it cracks on a timer that has nothing to do with
where the loop has got to.** As the code stands, the crack fires when the loop is only about
two-thirds of the way down. That is a real mismatch and the reference predicts exactly how it would
look wrong.

**The fix is to crack when the loop arrives.** How long that takes depends on how hard you snapped.
So one physical quantity — your snap — sets the crack's timing *and* its power *and* where the tip
is, all at once, using a wave that is already on screen.

## 4. The four beats of a real crack

1. **Lay it out.** The whip trails behind you, thong on the ground. You cannot crack a whip that is
   not laid out. This is a **state**, not a motion, and it is the best thing in the whole reference
   for us: it stops crack-spam dead and gives the weapon a rhythm — crack, recover, lay out, crack.
   After a crack the whip is out in front, so it has to come back before it can go again. That is the
   weapon's tempo and it costs nothing to implement.
2. **Lift.** Raise your arm quickly *but smoothly*, to about ninety degrees. The tip swings up and
   over. A jerky lift does not load it — and the game should agree with that.
3. **Snap down.** Before the tip lands behind you, snap your forearm down hard. **This is the energy,
   and the biggest single contributor to how hard the crack lands.** Snap too late — after the tip
   would already have hit the ground — and the crack fails. That timing window is a real skill, and
   it should be one in the game too.
4. **Flick your wrist** at the end of the motion. This sharpens the loop and launches it down the
   thong. It is separate from the snap, and it is what starts the loop travelling.

And the fifth step from the reference is the player's own feedback channel, free of charge: *"if you
see a clean loop racing down the whip, you're doing it right."* A botched crack — no snap, or a late
one — should visibly fail to form a clean loop. No tutorial text, no error message. You watch the
rope and you know.

## 5. The rope is the gauge. There is no meter.

A whip trailing behind you hangs with real, heavy droop. The lift and the snap **pull it taut**. Then
the loop races down a taut thong and cracks.

So the slack still tells you everything — it just reads something better than a charge number now. It
reads **what the whip is actually doing**: laid out and heavy, loading and tightening, or taut with a
loop running down it. Same visual language, real physics underneath, and still no HUD.

## 6. One hand or two

Real whip technique is one-handed, and the reference is explicit about it. The four beats above are a
one-handed crack.

You asked for two-handed momentum, and it is worth having — that is the Castlevania half, and
Castlevania was never trying to be accurate. So both ship: the authentic one-handed crack, and a
two-handed wind-up that feeds the same engine.

**My recommendation is that the one-handed crack is the default**, for one reason that is not about
authenticity: the four beats *are* the skill, and they are the thing that will make this weapon feel
like yours. The two-handed wind is a single accumulating number — easier, and shallower. Ship it as
the option for when you want to feel like Simon Belmont rather than Indiana Jones. **It is your
call; that is the reason behind my pick.**

## 7. Three ways to aim, and you pick

Winding up a whip *and* threading a needle with it is a lot to ask of one gesture, so aim is its own
setting:

- **Free.** No help. The lash goes where it goes. Honest, and bluntly **the hardest to enjoy** — the
  game only looks at your hand 35 times a second, so a fast lash is three or four snapshots and you
  will miss things you were sure you hit. *(The loop-arrival change does help here: the tip's position
  is now a real physical answer rather than a timer's guess, so mode 0 is better than it was.)*
- **Snap within arc** — **recommended default.** Aim roughly; on release the tip bends toward the best
  enemy in a cone around your throw. Aim badly, still miss. This is how every game that shipped a good
  whip does it, and it is what makes a whip feel like a whip rather than a physics demo.
- **Explicit lock.** Point, press to lock, wind up, release, it goes there. Reliable, readable, and
  **the most likely of the three to stop feeling like a whip and start feeling like a gun.** Worth
  shipping — you asked for it, and it is the only mode that works for someone who cannot aim
  precisely — but I would not default to it.

## 8. The disarm already works

Reference step six is popping a handgun out of a bad guy's hand. In Doom there is nothing to pop
loose: a monster's attack is a **state it is in**, not an item it is holding. So nothing can be
knocked out of anything.

But the whip's crack already forces the monster into its **Pain** state whatever its pain chance, and
**that breaks off an attack it was winding up.** Functionally that is the disarm: you crack the thing
that was about to shoot you, and it does not shoot you. It is the fantasy delivered by the only means
the engine allows, and it is already built and already measured. It belongs in the Indiana Jones
preset as a feature, not an apology.

---

# PART TWO — IMPLEMENTATION

## 0. The reframe, against the code

`01_GRAPPLE_TETHER.md` §1 located the sag correctly:

```
whip.zs:213   double bow = (age <= OUT_TICS) ? len * 0.25 * (1.0 - double(age)/OUT_TICS) : len * 0.08;
whip.zs:222   pb.z += bow * sin(180.0 * u) * (age <= OUT_TICS ? 1.0 : -1.0);
whip.zs:223   pb += side * (len * 0.06 * u * sin(360.0 * (1.5 * u - phase)));
whip.zs:26-27 const OUT_TICS = 4;  const BACK_TICS = 4;
```

Its prescription — decay the bow, freeze the wave — **is withdrawn.** What is wrong is not that those
lines draw slack and a travelling wave. It is that **every number in them is a constant, and the crack
is fired by a tic counter instead of by the wave.**

---

## 1. THE ANATOMY, MAPPED TO THE CODE

The reference's part names land on existing variables almost one for one. This is why the rework is an
M and not an L.

| Real whip | In the mod | Citation |
|---|---|---|
| Handle, butt / Turk's head | the hand position | `util.zs:63-69` `HandPos` |
| **Thong** (the braided length) | the drawn rope, 12 pieces | `whip.zs:28` `NLINK = 12`, placed `:218-242` |
| **Fall** (replaceable lace) | the last two or three pieces | — (a taper boundary, §4) |
| **Cracker / popper** | the tip | `whip.zs:35` `wtip[]`, `:207` |
| **Belly / spine** — "whip wants to roll along this axis" | `side`, the perpendicular the wave is applied along | `whip.zs:210-212` `side = (-wdir.y, wdir.x, 0).Unit()` |
| **The loop racing down the thong** | the travelling wave | `whip.zs:223` |
| **The taper** (thong → fall → popper) | one flat width, ripe for tapering | `whip.zs:216` `rsqgr_rope_width * 0.7` |

**The belly/spine point is worth stating out loud** because it retires a suspicion: `whip.zs:210-212`
and `:222-223` are not two arbitrary wiggles. They are the bow in the *vertical* plane and the loop in
the *roll* plane — the two components a real whip has, in the axis the reference names. Keep both, and
keep them separate.

---

## 2. LOOP-ARRIVAL CRACKING — the core change

### 2.1 The loop's position is already exactly computable

`whip.zs:223` is a travelling wave with spatial frequency 1.5 cycles over the thong:

```
pb += side * (len * 0.06 * u * sin(360.0 * (1.5 * u - phase)))
```

A point of constant phase satisfies `1.5·u − phase = const`, so **the loop sits at**

```
u_loop = phase / 1.5
```

and it reaches the popper (`u = 1`) exactly when **`phase = 1.5`**.

### 2.2 Which means today's crack is provably early, and by how much

`whip.zs:214` sets `phase = double(age) / OUT_TICS`, with `OUT_TICS = 4` (`:26`). So `phase` reaches
1.5 at `age = 6`. But the crack fires at `age == OUT_TICS`, i.e. `age = 4` (`whip.zs:99`), where
`phase = 1.0` and therefore

```
u_loop = 1.0 / 1.5 = 0.667
```

**The crack fires when the drawn loop is two thirds of the way down the thong.** It is not a feel
complaint; it is arithmetic over two lines of the shipped file, and it is exactly the mismatch the
owner's reference predicts would look wrong. (`phase` then keeps climbing past 1.5 into the retract —
`:214` never stops — so the loop reaches the popper two tics *after* the bang, with nothing drawn to
mark it.)

### 2.3 The replacement

Drive `phase` by a wave speed, and fire the crack when the loop arrives:

```zscript
// per tic while a lash is live
double thong   = wreach[i];                            // current thong length, map units
double dphase  = waveSpeed / thong;                    // cycles per tic
phase[i]      += dphase;
if (!wdone[i] && phase[i] >= LOOP_CYCLES)              // LOOP_CYCLES = 1.5, from :223's 1.5*u
    Crack(i, pmo, hand, org);                          // the loop has reached the popper
```

- **Loop travel time** `= LOOP_CYCLES · thong / waveSpeed` tics. A longer thong takes longer: correct,
  and free.
- **`waveSpeed` comes from the snap energy** (§3.5), so arm speed sets the crack's timing and its
  position from one quantity.
- **Calibration against today's behaviour, so the range is not invented.** Today's implied rate is
  `dphase = 1/OUT_TICS = 0.25`/tic at `thong = 192`, i.e. `waveSpeed = 48 units/tic`. Anchor the new
  range on that: a hard snap ≈ `96 u/tic` (loop arrives in 3 tics), a weak one ≈ `24 u/tic`
  (12 tics, and at the low end it should simply **fail** — §3.6). `whip.zs:102`'s hard halt at
  `OUT_TICS + BACK_TICS` becomes a timeout at `rsqgr_whip_life` tics, defaulting to ~16, which is also
  the failed-crack exit.
- **`OUT_TICS` and `BACK_TICS` stop existing as consts.** This is the stopwatch dying, and it is the
  change the owner will feel most.

**Label: BUILDABLE / M.** The expression work is S; the energy model feeding it (§3) is the M.

---

## 3. THE FOUR PHASES

All four read the per-frame hand stream. **Build on what exists** — `input.zs:91-106` already streams a
hand pose per rendered frame as a network event, `input.zs:141-146` receives it, `climb.zs:48-56` reads
it with a 3-tic freshness window, `climb.zs:68-78` packs it as 3×10 bits of half-units. It is the only
multiplayer-correct hand read in the mod (`01` §9.3). Three changes, all already scoped in `01` §8.1:
widen the send gate at `input.zs:101` (today `grabHeld` only) to include "a whip is armed"; keep an
8-entry ring instead of one sample; and **keep the height trick at `input.zs:103` verbatim** —
`(h.z - e.ViewPos.z) + p.viewheight`, the hand's Z against *that frame's own eye*, whose comment at
`input.zs:79-90` records the measured 6-14 unit per-stroke wobble from each wrong alternative. Every
phase below differences Z; a wobbling Z would fabricate lifts and snaps.

### 3.0 Units, once

`vr_vunits_per_meter` = 34.0 (`qzd173/src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:94`),
35 tics/s, so **1 unit/tic ≈ 1.03 m/s horizontally**. Vertically the hand transform divides the up axis
by `pixelstretch` (`qzd173/src/rendering/gl/stereo3d/gl_openvr.cpp:1282-1292`), default 1.2
(`qzd173/src/gamedata/g_mapinfo.cpp:314`), so raw Z reads ~17 % short. **Every phase here is vertical**,
so this is not optional. The mod already carries the term (`util.zs:43`, `:104`) and `HANDOFF.md`
records the direction verified equal to the engine's `AttackDir` at five poses — **already right**. One
change: `util.zs:21`'s `const PIXELSTRETCH = 1.2` should read `level.pixelstretch`
(`qzd173/wadsrc/static/zscript/doombase.zs:479`; MAPINFO can set it,
`qzd173/src/gamedata/g_mapinfo.cpp:1566`), falling back to 1.2. Normalise once at the sampler.

A useful reference: a 0.5 m arm rise ≈ **17 units** post-normalisation.

### 3.1 TRAIL / LAID OUT — a state, and the weapon's rhythm

**Representation.** A per-hand enum, not a flag: `WS_IDLE → WS_LAID → WS_LIFT → WS_SNAP → WS_FLY →
WS_RECOVER`. Only `WS_LAID` may begin a crack.

**Detection.** Three conditions held for `rsqgr_whip_lay_tics` consecutive tics (~6):

```zscript
Vector3 hoff = handRing[hand][0];                        // offset from the pawn, normalised Z
bool low    = hoff.z < pmo.Height * LAY_HEIGHT;          // LAY_HEIGHT ~ 0.45: hand down by the hip
bool behind = (hoff.xy dot Actor.AngleToVector(pmo.Angle, 1)) < 0;   // behind the body's facing
bool still  = (handRing[hand][0] - handRing[hand][1]).Length() < LAY_STILL;   // ~1.5 u/tic
```

Note `pmo.Angle` follows the **head**, not the hand — `VR_GetMove` returns
`*yaw = hmdorientation[YAW] + snapTurn` (`qzd173/src/QzDoom/qzdoom_common.cpp:119-121`) — which is the
right reference here: "behind you" means behind where you are looking.

**Showing it.** Draw the thong lying out behind, along the floor. `Draw` (`whip.zs:200-243`) already
takes an arbitrary tip; place it at `hand − facing · thong`, with Z at `pmo.floorz` (and the existing
`sag` term doing the hanging). One extra `RSQGR_Tracer` call per lay-out — not per tic — to find the
floor behind, reusing `hook.zs:390-411`. The chain look is already measured at mean 0.016 ms / max
0.061 ms per rope (`HANDOFF.md`), so a second drawn rope is affordable.

**Why this is the best thing in the reference for us.** It replaces the crude `~4-6 tic cooldown` that
this file's first draft and `05_GRAPPLE` §4.3 both proposed, with something **physical and visible**.
The player cannot spam because the whip is in front of them and has to come back. `WS_RECOVER` →
`WS_LAID` is the tempo, and it is self-explaining.

**Label: BUILDABLE / M.** The state machine is the cost, not the tests.

### 3.2 LIFT — and "smoothly" is a real signal

Rising, and *smooth*. Smoothness is cheap to measure and worth measuring, because the reference
singles it out:

```zscript
// over the lift window (up to 8 ring entries)
double rise = 0; int reversals = 0; double jerkMax = 0;
for (k = 1; k < n; k++) {
    double dz  = handRing[hand][k-1].z - handRing[hand][k].z;
    double dz2 = handRing[hand][k].z   - handRing[hand][k+1].z;
    rise += dz;
    if (dz * dz2 < 0) reversals++;                 // direction changed mid-lift
    jerkMax = max(jerkMax, abs(dz - dz2));         // second difference: the jerk
}
double smooth = clamp(1.0 - jerkMax / JERK_REF, 0.0, 1.0) * (reversals > 1 ? 0.0 : 1.0);
double loadLift = clamp(rise / LIFT_REF, 0.0, 1.0) * smooth;    // LIFT_REF ~ 17 u (0.5 m)
```

- **Second difference, not variance.** One number, no accumulator, and it is exactly "jerky": a smooth
  lift has near-constant `dz`, so `jerkMax → 0`.
- **More than one reversal zeroes the load outright.** A player shaking their arm gets nothing, which
  is the anti-waggle defence the first draft bought with a hold-to-arm button. Keep the button too
  (§3.7) — but the smoothness test is what makes it honest.
- `loadLift` is a *multiplier* on the snap, not an additive term: a snap without a lift behind it has
  nothing to drive into the loop, which is what the reference says.

**Label: BUILDABLE / S.**

### 3.3 SNAP DOWN — the energy, and the timing window that is the skill

The forearm reversal. Detection is the sign flip with a speed floor:

```zscript
double dzNow  = handRing[hand][0].z - handRing[hand][1].z;
double dzPrev = handRing[hand][1].z - handRing[hand][2].z;
bool snapped  = dzPrev > 0 && dzNow < 0 && (-dzNow) > RSQGR_A11y.Low(SNAP_MIN_SPEED);
double snapSpeed = -dzNow;                      // units/tic down; x1.03 for m/s
```

`SNAP_MIN_SPEED` ≈ **6 u/tic ≈ 6.2 m/s**, routed through `RSQGR_A11y.Low()` (`a11y.zs:48`), which
scales a speed threshold by ×1..×0.5 under `rsqa_gesture_ease` (`a11y.zs:46`) and reads as no-op when
`RSQ_Accessibility` is absent (the class-name guard at `a11y.zs:27`).

**The timing window — "before the tip would land".** The lift threw the tip up and over; it has its own
flight, and the snap must catch it before it comes down. Model the tip's flight from the lift rather
than inventing a constant:

```zscript
int peakTic = <tic the lift's dz last changed sign>;
int wantMin = peakTic + SNAP_EARLY;                      // ~1 tic: too early = nothing to drive
int wantMax = peakTic + int(TIP_FLIGHT * sqrt(thong / 192.0));   // longer thong, longer flight
double window = (snapTic < wantMin || snapTic > wantMax) ? 0.0
              : 1.0 - abs(snapTic - (wantMin+wantMax)/2) / double((wantMax-wantMin)/2 + 1);
```

`TIP_FLIGHT` ≈ 7 tics at a 192-unit thong; the `sqrt(thong)` term is the right shape because a
pendulum's period goes as `sqrt(L)`. `window` peaks at 1.0 mid-window and falls to 0 at the edges.

**This is the skill expression, and it must be allowed to fail.** `window = 0` → no clean loop, no
crack (§3.6). A player who snaps late gets a visible flop, which is step 5 of the reference doing the
teaching.

**Label: BUILDABLE / M.** The window is where the tuning time goes.

### 3.4 WRIST FLICK — and `AttackRoll` is the one field the two-hand trap cannot touch

Separate from the snap: the wrist sharpening that launches the loop. It needs a **wrist** signal
distinct from the hand's linear path, and the engine has one.

```
qzd173/wadsrc/static/zscript/actors/actor.zs:270   native readonly double AttackRoll;
qzd173/wadsrc/static/zscript/actors/actor.zs:274   native readonly double OffhandRoll;
qzd173/src/playsim/actor.h:1657, :1664             DAngle AttackRoll / OffhandRoll
qzd173/src/scripting/vmthunks_actors.cpp:2189, :2193   DEFINE_FIELD
```

Written from the controller every frame, on both backends:

```
qzd173/src/rendering/gl/stereo3d/gl_openvr.cpp:2537   AttackRoll  = weaponangles[ROLL]
qzd173/src/rendering/gl/stereo3d/gl_openvr.cpp:2553   OffhandRoll = offhandangles[ROLL]
qzd173/src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:591, :604   the same
```

**Two findings here, both load-bearing, both verified rather than assumed.**

1. **The roll is never zeroed, anywhere in the tree.** The coordinator's note is correct, and here is
   the precise reason. `VRMode::SetUp` runs every frame and resets the hand fields to the body centre
   (`qzd173/src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:432-448`) — it writes
   `AttackPos`/`OffhandPos` (`:442`), `AttackAngle`/`OffhandAngle` (`:443`),
   `AttackPitch`/`OffhandPitch` (`:444`) and `HmdPosition` (`:445`) — and **it never touches
   `AttackRoll` or `OffhandRoll` at all.** A grep for writes across `src/` returns only the two device
   files above. So the Wheel's claim that the roll is zeroed before `WorldTick` is false at this tag.
   **But be exact about what that buys:** because `SetUp` skips it, roll is not zeroed on a collapsed
   frame — it is **stale**, holding its last real value. So roll cannot itself tell you the frame was
   real. A roll-based detector must still gate on `OverrideAttackPosDir` and on the *position* spike
   filter (§3.7). Stale is a subtler failure than zero, and worth knowing before it costs a day.
2. **Roll survives `weaponStabilised`, and it is the only hand angle that does.** With
   `vr_two_handed_weapons` true (default, `hw_vrmodes.cpp:119`), holding the engine's offhand grip with
   the hands inside 0.50 m latches stabilisation (`gl_openvr.cpp:1704-1727`) and the engine overwrites
   the main hand's pitch and yaw with the hand-to-hand vector — but look at what it passes through:
   ```
   qzd173/src/rendering/gl/stereo3d/gl_openvr.cpp:1752-1753
     VectorSet(weaponangles, -RAD2DEG(atanf(y / zxDist)), -RAD2DEG(atan2f(x, -z)),
               weaponangles[ROLL]);          // <-- ROLL is preserved verbatim
   ```
   So in a two-handed grip — where pitch and yaw are hijacked — **the wrist-roll signal is still
   clean.** That is exactly the signal the two-handed style (§6) needs, and it arrives for free.

The detector:

```zscript
double roll0 = rollRing[hand][0], roll1 = rollRing[hand][1];
double dRoll = Actor.DeltaAngle(roll1, roll0);               // shortest arc, signed, deg/tic
double flick = clamp(abs(dRoll) / FLICK_REF, 0.0, 1.0);      // FLICK_REF ~ 120 deg/tic
```

`Actor.DeltaAngle` is the right tool and already in house — `monster.zs:279` uses it for a tumble's
residual facing. The roll must be streamed alongside the position: `climb.zs:68-78`'s packing has no
room, so send roll as a second net-event arg (the `rsqgr_pose` event already carries `hand` in
`Args[0]` and the packed offset in `Args[1]`; `Args[2]` is free — `input.zs:104`).

**Label: BUILDABLE / S.** The field is there, the idiom is there, the only work is one more packed arg.

### 3.5 Energy, and what it drives

```zscript
double E = clamp(loadLift * (W_SNAP * (snapSpeed / SNAP_REF) * window
                           + W_FLICK * flick), 0.0, 1.0);
// W_SNAP ~ 0.7, W_FLICK ~ 0.3 -- the reference: the snap is "the single biggest contributor"
```

`loadLift` multiplies rather than adds (§3.2). `window` can zero the whole thing (§3.3). Then one
quantity sets everything:

```zscript
waveSpeed = lerp(WAVE_SLOW, WAVE_FAST, E);      // 24 .. 96 u/tic -> loop arrives in ~12 .. 3 tics
reach     = lerp(REACH_MIN_FRAC * ceilReach, ceilReach, E);   // ceilReach = rsqgr_whip_reach
damage    = max(1, int(ceilDamage * (0.25 + 0.75 * E)));      // ceilDamage = rsqgr_whip_damage
```

**`rsqgr_whip_reach` (192) and `rsqgr_whip_damage` (12) become ceilings**, keeping their names, their
CVARINFO rows and their existing sliders (`MENUDEF.txt:117-118`). No ini breaks; the values become the
top of a range. Same convention as `01` §8.4. `REACH_MIN_FRAC ≈ 0.35` — a cold whip must visibly fail
to get there; that is what teaches the technique with no tutorial. **Spend the energy at the crack**
(`E = 0`), so the gauge visibly empties.

### 3.6 A failed crack must visibly fail

When `E < FAIL_E` (≈ 0.15) or `window == 0`:

- `waveSpeed` stays at `WAVE_SLOW` and the loop never reaches the popper inside `rsqgr_whip_life`.
- Wave amplitude is scaled by `E` (§5), so there is **no clean loop** — just a sagging thong flopping
  forward.
- No crack sound, no `RSQGR_Looks.Spark` (`whip.zs:144`), no damage, no snag. The lash times out and
  the state goes to `WS_RECOVER`.

That is reference step 5 as a game mechanic, at the cost of one comparison. It is also the only
honest way to make §3.3's window matter.

### 3.7 Arming, and the spike filter that must land first

Arming reuses the existing press/hold split: `KEYCONF.txt` gives `+rsqgr_main` / `+rsqgr_off` a real
press/release pair, and `RSQGR_Core` tracks `held[i]` / `pressTic[i]` (`core.zs:20-21`, `:59-60`)
against an 8-tic tap window (`TAP_TICS`, `core.zs:17`). **Tap = fire the claw, unchanged. Hold = the
whip is live and the phase machine runs.** The whip no longer needs a release event to throw — the
crack is fired by the loop.

**Prerequisite, and it is a shipped bug.** `RSQGR_Util.PoseGood` (`util.zs:51-60`) gates on
`OverrideAttackPosDir` and rejects a hand beyond `ARM_MAX = 96` (`util.zs:20`) — but it does **not**
reject the engine's hand-collapse. On any frame where the device update does not run, `SetUp` leaves
both hands at the body centre (`hw_vrmodes.cpp:442`), and the device code runs only inside
`if (gamestate == GS_LEVEL && menuactive == MENU_Off && !paused)` (`gl_openvr.cpp:1704`). A collapsed
pose sits ~41 units above the pawn — inside `ARM_MAX` — so `PoseGood` returns **true**. `HANDOFF.md`
already reports the symptom ("a press in the first frame after a menu closes fires from the body").
For the phase machine it is worse: a collapse-then-restore pair is a huge apparent lift **and** a huge
apparent snap, so **closing a menu would crack the whip at full power.** Two lines:

```zscript
if ((pmo.AttackPos - pmo.OffhandPos).Length() < 0.01) return false;    // both hands collapsed
if ((now - prev).Length() > SPIKE_REJECT) { invalidate ring; return false; }   // ~40 u/tic
```

---

## 4. THE TAPER

### 4.1 Draw it

`whip.zs:216` computes one width for the whole thong and `:239` applies it:

```
whip.zs:216   double w = RSQGR_Util.Num("rsqgr_rope_width", 0.6) * 0.7;   // a whip is thinner
whip.zs:239   r.scale = (w, l / RSQGR_Rope.ROPE_L);
```

Taper is one expression inside the loop that already exists (`whip.zs:218-242`, `u = (k+1.0)/NLINK` at
`:220`):

```zscript
double wk = w * (TAPER_TIP + (1.0 - TAPER_TIP) * (1.0 - u));   // TAPER_TIP ~ 0.25
r.scale = (wk, l / RSQGR_Rope.ROPE_L);
```

Thong thick at the handle, popper a quarter as thick. The **fall** is the last two or three pieces,
i.e. `u > 1 − 3/NLINK`; if it wants its own look, that is where. The cable style
(`RSQGR_Rope`, a tube model, `hook.zs:363-386`) takes a per-piece width natively because width is an
X-scale; the **chain** style (`RSQGR_Link`, `hook.zs:337-360`, placed at `hook.zs:249-279`) uses a
single `rsqgr_chain_scale` for every link, so tapering it means scaling per link index there too —
cheap, but note `hook.zs:230-231`'s `CHAIN_START = 15.0` / `CHAIN_STEP = 5.0` are matched to
jpalomo's measured original and the comment at `hook.zs:143-156` records what went wrong last time
someone changed link size ("fat white balls in the face"). **Taper the width, not the spacing.**

### 4.2 It is also where reach and damage come from

The taper is the physical reason a small input at the handle becomes a large effect at the tip, and
§3.5 already expresses that: `E` is a handle-side quantity, and `reach`/`damage`/`waveSpeed` are
tip-side consequences of it. The existing wave amplitude term at `whip.zs:223` is `len * 0.06 * u` —
**already proportional to `u`**, i.e. already growing toward the tip. That is the amplification,
already drawn. Tie its gain to `E` (§5) and the picture and the numbers tell the same story.

**Label: BUILDABLE / S** for the drawn taper. **The amplification is already the shape of `:223`.**

---

## 5. SLACK AS GAUGE, REVISED — it now reads the STATE

Three expression swaps inside `Draw`. The gauge no longer reads an accumulated circle count; it reads
which phase the whip is in, which is better physics and the same visual language.

```zscript
// --- replaces whip.zs:213 ---
// Depth of the hanging bow by phase. Laid out = heavy; loading = pulling taut; flying = near flat.
double bowRest = len * BOW_REST;                      // BOW_REST ~ 0.22 (deeper than today's 0.08)
double taut    = (st == WS_LAID)  ? 0.0
               : (st == WS_LIFT)  ? 0.45 * loadLift
               : (st == WS_SNAP)  ? 0.80
               : (st == WS_FLY)   ? 0.90 + 0.10 * E   // taut thong with a loop running down it
               :                    0.25;             // WS_RECOVER: slack coming back
double bow = bowRest * (1.0 - 0.85 * taut);           // never fully zero: a taut rope still bows

// --- replaces whip.zs:222 ---
// Sign follows the STATE, not the tic counter, and cross-fades so it never pops.
double lift = clamp((taut - 0.5) * 2.0, -1.0, 1.0);   // -1 hanging .. +1 lifted and travelling
pb.z += bow * sin(180.0 * u) * lift;

// --- replaces whip.zs:223 ---
// The loop: amplitude from E (a failed crack shows NO clean loop), position from phase (§2.1).
double amp = len * WAVE_AMP * (0.15 + 0.85 * E);      // WAVE_AMP ~ 0.06, as today
pb += side * (amp * u * sin(360.0 * (1.5 * u - phase[i])));   // phase[i] now accumulated, §2.3
```

Four deliberate choices:

1. **`bow` never reaches zero.** A perfectly straight line stops reading as a rope. Keep ~15 %.
2. **The lift cross-fades instead of flipping.** Today's flip is instantaneous at `age == OUT_TICS`
   (`whip.zs:222`) and pops. Fading through straight makes the rope *rise through* flat as it loads,
   which is the most legible moment in the gauge: **straight means loaded.**
3. **`phase` accumulates (§2.3) and is no longer `age / OUT_TICS`** (`whip.zs:214`). This is what
   removes the stopwatch from the visual as well as from the logic, and it is what makes the drawn
   loop and the crack the same event.
4. **The loop's amplitude carries `E`.** A botched crack draws a flat, dead rope. Step 5, free.

**One more, and it is free correctness:** pass any per-tic phase step through `RSQGR_A11y.Deg()`
(`a11y.zs:39`), which caps it at `MAX_DEG = 25.7` (2.5 Hz) under Photosensitive Mode
(`a11y.zs:20-23`, `:34`). An `E`-driven accelerating loop is exactly the flicker that rule exists for,
and the helper is already written. Do not reimplement the cap.

**No HUD.** A meter would need text and cue scaling (`a11y.zs:57-61`), a VR-safe position and a
multiplayer answer. The rope needs none and is already drawn and already measured. One concession,
routed properly: `RSQGR_A11y.CueVisual()` (`a11y.zs:64`) for an optional charge ring sized by
`RSQGR_A11y.Cue()` (`a11y.zs:57`), `CueRumble()` (`:65`) for a rising per-hand tick. Both inert
without `RSQ_Accessibility`.

**Label: BUILDABLE / S.**

---

## 6. TWO-HANDING — the stylised input to the same engine

Real technique is one-handed. The owner wants Castlevania as well. **Both ship, as two inputs to one
energy term.**

```
server int rsqgr_whip_style = 0;    // 0 authentic (one hand, lay-lift-snap-flick), 1 two-handed wind
```

**Recommendation: `0` as the default**, and the reason is not authenticity. The four phases *are* the
skill, and they are what will make the weapon feel earned; the two-handed wind is one accumulating
number, easier and shallower. Ship `1` for the Belmont fantasy and for players who cannot perform the
four beats. **The owner's call; that is the reason behind the pick.**

The two-handed wind, if selected, is the angular accumulator — and it must be derived from hand
**positions**, not from `AttackAngle`/`AttackPitch`, because a two-handed grip is inside the 0.50 m
stabilisation window and those two angles get hijacked (`gl_openvr.cpp:1739-1756`). Positions are
untouched (`:1733-1738`), and the **roll is passed through verbatim** (`:1752-1753`, §3.4), so a
position-plus-roll wind-up is immune on either backend:

```zscript
Vector3 mid = (handRing[MAIN][0] + handRing[OFF][0]) * 0.5;
double  rad = mid.xy.Length();
double dphi = Actor.DeltaAngle(phiPrev, atan2(mid.y, mid.x));     // signed, deg/tic
if (sign(dphi) != sign(dphiPrev)) gain = 0;                       // reversed: bank nothing
E = min(E + abs(dphi) * (rad / WIND_RADIUS) * K_GAIN, 1.0);
E *= WIND_DECAY;                                                  // ~0.951/tic = 0.4 s half-life
```

Exponential decay, not linear, for a concrete reason: it is scale-free, so the gauge in §5 moves at a
visually constant rate at every charge level. A linear drain empties a low charge suddenly and the rope
appears to collapse. Target full `E` in two to three deliberate revolutions.

The mod is otherwise **not exposed** to the two-hand trap at all, and that was verified: it defines
**no `Weapon` subclass** (24 classes under `zscript/rsqgr/`, none deriving from `Weapon`), and grab
state comes from its own KEYCONF aliases rather than the engine grip (`KEYCONF.txt`,
`input.zs:68-71`, `core.zs:536-547`) — precisely the defence `00_CAPABILITIES` gotcha 10a recommends,
reached independently. **Do not change that.**

**Label: BUILDABLE / M** (it is a second input path, not a second implementation).

---

## 7. RESOLVING THE HIT — the swept chord

A single ray on one tic (`whip.zs:118`) samples the lash's final line and nothing it swept. At a
192-unit thong with the wrist turning 60° in a tic, the tip bulges `192 × (1 − cos 30°) ≈ 25.7` units
past the chord — **larger than an imp**, which is radius 20 and, because Doom's actor test is
Chebyshev rather than circular (`qzd173/src/playsim/p_map.cpp:1417-1419`, trace path
`qzd173/src/playsim/p_trace.cpp:695-696`, neither reading `Angles.Yaw`), a 40×40 axis-aligned square.
Loop-arrival cracking makes this *more* urgent, because a hard snap now arrives in 3 tics.

`01_MELEE` §3.5, with the whip's adaptation — interpolate the **angles**, since the tip's motion is
rotation-dominated:

```zscript
const SUBS = 3;                      // 3 below ~24 u/tic of TIP speed; 5 above
Vector3 prevTip = TipAt(0.0), prevRoot = RootAt(0.0);
for (int i = 1; i <= SUBS; i++) {
    Vector3 tip = TipAt(double(i)/SUBS), root = RootAt(double(i)/SUBS);
    if (Sweep(prevTip, tip)) { resolve; break; }    // what the TIP swept across
    if (Sweep(root, tip))    { resolve; break; }    // something standing IN the lash
    prevTip = tip; prevRoot = root;
}
// angle lerp takes the shortest arc: a0 + Actor.DeltaAngle(a0, a1) * t   (monster.zs:279 idiom)
```

**Cost: 6 traces/tic per lashing hand, 12 both hands; 10 / 20 at `SUBS = 5`** — only during the lash.
`01_MELEE` §3.4 calls 210 traces/s "nothing"; a `LineTracer` call is one `FPathTraverse`; the mod's
measured budget is core `WorldTick` mean 0.041 ms / max 0.098 ms under the owner's full load order
with two hooks out (`HANDOFF.md`).

**No new primitive.** `RSQGR_Tracer` (`hook.zs:390-411`) is already a `LineTracer` subclass taking an
arbitrary start and direction (`qzd173/wadsrc/static/zscript/doombase.zs:310`), whose `TraceCallback`
already rejects the shooter, players, the mod's own rope actors, missiles and non-shootables
(`hook.zs:396-407`). That is `TRF_ABSPOSITION` without the flag, shipped and tested.

One guard, because the box is square: confirm `Distance3D(HitActor)` against `victim.radius + fatten`
after a hit, or a corner graze registers as a clean body hit from 1.41× the intended reach.

**A bonus from §2.** Because the crack now fires when the loop reaches the popper, the tip's position
at the crack is a **derived physical quantity** rather than a timer's guess. Aim mode 0 (§8.1) is
therefore meaningfully better than it was: "where the tip is" has a real answer.

---

## 8. THREE TARGETING MODES

### 8.0 The cvars, in the file's own style

```
// CVARINFO.txt, inside the existing ==== monster side ==== whip block, after rsqgr_whip_above
server int    rsqgr_whip_aim       = 1;    // where the crack goes: 0 free (what you threw), 1 snap to the best target in an arc, 2 the target you locked
server float  rsqgr_whip_arc       = 28;   // mode 1: half-angle of the arc searched, degrees
server float  rsqgr_whip_bend      = 18;   // mode 1: most the tip is bent toward it, degrees
server int    rsqgr_whip_lock_hold = 10;   // mode 2: tics the lock bind must be held to take a lock
```

```
// MENUDEF.txt -- beside RSQGR_MonsterModes / RSQGR_Tumbles / RSQGR_LookSets
OptionValue "RSQGR_WhipAims" { 0, "Free"  1, "Snap To Arc"  2, "Locked Target" }
// front page, immediately after MENUDEF.txt:75   Option "Whip", "rsqgr_whip", "OnOff"
    Option "Whip Aim",   "rsqgr_whip_aim",   "RSQGR_WhipAims"
    Option "Whip Style", "rsqgr_whip_style", "RSQGR_WhipStyles"      // §6
// Tuning and Debug, after MENUDEF.txt:120   Option "Whip Snag", ...
    Slider "Whip Arc", "rsqgr_whip_arc", 0, 60, 4, 0
    Slider "Whip Bend", "rsqgr_whip_bend", 0, 40, 2, 0
    Slider "Whip Lock Hold", "rsqgr_whip_lock_hold", 4, 24, 2, 0
```

Aim and Style earn front-page rows because they are real named choices, which is the rule stated at
`MENUDEF.txt:1-4`; numbers go to Tuning, where every other number lives.

### 8.1 Mode 0 — Free

The chord between the last two samples is traced (§7); whatever it crosses is hit. Closest to
`whip.zs:118` today, minus the stopwatch.

**Feels** honest and physical. **Fails** on sampling: poses are written per rendered frame
(`gl_openvr.cpp:2549-2552`) but read once per tic (`00_CAPABILITIES` §1.4), so a fast lash is three or
four snapshots of a motion the player felt as continuous. The swept chord recovers the *geometry* of
what was crossed, never the player's *intent* between samples — misses that feel unjust, the hardest
failure to forgive. **Hardest of the three to enjoy; not the default.** It is the right baseline for
every rig measurement in §11 because it has no assist to confound a reading.

**Label: EXISTS (modulo §7) / S.**

### 8.2 Mode 1 — Snap within arc — **RECOMMENDED DEFAULT**

**The picker — reuse the mod's furniture, do not write a second one.** `monster.zs` already knows
which monsters are legitimate whip/yank subjects:

```
monster.zs:199-208   Throwable(a, by): rejects bDONTTHRUST, bBOSS, over rsqgr_yank_mass,
                     wider than rsqgr_yank_radius (64), taller than rsqgr_yank_height (110)
hook.zs:396-407      RSQGR_Tracer.TraceCallback: rejects the shooter, players, the mod's own
                     rope actors, missiles, non-shootables, Health <= 0
monster.zs:513       BlockThingsIterator.Create(a, 8) -- the iterator idiom, already in use
```

One `BlockThingsIterator` over `reach`, filtered by the **existing** `Throwable` gate and the
**existing** `TraceCallback` predicate, then one line-of-sight `Trace` per survivor through the
**existing** `RSQGR_Tracer`. A Champions champion or a LegenDoom legendary is then classified by the
same code that already classifies it for the yank — which is the whole reason to reuse rather than
write a second rule that drifts.

**"Best" = most aligned, tie-broken by nearest. Explicitly not lowest health.**

- **Most aligned** (largest `dir dot toTarget.Unit()`) is the only criterion that is a function of
  **what the player did**. The player aimed; the assist's job is to honour the aim, not second-guess
  it. Anything else is felt immediately as the game choosing for you.
- **Nearest as tie-break**, because within a few degrees the near one is what they were looking at,
  and a whip's reach falls off.
- **Not lowest health.** Tempting, and wrong: it makes the whip jump to a dying enemy while the
  player swings at the healthy one in front of them. The single most-complained-about assist
  behaviour in the genre. Reject it.
- **Hard gate first:** line of sight. A target behind a wall is not a candidate however well aligned —
  one `RSQGR_Tracer` call, and why the iterator's wall-blindness does not matter here.

**Tuning.**

- **`rsqgr_whip_arc` 28° half-angle.** Reasoned from the sampling, not guessed: at 35 Hz a committed
  lash turns the wrist roughly 40-60° in a tic, so the player's own uncertainty is of that order.
  Under ~20° the assist seems not to work; over ~35° it hits things nobody aimed at.
- **`rsqgr_whip_bend` 18° max, and it is the one that matters** because it is what the player *sees*.
  Keep it well under the arc: the cone decides **whether** to help, the bend **how much it may show**.
  A target 28° off is bent toward by at most 18° and is still missed — correct, and what keeps the
  assist honest.
- **Scale the bend by `E`.** A wound whip finds its mark; a cold flop gets no free hit. This stops the
  assist masking a failed technique, so §5's gauge keeps its teaching job.
- **Route the arc through accessibility.** `RSQGR_A11y.Wide(tol)` (`a11y.zs:47`) multiplies a tolerance
  ×1..×2 under `rsqa_gesture_ease`, and `core.zs:566` already applies it to `rsqgr_grab_reach`. One
  setting, no second knob, inert without `RSQ_Accessibility`.

**Fails** in crowds — most-aligned among five is near-arbitrary from the player's view, and no
tie-break fixes that honestly. Mitigation: shrink the effective arc as candidates rise (`arc / sqrt(n)`),
so a pack forces a more precise throw. A tuning line, not a redesign; try it before reaching for
lock-on to solve it.

**Cost:** one blockmap walk plus a few LoS traces, **once per crack**, not per tic. Same order as
`monster.zs:513`, inside a budget measured at 0.036 ms mean for a thrown Baron (`HANDOFF.md`).

**Label: BUILDABLE / M.**

### 8.3 Mode 2 — Explicit lock

**The bind.** Four exist (`KEYCONF.txt`): `+rsqgr_main`, `+rsqgr_off`, `+rsqgr_grip_main`,
`+rsqgr_grip_off`. Three options were considered:

1. **Reuse the grab bind while a whip is live.** Free — nothing to climb then. **Rejected:**
   `rsqgr_grab_dg` defaults true and makes `RSQ_DistanceGrab`'s grips call `core.Grab`
   (`input.zs:149-150`), so a DG grip would silently take a whip lock. A cross-mod surprise is exactly
   what the series rule exists to prevent.
2. **A fifth bind.** `KEYCONF.txt:9-12` notes the owner binds the stick clicks himself and the mod
   ships **no default controller binds**, so a fifth key costs him one binding and nobody else
   anything. **Recommended.**
3. Hold-direction-for-N-tics, no button. Rejected: fires while just looking around, and is
   unteachable.

```
addmenukey "Lock Whip Target" "+rsqgr_whip_lock"
alias "+rsqgr_whip_lock" "event rsqgr_wlock 1"
alias "-rsqgr_whip_lock" "event rsqgr_wlock 0"
```

The event path is already laid: a local `event` reaches `ConsoleProcess` on the machine owning the
controller, the pose is read there and packed into a network event (`input.zs:1-20`, `:28-72`,
`:108-151`). Pick the candidate locally with §8.2's picker, confirm the actor over the net. It is the
only multiplayer-correct pattern in the mod.

**Showing the lock** — all three already exist:

- **A sound at the target**, via the one-shot carrier: `Sound2` spawns an `RSQGR_WhipTip` at a point
  and plays on it (`whip.zs:158-164`), name resolved per look set by `RSQGR_Looks.Sound`
  (`looks.zs:37-63`). One new name, three sets.
- **A per-hand rumble — already in the API and already named for this.** `RSQGR_A11y.Ready(who, hand)`
  is commented *"a lock / grab-ready cue on a hand (0 main, 1 off): a short rumble when Sound Cues has
  it"* (`a11y.zs:92-98`). It is literally the lock cue, written before anyone asked for a lock.
- **A marker** when `RSQGR_A11y.CueVisual()` (`a11y.zs:64`), sized by `Cue()` (`:57`).

**What breaks it** — the first four tests already exist:

| Breaker | Where the test is |
|---|---|
| dead, destroyed, or turned unshootable | `core.zs:332`'s live re-check (added after a measured Champions "Dark Red" bug, `HANDOFF.md`) |
| out of reach | `core.zs:337`'s range guard |
| line of sight lost | `core.zs:640-649` `Sees()`, every 4 tics, 3 strikes |
| no longer a legitimate subject | `monster.zs:199-208` `Throwable`, re-asked |
| the arm swings >~70° off the locked bearing | **new, and necessary** — without it the lock survives the player deliberately turning away and the whip curves like a homing missile |
| the crack, or a timeout | — |
| save/load, and the first tic after | `core.zs:162-170` `Settle()` — **mandatory** |

**`Settle()` is not optional.** `core.zs:154-170` resets per-hand latches on the first tic the handler
object has not seen, *not* in `WorldLoaded`, because a saved handler's fields are read back **after**
`WorldLoaded` — the comment records the bug ("a hold reset there came back from the save and kept
climbing"). A lock is that kind of latch, and an `Actor` pointer besides.

**Feels** like a gun. The intermediate skill — *throwing* at a thing — is removed, and with it most of
what makes a whip a whip. **Recommended against as default, recommended for shipping:** the owner
asked, and it is the only mode that serves a player who cannot aim precisely, an audience the series
already caters to (`a11y.zs:46-48`).

**Fails** on target churn. Champions and LegenDoom Lite both replace actors mid-life, and
`HANDOFF.md`'s `agent-compat` section records two measured instances under the hook (mass 1200
mid-pull; an actor turning unshootable and parked in a map corner). Holding an `Actor` across a
wind-up is the same exposure with a longer window. **Re-ask every breaker every tic, not at the
crack** — the lesson `core.zs:327-332` already teaches.

**Label: BUILDABLE / M** for the mechanism; **the compat hardening is the L-shaped part**, budgeted
separately.

### 8.4 Summary

| Mode | Label | Cost | Default? | Verdict |
|---|---|---|---|---|
| 0 Free | EXISTS (with §7) | S | no | Honest; hardest to enjoy; the clean measurement baseline |
| **1 Snap To Arc** | BUILDABLE | M | **YES** | The mode that makes the feature work |
| 2 Locked Target | BUILDABLE | M (+L hardening) | no | Reliable, readable, most likely to feel like a gun |

---

## 9. THE FOUR PRESETS

### 9.0 The matrix — this table is the argument that it is one system

| | **Bulletstorm** (Leash) | **Indiana Jones** | **Thief** (rope arrow) | **Bionic Commando** |
|---|---|---|---|---|
| **Anchor type** | actor | geometry | geometry, **persistent** | geometry |
| **Length policy** | reeling | fixed → free (pays out) | **free**, climbable | reeling (retract is the verb) |
| **Tension** | slack → rigid yank | rigid | rigid | rigid |
| **Hand count** | one | one (two optional) | **two** (hand over hand) | one |
| **Who the constraint moves** | **the TARGET** | the player | the player, **by climbing** | the player |
| **Already built** | ~80 % (the Yank) | ~90 % (+ this whip spec) | ~60 % (climb is done) | ~70 % (the solver) |
| **Missing** | hang phase + Score handoff | nothing structural | material gate + persistence | **`keepmomentum`** + no-jump |
| **Label / cost** | BUILDABLE / **S-M** | EXISTS / **S** | BUILDABLE / **M** | BUILDABLE / **S** core, M tuning |

**My read agrees with the coordinator's on three of four, with one correction:** Bulletstorm is
*cheaper* than expected, because the hang needs neither `freezetics` nor a per-actor `RSQ_SlowMo`
pulse — §9.1 explains why.

Each preset is a named cvar set over the one engine, delivered the way `MENUDEF.txt:38-59` already
delivers presets — generated by `tools/rsmenu/presets.py`, **not** hand-written into MENUDEF
(`MENUDEF.txt:1-4`).

```
OptionString "RSQGR_P_Preset"
{
    "<Bulletstorm set>",      "Leash"
    "<Indiana Jones set>",    "Indiana Jones"
    "<Thief set>",            "Rope Arrows"
    "<Bionic Commando set>",  "Bionic Arm"
}
```

---

### 9.1 BULLETSTORM — the Leash

**What the player does.** Fires at an enemy. He is yanked off his feet, tumbles through the air, and
instead of landing at your feet he **hangs there**, floaty and helpless, while you shoot him apart.
The rope moves *him*. It is not traversal; it is a setup move, and the payoff is the shooting.

**Cvars.**

```
rsqgr_geo_mode 0; rsqgr_monster_mode 1;       // Yank, not Slide
rsqgr_yank_tumble 3; rsqgr_yank_turns 1;      // spin + flip
rsqgr_yank_air 0.27; rsqgr_yank_stun 20;      // floaty on release; longer helpless window
rsqgr_yank_hang 35;                           // NEW: ~1 s aloft
rsqgr_whip 0;                                 // the claw, not the whip
rsqgr_mass_max 1000; rsqgr_yank_mass 1000
```

**What is already built.** Nearly all of it, and measured. `RSQGR_Flight` (`monster.zs:150-652`) is the
arc (`e = t^1.6` at `:443-446`, `MeasureArc` ceiling clamp at `:336-357`, weight terms at `:248-253`),
the tumble (`StepTumble`, `:627-645`; spin by angle, flip by roll on sprites and by pitch on voxels
through `SetModelFlag`), collision-correct placement (`PlaceMid`, `:362-390`), impact and landing
damage (`:536-589`), and flag borrow/restore on every exit path including level unload
(`:600-616`, `:648-651`). `HANDOFF.md` records identical arcs at full and quarter world speed across
nine monster classes. The service already answers `"yanked"` 1/2 (`service.zs:427-432`), `"struck"` /
`"struck.by"` / `"pin"` for bowling (`:441-443`) and `"landing"` (`:440`).

**What is missing — two things, and the first is small.**

1. **A hang phase.** The path always *terminates* at `LandMid` — a point in front of the player
   (`monster.zs:212-225`), reached at `ftt >= fT` (`:468-476`). Bulletstorm holds the enemy aloft
   instead. Add `FP_HANG` to the enum at `monster.zs:153`, entered at the arc's apex
   (`ftt / fT ≈ 0.5`, where the `4·arc·e·(1−e)` term at `:446` peaks), holding the actor by
   `PlaceMid` for `rsqgr_yank_hang` tics, then falling to `FP_AIR` as today.

   **Correction to the brief: this needs neither `freezetics` nor a per-actor `RSQ_SlowMo` pulse.**
   Two reasons, both from the shipped code:
   - **The actor is already fully puppeted.** `RSQGR_Flight.WorldTick` writes `a.Vel = (0,0,0)` and
     re-pins `a.tics = -1` **every tic** (`monster.zs:419-421`), borrows `bNOGRAVITY`
     (`:259`) and places the actor by position (`:362-390`). `freezetics` early-returns out of
     `AActor::Tick` before any movement (`qzd173/src/playsim/p_mobj.cpp:3948-3952`) — it would
     suppress a Tick whose every effect this code already overrides, and it would *also* stop the
     actor being placed. It is the right tool for an actor you want the engine to leave alone; it is
     the wrong tool for one you are already driving. Floatiness here is the **arc**, not a clock.
   - **Per-actor slow is not reachable through the service as written anyway.**
     `RSQGR_Util.WorldScale` (`util.zs:90-100`) asks `RSQSM_Service` for `GetInt("scale", "", 0, 0,
     null)` — a **global** query with a null `objectArg`. Nothing in that call names an actor. So a
     per-victim float would need a new request on RSQ_SlowMo's side, which is out of scope and
     unnecessary given the above.
   - **It renders smoothly for free.** Gotcha 12: a position written every tic interpolates cleanly;
     it is intermittent movement that strobes. The mod writes every tic.
2. **The handoff that tells Score a juggle started.** `service.zs` has no juggle notion. The Grapple's
   job is only to *announce* — the coordinator reports `RS_Bonus_Juggle` and the combo ledger already
   live in RSQ_Score. So: one `RSQGR_Events` call at the hang's start, a new `"hung"` request (tics
   remaining aloft; −1 otherwise), and one award push through the door that already exists —
   `Award(who, label, points)` at `service.zs:370-381`, which is how `"Indiana Jones"` is already paid
   (`:357`). Nothing links; absent RSQ_Score it answers −1 and nothing else changes.

**The no-progress guard will fight this preset specifically — flagged as asked.**

```
core.zs:338-339   if (d < h.bestDist - 0.25) { h.bestDist = d; h.noProgress = 0; }
                  else if (++h.noProgress > 52) { LetGo(h, false); return; }
```

A hung monster makes no progress **by definition**, so the rope is dropped after 52 tics (1.5 s). With
`rsqgr_yank_hang 35` it survives by luck, with a 0.5 s margin — unacceptable. Use the same fix as the
lasso (`01` §4.2): a `bool holding` on `RSQGR_Hook`, set by the monster side, read here —
`else if (!h.holding && ++h.noProgress > 52)`. Keep the out-of-range guard at `core.zs:337`. Then hang
time is free rather than capped by an unrelated constant.

**Label: BUILDABLE / S** for the hang, **S** for the Score handoff, **S** for the guard. **S-M total.**

---

### 9.2 INDIANA JONES

**What the player does.** Cracks the whip at a beam, a lintel, a ledge above — and swings. Lands,
runs, cracks again. And cracks *at* things: the forced Pain breaks off an attack a monster was
winding up, which is the disarm (§I.8) by the only means the engine allows.

**Cvars.**

```
rsqgr_whip 1; rsqgr_whip_style 0; rsqgr_whip_aim 1;   // one-handed crack, snap-to-arc
rsqgr_whip_snag 1; rsqgr_whip_above 16;                // the lash catches overhead and becomes a rope
rsqgr_whip_stun 1;                                     // the disarm
rsqgr_geo_mode 0;                                      // swing, not JP's yank
rsqgr_reel_speed 12; rsqgr_pump 0.25; rsqgr_hop 6;
rsqgr_max_speed 45;                                    // raised: the 30 cap is folklore, 01 §6.4
rsqgr_climb 1
```

**Already built, and this is the one to be least worried about.** The snag path —
`whip.zs:145-153` → `Snag()` (`:168-196`) → `RSQGR_Core.Latch()` (`core.zs:289-312`) — builds core's
hook **exactly as a claw that hit that spot would have**, so swing, reel, climb and tap-to-let-go are
core's own unchanged code. `HANDOFF.md`: *"Snag: tip on the alcove lintel 91 u above the hand -> core's
own Latch(); reel and swing are core's."* The pendulum is measured (L 643.2, half period 82 tics
against a theoretical 81), the Indiana Jones gap award ships (`service.zs:336-359`), and the disarm
ships (`whip.zs:131-136`: `FindState("Pain")` forced regardless of pain chance, exposed as `"stunned"`
at `service.zs:445-454` for `RSQ_TopHPBars`).

**Missing:** nothing structural. This preset is the whip spec above, plus `keepmomentum` (§9.4) so the
landing keeps its speed.

**Label: EXISTS / S.**

---

### 9.3 THIEF: THE DARK PROJECT — rope arrows

**What the player does.** Plants a rope high on a wooden beam. It **stays there** for the rest of the
level. Walks up to it later, grips, climbs hand over hand, and gets somewhere that was not reachable.
And — the rule that made it a puzzle rather than a traversal cheat — **it only bites wood.** Stone
refuses, and you have to find another way.

**Cvars.**

```
rsqgr_geo_mode 0; rsqgr_climb 1; rsqgr_climb_free_only 1;
rsqgr_grab_reach 16; rsqgr_climb_cap 8;
rsqgr_rope_persist 1;        // NEW: a latch plants a rope that outlives the throw
rsqgr_rope_material 1;       // NEW: 0 any surface, 1 the wood list only
rsqgr_whip 0; rsqgr_reel_speed 8; rsqgr_pump 0.1
```

**The material gate — the engine gives this for free, and no report has used it.**

**Correct the field, though.** The brief cites `FLineTraceData.HitTexture` (`actor.zs:53`), which is
the `LineTrace` result struct. **This mod does not use `LineTrace`** — it uses its own `LineTracer`
subclass, so the field is:

```
qzd173/wadsrc/static/zscript/doombase.zs:282   native TextureID HitTexture;   // struct TraceResults
qzd173/wadsrc/static/zscript/engine/base.zs:318  native static String TexMan.GetName(TextureID tex);
```

And the place to test it is the geometry branch of `Fly`, where `tr.Results` is already in hand and
the anchor is already being built:

```
core.zs:272-286   // geometry: wall, floor, ceiling, 3D floor side or plane
core.zs:276       h.anchorSec = tr.Results.HitSector;
core.zs:286       Latch(h, i);
```

So the gate is a name-list test immediately before `core.zs:286`, against
`TexMan.GetName(tr.Results.HitTexture)` — a mod-owned list of texture names (`BROWN*`, `WOOD*`,
`SUPPORT*`, `PLANKS*` for Doom 2; the list belongs in a data lump, not in code, so a mod author can
extend it). `rsqgr_rope_material 0` turns the gate off for players who want traversal without the
puzzle, exactly as asked.

**How the player learns a surface refused** — the whole vocabulary already exists:

- a distinct `RSQGR_Looks.Spark(at, ...)` (used at `whip.zs:144`, `core.zs:285`) and a "skid" sound
  through `RSQGR_Looks.Sound` (`looks.zs:37-63`), one new name across three look sets;
- the claw then retracts, which already plays its own sound (`hook.zs:97-101` on `PH_RETRACT`);
- optionally `RSQGR_A11y.CueVisual()` (`a11y.zs:64`) for a marker.

**Crucially: the refusal must be legible as "wrong surface", not as "you missed."** The claw visibly
*touching* the stone and sliding off is the read; an instant retract with no contact is not.

**How much of hand-over-hand is there: all of it.** `climb.zs` (79 lines) plus `core.Climb`
(`core.zs:551-605`) takes and keeps the hold, decides which hand *may* (only an empty one; BD22Q's
free-fist layer by state **name** at `climb.zs:38-44`, so nothing links), measures the hand's step
along the rope **by step and not by place** (`core.zs:586-592`, with the measured reason in the
comment: taken from the place, a still hand on a swinging body drifted the length 6-14 units a swing),
caps it at `rsqgr_climb_cap` (`:593-596`), and holds the body from the hand so a stroke stops when the
arm does (`:601-604`). `HANDOFF.md` lists `climbscript.py` / `clsave.py` in the test kit. **Missing
only: both hands on the rope at once** — `hook.zs:53` stores a single `gripHand`. `01` §5.2b: make it a
pair and **average, never sum**, or frantic shaking becomes the optimal climb. **S.**

**Persistence — what owns it, what cleans it up, and the save/load answer.**

The good news is that the hard part is already written, because the mod already had to survive a save:

- **`WorldLoaded` rebuilds `hooks[]` from the `RSQGR_Hook` actors actually present**, via a
  `ThinkerIterator` (`core.zs:656-669`), and then destroys stray `RSQGR_Rope` / `RSQGR_Link` actors not
  referenced by a live hook (`:670-681`). **So a planted rope survives a save/load for free** — it is
  an actor in the level, and the level is what gets saved.
- **It correctly does not survive a map change.** `core.zs:653-654` says it plainly ("A level's actors
  die with it") and `WorldUnloaded` destroys every hook (`:684-692`). That matches Thief, where ropes
  are per-mission. **Do not change it.**

Three real gaps, and the third is the actual cost of this preset:

1. `core.zs:187` destroys a hook whose `owner != pmo`, and `:188` lets go when the player dies. A
   planted rope must survive both — two guards on a new `PH_PLANTED` phase.
2. `h.losBad` drops a latch after 3 failed line-of-sight checks (`core.zs:410`, `:518-522`). A planted
   rope you walk away from, or round a corner from, would be dropped. Exempt it while planted and
   unheld.
3. **`hooks[]` is indexed `pn*2 + hand`** (`core.zs:19`, `:58`), so a planted rope occupies that hand's
   slot and blocks the next throw. Thief lets you carry several ropes. **Planted ropes therefore need
   their own list, not the per-hand slot** — a `Array<RSQGR_Hook> planted` on the core, with its own
   rebuild in `WorldLoaded` and its own `Settle()` clear. That is the M in this preset, and it is the
   honest answer to "what owns it".

**Label: BUILDABLE / M.** Material gate **S**; two-handed climb **S**; persistence **M**.

---

### 9.4 BIONIC COMMANDO (NES)

**What the player does.** **Cannot jump.** The arm is the whole movement system: fire it diagonally
up, it catches, you retract and swing up and over. Every traversal is an arc you commit to before you
know whether it works. Momentum and arc shape are the entire game.

**Cvars.**

```
rsqgr_geo_mode 0; rsqgr_whip 0;
rsqgr_nojump 1;              // NEW: the defining rule
rsqgr_fire_snap 45;          // NEW: 0 off; 45 = quantise the fire pitch to the NES diagonal
rsqgr_reel_speed 20;         // RETRACT is the verb (from 12)
rsqgr_retract_speed 96;      // from 64
rsqgr_max_speed 50;          // from 30 -- 01 §6.4: the 30 cap is folklore
rsqgr_pump 0.35; rsqgr_drag 0.001; rsqgr_rope_friction 0.015;
rsqgr_hop 0;                 // no assist hop: the arc is everything
rsqgr_climb 0;               // or climb_cap 3 -- climbing is not the verb
rsqgr_min_len 24; rsqgr_range 1024
```

**This preset lives or dies on the momentum fix, and the coordinator's read is right.**
`player.keepmomentum` is **absent from the entire mod** — `grep -rn "keepmomentum\|NOFRICTION"` over
`zscript/` returns **zero hits**, re-verified in the 10-30 clone. So
`qzd173/wadsrc/static/zscript/actors/player/player.zs:1373-1378` multiplies `vel.XY` by `0.0001` on
**every grounded tic** (`vr_momentum` defaults false, `hw_vrmodes.cpp:120`). A game whose every verb is
a committed arc that lands and keeps going is **unplayable** until `RSQGR_Core.Swing` sets
`pmo.player.keepmomentum = true` every tic, with a coast window after `LetGo`. It clears itself below
`vr_momentum_threshold` (`player.zs:1384-1387`, default 1.0 at `hw_vrmodes.cpp:121`) — deliberate
anti-nausea design; respect it. Full argument in `01` §6, including why `bNOFRICTION` does **not** help
(the wipe's gate reads `GetFriction()`, `qzd173/src/playsim/p_map.cpp:675-802`, which never looks at
`MF8_NOFRICTION`) and why the wipe silently corrupts `core.zs:423-431`.

**No jump — and there is a clean script route.** `JumpZ` is a **plain ZScript field on `PlayerPawn`**
(`qzd173/wadsrc/static/zscript/actors/player/player.zs:35`, exposed as a property at `:68`, default 8
at `:120`, consumed at `:1535`) — not native, therefore **writable from script**. So
`pmo.JumpZ = 0` while the preset is on, restored from `GetDefaultByType(pmo.GetClass()).JumpZ`
when it is off. Per-player, no dmflags, no engine change.

**Do not** reach for `level.IsJumpingAllowed()` (`qzd173/wadsrc/static/zscript/doombase.zs:499`): it is
native and read-only from script, and it is a level/dmflags property that would affect every other mod.

Two notes that make this feel deliberate rather than broken:
- `player.zs:1520-1524` already makes a jump press while `crouchfactor < 0.75` merely un-crouch. In VR
  at `vr_crouch_use_button 0` that factor is driven by real head height
  (`gl_openvr.cpp:2512-2517`), so a physically crouching player already cannot jump. The preset is
  consistent with behaviour the engine already has.
- **`rsqgr_hop 0` is the authentic setting and carries a real risk.** The hop is the small lift given
  when you let go near a wall latch while reeling (`core.zs:132-138`). With no jump *and* no hop, a VR
  player can be stuck on a 16-unit step with no recourse. **Recommend `rsqgr_hop 0` for the feel and
  `2-3` as the accessibility fallback**, surfaced in the preset's own description rather than buried.

**`rsqgr_climb_cap`:** climbing is not the verb here; retract is. Either `rsqgr_climb 0` outright, or
cap at 3 so hand-over-hand exists but never competes with the reel.

**The fixed diagonal — worth offering, default off.** `rsqgr_fire_snap 45` quantises the fire pitch to
the nearest multiple of 45° at the fire site, where the direction is already being built from the hand:
`input.zs:52-53` reads `HandYaw` / `HandPitch`, and `input.zs:126` / `:131` turn them into a direction
through `RSQGR_Util.MapDir` (`util.zs:41-45`). One `round(pit / snap) * snap` before `MapDir`. It is
genuinely the thing that made the NES game feel the way it did — the arm went where the arm went, and
you planned around it. **But in VR, overriding where the player is physically pointing is an agency and
comfort risk**, so: offer it, default **0** (off), and let the preset turn it on.

**Label: BUILDABLE / S** for the mechanisms (`keepmomentum` one line, `JumpZ` two, the snap one),
**M for the tuning** — arc shape, reel speed and `max_speed` against comfort is where the days go.

---

## 10. SERIES FIT

**The whip's aim mode and the presets stay in the Grapple's own menu. They do not belong in
`RSQ_Accessibility`.** The line is clean and the mod already draws it.

`RSQ_Accessibility` carries settings about the player's **body or senses**, applying to every mod the
same way: handedness (`rsqa_onehand` → `RSQGR_A11y.Ctrl`, `a11y.zs:81-84`, consumed at `util.zs:54`,
`:65`, `:74`, `:79`), gesture tolerance (`rsqa_gesture_ease` → `Wide` / `Low`, `a11y.zs:46-48`),
photosensitivity (`:34-43`), comfort (`:53`), cue and text scale (`:57-61`), sound-cue substitution
(`:64-65`), hold-to-toggle (`:50`). Three consequences:

1. **`rsqgr_whip_aim`, `rsqgr_whip_style` and the presets are gameplay choices about one mod.** They
   have no meaning in the saber, the wheel or the shield. Grapple cvars on the Grapple's page, exactly
   as `rsqgr_monster_mode` and `rsqgr_geo_mode` are.
2. **But their *tolerances* belong to accessibility and must be routed there** — `rsqgr_whip_arc`
   through `RSQGR_A11y.Wide()` (§8.2), every speed and hold threshold in §3 through
   `RSQGR_A11y.Low()` (`a11y.zs:48`), every per-tic phase rate through `Deg()` (`:39`), exactly as
   `core.zs:566` already does for `rsqgr_grab_reach`. One player's "Snap To Arc" is then wider than
   another's, from the setting that exists for it.
3. **Handedness is already handled; do not re-solve it.** `RSQGR_Util.Ctrl` (`util.zs:49`) and
   `RSQGR_A11y.Ctrl` (`a11y.zs:81-84`) route a hand's *part* to whichever controller plays it under the
   Off-Hand Modifier. The four phases read one hand, so the **one-handed** path must read its hand
   through `Ctrl()` like everything else in `util.zs`, or one-handed play breaks for a player using the
   modifier.

**Nothing hard-links, and the guard is written.** `a11y.zs:27` resolves `RSQA_Service` only while a
class named `RSQA_Menu` exists, so a value left in the ini does nothing on its own. Same for
`RSQ_SlowMo` (`util.zs:90-100`), `RSQ_Score` (`service.zs:370-381`), `RSQ_DistanceGrab`
(`input.zs:149-150`), BD22Q (by class name via `FindStateByString`, `climb.zs:38-44`) and the Gun
Bonsai patch (`perks.zs`). **No new hard link, no new `#include` of another mod, no new class
reference** — including for the Bulletstorm/Score handoff, which goes through `Award()`'s existing
`ServiceIterator.Find("RS_ScoreService")` door.

**Service contract: additive only.** `service.zs:1-7` promises every unknown request answers −1 (or
`null`). Add only:

| New request | objectArg | Answer |
|---|---|---|
| `"whip.state"` | pawn, intArg hand | 0 idle, 1 laid, 2 lift, 3 snap, 4 flying, 5 recover; −1 no whip |
| `"whip.energy"` | pawn, intArg hand | 0..100 (`E`); −1 none |
| `"whip.aim"` | pawn | the live `rsqgr_whip_aim` |
| `"hung"` | monster | tics remaining aloft on a Leash hang; −1 |
| `"planted"` | pawn | count of persistent ropes this player owns on this map; −1 |
| `"whip.lock"` (GetObject) | pawn, intArg hand | the locked actor, or null |

And **do not touch** `"whipped"` (`service.zs:444`) or `"stunned"` (`:445-454`) — `RSQ_TopHPBars` reads
them, and `"stunned"`'s 35-tic `Pain`-state window depends on `whip.zs:131-136`'s forced pain surviving
unchanged. It does, in everything above. **Do not add a sixth `hook.phase` value** for planted or
hung: its domain (−1/0/1/2/3/4) is documented in `HANDOFF.md` and read by other mods. Express both as
new requests (`01` §8.3).

**`rsqgr_reset` must be updated — flagged explicitly.** `KEYCONF.txt`'s `rsqgr_reset` is one line
listing every cvar with its default, invoked by `SafeCommand "Reset All"` (`MENUDEF.txt:124`), and it
currently lists all six whip cvars. **Any cvar added here and not added there will silently survive a
"Reset All"**, leaving a half-reset whip and no clue why. The new ones:

```
rsqgr_whip_aim 1; rsqgr_whip_arc 28; rsqgr_whip_bend 18; rsqgr_whip_lock_hold 10;
rsqgr_whip_style 0; rsqgr_whip_life 16; rsqgr_whip_lay_tics 6;
rsqgr_yank_hang 35; rsqgr_rope_persist 0; rsqgr_rope_material 1;
rsqgr_nojump 0; rsqgr_fire_snap 0
```

`HANDOFF.md` ("For core") already warns the alias is assembled from both halves' lists and both must be
regenerated when either changes. **Treat the reset line as part of the feature.** Related: `MENUDEF.txt`
does **not** in fact list every cvar — `rsqgr_whip_above` is in CVARINFO and in `rsqgr_reset` with no
menu row — so "every cvar keeps its row" (`MENUDEF.txt:1-4`) is already aspirational; do not use the
menu as the inventory.

---

## 11. WHAT THE RIG CAN SETTLE BEFORE ANYONE SPENDS DAYS

The rig is better suited to this than to anything it has done yet:
`tools/rigtest/gr.sh <name> <secs> "<console script>" [1 = full load order]`, `build_probe.sh` (probe
handler + the generated GRTEST map: ledges, a 768-wide pit, ceiling 512, a 3D slab, a tagged alcove),
and probe events `grp_tp x y ang`, `grp_summon idx dist angofs`, `grp_dir`, `grp_ceil`, `grp_hurt`,
with `grp_log 0..4` (`HANDOFF.md`). Two are exactly the instruments needed:

- **`grp_dir`** already prints, per hand: `OverrideAttackPosDir`, yaw / pitch / **roll**, the mod's own
  computed direction, **the engine's own `AttackDir` for comparison**, and **`hand-off(x y z)` — the
  hand's offset from the pawn** (`tools/rigtest/probe/zscript.txt:23-32`, the `T_DIR` line). That
  offset plus that roll are the entire input to §3.
- **`grp_summon idx dist angofs`** places a monster at a given distance and angular offset — the
  targeting cone's test, by construction.

### Test A — does slack-as-gauge read at a glance? (half a day, do this first)

Purely visual, so it needs frames.

```
gr.sh gauge 20 "rsqgr_whip 1; rsqgr_whip_aim 0; rsqgr_debug 1;
               grp_tp 512 512 0; grp_log 2"
```

Drive the state machine to each of the five phases and **dump one left-eye frame at each**:
`WS_LAID`, `WS_LIFT` at `loadLift 0.5`, `WS_SNAP`, `WS_FLY` at `E = 1.0`, `WS_FLY` at `E = 0.1` (the
failed crack). Capture as `test/evidence/frames_gauge.png`, a 5-up matching the existing
`frames_spin.png` / `frames_flip.png` / `frames_voxel_barrel.png` convention.

**Pass:** someone shown the five frames in random order sorts them by "how loaded is this whip",
first try, without being told what to look for — **and picks the failed crack out as the odd one.**
That is the whole bar; if it needs explaining it has failed, because in play there is no explanation.
**If it fails:** raise `BOW_REST` toward 0.3 before touching `WAVE_AMP` — depth separates better than
wave amplitude at a glance.

Also capture `WS_FLY` at `rsqgr_whip_reach 192` and `48` at fixed `E = 0.5`: the bow is proportional to
`len`, so a short whip has a shallower absolute sag and `BOW_REST` may need reach-compensating. Cheap
now, annoying later.

### Test B — does a wound release land where a player expects? (one day)

```
gr.sh wind 40 "rsqgr_whip 1; rsqgr_whip_aim 0;        // mode 0: NO assist, clean reading
              rsqgr_whip_style 0; rsqgr_debug 1; grp_log 3;
              grp_tp 512 512 0;
              grp_summon 0 160 0; grp_summon 0 160 15; grp_summon 0 160 30; grp_summon 0 160 45"
```

Four imps at 160 units and 0 / 15 / 30 / 45° off the throw bearing. Pose the four phases, drive to a
known `E`, crack along 0°, and from the `T_GRW` lines record: `E`, `window`, `waveSpeed`, the tic the
loop reached the popper, the resolved `reach`, which imp was hit, and the angular error. Sweep
`E = 0.25 / 0.50 / 0.75 / 1.00`, ten cracks each.

**Three things it settles in an afternoon:**

1. **That the loop-arrival crack fires where §2.1 says it does** — the cheapest possible falsification
   of the whole design. Log `phase` and `u_loop = phase / 1.5` at the crack tic; it must read 1.0.
   If it does not, stop and fix that before tuning anything.
2. **The scatter of the angular error**, which is what chooses `rsqgr_whip_arc`: the cone should be
   roughly the 90th percentile of the measured error, not a guess. My 28° is reasoned from the 35 Hz
   sampling (§8.2); measure it and replace it.
3. **Whether a 3-tic loop reads as a snap.** Dump `frames_wind.png` across the lash's whole life at
   `E = 1.0`. Three tics is three frames of loop travel; if the crack is invisible at that speed,
   `WAVE_FAST` wants to come down and §3.5's range narrows.

**Honest caveat:** a cvar-posed hand is not a tracked hand. The rig can prove the phase machine is
arithmetically right, and can prove it does **not** charge from a still hand or from a menu close
(§3.7's bug — worth a dedicated run). It cannot tell anyone whether the snap window is *fun*. That is a
headset question and nothing on a desk will answer it.

### Test C — the regressions, and they are not optional

One run, the owner's full load order (`gr.sh ... 1`):

```
gr.sh whipreg 60 "rsqgr_whip 1; rsqgr_whip_snag 1; rsqgr_debug 1; grp_log 2;
                 grp_tp 1600 512 180; grp_ceil 0"      // the tagged alcove, for the lintel snag
```

| Assertion | Baseline to match (all already measured) |
|---|---|
| a lintel snag still reaches `core.Latch()` and swings | `HANDOFF.md`: "tip on the alcove lintel 91 u above the hand -> core's own Latch()" |
| `"whipped"` still reads 0 at the kill | `HANDOFF.md` (agent-svc): "whip kill `whipped` 0" |
| the forced Pain still breaks an attack (the disarm) | `HANDOFF.md`: "crack at tic 4: imp at 120 u, 12 damage, Pain entered" |
| save → load ×2 mid-lash loses no state, clears the lock | `core.zs:162-170` `Settle()` |
| no new script errors beyond `BD_VoxelPack`'s 4 `voxeldef` lines | every round's baseline in `HANDOFF.md` |

**And one for the presets, cheapest of all:** `rsqgr_nojump 1`, then press jump. `pmo.JumpZ` reads 0
and the player does not leave the floor; set it back and they do. Five minutes, and it de-risks §9.4's
defining rule before any tuning.

**Do Test A first.** Half a day, cheapest, and it tests the premise the owner supplied — that the droop
is worth keeping. If the gauge does not read, §§2-4 still stand on their own merits but §5 becomes
ordinary visual polish and the ordering of the plan changes.

---

## 12. WHERE THE MOD IS ALREADY RIGHT

`01` §9 lists fourteen for the mod as a whole. These seven are the ones this document could plausibly
have broken:

1. **The sag itself** (`whip.zs:213`, `:222`, `:223`). The owner is right and `01`'s first draft was
   wrong. It is the only thing in the file with weight, and it is now the gauge.
2. **`whip.zs:223` draws the correct phenomenon in the correct plane.** A loop travelling down the
   thong, applied along `side` (`:210-212`) — the belly/spine axis the reference names. The spatial
   frequency `1.5 * u` is what makes §2.1's loop position exactly computable. Whoever wrote that line
   had the right physical picture; only the trigger was wrong.
3. **`whip.zs:166-196` `Snag()` → `core.Latch()`.** A crack that resolves to geometry builds core's
   hook exactly as a claw would have, so swing, reel and release are core's unchanged code. The
   whip-to-rope handover is the tether thesis already implemented, and measured. **Do not touch it.**
4. **`whip.zs:131-136`'s forced Pain.** `FindState("Pain")` entered regardless of pain chance, with
   the comment already explaining that a Doom monster's attack is a state and not an item. It is the
   disarm, and the header comment at `whip.zs:1-8` already says so in plain words.
5. **`whip.zs:97-98` re-reads the hand every tic** so the lash follows the wrist ("a whip goes where
   your wrist sends it"). §3 adds magnitude and timing; this line already supplies direction,
   correctly, through `PoseGood`'s gate.
6. **`whip.zs:118` passes `TRACE_HitSky` in the `traceFlags` slot** — correct, a real `ETraceFlags`
   value `0x0040` (`qzd173/wadsrc/static/zscript/doombase.zs:256`,
   `qzd173/src/playsim/p_trace.h:102`), not a confusion with `ETraceResult`.
7. **`whip.zs:129`'s call order** — `RSQGR_Events.Whipped` before `RSQGR_Perks.Hurt`, which is what
   makes the service answers safe for a scorer reading `WorldThingDied` (`service.zs:54-58`).

And one worth naming separately: **`whip.zs:1-22`'s header comment** states the design, the tic
timeline and the no-edit-to-core constraint in plain words before any code. Every number this document
changes is named there. **Rewrite that comment with the feature** — leaving it describing a stopwatch
that no longer exists would cost the next session a day.

---

## 13. OPEN

1. **The snap window** (`TIP_FLIGHT ≈ 7` tics at 192 units, `sqrt(thong)` scaling). The shape is
   defensible; the constant is not measured. The single biggest feel unknown, and Test B cannot settle
   it (§11).
2. **`W_SNAP` 0.7 / `W_FLICK` 0.3.** From the reference's "the snap is the biggest contributor", not
   from a measurement.
3. **`rsqgr_whip_arc` 28°.** Derived from the 35 Hz sampling; Test B measures the real scatter.
4. **Whether `rsqgr_whip_style 0` (one-handed) really should be the default.** My recommendation and
   the reason are in §6; it is the owner's call and he may simply prefer Castlevania.
5. **Whether a cold crack should be possible at all**, or whether below `FAIL_E` the bind should fire
   the claw instead. The latter is tidier, the former teaches. I lean to allowing the flop.
6. **Thief's wood list.** Which Doom 2 texture names count is a content decision, not a code one, and
   it wants the owner's eye on a real map. It belongs in a data lump so it can be extended without a
   rebuild.
7. **Bulletstorm hang vs. the owner's comfort.** An enemy held aloft for a second with
   `RSQ_SlowMo` running is a lot of simultaneous slow; `RSQGR_Core.Comfort` (`core.zs:200-217`) exists
   but is currently only driven for the *player's* own rope states.
8. **Could not find** any existing arc-search or target-picker helper in the mod to build §8.2 on
   beyond `Throwable` (`monster.zs:199-208`) and the `TraceCallback` predicate (`hook.zs:396-407`).
   The cone search is new code — a reuse of policy, not of implementation.
9. **Could not find** any per-actor request on `RSQSM_Service` in this mod's usage
   (`util.zs:90-100` queries globally with a null `objectArg`). Whether RSQ_SlowMo *could* answer a
   per-actor scale is a question for its own source, which is not in this tree. §9.1 argues the Leash
   does not need it either way.
