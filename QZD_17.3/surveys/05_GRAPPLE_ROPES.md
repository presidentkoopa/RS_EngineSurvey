# 05 — Grapple, Ropes, Whips, Swinging

**Engine surveyed:** tag `qzd17.3` of `github.com/emawind84/gzdoom`. All paths repo-relative at that tag.
**Constraint honoured:** sprite layer only. Nothing below authors a map, a linedef special, a sector
tag or any world geometry. Every grabbable surface is discovered at runtime by a trace.

---

## 1. Verdict

Grapple and rope mechanics are **buildable on this engine without touching C++**, and they are
buildable better than the baseline suggests, because the baseline is wrong on one point: the
weapon hand *is* readable. `AttackPos` / `AttackAngle` / `AttackPitch` / `AttackRoll` are exposed
to ZScript at `wadsrc/static/zscript/actors/actor.zs:268-271` and are written per frame with the
real controller pose on both VR backends — so hand-over-hand climbing has two hands, not one.
The surface-normal question, which the brief correctly called decisive, resolves in our favour:
`FLineTraceData` carries no normal field, but the normal is **exactly reconstructible** from
`HitSector.floorplane.Normal` / `.ceilingplane.Normal` for floors and ceilings (slopes included)
and from `HitLine.delta` signed by `LineSide` for walls.

The two things that will kill a naive implementation are both QuestZDoom-specific and both are
one line to fix: `MovePlayer` **wipes the player's horizontal velocity every tic while on the
ground** (`player.zs:1374-1379`), and `A_ChangeVelocity` on the player is a **no-op by default**
because `vr_recoil` defaults to false (`p_actionfunctions.cpp:90`, `actions.zs:77`). Write `Vel`
directly, set `player.keepmomentum = true` every tic, and set `bNOFRICTION` while roped.

Rocket Jockey — the hardest case — is **BUILDABLE, cost L**, and it is the mechanic most exposed
to the ground-velocity wipe, because unlike every other mechanic here it is played *on the floor
at speed*. The real walls are haptics (none reachable from script, baseline §4), rope rendering
(no line primitive; segment chain only) and multiplayer (hand poses exist only for the local
viewpoint).

---

## 2. What already exists

### 2.1 Tracing

| Thing | Citation |
|---|---|
| `Actor.LineTrace(angle, distance, pitch, flags, offsetz, offsetforward, offsetside, out FLineTraceData)` | `wadsrc/static/zscript/actors/actor.zs:805` |
| `struct FLineTraceData` (ZScript) | `wadsrc/static/zscript/actors/actor.zs:36-61` |
| `struct FLineTraceData` (C++) | `src/playsim/p_linetracedata.h:9-24` |
| Field registration | `src/scripting/vmthunks_actors.cpp:2250-2262` |
| `P_LineTrace` implementation, and where every output field is filled | `src/playsim/p_map.cpp:5165-5311` |
| `TRF_*` flag constants | `wadsrc/static/zscript/constants.zs:951-963` |
| `class LineTracer` — arbitrary start/direction, scriptable callback | `wadsrc/static/zscript/doombase.zs:307-320` |
| `struct TraceResults` (the richer result LineTracer gives) | `wadsrc/static/zscript/doombase.zs:279-304` |
| `BlockThingsIterator`, `BlockLinesIterator` | `wadsrc/static/zscript/doombase.zs:218-237` |

### 2.2 Hand pose — correction to `_BASELINE_FACTS.md` §1a

The baseline states: *"There is no main-hand pose field. This is the single most important fact in
the survey."* **That is not true at this tag.** Citing, as the baseline requires:

| ZScript | Declared | Native field | Thunk |
|---|---|---|---|
| `Actor.AttackPos` (vector3, readonly) | `actor.zs:268` | `src/playsim/actor.h:1654` | `vmthunks_actors.cpp:2187` |
| `Actor.AttackPitch` (double, readonly) | `actor.zs:269` | `actor.h:1655` | `vmthunks_actors.cpp:2188` |
| `Actor.AttackRoll` (double, readonly) | `actor.zs:270` | `actor.h:1657` | `vmthunks_actors.cpp:2189` |
| `Actor.AttackAngle` (double, readonly) | `actor.zs:271` | `actor.h:1656` | `vmthunks_actors.cpp:2190` |
| `Actor.AttackDir(actor, angle, pitch)` | `actor.zs:858` | `actor.h:1659` (fn ptr) | — |
| `Actor.OffhandRoll` (double, readonly) | `actor.zs:274` | `actor.h:1664` | `vmthunks_actors.cpp:2193` |

These are written with the **real weapon-controller pose**, per frame, exactly as the offhand ones are:

- OpenXR: `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:585-595` (mainhand),
  `:599-609` (offhand). Same function, same `weaponoffset[]` / `offhandoffset[]` treatment.
- OpenVR: `src/rendering/gl/stereo3d/gl_openvr.cpp:2525-2537` (mainhand),
  `:2540-2554` (offhand).
- Flat / non-VR fallback: `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:442-444` collapses
  `AttackPos == OffhandPos == ` the view position. So a two-hand design degrades gracefully to
  "both hands at the eye" on a desktop build rather than breaking.

**Consequence:** every "you only have one hand" caveat in the baseline is lifted for this topic.
You have two full 6-DoF hand poses. The baseline's §1b still stands in full — **there are still no
velocity fields for either hand**, so you difference stored previous-tic positions, at 35 Hz.

Note also `Actor.OverrideAttackPosDir` (`actor.zs:276`, set at `hw_vrmodes.cpp:438`) — read it to
know whether the hand poses are real controller poses or the flat-mode fallback.

### 2.3 Player movement, and the two QuestZDoom traps

**Trap 1 — the ground velocity wipe.** `PlayerPawn.MovePlayer`, `wadsrc/static/zscript/actors/player/player.zs:1373-1379`:

```
[friction, movefactor] = GetFriction();
//Taken from the Wolf-3D TC - Prevent player having momentum/acceleration to avoid puking
if (!vr_momentum && !player.keepmomentum
&& player.onground && friction == ORIG_FRICTION)
{
    vel.XY *= 0.0001;
    Speed = Default.Speed * 8;
}
```

`vr_momentum` is `CVAR(Bool, vr_momentum, false, ...)` — `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:120`.
Default **false**. So by default, **the instant a grappling or swinging player touches the ground,
all horizontal momentum is annihilated.**

