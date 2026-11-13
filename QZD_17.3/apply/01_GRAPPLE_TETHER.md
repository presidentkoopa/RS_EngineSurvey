# 01 APPLY — RSQ_Grapple → ONE TETHER

**Mod read:** `Grapple/` in `github.com/presidentkoopa/RSQ_UBQSeries`, **HEAD `6a5c98c`, 2026-10-30**.
~100 KB of ZScript, read in full: `core.zs`, `monster.zs`, `whip.zs`, `hook.zs`, `climb.zs`,
`input.zs`, `util.zs`, `service.zs`, plus CVARINFO / MENUDEF / MAPINFO / KEYCONF / HANDOFF.
Every `file:line` below resolves in that tree.
**Engine checked against:** `qzd173` (tag `qzd17.3`, emawind84/gzdoom). Every engine line below was
opened; nothing is quoted from the surveys without re-reading it.

> **Source note.** This document was first drafted against a 2026-10-28 snapshot unpacked from the
> repo's `src/*.zip`, then re-verified line by line against the 10-30 clone above. The two are
> **byte-identical modulo line endings** (the clone is CRLF, the zips were LF), so every line number
> is unchanged — confirmed by a normalised diff over all twelve `zscript/rsqgr/*.zs` files and over
> CVARINFO / MAPINFO / MENUDEF / KEYCONF / HANDOFF / ROADMAP / zscript.txt, all **SAME**. HEAD's three
> newest commits are RSQ_Wheel work and touch nothing under `Grapple/`. The load-bearing citations
> were additionally re-read individually in the clone: `whip.zs:26-27`, `:99`, `:102`, `:118`,
> `:193`, `:204-206`, `:213`, `:222`, `:223`; `core.zs:162-170`, `:289-293`, `:317-321`, `:338-339`,
> `:353-358`, `:423-431`, `:445-457`; `util.zs:21`, `:43`, `:51-60`, `:74`, `:79`, `:104`;
> `input.zs:91-106`; `monster.zs:30`, `:68`, `:199-208`, `:262-268`, `:362-366`, `:419-420`;
> `hook.zs:53`, `:112-115`, `:390-392`; `climb.zs:34`; `service.zs:427-432`, `:444-445`. The two
> negative findings were re-run in the clone as well: `grep -rn "keepmomentum\|NOFRICTION"` over
> `zscript/` returns **zero hits**, and there is **no `Weapon` subclass**.
**Scope:** ZScript and data only. No engine change is proposed anywhere in this document.

---

## 0. THE RECOMMENDATION, FIRST

**Do the rework. Accept the one-tether thesis — the code supports it more strongly than the thesis
claims. And do these three things in this order, because the order is the whole argument:**

1. **Set `player.keepmomentum = true` every tic in `RSQGR_Core.Swing`.** One line. It is the single
   highest-value change available in the mod, it is why the owner says "Momentum", and it is a
   prerequisite for every other item here feeling like anything. **The mod never sets it** — grep over
   all of `zscript/` returns zero hits — so `MovePlayer` multiplies the player's horizontal velocity
   by `0.0001` on **every grounded tic** (`qzd173/wadsrc/static/zscript/actors/player/player.zs:1373-1378`,
   cvar `vr_momentum` default **false** at
   `qzd173/src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:120`). The measured 445-unit Indiana
   Jones leap in `HANDOFF.md` lands and **stops dead**. Details and a second, nastier consequence in §6.
2. **Replace the whip's fixed 4-tic timer with a motion-driven lash** (§2), resolved by swept-chord
   tracing (§3). This is the "hook / whip / rope thing" the owner is asking for; the sag (§1) is a
   symptom of the timer, not a separate bug.
3. **Add a third length policy, "hold", to the monster side** (§4). It is the lasso, and `monster.zs`
   already contains roughly four fifths of it.

**Do NOT restructure `core.zs`.** The spine is right (§9). The rework is a mode matrix bolted on top
of it plus three one-line edits inside it.

### The tether verdict

**ACCEPTED, with one correction and one addition.**

The thesis says five features are one tether with a four-axis mode matrix. The code agrees, and it
agrees *more* than the thesis does, because two of the four axes **already exist in the shipped code
under other names**:

| Thesis axis | What it already is in this mod |
|---|---|
| anchor type (geometry / actor / none) | `RSQGR_Hook.anchorPlane` 0/1/2 plus `h.victim` + `h.victimOfs` (`hook.zs:33-48`). Geometry and actor anchors are already two resolved branches of one `Fly()` (`core.zs:240-287`), and `UpdateAnchor()` (`core.zs:608-638`) already re-derives a *moving* anchor every tic for a heavy monster. |
| length policy (fixed / reeling / free) | `h.ropeLen` with three live writers: reeling (`core.zs:390-394`), free-pay-out while grounded (`core.zs:403-407`), and hand-driven (`core.zs:597-600`). **Three policies, already switched per tic.** |
| tension policy (rigid / elastic / slack) | `h.slack` (`hook.zs:41`, set `core.zs:401`) and the taut test `d >= h.ropeLen - 0.5` (`core.zs:434`). Rigid and slack exist; **elastic does not** — there is no `BOUNCE` term. |
| hand count (one / two) | **Does not exist.** Every path is `hand = i & 1`, one hand per hook, two independent hooks. §5. |

**The correction.** "Whip = no anchor" is wrong as stated, and the shipped code already knows better:
`whip.zs:145-153` → `Snag()` → `core.Latch()` means the whip's crack **can** resolve to a geometry
anchor and hand the result to core's rope. The whip is not "no anchor"; it is **anchor resolved at
full extension, with a very short life if nothing resolves**. That distinction matters for the
matrix: the whip is not a fifth mode, it is the *hook* with a 192-unit range, an 8-tic life, and
damage on the resolving trace. `whip.zs:166-196` is already written as "build core's hook as a claw
that hit this spot would have been". Keep that sentence; it is the thesis, in the code, already.

**The addition.** A fifth axis the thesis omits and the code demands: **who the constraint moves** —
player / anchor / split. Today it is a binary decided once by mass (`TooHeavy`, `core.zs:317-321`):
either the monster comes to you entirely (`PH_BITE`) or you go to it entirely (`PH_LATCH` with
`heavy`). The lasso, the two-handed whip and Rocket-Jockey-style slinging are all points on the
continuum between those two, and the continuum is one mass-split line (§4.3).

So: **one tether, five axes.** The mode names (hook / whip / lasso / rope / two-hand) become presets
over the axes, which is what `MENUDEF.txt`'s `OptionString` preset rows already do for feel.

---

## 1. THE SAG — settled, with the line that draws it

> **SUPERSEDED, 2026-10-30 — see `apply/08_WHIP_DESIGN.md`.** The diagnosis below stands, but the owner says the droop is what made him see a whip in the first place: `08` keeps the slack as a charge gauge, fires the crack when the drawn loop reaches the popper (`whip.zs:99` fires it at `phase = 1.0`, two thirds of the way down a wave whose `1.5·u` term means the loop lands at `phase = 1.5`), and deletes only the constants — **do not delete the sag.**

**Verdict: the owner's first candidate is the right half of the answer, but the cause is not
`BACK_TICS` on its own. The sag is DRAWN, deliberately, by four lines in `Draw`. It is not a physics
artefact and it is not the bow-wave failing to clear.**

### The timeline, from the code

`whip.zs:26-27` — `OUT_TICS = 4`, `BACK_TICS = 4`. `WorldTick` (`whip.zs:87-104`) advances `wage[i]`
one per tic, cracks at `wage == OUT_TICS` (`:99`), and halts when `++wage > OUT_TICS + BACK_TICS`
(`:102`). So one press produces **8 tics / 229 ms of drawn lash**, of which exactly one tic is the
crack, and 4 tics (114 ms) come **after** it.

### The four lines that draw a sag

```
whip.zs:204-206   if (age <= OUT_TICS) ext = age / OUT_TICS;  else ext = 1 - (age-OUT_TICS)/BACK_TICS;
whip.zs:213       double bow = (age <= OUT_TICS) ? len * 0.25 * (1.0 - age/OUT_TICS) : len * 0.08;
whip.zs:222       pb.z += bow * sin(180.0 * u) * (age <= OUT_TICS ? 1.0 : -1.0);
whip.zs:223       pb += side * (len * 0.06 * u * sin(360.0 * (1.5 * u - phase)));
```

Read them together:

