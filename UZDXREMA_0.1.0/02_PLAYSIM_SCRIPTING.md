# 02 — The playsim and scripting: cost per tic

Teardown area 02. Primary tree: **main** (`/home/claude/uzdx_main`, f182cbcc). Every `path:line` below is in main
unless it is prefixed `renderer:` (`/home/claude/uzdxrema`, 175063a), `netplay:` (`/home/claude/uzdx_netplay`, 1c1805f8)
or `selaco:` (`/home/claude/uzdx_selaco`, 7e111a3d).

Marks: **[V]** = VERIFIED, I read the code that does it. **[I]** = INFERRED, the reasoning is given. Numbers marked
"previously measured" are quoted from `renderer:docs/plans/SIM_AT_MOD_SCALE.md` with the build named there; none are
mine. Struct sizes were taken with a compile-time probe (`clang++ -fsyntax-only` on main's headers, Linux x86-64,
in `/tmp`); MSVC layout may differ by padding [I].

---

## 1. Summary

1. The sim is one thread, run inside the frame: `D_DoomLoop` → `TryRunTics` → N × (`G_Ticker` → `P_Ticker`) →
   `P_PredictClient` → N × `P_RunClientSideLogic` → `D_Display` (d_main.cpp:2006-2013, d_net.cpp:2475-2509) [V]. Nothing
   overlaps the renderer. At 35 tics/s and 72 Hz, roughly every second frame carries a whole tic [I, from TICRATE=35 doomdef.h:57 and the loop].
2. A world tic walks **every world actor at least three times** (interpolation reset p_tick.cpp:727-757, `RunThinkers`
   dthinker.cpp:149, light-recreation scan dthinker.cpp:318-333), plus a fourth (`WorldStateHash`, d_net.cpp:1235) in any
   netgame [V]. `AActor` is **3,104 bytes** (49 cache lines); the reset alone touches ~11 of them per actor [V offsets, I lines].
3. Every thinker tick goes through the VM call layer even when nothing is scripted: `DThinker::CallTick` →
   `VMCallVoid` (runtime signature validation every call) → `VMCall` → native thunk → C++ `Tick` (dthinker.cpp:1584-1590, vm.h:688-714) [V].
4. ZScript JIT (asmjit) is compiled in on x86-64 by default and `vm_jit` defaults to true (CMakeLists.txt:304-306,
   vmframe.cpp:43) [V]; it is forced off by `-debug` or the archived cvar `vm_debug` (d_main.cpp:4188-4195) [V]. Whether the
   owner's Windows build was configured with it cannot be read from the repo [open].
5. Ranked hot spots: memory-bound multi-walks of 3 KB actors; per-thinker VM entry and mod `Tick`/`DoEffect`/inventory
   ticks; fast small projectiles doing up to `1+speed/(radius-1)` full `P_TryMove`s per tic (p_mobj.cpp:2707-2734);
   O(things-in-block) `P_CheckPosition`; O(n²) `P_ChangeSector`; un-cached `P_CheckSight` (more of it in multiplayer);
   O(channels) sound queries (main only; fixed on renderer); a heap copy per direct state action call (info.cpp:179).
6. Main has **no tic catch-up cap**: a slow tic makes the next frame run up to a second of tics (d_net.cpp:2374-2426) [V];
   renderer has `sim_maxcatchup` (renderer:d_net.cpp:207,2415) [V].
7. Fork additions per tic: world clock (cheap at 1.0), beam/stamp/drawn-line snapshots (fixed ≤128/64 slots), Affect
   (one test when idle), chase brain / hurtbox / guard (inert until a mod opts in), and the VR player chain
   (`P_LatchVRHands`, `UpdateCanonicalMainHandPose`, torso, gear, grab, **two 8192-unit laser traces per player per tic**,
   p_user.cpp:98-123,1994-2011) [V].
8. Per-frame sim work outside tics: prediction rollback serialise/deserialise of the local pawn **also in single player**
   (p_user.cpp:2226-2257, 2345) [V]; GC step; `S_UpdateSounds` incl. evicted-channel restart walk (main) [V].
9. The sim and the renderer **write each other's state**: the renderer writes sector planes (interpolation), `validcount`
   fields, actor positions (portal sprites), and in single-player VR runs `P_XYMovement` on the pawn every frame
   (p_vrdemo.cpp:494-548 from vk_openxrdevice.cpp:5850) [V]. Even `TObjPtr` reads write (dobjgc.h:137-143) [V].
10. Instrumentation that exists: `stat think/VM/sight/ACS/gc`, `profilethinkers`, `perf_track` (think_ms, csthink_ms, gc),
    `r_perflog`; renderer adds `ticprofile`, `simprofile`, `sim_prefetch` (measured: no effect). Several stats are
    mis-scoped or corruptible (section 6).

---

## 2. Entry points and call graph

### 2.1 Frame loop and tic scheduling

```
D_DoomLoop                                   d_main.cpp:1966
 ├─ I_StartFrame (if gametic advanced)        d_main.cpp:1998-2002
 ├─ TryRunTics                                d_net.cpp:2339
 │   ├─ GC::CheckGC  (PerfTrack REASON_GC)    d_net.cpp:2343-2346 → dobjgc.cpp:155-163
 │   ├─ doWait=false in VR                    d_net.cpp:2354-2365
 │   ├─ totalTics = EnterTic-LastEnterTic     d_net.cpp:2374   (no cap on main; renderer caps, see §7)
 │   ├─ NetUpdate(totalTics)                  d_net.cpp:2382
 │   ├─ runTics=min(total,available)(+1)      d_net.cpp:2420-2426
 │   ├─ [runTics<=0] P_ClearLevelInterpolation (full actor walk) / P_PredictClient / P_RunClientSideLogic×total  d_net.cpp:2431-2463
 │   ├─ P_UnPredictClient                     d_net.cpp:2474 → p_user.cpp:2345 (deserialise pawn)
 │   ├─ while(runTics--)                      d_net.cpp:2475-2499
 │   │    ├─ G_Ticker                         g_game.cpp:1983
 │   │    ├─ MakeConsistencies                d_net.cpp:1286 (netgame only: WorldStateHash)
 │   │    └─ ++gametic; PerfTrack::NoteTic    d_net.cpp:2486-2489
 │   ├─ P_PredictClient                       d_net.cpp:2500 → p_user.cpp:2226 (serialise pawn; replay predicted tics)
 │   ├─ for totalTics: P_RunClientSideLogic   d_net.cpp:2504-2505 → p_tick.cpp:155
 │   ├─ S_UpdateSounds                        d_net.cpp:2509
 │   └─ NetworkEntityManager::VerifyPredictedEntities  d_net.cpp:2510
 ├─ I_StartTic; D_ProcessEvents               d_main.cpp:2010-2011
 └─ D_Display → D_Render(DoInterpolations…)   d_main.cpp:2013, 690-709
```

[V] all lines read. VR pacing: `doWait` is forced false when `VRMode::IsVR()` (d_net.cpp:2357-2359), so the loop never
sleeps for a tic; frame pacing is the OpenXR runtime's. With TICRATE 35 (doomdef.h:57) and a 72 Hz headset, a frame
gets 0 or 1 tic (sometimes 2 after a slow frame) [I]. The whole tic cost lands inside one 13.9 ms frame [I].

### 2.2 G_Ticker (g_game.cpp:1983-2161) [V]

`C_TickQueuedInputs` (1987) → VR demo frame (1995-1999) → reborns (2003-2012) → `gameaction` switch (2016-2099) →
`BotInfo.Main` (2105) → per client: `RunPlayerCommands` + copy `usercmd_t` into `players[].cmd` (2107-2128) →
`C_RunDelayedCommands` (2130) → `P_Ticker` (2137) in GS_LEVEL/GS_TITLELEVEL.

### 2.3 P_Ticker (p_tick.cpp:644-920), in order [V]