The escape hatch is shipped and writable: `PlayerInfo.keepmomentum` is declared
`native bool keepmomentum;` at `player.zs:3108` (not readonly), backed by `src/playsim/d_player.h:414`
and registered at `src/playsim/p_user.cpp:1915`. The engine's own recoil code sets it
(`p_actionfunctions.cpp:1295`, `actions.zs:78`, `p_lnspec.cpp:1224`). A mod may set it the same way.

It is auto-cleared when the player is nearly stopped — `player.zs:1385-1388`, against
`vr_momentum_threshold` (default `1.f`, `hw_vrmodes.cpp:121`). That clear is harmless; it only
fires below 1 map unit/tic.

`player.onground` is computed at `player.zs:1370` as `(pos.z <= floorz + 2) || bOnMobj || bMBFBouncer || noclip2`.
So a player hanging on a rope above the floor is never `onground` and is never wiped — the trap
only bites at ground contact, which is exactly where swing-landing and Rocket Jockey live.

**Trap 2 — `A_ChangeVelocity` does nothing to the player.** `wadsrc/static/zscript/actors/actions.zs:75-78`:

```
if (player != NULL && player.mo == ref)
{
    if (!vr_recoil) return;
    player.keepmomentum = true;
```

`CVAR(Bool, vr_recoil, false, ...)` — `src/playsim/p_actionfunctions.cpp:90`. Default **false**.
`A_Recoil` has the identical gate at `p_actionfunctions.cpp:1294`. **The brief named
`A_ChangeVelocity` as a candidate API; at this tag it is not one for the player.** Write
`player.mo.Vel` directly — `native vector3 Vel;` at `actor.zs:112`, ungated — or use
`Thrust()` (`actor.zs:860`, thunk `src/scripting/vmthunks_actors.cpp:480-500`), which has no VR gate.

**Tic ordering — this decides where your code goes.** `src/p_tick.cpp:164-172`:

```
for (i = 0; i < MAXPLAYERS; i++)
    if (Level->PlayerInGame(i))
        P_PlayerThink(Level->Players[i]);     // :166  <- MovePlayer, the velocity wipe

Level->localEventManager->WorldTick();        // :170  <- your handler
Level->Tick();                                // :171
Level->Thinkers.RunThinkers(Level);           // :172  <- Actor::Tick -> P_XYMovement/P_ZMovement
```

So `StaticEventHandler.WorldTick()` (`wadsrc/static/zscript/events.zs:177`) runs **after** the wipe
and **before** the velocity is integrated into position. That is the correct and only clean
injection point for a rope constraint. `Inventory.DoEffect` (`inventory.zs:982`, driven from
`src/playsim/p_mobj.cpp:4007-4017`) also runs inside `RunThinkers` before the player's own movement,
and is a fine alternative if you want the rope to live on a carried token.

**Friction and clamps:**

- Airborne: friction is **skipped entirely** — `src/playsim/p_mobj.cpp:2447-2463` returns early
  when `Z() > floorz + 2`, and `airfriction` defaults to 1. A pendulum swing in the air is lossless.
- On ground: `Vel.X *= friction; Vel.Y *= friction;` — `p_mobj.cpp:2529-2531`. `ORIG_FRICTION` is
  0.90625, i.e. ~9% bled per tic, half-life ~7 tics. Suppress with `bNOFRICTION`
  (`MF8_NOFRICTION`, defined `src/scripting/thingdef_data.cpp:324`), which causes an early return at
  `p_mobj.cpp:2442`.
- Hard horizontal speed cap: `MAXMOVE` = 30 map units/tic = 1050 units/sec, applied in
  `P_XYMovement` at `p_mobj.cpp:2065-2066` and `:2100-2101`. **`Vel.Z` is not clamped there.**
  This is your ceiling for every swing and every Rocket Jockey slingshot — design the tuning
  numbers against 30/tic, not against some invented limit.
- Below `STOPSPEED` (`0x1000/65536` = 0.0625) with no input, velocity is zeroed outright —
  `p_mobj.cpp:2497-2515`.
- A `bNOGRAVITY` player in the air gets `Vel.Z *= FRICTION_FLY` unless `bNOFRICTION`
  (`p_mobj.cpp:2641-2643`), and gets a vertical bob added at `p_mobj.cpp:2637` when the user cvar
  `FViewBob` is on. **In VR that bob is a comfort problem** — if you use `bNOGRAVITY` for a hang,
  zero the pawn's `FlyBob`.

### 2.4 The offhand weapon system

Already established as a real shipped feature in `_BASELINE_FACTS.md` §2. For this topic the parts
that matter are: `bOffhandWeapon` routing (`player.zs:1977-1998`), `PSP_OFFHANDWEAPON`, and
independent per-hand fire latches — `player.attackdown` / `player.ohattackdown` set at
`player.zs:583-584` from `CheckWeaponFire(0)` / `CheckWeaponFire(1)`, with `ohattackdown` declared
`player.zs:3062` / `src/playsim/d_player.h:366`. **This is how you read two independent grips.**

### 2.5 Rope visuals available at the sprite layer

- `LevelLocals.SpawnParticle(FSpawnParticleParams)` — `wadsrc/static/zscript/doombase.zs:560`;
  the struct, with `texture`, `style`, `pos`, `vel`, `accel`, `size`, `startroll`, `rollvel`,
  `rollacc`, is `doombase.zs:382-403`.
- `Actor.A_SpawnParticleEx(...)` with a `TextureID`, render style and roll — `actor.zs:1226`.
- Ordinary sprite actors with `roll` + `RF_ROLLSPRITE`, positioned with `SetOrigin(pos, true)`
  (`actor.zs:753`; the `moving` argument keeps interpolation and re-links sector lists correctly).

There is **no line/beam primitive**. A rope is a chain of short segments, full stop.

---

## 3. Building blocks — the exact API set

