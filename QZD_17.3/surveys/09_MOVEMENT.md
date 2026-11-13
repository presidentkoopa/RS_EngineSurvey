# 09 — Advanced movement mechanics as a MOD

Engine surveyed: tag `qzd17.3` of `github.com/emawind84/gzdoom` (QuestZDoom PC edition).
All citations are repo-relative at that tag. Sprite-layer only: nothing below authors a map,
a linedef, a sector or a thing. Walls are found at runtime by trace or by block iterator.

Builds on `_BASELINE_FACTS.md` as corrected: **both hands are fully tracked** (main hand via
`Actor.AttackPos/AttackAngle/AttackPitch/AttackRoll`, `wadsrc/static/zscript/actors/actor.zs:268-271`),
there are still no hand or head *velocity* fields, and script still samples at 35 Hz.
Surface-normal reconstruction is shared with `05_GRAPPLE_ROPES.md` §3.1 — this report reuses that
function rather than defining a second one.

---

## 0. Read this first — two traps that decide the whole topic

**Trap 1 — the VR momentum kill deletes your velocity every tic.** `MovePlayer` multiplies
`vel.XY` by `0.0001` on *every tic the player is on the ground*, unless `vr_momentum` or
`player.keepmomentum` is set — `wadsrc/static/zscript/actors/player/player.zs:1373-1383`.
`vr_momentum` defaults **false** (`src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:120`).
`player.keepmomentum` is writable (`player.zs:3108`, native `src/playsim/d_player.h:414`) and the
engine itself sets it whenever it wants velocity to survive (`wadsrc/static/zscript/actors/actions.zs:78`).
**Every slide, dash, backstep and momentum mechanic in this report dies silently without
`player.keepmomentum = true` re-asserted each tic.** This single line is the difference between
this topic being BUILDABLE and appearing impossible. The engine clears the flag for you once speed
drops below `vr_momentum_threshold` (default 1.0) — `player.zs:1385-1387` — which is a convenient
natural end-of-dash, not something you must undo.

**Trap 2 — `A_ChangeVelocity` on the player is a no-op by default.** The brief named it as an API;
it is not one. `wadsrc/static/zscript/actors/actions.zs:63-79` short-circuits with
`if (!vr_recoil) return;` for the player, and `vr_recoil` defaults **false**
(`src/playsim/p_actionfunctions.cpp:90`). **Write `Actor.Vel` directly** (`actor.zs:112`).
Any flat-screen movement code ported in will do nothing in a default install and give no error.

---

## 1. Verdict

The whole player movement loop — `PlayerThink`, `MovePlayer`, `CheckJump`, `CheckCrouch`,
`CheckMoveUpDown`, `CalcHeight`, `ApplyAirControl` — is **ZScript, virtual, and overridable**
(`player.zs:1734`, `:1340`, `:1514`, `:1194`, `:1565`, `:604`, `:1328`), so every mechanic in this
brief is script work, not engine work. Past the two traps above, the ceiling is high: `LineTrace`
returns `HitLine` and `LineSide` (`wadsrc/static/zscript/actors/actor.zs:51`, `:59`) and
`Line.delta` is readable (`wadsrc/static/zscript/mapdata.zs:212`), so **a true wall normal is
reconstructible at runtime with no map changes** — that one fact is what makes wall running good
rather than a fudge. Gravity is per-actor and writable (`Actor.Gravity`, `actor.zs:176`, consumed
at `src/playsim/actorinlines.h:85-88`), friction is killable per-actor (`bNOFRICTION`,
`src/scripting/thingdef_data.cpp:324`), airborne friction is already effectively absent at default
settings (`src/playsim/p_mobj.cpp:2447-2463` with `airfriction == 1`, `src/g_level.cpp:1929-1931`),
and `P_SlideMove` already rewrites `Vel` to the wall-parallel component
(`src/playsim/p_map.cpp:3328-3329`), so holding a player against a wall while keeping tangential
speed is close to free. Comfort is the real constraint, and it is sharply limited by one hard WALL:
**script cannot change view pitch, roll or yaw in VR** — the HMD overwrites all three after
`R_SetupFrame` (`src/rendering/hwrenderer/hw_entrypoint.cpp:117` then `:143` →
`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:740-741`, `:750`). No dash roll, no
wall-run camera bank, and equally no accidental nausea from them. Haptics are the natural
substitute and they **are** reachable: `A_QuakeEx` (`wadsrc/static/zscript/actors/actor.zs:1284`)
routes intensity X and Y to the left and right controllers independently
(`src/rendering/r_utility.cpp:1085-1091`), while the camera shake that normally accompanies a quake
is off by default in VR (`r_quakeintensity` is 0.0f, "Defaulted to 0 for VR",
`src/rendering/r_utility.cpp:116`) — per-hand rumble with no view motion, which is exactly the
channel W7 took away. That is a HACK rather than a clean API (W2), but it works today. With it,
plus the flat full-screen blend `player.BlendR/G/B/A` (`player.zs:3120-3123`, drawn per eye at
`hw_entrypoint.cpp:200-201`), camera *position* (which script can still move), sound and the weapon
psprite, every mechanic can be given real impact and a blink/fade comfort variant without ever
touching the view.

---

## 2. What already exists

**The player think path is ZScript and virtual.**

| Thing | Where |
|---|---|
| `PlayerPawn.PlayerThink()` | `wadsrc/static/zscript/actors/player/player.zs:1734` |
| `HandleMovement()` | `player.zs:1612` (calls MovePlayer, CheckJump, CheckMoveUpDown) |
| `MovePlayer()` | `player.zs:1340` |
| `CheckJump()` | `player.zs:1514` |
| `CheckMoveUpDown()` | `player.zs:1565` |
| `CheckCrouch(bool)` | `player.zs:1194` |
| `CheckPitch()` | `player.zs:1460` |
| `CalcHeight()` | `player.zs:604` |
| `ApplyAirControl(out movefactor, out bobfactor)` | `player.zs:1328` |
| `Actor.FallAndSink(grav, oldfloorz)` — virtual, applies gravity to `Vel.Z` | `wadsrc/static/zscript/actors/actor.zs:522`; native body `src/playsim/p_mobj.cpp:2954-3029`; called from `P_ZMovement` at `src/playsim/p_mobj.cpp:2604` |

Native `P_PlayerThink` does almost nothing itself — it dispatches to the ZScript virtual
(`src/playsim/p_user.cpp:1356-1360`).

**QuestZDoom's VR momentum kill is already in `MovePlayer`.**

```
player.zs:1374   if (!vr_momentum && !player.keepmomentum
player.zs:1375   && player.onground && friction == ORIG_FRICTION)
player.zs:1377       vel.XY *= 0.0001;
player.zs:1378       Speed = Default.Speed * 8;
```
`vr_momentum` defaults false (`src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:120`).
So the stock VR feel is "no momentum, 8× per-tic speed, instant start/stop". `player.keepmomentum`
(`player.zs:3108`, native `src/playsim/d_player.h:414`) opts out of the kill — and the engine
clears it automatically once speed drops below `vr_momentum_threshold` (default 1.0),
`player.zs:1385-1387`. That auto-clear is the natural end of a dash or slide.

**Physical crouch is already engine-driven — in one of the two modes.** With
`vr_crouch_use_button` **false** the OpenXR layer writes `player->crouching = 10` and
`player->crouchfactor = hmdHeight / defaultViewHeight` every frame
(`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:574-579`), `CheckCrouch` honours the
`crouching == 10` sentinel (`player.zs:1209-1213`), and `PlayerPawn.Tick` scales the collision
volume by it (`Height = FullHeight * player.crouchfactor`, `player.zs:150-153`). So in that mode a
mod reading `player.crouchfactor` gets real physical crouch depth *and* a real ducked hitbox for
free. The cvar **defaults to true** (`src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:122`),
in which case physical crouch is a render-time eye offset only (`hw_openxrdevice.cpp:219`) and
`crouchfactor` never moves. See §3.5.

**Stick and head translation are separate.** In VR the stick goes through `VR_GetMove`'s
joystick pair into `cmd.forwardmove` / `cmd.sidemove` (`src/g_game.cpp:834-842`), while
*physical* head translation is applied by the renderer as a save/override/`P_XYMovement`/restore
on `player->mo->Vel` (`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:665-688`;
OpenVR equivalent `src/rendering/gl/stereo3d/gl_openvr.cpp:2613-2637`). So **`cmd.forwardmove`
in VR is pure stick** — leaning does not pollute it. That is what makes stick-flick detection viable.