| # | step | line | per |
|---|---|---|---|
| 1 | `interpolator.UpdateInterpolations()` | 648-651 → r_interpolate.cpp:186-192 | per level, per interpolation object |
| 2 | paused: walk all actors for `MF8_RECREATELIGHTS` then return | 662-680 | per frame while paused |
| 3 | `DPSprite::NewTick` | 682 | once |
| 4 | freeze toggle every 4 tics | 686-697 | once |
| 5 | `S_ResumeSound` unless timefreezer | 702-709 | once |
| 6 | `P_ResetSightCounters`, `R_ClearInterpolationPath` | 711-712 | once |
| 7 | **world clock** `AdvanceWorldClock` | 724 → g_levellocals.h:1579-1585 | per level |
| 8 | **interpolation reset walk**: `ClearInterpolation`, `ClearFOVInterpolation`, `modelData->ShiftSurfacePositions` | 727-757 | **per actor** |
| 9 | `P_ThinkParticles` (world step only) | 760 → p_effect.cpp:278 | per particle |
| 10 | `TickBillboards` | 762 → 267-297 | per billboard |
| 11 | surface stamps age (64 slots), beams prev-snapshot (128 slots), `SnapshotDrawnLines` | 782-837 | fixed |
| 12 | `P_StepBeamOutlines` | 856 | per outline record |
| 13 | `BeginEffectTic` (effect queues new generation) | 863 → g_levellocals.h:2284 | once |
| 14 | `P_PlayerThink` per player in game | 865-867 → p_user.cpp:1878 | per player |
| 15 | `localEventManager->WorldTick()` (every real tic) | 872 → events.cpp:646-651 | per handler |
| 16 | world step: `Level->Tick()` (zero Scrolls) | 880 → g_level.cpp:2417-2424 | per sector w/ carry |
| 17 | world step: **`Thinkers.RunThinkers`** | 881 → dthinker.cpp:149 | **per thinker** |
| 18 | world step: `P_UpdateSpecials` (deathmatch timelimit only) | 886 → p_spec.cpp:688-699 | once |
| 19 | world step: `time/maptime/totaltime++` | 890-892 | once |
| 20 | world step: `localEventManager->WorldStep()` | 896 | per handler |
| 21 | not a world step: `P_TickRealTimeActors` | 902 → 79-120 | per actor (dormant at scale 1.0) |
| 22 | `realtime++`, log window | 908-915 | once |
| 23 | `VRSyncLog_OnTick` | 919 → p_vrsynclog.cpp:179 | one cvar read; full walk if on |

### 2.4 RunThinkers → AActor::Tick

```
FThinkerCollection::RunThinkers           dthinker.cpp:149
 ├─ ThinkCycles.Clock                      :159
 ├─ RunSleepCycle → CheckSleepingThinkers  :162, :123, :928  (STAT_SLEEP walk; ShouldWake/Wake via VM)
 ├─ for stat 32..127 (skip 60,61): Thinkers[i].TickThinkers(nullptr)   :167-171
 ├─ repeat until none: FreshThinkers[i].TickThinkers(&Thinkers[i])     :174-182
 │     TickThinkers                         :962-1000
 │      ├─ NextToThink = node->NextThinker (global)                     :975
 │      ├─ fresh: Remove/AddTail, CallPostBeginPlay (VM; actor → WorldThingSpawned) :976-985, p_mobj.cpp:6103-6107
 │      └─ CallTick                         :994 → :1584-1590
 │            IFVIRTUAL(DThinker,Tick) → VMCallVoid → VMValidateSignatureSingle → VMCallCheckResult → VMCall
 │            (vm.h:1066-1076, 688-714; vmframe.cpp:686-691, 705-752)
 │            ├─ native slot (Thinker.Tick, doombase.zs:277): NativeCall → NativeTick → virtual DThinker::Tick
 │            └─ script override: VMCycles[0].Clock → ScriptCall (JIT or VMExec) → … super.Tick → CALL_K native
 ├─ profile path when `profilethinkers` armed  :184-261
 └─ ThinkCycles.Unclock; PerfLog::AddThinkSample when perf_track/r_perflog on  :263-268
```

`AActor::Tick` (p_mobj.cpp:4741-5432) [V], in order:
`CheckUnmorph` VM call for morphed non-players (4781-4796) → `freezetics` (4798) → **`Affect_Tick`** (4807; returns on
`!Active()`, p_affect.cpp:821-824) → `CalcBones` if decoupled anims (4809-4812) → `TickBehaviors` (4826-4827; TMap walk,
VM per behavior, p_mobj.cpp:712-751) → NOINTERACTION fast path: unlink/move/relink (4829-4863) → else: per carried
inventory item `Inventory.DoEffect` VM call (4872-4882) → rocket/grenade trail `Spawn` (4906-4938) → VFRICTION, visibility
pulse, stealth (4945-5005) → `BotTick` if bots (5007-5011) → scroll carry over `touching_sectorlist` (5016-5150) → slope
push (5153-5189) → **`P_XYMovement`** (5215) → `Affect_SweepStep` (5220) → sector hit-floor/ceiling actions (5222-5225) →
Z: `P_CheckOnmobj` (blockmap) or **`P_ZMovement`** (5239-5319; `CallFallAndSink` VM at 3158) → `CheckPortalTransition`
(5321) → `UpdateWaterLevel` (5323) → poison (5336) → KILLMONSTERS (5348) → `P_ActorOnSpecial3DFloor` (5358) →
special sector/terrain (5360-5366) → `CheckNoDelay` (5374) → **`UpdateRenderSectorList`** (5377) → state countdown and
`SetState` → `CallAction` (5382-5392; skipped while `Affect.HoldsAI()`) → `CalcBones` for model actors (5394-5397) →
nightmare respawn (5399-5431).

`SetState` (p_mobj.cpp:1000-1105) loops 0-tic states (limit 300000), calls `FState::CallAction` (info.cpp:148) per state
entered, and sets `MF8_RECREATELIGHTS` whenever the class has light associations or the old/new state has a light
(1099-1102) [V].

Movement: `P_XYMovement` (p_mobj.cpp:2593) splits the move into `steps = 1 + speed/(radius-1)` when speed exceeds
`radius-1` (2707-2734) and calls `P_TryMove` per step (2769) [V]. `P_TryMove` (p_map.cpp:3717) → `P_CheckPosition`
(3170: `PointInSector` 3182; things via `FMultiBlockThingsIterator` + `PIT_CheckThing` 3267-3317; lines via
`FMultiBlockLinesIterator` + `PIT_CheckLine` 3341-3368) → on success `UnlinkFromWorld` + `LinkToWorld` (4079-4093) →
spechit line crossings → `CheckSectorTransition`. `LinkToWorld` (p_maputl.cpp:412-534) does `PointInSector` +
`PointInRenderSubsector` (428, 437), sector thing list, `P_CreateSecNodeList` (blockmap-lines walk, 468;
p_secnodes.cpp:216-283), render sector list (471), `CollectConnectedGroups` (485), blockmap node per touched block
(507-528), `UpdateRenderSectorList` (533) [V].

### 2.5 Monster AI path (state action functions)

`A_Chase` (p_enemy.cpp:3425-3450) → `A_DoChase` (2722-3214) [V]: target validation → friendly re-target →
`P_LookForPlayers` when no target (2849) → **chase brain** VM consult if `ChaseBrain != nullptr` (2891-2936,
`P_ConsultChaseBrain` 2609-2689) → melee: `P_CheckMeleeRange` (incl. sight) + `P_DeclareMonsterStrike` (chain walk) +
`SetState` (3070-3085) → missile: `P_CheckMissileRange` (sight + RNG) (3090-3120) → **multiplayer/TIDtoHate only:
`P_CheckSight(actor,target)` then `P_LookForPlayers`** (3125-3147) → walk: `P_SmartMove`/`P_Move` → `P_TryMove`,
`P_NewChaseDir` (3163-3201) → active sound (3208).

`A_Look` (p_enemy.cpp:2000) → sound-target check → `P_LookForPlayers` (1845-1989) → per player in game
`isTargetablePlayer` → `P_IsVisible` (1333-1379: distance, FOV, then `P_CheckSight`). `P_CheckSight` (p_sight.cpp:840):
REJECT (861) → visibility RNG (873-883) → water boundaries → blockmap path traverse with intercept sort
(`SightCheck::P_SightPathTraverse`, 619) and portal re-traces (931-945) [V].