| Need | API | Citation |
|---|---|---|
| Fire a hook from the real controller | `LineTrace(self.Angle, dist, self.Pitch, TRF_USEWEAPON\|TRF_ISOFFHAND\|TRF_SOLIDACTORS\|TRF_BLOCKUSE, data: d)` | `actor.zs:805`; the offhand remap is `p_map.cpp:5183-5192` |
| …from the main hand | same, `TRF_USEWEAPON` without `TRF_ISOFFHAND` | `p_map.cpp:5193-5199` |
| Trace between two arbitrary world points (rope line-of-sight) | `LineTracer.Trace(start, sec, dir, maxDist, flags, wallMask, ignoreAllActors, ignore)` | `doombase.zs:309` |
| Sector for a LineTracer start | `Level.PointInSector(vec2)` | `doombase.zs:514` |
| What the trace hit | `d.HitType` (`TRACE_HitFloor/Ceiling/Wall/Actor/HasHitSky`) | `actor.zs:38-46` |
| Where | `d.HitLocation` (vector3), `d.Distance` | `actor.zs:54,56`; filled `p_map.cpp:5304,5306` |
| Which actor | `d.HitActor` | `actor.zs:50` |
| Which wall, which side, which texture tier | `d.HitLine`, `d.LineSide`, `d.LinePart` (0 upper / 1 mid / 2 lower) | `actor.zs:51,59,60`; filled `p_map.cpp:5271-5300` |
| Which sector / plane | `d.HitSector`, `d.SectorPlane` (0 floor / 1 ceiling) | `actor.zs:52,61`; filled `p_map.cpp:5269-5270` |
| Surface texture, for a grabbability rule | `d.HitTexture` → `TexMan.GetName(...)` | `actor.zs:53` |
| 3D floor | `d.Hit3DFloor` | `actor.zs:53` |
| Portal warning | `d.NumPortals` | `actor.zs:57`; filled `p_map.cpp:5308` |
| Set player velocity | `player.mo.Vel = ...` | `actor.zs:112` |
| Keep it through ground contact | `player.keepmomentum = true;` **every tic** | `player.zs:3108`; wipe at `player.zs:1374-1379` |
| Stop ground friction eating it | pawn `bNOFRICTION = true` | `thingdef_data.cpp:324`; effect `p_mobj.cpp:2442` |
| Move the pawn without re-solving physics | `SetOrigin(newpos, true)` | `actor.zs:753` |
| Move it *with* collision | `TryMove(xy, dropoff)` / test with `CheckMove(xy, flags)` | `actor.zs:831-832` |
| Portal-correct difference between two points | `Level.Vec3Diff(a, b)`, `Level.Vec3Offset(pos, dir)` | `doombase.zs:523,528` |
| Hand poses | `AttackPos/Angle/Pitch/Roll`, `OffhandPos/Angle/Pitch/Roll`, `HmdPosition` | `actor.zs:268-277` |
| Pure controller forward vector | `OffhandDir(self, self.Angle, self.Pitch)` / `AttackDir(...)` | `actor.zs:858-859`; maths `hw_vrmodes.cpp:390-416` |
| Grip latches | `player.attackdown`, `player.ohattackdown` | `player.zs:3061-3062` |
| Per-tic hook | `StaticEventHandler.WorldTick()` | `events.zs:177`; ordering `p_tick.cpp:170` |
| Actor mass, for pull-me-or-pull-you | `Actor.Mass` | `actor.zs` (`native int Mass`) |
| Rope visual | `Level.SpawnParticle` / `A_SpawnParticleEx` / roll sprites | `doombase.zs:560`, `actor.zs:1226` |

### 3.1 Surface normals — the decisive answer

**`FLineTraceData` has no normal field.** `HitDir` is *not* a normal: it is assigned
`trace.HitVector` (`p_map.cpp:5305`), and `HitVector` is set to `Vec`, the trace's own direction
vector (`src/playsim/p_trace.cpp:99`, `:237`, `:279`). Do not use it as a normal — it will point
*into* the wall no matter which wall you hit.

**But the normal is exactly reconstructible, with no engine change:**

```
Vector3 SurfaceNormal(FLineTraceData d)
{
    if (d.HitType == FLineTraceData.TRACE_HitFloor && d.HitSector)
        return d.HitSector.floorplane.Normal;        // slope-correct, points up
    if (d.HitType == FLineTraceData.TRACE_HitCeiling && d.HitSector)
        return d.HitSector.ceilingplane.Normal;      // slope-correct, points down
    if (d.HitType == FLineTraceData.TRACE_HitWall && d.HitLine)
    {
        Vector2 dl = d.HitLine.delta.Unit();
        Vector3 n = (dl.y, -dl.x, 0);                // right-hand normal of the front side
        if (d.LineSide == Line.back) n = -n;
        return n;                                    // Doom walls are always vertical
    }
    return (0, 0, 0);
}
```

Citations for the pieces: `SecPlane.Normal` is `native readonly Vector3 Normal;` at
`wadsrc/static/zscript/mapdata.zs:265`, with `isSlope()` / `PointOnSide()` / `ZatPoint()` at
`:269-271`. `Sector.floorplane` / `.ceilingplane` are `mapdata.zs:404-405`. `Line.delta` is
`native readonly Vector2 delta;` at `mapdata.zs:212`, documented "precalculated v2 - v1".
`Line.ESide { front=0, back=1 }` is `mapdata.zs:167-171`. `Line.v1`/`v2` (for the wrap algorithm)
are `mapdata.zs:211`. For 3D floors, `d.Hit3DFloor` is an `F3DFloor` (`mapdata.zs:280`) carrying
its own top/bottom planes.

**Verify the sign empirically once** (print `normal dot (player - hitpos)` on a known wall and a
known floor) before trusting the polarity in shipped tuning — the convention is consistent but
the cost of being wrong is a rope that pushes you into geometry.

With the normal in hand, grabbability is a pure runtime rule and needs no map authoring:

- "ceiling you can hook" → `HitType == TRACE_HitCeiling && normal.z < -0.7`
- "ledge you can pull onto" → trace the ceiling just past the wall hit; if it is far above, there's a lip
- "too steep to grip" → `abs(normal.z) < 0.3` for a wall-run
- "wooden beam" → `TexMan.GetName(d.HitTexture)` matched against a mod-owned name list
- "sky — never anchor" → `TRACE_HasHitSky` (`actor.zs:45`), or pass `TRF_NOSKY` (`constants.zs:957`)

---

## 4. Mechanic designs

Conventions used below: `dt = 1.0/TICRATE` (TICRATE = 35). Positions in map units. `P` is
`player.mo.pos`. `A` is the anchor. `u` is the unit vector **anchor → player**. `L` is current
rope length. All of this runs in `StaticEventHandler.WorldTick()` unless stated.

Boilerplate every design below assumes, run once per tic while a rope is live:

```
let p = players[consoleplayer];            // or iterate PlayerInGame
p.keepmomentum = true;                     // player.zs:3108 — defeats the wipe at player.zs:1374
p.mo.bNOFRICTION = true;                   // thingdef_data.cpp:324 — defeats p_mobj.cpp:2529
// ...constraint...
// on release: clear bNOFRICTION; leave keepmomentum set until the engine clears it
// itself at player.zs:1385 when |vel| drops under vr_momentum_threshold.
```