**Already-shipped movement modifiers a mod can reuse:** `PowerSpeed`
(`wadsrc/static/zscript/actors/inventory/powerups.zs:1164`, applied via `GetSpeedFactor` at
`player.zs:1320`), `PowerHighJump` (`powerups.zs:1827`, applied at `player.zs:1537-1548`),
`Player.JumpZ` property (`player.zs:68`, default 8 at `:120`), `bFLY` / `bNOGRAVITY`
flight through `CheckMoveUpDown` (`player.zs:1577-1590`).

**Wall sliding already preserves tangential speed.** `P_XYMovement` routes blocked player moves
into `P_SlideMove` (`src/playsim/p_mobj.cpp:2277`, `:2281`), and `FSlide::SlideMove` writes the
clipped wall-parallel move straight back into `mo->Vel` (`src/playsim/p_map.cpp:3328-3329`).
A player with velocity pressed into a wall keeps the along-wall component with no script help.

---

## 3. Building blocks

### 3.1 Where to inject each tic

Per-tic order inside one game tic (`src/p_tick.cpp:165-172`):

1. `P_PlayerThink` for every player — `src/p_tick.cpp:168` → ZScript `PlayerThink`
   (`src/playsim/p_user.cpp:1356`). This is where `MovePlayer` zeroes velocity and
   `CheckJump` adds jump velocity.
2. `EventHandler.WorldTick` — `src/p_tick.cpp:170` (`wadsrc/static/zscript/events.zs:177`).
3. `Level->Thinkers.RunThinkers` — `src/p_tick.cpp:172` → `AActor::Tick`, which runs
   **`Inventory.DoEffect` on every carried item** (`src/playsim/p_mobj.cpp:4005-4017`,
   declared `wadsrc/static/zscript/actors/inventory/inventory.zs:982`) and *then*
   `P_XYMovement` (`src/playsim/p_mobj.cpp:4344`) and `P_ZMovement`
   (`src/playsim/p_mobj.cpp:4372`, `:4435`).

**Hook points, in order of preference:**

- **`Inventory.DoEffect()` on a carried controller item** — runs after `MovePlayer` has already
  applied its velocity kill, and before `P_XYMovement` consumes `Vel`. Velocity written here lands
  this same tic and cannot be clobbered by `MovePlayer`. Per-player, net-safe, ordered by inventory
  order. **This is the right hook for a mod that must coexist with arbitrary other mods**, because
  it needs no player-class replacement.
- **Overriding `PlayerPawn.MovePlayer` / `CheckJump` / `ApplyAirControl` / `FallAndSink`** — cleanest
  and most precise, but requires replacing the player class, which collides with any other mod that
  does the same.
- **`EventHandler.WorldTick`** (`wadsrc/static/zscript/events.zs:177`, ordering
  `src/p_tick.cpp:166-172`) — **the established injection point across this survey.** It runs after
  all player input processing and before `P_XYMovement` / `P_ZMovement`
  (`src/playsim/p_mobj.cpp:4344`, `:4372`), so writes to `Vel`, `bFly`, `bNoGravity` and `Gravity`
  land the same tic. Not per-player, so key everything on `players[i]` explicitly — and on the
  owner, per the netplay rule.
- **`PlayerPawn.TickPSprites()`** (called `player.zs:1796`) — the other clean hook, and the right
  one if the mechanic's visible component lives on the weapon psprite (which, given W7, is where
  most of the "feel" for these mechanics has to go).

### 3.2 Moving the player

| API | Where | Notes |
|---|---|---|
| `Actor.Vel` (vector3, writable) | `actor.zs:112` | The one reliable mover. Write it, then set `player.keepmomentum = true`. |
| `Actor.Thrust(speed, angle)` | `actor.zs:860` | **Adds `ThrustAngleOffset`** (`src/playsim/actor.h:1607-1613`), which the OpenVR layer sets from the offhand controller when `vr_move_use_offhand` (`src/rendering/gl/stereo3d/gl_openvr.cpp:2603-2606`) and which is **not exposed to ZScript** (no `DEFINE_FIELD`). So `Thrust` has an unreadable direction bias in VR. Do not use it for a directional dash. |
| `A_ChangeVelocity` | `wadsrc/static/zscript/actors/actions.zs:63-79` | **Trap 2 (§0). Unusable as a mover on the player.** It does show the correct idiom: it sets `player.keepmomentum = true` (`actions.zs:78`). |
| `Actor.TryMove(newpos, dropoff, missilecheck, tm)` | `actor.zs:831` | Collision-checked relocation; use for blink/teleport comfort variants. |
| `Actor.CheckMove(newpos, flags, tm)` | `actor.zs:832`, flags `PCM_DROPOFF/NOACTORS/NOLINES` at `constants.zs:545-547` | Probe a position without moving — ledge and clearance tests. |
| `Actor.SetOrigin(newpos, moving)` | `actor.zs:753` | Unchecked relocation. Only after a `CheckMove`. |
| `Actor.Vel3DFromAngle` | `src/playsim/actor.h:1620` (native) | |

**The hard speed cap: `MAXMOVE = 30` map units per tic** — `src/playsim/p_local.h:61`, applied in
`P_XYMovement` at `src/playsim/p_mobj.cpp:2065-2066` (quartered to 7.5 in water unless deeply
crouched). That is the ceiling on every dash, slide and wall run here: ~1050 units/sec, about 31
m/s at `vr_vunits_per_meter = 34` (`hw_vrmodes.cpp:94`). Writing a larger `Vel` does not produce a
larger move; it is clipped. Tune dash speeds against 30, not against a number that feels big.
Separately, `P_XYMovement` substeps any move larger than `radius - 1` (`p_mobj.cpp:2166-2180`) so
fast movement does not tunnel through lines — you do not need to substep yourself.

### 3.3 Gravity and friction — what a mod may change, and what is baked in

**Changeable:**

| Knob | Where | Effect |
|---|---|---|
| `Actor.Gravity` (double, writable) | `actor.zs:176`, default `1` at `actor.zs:432` | Per-actor multiplier. `GetGravity() = Level->gravity * Sector->gravity * Gravity * 0.00125` — `src/playsim/actorinlines.h:85-88`. Per-player, net-safe, instant. **The right lever for wall-run and hang-time.** |
| `bNOGRAVITY` | `MF_NOGRAVITY`, `src/scripting/thingdef_data.cpp:127` | Zeroes gravity entirely, but on a player above the floor it *also* enables FlyBob vertical bobbing and applies `FRICTION_FLY` (0.918) to `Vel.Z` — `src/playsim/p_mobj.cpp:2630-2645`. The bobbing is a VR nausea source. Prefer `Gravity = small` over `bNOGRAVITY`. |
| `bNOFRICTION` (`MF8_NOFRICTION`) | `src/scripting/thingdef_data.cpp:324` | Early-returns out of all ground friction, air friction and the stop-speed clamp — `src/playsim/p_mobj.cpp:2442-2444`. A momentum-preserving slide in one flag. Also suppresses water sink damping (`p_mobj.cpp:2646`) and icy bounce (`p_map.cpp:3037`). |
| `level.gravity`, `level.aircontrol`, `level.airfriction` | `wadsrc/static/zscript/doombase.zs:454-456` — all declared `play`, i.e. **writable** | Global, shared by every player. **Writing `level.aircontrol` does not recompute `airfriction`** — the native `AirControlChanged()` (`src/g_level.cpp:1927-1938`) is not exposed, so a mod must write `level.airfriction` itself with the same formula: `airfriction = (aircontrol <= 1/256) ? 1.0 : aircontrol * -0.0941 + 1.0004`. In multiplayer, don't. |
| `ApplyAirControl(out movefactor, out bobfactor)` override | `player.zs:1328-1332` | **Per-player air control with no global side effects.** The correct lever if you own the player class. |
| `Sector.friction`, `Sector.movefactor` | `wadsrc/static/zscript/mapdata.zs:370` | Writable, but this is a world-geometry edit in all but name. Out of scope under the hard rule. |
| `Player.JumpZ` property / `PowerHighJump.Strength` | `player.zs:68`, `powerups.zs:1827` | Jump impulse. |
| `player.keepmomentum` | `player.zs:3108` | Opts out of the VR velocity kill. **Side effect: in the non-kill branch `Speed = Default.Speed` instead of `Default.Speed * 8`** (`player.zs:1378` vs `:1382`), so stick walking is 8× weaker for as long as you hold it. Hold it only for the duration of a dash/slide. |