- **`:213`, second branch.** Before the crack the bow decays correctly to zero (`1.0 - age/OUT_TICS`
  reaches 0 at `age == OUT_TICS`) — so **the owner's second candidate is wrong: the bow-wave does
  straighten exactly at the crack.** But after the crack the bow is reassigned to a *constant*
  `len * 0.08`. It does not decay at all over the four retract tics.
- **`:222`, the sign flip.** `(age <= OUT_TICS ? 1.0 : -1.0)`. Before the crack the bow is **up**;
  after it, the same 8 % is applied **downward**. This line is literally a sag: a half-sine, 8 % of
  the span deep, hanging below the straight line, for 114 ms.
- **`:223`.** `phase = age / OUT_TICS` (`:214`) keeps climbing past 1.0 — it reads 1.25, 1.50, 1.75,
  2.00 on the four retract tics. So the lateral travelling wave **keeps rolling down a rope that has
  already cracked**, while the rope is bowed downward. A snaking, drooping rope.
- **`:204-206`.** This is the owner's `BACK_TICS = 4`, and it is the *duration* of all of the above.
  It is a necessary condition — with `BACK_TICS` at 1 there would be no time to see the droop — but
  it is not the thing that looks like sag. Shortening it alone would make the lash vanish abruptly
  while still drooping for one tic.

**So: `whip.zs:213` + `:222` draw the sag; `whip.zs:27` + `:204-206` give it 114 ms to be seen in.**

### And the half the owner has not named

There is a second softness, upstream of the sag and probably the larger part of "almost there":
**the crack happens 4 tics — 114 ms — after the button press**, at a reach fixed by
`rsqgr_whip_reach` (`whip.zs:78`, `:99`). For 114 ms the player sees a bowed, waving rope extending
at a constant rate that has nothing to do with how fast they moved their arm. `HANDOFF.md`'s own
open question is exactly this: *"Whip: whether a button-cracked lash along the controller reads as a
whip at all."* The answer is no, and §2 is why.

### The runtime test that separates the two, with no rebuild

Both timings are `const`, so they cannot be probed from the console. But the **amplitudes** are both
proportional to `len`, which is `rsqgr_whip_reach` — a live cvar (`CVARINFO.txt`, `whip.zs:78`).

```
rsqgr_whip 1; rsqgr_debug 1; rsqgr_whip_reach 192     ; // crack at a wall ~4 m away, a dozen times
rsqgr_whip_reach 48                                   ; // same wall, same dozen cracks
```

The timing is identical in both runs (8 tics; the `T_GRW … start` / `T_GRW … CRACK` lines confirm it).
Only the bow (`len * 0.08`) and the wave (`len * 0.06`) shrink, by 4×.

- **Droop gone at reach 48, mushiness unchanged** → the sag is the `Draw` geometry (`:213`/`:222`).
  Fix: make the post-crack bow decay to 0 over `BACK_TICS` and freeze `phase` at 1.0.
- **Feels identical in both runs** → it is the timing, and only §2 fixes it.

Expect the first result for "sags", the second for "almost there". They are independent and both
want fixing.

**Label: the fix is EXISTS/S** (four lines in one function, no new concepts). **The timing fix is §2.**

---

## 2. MOTION-DRIVEN CRACKING

**Today:** bind press → `rsqgr_fire` net event → `RSQGR_Whip.NetworkProcess` (`whip.zs:52-71`) takes
back the claw core just fired, `Start()` (`:73-85`) sets `wreach` from a cvar and `wage = 0`,
`WorldTick` cracks at tic 4. Arm speed contributes **nothing**: `wdir` is re-read from the hand each
tic (`whip.zs:97-98`) so the lash *aims* where the wrist points, but reach, timing and damage are
constants.

**There are no velocity fields** (`00_CAPABILITIES` §1.3; the complete VR surface is
`qzd173/wadsrc/static/zscript/actors/actor.zs:268-277` and it contains no velocity). So speed is
differenced from stored positions. All four pieces below are BUILDABLE.

### 2.1 The sampler — reuse what the mod already has (EXISTS / S)

**Do not write a new sampler in `WorldTick`.** The mod already streams a per-frame hand pose, and it
is in the right place for the right reason:

```
input.zs:91-106   RenderOverlay(RenderEvent e)  ->  SendNetworkEvent("rsqgr_pose", hand, Pack(off))
input.zs:141-146  NetworkProcess                ->  core.poseOff[i], core.poseTic[i]
climb.zs:48-56    HandOffset()                  ->  (ok, off), 3-tic freshness window
```

This is the only code in any of these surveys that reads the hand **per rendered frame** rather than
per tic, and the only code that is multiplayer-correct (`05_GRAPPLE` §5.3 calls MP a WALL for
hand-driven mechanics; this mod solved it — see §9.3). Extend it, do not replace it:

- Gate it on `grabHeld` **or** a lash being armed (today it is `grabHeld` only, `input.zs:101`).
- On the receiving side keep a ring of the last 8 samples per hand instead of one
  (`core.poseOff[i]` → `core.poseRing[i][8]`, with `core.poseTic`). 8 tics ≈ 229 ms, which is
  `01_MELEE` §3.2's budget and is generous for a flick.
- **The height trick at `input.zs:103` must survive verbatim.** `(h.z - e.ViewPos.z) + p.viewheight`
  — the hand's height measured against *that frame's own eye*, not `pmo.Pos` and not `player.viewz`.
  The comment at `input.zs:79-90` records a measured 6-14 unit per-stroke wobble from getting this
  wrong. A speed signal differenced from a wobbling Z is pure noise, so this line is load-bearing for
  §2 in a way it was not for climbing.

**Add the spike filter the mod is missing.** `RSQGR_Util.PoseGood` (`util.zs:51-60`) gates on
`OverrideAttackPosDir` (correct, §9.1) and rejects a hand further than `ARM_MAX = 96` from the body
— but it does **not** catch the engine's own collapse. On any frame where the device update does not
run, `VRMode::SetUp` leaves both hands at the body centre
(`qzd173/src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:438-446`), and the device code only runs
inside `if (gamestate == GS_LEVEL && menuactive == MENU_Off && !paused)`
(`qzd173/src/rendering/gl/stereo3d/gl_openvr.cpp:1704`). A collapsed pose sits ~41 units above the
pawn, well inside `ARM_MAX`, so `PoseGood` returns **true** for it. This is already a shipped bug,
already in `HANDOFF.md`: *"A press in the first frame after a menu closes fires from the body."* Two
lines fix it and the fix is a prerequisite for speed detection:

```zscript
if ((pmo.AttackPos - pmo.OffhandPos).Length() < 0.01) return false;   // both hands collapsed
if ((now - prev).Length() > SPIKE_REJECT) { invalidate; return false; }  // SPIKE_REJECT ~40 u/tic
```

**Label: BUILDABLE / S.**

### 2.2 Units — get this right or the overhead lash is wrong by 17 % (EXISTS / S)

`vr_vunits_per_meter` is **34.0** (`hw_vrmodes.cpp:94`), 35 tics/s, so **1 map unit per tic ≈
1.03 m/s horizontally**. Vertically the hand transform divides the up axis by `pixelstretch`
(`gl_openvr.cpp:1282-1292`), default **1.2** (`qzd173/src/gamedata/g_mapinfo.cpp:314`), so a vertical
metre is only 28.33 units and an unnormalised height reads ~17 % short.

**The mod already knows this.** `RSQGR_Util.MapDir` (`util.zs:41-45`) carries `sin(pitchUp) /
PIXELSTRETCH` and `RSQGR_Util.Metres` (`util.zs:102-106`) multiplies Z by it before dividing by 34.
`HANDOFF.md` records the direction verified equal to the engine's own `AttackDir` at five poses.
**This is already right and must not be "fixed".**

One change only: `util.zs:21` hardcodes `const PIXELSTRETCH = 1.2`. It is script-readable as
`level.pixelstretch` (`qzd173/wadsrc/static/zscript/doombase.zs:479`, also on `LevelInfo` at `:366`)
and a MAPINFO may set it (`g_mapinfo.cpp:1566`). Read the live value, fall back to 1.2. **An overhead
lash is the one gesture where this bites**, because its signal is almost entirely vertical.

Normalise once, at the sampler: `s.Z *= level.pixelstretch` on the way in, and invert only when
handing a Z back to the engine. Every threshold below is in post-normalisation units.

**Label: EXISTS (nearly) / S.**