---

### 4.1 Hookshot — fire, attach, reel in (both directions)

**What the player physically does.** Points a hand at a wall, ceiling or enemy and presses the
trigger. The hook flies out; on a hit the rope goes taut and either drags the player to the point
or drags the thing to the player. Holding the trigger reels; releasing drops the rope.

**How script detects it.** Implement the hookshot as a weapon. For the offhand, give it
`bOffhandWeapon` (`player.zs:1977-1998`) and the engine routes the trigger for you; the latch is
`player.ohattackdown` (`player.zs:584`). In the Fire state:

```
LineTrace(self.Angle, 2048, self.Pitch,
          TRF_USEWEAPON | TRF_ISOFFHAND | TRF_SOLIDACTORS | TRF_BLOCKUSE | TRF_NOSKY,
          data: d);
```

Because `TRF_USEWEAPON|TRF_ISOFFHAND` is set, `P_LineTrace` replaces the start with
`player->mo->OffhandPos` and the direction with `OffhandDir(t1, angle, pitch)`
(`p_map.cpp:5183-5192`). Passing `self.Angle` / `self.Pitch` makes `MapWeaponDir` subtract the
actor's own angles (`hw_vrmodes.cpp:401-402`) so the local vector is `(1,0,0)` — i.e. **straight
out of the controller**. That is the call you want; do not try to compute the ray yourself.

For the main hand, drop `TRF_ISOFFHAND` and the same trace starts at `AttackPos` with `AttackDir`
(`p_map.cpp:5193-5199`).

Prefer a flying hook *actor* over an instant trace if you want travel time — spawn it with
`Vel3DFromAngle` (`actor.zs:857`) and resolve the anchor in its `Die`/`Crash` or a
`TRACE_HitWall` check each tic. The trace version is better for a first prototype: it is one call
and it already tells you the surface.

**How it resolves.** Classify with `d.HitType` and the reconstructed normal (§3.1):

| Hit | Rule | Result |
|---|---|---|
| `TRACE_HitActor`, `d.HitActor.Mass <= PullLimit` | light enemy or item | **pull it to you** (4.1b) |
| `TRACE_HitActor`, heavier | big monster, immovable | **pull yourself to it** |
| `TRACE_HitCeiling`, `normal.z < -0.7` | overhead anchor | **swing** (4.2) |
| `TRACE_HitWall`, grabbable texture/normal | wall anchor | **reel in flat** |
| `TRACE_HasHitSky` or `d.NumPortals > 0` | refuse | sproing, no attach |

**Reel the player to the point** — the simple, non-pendulum mode:

```
Vector3 r   = Level.Vec3Diff(P, A);          // portal-correct, doombase.zs:523
double  dist = r.Length();
if (dist < ARRIVE) { Detach(); return; }
Vector3 u   = r / dist;                      // player -> anchor here
p.mo.Vel    = u * REEL_SPEED;                // REEL_SPEED in units/tic, cap at ~24 (MAXMOVE is 30)
p.mo.bNOGRAVITY = true;                      // otherwise gravity bends the line into a sag
```

Clear `bNOGRAVITY` on detach. Remember the `FViewBob` bob at `p_mobj.cpp:2637` — zero the pawn's
`FlyBob` property or the player gets a sine wave added to their eye height for the whole ride.

**4.1b Pull the enemy to you.** Each tic:

```
Actor t = hookTarget;
t.Vel = Level.Vec3Diff(t.pos, hookPos).Unit() * REEL_SPEED;
t.bNOGRAVITY  = true;
t.bNOFRICTION = true;                       // p_mobj.cpp:2442 — else ORIG_FRICTION eats it
```

The monster will fight you, because `A_Chase` repositions it directly. The fix is to park it in a
mod-owned state with a long `tics` and no action function — `Actor::Tick` reaches `P_XYMovement`
(`p_mobj.cpp:4344`) regardless of which state the actor is in, so velocity remains the only mover.

**Do not use `freezetics`.** `AActor::Tick` returns at `p_mobj.cpp:3948-3952` when `freezetics > 0`,
before any movement runs — a frozen actor cannot be dragged at all. Likewise `bSKULLFLY` is
tempting (it suppresses friction at `p_mobj.cpp:2442` and does contact damage at
`p_map.cpp:1553`) but it is **auto-cleared on the first wall contact** unless
`MF9_NOAUTOOFFSKULLFLY` is set — `p_mobj.cpp:2134-2137`.

**Feel knobs.** `REEL_SPEED` (units/tic; 30 is the engine ceiling), hook travel speed, `ARRIVE`
radius, `PullLimit` mass threshold, whether the rope stays attached on arrival, reel acceleration
curve (ease in over ~8 tics reads far better in VR than an instant snap), and a yank impulse on
the first tic.

**Label: BUILDABLE. Cost: S** for the player-pull direction; **M** including enemy-pull and the AI
suppression.

---

### 4.2 Swinging rope — pendulum, momentum carry, release window

**What the player physically does.** Hooks an overhead point, leaves the ground, and swings. At the
bottom of the arc they let go and fly.

**The per-tic maths.** This is a position-based inextensible-rope constraint, which is the right
choice here because the engine integrates position from velocity for us and we only get one
correction opportunity per tic. Run it in `WorldTick` (`p_tick.cpp:170`) — after the velocity wipe,
before `P_XYMovement`:

```
// --- per tic, rope live ---
Vector3 P = p.mo.pos;
Vector3 r = Level.Vec3Diff(P, A);        // A - P  ... anchor relative to player
double  d = r.Length();
if (d <= L) { /* rope slack: do nothing at all, free fall */ }
else
{
    Vector3 u = r / d;                   // unit, player -> anchor

    // 1. positional correction: remove the stretch so the rope cannot creep longer
    p.mo.SetOrigin(P + u * (d - L), true);     // actor.zs:753, moving=true keeps interpolation

    // 2. velocity projection: cancel only the component pulling AWAY from the anchor
    double radial = p.mo.Vel dot u;            // >0 means moving toward the anchor
    if (radial < 0)                            // moving away = stretching the rope
        p.mo.Vel -= u * radial * (1.0 + BOUNCE);   // BOUNCE 0 = dead rope, 0.15 = elastic
}

// 3. no damping term is needed: airborne friction is skipped, p_mobj.cpp:2447-2463
// 4. gravity is already supplied by the engine in P_ZMovement (p_mobj.cpp:2967-2971) -
//    do NOT set bNOGRAVITY for a swing, gravity IS the pendulum's driver
// 5. reel while a button is held:
if (reelIn)  L = max(L - REEL_RATE, MIN_LEN);
if (reelOut) L = min(L + REEL_RATE, MAX_LEN);
p.keepmomentum = true;                   // player.zs:3108 - survives any ground brush
```