### 2.6 Client-side logic per real tic (p_tick.cpp:155-218) [V]

`C_Ticker`, `M_Ticker`, `UiTick` (ZScript per handler), inventorytics, per level: `AdvanceClientWorldClock`, walk
client-side actors to clear interpolation (188-197), **`RunClientSideThinkers`** (199 → dthinker.cpp:294-477: sleep
cycle, tick client-side thinkers, then `recreateLights` walks **all world actors and all client-side actors**
(318-333, 384) doing `SetDynamicLights` for flagged ones and `P_RunEffect` for actors with `effects`, then **ticks every
`FDynamicLight`** (385-410)), `StatusBar->CallTick` (205), automap (209), `PostUiTick` (217). Runs `totalTics` times
per frame, independent of how many world tics ran (d_net.cpp:2504-2505).

### 2.7 Sim work that runs per frame, not per tic [V]

- `P_UnPredictClient`/`P_PredictClient` every `TryRunTics` that ran tics: serialise/deserialise the local pawn,
  `ViewPos`, `modelData`, `players[]` (incl. `VRTorso/VRGear/VRGrab_Serialize`, p_user.cpp:2513-2516), and RNG state
  (p_user.cpp:2238-2257, 2352-2378). No `netgame` test on this path; in single player no tic is re-simulated
  (`ClientTic == gametic`), but the write and read happen [V code; cost I].
- In a netgame, `P_PredictClient` re-runs `P_PlayerThink` + `player->mo->CallTick()` for every predicted tic
  (p_user.cpp:2270-2302) — including the VR chain and the two laser traces.
- `S_UpdateSounds` (s_sound.cpp:1366-1392): every channel, and `RestoreEvictedChannels` on every call because
  `RestartEvictionsAt` is reset to 0 after use and only `SetRestartTime` raises it (s_soundinternal.h:243,427;
  s_sound.cpp:1387-1390); the restore is **recursive over the whole channel list** (1316-1357).
- **Single-player VR:** `VKOpenXRDeviceMode::SetUp` → `UpdateControllerState` (vk_openxrdevice.cpp:3575,3628,3906)
  writes `HmdPos`, `Attack*`, `Offhand*` on the pawn (5603-5617, 5690-5696, 5774-5778) and calls
  `VR_ApplyRenderMove` (5831, 5850) → `ApplyRenderMoveBody` → **`P_XYMovement`** on the pawn (p_vrdemo.cpp:494-548)
  every frame; the teleport aim calls `P_LineTrace` from the frame (5806). Gated by `!VR_InputViaCommand()` (5842).

---

## 3. Per-function table

Cost classes: C = constant; P = per player; T = per thinker; A = per actor; M = per moving actor (× steps);
S = per state change / action call; L = per light; B = O(objects in blockmap cells touched); N² = quadratic.