### 2.3 Lash signature detection (BUILDABLE / M)

A whip crack is not "the hand moved fast". It is **acceleration out, then a sharp deceleration or
reversal** — the arm throws the lash and then stops, and the stored energy runs down the rope. At
35 Hz a real flick is 3-5 samples, which is enough for exactly that three-point test and nothing
more subtle.

```zscript
// post-normalisation, map units per tic; handRing[0] = newest
Vector3 v0 = handRing[0] - handRing[1];      // this tic
Vector3 v1 = handRing[1] - handRing[2];      // last tic
Vector3 v2 = handRing[2] - handRing[3];
double  s0 = v0.Length(), s1 = v1.Length(), s2 = v2.Length();

bool accelOut = s1 > CRACK_SPEED && s1 > s2 * 1.25;        // it was building
bool brake    = s0 < s1 * 0.55                              // sharp decel
             || (v0 dot v1) < 0;                            // or an outright reversal
```

Fire the crack on `armed && accelOut && brake`, with the **peak** sample `s1` (not `s0`) as the drive.
`CRACK_SPEED` wants to be a cvar in m/s and converted: **6 m/s ≈ 5.8 u/tic** is a deliberate flick,
**9 m/s ≈ 8.7 u/tic** is a hard one. Start the threshold at 6 m/s.

Direction: take the lash axis from the **peak** sample's displacement `v1.Unit()` blended with the
live wrist direction `RSQGR_Util.MapDir(HandYaw, HandPitch)` (`whip.zs:98`). Pure wrist direction
loses the throw; pure displacement makes a sideways flick lash sideways even when the wrist points
forward. 0.6 displacement / 0.4 wrist reads correctly, and it is a cvar.

**Anti-waggle: hold-to-arm.** This is the important half and it is cheap, because the mod already
has the binds. `KEYCONF.txt` gives `+rsqgr_main` / `+rsqgr_off` a real press/release pair, and
`RSQGR_Core` already tracks `held[i]` and `pressTic[i]` (`core.zs:20-21`, `:59-60`) with an 8-tic tap
window (`TAP_TICS`, `core.zs:17`). So:

- **Press and release inside `TAP_TICS`** → today's behaviour: fire the claw. Unchanged.
- **Press and hold** → the lash arms. A crack can fire only while armed. Release disarms.
- No crack can fire within `COOLDOWN` tics of the last one. `05_GRAPPLE` §4.3 recommends ~6 tics;
  with the brake test above, the recovery stroke is already rejected (it reverses, so `accelOut`
  fails on it), so 4 tics is enough. Keep it a cvar.

Holding a button to arm a gesture is also what kills the whole class of false positives that makes
gesture detection miserable in VR — walking, reloading, gesturing at a teammate. The engine offers
four free user bits (`BT_USER1..4`, `qzd173/wadsrc/static/zscript/constants.zs:859-862`) but
`input.zs:3-4` records that Brutal Doom Quest uses all four, which is why the mod rides KEYCONF
aliases. **That decision is already right; build on the existing binds, do not take a BT bit.**

**Label: BUILDABLE / M.**

### 2.4 Reach and damage from peak hand speed (BUILDABLE / S)

```zscript
double k = clamp((peak_u_per_tic * 1.03 - CRACK_MS) / (CRACK_MS_FULL - CRACK_MS), 0.0, 1.0);
reach  = lerp(MIN_REACH, rsqgr_whip_reach, k);     // rsqgr_whip_reach becomes the CEILING
damage = max(1, int(rsqgr_whip_damage * (0.25 + 0.75 * k)));
```

With `CRACK_MS = 6` m/s and `CRACK_MS_FULL = 14` m/s, a lazy wave reaches ~40 % and does a quarter
damage; a committed throw reaches the full 192 and the full 12. Note the pleasing consequence: the
reach is no longer the thing that decides whether you hit, so the owner's "lasso shit" — a long,
deliberate throw — and a short defensive flick become different moves with the same bind.

`rsqgr_whip_reach` and `rsqgr_whip_damage` keep their names and become ceilings. Nobody's ini breaks.

**Label: BUILDABLE / S.**

### 2.5 What the lash looks like while it travels

The `Draw` function (`whip.zs:200-243`) stays, with three changes, and it becomes the *consequence*
of the signal rather than a timer:

- Extension is driven by the measured throw, not `age / OUT_TICS`.
- The post-crack bow decays to zero over the retract (fix `:213`) and `phase` freezes at 1.0 (fix
  `:223`) — §1.
- The bow *amplitude* during the throw scales with `1 - k`: a hard crack is a nearly straight line,
  a soft one visibly loops. That is the whole read of a whip, for free, from a number you now have.

---

## 3. SWEPT-CHORD TRACING

**The problem is worse here than for a blade, and the mod's current trace cannot see it.** The crack
today is a single `tr.Trace(org, …, dir, wreach[i], …)` (`whip.zs:118`) fired on exactly one tic — a
straight ray from the hand at the instant of the crack. It samples the lash's *final* line and nothing
the lash swept through on the way.

Quantify it. A lash tip at 192 units from the wrist, with the wrist rotating 60° in one tic, bulges
`192 × (1 − cos 30°) ≈ 25.7` units past the chord between samples. An imp is radius 20 — a 40×40
**square** footprint, because Doom's actor test is Chebyshev, not a circle
(`qzd173/src/playsim/p_map.cpp:1417-1419`, trace path `qzd173/src/playsim/p_trace.cpp:695-696`,
neither reads `Angles.Yaw`). So the bulge is larger than the target. A fast overhead lash will sweep
clean past a monster and report a miss.

### The fix, applied (BUILDABLE / M)

`01_MELEE` §3.5's recipe, with one adaptation: for a whip the fast-moving thing is the **tip**, and
the tip's motion is dominated by *rotation*, not translation, so the sub-samples must interpolate the
**angles**, not just the position.

```zscript
// TipAt(t): lerp the hand position linearly, lerp (yaw, pitch) along the SHORTEST arc,
// rebuild the tip at reach(t). Angle wrap: a0 + Actor.DeltaAngle(a0, a1) * t  -- the mod
// already uses Actor.DeltaAngle at monster.zs:279, so the idiom is in-house.
const SUBS = 3;                  // 3 below ~24 u/tic of TIP speed; 5 above
Vector3 prevTip = TipAt(0.0);
Vector3 prevRoot = RootAt(0.0);
for (int i = 1; i <= SUBS; i++)
{
    Vector3 tip  = TipAt(double(i) / SUBS);
    Vector3 root = RootAt(double(i) / SUBS);
    if (Sweep(prevTip, tip)) { resolve; break; }      // what the TIP swept across
    if (Sweep(root, tip))    { resolve; break; }      // something standing IN the lash
    prevTip = tip; prevRoot = root;
}
```

**No new primitive is needed.** `RSQGR_Tracer` (`hook.zs:390-411`) is already a `LineTracer` subclass
taking an arbitrary start and direction (`doombase.zs:310`), with a `TraceCallback` that already
rejects players, the mod's own rope actors, missiles and non-shootables (`hook.zs:396-407`). That is
precisely what `TRF_ABSPOSITION` would have bought, and it is already shipped and already tested.
Reuse it; one `Trace` call per sub-chord. See §9.11.

### Cost

**2 traces per sub-step × SUBS = 6 traces per tic per lashing hand; 12 with both hands; 10 / 20 at
SUBS = 5.** It runs only while a lash is live (~8 tics per crack), not continuously.

For scale: `01_MELEE` §3.4 calls 210 traces/second "nothing", and a `LineTracer` call is the same
order as a hitscan bullet — one `FPathTraverse`. The mod's own measured budget (`HANDOFF.md`, full
load order, two hooks) is **core `WorldTick` mean 0.041 ms, max 0.098 ms**, and the existing crack
already spends one trace. 12 traces on a lash tic is comfortably inside that envelope. Compare the
alternative: a hitbox actor per sub-step would be 70 spawn/destroy per second and would pollute
every other mod's `ThinkerIterator`.

One guard worth adding, because the collision box is square: after a hit, confirm with
`Distance3D(HitActor)` against `victim.radius + fatten` and reject beyond it, or a trace that grazes
a corner registers as a clean body hit from 1.41× the intended reach.

**Label: BUILDABLE / M.**

---

## 4. LASSO — how much of `monster.zs` is already one

**Verdict: roughly four fifths. Everything except the length policy. And the hardest part — the
monster's own state while held — is not only done, it is done *better* than either survey
recommends.**