That is the whole pendulum. Projecting out the outward radial component is *exactly* what a taut
inextensible rope does; the remaining velocity is tangential by construction, and gravity supplies
the acceleration next tic. There is no need to compute an angle, a period or a restoring force.

**Which engine call applies the resulting velocity.** None — you write `Actor.Vel` (`actor.zs:112`)
and the engine integrates it in `P_XYMovement` / `P_ZMovement` during `RunThinkers`
(`p_tick.cpp:172`, dispatched at `p_mobj.cpp:4344` and `:4372`). Do **not** use
`A_ChangeVelocity`: gated off by `vr_recoil` (`actions.zs:77`, cvar `p_actionfunctions.cpp:90`).
`Thrust()` (`actor.zs:860`) is ungated and fine for one-shot impulses but it is XY-only, so it is
no use for the Z part of a swing.

**Momentum carry on release.** Stop running the constraint. That's it — velocity is already
tangential, friction does not apply in the air (`p_mobj.cpp:2447-2463`), and the only thing that
can take it away is landing, which `keepmomentum` already covers. Keep setting
`p.keepmomentum = true` for the whole coast; the engine clears it itself once the player is nearly
stopped (`player.zs:1385-1388`). Leave `bNOFRICTION` on for a few tics after landing if you want a
skid, then clear it.

Speed is capped at 30 units/tic by `MAXMOVE` (`p_local.h:61`, applied `p_mobj.cpp:2100-2101`).
A long rope from a high ceiling will hit that cap at the bottom of the arc. Tune the rope length
and gravity so the *good* swing lands around 22-26/tic, leaving headroom — otherwise every swing
feels identical because they all clip.

**Release timing window.** You sample at 35 Hz (baseline §3), so a release decision has 28.6 ms of
granularity and the player will systematically release late. Buffer the last 4 tics of
`p.mo.Vel` in a ring; on release, apply the **largest-magnitude** entry from the last 3 tics
(~85 ms of grace). This costs nothing and turns "I let go at the right moment and got nothing"
into a mechanic that rewards intent rather than reflex sampling.

**Rope wrapping around corners (Worms 2 ninja rope).** Keep a stack of pivots. Each tic, run a
`LineTracer.Trace` (`doombase.zs:309`) from the current pivot to `P`. If it is blocked by a wall:
take `d.HitLine.v1` / `.v2` (`mapdata.zs:211`), pick the endpoint nearest the blocked segment,
offset it along the reconstructed wall normal (§3.1) by ~2 units, push it as the new pivot, and
set `L -= |previousPivot - newPivot|`. To unwrap, test each tic whether the top pivot has gone
convex (the angle from pivot-below → pivot → player exceeds 180° on the far side) and pop it,
restoring the length. This needs only `HitLine`, `v1`, `v2`, `LineSide` and `delta` — all cited
above. It is real work but it is unblocked.

**Feel knobs.** `BOUNCE` (0-0.2), `MIN_LEN` / `MAX_LEN`, `REEL_RATE`, gravity multiplier on the
pawn while swinging, release grace window (tics), whether reeling in during the upswing is allowed
(it is the pumping mechanic — allow it, it is what makes a ninja rope expressive), and a small
air-control thrust allowance.

**Label: BUILDABLE. Cost: M** for the single-anchor pendulum; **L** with corner wrapping.

---

### 4.3 Whip — crack, wrap-and-pull, swing from points

**What the player physically does.** Three distinct gestures on one tool:
1. **Crack** — a fast forward flick of the hand, no button, or button + flick.
2. **Wrap-and-pull** — aim at an enemy or item, press, the lash coils and yanks.
3. **Swing** — aim up at a beam, press, you're on a pendulum (4.2 verbatim).

**How script detects it.** The whip is a weapon; which hand it is in determines which pose fields
you read (`AttackPos`/`AttackAngle` for mainhand, `OffhandPos`/`OffhandAngle` for offhand —
`actor.zs:268-275`).

The crack is a gesture and there are **no hand velocity fields** (baseline §1b). Store the last 4
tics of hand position and difference:

```
Vector3 handNow  = p.mo.AttackPos;          // or OffhandPos
Vector3 delta    = handNow - handPrev[0];   // one tic, ~28.6 ms
double  speed    = delta.Length() / dt;     // map units / second
Vector3 dir      = p.mo.AttackDir(p.mo, p.mo.Angle, p.mo.Pitch);   // actor.zs:858
double  forward  = delta dot dir;           // is the motion along the whip, or sideways?
```

Fire the crack when `speed > CRACK_SPEED` **and** `forward / delta.Length() > 0.6` **and** the
previous tic was below threshold (rising edge, so one crack per flick). At 35 Hz a real flick is
2-4 samples — so threshold on *this tic's* displacement, not on a smoothed average, and accept
that you cannot see the arc shape. Also compare the two-tic displacement as a fallback, since a
genuinely fast flick can land its peak between samples.

**How it resolves.**

- **Crack:** `LineTrace` along the hand direction out to the whip's reach (~128-192 units) with
  `TRF_USEWEAPON [| TRF_ISOFFHAND] | TRF_ALLACTORS`. On `TRACE_HitActor`, `DamageMobj`. On
  `TRACE_HitWall`, play the crack sound and a dust puff at `d.HitLocation` — reading the surface
  from `d.HitTexture` lets you pick the right impact sound with no map data. Scale damage with
  `speed` so a lazy wave does nothing.
- **Wrap-and-pull:** identical to 4.1b. The mass test (`Actor.Mass`) decides whether the enemy
  comes to you or you go to the enemy, which is the whole Indiana Jones joke and it is free.
- **Swing:** identical to 4.2, with reach as the initial `L`.

**Feel knobs.** `CRACK_SPEED` threshold, forward-dot threshold, reach, damage-from-speed curve,
disarm chance on crack, pull strength, and the single most important one in VR: a **cooldown** of
~6 tics after a crack, or the 35 Hz sampling will re-trigger on the recovery stroke.

**Label: BUILDABLE. Cost: M.** The crack gesture alone is S; it is the three-modes-on-one-tool
state machine that costs.

---

### 4.4 Rope arrows — a persistent rope you then climb