| function | file:line | called from | per | cost class | notes |
|---|---|---|---|---|---|
| `TryRunTics` | d_net.cpp:2339 | `D_DoomLoop` d_main.cpp:2006 | frame | C + tics | no catch-up cap on main [V] |
| `GC::CheckGC` / `Step` | dobjgc.cpp:155, 482 | d_net.cpp:2345 | frame | ∝ allocation | incremental mark/sweep; `stat gc`, perf_track `gc` [V] |
| `NetUpdate` | d_net.cpp | :2382 | frame | C | builds/sends commands [V call] |
| `G_Ticker` | g_game.cpp:1983 | d_net.cpp:2484 | tic | C + P | [V] |
| `MakeConsistencies` → `WorldStateHash` | d_net.cpp:1286, 1235 | :2485 | tic (netgame) | **A**, 72 B byte-wise FNV per actor | serial multiply chain per byte [V code, I cost] |
| `P_Ticker` | p_tick.cpp:644 | g_game.cpp:2137 | tic | see 2.3 | [V] |
| `FInterpolator::UpdateInterpolations` | r_interpolate.cpp:186 | p_tick.cpp:650 | tic | per interpolation object | movers/scrollers/polys [V] |
| interpolation reset loop | p_tick.cpp:727-757 | P_Ticker | tic | **A** (all statnums, `IsKindOf` per thinker) | prev. measured 7.75 ms/tic @39,500 thinkers (renderer `doomxr_rend1112c.exe`) |
| `AActor::ClearInterpolation` | actorinlines.h:46-59 | above, prediction | A | ~11 cache lines (offsets §4) | reads `Sector->PortalGroup` (extra miss) [V/I] |
| `DActorModelData::ShiftSurfacePositions` | actor.h:1400-1408 | p_tick.cpp:756 | actor w/ modelData | 32 slots × 3 arrays (~1 KB copy) | RS_SURF_SLOTS=32 actor.h:1312 [V] |
| `P_ThinkParticles` | p_effect.cpp:278 | p_tick.cpp:760 | particle | `PointInRenderSubsector` per particle (330) | BSP walk each [V] |
| `FLevelLocals::TickBillboards` | p_tick.cpp:267 | :762 | billboard | C each | [V] |
| beam/stamp/drawn-line snapshot | p_tick.cpp:782-837 | P_Ticker | tic | 64 + 128 + DrawnLineHigh | fixed loops [V] |
| `P_StepBeamOutlines` | p_beamoutline.cpp | p_tick.cpp:856 | record | C when idle | [V call] |
| `P_PlayerThink` | p_user.cpp:1878 | p_tick.cpp:867; prediction 2300 | P (+ predicted tics) | VM `PlayerThink` + VR chain | [V] |
| `P_LatchVRHands` | p_vrhand.cpp:73 | p_user.cpp:1994 | P | C | reads usercmd only [V] |
| `UpdateCanonicalMainHandPose` | p_user.cpp:125-287 | :1995 | P | C | SP local VR reads live device pose via `GetWeaponTransform` (173,183 → hw_vrmodes.cpp:1644-1682) [V] |
| `VRTorso_Update` | p_vrtorso.cpp:222 | :1998 | P | C + 4 userinfo cvar lookups (165-190) | [V] |
| `VRGear_Tick` | p_vrgear.cpp:~417 | :2001 | P | per holster spot | [V partial] |
| `VRGrab_Tick` / `GrabConePick` | p_vrgrab.cpp:880 / 674-702 | :2008 | P × hand | B in reach + `P_CheckSight` per candidate | sight last (698) [V] |
| `P_VRLaserTraceTic` | p_user.cpp:98-123 | :2011 | P × 2 hands | two `Trace` up to 8192 units | runs whenever a hand pose is valid [V] |
| `VRTelemetry_Tic` | p_vrtelemetry.cpp:126 | :2037 | P | one cvar string test when off | [V] |
| PlayerPawn.`PlayerThink` (ZScript) | p_user.cpp:2046-2050 | P_PlayerThink | P | VM | weapon/psprite ticking inside [V call] |
| `EventManager::WorldTick` / `WorldStep` | events.cpp:646-651, 2096-2118 | p_tick.cpp:872, 896 | tic × handler | VM per overriding handler; `isEmpty` short-cut | prev. measured WorldTick 3.5 ms/tic (renderer, owner list) |
| `FThinkerCollection::RunThinkers` | dthinker.cpp:149 | p_tick.cpp:881 | tic | **T** | `ThinkCycles` = `stat think`, perf_track `think_ms` [V] |
| `RunSleepCycle` / `CheckSleepingThinkers` | dthinker.cpp:123, 928 | :162, :341 | sleeper | VM `ShouldWake` when due | [V] |
| `FThinkerList::TickThinkers` | dthinker.cpp:962 | :170, :180 | T | list walk + `CallTick` | global `NextToThink` [V] |
| `DThinker::CallTick` | dthinker.cpp:1584 | :994, p_tick.cpp:115 | T | VM call layer always (see §5.2) | [V] |
| `AActor::Tick` | p_mobj.cpp:4741 | CallTick | A | see 2.4 | [V] |
| `Affect_Tick` | p_affect.cpp:821 | p_mobj.cpp:4807 | A | one `Active()` test idle | [V] |
| `AActor::TickBehaviors` | p_mobj.cpp:712 | :4827 | A | TMap walk (+VM per behavior) | two local TArrays (no alloc when empty) [V] |
| `Inventory.DoEffect` dispatch | p_mobj.cpp:4872-4882 | AActor::Tick | A × items | VM per item (empty base returns early in `VMCall`, vmframe.cpp:720) | [V] |
| `P_XYMovement` | p_mobj.cpp:2593 | :5215; p_vrdemo.cpp:536,546 | M | **steps = 1+speed/(radius−1)** × `P_TryMove` | [V] |
| `P_TryMove` | p_map.cpp:3717 | P_XYMovement, P_Move, scripts | per step | `P_CheckPosition` + relink | [V] |
| `P_CheckPosition` | p_map.cpp:3170 | P_TryMove, many | call | 1 BSP + **B** things + B lines | corpses iterated and rejected at 2617 [V] |
| `PIT_CheckThing` | p_map.cpp:2600 | :3274 | thing in cells | C; VM `CanCollideWith` ×2 when solid contact (2713-2717 → 145-178) | hurtbox/guard tests for missiles (2887-2990) [V] |
| `PIT_CheckLine` | p_map.cpp:2062 | :3352 | line in cells | C | [V call] |
| `AActor::UnlinkFromWorld` | p_maputl.cpp:256 | relink sites | call | O(sector nodes + block nodes) | freelist ops [V] |
| `AActor::LinkToWorld` | p_maputl.cpp:412 | relink sites | call | 2 BSP walks + line walk + blocks | [V] |
| `P_CreateSecNodeList` | p_secnodes.cpp:216 | LinkToWorld 468, 471 | call | blockmap lines in radius | runs twice when `renderradius>=0` [V] |
| `AActor::UpdateRenderSectorList` | p_secnodes.cpp:375 | p_mobj.cpp:5377, p_maputl.cpp:533 | **A every tic** | gate never closes (§5.9); portal maps: all line portals | [V] |
| `P_ZMovement` | p_mobj.cpp:3134 | :5245, :5308 | A w/ Z motion | C + VM `FallAndSink` (3158 → 3631-3642) | [V] |
| `P_CheckOnmobj` / `P_TestMobjZ` | p_map.cpp:3430, 3457 | :5243 | PASSMOBJ/SPECIAL actors | B | [V call] |
| `CheckPortalTransition` | p_mobj.cpp:4467 | :5321 | A | C without portals | [V] |
| `UpdateWaterLevel` | p_mobj.cpp:5659 | :5323 | A | C + 3D floors | [V] |
| `AActor::SetState` | p_mobj.cpp:1000 | Tick 5389, actions | S | loop over 0-tic states | sets RECREATELIGHTS (1099-1102) [V] |
| `FState::CallAction` | info.cpp:148 | SetState, CheckNoDelay | S | **TArray copy (malloc/free) when function has defaults** (179) + 2 rdtsc | [V] |
| `A_DoChase` | p_enemy.cpp:2722 | A_Chase 3425-3450 | chasing monster × chase state | sight in MP, move | [V] |
| `P_LookForPlayers` | p_enemy.cpp:1845 | A_Look, A_DoChase | call | up to MAXPLAYERS `P_CheckSight` | [V] |
| `P_CheckSight` | p_sight.cpp:840 | AI, scripts, renderer (hw_bsp.cpp:1549) | call | REJECT, then blockmap traverse ∝ distance | static `intercepts`/`portals` (89-90) [V] |
| `P_NoiseAlert` | p_enemy.cpp:228 | weapon fire / scripts | shot | flood over connected sectors + their thing lists | [V] |
| `P_ChangeSector` | p_map.cpp:9345 | movers (DFloor/DCeiling etc.) | mover × tic | **N²** in things touching sector (9445-9464) + `P_Recalculate3DFloors` | [V] |
| `DScroller::Tick` (carry) | mapthinkers/a_scroll.cpp:187-206 | RunThinkers | scroller | O(things touching sector) flag set | [V] |
| `DACSThinker::Tick` | p_acs.cpp:3452 | RunThinkers / client-side | running script | interpreter | `ACSTime` reset per thinker (3454) [V] |
| `FDynamicLight::Tick` / `UpdateLocation` / `LinkLight` | a_dynlight.cpp:253, 377, 671 | dthinker.cpp:389-408 | **L per client tic** | relink = section flood (541) when moved/radius changed | `::validcount++` (687) [V] |
| `RecreateDynamicLights` scan | dthinker.cpp:279-292, 318-333 | RunClientSideThinkers | A (world + client) per client tic | C each | [V] |
| `EventManager::WorldThingSpawned` | events.cpp:690 | p_mobj.cpp:6106 | spawn × handler | VM | [V] |
| `EventManager::WorldThingDestroyed/Damaged/Died` | events.cpp:817, 805, 702 | p_mobj.cpp:6235, p_interaction.cpp:1649,1695,417 | event × handler | VM | [V] |
| `P_LineAttack` | p_map.cpp:6424 | hitscan actions | shot | Trace + 2 events + tracer queue + puff spawn | [V calls] |
| `P_QueueHitscanTracer` | p_hitscantracer.cpp:144 | events.cpp:759, 773 | shot | C | [V] |
| `VRSyncLog_OnTick` | p_vrsynclog.cpp:179 | p_tick.cpp:919 | tic | cvar test; when on A + fprintf/fflush | [V] |
| `P_PredictClient` / `P_UnPredictClient` | p_user.cpp:2226 / 2345 | d_net.cpp:2500 / 2474 | frame | serialise pawn + player + VR side tables | also in SP [V] |
| `SoundEngine::UpdateSounds` | s_sound.cpp:1366 | S_UpdateSounds d_net.cpp:2509 | frame | channels + recursive evicted restore | [V] |
| `SoundEngine::CheckSoundLimit` | s_sound.cpp:843 | every StartSound | sound | O(channels) | [V] |
| `SoundEngine::ChangeSoundPitch` etc. | s_sound.cpp:926-1100, 1259 | A_SoundPitch/StopSound/... | call | O(channels) on main | prev. measured Shifter 174 ms/tic before fix (renderer 1110a) |

---

## 4. Data structures that matter for speed

**Thinker lists** [V]. `FThinkerCollection` holds `Thinkers[MAX_STATNUM+2]` and `FreshThinkers[MAX_STATNUM+1]`,
`MAX_STATNUM = 127` (dthinker.h:42, 97-98). Each `FThinkerList` is a doubly linked ring through a heap-allocated
sentinel `DThinker` (dthinker.cpp:634-657). Links live inside each object (`NextThinker/PrevThinker`, dthinker.h:160),
so every walk is a pointer chase across scattered heap objects. Tick order: statnum ascending 32..127, 60/61 (sleep)
skipped, then fresh lists repeatedly until empty (dthinker.cpp:167-182). Statnums: `STAT_PLAYER` 33, `STAT_INVENTORY`
37, `STAT_LIGHT` 38, `STAT_DEFAULT` 100 (most actors), `STAT_SECTOREFFECT` 101, `STAT_SCRIPTS` 103,
`STAT_VISUALTHINKER` 105 (statnums.h:42-69). Link/unlink use GC write barriers (dthinker.cpp:653-656, 1129-1130).

**FThinkerIterator** (dthinker.cpp:1617-1729) [V]: with no statnum it visits 32..127 and both lists of each, and runs
`IsKindOf` per thinker — a parent-chain walk (`PClass::IsAncestorOf`, dobjtype.h:133-142) — so a walk over "all
actors" also touches every non-actor thinker and walks its class chain to the root. Sleep lists (60/61) are included.