**Baked in — a mod cannot change these:**

- `ORIG_FRICTION = 0.90625`, `ORIG_FRICTION_FACTOR = 0.03125` — compile-time, `src/doomdef.h:274-275`,
  mirrored as ZScript consts at `actor.zs:88-89`. The *application* of friction in
  `P_XYMovement` (`src/playsim/p_mobj.cpp:2531-2533`) is native and has no virtual.
- The `STOPSPEED` stop clamp and `MinVel` zeroing — `src/playsim/p_mobj.cpp:2497-2512`, `:2546-2555`. Native.
  Only `bNOFRICTION` bypasses them.
- `FRICTION_FLY = 0.91797` — `src/doomdef.h:277`.
- The air-friction formula itself (`aircontrol * -0.0941 + 1.0004`) — `src/g_level.cpp:1936`. You can
  write the result, not change the curve.
- `MAXPLMOVE` clamping of `cmd.forwardmove`/`sidemove` — `src/g_game.cpp:863-870`. Script reads the
  clamped value.
- The per-frame head-translation `P_XYMovement` call in the renderer
  (`hw_openxrdevice.cpp:665-688`). It saves and restores `Vel`, so it will not eat your dash velocity,
  but it *will* keep shoving the player when they move their head — including while wall-running.
- `ThrustAngleOffset` — written by the renderer, no ZScript field.

### 3.4 Reading the wall — the trace API, and yes you get a normal

`Actor.LineTrace(angle, distance, pitch, flags, offsetz, offsetforward, offsetside, out FLineTraceData data)`
— `wadsrc/static/zscript/actors/actor.zs:805`, native `src/playsim/p_map.cpp:5165`,
thunk `src/scripting/vmthunks_actors.cpp:1272`.

`FLineTraceData` (`actor.zs:36-62`) gives you:

- `HitType` — `TRACE_HitWall` / `HitFloor` / `HitCeiling` / `HitActor` / `HasHitSky` (`actor.zs:38-47`)
- `HitLine` — **the actual `Line`** (`actor.zs:51`), populated at `src/playsim/p_map.cpp:5274`
- `LineSide` — 0 front / 1 back, set only on `TRACE_HitWall` (`actor.zs:58`, `p_map.cpp:5278-5280`)
- `LinePart` — upper(0) / middle(1) / lower(2) (`actor.zs:59`, `p_map.cpp:5281-5300`)
- `HitLocation`, `HitDir`, `Distance` (`actor.zs:55-57`)
- `HitSector`, `Hit3DFloor`, `HitTexture` (`actor.zs:52-54`)
- `SectorPlane` — 0 floor / 1 ceiling (`actor.zs:60`)

**It does not hand you a normal, and `HitDir` is not one** — it is the trace's own direction vector
(`src/playsim/p_map.cpp:5305`), so it points *into* the wall. **Use the reconstruction function
already written up in `05_GRAPPLE_ROPES.md` §3.1 rather than writing a second one**; it covers
floors and ceilings (`SecPlane.Normal`, `wadsrc/static/zscript/mapdata.zs:265`, via
`Sector.floorplane`/`.ceilingplane`, `mapdata.zs:404-405` — slope-correct) as well as walls.

The wall branch, which is the one this topic needs, is derived as follows. `Line` exposes `v1`,
`v2` (`mapdata.zs:211`) and `delta` (`mapdata.zs:212`). The engine's own side test is
`P_PointOnLineSidePrecise`: `(p.y - v1.y)*delta.x + (v1.x - p.x)*delta.y > eps` → back
(`src/playsim/p_maputl.h:42-45`). Front side is therefore the half-plane in direction
`(delta.y, -delta.x)`, and `Line.ESide { front=0, back=1 }` is `mapdata.zs:167-170`:

```
Vector2 d = data.HitLine.delta.Unit();
Vector2 n = (d.y, -d.x);                  // outward normal of the FRONT side
if (data.LineSide == Line.back) n = -n;   // n now points back toward the tracer
```

That is a geometry-exact wall normal, derived entirely at runtime. Doom walls are always vertical,
so there is no vertical component to recover and none is needed. Verify the sign empirically once
(§6 step 5) before trusting the polarity in shipped tuning.

**Richer alternative:** `LineTracer` (`wadsrc/static/zscript/doombase.zs:306-321`) with an overridable
`TraceCallback()` and a `TraceResults` struct carrying `Side`, `Tier`, `HitLine`, `HitPos`,
`Fraction` (`doombase.zs:278-304`). Use it when you want to skip non-solid hits (e.g. ignore
two-sided lines whose back sector is walkable) rather than stopping on the first thing.

**Cheaper alternative for wall-run, and the one to actually ship:** `BlockLinesIterator`
(`doombase.zs:229-238`) — `Create(origin, checkradius)` walks every line in the blockmap cells
around the player. One iteration per tic replaces an 8-trace fan, and gives `CurLine` directly,
so you get `delta`, `flags` (`ML_BLOCKING`, `ML_TWOSIDED`, `ML_BLOCKEVERYTHING` — `mapdata.zs:175-203`),
`frontsector`/`backsector` and `sidedef[]` for free. Compute perpendicular distance from the
player to each candidate line yourself; keep the nearest one whose normal opposes your velocity.

**Deciding a line is a runnable wall (no map data, all runtime):**

1. `ML_BLOCKING` set, or not `ML_TWOSIDED` → one-sided wall, always runnable.
2. If `ML_TWOSIDED`: compare `frontsector`/`backsector` floor and ceiling at the hit point via
   `Sector.floorplane.ZatPoint(xy)` / `ceilingplane.ZatPoint(xy)` (`SecPlane.ZatPoint`,
   `mapdata.zs:271`). If the opening is shorter than the player's height, it is a wall.
3. Verticality: Doom lines are always vertical, so any `TRACE_HitWall` is a vertical surface.
   There is no sloped-wall case to handle.
4. Sky check: pass `TRF_NOSKY` (`constants.zs:957`) so you don't try to run on a sky edge.

### 3.5 Reading VR input: body, button, stick

| Input | How | Reliability |
|---|---|---|
| **Button** | `player.cmd.buttons & BT_*` (`constants.zs:834-869`) with `player.oldbuttons` (`player.zs:3064`) for edges. `BT_USER1..4` (`:859-862`) are free; `BT_JUMP`, `BT_CROUCH`, `BT_RUN`, `BT_OFFHANDATTACK` (`:866`) are in use. | **Highest.** No false positives, no 35 Hz loss. Default every mechanic to a button. |
| **Stick flick** | `cmd.forwardmove` / `cmd.sidemove` — pure stick in VR (`src/g_game.cpp:834-842`), clamped to `MAXPLMOVE` (`:863-870`). Detect a double-tap: value crosses zero then exceeds ~80% of max within ~8 tics. | **Good — with one hard exception.** `vr_teleport` **hard-zeroes `side` and `forward`** (`src/g_game.cpp:844-847`), so every stick-flick mechanic is dead for players using comfort teleport locomotion. Detect that cvar and fall back to buttons, or the mechanic silently vanishes for exactly the players most likely to have it on. Otherwise clean, but a double-tap costs ~0.2 s of latency and players who ride the stick at full deflection will mis-fire it. Make threshold and window cvars. |
| **Physical crouch** | `player.crouchfactor` (`player.zs:3126`) when `vr_crouch_use_button` is false — the engine writes it from head height each frame (`hw_openxrdevice.cpp:574-579`). Or read `HmdPosition.y` directly (metres; `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:445`, raw tracking-space pose from `VR_SetHMDPosition`, `src/QzDoom/qzdoom_common.cpp:93-95`, fed from the raw HMD matrix at `src/rendering/gl/stereo3d/gl_openvr.cpp:2400`). | **Good for a held state, poor as a trigger.** Depth is reliable; *rate* of crouch is 2-3 samples at 35 Hz (BASELINE §3). Use "below X% of standing height" as a gate, never "crouched fast" as a trigger. See the collision caveat immediately below — it changes which mode you must ask the player to run. |
| **Lean** | Differencing `HmdPosition.xz` across tics, in metres. | **Weakest, and it fights the engine.** Head translation is already converted to movement every frame by the renderer's own `P_XYMovement` call (`hw_openxrdevice.cpp:665-688`), so a lean *already* moves the player; a lean gesture is read against a moving baseline. Standing height and play-space origin also vary per player, and nothing recentres `HmdPosition` for you. Use lean only as a **modifier on an already-committed mechanic** (which way you leave a wall run), never as the trigger. |
| **Main hand pose** | `AttackPos` / `AttackAngle` / `AttackPitch` / `AttackRoll`, `native readonly` at `wadsrc/static/zscript/actors/actor.zs:268-271`, thunks `src/scripting/vmthunks_actors.cpp:2187-2190`, natives `src/playsim/actor.h:1654-1657`. Filled from the real controller transform at `hw_openxrdevice.cpp:586-597` (OpenVR `gl_openvr.cpp:2525-2537`). Gate on `OverrideAttackPosDir` (`actor.zs:276`, set `hw_vrmodes.cpp:438`). | Both hands are fully tracked; BASELINE §1a has been corrected. For *locomotion* this changes little — the weapon hand is busy — but it means a two-hand gesture is available if a mechanic ever wants one. No velocity field either way (BASELINE §1b): difference positions yourself. |
| **Offhand motion** | `OffhandPos` differenced per tic. | Usable, 2-4 samples per gesture. Competes with weapon handling. Not recommended for locomotion. |