**What the player physically does.** Shoots an arrow into a wooden surface high up. The arrow
plants; a rope unspools down to the floor and **stays there**. Later, the player walks to it,
grips, and climbs.

**How script detects it.** The arrow is an ordinary projectile. On impact, classify the surface:
`d.HitType == TRACE_HitWall` plus a texture-name test on `d.HitTexture` (`actor.zs:53`), or a
normal test from §3.1, or a `d.HitLine.special` test. Thief's own rule was "wood only" — reproduce
it with a mod-owned name list and `TexMan.GetName()`. **No map authoring, per the hard rule.**

On a valid plant, spawn a persistent `RopeAnchor` actor at `d.HitLocation`, offset ~2 units along
the surface normal, with `bNOGRAVITY`, `bNOINTERACTION`, `bNOBLOCKMAP` and no sprite. It stores
the anchor point and the rope's hanging direction (straight down, or along `-normal` for a
ceiling plant). It is the rope's identity for the rest of the level.

**How it resolves — rendering and grip.** Each tic, the anchor walks down its own length placing
rope segments. Cheapest: `Level.SpawnParticle` (`doombase.zs:560`) with a rope texture and a roll
matching the rope's screen angle, one particle per 8 units, lifetime 2 tics, respawned each tic.
That costs no thinkers and no blockmap. If you need the rope to be lit and sorted like world
geometry, use real sprite actors with `bNOINTERACTION` and `SetOrigin(..., true)` instead, and
pool them.

Grip detection is a proximity test, not a trace: on each tic where a hand's grip latch
(`player.attackdown` / `player.ohattackdown`, `player.zs:3061-3062`) rises, test every live
`RopeAnchor` for whether that hand's world position (`AttackPos` / `OffhandPos`) is within
`GRIP_RADIUS` (~14 units) of the rope's line segment. Point-to-segment distance, one dot product.
No `BlockThingsIterator` needed unless you have very many ropes.

While any hand grips: `bNOGRAVITY = true` on the pawn, `Vel = (0,0,0)` — the player hangs. Then 4.5.

**Feel knobs.** Which surfaces accept an arrow, rope length, unspool speed and sound, `GRIP_RADIUS`,
whether the rope can be cut or retrieved, and whether letting go mid-climb restores gravity
instantly or after a few tics of grace.

**Label: BUILDABLE. Cost: M.** The rope *visual* is the fiddly part, not the logic.

---

### 4.5 Hand-over-hand climbing

**The brief says: you have one readable hand, confront it. You have two.** See §2.2. This mechanic
is unblocked, and it is the clearest payoff from that correction, so I want to be explicit about
what is and is not true:

- **True:** two full 6-DoF hand poses, `AttackPos/Angle/Pitch/Roll` (`actor.zs:268-271`) and
  `OffhandPos/Angle/Pitch/Roll` (`actor.zs:272-275`), both written from real controller transforms
  per frame on OpenXR (`hw_openxrdevice.cpp:585-609`) and OpenVR (`gl_openvr.cpp:2525-2554`).
- **True:** two independent grip latches, `player.attackdown` and `player.ohattackdown`
  (`player.zs:583-584`), because the offhand weapon system is real (baseline §2).
- **Still true, and it is the real constraint:** **no hand velocity fields** (baseline §1b) and
  **35 Hz sampling** (baseline §3). A climbing pull is maybe 6-10 tics long, so you get 6-10
  samples of a 40 cm motion. That is plenty — *if* you integrate displacement rather than
  thresholding on instantaneous speed.
- **Caveat:** in flat/non-VR mode both hands collapse to the viewpoint (`hw_vrmodes.cpp:442-444`),
  so guard on `p.mo.OverrideAttackPosDir` (`actor.zs:276`) and fall back to a button-held climb.

**What the player physically does.** Grabs the rope with one hand, pulls down, grabs higher with
the other hand, releases the first, repeats.

**The per-tic maths.** Store each hand's previous position. For each hand that is currently
gripping the rope:

```
Vector3 handNow  = (hand == MAIN) ? p.mo.AttackPos : p.mo.OffhandPos;
Vector3 handDelta = handNow - handPrev[hand];
double  along     = -(handDelta dot ropeUp);   // ropeUp = unit vector up the rope
                                               // pulling DOWN the rope = positive 'along'
if (along > 0)
    climbVel += ropeUp * along * PULL_GAIN;    // you go up by what you pulled down
handPrev[hand] = handNow;
```

Then once per tic:

```
p.mo.bNOGRAVITY = true;                        // hanging
p.mo.Vel = climbVel;                           // actor.zs:112
climbVel *= HAND_DECAY;                        // 0.85 - coasts between pulls, reads as weight
// keep the player pinned to the rope line laterally:
Vector3 toLine = ClosestPointOnRope(p.mo.pos) - p.mo.pos;
p.mo.Vel.XY += toLine.XY * LATERAL_SNAP;       // LATERAL_SNAP ~ 0.3
```

The hand positions are already **body-relative** — look at `hw_openxrdevice.cpp:593-595`:
`AttackPos.X = player->mo->X() - (weaponoffset[0] * vr_vunits_per_meter)`. The pawn's own position
is the origin. So when you move the pawn upward in response to a pull, the hand positions move
with it, and the *next* tic's delta correctly measures only new arm motion. There is no feedback
loop to cancel out. This is the single property that makes the whole mechanic work and it is free.

If neither hand grips, clear `bNOGRAVITY` and the player falls with `climbVel` intact — which is
the "let go at the top and launch" moment, and it costs nothing extra.

**Feel knobs.** `PULL_GAIN` (map units of climb per map unit of hand pull; start at 1.0 for 1:1,
go to 1.4 for heroic), `HAND_DECAY`, `LATERAL_SNAP`, `GRIP_RADIUS`, a minimum per-tic displacement
to count as a pull (kills tracking jitter), whether both hands gripping doubles the rate or
averages it (**average it** — doubling makes frantic shaking the optimal strategy), and a
maximum climb speed.

**Label: BUILDABLE. Cost: M.** Would have been WALL under the baseline's one-hand premise.

---

### 4.6 Rocket Jockey — rope to a moving actor, at speed

This is the hardest case in the topic and it deserves the space. The player is themself moving fast
along the ground; the rope is anchored to another moving actor; the rope is the *steering*, not a
climbing aid. Everything about it fights this engine's VR comfort code.

**What the player physically does.** Moving at speed, fires a rope at a passing object — a moving
enemy, a vehicle, a pillar. The rope goes taut. Instead of stopping, they get slung into an arc
around it, trading the rope's length for turn radius. Reeling in tightens the turn. Letting go at
the right instant slingshots them out at a speed they could never have reached by moving.