### 4.1 What is already a lasso (EXISTS)

| Lasso part | Where it already is |
|---|---|
| **Attach to an actor, at a point on its body** | `core.zs:240-248`: `h.victim = v`, `h.victimOfs` clamped to `v.radius` / `v.Height`. A real body-relative attach point, not a centre. |
| **The anchor follows the body** | `hook.zs:136-139` and `core.zs:610-614` (`UpdateAnchor`) re-derive `h.anchor = victim.Pos + victimOfs` every tic. |
| **Drag the thing** | Two complete implementations: `Slide` (`monster.zs:63-91`, `TryMove` along the floor) and `RSQGR_Flight` (`monster.zs:150-652`, an authored arc placed by `PlaceMid`). |
| **The monster's own state while held** | `monster.zs:262-268` puts it in its `Pain` state and pins `tics = -1`; `WorldTick` re-pins it every tic (`monster.zs:419-420`) so a fresh hit cannot wake it mid-hold; `Restore` (`monster.zs:600-616`) hands back every borrowed flag and sets `tics = 1`. §4.4 — this is the part to be proud of. |
| **Release** | `OnRelease` → `Cut` (`monster.zs:301-307`) hands it to free flight at the path's own velocity; `LetGo` (`core.zs:124-142`) is the player-side half. |
| **Collision-correct placement at speed** | `PlaceMid` (`monster.zs:362-390`): `SetZ` clamped to floor/ceiling, vertical move *refused* if `TestMobjLocation` fails, then `TryMove`, then three 6-unit lift-outs. `05_GRAPPLE` §4.6 names unvalidated positional correction as "the main implementation risk in this design"; this mod solved it. |
| **Damage from speed, both ways** | `Impact` / `Damage` / `Land` (`monster.zs:536-589`). |
| **Mass / size gating** | `Throwable` (`monster.zs:199-208`) and `TooHeavy` (`core.zs:317-321`), re-asked every tic (`core.zs:332`) after a measured Champions bug. |
| **Save / load mid-hold** | All state in handler fields; `HANDOFF.md` measured a BullDemon loaded twice mid-throw landing in the same spot. |
| **A one-sided rope-to-actor constraint, already running** | `core.zs:253-257`: a heavy monster becomes `h.anchor` and goes to `PH_LATCH`, after which `Swing` constrains **the player** to a moving actor. The maths of a tether between a player and a monster is shipped — with the signs the other way round. |

### 4.2 What is missing (BUILDABLE / M)

**One thing, and it is the length policy: there is no "hold at L".** Both drag modes have a
*destination* — `Slide` stops at `pmo.radius + victim.radius + rsqgr_slide_gap` (`monster.zs:68`),
`Flight` lands at `LandMid` in front of you (`monster.zs:211-225`). A lasso holds the thing at
whatever radius the rope currently is, wherever it happens to be, for as long as you want.

And core actively fights it:

```
core.zs:338-339   if (d < h.bestDist - 0.25) { h.bestDist = d; h.noProgress = 0; }
                  else if (++h.noProgress > 52) { LetGo(h, false); return; }
```

**A held monster is, by definition, making no progress, so core drops the rope after 52 tics
(1.5 s).** `HANDOFF.md` already names this exactly: *"Carry/hold (monster dangling on the rope):
core's 52-tic no-progress guard would drop it; would need a seam change."* The last session was
right and the seam change is two lines.

So the lasso is:

1. **`rsqgr_monster_mode 2 = Hold`** alongside 0 Slide / 1 Yank. The mode already exists as a cvar
   with a dispatch point (`monster.zs:30`, `:35-44`); add a third branch.