**Collision caveat for every physical-body mechanic — room-scale is XY-only.** The renderer's
positional-movement block moves the player in XY and then explicitly restores Z
(`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:674-686`, OpenVR
`src/rendering/gl/stereo3d/gl_openvr.cpp:2618-2632`). Head *height* reaches the game only as a
render-time eye offset (`hw_openxrdevice.cpp:219`) and, in physical-crouch mode, through
`crouchfactor`. Those are different things:

- **`vr_crouch_use_button` true (the default, `hw_vrmodes.cpp:122`)** — physically ducking changes
  the eye offset and nothing else. `crouchfactor` stays 1, so the collision volume does **not**
  duck. Physical crouch is view-only, and a slide keyed on it will never fire.
- **`vr_crouch_use_button` false** — the engine writes `crouchfactor` from head height every frame
  (`hw_openxrdevice.cpp:574-579`), and `PlayerPawn.Tick` sets `Height = FullHeight * crouchfactor`
  every tic (`player.zs:150-153`). **In this mode the collision volume does follow the head**,
  subject to `CanCrouch()` and `level.IsCrouchingAllowed()`.

So: `HmdPosition` makes lean and duck *detection* work in both modes, but a mechanic whose
resolution depends on a shorter player (sliding under something) requires
`vr_crouch_use_button 0`. State that requirement in the mod's options rather than assuming it.

**Calibration and units — get this wrong and every duck and lean threshold is wrong.**

- `HmdPosition` is **raw tracking-space metres**, not map units and not an offset from anything.
  A mod that wants "is the player crouching" must learn the standing height itself (running max
  over the first few seconds, or a cvar) — the engine's own code does exactly this hack
  (`hw_openxrdevice.cpp:157-171`).
- To convert to map units, horizontal is `× vr_vunits_per_meter` (**34**,
  `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:94`), but **vertical additionally divides by
  `level.pixelstretch`** (default 1.2) — see `getHmdAdjustedHeightInMapUnit`,
  `hw_openxrdevice.cpp:151-155`, and the weapon transform at `gl_openvr.cpp:1282-1292`.
  `pixelstretch` is readable from ZScript (`wadsrc/static/zscript/doombase.zs:479`).
  **Forget the divide and vertical head movement reads ~17% short**, so a duck threshold tuned in
  map units fires late or not at all. Horizontal lean is unaffected.
- `vr_height_adjust` is applied downstream at the use sites (`hw_openxrdevice.cpp:154`), **not** at
  the source, so `HmdPosition` as script sees it does *not* include it. Add it yourself if you are
  comparing against anything the engine computed.

### 3.6 Feedback channels available to script

- **Full-screen blend** — `player.BlendR/G/B/A` (`player.zs:3120-3123`, native
  `src/playsim/d_player.h:434`, thunk `src/playsim/p_user.cpp:1926`). Composited into
  `V_CalcBlend` (`src/rendering/2d/v_blend.cpp:370`) and drawn by `V_DrawBlend` **inside the
  per-eye loop** (`src/rendering/hwrenderer/hw_entrypoint.cpp:200-201`). So it works in VR.
  It is a flat full-screen `Dim` (`src/rendering/2d/v_blend.cpp:392-393`) — a fade, not a vignette.
- **View height** — `player.viewheight` / `player.deltaviewheight` (`player.zs:681-699`) feed
  `player.viewz`, which becomes the camera position (`src/rendering/r_utility.cpp:1023`) and
  then `CenterEyePos` (`:1236`). **Writing viewheight moves the VR camera vertically.** Real lever,
  real hazard.
- **View offset** — `Actor.SetViewPos(offset, flags)` (`actor.zs:1278`, flags `VPSF_*` at
  `constants.zs:431-434`), applied at `src/rendering/r_utility.cpp:648-673` before `CenterEyePos`
  is taken (`:1236`). The VR layer never rewrites `vp.Pos` — `updateHmdPose` touches only angles
  (`hw_openxrdevice.cpp:740-750`) — so **camera *position* offsets from script do reach the
  headset.** Comfort-hostile; off by default.
- **View angles — NOT available in VR. See W7.** `A_SetViewAngle` / `A_SetViewPitch` /
  `A_SetViewRoll` (`actor.zs:1275-1277`) are overwritten by the HMD pose. No dash roll, no
  wall-run bank. This is a constraint *and* a safety net: the most common VR-comfort mistake is
  not available to make.
- **Haptics — available as a HACK via `A_QuakeEx`, per-hand, with no view motion.**
  `A_QuakeEx(intensityX, intensityY, ...)` (`actor.zs:1284`) → `DEarthquake` → left/right
  controller (`src/rendering/r_utility.cpp:1085-1091`); view shake is separately gated on
  `r_quakeintensity`, which defaults to 0 in VR (`r_utility.cpp:116`, applied `:1065-1082`).
  **Always pass `damrad = 0` and `thrustMultiplier = 0`** or the quake will add randomised velocity
  to the player (`src/playsim/mapthinkers/a_quake.cpp:206-211`) and fight your own movement code.
  Never pass `QF_GROUNDONLY` — it disables the effect off the floor (`a_quake.cpp:354`). Full
  caveats in W2. This is the primary impact channel for every mechanic in §4; sound and the weapon
  psprite are the others.
- **No eye-space vignette.** Script 2D drawing in VR is composited as a world-locked quad at
  `vr_hud_distance`/`vr_hud_scale` (`hw_openxrdevice.cpp:299-301`), not as an eye overlay, so a
  tunnelling ring drawn in `RenderOverlay` would appear as a floating billboard. See §5.

---

## 4. Mechanic designs

Throughout: "controller item" means a `CustomInventory`/`Inventory` subclass given to the player on
spawn, whose `DoEffect()` is the per-tic hook (§3.1). Every design assumes
`player.keepmomentum = true` is re-asserted each tic while the mechanic is active, and relies on the
engine clearing it automatically below `vr_momentum_threshold` (`player.zs:1385-1387`).

---

### 4.1 Ground dash

**What the player does.** Presses a bound button (recommended), or double-flicks the movement stick
in the direction they want.

**How script detects it.** In `DoEffect`: rising edge of `cmd.buttons & BT_USER1` against
`player.oldbuttons`. Stick variant: track `cmd.sidemove`/`cmd.forwardmove`
(`src/g_game.cpp:834-842`); a dash fires when the magnitude goes below 20% of `MAXPLMOVE` and back
above 80% within 8 tics with the same sign.

**How it resolves.** Direction = the player's facing (`angle`) combined with the current
`cmd.forwardmove`/`sidemove` ratio, normalised — **not `Thrust`**, which carries the unreadable
`ThrustAngleOffset` (§3.2). Write `Vel.XY = dir * dashSpeed` directly, set
`player.keepmomentum = true`, set `bNOFRICTION` for `dashTics` so friction does not eat it
(`src/playsim/p_mobj.cpp:2442-2444`), then clear the flag and let the normal friction path
(`p_mobj.cpp:2531-2533`) decelerate the player back to walking. Gate on `player.onground`
(`player.zs:1369`) and a cooldown counter.

**Feel tuning knobs.** `dashSpeed`, `dashTics`, cooldown, whether `bNOFRICTION` holds for the whole
dash or releases at 60%, whether direction locks at the start or steers during.