**AActor** [V sizes via compile probe, main]: `sizeof(AActor) = 3104` (netplay 3096, selaco 2464; renderer's actor.h
equals main's). `sizeof(DThinker) = 80`, `player_t = 1240`, `sector_t = 912`, `line_t = 176`, `FBlockNode = 48`,
`msecnode_t = 56`, `FLevelLocals = 542,816`. Offsets used by the per-tic walks: `__Pos` 96, `Angles` 168, `Scale` 224,
`Alpha` 240, `flags` 308, `flags9` 340, `Vel` 408, `modelData` 488, `PrevFollowBodyYaw` 552, `BlockNode` 1104, `Sector`
1112, `tics` 1260, `state` 1264, `player` 1376, `alternative` 1432, `freezetics` 1744, `Affect` 1752,
`touching_sectorlist` 1896, `Inventory` 1936, `Prev` 2208, `PrevAngles` 2232, `PrevScale` 2256, `PrevAlpha` 2272,
`PrevFOV` 2280, `PrevPortalGroup` 2320, `Behaviors` 2360. `ClearInterpolation`+`ClearFOVInterpolation` therefore read or
write lines at ≈0,1,2,3,7,8,17,21,34,35,36 of the object plus the sector's line [I from offsets]. 40,000 actors ×
3,104 B ≈ 124 MB of actor memory, far above the i9-9900KF's 16 MB L3 [I]; each full walk streams the touched lines from
DRAM.

**Blockmap** [V]: 128-unit cells (`MAPBLOCKUNITS`), each with a singly linked `FBlockNode` chain; an actor is linked into
every cell its radius overlaps (p_maputl.cpp:507-528). Nodes come from a global free list `FBlockNode::FreeBlocks`
backed by `secnodearena` (p_secnodes.cpp:447-475). `FBlockThingsIterator::Next` de-duplicates multi-cell actors with a
32-bucket hash + 10 fixed entries + growable `DynHash` (p_maputl.cpp:949-1017; p_maputl.h). Dead monsters stay in the
blockmap (no `MF_NOBLOCKMAP`), so they are returned by every thing scan of their cells and rejected inside
`PIT_CheckThing` (p_map.cpp:2617) [V; that corpses keep blockmap links is I from the flag test].

**Sector links** [V]: `sector_t::thinglist` (by center), `touching_thinglist`/`touching_renderthings` (msecnode lists
built by `P_CreateSecNodeList`), `sectorportal_thinglist`; nodes from a global free list `headsecnode`
(p_secnodes.cpp:30-69). The renderer reads `touching_renderthings` and the portal lists.

**VM** [V]: `VMFunction` carries `DefaultArgs` (TArray), `VarFlags`, `ScriptCall` pointer (vm.h:431-482). Virtual
dispatch from C++ is `PClass::Virtuals[index]` with the index cached in a function-local static per call site
(vm.h:1066-1099). Script frames live on `thread_local GlobalVMStack` (vmexec.cpp:146). JIT functions use a stack frame
unless they have string registers, `SpecialInits` or `ExtraSpace` (then `AllocFrame` on the VM stack; jit.cpp:302-320,
409-458).

**TObjPtr** [V]: `target`, `tracer`, `master`, `goal`, `lastenemy`, `LastHeard`, `BlockingMobj`, `Inventory` are
`TObjPtr` (actor.h:2549-2644). Every read goes through `GC::ReadBarrier`, which loads the pointee's `ObjectFlags` and
**nulls the field** if the target is being destroyed (dobjgc.h:137-143): one extra cache line per pointer read, and a
store on a read path.

**Event handlers** [V]: a linked list per `EventManager` (`FirstEventHandler` … `next`); each world event loops the
static manager (only for the primary level) then the level's list (events.cpp:646-651, 685-700).

**Sound channels (main)** [V]: one global linked list `Channels`; every per-source query and `CheckSoundLimit` walk it
(s_sound.cpp:843-875, 926-1100). Evicted looping sounds stay in that list.

**usercmd_t** [V]: base fields + `vrhandpose_t` ×2 (12 B each), `vrhandinput_t` (10 B), `vrheadpose_t` (12 B),
`vrtrackers_t` (d_protocol.h:73-186). All controller data the netgame sim reads arrives here; the single-player sim
still reads the live device (§2.7, §5.13).

---

## 5. What slows it down (ranked)

Ranking is for a slaughtermap with heavy mods (tens of thousands of thinkers, ZScript-heavy actors). Confidence:
H = code read and previously measured; M = code read, cost inferred; L = plausible, needs measurement.

### 5.1 Memory-bound repeated walks over 3 KB actors — H
Evidence: the interpolation reset (p_tick.cpp:727-757), `RunThinkers` (dthinker.cpp:149-183), the light-recreation
scan over all world and client-side actors (dthinker.cpp:318-333, run `totalTics` times a frame), `WorldStateHash` in
netgames (d_net.cpp:1243-1253), `P_ClearLevelInterpolation` when waiting on the network (d_net.cpp:2440,
p_tick.cpp:606-639), and the paused branch (p_tick.cpp:665-678) [V]. Each uses `FThinkerIterator` with `IsKindOf`
over every thinker. Previously measured (renderer `SIM_AT_MOD_SCALE.md`, `doomxr_rend1112c.exe`, rr136, Abandon MAP13,
39,500 thinkers): reset 7.75 ms/tic, ~200 ns/actor; `sim_prefetch` 0/8/24/48 lines changed nothing
(`doomxr_rend1112d.exe`, rr139-142). My reading of why prefetch failed [I]: the reset's fields span lines 0-36 of a
3,104 B object (§4), and the sector dereference is a second dependent miss; the walk is bandwidth- and
dependent-load-bound, not first-touch-bound. Settle with: `ticprofile` (renderer) at 10k/20k/40k thinkers, and a
VTune/uProf memory-bound breakdown of the reset loop.

### 5.2 Per-thinker VM entry and ZScript ticks — H
Evidence [V]: `DThinker::CallTick` always finds a non-null virtual (Thinker.Tick is `virtual native`, doombase.zs:277),
so every thinker — sector movers, lights, scrollers, ACS thinker, plain actors — goes `VMCallVoid` →
`VMValidateSignatureSingle` (three out-of-line checks: `VMCheckParamCount`, `VMCheckReturn<void>`,
`VMCheckParam<DObject*>` with `isObjectPointer`, vmframe.cpp:578-632) → `VMCallCheckResult` → `VMCall` → native thunk
→ `PARAM_SELF_PROLOGUE` → C++ virtual `Tick`. Scripted overrides add `VMCycles[0]` rdtsc pair (vmframe.cpp:732-736),
JIT prologue with `VMCalls[0]++` (jit.cpp:460-469), and `super.Tick()` back to native. Every `Inventory` actor ticks
through the ZScript `Inventory.Tick` override (inventory.zs:115-150), owned or not. Each carried item gets a VM
`DoEffect` call per carrier tic (p_mobj.cpp:4872-4882). Previously measured (renderer, owner's list, rr102/103): sim
tic ~46 ms, 37,000 thinkers "at about a microsecond each", top classes `RSQTHP_Bar` ×1,888 6.5 ms,
`champion_CyanController` ×69 3.5 ms, `champion_Icon` ×906 2.9 ms. Settle with: `profilethinkers 8 50` on main, and a
build that bypasses `VMValidateSignatureSingle` in `CallTick` to measure the call-layer share.

### 5.3 Fast, small projectiles: many full moves per tic — M
Evidence [V]: `steps = int(1 + speed/(radius-1))` when speed > radius−1, radius ≤ 1 uses MAXMOVE
(p_mobj.cpp:2707-2734); each step is a `P_TryMove` (2769) with `P_CheckPosition` and, on success, a full
`UnlinkFromWorld`/`LinkToWorld` (p_map.cpp:4079-4093) including two BSP walks and `P_CreateSecNodeList`. A radius-2,
speed-100 projectile does 101 moves a tic [I arithmetic]. Settle with: `profilethinkers` sorted by average time while
firing projectile-bullet weapons; count `P_TryMove` calls per tic with a counter.

### 5.4 Collision scans: O(things in touched cells), including corpses — M
Evidence [V]: `P_CheckPosition` iterates every actor in every overlapped 128-unit cell (p_map.cpp:3269-3317);
`PIT_CheckThing` rejects corpses/non-solid only after loading their flags (2617) and does `blockdist` after loading
positions (2620-2622). Where solid contact happens, `P_CanCollideWith` makes up to two VM calls (145-178).
`P_CheckOnmobj`, `P_TestMobjZ`, `P_RadiusAttack`, `FMultiBlockThingsIterator` from script (vmiterators.cpp:165-222,
allocates a GC object per `Create`) are the same shape. In crowds this is ~O(k²) per cell [I]. Settle with: a per-tic
counter of things visited in `FBlockThingsIterator::Next`.

### 5.5 Movers in crowded sectors: `P_ChangeSector` restart loop — M
Evidence [V]: for each mover tic the loop restarts from the head after each processed thing (p_map.cpp:9445-9464; same
for 3D-floor attachments 9385-9401 and deep-water transfers 9479-9494), i.e. O(n²) node visits for n things touching the
sector; each `PIT_*` may run `P_CheckPosition`-class work. A lift or crusher under a pile of corpses scales
quadratically [I]. Settle with: node-visit counter in that loop on a slaughtermap crusher.

### 5.6 Sight checks without a cache, more of them in co-op — M
Evidence [V]: `P_LookForPlayers` tests every player in game (p_enemy.cpp:1932-1987) → `P_IsVisible` → `P_CheckSight`
(1378). In `A_DoChase`, `multiplayer || TIDtoHate` adds a `P_CheckSight(actor,target)` every chase call when
`threshold==0`, and on failure another `P_LookForPlayers` (3125-3147). Single player skips that block. So 4-player co-op
multiplies sight work by up to 4 in `A_Look` and adds a per-chase sight check [I from code]. `P_CheckSight` uses
REJECT first (p_sight.cpp:861); many PWADs ship an all-pass reject [I]. No result is cached across monsters or tics.
Counters: `stat sight` (SightCycles, sightcounts). Settle with: `stat sight` in SP vs a 4-player netgame on the same
demo spot.

### 5.7 Sound engine O(channels) (main only) — H
Evidence [V]: `CheckSoundLimit` on every `StartSound` and every per-source query walk the global channel list
(s_sound.cpp:843-875, 1073-1085 …); `UpdateSounds` restarts evicted channels on every call through a recursion as deep as
the channel list (1316-1357, 1387-1390; `RestartEvictionsAt` reset at 1389, raised only by `SetRestartTime`
s_soundinternal.h:427). Previously measured on renderer before its fix: Universal Entropy `Shifter` ×4,752 = 174 ms/tic
(1110a), restore walk ~35 ms/frame (1110f). Main has none of the renderer fixes (`SourceChannels`, `snd_evictretry`,
`snd_evictbudget`: 0 hits in main, renderer:s_sound.cpp:74,1341,1345) [V].

### 5.8 Tic catch-up spiral (main) — H
Evidence [V]: on main `totalTics` is the clock difference with no cap outside `singletics` (d_net.cpp:2374-2376), and
`runTics` may exceed it by one to catch up (2421-2426). A tic slower than 28.6 ms makes the next frame run more tics.
Previously measured (renderer stage 0, Abandon MAP13): 175 ms tic → 35 tics → 6.5 s frame before `sim_maxcatchup`.

### 5.9 `UpdateRenderSectorList` never short-circuits — M
Evidence [V]: the gate is `Pos() != OldRenderPos` (p_secnodes.cpp:378), and `OldRenderPos` is written only at spawn to
`FLT_MAX` (p_mobj.cpp:5716; grep: no other writer in any branch). So every non-NOSECTOR actor clears and rebuilds its
render line/sector-portal lists every tic (AActor::Tick 5377). Without portals this is two empty deletes and two
`PortalBlocksMovement` tests; on maps with line portals, every actor in a cell near one loops over **all**
`linePortals` and allocates nodes every tic (391-398). Upstream behaviour [I: not fork-marked].

### 5.10 Per-action heap copy and always-on clocks — M
Evidence [V]: `FState::CallAction` copies `ActionFunc->DefaultArgs` into a local `TArray` (malloc + free) whenever the
action function has default arguments (info.cpp:177-197; TArray copy allocates, tarray.h:247-250, 848-858). Direct
state actions bind the function itself when called without arguments (codegen.cpp:9976-9991), and `A_Chase`,
`A_FaceTarget`, `A_Wander` declare defaults (actor.zs:2384, 2407, 2456), so `POSS A 4 A_Chase` allocates on every entry
[I: depends on how each state was written]. Each `CallAction` also does 2 `rdtsc` (`ActionCycles`, info.cpp:152, 227),
each script `VMCall` 2 more (`VMCycles[0]`), each `NativeScriptCall` 2 more (vmframe.cpp:351-353), each `P_CheckSight`
2 more; `cycle_t` is unconditional rdtsc on Windows (stats.h:214-237). Same pattern in
p_actionfunctions.cpp:175-194.

### 5.11 Event handlers per world event — M (H for WorldTick)
Evidence [V]: `WorldThingSpawned` per spawn (incl. client-side effect actors, via `CallPostBeginPlay`, p_mobj.cpp:6103-
6107), `WorldThingDamaged` per damage (p_interaction.cpp:1649, 1695), `WorldThingDestroyed`, `WorldHitscanPreFired/Fired`
per hitscan (p_map.cpp:6428, 7010), each a VM call per handler that overrides it; a world event struct is built per
handler (events.cpp:1811-1822). Previously measured WorldTick handlers 3.5 ms/tic (renderer, owner list).

### 5.12 Dynamic lights on the client tic — M
Evidence [V]: every `FDynamicLight` ticks every client tic (dthinker.cpp:389-395); a moved light or changed radius relinks
with a section flood (a_dynlight.cpp:434-439, 541-689); `SetState` raises `MF8_RECREATELIGHTS` on any state change of a
class with light associations (p_mobj.cpp:1099-1102), and the recreate pass then rebuilds that actor's lights.
Measured `csthink_ms` covers this (perf_track). Previously measured client-side think 5.4-5.5 ms (renderer, owner list
and full list).

### 5.13 VR player chain — L (bounded by player count)
Evidence [V]: per player per tic two 8192-unit traces (`P_VRLaserTraceTic`, p_user.cpp:98-123), grab cone search with
sight checks while aiming (p_vrgrab.cpp:674-702), torso/gear math, userinfo cvar lookups, haptic events; in a netgame all
of it re-runs for each predicted tic each frame (p_user.cpp:2270-2302). Constant per player, so it does not scale with
the map, but it lands on the prediction path too.

### 5.14 Netgame-only per-tic hashing — M
Evidence [V]: `WorldStateHash` byte-wise FNV-1a over 72 bytes per actor (d_net.cpp:1238-1253) every tic in any netgame
(1288). One dependent 64-bit multiply per byte, ~72 per actor [I]. Settle with: `simprofile`-style clock around
`MakeConsistencies` in a 4-player session.

### 5.15 Prediction serialisation every frame, including single player — L
Evidence [V]: `P_PredictClient`'s writer and `P_UnPredictClient`'s reader run whenever tics ran in the frame, with no
`netgame` test (p_user.cpp:2226-2257, 2345-2381; d_net.cpp:2474, 2500). It serialises the pawn, its `ViewPos` and
`modelData`, `players[]` with VR side tables. Cost per frame, not per map size. Settle with: renderer's `simprofile`
"predict" column on main.

### 5.16 VM interpreter fallback and the DAP hook — L
Evidence [V]: functions with ≥200 registers or containing a function-pointer call (`blockJit`, codegen.cpp:10010) never
JIT (vmframe.cpp:300-311); the interpreter calls `DebugServer::RuntimeEvents::EmitInstructionExecutionEvent` on every
instruction (vmexec.cpp:57; RuntimeEvents.cpp:74-80, a global-flag test in another TU). On MSVC `COMPGOTO` is not
defined, so the interpreter is a `switch` (vmexec.cpp:52-61). If the build lacks JIT or `vm_debug` is set, every
script function takes this path.

### 5.B Shared mutable state that blocks a multi-threaded sim or sim ∥ render

Global sim state (any parallel thinker execution must privatise these) [V]:
- `validcount` (r_utility.cpp:253) and per-object `validcount` fields of `line_t`, `sector_t`, `AActor`, polyobjects —
  bumped by blockmap iterators (p_maputl.cpp:565, 576), `P_CheckPosition` (p_map.cpp:3245, 3330), sight (p_sight.cpp:917),
  noise (p_enemy.cpp:236), light linking (a_dynlight.cpp:687).
- `spechit`, `portalhit` (p_map.cpp:109-110); `NextToThink` (dthinker.cpp:65); `intercepts`, `portals`, `SightCycles`
  (p_sight.cpp:53-55, 89-90); free lists `headsecnode`, `FBlockNode::FreeBlocks` (p_secnodes.cpp:30, 447); `actionParams`
  (info.cpp:146); `NoiseList` (p_enemy.cpp:118); `currentVMLevel` (used vmiterators.cpp:47, 108, 181); RNG streams `pr_*`
  (e.g. p_mobj.cpp:133-136, p_sight.cpp:39-40) whose draw order is the netplay contract; `static previous_health` in
  `P_PlayerThink` (p_user.cpp:1933) shared across all players.
- `TObjPtr` reads write (`GC::ReadBarrier`, dobjgc.h:137-143); GC write barriers on list ops (dthinker.cpp:653-656).
- Thinker order is semantics: statnum order, then fresh-list re-ticks (dthinker.cpp:167-182); any actor's Tick may
  spawn, destroy, move or relink any other.

Renderer reads/writes of sim state (block running the sim while drawing) [V]:
- `D_Render` → `FInterpolator::DoInterpolations` **writes sector planes, texture Z, 3D floors and portal planes**, then
  restores them (d_main.cpp:700, 708; r_interpolate.cpp:236-271, 448-462); `P_FindParticleSubsectors`,
  `PO_LinkToSubsectors` (d_main.cpp:701-702).
- hw_bsp writes `line->validcount`, `sector->validcount`, `thing->validcount` and bumps `validcount`
  (hw_bsp.cpp:681, 720, 924, 1069, 1436); hw_sprites temporarily moves actors with `SetXYZ` and edits `Prev` and
  `Angles.Yaw` for portal copies (hw_sprites.cpp:3285-3300).
- Renderer calls `P_CheckSight` (hw_bsp.cpp:1546-1552), which can draw `pr_checksight` from the frame when the target is
  invisible (p_sight.cpp:873-883) — a frame-rate RNG draw from a playsim stream. Only with `r_dithertransparency` and an
  out-of-body view (hw_bsp.cpp:1085) [V]; DORMANT in VR.
- Renderer reads directly: `AActor` (Pos/Prev/Angles/state/sprite/frame/renderflags/Alpha/Scale/modelData),
  `sector_t::touching_renderthings`/`sectorportal_thinglist`, `Level->Particles`, `Level->lights` + light lists,
  `Billboards`, `Beam*`/`PrevBeam*`, stamps, drawn lines, effect queues — all mutated during the tic [V for the writers in
  §2.3; reader sites are area 03/04].
- Single-player VR frame code writes the pawn and **runs `P_XYMovement`** on it (§2.7), and `UpdateCanonicalMainHandPose`
  reads live device poses inside the tic (p_user.cpp:170-191 → hw_vrmodes.cpp:1644-1682).
- `r_NoInterpolate` and `R_ClearInterpolationPath` are written by `P_Ticker` (p_tick.cpp:652, 712, 715).
- Lights are relinked on the client tic into `Level->lightlists`, which the renderer reads (a_dynlight.cpp:455-465, 671).

Renderer branch has started the separation: `R_SnapshotWrite` once per `TryRunTics` after `P_PredictClient`, double
buffer, off by default (`r_snapshot 0`) (renderer:d_net.cpp:2557, renderer:src/rendering/r_snapshot.h:1-110) [V].

---

## 6. Hooks, cvars and dead code in this area

| item | where | status | evidence |
|---|---|---|---|
| `stat think` | dthinker.cpp:1737-1743 | ACTIVE | `ThinkCycles` covers `RunThinkers` incl. sleep cycle only — not the reset walk, player think, WorldTick, particles, client side. `ActionCycles` double-counts nested actions (Clock/Unclock nest) and is not unclocked on the exception path (info.cpp:205-227) [V] |
| `stat VM` | vmframe.cpp:896-912 | ACTIVE, approximate | `VMCycles[0]` clocked only at top-level `VMCall` of script functions and suspended in `NativeScriptCall`; rotated per stat draw (per frame), not per tic as its label says; nested `VMCall`s from natives double-count [V] |
| `stat sight` | p_sight.cpp:953-960 | ACTIVE, **corruptible** | early returns at 848 and 853 skip `SightCycles.Unclock()` after `Clock()` at 842, leaving a −rdtsc term; render-path calls (hw_bsp.cpp:1549) are counted into the next tic [V] |
| `stat ACS` | p_acs.cpp:11010-11013 | ACTIVE, mis-scoped | `ACSTime.Reset()` in every `DACSThinker::Tick` (3454); the world and client-side ACS thinkers both exist (10473-10481), so the stat shows whichever ticked last [V] |
| `stat gc`, `gc` CCMD | dobjgc.cpp:737, 844-855 | ACTIVE | [V] |
| `profilethinkers`, `profilecsthinkers` | dthinker.cpp:1462-1560 | ACTIVE (one tic) | per-class totals [V] |
| `perf_track` columns `tics, think_ms, csthink_ms, thinkers, csthinkers`; `REASON_GC`, `REASON_SCRIPT` | hw_perftrack.cpp:280, 305, 537-540, 627, 752-753; d_net.cpp:2344 | DORMANT (off by default) | `think_ms` = `RunThinkers` only (dthinker.cpp:267-268) [V] |
| `r_perflog` think samples | hw_perflog.cpp:704-717; hw_perflog.h:83 | DORMANT | [V] |
| `ticprofile` | renderer:p_tick.cpp:646-660 | renderer only | stage clocks around P_Ticker [V] |
| `simprofile` | renderer:d_net.cpp:198 | renderer only | gc/netupdate/ticker/predict/cslogic/sounds/verify [V] |
| `sim_prefetch` | renderer:dthinker.cpp:73-88, 998, 1044, 1723 | renderer only, DORMANT (default 0) | measured no effect (rr139-142) |
| `sim_maxcatchup` (default 2) | renderer:d_net.cpp:207, 2415 | renderer only, ACTIVE there | SP/demo only [V] |
| `PrefetchL3` in blockmap things iterator | p_maputl.cpp:955 | ACTIVE all branches | upstream [V] |
| `vm_jit` (default true), `vm_jit_aot` (true) | vmframe.cpp:39-59 | ACTIVE when built with `HAVE_VM_JIT` | not saved; restart required [V] |
| `vm_debug` (archived) / `-debug` | d_main.cpp:3662, 4188-4195 | DORMANT | sets `vm_jit=false` for the session [V] |
| `vmengine` CCMD | vmframe.cpp:919-939 | ACTIVE | checked/unchecked interpreter [V] |
| DAP per-instruction hook | vmexec.cpp:57; RuntimeEvents.cpp:74-80 | interpreter only; no-op unless debugger subscribed | not emitted by JIT (no reference in jit/*.cpp) [V] |
| `vr_synclog`, `vr_synclog_actors` | p_vrsynclog.cpp:38-39 | DORMANT | per-tic hash+fflush when set [V] |
| `vr_telemetry` | p_vrtelemetry.cpp:32 | DORMANT | [V] |
| world clock: `P_TickRealTimeActors`, `worldclock` CCMD, `worldclock_log` | p_tick.cpp:79-143; read at vmthunks.cpp:2822 | DORMANT at scale 1.0 | `AdvanceWorldClock` returns true every tic at full speed (g_levellocals.h:1581) [V] |
| `WorldStep` event | events.cpp:2109-2118; p_tick.cpp:896 | ACTIVE every tic, empty unless overridden | [V] |
| `RenderFrame` event | events.cpp:2132-2144; d_main.cpp:1690 | **DEAD** | body commented out; caller commented out [V] |
| `MBF_FRIENDS` block | p_enemy.cpp:1867-1898 | **DEAD** | `MBF_FRIENDS` defined nowhere (grep src, CMake); block would not compile (`i` undefined at 1877) [V] |
| `cl_noprediction` | p_user.cpp:297 | **DEAD** | no reader in src or wadsrc; only `EXTERN_CVAR` in src/gl/stereo3d/gl_openxrdevice.cpp:65, a file not in src/CMakeLists.txt [V] |
| `src/gl/stereo3d/gl_openxrdevice.cpp` | — | **DEAD** (not compiled) | CMake lists only rendering/gl/stereo3d/gl_openvr.cpp (src/CMakeLists.txt:1069) [V] |
| `VMException` code | vmframe.cpp:64-66, 707-751, 880-885 | **DEAD** (`#if 0`) | [V] |
| `cl_predict_specials` | p_user.cpp:295; p_spec.cpp:375 | ACTIVE | [V] |
| `nomonsterinterpolation` | p_enemy.cpp:74, 3191 | DORMANT | [V] |
| `netplay_desynctest` | p_user.cpp:291, 2014-2019 | DORMANT (debug) | [V] |
| `hurtvolume_debug`, `drawnshape_debug`, `vr_slotload_debug`, `vr_shotclamp_debug`, `aimdebug` | p_map.cpp:264, 665, 309, 5245; p_user.cpp:2137 | DORMANT (debug prints) | [V] |
| Thinker sleep API (`Sleep`, `SleepIndefinite`, `STAT_SLEEP*`) | dthinker.cpp:97-141, 1209-1335 | ACTIVE, opt-in by script | ported from GZSelaco [V] |
| Chase brain (`ChaseBrain`, `RS_ChaseBrain.Decide`) | p_enemy.cpp:2609-2689, 2891 | DORMANT per actor (null check) | [V] |
| Strike intent (`P_DeclareMonsterStrike`) | p_enemy.cpp:2517-2546, 3079, 3112 | ACTIVE on every monster attack commit | chain walk `P_StrikeChainTics` [V] |
| Affect (`Affect_Tick`, `Affect_SweepStep`) | p_mobj.cpp:4807, 5212-5220 | ACTIVE, one test when idle | [V] |
| Body hurtbox / guard intercept | p_map.cpp:2887-2990; 449 | ACTIVE code, inert until a volume is published | [V] |
| `Behaviors` | p_mobj.cpp:712-751 | ACTIVE, empty map typical | [V] |
| `P_VRLaserTraceTic` | p_user.cpp:98-123 | ACTIVE when a hand pose is valid | [V] |
| `VRGear_PlayTick/PlayAfter` (`vr_gear_play`) | p_user.cpp:1891, 2004 | DORMANT (desk scenario) | [V] |
| `P_QueueHitscanTracer` (`vr_hitscan_tracer`) | p_hitscantracer.cpp:144-174 | ACTIVE per hitscan, early-outs | [V] |
| haptics in `P_PlayerThink` | p_user.cpp:1933-1946 | ACTIVE, **wrong in MP** | `static previous_health` shared by all players; `VR_HapticEvent` fires for every player's health, and again inside prediction [V] |
| `UpdateRenderSectorList` gate | p_secnodes.cpp:378 | gate DEAD (never false) | `OldRenderPos` written only at p_mobj.cpp:5716 [V] |

---

## 7. Branch differences

| topic | main | renderer | netplay | selaco |
|---|---|---|---|---|
| `p_tick.cpp` | world clock, beams, outlines, VRSyncLog | = main + `ticprofile` clocks (renderer:p_tick.cpp:644-660, 743-930) | = main | no world clock, no outlines, no VRSyncLog; adds `P_ThinkDefinedParticles` after RunThinkers (selaco:p_tick.cpp:576); interp loop same incl. ShiftSurfacePositions |
| `dthinker.cpp/.h` | sleep, perflog, worldStep | = main + `sim_prefetch` | = main | sleep; no perflog; no worldStep param |
| `TryRunTics` | no catch-up cap | `sim_maxcatchup`=2 SP, `simprofile`, `R_SnapshotWrite` (renderer:d_net.cpp:184-214, 2413-2416, 2493, 2557) | no cap | no cap |
| `WorldStateHash` in consistency | yes (d_net.cpp:1235) | yes (renderer:d_net.cpp:1267) | yes (netplay:d_net.cpp:1218) | no |
| sound per-source lists / evict budget | no | yes (renderer:s_sound.cpp:74, 1341-1345) | no | no |
| `P_PlayerThink` VR chain | latch, canonical pose, torso, gear, gear-play, grab, laser trace, sim roomscale, telemetry | latch, canonical, torso, gear (older), laser trace, sim roomscale; **no VRGrab**, no gear-play (renderer:p_user.cpp:1985-2015) | latch + canonical pose only; no torso/gear/grab/laser trace/sim roomscale | none |
| `sizeof(AActor)` | 3104 | 3104 (actor.h identical) | 3096 | 2464 |
| `Affect_Tick` in AActor::Tick | yes | yes | yes | no |
| p_map.cpp | — | `shotclamp` print differs only | missile origin uses `AttackPos/OffhandPos` directly instead of `P_ShotOrigin` (netplay:p_mobj.cpp:8513-8520) | separate fork |
| VM / JIT sources | — | identical | identical | frontend + vmnatives differ (zcc_parser, vmnatives.cpp) |
| `UpdateRenderSectorList` gate bug, `CallAction` copy, `P_ChangeSector` N², sight stat bug | present | present | present | present (upstream code) [V grep OldRenderPos 4 hits each] |

---

## 8. Open questions and the measurements that would settle them

1. **Is JIT on in the owner's Windows build?** Read `HAVE_VM_JIT` from `E:\DOOMWork\UZDXREMA\build-dxr\CMakeCache.txt`
   and `vm_debug` from the owner's ini; at runtime `vm_jit` in the console. A JIT-off build would make §5.2 and §5.16
   dominant.
2. **Split of a world tic on main** (renderer has the tool): port `ticprofile` + `simprofile` clocks to a test build of
   main, run the owner's list on Abandon MAP13 and Sunder MAP20 standing and fighting, single player and 4-player.
3. **VM call-layer share**: build with `DThinker::CallTick` calling `Tick()` directly when the virtual slot is the
   native `Thinker.Tick` (compare pointer), and with `VMValidateSignatureSingle` skipped; compare `stat think`.
4. **Projectile substeps**: per-tic counters of `P_TryMove`, `LinkToWorld`, and `FBlockThingsIterator::Next` visits,
   bucketed by class, with BD22 weapons firing.
5. **`P_ChangeSector`**: node-visit counter on a lift/crusher under 200+ corpses.
6. **Sight**: fix the `SightCycles` early returns first (§6), then `stat sight` SP vs 4-player netgame on one demo;
   count `P_CheckSight` calls per tic by caller.
7. **`CallAction` allocations**: count calls taking the `DefaultArgs` branch per tic (allocation counter), or an
   allocator profile on a 5,000-monster fight.
8. **`WorldStateHash` and prediction cost**: clock `MakeConsistencies` and `P_PredictClient`/`P_UnPredictClient` per
   frame in SP and in a 4-player session.
9. **Memory-bound walks**: hardware counters (L3/DRAM misses) over the interpolation reset and the recreate-lights scan;
   then a test with a dense array of actor pointers (renderer doc's proposal) to see whether the walk becomes
   bandwidth- or latency-bound.
10. **Frame placement in VR**: per-frame log of `tics run` vs frame time on the headset (perf_track `tics` column) to
    confirm whether tic frames are the ones that miss 13.9 ms.
11. **Does anything still read `OldRenderPos` intentionally?** If not, writing it after the rebuild in
    `UpdateRenderSectorList` restores the intended skip; measure on a line-portal map.
12. **Multiplayer haptics**: confirm on the rig that `VR_HapticEvent` fires for remote players' health (p_user.cpp:1933-1946).