2. **The length policy, in `RSQGR_Monster.Tick`'s new Hold branch.** Reuse `h.ropeLen` with the
   existing reel writer (`core.zs:390-394`) so the same held bind that reels a wall-rope reels in a
   lassoed monster. Per tic: project out the separating radial component in the *monster's* frame,
   then place by `PlaceMid` — never by `Vel`, for the reason `monster.zs:96-111` documents at length
   (RSQ_SlowMo's `Impulses` mis-scales a push given from the floor; measured 16× overshoot).
3. **The seam change.** `core.zs:338-339` must not fire while the hook's victim is in a hold. Cleanest
   shape that keeps the service contract intact (§8): give `RSQGR_Hook` a `bool holding`, set by the
   monster side, and read it in `Bite`:
   `else if (!h.holding && ++h.noProgress > 52) { LetGo(h, false); return; }`
   Keep the out-of-range guard (`core.zs:337`) — a lasso should snap if you walk away.
4. **Steering.** With the thing held at a radius, swinging it is the player turning: the monster's
   held position is `pmo.Pos + dir * ropeLen`, and `dir` can follow the hand (`HandPos`/`MapDir`,
   already available) instead of the original bite direction. That is "lasso shit": a monster on the
   end of a rope that goes where your arm goes, and strikes what it hits via `Impact`
   (`monster.zs:536-550`, already wired to `RSQGR_Events.Struck` and the Bowling perk).

### 4.3 The mass split — the fifth axis, one block of maths

Today `TooHeavy` (`core.zs:317-321`) is binary. Replace the *consequence*, not the test:

```zscript
double mA = v.Mass, mP = pmo.Mass;
double shareP = clamp(mA / (mA + mP), 0.0, 1.0);    // heavy anchor -> the player absorbs it
// correction applied to the player  * shareP   (by Vel, as core already does)
// correction applied to the monster * (1-shareP) (by PlaceMid, as monster.zs already does)
```

Keep `rsqgr_mass_max` as a **hard clamp of `shareP` to 1.0** (a pillar or a boss never moves), not as
the switch it is today. One continuum replaces two modes, bosses still behave, and nothing in the
service API changes.

### 4.4 Two engine gotchas, and the mod is already on the right side of both

The brief asks me to note `freezetics` and `+NOINTERACTION`. Both are real; **neither is what this
mod should use, and it already uses something better.**

- **`freezetics`** (`qzd173/wadsrc/static/zscript/actors/actor.zs:265`) early-returns at the top of
  `AActor::Tick` (`qzd173/src/playsim/p_mobj.cpp:3948-3952`) *before* any movement, and the actor
  **stays in the blockmap** — true, and that is gotcha 11's point. But it stops movement too, so a
  `freezetics` monster **cannot be dragged at all**. `05_GRAPPLE` §4.1b says this outright. It is
  wrong for a lasso.
- **`+NOINTERACTION`** forces `MF_NOBLOCKMAP` (`qzd173/src/playsim/p_mobj.cpp:3988-3992`), taking the
  actor out of the world — unshootable, uncollidable. Wrong for a lasso for the opposite reason.
- **What the mod does instead:** pins the monster in its own `Pain` state with `tics = -1`
  (`monster.zs:262-268`, re-pinned `:419-420`), clears `bNOBLOCKMONST` so the AI's monster-blocking
  lines do not apply to a thrown body (`monster.zs:260`), and borrows/restores `bNOGRAVITY` and the
  roll flags (`:255-257`, `:600-616`). The actor stays in the blockmap, stays shootable, stays
  movable, and nothing it was winding up fires in mid-air. **This is the correct third answer and it
  is better than either survey option. Do not replace it.**

**Label: BUILDABLE / M.** (`L` if the steering and the mass split land together with tuning.)

---

## 5. TWO-HANDED WHIP / ROPE

### 5.1 Exposure to the two-hand trap — checked, and the answer is good

**The mod is NOT exposed in the fatal way, and the reason is structural, not lucky.**

Gotcha 10a: with `vr_two_handed_weapons` true (default, `hw_vrmodes.cpp:119`), pressing the **offhand
grip** with the hands within **0.50 m** sets `weaponStabilised`
(`qzd173/src/rendering/gl/stereo3d/gl_openvr.cpp:1704-1727`, distance at `:1709-1714`, latch on the
rising edge at `:1718-1723`, cleared with no distance test at `:1724-1726`), after which the offhand
`Weapon` fires nothing, checks no buttons and renders nothing, and `WF_TWOHANDSTABILIZED` is
re-asserted on every `P_SetPsprite` so script cannot clear it.

Checked against the mod:

- **`grep -rn "^class" zscript/rsqgr/` — the mod defines no `Weapon` subclass at all.** Twenty-four
  classes: six `EventHandler`s (`RSQGR_Core`, `_Input`, `_Flight`, `_Whip`, `_Perks`, `_Events`), one
  `StaticEventHandler` (`_Voxels`), one `Service` (`_Service`), five `Actor`s (`_Hook`, `_Link`,
  `_Rope`, `_Streak`, `_WhipTip`), one `LineTracer` (`_Tracer`), three menu classes and seven static
  helper classes. **There is no offhand weapon for the latch to switch off.**
- **The only read of `OffhandWeapon` in the whole mod is `climb.zs:34`**, and it is asking
  "is this hand free" by class name. Stabilisation does not null `OffhandWeapon`, so that test reads
  the same answer either way. Safe.
- **Grab state comes from the mod's own KEYCONF aliases**, not the engine grip:
  `+rsqgr_grip_main` / `+rsqgr_grip_off` → `event rsqgr_grip` → `rsqgr_grab` net event → `core.Grab`
  (`KEYCONF.txt`, `input.zs:68-71`, `core.zs:536-547`). This is precisely the defence
  `00_CAPABILITIES` gotcha 10a recommends ("use no offhand weapon at all … take grab state from the
  trigger rather than the grip"), arrived at independently. **Already right.**

**One real exposure remains, and it matters for exactly this feature.** While stabilised, the engine
overwrites the **main hand's angles** with the hand-to-hand vector
(`gl_openvr.cpp:1739-1756`: `VectorSet(weaponangles, -atan(y/zxDist), -atan2(x,-z), roll)`).
`weaponoffset` — and therefore `AttackPos` — is written just above at `:1733-1738` and is **not**
touched. The mod's fire and lash direction both come from the angles:

```
util.zs:74   HandYaw   -> pmo.AttackAngle + 90
util.zs:79   HandPitch -> pmo.AttackPitch
whip.zs:98   wdir = MapDir(HandYaw, HandPitch)
input.zs:52  the packed direction in the rsqgr_fire event
```

So **any two-handed stance with the offhand grip held silently redirects the main hand's hook and
lash along the hand-to-hand line.** Today that is an arbitrary-looking bug waiting to happen. For a
two-handed whip it is the vector you actually want — but relying on it would make the feature depend
on a cvar and on an OpenVR-only code path (`weaponStabilised`'s only writes anywhere are
`gl_openvr.cpp:1723` and `:1726`; the OpenXR path never sets it, so on Quest it does not fire today
and is one patch from doing so).

### 5.2 The design, made immune by construction (BUILDABLE / M)

**Derive the handle axis from the two hand POSITIONS, never from `AttackAngle`.** Then the result is
bit-identical whether stabilisation is on, off, latched, or on a backend that has never heard of it.

```zscript
// post-normalisation (Z already * level.pixelstretch), both from the per-frame stream of §2.1
Vector3 fwd  = handRing[MAIN][0];           // the hand nearer the lash
Vector3 back = handRing[OFF][0];
Vector3 axis = (fwd - back);
double  span = axis.Length();
if (span < 0.01) return;                    // collapsed pose; §2.1's spike filter catches it anyway
axis = axis.Unit();
```

Three signals, all free:

1. **Aim** = `axis`. The lash leaves the forward hand along the two-hand line. Honest, stable, and
   it makes a long rope read as a rope rather than a wrist flick.
2. **Drive** = the **angular rate of `axis`** plus the forward hand's linear speed. A two-handed whip
   is cracked by *rotating the handle*, not by punching forward:
   `omega = acos(clamp(axis_now dot axis_prev, -1, 1))` degrees/tic. At 192 units of lash,
   `tipSpeed ≈ span_term + reach * omega * pi/180`. Feed `tipSpeed` to §2.3's signature test in place
   of the one-hand `peak`. A two-handed crack then genuinely out-reaches and out-damages a one-handed
   flick, because it physically does.
3. **Hand count** = whether both hands are armed. `core.held[i]` already tracks both
   (`core.zs:20`, `:59`), so "both binds held" is one `&&`.

**The two-handed rope (the climbing case) is already two-handed.** `RSQGR_Climb` + `core.Climb`
(`climb.zs:25-79`, `core.zs:551-605`) already takes the rope over between hands, measures each
hand's step along the rope (`core.zs:590-592`), and caps it (`rsqgr_climb_cap`). Hand-over-hand is
done. What two-handed adds there is **both hands on the rope at once** — and `core.zs` only stores a
single `h.gripHand` (`hook.zs:53`). Making it a pair is the whole change, with the survey's warning
heeded: **average the two hands' contributions, never sum them**, or frantic shaking becomes the
optimal climb.

**Do not add a BT bit, do not touch the engine grip.** The mod's alias binds are the correct input
surface and they are already immune.

**Label: BUILDABLE / M** for the two-handed lash; **S** for both hands on the rope.

---

## 6. MOMENTUM — the single highest-value change

### 6.1 The finding

**`keepmomentum` is never set anywhere in the mod.** Verified:
`grep -rn "keepmomentum\|NOFRICTION" zscript/` returns **zero** hits for either.

The wipe, re-read in full:

```
qzd173/wadsrc/static/zscript/actors/player/player.zs:1368   player.onground = (pos.z <= floorz + 2) || bOnMobj || ...
qzd173/wadsrc/static/zscript/actors/player/player.zs:1373-1378
    if (!vr_momentum && !player.keepmomentum && player.onground && friction == ORIG_FRICTION)
    { vel.XY *= 0.0001; Speed = Default.Speed * 8; }
```

`vr_momentum` defaults **false** (`hw_vrmodes.cpp:120`). `keepmomentum` is writable
(`player.zs:3108`, native `qzd173/src/playsim/d_player.h:414`) and the engine sets it itself. So:

**Every grounded tic, the player's horizontal velocity is multiplied by 0.0001.**

### 6.2 Three consequences, in order of how much they hurt

1. **Landing kills the swing.** `P_PlayerThink` runs at `qzd173/src/p_tick.cpp:166`, `WorldTick` at
   `:170`. Core's `Swing` early-outs the instant the player is grounded
   (`core.zs:382`, `:403-415` — `grounded && !reelHeld && !pullUp` → `continue`), so on the landing
   tic nothing restores what the wipe took. The 445-unit Indiana Jones leap measured in `HANDOFF.md`
   arrives at a ledge and **stops where it touches down**. This is the owner's "Momentum", stated as
   plainly as a one-word note can state it.
2. **`bNOFRICTION` does not save you, and it is worth knowing why.** The wipe's gate is
   `friction == ORIG_FRICTION`, and `friction` comes from `GetFriction()` → `P_GetFriction`
   (`qzd173/src/playsim/p_map.cpp:675-802`), which reads sector and terrain friction and **never
   looks at `MF8_NOFRICTION`**. `MF8_NOFRICTION` only short-circuits the separate friction pass at
   `qzd173/src/playsim/p_mobj.cpp:2442`. `05_GRAPPLE` §4's boilerplate prescribes both flags as if
   either would do; **only `keepmomentum` escapes the wipe.** Set `bNOFRICTION` as well, for the skid
   — but do not expect it to do this job.
3. **The wipe silently corrupts core's cleverest line.** `core.zs:423-431`:
   ```
   Vector3 moved = pmo.Pos - lastPos[pn];
   if (moved.Length() < 64) pmo.vel = moved + (pmo.vel - lastSet[pn]);
   ```
   This reconstructs "what he actually did last tic, plus whatever the engine and his input changed
   since" — a genuinely good idea (§9.7). But on any grounded tic that reaches it (reeling along the
   floor, or pulling up), the wipe has already made `pmo.vel ≈ 0`, so `(pmo.vel - lastSet)` is
   `≈ -lastSet` and **cancels the `moved` term**. The reconstruction reads "he moved, and then
   something took it all away", which is exactly true and exactly useless. A grounded zip therefore
   restarts from near zero every tic and can never exceed `rsqgr_reel_speed`. Setting `keepmomentum`
   fixes this line too, at no cost.

### 6.3 The change

```zscript
// RSQGR_Core.Swing, once per player per tic, UNCONDITIONALLY -- not only while taut
pmo.player.keepmomentum = true;
```

Set it whenever **any** hook is in `PH_LATCH` or `PH_BITE`, and keep setting it for a coast window
after `LetGo` (10-20 tics) so the flight survives the landing. It clears itself: `player.zs:1384-1387`
drops it once both `|vel.x|` and `|vel.y|` fall below `vr_momentum_threshold` (default 1.0,
`hw_vrmodes.cpp:121`). That auto-clear is deliberate anti-nausea design; respect it, do not fight it.

**Do not set `vr_momentum` globally.** It changes movement feel for every mod in the series and it is
the user's cvar.

### 6.4 The speed cap — rsqgr_max_speed 30 is folklore, and the folklore is wrong

`CVARINFO.txt`: `rsqgr_max_speed = 30`, enforced by `CapSpeed` (`core.zs:353-358`) on **the full 3D
velocity magnitude**. The 30 comes from `MAXMOVE`, and **the `MAXMOVE` clamp is conditional and
normally does not apply**:

```
qzd173/src/playsim/p_mobj.cpp:2096-2097
  if ((mo->player && (mo->Level->i_compatflags & COMPATF_WALLRUN))
   || ((mo->waterlevel >= 1 || (mo->player && mo->player->crouchfactor < 0.75)) && !(mo->flags8 & MF8_NOFRICTION)))
```

`compatmode` defaults to 0, so `COMPATF_WALLRUN` is off. Standing, dry, default compat → **no 30-unit
cap**. The unconditional ceiling is `VELOCITY_THRESHOLD = 5000` (`p_mobj.cpp:2107-2111`). And
`05_GRAPPLE` §2.3's claim that the clamp is applied unconditionally at `p_mobj.cpp:2100-2101` is
**wrong at this tag** — those lines are inside the conditional. The topic report's tuning advice
("design against 30/tic") was built on it.

Two actions:

- **Keep `rsqgr_max_speed` as a comfort ceiling, not as an engine limit.** Rename nothing; it is in
  presets (`MENUDEF.txt`: Floaty 22 / Standard 30 / Snappy 45 — note Snappy already exceeds 30 and
  nothing breaks, which is itself the proof). Once `keepmomentum` is on, Standard probably wants to
  rise; that is a headset call.
- **Read `crouchfactor` and stop fighting it.** At `vr_crouch_use_button 0` (the non-default) the
  engine drives `crouchfactor` from real head height (`gl_openvr.cpp:2512-2517`), so a player who
  physically ducks mid-swing drops below 0.75 and **re-enables the 30-unit clamp** for as long as
  they stay low. A swing that mysteriously tops out when you crouch is this, and it would cost a day.

**Label: EXISTS / S** for the `keepmomentum` line. **S** for the cap work.

---

## 7. THE CRACK HAPTIC

**Pick `LineAttack` with `LAF_NOINTERACT`. Reserve `A_QuakeEx` for sustained rope tension. Never both
on the same hand in the same tic.**

### Why

Verified by reading `P_LineAttack` top to bottom:

```
qzd173/src/playsim/p_map.cpp:4724   bool nointeract = !!(flags & LAF_NOINTERACT);
qzd173/src/playsim/p_map.cpp:4756   if (t1->player != NULL) {
qzd173/src/playsim/p_map.cpp:4763       if (damage > 0) {
qzd173/src/playsim/p_map.cpp:4765           long rightHanded = vr_control_scheme < 10;
qzd173/src/playsim/p_map.cpp:4767           rightHanded = (flags & LAF_ISOFFHAND) ? 1 - rightHanded : rightHanded;
qzd173/src/playsim/p_map.cpp:4768           vrmode->Vibrate(150, rightHanded ? 1 : 0, 0.8);
```

Three things confirmed at the source, not inferred:

1. **The pulse is upstream of every `nointeract` branch.** The `nointeract` tests that suppress the
   miss sound, the damage and the blood are at `:4922` and `:4927`, ~160 lines later. So
   `LAF_NOINTERACT` buys "no world effect" and keeps the buzz, provided `damage > 0`.
2. **It respects the user's handedness.** `vr_control_scheme < 10`, flipped by `LAF_ISOFFHAND`. For a
   per-hand whip this is the whole point: the crack buzzes the hand that cracked, in the user's own
   left/right convention, with no cvar for them to have zeroed. `A_QuakeEx`'s channels are raw axes
   (intensity X → left, Y → right, `qzd173/src/rendering/r_utility.cpp:1085-1091`) and it is on the
   mod to know which is which.
3. **The shape is right for a crack.** 150 ms at 0.8 is a hard, discrete thump. A crack is exactly
   one discrete event per lash. There is nothing to shape.

Against `A_QuakeEx` for *this* job: it depends on the user's `vr_quake_haptic_level`; it will shake
the view for anyone who has raised `r_quakeintensity` off its VR default of 0.0
(`r_utility.cpp:116`); `QuakePower` is **random** (`r_utility.cpp:911-921`), so it is noise — fine
for a thud, useless for a crisp click; and its `thrustMultiplier` defaults to **0.5**
(`qzd173/wadsrc/static/zscript/actors/actor.zs:1284`), which **adds randomised velocity to the
player** unless `damrad = 0`. On a mod whose whole subject is the player's velocity, an accidental
shove is not a cosmetic bug.

### The call

```zscript
pmo.LineAttack(ang, 0.1, pit, 1, 'None', "RSQGR_NullPuff",
               LAF_NOINTERACT | (hand == 1 ? LAF_ISOFFHAND : 0));
```

`LAF_NOINTERACT` (`constants.zs:941`), `LAF_ISOFFHAND` (`:946`). A 0.1-unit range and a
`+NOINTERACTION` invisible puff; nothing in the world changes.

### Where it lives — the series rule applies

**Do not call it inline from `whip.zs`.** The mod already routes feel cues through a named service:
`RSQGR_A11y.Ready(who, hand)` (`a11y.zs:93-99`) asks RSQ_Accessibility's Sound Cues for a per-hand
rumble and does nothing when that mod is absent. **That is the right home for the crack pulse too** —
one more cue name, `GetInt("crack", "", pn, hand)`, with the `LineAttack` fallback inside RSQ_Grapple
for when RSQ_Accessibility is not loaded. Keeps the owner's "each mod works alone and they talk
through services" rule intact, and keeps the intensity curve in the mod whose job it is.

### The one caveat to design around

`OpenVRHaptics::Vibrate` **early-returns if a pulse is already running on that channel**
(`gl_openvr.cpp:517-528`). A 150 ms crack therefore blocks all other rumble on that hand for 150 ms.
Fine for one crack per lash. It means a continuous rope-tension buzz must be `A_QuakeEx` (10 ms
self-expiring pulses, chainable per tic) and must not be fired on a hand in the tic a crack lands on
it. One `lastPulseTic[hand]` guard.

**Label: HACK / S.** A hack because it is a damage call that deals no damage, in order to reach a
buzz. It works, it is cited, it is the cheapest correct route, and §9's engine wishlist is somebody
else's document.

---

## 8. THE MIGRATION

### 8.1 What moves where

| Piece | Where it goes | Why |
|---|---|---|
| The mode matrix (five axes), the presets that name them | **new** `zscript/rsqgr/tether.zs` | Keeps `core.zs` the solver and nothing else. Register after `RSQGR_Core` in MAPINFO, before `RSQGR_Events`. |
| The per-frame hand ring + lash signature | **extend** `input.zs:91-106` (`RenderOverlay`) and the `rsqgr_pose` receiver (`input.zs:141-146`) | The only per-frame, MP-correct hand read in the mod. §2.1. Do not start a second sampler. |
| Spike filter (collapsed pose, teleport) | **extend** `util.zs:51-60` (`PoseGood`) | Two lines; fixes a shipped bug (`HANDOFF.md`, "a press in the first frame after a menu closes"). |
| `level.pixelstretch` instead of `const 1.2` | `util.zs:21`, used at `:43`, `:104` | §2.2. |
| Motion-driven arm / crack / reach / damage | **rewrite** `whip.zs:52-155` (`NetworkProcess`, `Start`, `Crack`) | §2. `Snag()` (`:168-196`) is untouched — see 8.3. |
| The sag fix | `whip.zs:213`, `:222`, `:223` (+ `:27`, `:204-206` for duration) | §1. |
| Swept-chord resolve | **new** method in `whip.zs`, calling the existing `RSQGR_Tracer` | §3. No new primitive. |
| Lasso = `rsqgr_monster_mode 2` | **extend** `monster.zs:30`, `:35-44`, new Hold branch beside `Slide` / `RSQGR_Flight` | §4. Reuses `PlaceMid`, `Restore`, the tumble, `Impact`. |
| Mass split | **extend** `core.zs:317-321` (`TooHeavy` → `ShareOfCorrection`) and the two appliers | §4.3. |
| Both hands on one rope | `hook.zs:53` (`gripHand` → a pair) and `core.zs:551-605` (`Climb`) | §5.2. Average, never sum. |
| `keepmomentum` | `core.zs` `Swing`, one line, plus a post-`LetGo` coast window | §6. |
| Crack pulse | `a11y.zs` (new cue name) + a local `LineAttack` fallback | §7. |

Three edits inside `core.zs`, and that is all: `keepmomentum` in `Swing`; the `!h.holding` guard at
`core.zs:338-339`; and the `RSQGR_Whip.Takes()` seam below.

### 8.2 The one core seam worth taking, and the last session already specified it

`whip.zs:52-71` currently works by letting core fire a claw and then **taking it back before it
moves** — it checks `phase == PH_FLY && phaseAge == 0 && traveled == 0`, stops its sounds, destroys
it, nulls `core.hooks[i]`, and starts the lash from the same origin and direction. It works because
MAPINFO registers `RSQGR_Whip` after `RSQGR_Input` (`MAPINFO.txt`, `whip.zs:10-16`). It is also
fragile: it depends on handler registration order, on three fields reading exactly zero, and on
nothing else having touched the claw in between.

`HANDOFF.md` ("For core") already asks for the clean version:

```zscript
// RSQGR_Core.Press, before Fire():
if (RSQGR_Whip.Takes(pn, hand, org, dir)) return;
```

**Take it.** With motion-driven lashing the whip needs to arm on a *hold* and crack later, so the
"take back the fresh claw" trick stops being merely fragile and starts being wrong — there may be no
claw to take back, because arming is not firing. One line in core, and `whip.zs`'s whole
`NetworkProcess` collapses into it.

### 8.3 What must not break

1. **Core's swing / reel / release maths — `core.zs:363-527` and `551-605`.** Untouched by every item
   above. In particular do not touch `core.zs:445-457` (the "a rope does no work on what hangs from
   it" speed restoration) or `core.zs:417-431` (velocity from the actual last-tic move); both are
   measured, both are better than the survey's recipe, §9.6 / §9.7.
2. **The snag path: `whip.zs:168-196` → `RSQGR_Core.Latch()` (`core.zs:289-312`).** This is the thesis
   in code — a crack that resolves to geometry builds core's hook exactly as a claw that hit that
   spot would have, and everything after it (swing, reel, tap-to-let-go) is core's unchanged. **Every
   new tether mode must enter through `Latch()` the same way.** If a mode needs state `Latch()` does
   not set, add a defaulted parameter; do not fork it.
3. **The service API — `service.zs:9-53`, tabulated in `HANDOFF.md`.** Additive only. Specifically:
   - **`hook.phase`'s domain must not change.** It is documented as `-1/0/1/2/3/4` and other mods
     read it. **Do not add a sixth phase for "holding".** Express a lasso as `PH_BITE` plus a new
     request. Cheapest honest shape: `h.holding` as a field, surfaced as a *new* integer request
     `"lassoed"` (1 held, 0 not, -1 never), beside the existing `"yanked"` (`service.zs:427-432`).
   - New requests only, all answering `-1` when unknown (the contract at `service.zs:1-7`):
     `"tether.mode"`, `"lash"` (tics since a crack, like `"whipped"` but player-side),
     `"lassoed"`, `"hands"` (1 or 2 on the tether).
   - `"whipped"` (`service.zs:444`) keeps its exact meaning — tics since a crack hit *that monster*.
     RSQ_TopHPBars reads `"stunned"` (`:445-454`) off it.
   - **Everything that hurts must still be noted BEFORE the blow lands** (`service.zs:54-58`): a
     motion-driven crack calls `RSQGR_Events.Whipped(v, pmo)` before `RSQGR_Perks.Hurt`, exactly as
     `whip.zs:129` does today. A scorer hearing the death inside `DamageMobj` must still read 0.
4. **`RSQGR_Events` registered last in MAPINFO** (`MAPINFO.txt`, comment at `service.zs:56-58`). A new
   `tether.zs` handler goes *before* it.
5. **Every cross-mod link stays a name lookup.** RSQ_Accessibility (`a11y.zs`), RSQ_SlowMo
   (`util.zs:90-100`), RSQ_Score (`service.zs:370-381`), RSQ_DistanceGrab (`input.zs:149-150`),
   BD22Q (`climb.zs:38-44`, by class *name*, `FindStateByString`), the Gun Bonsai patch
   (`perks.zs`). **No new hard link, no new `#include` of another mod, no new class reference.** The
   owner's series rule, and the mod currently honours it everywhere.
6. **`rsqgr_reset` in `KEYCONF.txt` must be regenerated** whenever CVARINFO changes. `HANDOFF.md`
   ("For core") already flags that it is assembled from both halves' lists.
7. **Save/load.** All new per-hand state belongs in handler fields (serialised with the level), and
   `RSQGR_Core.Settle()` (`core.zs:162-170`) must reset it on the first tic after a load, for the
   reason documented there: `WorldLoaded` runs *before* a saved handler's fields are read back, so a
   reset placed there comes back from the save. Any new "armed" or "held" latch is exactly the kind
   of state that bug bites.

### 8.4 Cvars: which survive as values, which become ceilings

| Cvar | Today | After |
|---|---|---|
| `rsqgr_whip_reach` 192 | the reach | **CEILING.** Reach scales with peak hand speed. §2.4 |
| `rsqgr_whip_damage` 12 | the damage | **CEILING.** Scales with peak hand speed. §2.4 |
| `rsqgr_max_speed` 30 | believed an engine limit | **COMFORT CEILING**, raiseable; real engine ceiling 5000. §6.4 |
| `rsqgr_mass_max` 1000 | the switch between "it comes" and "you go" | **CLAMP** on the mass split to 1.0. §4.3 |
| `rsqgr_whip` bool | whip instead of claw | **alias** that selects the whip preset of `rsqgr_tether_mode`. Existing inis keep working. |
| `rsqgr_whip_above` 16, `rsqgr_whip_snag`, `rsqgr_whip_stun` | — | unchanged values |
| `rsqgr_reel_speed`, `min_len`, `pump`, `drag`, `rope_friction`, `attach`, `hop`, `climb*`, `slide*`, `yank*`, all `rope_style`/`chain_*` | — | **unchanged**. The solver is untouched. |
| `OUT_TICS` / `BACK_TICS` (`whip.zs:26-27`, consts) | the timing | **gone.** Replaced by `rsqgr_whip_crack_speed` (m/s), `rsqgr_whip_arm_tics`, `rsqgr_whip_cooldown`, `rsqgr_whip_subs`. |

`MENUDEF.txt`'s preset rows (`RSQGR_P_Range` / `_Swing` / `_Pull`) are generated by
`tools/rsmenu/presets.py` — the mode-matrix presets belong there, not hand-written into MENUDEF.

---

## 9. WHERE THE MOD IS ALREADY RIGHT

Listed because every one of them is something a rework could plausibly "fix" and must not.

1. **`util.zs:51-60` gates every hand read on `OverrideAttackPosDir`** and falls back to the body's
   own aim. `00_CAPABILITIES` §1.2 names this as mandatory; `01_MELEE` §0.2(1) makes it asymmetry #1.
   The mod does it in one place, for both hands, with a clean fallback. **Do not touch it** — only
   add the collapse test (§2.1), which is a new case, not a correction.
2. **`util.zs:41-45` carries the pixelstretch term in the direction**
   (`sin(pitchUp) / PIXELSTRETCH`) and `HANDOFF.md` records it verified equal to the engine's own
   `AttackDir` at five poses. This is the trap `00_CAPABILITIES` §1.2a says costs ~17 % on every
   height, and the mod did not fall into it. The only change is reading `level.pixelstretch`.
3. **`input.zs:1-20, 28-72, 108-151` — the two-hop local-event → net-event pose packing.** The VR
   pose fields are written for the camera player only, so the hand is read on the machine that owns
   it and packed into the network event. `05_GRAPPLE` §5.3 calls multiplayer a **WALL** for every
   hand-driven rope mechanic. This mod solved it, in 10 bits per axis, and degrades to the live pose
   when no pose is packed (`input.zs:128-132`). Keep the design exactly.
4. **`input.zs:91-106` + the comment at `:79-90` — the hand's height taken against the frame's own
   `e.ViewPos.z`, not `pmo.Pos` and not `player.viewz`.** The comment records a measured 6-14 unit
   per-stroke wobble from each wrong choice and explains why (`viewz` is set in `P_PlayerThink`
   before the pawn moves that tic). No survey in this set contains this fix, and §2's whole speed
   signal depends on it.
5. **`core.zs:2-13` and the whole of `Swing` — the player is moved by VELOCITY ONLY, never placed.**
   `05_GRAPPLE` §4.6 names unvalidated `SetOrigin` correction as "the main implementation risk in
   this design", and §7's rough edges list it first. The mod's header comment is the correct answer
   to it: solve the rope in `WorldTick`, on the velocity the engine is about to move by, and let the
   engine's own collision stop him. **Do not introduce positional correction for the player.** (The
   *monster* side is placed, by `PlaceMid`, for its own documented reason.)
6. **`core.zs:445-457` — "a rope does no work on what hangs from it".** Projecting the step back onto
   the rope's sphere cuts the chord and bleeds speed; the mod restores the pre-projection magnitude
   on a free swing. The comment records the measurement it fixes (~20 % of height lost per half
   swing). This is strictly better than the naive projection `05_GRAPPLE` §4.2 prescribes.
7. **`core.zs:417-431` — velocity reconstructed from the actual last-tic move.** The right answer to
   a real engine artefact: a blocked substep aborts the rest of the move (`p_mobj.cpp:2283-2285`) but
   leaves the blocked component in `Vel`, so a rope that keeps adding to it builds a fling. The
   comment carries the measurement (14.7 → 17.6 u/tic over 8 tics with the pawn not moving). Keep it
   — and note §6.2(3): `keepmomentum` is what makes it correct on grounded tics too.
8. **`monster.zs:255-268`, `:419-420`, `:600-616` — the held monster.** Pain-state pinning with
   `tics = -1`, `bNOBLOCKMONST` cleared, every borrowed flag captured before anything is touched and
   restored on every exit path including level unload (`:648-651`). It keeps the actor in the
   blockmap, shootable and movable — which `freezetics` does not (it cannot be dragged) and
   `+NOINTERACTION` does not (it leaves the world). **Better than both options the surveys offer.**
9. **`monster.zs:362-390` `PlaceMid`** — `SetZ` clamped to the room, the vertical move *refused* when
   `TestMobjLocation` fails, then `TryMove`, then three 6-unit lift-outs, and a nudge-out at landing
   (`:568-580`). `HANDOFF.md`: "TestMobjLocation at landing: fits 1 in every run."
10. **`hook.zs:1-11` — the rope is drawn from the claw actor's own `Tick`, not from `WorldTick`,
    because thinkers run after `WorldTick` and the player has already moved by then.** Drawn earlier
    the rope's hand end would trail the hand by 20 units at the bottom of an arc. Precise, measured,
    and the kind of thing that gets "simplified" into a bug.
11. **`hook.zs:390-411` `RSQGR_Tracer`** — a `LineTracer` subclass with arbitrary start and direction
    and a `TraceCallback` that already rejects the shooter, players, the mod's own rope actors,
    missiles and non-shootables. This is `TRF_ABSPOSITION` without the flag, already shipped and
    already tested, and it is what makes §3 free.
12. **`whip.zs:118` / `core.zs:228` pass `TRACE_HitSky` in the `traceFlags` slot** — correct, it is a
    real `ETraceFlags` value `0x0040` (`qzd173/wadsrc/static/zscript/doombase.zs:256`,
    `qzd173/src/playsim/p_trace.h:102`), and the sky is then handled explicitly
    (`core.zs:236`). Not a confusion with `ETraceResult`.
13. **The grab binds are mod aliases, not the engine grip** (`KEYCONF.txt`, `input.zs:68-71`). This
    is gotcha 10a's recommended defence, reached independently. §5.1.
14. **`core.zs:154-170` `Settle()`** — the first-tic-after-load reset, placed there rather than in
    `WorldLoaded` because a saved handler's fields are read back *after* `WorldLoaded`. The comment
    records the bug it fixes ("a hold reset there came back from the save and kept climbing").

---

## 10. LABEL AND COST SUMMARY

| # | Piece | Label | Cost |
|---|---|---|---|
| 1 | Sag fix — post-crack bow decay, frozen wave phase (`whip.zs:213`, `:222`, `:223`) | EXISTS | S |
| 1b | Reach-cvar A/B test to separate sag from timing (no rebuild) | EXISTS | S |
| 2.1 | Per-frame hand ring, on the existing `rsqgr_pose` stream | BUILDABLE | S |
| 2.1b | Spike filter in `PoseGood` (collapsed pose) — fixes a shipped bug | BUILDABLE | S |
| 2.2 | `level.pixelstretch` instead of `const 1.2` | EXISTS | S |
| 2.3 | Lash signature (accel-out → brake/reverse) + hold-to-arm + cooldown | BUILDABLE | M |
| 2.4 | Reach and damage from peak hand speed | BUILDABLE | S |
| 3 | Swept-chord with angle-lerped sub-sampling, via `RSQGR_Tracer` | BUILDABLE | M |
| 4 | Lasso: `rsqgr_monster_mode 2` Hold + the `!h.holding` seam | BUILDABLE | M |
| 4.3 | Mass split replacing the `TooHeavy` binary | BUILDABLE | M |
| 5.2 | Two-handed lash from hand POSITIONS (immune to `weaponStabilised`) | BUILDABLE | M |
| 5.2b | Both hands on one rope (`gripHand` → a pair, averaged) | BUILDABLE | S |
| 6 | `player.keepmomentum = true` in `Swing` + coast window | EXISTS | S |
| 6.4 | `rsqgr_max_speed` as a ceiling; read `crouchfactor` | EXISTS | S |
| 7 | Crack haptic via `LineAttack` + `LAF_NOINTERACT`, routed through the a11y service | HACK | S |
| 8.2 | `RSQGR_Whip.Takes()` seam in `RSQGR_Core.Press` | BUILDABLE | S |
| — | Sub-tic hand history (true gesture curvature) | WALL | — |
| — | Script-callable `VRMode::Vibrate` (a binding, not a feature) | WALL | — |
| — | Script-set view pitch/roll for a swing-lean | WALL | — |
| — | A line/beam primitive for the rope | WALL | — |

The four WALLs are named for completeness. None of them blocks anything in §§1-8: the swept chord
replaces the hand history for *hit testing* (it only costs curved-gesture recognition), `LineAttack`
reaches the buzz, the rope is already drawn as a segment chain to a measured standard
(`hook.zs:143-161`), and nothing here wants to move the camera.

---

## 11. OPEN, AND HONEST ABOUT IT

1. **Which VR backend the owner's binary is.** `00_CAPABILITIES` §0 is right that this decides which
   citations are operative. The mod's own comment (`util.zs:3-10`) names OpenXR
   (`hw_openxrdevice.cpp:587-609`) and its rig measurements agree with it. Everything in §§1-8 reads
   `AttackPos` / `OffhandPos` and the angles, which both paths fill, so nothing above depends on the
   answer — **except `weaponStabilised` (§5.1), which exists only on the OpenVR path in this tree.**
   §5.2's position-based design is immune either way, which is why I recommend it.
2. **Whether the owner's "sags for a bit" is the droop or the 114 ms soft lash.** §1's reach A/B test
   settles it in one session with no rebuild. I would not write the §1 fix before running it, because
   if the answer is "the timing", §2 subsumes §1 and the four `Draw` lines want designing against the
   new signal rather than patching.
3. **`CRACK_SPEED`'s real value.** 6 m/s is reasoned from the unit conversion, not measured on a
   headset. It is the one number in this document that wants a slider before it wants a default.
4. **Whether a two-handed lash reads as a whip at all.** `HANDOFF.md` asked this of the one-handed
   version and the answer was no. The angular-rate drive (§5.2) is my argument that two hands is
   where the gesture becomes legible, because rotating a handle is what cracking a whip physically
   is. It is an argument, not a measurement.
5. **`rsqgr_max_speed`'s new default once `keepmomentum` is on.** Raising it is free; how far is a
   comfort judgement and the mod has a vignette service for exactly that
   (`core.zs:200-217`, `a11y.zs`). Headset call.
6. **Cost of the Hold branch per tic with several monsters lassoed.** `RSQGR_Flight` has 16 slots and
   measures 0.036 ms mean for one Baron; a hold is cheaper per tic than a path (no `MeasureArc`, no
   tumble) but runs for longer. Unmeasured.