**How script detects it.** Hook fire as 4.1, but store the **actor reference**, not the point:

```
ropeAnchorActor = d.HitActor;
ropeLocalOffset = Level.Vec3Diff(d.HitActor.pos, d.HitLocation);   // where on it you hooked
```

Re-derive `A` every tic from the live actor so the anchor moves with it:

```
if (!ropeAnchorActor) { Detach(); return; }      // it died, or was destroyed
Vector3 A = ropeAnchorActor.pos + ropeLocalOffset;
```

(If you want the hook point to rotate with the anchor, rotate `ropeLocalOffset` by the anchor's
yaw delta each tic. For a monster this is usually not worth it; for a vehicle-like actor it is.)

**The per-tic maths.** Identical constraint to 4.2 but in the anchor's **relative** frame, which is
the whole difference between a swing and a jockey:

```
Vector3 r = Level.Vec3Diff(P, A);
double  d = r.Length();
Vector3 u = r / d;                               // player -> anchor

if (d > L)
{
    // --- work in the anchor's frame ---
    Vector3 vRel = p.mo.Vel - ropeAnchorActor.Vel;
    double  radial = vRel dot u;                 // <0 = separating = stretching

    if (radial < 0)
    {
        // mass split: how much of the correction each end absorbs
        double mA = ropeAnchorActor.Mass;
        double mP = p.mo.Mass;
        double shareP = mA / (mA + mP);          // heavy anchor -> player absorbs nearly all
        double shareA = 1.0 - shareP;

        Vector3 impulse = u * radial;            // what must be removed, in total
        p.mo.Vel            -= impulse * shareP * (1.0 + BOUNCE);
        ropeAnchorActor.Vel += impulse * shareA * (1.0 + BOUNCE);   // <- the Jockey reaction

        tension = abs(radial) / dt;              // units/sec^2 equivalent; see below
    }

    // positional correction, split the same way
    Vector3 corr = u * (d - L);
    p.mo.SetOrigin(p.mo.pos + corr * shareP, true);
    if (shareA > 0.01)
        ropeAnchorActor.SetOrigin(ropeAnchorActor.pos - corr * shareA, true);
}
else tension = 0;

// ground-speed survival - THE make-or-break pair:
p.keepmomentum   = true;      // player.zs:3108, defeats the wipe at player.zs:1374-1379
p.mo.bNOFRICTION = true;      // thingdef_data.cpp:324, defeats friction at p_mobj.cpp:2529-2531

if (reelIn) L = max(L - REEL_RATE, MIN_LEN);     // tighten the arc
if (tension > BREAK_FORCE) Detach();             // the rope snaps
```

**Why the two flagged lines are make-or-break, and no other mechanic here needs them.** Rocket
Jockey is played *on the ground*. That means `player.onground` is true
(`player.zs:1370`), which means `vel.XY *= 0.0001` fires **every single tic**
(`player.zs:1374-1379`) unless `keepmomentum` is set; and it means ground friction applies
(`p_mobj.cpp:2529-2531`, `ORIG_FRICTION` 0.90625, ~9% bled per tic, half-life ~7 tics) unless
`bNOFRICTION` is set. Swinging, climbing and hookshot-to-ceiling are all airborne and dodge both.
This one does not. **If a build of this feels like the rope has no effect, check these two lines
first** — the symptom is "the constraint maths is obviously running and the player still crawls."

**Tension, and what you can do with it.** `tension` is the magnitude of the impulse the rope had to
remove, per second. Use it for:
- **Snap:** `tension > BREAK_FORCE` → detach. This is the risk that makes tightening the rope a
  decision.
- **Rope stretch visual:** scale the segment sprite spacing or tint by tension.
- **Anchor reaction:** already in the maths above — a light anchor gets dragged, a heavy one does
  not. This falls straight out of `Actor.Mass` with no special-casing.
- **Haptics:** this is where you would want it, and **you cannot have it** — see §5.

**Being slung.** Nothing extra is required. Once the radial component is removed each tic, what
remains is tangential in the anchor's frame plus the anchor's own velocity. That sum *is* the
Rocket Jockey arc. Reeling in reduces `L`, which shortens the radius, which (angular momentum being
conserved by the projection) increases angular speed — the classic tighten-to-accelerate. You get
that for free from the same four lines; you do not need to model it.

**Cutting loose.** Stop the constraint and clear `bNOFRICTION` after ~10 tics. The player departs at
tangential + anchor velocity, which can exceed what they could ever reach under their own power —
until `MAXMOVE` clips it at 30 units/tic in `P_XYMovement` (`p_local.h:61`, `p_mobj.cpp:2100-2101`).
**Budget for that ceiling.** Tune the arc so a *good* release lands near 26-28 and a sloppy one
lands near 16, or every release will clip to the same speed and the skill expression disappears.

**Known rough edges, stated rather than hidden:**
- `SetOrigin(pos, true)` (`actor.zs:753`) does not re-solve collision. Applying a positional
  correction into a wall will put the pawn inside geometry. Guard every correction with
  `CheckMove` (`actor.zs:832`) and fall back to velocity-only correction when the move is blocked.
  At Rocket Jockey speeds against level walls this will happen constantly; it is the main
  implementation risk in this design.
- Writing the anchor's `Vel` fights its AI, same as 4.1b. Park it in a no-action state.
- `d.NumPortals > 0` (`actor.zs:57`) means the rope crossed a portal. Refuse the attach. A
  positional correction across a portal will teleport the player somewhere surprising.

**Feel knobs.** `BOUNCE`, `BREAK_FORCE`, `REEL_RATE`, `MIN_LEN`, the mass split curve (clamp
`shareA` to 0 for world-scale anchors so pillars never move), the post-release `bNOFRICTION`
window, and whether the player can steer with the stick while roped (allow a little — pure physics
feels unresponsive in VR).

**Label: BUILDABLE. Cost: L.** Not because the maths is hard — it is the same twenty lines as 4.2 —
but because collision-correct positional correction at speed, plus AI suppression on a moving
anchor, plus tuning against `MAXMOVE`, is genuinely a few weeks of iteration.

---

## 5. Walls