**Haptic signature.** Both hands, one sharp tic: `A_QuakeEx(0.6, 0.6, 0, 2, 0, 256, "", QF_SHAKEONLY, thrustMultiplier: 0)` on the frame the dash fires. `damrad = 0` and `thrustMultiplier = 0` are mandatory — see W2. This is what replaces the camera punch W7 denies you.

**Comfort variant.** *Blink dash.* Instead of velocity: `CheckMove` the destination
(`actor.zs:832`, with `PCM_DROPOFF` cleared so you don't blink off a ledge), raise
`player.BlendA` to 1.0 over 2 tics, `SetOrigin` (`actor.zs:753`) or `TryMove` (`actor.zs:831`) to
the destination, fade `BlendA` back over 3 tics. No optical flow at all, so no vestibular conflict.
The engine already ships a comfort teleport (`vr_teleport`,
`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:634-651`), so this is a familiar idiom
for the user. Make blink-vs-smooth a cvar and **default it to blink**; smooth dash must additionally
ramp velocity over 3-4 tics rather than stepping it, because the step is the nausea, not the speed.

**Label: BUILDABLE. Cost: S.**

---

### 4.2 Air dash

**What the player does.** Same input as the ground dash, while airborne.

**How script detects it.** Same edge detection, but gated on `!player.onground` (`player.zs:1369`)
and on an `airDashesUsed` counter that resets on landing.

**How it resolves.** Write `Vel.XY` directly as in 4.1. Additionally set `Vel.Z = 0` at the moment of
the dash to give it a clean horizontal read, then restore gravity. **Air behaves better than you
would expect:** `P_XYMovement` takes an early return for airborne actors that skips all ground
friction and the stop-speed clamp (`src/playsim/p_mobj.cpp:2447-2463`), and the only thing applied
on that path is `airfriction` — which is exactly `1.0` at the default `sv_aircontrol` of 1/256
(`src/g_level.cpp:1929-1931`, cvar `src/playsim/p_user.cpp:1072`). So an airborne dash is lossless
by default and needs no `bNOFRICTION` at all. Set `bNOFRICTION` only if the mod (or a map, via the
MAPINFO `aircontrol` key, `src/g_level.cpp:1849-1858`) raises `aircontrol`; a map that does will
otherwise bleed your dash and nothing else will.

**Feel tuning knobs.** Air dash count per jump, whether `Vel.Z` is zeroed, hang-time (set
`Gravity = 0.2` for the dash's duration via `actor.zs:176`), horizontal speed relative to the
ground dash.

**Haptic signature.** A lighter version of 4.1 on launch, and a second, heavier double-hand pulse on landing. Do **not** pass `QF_GROUNDONLY` — it would mute exactly the airborne part (W2).

**Comfort variant.** The blink variant is *worse* in the air than on the ground — a blink with no
ground plane for reference is badly disorienting. Comfort air dash instead = short, slow, and with
a hard duration cap (≤6 tics), plus a partial `BlendA` dim (0.35) for the duration so peripheral
flow is damped without a full blackout. Offer "air dash off" as a setting; it is the one mechanic
here some players will simply not tolerate.

**Label: BUILDABLE. Cost: S.**

---

### 4.3 Slide

**What the player does.** Crouches — physically, if `vr_crouch_use_button` is false, or on the
crouch button — while moving at speed.

**How script detects it.** In `DoEffect`: `player.crouchfactor < 0.7` (`player.zs:3126`, and only when `vr_crouch_use_button` is 0 — §3.5; in physical
crouch mode the engine writes it from head height every frame,
`hw_openxrdevice.cpp:574-579`) **and** `Vel.XY.Length() > slideEntrySpeed` **and**
`player.onground`. Physical crouch is the better input here precisely because it is a *held state*,
not a gesture — §3.5 says crouch depth is reliable where crouch *rate* is not, and a slide only
needs depth.

**How it resolves.** Set `bNOFRICTION` (`thingdef_data.cpp:324`) so `P_XYMovement`'s friction block
is skipped entirely (`p_mobj.cpp:2442-2444`), re-assert `player.keepmomentum = true` each tic, and
apply your own per-tic decay (`Vel.XY *= 0.97`) so you control the curve instead of inheriting
`ORIG_FRICTION`. Suppress steering by zeroing or scaling your own contribution — you cannot stop
`MovePlayer` from thrusting on `cmd.forwardmove` without overriding it (`player.zs:1401-1437`), so
either override `MovePlayer` (if you own the player class) or counter-thrust in `DoEffect`, which
runs after it. Exit when speed drops below threshold or the player stands up; clear `bNOFRICTION`.
Slide under low ceilings is free: `CheckCrouch` already refuses to stand up when
`pos.Z + height >= ceilingz` (`player.zs:1221`).

**Feel tuning knobs.** Entry speed threshold, decay rate, whether the slide gets a speed *boost* on
entry, steer authority during the slide, minimum slide duration.

**Haptic signature.** A low sustained buzz re-fired every tic for the slide's duration, tapering with speed — the 10 ms pulse length (W2) means per-tic re-firing is how you get sustain, and it lets the rumble decay with the slide instead of ending abruptly.

**Comfort variant.** **Do not dip the camera.** The obvious flourish — drop `player.viewheight`
(`player.zs:681`) for the slide — moves the VR camera (`r_utility.cpp:1023`, `:1236`) with no matching
head movement, and that is textbook vestibular conflict. In VR the player's head is *already* low,
because they physically crouched. Comfort slide = constant velocity decay (constant velocity is far
easier on the inner ear than acceleration), no viewheight change, no roll, and a cap on entry speed.
If you want the slide to read visually, do it on the weapon psprite, not the camera.

**Label: BUILDABLE. Cost: M.**

---

### 4.4 Backstep

**What the player does.** Button press, or back-flick on the stick.

**How script detects it.** Rising edge of a bound button, or `cmd.forwardmove` crossing strongly
negative twice within the double-tap window (§3.5).

**How it resolves.** Identical to 4.1 with direction = `angle + 180`, a shorter duration, and a
lower speed. Worth adding: a `CheckMove` probe at the destination with `PCM_DROPOFF` *not* set, so
a backstep cannot walk you off a ledge you cannot see.

**Feel tuning knobs.** Distance, duration, i-frames (if the mod has them), whether it cancels a
reload.

**Haptic signature.** One short double-hand pulse. Keep it lighter than the forward dash — a backstep should read as evasive, not as impact.

**Comfort variant.** Backward optical flow is the worst case in VR — the player sees the world
rushing toward them from a direction their head is not facing. **Default the backstep to blink**
(fade / `SetOrigin` / fade, as in 4.1), with smooth available but not the default. If smooth is
chosen, cap distance hard and keep duration ≥6 tics so the acceleration is gentle.

**Label: BUILDABLE. Cost: S.**

---

### 4.5 Wall run

**What the player does.** Jumps toward a wall and holds the movement stick along it (and, optionally,
leans into the wall to extend it).

**How script detects it.** Per tic in `DoEffect`, while `!player.onground`:

1. Build a `BlockLinesIterator` around the player (`doombase.zs:235`, radius ≈ `radius + 24`).
2. For each `CurLine`: reject if not blocking (§3.4 step 1-2 — `ML_BLOCKING` /
   `!ML_TWOSIDED` / opening shorter than the player). Compute the perpendicular distance from
   `pos.xy` to the line, and the normal `n = (delta.y, -delta.x).Unit()`, flipped so it points at
   the player.
3. Keep the nearest candidate within `wallGrabDist` whose normal satisfies
   `(Vel.XY.Unit() dot n) < -0.2` (you are moving into it) and
   `abs(Vel.XY.Unit() dot tangent) > 0.5` (you have along-wall speed, not just a face-plant).
4. Confirm with a single `LineTrace` along `-n` from the player's chest height
   (`actor.zs:805`, flags `TRF_NOSKY | TRF_THRUACTORS`), check `data.HitType == TRACE_HitWall`
   and `data.Distance` — this rejects lines the blockmap offered you through a corner.

**How it resolves.**

- **Hold against the wall:** add a small inward velocity each tic, `Vel.XY += -n * stickForce`.
  You do not need to zero the outward component yourself: `P_SlideMove` already rewrites `Vel` to
  the wall-parallel component when the move is blocked (`src/playsim/p_map.cpp:3328-3329`,
  reached from `src/playsim/p_mobj.cpp:2277`). This is why wall running is cheap on this engine.
- **Reduce gravity:** `Gravity = wallRunGravity` (e.g. 0.12) — `actor.zs:176`,
  consumed at `src/playsim/actorinlines.h:85-88`. Restore to `1` on exit. **Do not use
  `bNOGRAVITY`**: on a player above the floor it switches on FlyBob vertical bobbing and
  `FRICTION_FLY` damping of `Vel.Z` (`src/playsim/p_mobj.cpp:2630-2645`), and the bobbing is a
  nausea source you did not ask for.
- **Drive along the wall:** project the player's stick input onto the wall tangent and add it as
  velocity. `player.keepmomentum = true` each tic.
- **Exit jump:** on `BT_JUMP` rising edge, `Vel.XY = (tangent * exitForward + n * exitOut).Unit() * exitSpeed`
  and `Vel.Z = JumpZ * exitUp` — note `CheckJump` will not fire for you, because it requires
  `player.onground` (`player.zs:1532`), so you must do the whole jump yourself.
- **Exit conditions:** the trace stops finding the wall, a timer expires, the player lets go of
  the along-wall input, or `Vel.XY` tangential magnitude falls below a floor. Restore `Gravity = 1`
  and clear any flags on every exit path, including death and level change.

**Feel tuning knobs.** `wallRunGravity`, max duration, `stickForce`, whether tangential speed is
held constant or decays, minimum entry speed, re-grab cooldown on the same `Line` (store
`HitLine.Index()`, `mapdata.zs:239`, to stop re-attaching to the wall you just left).

**Caveat you cannot design away:** the renderer's per-frame head-translation
`P_XYMovement` (`hw_openxrdevice.cpp:665-688`) keeps pushing the player while they move their head,
including during the run. It saves and restores `Vel`, so it will not eat your velocity, but a
player who leans away from the wall will physically translate off it. Build the re-grab window to
tolerate a tic or two of lost contact.

**Haptic signature — this is where the per-hand split earns its keep.** Drive only the axis matching the wall side: `intensityX` for a wall on the left, `intensityY` for one on the right (`src/rendering/r_utility.cpp:1086-1091`). A continuous low buzz in the hand nearest the wall, re-fired per tic, tells the player which side they are attached to without a single degree of camera motion. On the exit jump, a sharp pulse in both hands. This substantially recovers what W7 took away, and it is the strongest argument for building the mechanic at all.

**Comfort variant.** This is the mechanic most likely to make people ill, because it is sustained
and lateral. Comfort wall run =

- **Camera bank is not on the table at all** (W7) — `A_SetViewRoll` does not reach the headset.
  The cinematic instinct to roll into the wall is simply unavailable, which here is a mercy. Sell
  the wall contact on the per-hand rumble above, the weapon psprite, and sound instead.
- **No `SetViewPos` lateral shove** (`actor.zs:1278`) — this one *does* reach the headset
  (§3.6) and would be the same mistake by another route.
- **Constant tangential speed, not accelerating.** Steady optical flow is tolerable; changing flow
  is not. Cap it well under `MAXMOVE = 30` (§3.2).
- **Hard duration cap** (2 s default) so the exposure is bounded.
- **Partial blend dim** at `BlendA ≈ 0.3` for the duration (`player.zs:3123`) to damp peripheral
  flow — the nearest thing to a vignette this engine gives script (§5).
- **Comfort-max setting: wall *hang* instead of wall *run*** — same detection, same gravity
  reduction, but tangential velocity is clamped to zero, so the player is stationary against the
  wall and only the exit jump moves them. Preserves the traversal (reach a ledge you otherwise
  can't) with almost no optical flow.

**Label: BUILDABLE. Cost: L.**

---

### 4.6 Double jump

**What the player does.** Presses jump again while airborne.

**How script detects it.** `cmd.buttons & BT_JUMP` rising edge against `player.oldbuttons`
(`player.zs:3064`) while `!player.onground` (`player.zs:1369`), with a `jumpsUsed` counter that
resets on landing. The stock `CheckJump` will ignore the press for you — it requires
`player.onground && player.jumpTics == 0` (`player.zs:1532`) — so there is no conflict to resolve.
One trap: `CheckCrouch` clears `BT_CROUCH` whenever `BT_JUMP` is held (`player.zs:1199-1202`),
so do not try to read a crouch+jump chord.

**How it resolves.** `Vel.Z = JumpZ * 35 / TICRATE * doubleJumpFactor` — mirroring the stock
computation at `player.zs:1535` — plus `player.jumpTics = -1` and `bOnMobj = false`
(`player.zs:1551-1553`) so the engine's own bookkeeping stays consistent. If you want air steering
on the second jump, that is the `ApplyAirControl` override (`player.zs:1328`) or a transient
`level.aircontrol`/`level.airfriction` pair (§3.3) — prefer the override.

**Feel tuning knobs.** `doubleJumpFactor`, number of extra jumps, whether the second jump zeroes
downward velocity first (it should — a double jump that cannot save a bad fall feels broken),
whether it adds horizontal velocity in the stick direction.

**Haptic signature.** A light both-hands blip on the second jump, and a firmer one on landing. Use it instead of the landing viewheight kick.

**Comfort variant.** The gentlest mechanic here: vertical motion with the head upright is the best
tolerated axis, and the arc is self-limiting. Comfort variant = cap the second jump's `Vel.Z` so
the total apex is not much higher than a single jump, and **do not add a landing viewheight kick**
(`player.deltaviewheight`, `player.zs:681`) — the stock engine already applies one on hard landings
(`player.zs:1476`), and in VR it is a camera move with no head move. If the mod ships a landing kick
at all, make it a weapon-psprite dip, not a camera dip.

**Label: BUILDABLE. Cost: S.**

---

### 4.7 Summary table

| Mechanic | Label | Cost | Best input | Comfort default |
|---|---|---|---|---|
| Ground dash | BUILDABLE | S | Button | Blink |
| Air dash | BUILDABLE | S | Button | Short + dim, off-switch |
| Slide | BUILDABLE | M | Physical crouch (held state) | No camera dip |
| Backstep | BUILDABLE | S | Button | Blink |
| Wall run | BUILDABLE | L | Auto-detect + stick | Wall *hang*, no roll |
| Double jump | BUILDABLE | S | Button | Capped apex, no landing kick |
| Eye-space comfort vignette | **WALL** | — | — | — (W1) |
| Per-hand haptic feedback via `A_QuakeEx` | **HACK** | S | — | On by default; degrades silently at `vr_quake_haptic_level 0` (W2) |
| Direct haptic API (`VRMode::Vibrate` binding) | **WALL** | — | — | — (W2) |
| Camera roll / pitch for feel | **WALL** | — | — | — (W7) |
| Offhand-grip / two-hand-brace gesture as a trigger | **WALL** | — | — | — (W9) |

---

## 5. Walls

**W1 — No eye-space comfort vignette.** The standard VR locomotion comfort tool is a tunnelling
vignette that darkens the periphery of each eye during motion. Script cannot draw one.
The only per-eye overlay a mod can drive is `player.BlendR/G/B/A`
(`player.zs:3120-3123` → `src/rendering/2d/v_blend.cpp:370` → `V_DrawBlend`,
`src/rendering/hwrenderer/hw_entrypoint.cpp:200-201`), and `V_DrawBlend` is a flat full-screen
`Dim` over the whole viewport (`src/rendering/2d/v_blend.cpp:392-393`) — uniform, no radial falloff.
Script 2D drawing (`RenderOverlay`) is not an alternative: in VR it is composited as a world-locked
quad at `vr_hud_distance` / `vr_hud_scale` (`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:299-301`),
so a drawn ring would float in the world rather than mask the eye.
**Exact missing piece:** a per-eye radial mask in the post-process chain with a ZScript-settable
strength — e.g. a `vr_vignette`-style uniform applied in `screen->PostProcessScene`
(`src/rendering/hwrenderer/hw_entrypoint.cpp:198`) and a `PlayerInfo` field to drive it.
Until then, the comfort fallback is uniform dimming via `BlendA`, which costs the whole image.

**W2 (revised) — Haptics have no direct ZScript binding, but `A_QuakeEx` is a working back door.
Not a WALL. HACK / S.**

I previously wrote that haptics were unreachable and that W7 + W2 together left only sound and the
weapon psprite. **That was wrong, and it was this report's headline, so read the correction first:**
an ordinary ZScript action function gets you independent per-hand rumble with no view motion.

**The route.** `A_QuakeEx` (`wadsrc/static/zscript/actors/actor.zs:1284`) spawns a `DEarthquake`
thinker; the view code collects its intensities each frame
(`DEarthquake::StaticGetQuakeIntensities`, `src/playsim/mapthinkers/a_quake.cpp:341-371`) and feeds
them straight to the controllers, **one hand per axis**:

```
src/rendering/r_utility.cpp:1085   if (vr_quake_haptic_level > 0.0) {
src/rendering/r_utility.cpp:1086       double left  = QuakePower(vr_quake_haptic_level, jiggers.Intensity.X, jiggers.Offset.X);
src/rendering/r_utility.cpp:1087       double right = QuakePower(vr_quake_haptic_level, jiggers.Intensity.Y, jiggers.Offset.Y);
src/rendering/r_utility.cpp:1090       vrmode->Vibrate(10, 0, (float)left);    // LEFT hand
src/rendering/r_utility.cpp:1091       vrmode->Vibrate(10, 1, (float)right);   // RIGHT hand
```

So `intensityX` → left hand, `intensityY` → right hand, independently.

**And the camera shake that would normally come with it is off by default in VR.** The view-offset
branches above it use `quakeFactor = r_quakeintensity` (`src/rendering/r_utility.cpp:1065`,
applied `:1067-1082`), and `r_quakeintensity` defaults to **0.0f** with the literal comment
"Defaulted to 0 for VR" (`src/rendering/r_utility.cpp:116`). The haptic branch reads
`vr_quake_haptic_level` instead (default 0.8, `hw_vrmodes.cpp:127`), so by default you get rumble
with **zero view motion** — which is exactly the channel W7 took away.

**What this buys this topic.** A dash can thump, a landing can thud, a wall run can buzz the hand
nearest the wall and leave the other silent. Per-mechanic use is in §4.

**Caveats, all of which must be carried into any implementation:**

- **Set `damrad = 0`.** Quake damage *and actor thrust* are gated on `dist < m_DamageRadius`
  (`a_quake.cpp:191`), and the thrust adds randomised velocity to the player
  (`a_quake.cpp:206-211`). For a movement mod that is catastrophic — your dash would be perturbed by
  your own feedback. `damrad = 0` disables both; `tremrad` must still be > 0 for the player to be
  inside the tremor (`a_quake.cpp:361`). Belt and braces: `QF_SHAKEONLY` (`constants.zs:677`)
  suppresses damage, and `thrustMultiplier = 0` suppresses thrust (default is **0.5**,
  `actor.zs:1284`).
- **Do not set `QF_GROUNDONLY`** (`constants.zs:675`) — it drops the quake when the victim is off
  the floor (`a_quake.cpp:354`), killing the effect for air dash, double jump and wall run,
  i.e. the three mechanics that need it most.
- **`vr_quake_haptic_level` is a user cvar** (`hw_vrmodes.cpp:127`) and the whole branch is skipped
  at 0 (`r_utility.cpp:1085`). Detect it and degrade; never assume rumble landed.
- **A user who has raised `r_quakeintensity` gets camera shake along with it.** That is their
  choice, but it means the mechanic is not unconditionally view-free. Worth saying in the mod's
  options text.
- **`Vibrate` is hard-coded to a 10 ms pulse** (`r_utility.cpp:1090-1091`), and the queued duration
  is decremented by real frame time in `ProcessHaptics` (`gl_openvr.cpp:530-560`). Sustained feel
  therefore means re-firing per tic — which is a feature, not a tax: it lets you **shape** the
  envelope, a single spike for a dash versus a swell over several tics for a landing.
- **The intensity is noise, not a clean pulse.** `QuakePower` returns a random value in
  `[-intensity, +intensity]` (`src/rendering/r_utility.cpp:911-921`). Good for a thud or a buzz;
  it will not give you a crisp click.
- **This is the OpenVR path** (`OpenVRHaptics::Vibrate`, `gl_openvr.cpp:517-528`), so it inherits
  the backend question in `00_CAPABILITIES.md` §0. OpenXR parity is unverified here.

**Still true, and still the clean fix:** neither `VRMode::Vibrate`
(`src/common/rendering/hwrenderer/data/hw_vrmodes.h:94`, real implementation at
`gl_openvr.cpp:517-528`) nor `VR_HapticEvent` is bound to ZScript, and `VR_HapticEvent` is in any
case an empty stub on this build (`src/QzDoom/qzdoom_common.cpp:65-67`), so the playsim's named
haptic calls (e.g. `src/playsim/p_user.cpp:1301-1304`) do nothing. **Exact missing piece:** a
ZScript-callable `PlayerInfo.Vibrate(hand, duration, intensity)` thunking to `VRMode::Vibrate` —
**a binding over an existing, working implementation, not a new feature.** One thunk, additive, no
behaviour change for existing callers. It would turn the HACK above into a BUILDABLE and remove
every caveat in this entry except the backend question.

**W3 — `A_ChangeVelocity` is dead on the player.** Trap 2 of §0. `actions.zs:63-79` short-circuits
on `!vr_recoil`, default false (`src/playsim/p_actionfunctions.cpp:90`). Any mod code (or borrowed
mod code) that moves the player with `A_ChangeVelocity` silently does nothing in a default install.
Not a blocker — write `Vel` directly — but it will bite anyone porting a flat-screen movement mod.

**W4 — `Thrust` has an unreadable direction bias.** `AActor::Thrust` adds `ThrustAngleOffset`
(`src/playsim/actor.h:1607-1613`), which the OpenVR path sets from the offhand controller when
`vr_move_use_offhand` (`src/rendering/gl/stereo3d/gl_openvr.cpp:2603-2606`), and the field has no
`DEFINE_FIELD`. So script can neither read nor cancel it. **Exact missing piece:** a readonly
ZScript field for `ThrustAngleOffset`, or a `ThrustAbsolute()` that skips it. Workaround is total
(write `Vel`), so this is a WALL only for code that insists on `Thrust`.

**W5 — `level.aircontrol` has no recompute hook.** The field is writable from ZScript
(`wadsrc/static/zscript/doombase.zs:455`) but `FLevelLocals::AirControlChanged()`
(`src/g_level.cpp:1927-1938`) is not exposed, so writing `aircontrol` alone leaves `airfriction`
stale. A mod must reimplement the formula. Minor, and documented in §3.3, but it is exactly the kind
of thing that produces a mechanic that works on one map and not another (map `aircontrol` is applied
at `src/g_level.cpp:1849-1858`).

**W6 — No per-player gravity *scaling* hook that survives other mods.** `Actor.Gravity` is writable
and per-player, which covers it — but two mods that both write `Gravity` on the same player will
fight, and the last writer wins with no arbitration. Not an engine defect; a coordination hazard
worth stating because this engine family is explicitly shared ground.

**W7 — Script cannot change view pitch, roll or yaw in VR.** `R_SetupFrame`
(`src/rendering/hwrenderer/hw_entrypoint.cpp:117`) computes `viewPoint.Angles` including any
`A_SetViewAngle` / `A_SetViewPitch` / `A_SetViewRoll` offsets (`actor.zs:1275-1277`, applied
`src/rendering/r_utility.cpp:676-690`) — and then `vrmode->SetUp()` at `hw_entrypoint.cpp:143`
calls `updateHmdPose`, which overwrites `vp.HWAngles.Pitch` and `vp.HWAngles.Roll` from the raw
HMD (`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:740-741`) and `vp.Angles.Yaw` from
the accumulated HMD yaw (`:750`); OpenVR does the same (`src/rendering/gl/stereo3d/gl_openvr.cpp:1467`,
`:1471`), and the view matrix is built from `HWAngles` downstream
(`src/rendering/hwrenderer/scene/hw_drawinfo.cpp:443`). This is correct behaviour — the headset
must own the head — but it means **no camera tilt on a wall run, no roll on a dash, no recoil
pitch kick.** Camera *position* is unaffected (§3.6), so viewheight and `SetViewPos` still work.
**What replaces it:** **per-hand haptics via `A_QuakeEx` (W2) — the primary substitute**, since it
delivers impact directly and, uniquely, can be *sided* to tell the player which wall they are on;
plus weapon-psprite motion (in the sprite layer and fully scriptable), sound, the `BlendA` dim,
and — for wall run specifically — the `HmdPosition` lean read, letting the player's own head do
the banking. Found by the flight survey (`04_FLIGHT_6DOF.md`) and confirmed here.

**W8 — Room-scale Z is discarded.** Positional tracking is applied in XY only and Z is explicitly
restored (`hw_openxrdevice.cpp:674-686`, `gl_openvr.cpp:2618-2632`). A mechanic cannot read
"player physically jumped" or "player physically ducked" as a change in the collision volume —
only as `HmdPosition.y` (detection still works) and, when `vr_crouch_use_button` is 0, as
`crouchfactor` driving `Height` (`player.zs:150-153`). See §3.5 for which mode gives what.
**Exact missing piece:** a vertical component in the positional-movement block, or a
ZScript-writable per-player height offset that the collision code honours.

**W9 — Do not build a gesture on the offhand grip or on an offhand-weapon gadget.** With
`vr_two_handed_weapons` on (default true, `hw_vrmodes.cpp:119`), pressing the **offhand grip**
with the hands closer than 0.50 m sets `weaponStabilised`
(`src/rendering/gl/stereo3d/gl_openvr.cpp:1716-1727`) — and this is **not** gated on the weapon's
`bTwoHanded` flag, so it fires for any weapon. The offhand weapon then goes completely dark:
it fires nothing (`player.zs:393` in `FireWeapon`, `:426` in `FireWeaponAlt`), checks no buttons
(the `hand == 1` early return in `P_CheckWeaponButtons`, `src/playsim/p_pspr.cpp:755-757`), and
renders nothing, with `WF_TWOHANDSTABILIZED` re-asserted on every state set
(`src/playsim/p_pspr.cpp:512-515`, the set itself at `:514`). Mostly peripheral to locomotion — none of the six designs above need it —
but it rules out two tempting input routes: **a two-handed brace pose as a dash trigger**, and
**an offhand-weapon "gadget" (grapple, booster) that fires a movement mechanic**. Either will
silently stop working whenever the player brings their hands together, which is exactly when a
brace gesture is being made. Use `BT_USER1..4` instead.

---

## 6. Smallest prototype

**Goal: prove the wall normal is real, in the engine, in one build.** Everything else in this report
rests on §3.4; if the normal is wrong, wall run is a fudge.

One `EventHandler` plus one `Inventory` controller item, no player class replacement, no map data:

1. `Inventory` subclass `WallProbe`, given on `WorldThingSpawned` for the player. In `DoEffect()`:
2. `BlockLinesIterator.Create(owner, owner.radius + 32)` (`doombase.zs:235`). For each `CurLine`,
   skip unless `ML_BLOCKING` or `!ML_TWOSIDED` (`mapdata.zs:175-177`). Compute
   `n = (CurLine.delta.y, -CurLine.delta.x).Unit()` and perpendicular distance.
3. Confirm the nearest candidate with `owner.LineTrace(atan2(-n.y,-n.x), 64, 0, TRF_NOSKY|TRF_THRUACTORS, offsetz: 20, data: d)`
   (`actor.zs:805`); require `d.HitType == FLineTraceData.TRACE_HitWall`.
4. Flip the normal by `d.LineSide` (`actor.zs:58`) per §3.4.
5. **Verification that is not circular:** spawn a short-lived marker actor at
   `d.HitLocation + n * 8` each tic, and `Console.Printf` the dot product
   `n dot (owner.pos.xy - d.HitLocation.xy).Unit()`. **If the derived normal is correct that dot
   product is ≈ +1 on every wall, at every approach angle, on both one-sided and two-sided lines, and
   the markers stand off the wall rather than inside it.** A value near -1 means the `LineSide` flip
   is inverted; a value wandering near 0 means the blockmap handed you a line you are not actually
   facing and step 3's trace confirmation is not filtering.
6. Only once that reads +1 everywhere, add: `Gravity = 0.12` while a wall is detected and
   `!player.onground`, `Vel.XY += -n * 0.6`, `player.keepmomentum = true`. That is a wall hang —
   the comfort-max variant of 4.5 — in about 60 lines, and it is also the safest thing to put in
   front of a headset first.

Deliberately excluded from the prototype: camera roll, viewheight changes, `SetViewPos`, blink
fades. Those are polish on a mechanic that is not yet proven, and each is an independent nausea risk.

---

## 7. Open questions

1. **Is `HmdPosition` recentred by the engine's own recentre, or only by the runtime's?**
   `VR_SetHMDPosition` (`src/QzDoom/qzdoom_common.cpp:93-101`) stores the raw pose and never applies
   an origin offset, and `vr_height_adjust` is added downstream at use sites
   (`hw_openxrdevice.cpp:154`) rather than at the source. So script reading `HmdPosition` sees raw
   tracking space *without* `vr_height_adjust`. I did not find where (or whether) a VDXR recentre
   resets that origin. Any lean or crouch design needs to know, and it should be measured in the
   headset, not reasoned about.
2. **How much does the renderer's per-frame head-translation `P_XYMovement`
   (`hw_openxrdevice.cpp:665-688`) actually displace a wall-running player?** It runs at frame rate,
   not tic rate, so its magnitude depends on framerate and on how much the player's head moves.
   It save/restores `Vel` so it cannot eat the run, but it can break contact. Needs a measured
   number before `wallGrabDist` and the re-grab window can be chosen.
3. **Does the `Speed = Default.Speed` / `Default.Speed * 8` split (`player.zs:1378` vs `:1382`)
   make held `keepmomentum` feel broken?** Holding `keepmomentum` through a long slide drops stick
   walking to 1/8 authority. I expect this reads as "heavy", which may be *right* for a slide and
   wrong for a wall run. Only the headset can answer.
4. **Which one-sided-wall edge cases does the `BlockLinesIterator` + confirm-trace pair miss?**
   Specifically: 3D floors (`Hit3DFloor`, `actor.zs:53`), line portals (`Line.isLinePortal()`,
   `mapdata.zs:229`), and polyobjects. None of these are map *edits*, but a wall run that silently
   grabs a portal line will teleport the player mid-run.
5. **Can a mod own the player class at all in the target deployment?** Every "cleanest" option in
   §3.1 (overriding `MovePlayer`, `ApplyAirControl`, `FallAndSink`) needs the player class. If this
   has to coexist with mods that replace the player, everything must go through `Inventory.DoEffect`,
   which is strictly weaker for `MovePlayer`-stage behaviour (you can counter-thrust after it, not
   prevent it). This is a scoping decision, not an engine fact.
6. **Netplay.** `level.gravity` / `aircontrol` / `airfriction` are level-global
   (`doombase.zs:454-456`) and shared by every player. Per-player `Actor.Gravity` is the net-safe
   lever. I did not audit prediction (`CF_PREDICTING`, e.g. `player.zs:1442`) against velocity
   written in `DoEffect` or `WorldTick`; a dash may mispredict. Worth a pass before any of this
   ships multiplayer.
7. **Will the owner run `vr_crouch_use_button 0`?** The slide design (4.3) and any
   duck-under-geometry mechanic need it, because that cvar decides whether the collision volume
   follows the head at all (§3.5, W8). It defaults to **true**, i.e. off for our purposes. This is
   a product decision — if the answer is no, 4.3 must be re-keyed onto a button and loses its best
   input.
8. **Does the `A_QuakeEx` haptic route work on the OpenXR backend?** W2's chain is verified on the
   OpenVR path (`OpenVRHaptics::Vibrate`, `src/rendering/gl/stereo3d/gl_openvr.cpp:517-528`). The
   call site in `src/rendering/r_utility.cpp:1085-1091` goes through `VRMode::Vibrate`, which is
   virtual (`src/common/rendering/hwrenderer/data/hw_vrmodes.h:94`), so whether OpenXR implements it
   decides whether the primary impact channel exists on the owner's VDXR setup at all. This
   inherits the backend question in `00_CAPABILITIES.md` §0 and should be settled before any
   mechanic's feel is designed around rumble.
9. **How much "feel" must the weapon psprite still carry?** W7 removes the camera-motion channel;
   W2's haptic route recovers most of the impact, but not all of it. I have not surveyed how far a
   psprite can be pushed in VR, where it is a tracked 3D model rather than a 2D overlay. If
   question 8 comes back negative, this becomes load-bearing rather than supplementary.
10. **Is `pr_torchflicker` consumption in `QuakePower` (`src/rendering/r_utility.cpp:911-921`) a
   determinism concern?** It is called from the render path, so it should be local-only, but a
   movement mod firing quakes every tic will consume RNG at frame rate. Worth confirming it does
   not touch the playsim stream before shipping this multiplayer, given the standing rule about
   playsim RNG on local paths.
