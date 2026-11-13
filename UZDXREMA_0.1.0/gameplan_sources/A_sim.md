# A_sim — The fork's playsim and scripting cost per tic, against playbook Phase 0 (sim), 0.5 and 9

Area: playsim + ZScript cost per tic on UZDXREMA `main` @ e84623f, the engine of record. Sources: [F] = `/home/claude/uzdxrema`
(main), [FR] = `/home/claude/uzdxrema_renderer` (renderer @ abb19af), [U] = stock UZDoom 5.0.0, [M] = owner's measurements
(`10_MEASUREMENTS.md`). The owner's own mod sources were also read where they decide a recommendation:
`/home/claude/presidentkoopa/rsq_ubqseries/...` (cited as [MOD]). All paths are repo-relative. Every saving is **Inference**,
with its derivation. Struct offsets were probed on [F] e84623f with `clang++ -fsyntax-only -DNDEBUG` (Linux x86-64; MSVC may pad
differently).

**Key reading before the table.** `main` is not the build the owner measured. [M] used `doomxr_rend1113a.exe` (renderer
branch). The renderer branch carries four sim-side fixes that `main` lacks: the catch-up cap, the per-source sound index, the
eviction budget, and the `ticprofile`/`simprofile` clocks (§B.1). On `main` the measured 130.8 ms/tic would be the *best* case.
The `Shifter` class alone cost 174 ms/tic on a renderer build from before the sound fix.

The playbook's targets are stock-5.0 files. Most of those files are **byte-identical in the fork**:
- `info.cpp`, `p_secnodes.cpp`, `p_sight.cpp`, `p_maputl.cpp`, `portal.cpp`, `dobjgc.cpp`, `p_blockmap.h` and `vm.h` diff to
  0 lines against [U];
- `vmframe.cpp` differs by 3 lines (log text only);
- `dobjtype.cpp` differs only by class aliases (`diff` run on each).

So every [U] line the playbook gives for those files is also the [F] line. The fork-modified hot files are `dthinker.cpp`
(+331 diff lines), `p_tick.cpp` (+668), `p_mobj.cpp`, `d_net.cpp` and `actorinlines.h`.

**Owner direction (2026-11-13): build the permanent engine fix.** Where a cheap workaround and a proper engine-side fix both exist, the engine fix is the plan of record. A workaround is kept only as a labelled **stopgap**, behind a cvar, and is removed once the permanent fix passes the netplay suite and `-benchdemo` GREEN. The netplay-determinism and lane rules still apply to both.