1. **Haptics.** `VRMode::Vibrate(duration, channel, intensity)` exists at
   `src/common/rendering/hwrenderer/data/hw_vrmodes.h:94`, and `VR_HapticEvent` at `:97`, but
   there are **zero** references to haptic/rumble/vibrate anywhere in `wadsrc/static/zscript`
   (baseline §4, re-confirmed). A rope with no tension feedback in the hand is a rope you have to
   *look* at to understand. **WALL. Missing piece: a ZScript-callable thunk for
   `VRMode::Vibrate`.** This is the single highest-value engine addition for this entire topic, and
   it is general — weapons, impacts, pickups and ropes all want it, so it has a home.
   **Cost to add, if it were ever authorized: S.**
2. **Rope rendering.** No line, beam or spline primitive reaches ZScript. The best available is a
   per-tic chain of particles (`Level.SpawnParticle`, `doombase.zs:560`) or roll-sprites. For a
   taut straight rope this is fine; for a slack catenary with 20 segments at 35 Hz it will look
   steppy. **HACK, cost M.** Missing piece: a textured quad-strip primitive.
3. **Multiplayer.** Hand poses are written from the renderer for `r_viewpoint.camera->player` only
   (`hw_openxrdevice.cpp:571`, `gl_openvr.cpp:2525`). Remote players' hands do not exist in the
   playsim, so no hand-driven rope mechanic replicates. **WALL for MP. Missing piece: hand pose in
   the netcode / usercmd.** Single-player and local play are unaffected.
4. **`A_ChangeVelocity` on the player.** Gated off by `vr_recoil` (default false,
   `p_actionfunctions.cpp:90`; gate at `actions.zs:77`, and `A_Recoil` at
   `p_actionfunctions.cpp:1294`). **Not a wall — a trap with a trivial workaround** (write `Vel`).
   Listed here because the brief named it as an API to use and it is not one.
5. **No surface normal in `FLineTraceData`.** Reconstructible exactly (§3.1), so **not a wall**.
   The residual gap is 3D-midtex and polyobject surfaces, where `HitLine` is a real line but the
   geometric normal may not match the rendered surface. Minor; refuse those anchors.
6. **35 Hz gesture sampling** (baseline §3). Affects the whip crack and hand-over-hand. Mitigated
   by integrating displacement instead of thresholding speed, and by a multi-tic release grace
   window (4.2). **Not a wall; a design tax paid in every gesture design.**
7. **Portals.** `d.NumPortals` (`actor.zs:57`) tells you a rope crossed one, and `Level.Vec3Diff`
   (`doombase.zs:523`) is portal-aware, but `SetOrigin` corrections across a portal are not safe.
   **Scope it out: refuse to anchor when `NumPortals > 0`.** Cheap, honest, and nobody will notice.

---

## 6. Smallest prototype

One sitting. Proves the three load-bearing claims — the trace reaches the controller, the normal
reconstructs, and the momentum survives — and nothing else.

1. **One offhand weapon**, `bOffhandWeapon` (`player.zs:1977-1998`), one Fire state.
2. **Fire state does one call:**
   ```
   LineTrace(invoker.owner.Angle, 2048, invoker.owner.Pitch,
             TRF_USEWEAPON|TRF_ISOFFHAND|TRF_SOLIDACTORS|TRF_BLOCKUSE|TRF_NOSKY, data: d);
   ```
   Print `d.HitType`, `d.HitLocation`, and the reconstructed normal from §3.1.
   **Checkpoint 1:** the printed `HitLocation` must move when you move the controller *without*
   turning your head. If it tracks the head instead, `OverrideAttackPosDir` is false and you are
   in flat mode.
   **Checkpoint 2:** the normal on a flat floor must read `(0,0,1)` and on a wall must have
   `z == 0` and point back toward you. Verify the sign once, here, not later.
3. **Store the anchor** in a `StaticEventHandler`, plus `L = d.Distance`.
4. **In `WorldTick()`**, run exactly the 4.2 block: `Vec3Diff`, project out the outward radial,
   `SetOrigin` the stretch away, and `p.keepmomentum = true`.
5. **Draw nothing.** Place one `Level.SpawnParticle` at the midpoint so you can see the anchor.
6. **Checkpoint 3:** hook a ceiling, step off a ledge, swing, release at the bottom, and watch
   whether you *keep going* after landing. If you stop dead the instant you touch the floor,
   `keepmomentum` is not being set every tic — re-read `player.zs:1374-1379`.

If all three checkpoints pass, every mechanic in §4 is downhill from there, including Rocket Jockey.

---

## 7. Open questions

1. **`SecPlane.Normal` polarity.** I did not find a line stating whether the floor normal points up
   or down in this build. The getter is `mapdata.zs:265` and the C++ is `secplane_t`; the
   convention is consistent but the sign should be confirmed empirically once (prototype
   checkpoint 2) before any tuning number is written against it.
2. **Does the OpenXR `AttackPos` Z term behave while crouching?** `hw_openxrdevice.cpp:595` subtracts
   `getDoomPlayerHeightWithoutCrouch(player)` with a comment about water. The hand height may shift
   relative to the pawn when `vr_crouch_use_button` is off and `crouchfactor` is being driven by HMD
   height (`hw_openxrdevice.cpp:574-579`). For hand-over-hand this matters: a crouch could read as a
   pull. **Measure before shipping 4.5.**
3. **`MAXMOVE` and `Vel.Z`.** The 30/tic clamp at `p_mobj.cpp:2100-2101` is horizontal only, and I
   did not find a corresponding vertical clamp in `P_ZMovement`. If Z is genuinely unclamped, a
   long vertical hookshot reel could reach uncomfortable speeds in VR. Worth a deliberate cap in
   the mod regardless of what the engine does.
4. **Thinker ordering between a rope handler and its anchor actor.** `WorldTick` is deterministic
   relative to the player (`p_tick.cpp:170` before `:172`), but if you instead drive the rope from
   an actor's `Tick`, whether it runs before or after the anchor's own `Tick` depends on thinker
   list order. **Use `WorldTick` for anything that reads an anchor's `Vel`** — 4.6 depends on this.
5. **`vr_momentum`.** Setting this cvar to true would make `keepmomentum` unnecessary
   (`player.zs:1374`), but it changes movement feel globally for every mod, which is exactly the
   kind of change the baseline warns against. I have assumed the mod must work with it **off**.
   Worth confirming with the owner whether a rope mod may require it on.
6. **Haptics reachability.** Baseline §4 says to find the caller of `VR_HapticEvent` and report
   whether a mod can provoke a hardcoded event (a damage/pickup/fire rumble) as a HACK. I did not
   trace its callers — out of scope for this topic's budget, and the answer does not change any
   label above. Flagging it as the one unfinished thread.