| Topic | Plan of record (permanent) | Stopgap (labelled; remove when the permanent fix lands) |
|---|---|---|
| Per-actor side walks (reset, light scan, netgame hash, slow-mo walk) | One **dense actor array** maintained by the thinker lists. All four walks run over it, with hot fields co-located in `AActor`. | none |
| Lazy interpolation reset with mods that write `Actor.Prev` | **Codegen store hook**: a store to `Actor.Prev` calls `NotePrevWrite`, so Step B stays exact on this modlist | Step A only (full exact pass) |
| `r_ticstability` | Delete the spin. Under OpenXR, take `TicFrac` from `predictedDisplayTime`; on desktop, from a smoothed frame-start timestamp (playbook Phase 3 D) | `r_ticstability 0` forced in VR |
| Netgame consistency hash | A word-wise 64-bit hash over the dense array, every actor, every tic | Striping (¼ of actors per tic) |
| Catch-up spiral | The sim thread (Phase 8). `sim_maxcatchup` stays as a single-player safety net, not as the fix | `sim_maxcatchup` |
| Sound O(channels) | Per-source index (renderer branch), plus evicted loops kept in their own list, plus a spatial bucket for `CheckSoundLimit` (renderer lane's §6 item 1 and catalogue) | `snd_evictbudget` restart throttling |
| Per-monster helper thinkers (HP bars, icons, tokens) | Engine **overlays, attachments and tickless/event mechanisms** (playbook Phase 10, another agent's area) | Parking plus one batch updater in script; Tick LOD |
| Portal render lists (F2) | The full `RenderListKey` with `portalEpoch` | The conservative "skip only when the sector has no portal planes" key |
| Sim profiling | Tracy zones plus the continuous `sim_profile` / `vm_profile` (0.1 + 0.5) | `ticprofile` / `simprofile` port |

Tick LOD (P9.3) is a separate case. It is approximate by design, so it is not a "permanent fix" for anything. It is a deliberate, opt-in, deterministic game-rule switch for classes the owner chooses, used where Phase 10 cannot remove the work.

---

## Delta table

| ID | Item | Fork status | Where in fork | What is left | Done when (measurable) | Don't touch | Netplay/determinism impact | Est. saving on modlist (Inference) |
|---|---|---|---|---|---|---|---|---|
| P0-F1 | `CallAction`/`CheckCallerType` TArray copies | Not started | [F] src/gamedata/info.cpp:135 (`auto argtypes`), :179 (`auto defs`); second copy at [F] src/playsim/p_actionfunctions.cpp:177 | Make all three `const auto&`. 135 runs on **every** action call with `ImplicitArgs>=1`, so every actor action pays at least one malloc/free, not only those with defaults. | Allocation counter (0.5-3) shows 0 allocs from `CallAction`. `-benchdemo` GREEN. | `actionParams` re-entrancy (info.cpp:180-197): leave the Reserve/Clamp pattern alone. | None. Pure C++ copy removal; identical VM args; bit-exact. | 0.2-1.5 ms/tic. Calls per tic are not measured. Bound: 53,116 thinkers × ≤1 state entry/tic × 40-100 ns per malloc/free pair ≤ 2-5 ms. Realistic 10-30% of thinkers change state in a tic. |
| P0-F2 | `UpdateRenderSectorList` never early-outs | Not started (code = stock) | [F] src/playsim/p_secnodes.cpp:375-427; `OldRenderPos` written only at [F] src/playsim/p_mobj.cpp:5716; field [F] src/playsim/actor.h:1905; callers p_mobj.cpp:5377, p_maputl.cpp:533, p_map.cpp:9046-9282 | **Plan of record:** the full `RenderListKey`, including `Level->portalEpoch` bumped at every portal-state change, with all five traps handled. Do not ship the conservative "skip only without portal planes" key except as a labelled stopgap. Add the `r_verifyrenderlists` shadow check. Put a line-portal map into the benchdemo set. | `r_verifyrenderlists 1` silent over a portal-map demo. `Sim.RunThinkers` A/B on a portal map. | Portal render-list order is read by ZScript (`sectorportal_thinglist`) — do not change membership. | Cross-peer: deterministic (same inputs give same lists on every peer). Cross-build: order of `sectorportal_thinglist` changes, so golden benchdemos that iterate it may report DESYNC. Re-record those. | Portal-free map: ≤0.5-1 ms (≈10-20 ns × actors that run the `AActor::Tick` tail). Line-portal maps: actors near a portal scan **all** `linePortals` per tic (p_secnodes.cpp:390-398), which is large. Sunder MAP15's portal content is unknown (Q4). |
| P0-F4 / P9.1a | `CallTick` VM detour + `VMValidateSignatureSingle` per call | Not started | [F] src/playsim/dthinker.cpp:1584-1590; [F] src/common/scripting/vm/vm.h:690 (validate), checks [F] vmframe.cpp:578-632 | Validate once per `VMFunction*` (or under `#ifndef NDEBUG`). In `CallTick`, call `Tick()` directly when `func` is the stock native. | `stat think` A/B at equal thinker count. Shortcut hit-count printed by `sim_profile`. | `IFVIRTUAL` index caching (vm.h:1066-1076). | None. The stock native ran the same code. | 0.5-1.6 ms. Derivation: 53,116 thinkers × 10-30 ns. The shortcut helps only stock-Tick classes; validation removal helps all. |
| P0-F5 | O(n²) sight intercept selection | Not started (stock) | [F] src/playsim/p_sight.cpp:534-583 | Use a stable sort by `frac`, then walk in order. | `stat sight` (after fixing its unclock bug, 0.5-T) A/B on Sunder MAP15. | — | None: the stable sort reproduces first-minimum order exactly. | Unknown. Idle monsters spend sight time in `A_Look`, ×players in co-op. Sunder's long open sightlines favour the fix. Needs Q6. |
| P0-F9 | Particle bucketing per frame | Not started | `P_FindParticleSubsectors` from `D_Render` (render side) | Render agent's item. Sim-relevant because Phase 8 needs it moved to tic end. | — | — | Presentation only. | Per frame, not per tic (not counted here). |
| P0-F12 | `r_ticstability` busy-wait | Not started; **not VR-gated** | [F] src/d_net.cpp:2251 (default true), 2256-2270 spin, 2279-2284 (duration = last tic, capped at **1 s**), call 2436 | **Plan of record:** remove the spin. Derive `TicFrac` from OpenXR's `predictedDisplayTime` in VR, and from a smoothed frame timestamp on desktop. **Stopgap:** force it off in VR and with any sim thread. | VR: `perf_track` shows no spin time. With a 20 ms tic, tic-less frames ≤ render time. | Netplay lane owns `TryRunTics` ([F] RENDER_LANE_HANDOFF_2026-11-09.md:197-199): co-sign. | None. Pacing only; no sim state. | Per frame. The spin equals the *last tic's* duration on every tic-less frame. At a 20 ms tic and 90 Hz, 55 of 90 frames/s would spin 20 ms (all miss). Off: −T ms on those frames. [M] §4: −2.3 ms/frame with no mods. |
| P0-F13 | JIT verification | Not started (no startup line) | Build: [F] CMakeLists.txt:304-306 (`HAVE_VM_JIT` ON on x86_64). Cvars [F] src/common/scripting/vm/vmframe.cpp:39-59. Forced off by `-debug` or archived `vm_debug`: [F] src/d_main.cpp:3662, 4188-4195. Per-function fallback [F] vmframe.cpp:297-311 | Add the one-line `VM: JIT built=… compiled=… interpreted=…` report and a `vmstats` ccmd. Check the owner's ini for `vm_debug`. | Startup log line on the owner's build; `interpreted` count by name. | — | All peers must run the same JIT mode (same build, `vm_debug 0`). Mixed JIT/interpreter peers risk FP or edge-case divergence (Inference). | 0 if JIT is on. If it is off, ZScript runs interpreted, and most of the 130.8 ms is script (Q1). |
| P0.5-1 | `sim_profile` continuous per-class | Partial: one-tic `profilethinkers` only | [F] src/playsim/dthinker.cpp:164-262, 1007-1048; `csprofilethinkers` 362-380 | Keep the TMap across tics. Dump top-20 every 35 tics. Accumulate closed intervals in u64. Self-test: per-class sum ≈ `RunThinkers`. | Per-class totals over 300 tics, sum within 5% of the thinker clock. | — | Read-only. | Enabler (finds the ≈78 ms tail, §B.3). |
| P0.5-2 | Per-virtual zones (`vm_profile`), per-`WorldTick`-handler time | Not started | Dispatch [F] src/events.cpp:646-651, 2096-2105; `VMCall` vmframe.cpp:705-737 | Clock per `VMFunction` at `VMCall` and in the JIT prologue. Attribute `WorldTick` by handler class. | 4.07 ms of WorldTick split by handler. | — | Read-only. | Enabler (WorldTick 4.07 ms is dozens of handlers; the owner's RSQ set alone has ≥27 overrides, §B.6). |
| P0.5-3 | Per-class allocation counter | Not started | `PClass::CreateNew` (stock), TArray growth | As in the playbook. Also count F1's copies. | allocs/tic by class. | — | Read-only. | Enabler. |
| P0.5-4 | Spike trigger | Not started | — | Log the per-class top 10 when a tic exceeds `sim_spike_ms`. | — | — | Read-only. | Enabler. |
| P0.5-5 | Action/sight/trymove counters | Not started | `CallAction` info.cpp:148-234; `P_CheckSight` p_sight.cpp:840; `P_TryMove` | Per-tic counters by caller class. | Counters in `sim_profile` output. | — | Read-only. | Enabler (sizes F1, F5, P9.4-9.6). |
| P0.5-T | Port `ticprofile`/`simprofile` to main; fix `simprofile` | **On renderer only** | [FR] src/p_tick.cpp:646-660, 743-930; [FR] src/d_net.cpp:194-205, 2375-2376, 2530-2533 | Port both. **Fix the simprofile bug:** the CCMD runs from `C_RunDelayedCommands` inside `G_Ticker` ([F] src/g_game.cpp:2130), i.e. inside the open `SimWholeCycles`/`SimTickerCycles` intervals. It prints `accum − rdtsc_start` (negative) and then `Reset()`s mid-interval. Fix: defer printing to the end of `TryRunTics`, or accumulate closed intervals only. Also fix the `stat sight` early-return unclock leak ([F] p_sight.cpp:842, 848, 853) and the `ActionCycles` exception path (info.cpp:152, 227). | `simprofile` totals ≥ 0 and whole ≥ sum of parts. | — | Read-only. | Enabler. |
| P9.1b | `CallAction` direct-native call | Not started | [F] src/gamedata/info.cpp:148-234 | Per-signature thunks; leave the rest on the VM path. | Action time −30% at equal call count. | `try`/`CVMAbortException` diagnostics. | None (same native, same args). | 0.3-1.5 ms (N action calls × 50-100 ns; N from 0.5-5). |
| P9.1c | Native owned-inventory tick | Not started | ZScript `Inventory.Tick` [F] wadsrc/static/zscript/actors/inventory/inventory.zs:115-149; carrier `DoEffect` per item [F] src/playsim/p_mobj.cpp:4872-4882 | Native body for the stock `Inventory.Tick`. For an owned item with `tics==-1 && DropTime==0`, the stock body does nothing: skip it natively. | `profilethinkers` shows the stock-Tick inventory classes at ≈0. | Classes that override `Tick` keep VM dispatch. | Exact (same field updates). | 0.5-2 ms. If ~10k owned helper items (crit and champion tokens are plausible, not verified) × 50-150 ns. Shifter (7,686 × 0.29 µs = 2.23 ms) only if it does not override `Tick`. |
| P9.2 | Dormancy (exact) + closet look | Not started (engine). **Script opt-in sleep exists** | [F] src/playsim/dthinker.cpp:93-141 (`[SLEEP]`, GZSelaco port), statnums 60/61 [F] src/playsim/statnums.h:58-59 | Engine predicate per the playbook, plus fork terms: `Affect.Active()` (p_mobj.cpp:4807), `MF9_DECOUPLEDANIMATIONS`/bone info (4809-4812, 5394-5397), `IsRealTimeActor` under slow-mo, `modelData` (ShiftSurfacePositions). `sv_dormancy_verify` shadow-tick. | `sv_dormancy_verify 1` silent over the benchdemo set. `-benchdemo` GREEN. | Thinker order: dormant actors stay in their lists. | Cross-peer: deterministic even if the predicate is incomplete (same state gives the same skip on every peer). An incomplete predicate shows up as a *stock-fidelity* bug, not a desync. Gate with verify + benchdemo vs stock. | 2-4 ms. Only stock-Tick, idle actors qualify. If 10-15k qualify × ~0.25 µs saved each = 2.5-3.8 ms. Script-heavy helpers are excluded. |
| P9.3 | Tick LOD (`sv_ticklod`, class `TickLOD`) | Not started | — | An opt-in game-rule switch, not a fix. For helper actors, the permanent route is Phase 10 (overlays, attachments, tickless): use LOD only where Phase 10 cannot remove the work. Build (a)-(d) as in the playbook. Fork add: a **lump-side opt-in list** (e.g. `TICKLOD` lump in the owner's own pk3), so third-party classes (mk-champions, crits) can be opted in without editing their pk3s. | Benchdemo with `sv_ticklod 1` documented as non-stock. Netplay suite IN SYNC with it on. | (e) wall-time watchdog: **never** in netgames or demos. | Deterministic if the criterion is sim state (distance to nearest player body, maptime stagger) and the cvar is `CVAR_SERVERINFO`. Approximate by design (behaviour changes). The watchdog is non-deterministic. | The largest engine lever after the sim thread: −10 to −40 ms, depending on the classes opted in. Example: cosmetic helpers champion_Icon 4.79 + champion_Leaves 2.18 + IA_Sound 0.89 + aggro ≈4.3 = 12.2 ms; at k=4 for 75% of them: 12.2 × 0.75 × 0.75 ≈ −6.9 ms. Adding controllers (7.16 + 2.11 + 1.53) and LoB monsters (≈11) at 50% far: another −8 to −15 ms. |
| P9.4 | BSP point grid (game + render nodes) | Not started (stock walk) | [F] src/playsim/p_maputl.cpp:1946-1962 (= [U]) | As in the playbook, with fixed-point corner tests and separate x/y bounds. | `PointInSector` calls/tic × ns before and after; benchdemo GREEN. | — | Exact by construction (same `R_PointOnSide`). | 0.5-3 ms. Every relink does 2 walks (p_maputl.cpp:428, 437). RSQ bars `SetOrigin` most tics ([MOD] bar.zs:221, 498); 3,622 bars + parts + moving monsters ≈ 5-10k relinks × 2 × ~150 ns saved. Particles add `PointInRenderSubsector` per particle (0.72 ms bucket). |
| P9.5 | Inline blockmap line data | Not started | `FBlockLinesIterator` order (stock) | As in the playbook (lump order preserved). | `PIT_CheckLine` time −40% in a fight demo. | Iteration order. | Exact. | ~0 standing at the map start ([M] runs); 0.5-2 ms in fights (unmeasured). |
| P9.6 | Within-tic sight cache | Not started | `P_CheckSight` [F] p_sight.cpp:840-950 | Only if the 0.5-5 counter shows repeated pairs. Keep the RNG reject outside the cache. | Repeated-pair rate; sight time −50% with F5. | `pr_checksight` draw order. | Exact barring ZScript `Line.flags` writes. Cross-peer deterministic regardless. | 0-3 ms SP; up to ×4 in 4-player co-op (one check per player in `P_LookForPlayers`). |
| P9.7 | Portal link acceleration | Not started | `CollectConnectedGroups` (stock) | As in the playbook. | — | — | Exact. | 0 on portal-free maps. |
| P9.8 | Arenas + `AActor` hot-field layout | Not started. **The fork's actor is 3,104 B** | Offsets (probe): `__Pos` 96, `Angles` 168, `Scale` 224, `Alpha` 240, `flags8` 336, `Vel` 408, `modelData` 488, `FollowBodyYaw` 512, `PrevFollowBodyYaw` 552, `FollowHandRot` 600, `PrevFollowHandRot` 888, `Sector` 1112, `CameraFOV` 1224, `tics` 1260, `player` 1376, `Inventory` 1936, `Prev` 2208, `PrevAngles` 2232, `PrevScale` 2256, `PrevAlpha` 2272, `PrevFOV` 2280, `PrevPortalGroup` 2320, `Behaviors` 2360 | Co-locate the reset block (including the fork's FollowBody/FollowHand fields) and the light-scan fields. Slab allocation for `CreateNew`. | The reset touches ≤4 lines per actor; savegame round trip passes. | `DEFINE_FIELD` offsets (the JIT uses them: safe, but test). | Addresses do not feed the sim. | Folded into P9.9 (≈−1 to −2 ms of it); slabs 0-1 ms. |
| P9.9 | Interpolation-reset pass (+ the recreate-lights walk) | Not started; **the fork widened it** | Reset loop [F] src/p_tick.cpp:719-757 (iterator 727, clear 738-739, `ShiftSurfacePositions` 756); `ClearInterpolation` [F] src/playsim/actorinlines.h:46-59 (fork lines 52-56 add FollowBodyYaw/HandRot); light walk [F] src/playsim/dthinker.cpp:318-331, called 384 | **Plan of record:** Step A (dense array, co-located fields, cached `PortalGroup`), then Step B made exact with the codegen `Prev`-store hook (§B.4). **Step B as the playbook writes it is blocked on this modlist**: the owner's mods store to `Actor.Prev` ([MOD] bar.zs:222, 499; words.zs:33; numbers.zs:250; glyphs.zs:30; QuestSaber zscript.txt:3928, 3978, 4113). The playbook's compile-time detection would force Step A for every session. Alternative in §B.4. Move the light walk onto the same array. | Reset ≤2.5 ms/tic on Sunder MAP15 + owner set; `csthink` −50%. | The world-clock skip (`!worldStep && !IsRealTimeActor`, p_tick.cpp:735) and the model-surface snapshot point (comment p_tick.cpp:741-755) must stay at this instant. | Order-free per actor, so exact. The dense array must be maintained deterministically (spawn order, swap-remove on destroy). | −6.0 to −6.7 ms: 9.02 → 2.3-3.0 (3-4×). Light walk: −2 to −3 ms of the ≈5.5 ms client tic (FR doc measurement, §B.5). |
| P9.10 | GC pacing | Not started | `GC::CheckGC` per frame [F] src/d_net.cpp:2343-2346 (stock GC) | Deadline-bounded `Step` on the sim thread. Tune stepmul/pause on Sunder. | `perf_track` gc column p99 < 1 ms. | — | GC is not sim-observable except script `OnDestroy` of unreachable non-thinkers, whose timing is already frame-rate dependent today (CheckGC per frame). | Unknown (Q8). |
| P9-WT | `WorldTick` handlers 4.07 ms | No engine work possible without attribution | [F] src/p_tick.cpp:872; [F] src/events.cpp:2096-2105 (empty overrides already skipped) | P0.5-2 attribution. Per-class actor lists (P10.5, another agent) remove `ThinkerIterator` scans. | Per-handler ms. | — | — | 0 from Phase 9. Up to −3 ms via mod fixes. |
| F-SIM-1 | Catch-up cap (`sim_maxcatchup`) missing on main | **Renderer only** | [FR] src/d_net.cpp:207-213, 2413-2416; [F] d_net.cpp:2374 (no cap) | **Stopgap:** merge [FR] cfaf6ac to main (netplay handoff item 1: [F] HANDOFF_NETPLAY_2026-11-11.md:35). **Permanent fix:** the sim thread (Phase 8); the cap then stays as a single-player safety net. | Sunder MAP15 + owner set does not spiral (tics run per frame ≤2). | Netgames keep stock catch-up. | `!netgame` only, so netplay is untouched. SP/demo: changes *when* tics run, not their results. | Stability: on main a 145 ms tic makes the next frame run 5+ tics and spirals (FR measured 6.5 s frames before the cap). |
| F-SIM-2 | Per-source sound index + eviction budget missing on main | **Renderer only** | [F] src/common/audio/sound/s_sound.cpp:1073-1085 (full channel walk) vs [FR] s_sound.cpp:1089-1102, 1343-1345, 1403-1404 | Merge the renderer per-source index after the netplay suite (owed, handoff item 1). **Plan of record** beyond it: keep evicted loops in their own list and add a spatial bucket for `CheckSoundLimit`, so no walk scales with the channel count. **Stopgap:** the `snd_evictbudget` throttle. | `Shifter` ≤ 2.3 ms/tic on main with the owner set. | — | Per-source queries return the same channel set; if any query returns "first match", order may differ (check). Sound state is already machine-local (eviction depends on listener), a pre-existing netplay hazard for play-scope `IsActorPlayingSound`. | Main vs renderer: avoids a regression of tens to >150 ms/tic. FR doc measured `Shifter` 174 → 1.6 ms (4,752 Shifters × 2,914 channels). Sunder has 7,686 Shifters. |
| F-SIM-3 | `r_ticstability` scales with the tic | See P0-F12 | [F] src/d_net.cpp:2279-2284 | Same as P0-F12. | — | — | — | With a slow tic, it pads tic-less frames by up to 1 s. Mandatory off for VR and for any sim thread. |
| F-SIM-4 | Netgame `WorldStateHash` | Fork-added full walk | [F] src/d_net.cpp:1235-1256 (byte-wise FNV over 72 B/actor), called 1291 every tic in any netgame | **Plan of record:** hash 9 words (64-bit mix) per actor over the dense array, every actor, every tic. **Stopgap only:** stripe 1/4 of actors per tic (`index % 4 == gametic % 4`). | `MakeConsistencies` ≤ 0.5 ms at 53k actors. Suite IN SYNC; `netplay_desynctest` still detected. | Netplay lane owns `MakeConsistencies` ([F] RENDER_LANE_HANDOFF_2026-11-09.md:197-200). | Changes consistency values; all peers run the same build, so it is compatible. Striping delays detection by ≤3 tics. | 4-player only: −6 to −10 ms/tic with the word-wise hash (8-12 → ≈1.5-2.5), −7 to −11 with the striping stopgap. Today ≈150-230 ns/actor (a reset-like walk + 72 dependent FNV steps ≈ 60 ns) × 53,116 = 8-12 ms. |
| F-SIM-5 | Fork widening of the reset | Present | actorinlines.h:52-56 (+3 cache lines: 512-552, 600, 888); `ShiftSurfacePositions` 3 × 32 slots ≈ 1 KB copy per `modelData` actor ([F] actor.h:1312, 1400-1408) | Fold into P9.9 co-location. Copy only slots < high-water mark. | — | The snapshot instant (p_tick.cpp:741-755). | Exact. | Part of the 9.02 ms. |
| F-SIM-6 | `P_TickRealTimeActors` | Dormant at scale 1.0 | [F] src/p_tick.cpp:79-120, 899-903 | Use the dense array when slow-mo is used. | — | — | Deterministic (iterator order). | 0 at full speed. |
| F-SIM-7 | VR player chain per player per tic | Present | [F] src/playsim/p_user.cpp:1994-2037; laser 98-123 (2 × 8192-unit `Trace`) | Measure in the headset. Cap or LOD the laser trace distance by blockmap occupancy if large. | Player think ≤0.15 ms/player in VR. | `usercmd_t` latch order (p_user.cpp:1988-1995). | Deterministic (latched usercmd). Re-runs per predicted tic in netgames ([F] p_user.cpp:2270-2302). | Flat: 0.07 ms total [M] §2 (torso invalid flat). VR, 4 players: ≤0.5 ms (unmeasured). |
| F-SIM-8 | Engine-registered `WorldTick` handlers | Present | [F] wadsrc/static/mapinfo/common.txt:8 | None for perf. | — | — | `OtherPlayerTagHandler` reads `consoleplayer` in play scope ([F] zscript/otherplayertags.zs:169-184): check that it only touches client-side objects. | µs. |
| F-SIM-9 | Inert fork hooks per actor | Present | `Affect_Tick` [F] p_affect.cpp:821-824; hurtbox/guard in `PIT_CheckThing` [F] p_map.cpp:2887-2990; chase brain p_enemy.cpp:2891 | None. | — | — | — | ≈1 load/actor: <0.3 ms. |
| F-SIM-10 | Measurement hygiene | Present | `perf_bench_hash` default **on** walks every actor per tic while recording/playing hashed demos ([F] src/g_perfbench.cpp:55, 140-180, 230, 319) | Subtract it, or turn it off for timing runs; keep it on for GREEN runs. | — | — | It is the SP determinism gate. | Inflates benchdemo sim time by ≈ one reset walk (~5-9 ms at 53k). |
| F-SIM-11 | Snapshot write (render thread prerequisite) | Renderer only, off | [FR] src/rendering/r_snapshot.h:11-21; [FR] d_net.cpp:2493, 2557 | Budget it on the sim thread. Write it from the P9.9 dense array. | `snapshotstats` ≤1 ms at 53k. | Netplay rule: read-only, never blocks ([F] RENDER_LANE_HANDOFF:204-212). | Read-only. | Cost to add: unknown for mode 1 (mode 2 = 24 ms on Abandon MAP13). |

---

## Findings

### B.1 Main is not the measured build; four sim fixes live only on `renderer`

- **No catch-up cap on main.**
  - [F] src/d_net.cpp:2374 `int totalTics = EnterTic - LastEnterTic;`. The only cap is `singletics`, and `runTics` may add one more (2420-2426).
  - [FR] src/d_net.cpp:2415-2416 adds `if (!netgame && sim_maxcatchup > 0 && totalTics > sim_maxcatchup) totalTics = sim_maxcatchup;`.
  - [M]'s frame of 315 ms with 250 ms of think is about 2 tics a frame: the cap at work.
  - On main the same map runs every owed tic. That is the spiral the renderer lane's doc measured at 6.5 s a frame ([FR] docs/plans/SIM_AT_MOD_SCALE.md §1, a lead, not code).
- **O(channels) sound queries on main.**
  - [F] src/common/audio/sound/s_sound.cpp:1075 walks `Channels` for every `ChangeSoundPitch`.
  - [FR] s_sound.cpp:1091-1092 walks only `FirstSourceChannel(source)`.
  - Main also restarts evicted channels with no budget ([F] s_sound.cpp:1387-1390, against [FR] :1343-1345, 1403-1404).
- **No tic clocks on main.** `grep ticprofile|simprofile|sim_maxcatchup|sim_prefetch` over [F] src finds nothing. [FR] has all four.
- The netplay lane already lists the merge as its item 1 ([F] HANDOFF_NETPLAY_2026-11-11.md:35). The sound commits have never run through the suite.

**Inference:** until these land on main, any headset build of `main` on the owner's set is slower than [M], possibly by an order of magnitude. Port them first. This is a merge, not new work.

### B.2 The tic as the fork runs it (P_Ticker on main)

`P_Ticker` [F] src/p_tick.cpp:644-920, per level:
- the world-clock decision (724);
- the **reset walk** (727-757): `GetThinkerIterator<AActor>`, then `ClearInterpolation` + `ClearFOVInterpolation`, plus `modelData->ShiftSurfacePositions()` per model actor;
- particles (760), billboards (762);
- the beam/stamp/drawn-line snapshots (782-837), beam outlines (856), effect queues (863);
- `P_PlayerThink` per player (865-867);
- `WorldTick` (872);
- the world step (878-897): `Level->Tick`, **`RunThinkers`** (881), `WorldStep` handlers (896);
- the real-time actors under slow-mo (899-903);
- `VRSyncLog_OnTick` (919).

`RunThinkers` [F] src/playsim/dthinker.cpp:149-271:
1. `RunSleepCycle` first (162).
2. Statnums 32..127 ticked in order, skipping the sleep lists 60/61 (167-171).
3. The fresh lists re-ticked until none are left (174-182).
4. `TickThinkers` (962-1000) → `CallTick` (1584-1590):
   ```cpp
   IFVIRTUAL(DThinker, Tick)
       VMCallVoid<DThinker*>(func, this);
   ```
   Every thinker goes through `VMCallVoid`, which runs `VMValidateSignatureSingle` ([F] vm.h:690 → `VMCheckParamCount` and `VMCheckParam<DObject*>`, vmframe.cpp:578-584, 628-632: three out-of-line checks that read `func->Proto`) and then `VMCall` (705-737).

Client-side logic [F] src/p_tick.cpp:155-218 runs `totalTics` times a frame:
- a client-side reset walk (188-197);
- `RunClientSideThinkers` (199 → [F] dthinker.cpp:294-477). It ends with `recreateLights()` (lambda 318-331, called 384), a **second full walk over every world actor** plus the client-side actors, then `FDynamicLight::Tick` for every light (385-410).
- This whole block is outside [M]'s `ticprofile` and inside `ClientSideThinkCycles`.

**Inference — per real tic the main thread walks every world actor three times:**
1. the reset;
2. `RunThinkers`;
3. the recreate-lights scan.

In a netgame there is a fourth walk: `WorldStateHash` ([F] d_net.cpp:1235-1256, called at 1291 from `MakeConsistencies`, which runs at 2485 after each `G_Ticker`).

### B.3 What the 130.8 ms is made of (from [M], with the arithmetic)

**Averages per thinker:**
- 130.8 ms / 53,116 thinkers = **2.46 µs** per thinker per tic.
- With no mods: 3.24 ms / 8,065 = 0.40 µs ([M] §2).
- A stock monster is 0.27-0.47 µs ([M] §3).

**Inference, the engine floor:** if every thinker were stock-cost, 53,116 × 0.30 µs ≈ 16 ms. That is roughly the most Phase 9 "engine overhead" work can touch inside `RunThinkers`. The other ≈115 ms is script work and the engine work that script triggers.

**Breakdown of one profiled tic.** [M] §3's listed classes sum to ≈52.5 ms:

| ms | class |
|---|---|
| 16.46 | `RSQTHP_Bar` |
| 7.16 | `champion_CyanController` |
| 4.79 | `champion_Icon` |
| 4.24 | `SkeletonImp` |
| 2.23 | `Shifter` |
| 2.18 | `champion_Leaves` |
| 2.11 | `champion_DarkRedController` |
| 1.88 | `PhantomRevenant` |
| 1.86 | `SlayerKnight` |
| 1.53 | `champion_SilverController` |
| 1.31 | `CacoLich` |
| ≈4.3 | `champion_Aggro*` (4 classes) |
| 0.89 | `IA_Sound` |
| 1.57 | crits |

That leaves **≈78 ms in classes [M] does not list**, over ≈17,400 thinkers (53,116 − 35,681 listed calls), ≈4.5 µs each.

Two caveats:
- `profilethinkers` adds an FName `TMap` lookup and two `rdtsc` per thinker ([F] dthinker.cpp:1037-1042).
- The profiled tic is not the `ticprofile` run.

So the tail is an estimate. **It is the first thing P0.5-1 must resolve.** It cannot be planned against from [M] alone.

**The owner's own bar.** `RSQTHP_Bar` overrides `Tick` without `Super.Tick()` ([MOD] bar.zs:164-175). It is `+NOINTERACTION +NOBLOCKMAP` (116-117), so no engine tail runs. Its 4.5 µs is script:
- `EventHandler.Find("RSQVR_FxHandler")` per bar under a ceiling, a linear handler scan (215);
- `SetOrigin` when moved (221), which is a full relink;
- `Prev` written (222);
- per-camera scale using `players[consoleplayer]` (226).

That last read is play-scope, machine-local state written into an actor's `Scale`. It is not in `WorldStateHash`, so it never desyncs the hash. It is still a cross-machine divergence of sim state. **Inference:** any engine LOD that keyed on `Scale` or visibility would inherit it.

### B.4 The reset pass in the fork (P9.9): what is measured, and why Step B is blocked

**What it costs.** 9.02 ms / 53,116 = **170 ns per actor**. With no mods: 1.25 / 8,065 = 155 ns ([M] §2).

**Fields touched**, from `ClearInterpolation` [F] actorinlines.h:46-59 plus `ClearFOVInterpolation` 61-66, at the probed offsets:

| Offset (B) | Field |
|---|---|
| 96 | `Pos` |
| 168 | `Angles` |
| 224 / 240 | `Scale` / `Alpha` |
| 488 | `modelData` |
| 512-552 | `FollowBodyYaw*` (fork) |
| 600 | `FollowHandRot` (fork) |
| 888 | `PrevFollowHandRot` (fork) |
| 1112 | `Sector` |
| 1224 | `CameraFOV` |
| 1376 | `player` |
| 2208-2280 | `Prev`, `PrevAngles`, `PrevScale`, `PrevAlpha`, `PrevFOV` |
| 2320 | `PrevPortalGroup` |

- That is **15 distinct 64 B lines per actor**, plus the thinker link and class pointer (line 0), plus `Sector->PortalGroup`.
- The fork added lines 8, 9 and 13 (`// RS FORK` at actorinlines.h:52-56).
- Model actors add a ≈1 KB copy: 3 arrays × 32 slots × (4 + 12 + 16 B) ([F] actor.h:1312, 1326, 1346-1347, 1400-1408).

**Why prefetch did nothing.** 53,116 × ~1 KB ≈ 54 MB per tic. Over 9 ms that is 6 GB/s, far under DRAM bandwidth, so the walk is **latency-bound on the serial list chase**. That matches the renderer lane's finding that `sim_prefetch` changed nothing ([FR] dthinker.cpp:69-88; doc §1).

**Step A: a dense `TArray<AActor*>`.** It turns the walk into independent loads the CPU can overlap. The order is free, because each reset touches only its own actor.

Maintain it at:
- `FThinkerCollection::Link` ([F] dthinker.cpp:72-90, which includes the `[SLEEP]` list choice);
- `DThinker::Remove`/`OnDestroy` (1068-1080);
- `ChangeStatNum`.

Keep the sleep statnums 60/61 in it: today's iterator visits them ([F] dthinker.cpp:1620-1629, 1683-1729), and a sleeper can still be moved by another thinker.

**Inference:** 3-4× faster. 9.02 → 2.3-3.0 ms.

**Step B (lazy, exact for position) cannot be default on this modlist.**
- The playbook makes Step B fall back to Step A whenever any loaded script stores to `Actor.Prev`.
- The owner's mods do: [MOD] bar.zs:222, 499; rsqvr/words.zs:33; damnums/numbers.zs:250, glyphs.zs:30; QuestSaber/zscript.txt:3928, 3978, 3983, 4113, 4158, 4163.

**Fork-specific alternative, the plan of record (Inference).** The Step A-only fallback is the stopgap.
1. Teach the ZScript code generator to emit, after any store to `Actor.Prev`, a call to a native `AActor::NotePrevWrite(self)` that pushes the actor onto `moved[]`.
2. Do this by flagging the field at `DEFINE_FIELD` registration and checking the flag in the member-assignment emitter.
3. Angles, Scale and Alpha still need the full pre-tic pass over the co-located block, as the playbook says.

That makes Step B usable for position, which is the widest part of the record. Effort ≈1-2 person-weeks. Gate it on `-benchdemo` GREEN plus the netplay suite.

**The recreate-lights walk** (dthinker.cpp:318-331) reads `flags8` (offset 336), `effects` and `fountaincolor`. It runs per client tic.
- The renderer lane's doc measured client-side think at 5.4-5.5 ms on the owner's list at 39.5k thinkers ([FR] SIM_AT_MOD_SCALE.md §1).
- **Inference:** at ≈100 ns/actor × 39.5k ≈ 4 ms, most of that is this walk.
- On the dense array with those three fields co-located: −2 to −3 ms per tic.
- A native "flagged list" alone is not exact, because ZScript sets the bits with raw stores. That is the playbook's point, and it holds in the fork.

### B.5 `r_ticstability` in the fork is a VR hazard that grows with the tic

[F] src/d_net.cpp:2256-2270 spins until `stabilityticduration` has passed. That duration is the measured length of the **last tic**, capped at one second (2279-2284). It runs on every frame where `runTics <= 0 && totalTics <= 0` (2431-2436).

The VR path sets `doWait = false` (2357-2359) but never touches `r_ticstability` (default true, 2251).

**Inference:**
- **Today on the modlist:** every frame owes ≥1 tic, so the spin never runs.
- **Once tics fit 28.6 ms but exceed the 11.1 ms frame:** every tic-less frame spins the full tic length. At 90 Hz and 35 tics/s, 55 of every 90 frames.
- **With a sim thread:** if the spin stayed in the render loop, it would stall the render thread by a tic length on every frame.

The plan of record is to delete the spin and compute `TicFrac` from the predicted display time (VR) or from a smoothed frame-start timestamp (desktop). Forcing the cvar off in VR is the stopgap. Either way, removing it is a **hard prerequisite** for Phase 8. It is not a 2.3 ms nicety. It changes no sim state, so netplay is unaffected. The netplay lane owns `TryRunTics` and should co-sign.

### B.6 `WorldTick` (4.07 ms) and the fork's handlers

Dispatch is cheap: [F] src/events.cpp:2096-2105 returns before any VM call for empty overrides. So the 4.07 ms is handler bodies.
- **Engine-registered** ([F] wadsrc/static/mapinfo/common.txt:8): `VRStabilizeSyncHandler`, a `MAXPLAYERS` loop ([F] zscript/vr_stabilizesync.zs:17-28), and `OtherPlayerTagHandler`, which returns at once in single player (otherplayertags.zs:169-175). Both are µs.
- **The owner's RSQ set alone** has ≥27 `WorldTick` overrides (grep of [MOD] sources: VisualRewards 6, Score 6, DistanceGrab 13, GlowInTheDark, Graveyard, ShieldSaw) and 71 `ThinkerIterator.Create` sites.
- Third-party mods add more.

**Inference:** without per-handler attribution (P0.5-2), no engine change in Phase 9 can be aimed at this number. The levers are per-class actor lists (Phase 10.5) and mod fixes.

`WorldTick` runs before `RunThinkers` (p_tick.cpp:872 vs 881). Handlers that place helpers therefore read last tic's positions, as the playbook notes.

### B.7 Fork-only per-tic sim cost besides the reset

**Netgame hash** ([F] d_net.cpp:1235-1256):
```cpp
for (size_t i = 0; i < n; ++i) { h ^= b[i]; h *= 1099511628211ULL; }
```
- This runs over 48 + 24 B per actor: a serial chain of 72 dependent 64-bit multiplies. At ≈4 cycles a step, that is ≈60 ns, on top of a reset-like walk.
- **Inference:** 8-12 ms per tic in any 4-player game on this set. This cost is absent from every [M] number, which are all single player.
- **Plan of record:** a 9-word 64-bit mix (one multiply per word) over the dense array, covering every actor every tic. **Inference:** ≈5× cheaper, ≈1.5-2.5 ms at 53k actors, and full detection strength.
- **Stopgap only:** striping a quarter of the actors per tic (`denseIndex % 4 == gametic % 4`) is a further 4× cheaper, but delays detection by up to 3 tics.
- Both are deterministic. Both change the consistency values, which only same-build peers compare.

**VR chain** ([F] p_user.cpp:1988-2037). Per player per tic it runs:
- latch;
- canonical hand pose;
- torso;
- gear;
- gear-play;
- grab;
- **two 8192-unit `Trace` calls** (98-123, one per hand whose latched pose is valid, 107);
- room move;
- telemetry.

The avatar IK and body inference run **per frame on the render side** (`VRBodyInfer_*` callers are in [F] src/r_data/vr_body.cpp, vr_rig.cpp and src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:3748-3788; none are in playsim). So IK is not sim cost. It does share the main thread with the tic until a sim thread exists.

BD22 world weapons and gun fit are render-side too: `VRGunFit_*` is read by [F] src/r_data/models.cpp:2771, 2867 and by script natives.

In netgames `P_PredictClient` re-runs the player chain and the pawn tick for every predicted tic, every frame ([F] p_user.cpp:2226-2302).

**Inference:** ≤0.5 ms per tic for 4 VR players plus prediction. It is not map-scaled.

**Slow-mo.** `P_TickRealTimeActors` ([F] p_tick.cpp:79-120) does a full thinker walk plus a `static TArray` collect per real tic, only while the world clock is slowed (`RSQ_SlowMo` exists in the owner's pk3 list). With the modlist that walk costs ≈ a reset (≈9 ms) on every slowed tic. Put it on the dense array.

**Effects governor.** It is presentation-only. Its one playsim-adjacent hook trims GPU particles with a hash-keyed keep test ([F] src/g_levellocals.h:3004-3017), states "no playsim RNG… nothing in the simulation reads the ring", and draws no RNG. It does not reduce sim cost.

**Inference:** the fork has no sim budget governor. Phase 9.3(e) would be the first, and it must stay single-player and non-demo.

**Thinker sleep** ([F] dthinker.cpp:93-141). GZSelaco's `Sleep(n)`/`SleepIndefinite` is already in the fork.
- It is a zero-engine-work dormancy: mods can park periodic helpers today.
- It is deterministic (playsim tics, saved: `Serialize` at 1093-1105).
- The reset walk still visits sleepers. Keep that in P9.9, because sleepers can be moved by other thinkers.

### B.8 JIT status (F13)

- `vm_jit` defaults true where `HAVE_VM_JIT` is defined ([F] vmframe.cpp:39-44). The CMake option is ON for x86_64 ([F] CMakeLists.txt:304-306).
- Both are forced false by `-debug` or by the archived `vm_debug` ([F] d_main.cpp:4188-4195, cvar 3662 `CVAR_ARCHIVE | CVAR_GLOBALCONFIG`). A single debugger session leaves the owner's ini interpreting every script from then on.
- Per function, `CanJit` prints only the register-limit fallback (vmframe.cpp:303-309). `blockJit` functions fall back silently (300).
- There is no startup report. Q1 settles it in one boot.

### B.9 Determinism rules that apply to every row

The netplay suite ([F] tools/netplay/README.md) compares peers of **one build** tic by tic. It proves cross-machine determinism. It does not prove fidelity to stock.

**Inference:** a deterministic-but-inexact change stays IN SYNC. Examples: an incomplete dormancy predicate, or Tick LOD. The ways to break lockstep are:
- machine-local inputs: wall time (P9.3e, GC deadlines if GC were sim-visible), `consoleplayer`/camera reads in play scope (B.3), render-side writes (F9-style), sound-channel state;
- address-ordered containers.

Fidelity is gated by:
- `-benchdemo` with `perf_bench_hash` (SP, [F] g_perfbench.cpp:51-55, 140ff), compared against demos recorded on the stock-path build;
- the shadow-verify modes (`r_verifyrenderlists`, `sv_dormancy_verify`).

Every default-on row needs both gates. Opt-in rows (P9.3) need the netplay gate plus a `CVAR_SERVERINFO` cvar recorded in demo and savegame headers.

All changes to `TryRunTics`, `MakeConsistencies` and prediction are the netplay lane's code ([F] RENDER_LANE_HANDOFF_2026-11-09.md:197-200). Per ROADMAP order, default-on sim changes belong **after** step 3 freezes the suite, or behind off-by-default cvars on the `renderer` branch with the suite run on and off. That is how the render lane handled its own sim fixes. The read-only tooling (0.5, F13) can land anytime.

---

## 90 Hz contribution

### Today, per tic (Sunder MAP15, owner set, SP flat, [M] §2 rr347)

| Stage | ms |
|---|---|
| reset | 9.02 |
| particles/beams | 0.72 |
| player think | 0.07 |
| `WorldTick` | 4.07 |
| thinkers | 130.80 |
| **P_Ticker total** | **144.7** |
| client-side logic per tic (not in `ticprofile`; ≈5.5 ms measured by the renderer lane at 39.5k thinkers) | ≈5.5 |
| **Single-player total** | **≈150 ms** |
| 4-player co-op: `WorldStateHash` (B.7) | +8-12 |
| 4-player co-op: sight ×(1-4) | unmeasured |
| **4-player total** | **≈160 ms per tic** |

On main, before F-SIM-1 and F-SIM-2, it is worse.

### Budget with a sim thread (Phase 8)

The sim thread must hold 35 tics/s on its own core: **≤28.6 ms p99** (hard: above it the game slows, and in a netgame every peer waits) and **≤20 ms median**, leaving ≈8 ms of spike headroom.

Proposed per-tic allocation (Inference), 4-player VR:

| Item | Budget (ms) | Today (ms) | Gap (ms) |
|---|---|---|---|
| thinkers (`RunThinkers`) | 13.0 | 130.8 | −117.8 |
| interpolation reset | 1.5 | 9.02 | −7.5 |
| `WorldTick` handlers | 1.5 | 4.07 | −2.6 |
| particles/beams/effect queues | 0.7 | 0.72 | 0 |
| player think ×4 + prediction | 0.5 | 0.07 (1 player, flat) | unmeasured |
| client-side logic (lights walk, cs thinkers, `FDynamicLight::Tick`) | 1.0 | ≈5.5 | −4.5 |
| netgame consistency hash | 0.5 | 8-12 (est.) | −8 to −11 |
| snapshot write (render thread feed) | 1.0 | n/a (unbuilt; mode 2 = 24) | new cost |
| GC step (amortised) | 0.3 | unmeasured | — |
| **total** | **20.0** | **≈160** | **≈−140 (−88%)** |

The three named stages: 130.8 + 9.0 + 4.1 = **143.9 ms → ≤16 ms**. That is −128 ms, a **9× reduction**.

### Without a sim thread

The tic shares the 11.1 ms frame. At 90 Hz only 35 of 90 frames carry a tic, but each of those frames must fit `tic + client logic + render CPU + XR/submit` in 11.1 ms. Otherwise they miss, which is 39% of frames: not a steady 90.

Budget derivation (Inference):
- 11.1 ms
- − scene build ≈4.4 ms (with mods, flat, one view, [M] §1 4.35 ms; assumes Phase 2's single scene build for both eyes)
- − ≈1.5 ms for submit, XR calls and margin
- = **≈5 ms for the whole tic + client logic.**

Today that is ≈150 ms: a **30× gap**. Even the no-mod tic is 1.25 + 3.24 + 0.03 + 0.01 = 4.5 ms ([M] rr344) plus client logic, so it already fills the slot.

**Conclusion:** on the modlist, a sim thread is not optional for 90 Hz. Phase 0/9 work cannot close a 30× gap. With the thread, it must close a 9× gap together with Phase 10 and mod work.

### What this area can deliver (exact, default-on), with derivations

| Item | ms/tic | Derivation |
|---|---|---|
| P9.9 Step A reset | −6.0 to −6.7 | 9.02 → 2.3-3.0 |
| Light walk on the dense array | −2 to −3 | B.4 |
| F4/9.1a | −0.5 to −1.6 | 53,116 × 10-30 ns |
| F1 + 9.1b | −0.5 to −2.5 | needs counters |
| 9.1c | −0.5 to −2 | owned inventory |
| F2 | −0.3 to −1 | more on portal maps |
| 9.2 dormancy | −2 to −4 | stock-Tick idle actors |
| 9.4 BSP grid | −0.5 to −3 | relinks × 2 walks |
| 9.5/9.6/9.8 | −0.5 to −4 | fights and co-op, unmeasured |
| **SP sum** | **≈ −13 to −28** | ≈150 → ≈122-137 ms |
| F-SIM-4, 4-player only | −6 to −10 | word-wise hash on the dense array (plan of record) |

These rows close ≈10-20% of the gap.

**Opt-in, deterministic, approximate:** P9.3 Tick LOD, **−10 to −40 ms** depending on the classes the owner opts in (table row derivation).

**The rest (≈80-110 ms) is script work.** It is not reachable by engine fast paths:
- RSQTHP_Bar 16.5 → ≈0 via Phase 10 overlays, or parking plus one batch updater;
- mk-champions ≈25;
- LoB ≈11;
- crits 1.6;
- Shifter 2.2;
- the unattributed ≈78 ms (B.3).

P0.5's attribution decides how to split it.

---

## Open questions (each with the run that settles it)

All runs: bridge `boottest`, owner's 57-file zdl as used for rr342, `-Map MAP15` (Sunder), `-Tics 700`, `-ExtraArgs "+vid_vsync 0 +vid_maxfps 0 +cl_capfps 0 -rngseed 1113"`, `-ScriptErrorOk` as for the full list. Every new exe gets a new name (`scripts\build_named.py <name>`).

**Q1. Is the JIT on in the owner's build?**
- `-Exec jitcheck.cfg` with `wait 35; vm_jit; vm_jit_aot; vm_debug` on the current headset exe.
- Grep the boot log for `is using too many registers`.
- If `vm_jit` prints false, every number in [M] §3 is interpreted ZScript. Fix that before anything else.

**Q2. Main vs renderer on the same map.**
- Same zdl and spot on the newest main exe and on `doomxr_rend1113a.exe`, with `+perf_track 1`.
- Compare `think_ms`, `tics` per frame and the `Shifter` row of `profilethinkers 8 60` (`-Exec` with `wait 300; profilethinkers 8 60`).
- Shows the size of F-SIM-1/2 on main.

**Q3. The unattributed ≈78 ms.**
- Run `profilethinkers 8 400` (all classes) at `wait 300` and `wait 330`.
- Sum the output and compare it with `ticprofile` in the same run (renderer exe, `wait 300; ticprofile; wait 335; ticprofile`).
- Afterwards, `sim_profile` (P0.5-1) over 300 tics on a port build.

**Q4. Portal content of Sunder MAP15 and Abandon MAP13** (sizes F2/P9.7). Add a one-line ccmd printing `PortalBlockmap.containsLines`, `linePortals.Size()` and the sector-portal count.

**Q5. Client-side split** (sizes the light-walk saving in B.4).
- Renderer exe, `-Exec` with `wait 300; csprofilethinkers 8 40`.
- Plus `perf_track` `csthink_ms`.
- The `InternalDynamicLight` row versus the total gives `FDynamicLight::Tick`; the remainder minus the client thinkers is the recreate-lights walk.

**Q6. Sight.** Fix the `stat sight` unclock leak (p_sight.cpp:848, 853), then run `stat sight` SP vs a 4-player game on the rig with `NOMONSTERS="" bash tools/netplay/run.sh sight4 4 700` and the plain set. The rig cannot hold 4 × BD22 ([F] tools/netplay/README.md), so use a monster-heavy stock map.

**Q7. Netgame hash cost.** Clock `MakeConsistencies` on a port build in a 2-player game on the owner's PC: host plus a flat joiner (`REMA NETPLAY HOST 6.bat` + `scripts\nettest_join5.py`), owner set, Sunder MAP15.

**Q8. GC.** Read the `perf_track` `gc` column in the Q2 runs. `gc stats` at `wait 300`.

**Q9. VR player chain in the headset** (F-SIM-7). This is not runnable by the bridge. With the owner in the headset, use the renderer exe's `ticprofile` "player think" column, SP and then 2-player.

**Q10. Action-call and `PointInSector` counts per tic** (sizes F1, 9.1b, 9.4). Needs the P0.5-5 counters in a port build. Run the Q2 map with `wait 300; sim_counters`.

**Q11. Which heavy classes override `Tick`** (sizes 9.1a, 9.1c, 9.2). Dump `GetClass()->Virtuals[TickIndex] == stockTick` per class in `sim_profile`. The mod sources for mk-champions, crits and Universal Entropy are not in any tree here.
