# G — Use the whole machine: deterministic multi-core sim, faster script, data-oriented actors, one job system

Area: the structural "use the whole machine" track the owner asked for (OWNER_DECISIONS.md item 7). It covers four things on
UZDXREMA `main` @ e84623f:
1. deterministic parallel simulation;
2. faster ZScript execution (JIT and AOT);
3. data-oriented actor storage;
4. one job system shared by the sim and the renderer, plus what the GPU may take.

Every option is quantified against the owner's i9-9900KF (8 cores / 16 threads), RTX 3080 Ti and 16 GB RAM.

Sources: [F] `/home/claude/uzdxrema` @ e84623f, [FR] `/home/claude/uzdxrema_renderer` @ abb19af, [U] stock UZDoom 5.0.0,
[M] `10_MEASUREMENTS.md`. [A] = `fork/A_sim.md` (the playsim delta), [C] = `fork/C_frame_xr.md`, [D] = `fork/D_vulkan_gpu.md`.
[MOD] = the owner's RSQ sources at `/home/claude/presidentkoopa/rsq_ubqseries/`. Paths without a tree tag are [F] `src/...`.

**Inference** marks reasoning. Each estimate shows its derivation from [M] numbers. A **design bet** is a mechanism that no code
here proves; the code only shows that it can be built. **Proven by code** means the cited lines establish the fact.

## Key reading before the table

**What "10-20%" meant, and why it is not the ceiling.**
- [A]'s 10-20% is the sum of *exact, single-threaded engine-overhead* rows: the reset, light walk, call layer, dormancy and BSP
  grid ([A] "What this area can deliver").
- Those rows leave the structure as it is: one core, every thinker in list order, every ZScript call through the
  JIT/interpreter boundary.
- This document attacks the other 80-90% with three structural levers:
  1. run the script faster (closed-world JIT fixes, then AOT for the fixed, versioned mod set);
  2. shrink the bytes each actor pass touches (hot prefix, dense index, slabs);
  3. run thinkers on 6-7 cores with a schedule whose result does not depend on thread count or timing.

**The verdict (Inference, derivation in §C).**
- **Structural track alone** (on top of [A]'s exact rows): the 4-player modlist tic goes from **≈160 ms to ≈21-101 ms**,
  ≈56 ms in the median case. That removes **37-87% of the tic (65% median, a 2.9× cut)**, not 10-20%.
- **With Phase 10 helper overlays and opt-in Tick LOD:** median ≈35-40 ms, best case ≈16 ms.
- **An order of magnitude (≤16 ms)** is reachable only at the favorable end of three numbers this repo cannot yet measure:
  - the script share of thinker time (s);
  - the AOT speed-up over the JIT (k);
  - the fraction of thinker work that can run in the parallel phase (p ≥ 0.9).
- §D names the runs that settle s, k and p. Two are cheap and come first: `stat vm` and the classifier dry run.

**The determinism contract this design keeps (proven by code + design rule).**
- The netplay suite compares peers of *one build* tic by tic ([F] tools/netplay/README.md:151-174; [A] B.9). It proves
  cross-machine determinism. It does not prove fidelity to stock.
- Every parallel mechanism below therefore obeys one rule: **the committed state after a tic is a pure function of the state
  before it and the tic's commands. Thread count, timing and work stealing have no effect on it.**
- This makes a strong single-machine gate possible: `sim_mt_threads 1` and `sim_mt_threads 12` must give identical per-tic
  `WorldStateHash` values ([F] d_net.cpp:1235-1256), checked with `sim_mt_verify`. It runs before the four-machine suite, and
  on the Linux rig.

---

## Delta table

| ID | Item | Fork status | Where in fork | What is left | Done when (measurable) | Don't touch | Netplay/determinism impact | Est. saving on modlist (Inference) |
|---|---|---|---|---|---|---|---|---|
| F-MT-0 | Measure the three unknowns (s, k, p) | Not started; one tool exists | `ADD_STAT(VM)` [F] common/scripting/vm/vmframe.cpp:895-911 (script time, call count); `profilethinkers` [F] playsim/dthinker.cpp:1007-1048 | Add a `vmstats` ccmd that prints the `stat vm` numbers to the log, so the bridge can read them. Add a **classifier dry run**: run the effect analyzer (F-MT-9) on the loaded mod set and print, per class, L/M/S, the blocking native or store, and that class's ms from `sim_profile`. | One log per run with script ms/tic, script calls/tic, and Σms of L, M and S classes. | — | Read-only. | Enabler. It decides whether 10× is reachable (§C). |
| F-MT-1 | Thread-safety groundwork (no behavior change) | Not started | Globals: `actionParams` gamedata/info.cpp:146; `spechit`/`portalhit` playsim/p_map.cpp:109-110; sight `intercepts`/`portals` p_sight.cpp:89-90; `FPathTraverse::intercepts` p_maputl.h:397; `validcount` p_maputl.cpp:565,576, p_map.cpp:1551,3245,3330, p_sight.cpp:629,917; `headsecnode` p_secnodes.cpp:30; `FBlockNode::FreeBlocks` :447; `NextToThink` dthinker.cpp:65; `VMCycles`/`VMCalls` vmframe.cpp:61-62 and the JIT prologue jit.cpp:460-469; `ActionCycles` info.cpp:152,227; `SightCycles`/`sightcounts` p_sight.cpp:53-55; `GC::Root` dobject.cpp:219-221; `GC::Gray` dobjgc.cpp:583-585; FString refcount utility/zstring.h:72-92; FName table utility/name.cpp:135-153 | Make each one per-thread or per-transaction: thread_local scratch, per-thread line/sector/polyobj visit stamps, per-thread GC new-object and gray lists, immortal constant strings plus an atomic refcount option, a reader/writer lock on the name table, per-thread clocks. Serial behavior stays identical. | Netplay suite IN SYNC, and `-benchdemo` hashes **bit-identical** to the pre-change build (it is a pure refactor). | — | None; pure refactor. | 0 by itself. A few ms come back from F-VM-1/4, which delete the shared clocks. |
| F-MT-2 | One job system (sim + render) | Not started; `ctpl::thread_pool renderPool(8)` used by the render path only | [F] rendering/hwrenderer/scene/hw_bsp.cpp:87, 401, 1447 | A work-stealing pool with priority lanes and a **deterministic fold API**: tasks write slot `i`, and results are reduced in index order. It replaces `renderPool`. | Render A/B unchanged. The sim fold API gives an identical hash at 1 vs N threads. | Render lane owns hw_bsp.cpp: co-sign. | Deterministic by API: no completion-order reductions, no float atomics. | Enabler. |
| F-MT-3 | Engine-native passes in parallel (option a) | Not started | Reset loop [F] p_tick.cpp:719-757 (`ClearInterpolation` actorinlines.h:46-59); netgame hash d_net.cpp:1235-1256; recreate-lights walk dthinker.cpp:318-331, 384; `FDynamicLight::Tick` loop :385-410; `P_ThinkParticles` p_tick.cpp:760 | Run the reset, per-actor hash digests (folded in index order), the lights scan (scan in parallel, apply serially in index order) and the particle step over the dense array ([A] P9.9 Step A). Keep the snapshot instant (p_tick.cpp:741-755). | Reset ≤0.7 ms, hash ≤0.5 ms, client tic ≤1.2 ms at 53k actors. Hash identical across thread counts. | World-clock skip (p_tick.cpp:735). | Exact: per-actor work is order-free, and folds run in index order. | −5.5 to −6.5 ms/tic beyond [A]'s single-thread rows (§1.2). |
| F-MT-4 | Re-entrant sight + within-tic sight prefetch | Not started | `P_CheckSight` [F] p_sight.cpp:840-950: global `validcount` 629, 917; static arrays 89-90; RNG reject 878 | Per-thread visit stamps and scratch, so sight can run on any thread. Then an optional **prefetch**: before `RunThinkers`, compute sight for (monster in `A_Look` or `A_Chase`, each player) in parallel into a cache keyed by every input. The RNG reject stays outside the cache. | `stat sight` (after fixing its unclock leak, [A] P0.5-T) −50 to −80% in a 4-player game. | `pr_checksight` draw order (878). | Exact if the key covers positions, heights, sector plane epochs, polyobj and portal epochs, and line flag epochs. | 0-3 ms SP; up to ×4 in 4-player co-op (unmeasured, [A] Q6). |
| F-MT-5 | Parallel helper batch (option d) | Not started | Example class: `RSQTHP_Bar` [MOD] VisualRewards/pk3/zscript/rsqthp/bar.zs:164-240 | First product of the phase-A machinery (F-MT-8/9), restricted to **L classes**: Tick and actions write only self, owned parts and fresh objects, and anything shared is deferred. Run them on the pool; commit in canonical order. | `sim_mt_verify` silent; L-class ms ÷5-6 wall. | — | Rule D (F-MT-7) if any L class draws RNG; otherwise exact against stock *only if* nothing between a helper and the list end writes what it reads (not provable, so treat as rule D). | Known and probable helpers ≈28 ms of [M] §3 → ≈5-7 ms wall (§1.6). |
| F-MT-6 | Parallel movers (option b, full) | Not started | `AActor::Tick` p_mobj.cpp:4741-5440; `P_XYMovement` 2593; `P_TryMove`/`P_CheckPosition` p_map.cpp; spawn 5894-6106; damage p_interaction.cpp | **M classes** (monsters and missiles) in phase A, using **transactional movement natives**: they read blockmap/sector structures (frozen in phase A) plus a frozen public-prefix snapshot of other actors, write self, and defer relinks, line crossings, pushes, pickups and damage. At commit the move is re-validated against committed positions; on failure the actor is rolled back (undo log) and re-run serially at its slot. | Re-execution rate ≤5% on Sunder MAP15 hordes. `sim_mt_verify` silent. Suite IN SYNC. | `sectorportal_thinglist` order ([A] P0-F2). | Rule D. Deterministic by construction: phase A reads only frozen state; commit runs in canonical order. | The main lever: from ≈11 ms (LoB monsters) up to the whole unattributed ≈78 ms tail, depending on p (§1.6). |
| F-MT-7 | Rule-D RNG: per-actor deterministic streams | Not started | PCG step [F] common/engine/m_random.h:64-72; script RNG bound at compile time to `&pr_exrandom` or a named RNG, codegen.cpp:8464-8485; 78 named `FRandom` objects in playsim/*.cpp | Inside phase A, `GenRand32` draws from the running transaction's stream, keyed by (level seed, `NameCRC`, actor spawn key, gametic). Serial phases keep today's global streams. `CVAR_SERVERINFO sim_rule` is written to demo and savegame headers. | Same hash on 1 vs N threads. Old demos play back with `sim_rule 0`. | Client RNGs (`M_Random`, `FCRandom`) in serial code. | **Changes game dice versus stock**, deterministically. Peers of one build agree. Benchdemo goldens must be re-recorded under rule D. | Enables F-MT-5/6 for every RNG user. Cost +2-5 ns per draw (Inference: one TLS load + branch). |
| F-MT-8 | Phase-A runtime: snapshot, deferred effects, commit, rollback, verify | Not started | Wave structure exists: fresh thinkers tick in later waves, `RunThinkers` dthinker.cpp:174-182, `Link` 72-90 | Transaction context; foreign-load redirection to the public-prefix snapshot (JIT/AOT); self-store undo log; deferred-effect queue (relink, spawn linkage + netID + TID, destroy, damage, sound, events, line specials, inventory moves); canonical commit; S-class transactions executed inline at their slot; `sim_mt_verify` (serial vs parallel hash per tic). | Deterministic-equivalence gate green on the rig for 10k tics; suite IN SYNC with `sim_mt 1`. | Netplay lane's `TryRunTics`, `MakeConsistencies`, prediction ([F] RENDER_LANE_HANDOFF_2026-11-09.md:197-200). | Core of rule D. | Carries F-MT-5/6. Overhead ≈+15-25% on phase-A work (§1.3). |
| F-MT-9 | Static effect analyzer (closed world) | Not started | VM store opcodes `SB…SO` [F] common/scripting/vm/vmops.h:94-108; `LO` read barrier vmexec.h:238-246; virtual dispatch codegen.cpp:10366 + `OP_VTBL` vmbuilder.cpp:1169; natives: 1,116 `DEFINE_ACTION_FUNCTION_NATIVE` + 798 `DEFINE_ACTION_FUNCTION` (grep of [F] src) | Bytecode dataflow (self/owned/fresh/foreign provenance per address register), call graph with class-hierarchy analysis over the loaded class set, a native effect table (top ~300 natives by call count first; every unclassified native = S). | Classifier output stable across boots; a fuzz test (random S→L promotions disabled) shows `sim_mt_verify` catching mislabels. | — | A wrong label must never cause a desync, only a verify failure: the verify mode and the rig gate catch it before the suite. | Enabler. Decides p. |
| F-MT-10 | Spatial islands (option c) | Not started | Blockmap `FBlockThingsIterator` keeps its own hash, p_maputl.h:313-338 | Use islands only as a **scheduling hint** (co-locate transactions of one blockmap cluster on one worker to cut re-validation failures), not as a correctness boundary. | Re-execution rate falls at equal hash. | — | None (hint only). | Folded into F-MT-6. As a standalone partitioning scheme: rejected (§1.4). |
| F-VM-1 | C++→VM entry costs | Not started | `CallTick` dthinker.cpp:1584-1590 → `VMValidateSignatureSingle` vm.h:690 → checks vmframe.cpp:578-632 → `VMCall` 705-737, two `rdtsc` at 732, 736 | Cache one `TickEntry` per `PClass` (stock native → direct C++; script → `ScriptCall`/AOT symbol). Validate once per function. Make `VMCycles` per-thread, sampled behind `vm_profile`. | `stat think` −1.5 ms at 53k thinkers. | `IFVIRTUAL` index caching. | Exact. | −1 to −2.5 ms (53,116 × 20-45 ns). [A] P0-F4 covers part of it; do not double-count. |
| F-VM-2 | Native calls without a direct entry | Not started | Fallback `NativeScriptCall` vmframe.cpp:347-356 (`try` plus `Unclock`/`Clock`, i.e. two `rdtsc`, plus VMValue unpacking); JIT chooses direct only if `DirectNativeCall` is set, jit_call.cpp:71-91; virtual natives always fall back, :356-359 | Give the hottest plain natives (169 in p_actionfunctions.cpp, 44 in p_mobj.cpp, 32 in p_user.cpp, …) a `_NATIVE` direct entry. Drop the per-call `rdtsc`. Emit a direct call for virtual natives when CHA proves the target. | Calls through `NativeScriptCall` per tic, counted, −80%. | Exception stack-trace text. | Exact (same native, same args). | −2 to −10 ms (unmeasured count; 160k-530k calls/tic × 15-25 ns, §2.2). |
| F-VM-3 | Closed-world devirtualization + compile after all units | Not started | Eager JIT runs *inside* the per-unit compile loop, vmbuilder.cpp:955-958, so subclasses from later mods are unknown; `OP_VTBL` is emitted unless `VARF_Final`, codegen.cpp:10366 | Move `JitCompile` after `VMFunction::CreateRegUseInfo()` (vmbuilder.cpp:969). At that point every class is loaded: replace `VTBL`+`CALL` with `CALL_K` when no subclass overrides the method below the static receiver type. | Count of devirtualized sites printed. | — | Exact (same target). Peers must load the same mod set, which netplay already requires. | Part of F-VM-1..4's −5 to −20 ms. |
| F-VM-4 | JIT prologue and frame weight | Not started | `IncrementVMCalls` is a global load/add/store in every prologue, jit.cpp:460-469; every register is zeroed on entry, :391-398; a full heap VM frame whenever a function has strings, locals with constructors or extra space, :302-316, 409-431; params pass through the `VMValue` array in memory, jit_call.cpp:93-127, 129-237 | Count calls per thread (or not at all outside `vm_profile`). Zero only registers that are read before written. Keep string registers on the native stack with explicit destructors. | `stat vm` calls/ms ratio up. | — | Exact. | Inside F-VM-1..4's range. |
| F-VM-5 | State-action dispatch | Not started | `FState::CallAction` info.cpp:148-234: `ActionCycles` (two `rdtsc`), `CheckCallerType` with a TArray copy (135), a `DefaultArgs` copy (179), a global `actionParams` Reserve/Clamp | Per-`FState` prebuilt thunk: argument block built once at load, a direct `ScriptCall`/AOT pointer, no copies, no global. | 0 allocations from `CallAction` ([A] P0-F1). | `CVMAbortException` diagnostics. | Exact. | [A] P0-F1/P9.1b: −0.5 to −2.5 ms (do not double-count). |
| F-VM-6 | AOT: ZScript bytecode → C++ at pack-build time | Not started | The JIT already consumes the same bytecode ([F]=[U] jit/*.cpp, 0-line diff); math goes through `g_sin`/`g_pow`/`g_atan2` in both the VM and the JIT (vmexec.h:1492-1522, 2038ff; jit_math.cpp:874-1009) | Stage 1: a transpiler with the `ScriptCall` ABI (drop-in). Stage 2: typed direct calls, CHA devirtualization, inlining, null-check elimination for `self`. Manifest hash per function; a mismatch falls back to the JIT per function. | `stat vm` ms ÷k on the owner's set; all peers' AOT hash compared at net handshake. | DAP debugger (src/common/scripting/dap): AOT off when debugging, as JIT is today (d_main.cpp:4188-4195). | Must be FP-identical to JIT and interpreter: no FP contraction, no fast-math, same `g_*` calls, same null and divide-by-zero throws. | −11 to −44 ms (§2.5). |
| F-VM-7 | `FName` lookup allocates | Not started | `FName::NameManager::FindName` builds a lowered `std::string` on every lookup (name.cpp:137) and inserts into an unsynchronised map plus a growing vector (:146-150) | Stack buffer for the lowered key; insert under a lock; stable chunked string storage. | 0 heap allocations per `(Name)` string conversion. | — | Exact. | Small; needed for thread safety. |
| F-DOD-1 | Hot "public prefix" + dense index | Not started ([A] P9.8/P9.9) | `AActor` 3,104 B, hot fields spread over 15 lines ([A] B.4 offsets); offsets are registered by `DEFINE_FIELD` (vm.h:967-987) and baked into JIT code as constants | Reorder `AActor` so that (a) the per-tic engine pass fields ([A] B.4) and (b) the fields other actors read (pos, vel, angles, radius/height, health, flags, sprite/frame, target/master/tracer, `Sector`, floor/ceiling z, player) sit in the first 384 B. Add a `denseIndex`. | Reset ≤4 lines/actor; phase-A snapshot = one `memcpy` of 384 B/actor. Savegame round trip. | Native field *names*. Script subclass fields start at the parent's `Size` and do not move. | Exact. Addresses never feed the sim. | Inside F-MT-3, F-MT-8 and [A] P9.9. |
| F-DOD-2 | Slab pools per class size | Not started | `PClass::CreateNew`: `M_Malloc(Size)` + `memcpy(Defaults)` (dobjtype.cpp:483-509) | Per-thread slab pools keyed by `Size`. Free lists return in deterministic commit order. | allocs/tic in `CreateNew` → slab hits. | — | Addresses are not sim-visible. Safe. | 0-1 ms single-threaded; removes malloc lock contention in phase A. |
| F-DOD-3 | Engine-only SoA mirrors | Not started | `Pos` is script-readonly (wadsrc/static/zscript/actors/actor.zs:408) and written in C++ only through actor.h setters (16 `__Pos` uses: actor.h 11, p_mobj.cpp 1, vmthunks_actors.cpp 4); `Vel`/`Prev` are script-writable (:409, 424) | Mirror only engine-owned values (position via the setters, `PortalGroup`, light-scan bits) in dense arrays for the engine passes. Never make a mirror the authority for a script-writable field. | Mirror == field at every tic end (debug check). | Script-writable fields. | Exact. | Folded into F-MT-3. |
| F-JOB-1 | Core plan for the 9900KF | Not started | Threads today: [C] B.5 | R = render thread (pinned), S = sim coordinator, 6 cores + SMT siblings = shared pool; XR runtime and audio float. | `perf_track` per-thread busy. | — | — | Enabler. |
| F-GPU-1 | GPU work: presentation only | See [D] | [D] plan of record items 2-6 | Nothing from the sim goes to the GPU (peers have different GPUs and drivers). Particles, light culling, multiview, frames in flight, multi-draw indirect, as [D] says. | — | — | Must stay sim-invisible. | Frame-side ([D]). |
| P8 | Sim thread | Not started on main; snapshot writer on [FR] | [FR] r_snapshot.cpp; hazards [C] B.6 | The phase-A pool runs *under* the sim thread. [C] B.6 must close first. | [C] C.1. | — | — | Turns "tic ≤3 ms" into "tic ≤22 ms" ([C] C.2). |

---

## Findings

### 1. Deterministic parallel simulation

#### 1.0 What runs on one core today, and the shape of the tic

`P_Ticker` ([F] p_tick.cpp:644-920) runs these stages in order:
1. the reset walk (727-757);
2. particles (760);
3. `P_PlayerThink` per player (865-867);
4. `WorldTick` (872);
5. `RunThinkers` (881).

`RunThinkers` ([F] playsim/dthinker.cpp:149-182):
- wakes sleepers;
- ticks statnums 32..127 in list order, skipping 60/61;
- then repeats over the **fresh** lists until no thinker is newly spawned.

Every tick goes through `TickThinkers` (962-1000): the global `NextToThink` (65) keeps the walk safe against `Destroy` of the next node, then `CallTick` (1584-1590) does `IFVIRTUAL` → `VMCallVoid`.

**Proven by code:** the spawn path already has *waves*. A thinker spawned during a pass is linked into `FreshThinkers[stat]`
(`Link`, dthinker.cpp:72-90) and ticks in a later pass of the same tic. A deferred-spawn design that links new actors at commit,
in canonical order, reproduces that structure exactly.

#### 1.1 What blocks running any of this on two threads (all [F], with the fix)

| Blocker | Code | Kind of hazard | Fix (F-MT-1 unless noted) | Cost of the fix |
|---|---|---|---|---|
| `validcount` | One global int, incremented per query (p_maputl.cpp:565, 576, 1436, 1700; p_map.cpp:1551, 3245, 3330; p_sight.cpp:629, 917). Stamped into `line_t::validcount`, polyobj and sector `validcount` (r_defs.h:733, 1711, 1865), `AActor::validcount` (actor.h:2641) | Shared counter + shared stamps: two threads corrupt each other's visited sets | Per-thread stamp arrays indexed by line/sector/polyobj index, plus a per-thread epoch. The renderer's `validcount` (r_utility.cpp:1371) becomes the render thread's own ([C] B.6 A3). | Memory: (lines + sectors + polyobjs) × 4 B × threads. Sunder MAP15's counts are unknown; at 60k lines + 20k sectors that is 320 KB × 16 = 5 MB. |
| Static scratch arrays | `spechit`, `portalhit` (p_map.cpp:109-110); sight `intercepts`, `portals` (p_sight.cpp:89-90); `FPathTraverse::intercepts` (p_maputl.h:397); `actionParams` (info.cpp:146); `intersectors` (p_map.cpp:8755); `collected_ss` (a_dynlight.cpp:539) | Shared mutable scratch | `thread_local`. | ≈100-500 KB per thread. |
| Free lists | `headsecnode` + `secnodearena` (p_secnodes.cpp:30-31), `FBlockNode::FreeBlocks` (:447-474) | Shared allocators used by every relink | Phase A never relinks: relinks are deferred to commit (F-MT-8), so these stay serial. | 0 |
| Profiling clocks | `VMCycles[0]` `Clock`/`Unclock` per C++→VM call and per non-direct native (vmframe.cpp:351-353, 732-736); `VMCalls[0]++` inside every JIT prologue (jit.cpp:460-469) and the interpreter (vmexec.h:2312); `ActionCycles` (info.cpp:152, 227); `SightCycles`, `sightcounts` (p_sight.cpp:53-55) | Data races. The JIT's `VMCalls` write hits **one cache line from every core on every script call**: false sharing that would erase most of the speed-up | Per-thread counters; summed on read. | ~0 |
| `NextToThink` | dthinker.cpp:65, 883-891 | Serial list-walk protocol | Phase A walks a frozen `txs[]` array, not the list. | 0 |
| GC root list | Every `DObject` constructor pushes onto `GC::Root` (dobject.cpp:219-221, 228-230) | Shared linked list | Per-thread new-object lists, spliced at commit in canonical order. Sweep order then stays deterministic, and sweep calls script `OnDestroy` on unreachable objects. | 0 |
| GC barriers | `SO` store → `GC::WriteBarrier` (vmexec.h:458-462; dobject.h:372-378) → `Barrier` pushes to global `Gray` in `GCS_Propagate` (dobjgc.cpp:573-593). `AllocBytes` counters (dobjgc.h:213-220) | Shared list and counters | Per-thread gray stacks drained after phase A; per-thread byte counters. GC steps only between phases (today they run per frame on the main thread, d_net.cpp:2343-2346, d_main.cpp:1566). | 0 |
| Read barrier writes | The interpreter's `LO` calls `GC::ReadBarrier(T*&)`, which **writes `nullptr` back into the source field** when the pointee is dead (vmexec.h:238-246; dobjgc.h:137-145). The JIT's inline barrier does **not** write back (jit_load.cpp:229-261, `#if 1`) | Interpreter: a load from a foreign actor is a store to it. (Also an interpreter/JIT semantic difference; [A] P0-F13 already requires all peers to run one JIT mode.) | Phase A requires JIT or AOT code, never the interpreter. Destroys are deferred, so `OF_EuthanizeMe` cannot flip during phase A. | 0 |
| FString refcount | Plain `RefCount++`/`--` (zstring.h:72-92) | Shared strings (constant `KonstS`, class defaults) race on copy | Mark constant and default strings immortal: a new sentinel, handled in `AddRef`/`Release`, with no count traffic. Optional atomic refcount for the rest. | +0-1% (Inference: actor code mostly uses `Name`, not `String`). |
| FName table | Lowered `std::string` key allocated per lookup; insert into the map + `push_back` (name.cpp:135-153) | Rehash or vector growth while another thread reads | RW lock on miss; chunked stable storage (F-VM-7). | ≈0; removes an allocation per conversion. |
| Allocation | `M_Malloc(Size)` per object (dobjtype.cpp:485) | Allocator lock contention | Per-thread slabs (F-DOD-2). | Memory: ≤1 slab page per size class per thread (≈64 KB × classes in use × threads; bounded to ≈16-32 MB by a pool cap). |
| Net IDs | `AddNetworkEntity`: first-come IDs from a free list (dobject.cpp:731-751) | ID order depends on spawn order | Assign at commit, in canonical order. | 0 |
| `SpawnOrder` | `actor->SpawnOrder = Level->spawnindex++` (p_mobj.cpp:5698) | Order-dependent counter | At commit, in canonical order. Phase-A RNG uses a provisional spawn key = (parent key, child sequence) (F-MT-7). | 0 |
| Synchronous events | `WorldThingSpawned` (p_mobj.cpp:6106), `WorldThingDestroyed` (6235), `WorldThingDied` (p_interaction.cpp:417), `WorldThingDamaged` (1649, 1695) run handler ZScript inline | Handlers read and write anything | Deferred-effect queue: the event fires during commit, at the effect's canonical position. | 0 |
| Sound | `S_StartSound`/`ChangeSoundPitch` walk the global `Channels` ([A] F-SIM-2) | Machine-local, shared | Deferred-effect queue (sound is not sim state). | 0 |
| RNG | Script `random()` binds a pointer to `pr_exrandom` or a named stream at compile time (codegen.cpp:8464-8485); 78 static `FRandom` streams in playsim; PCG step (m_random.h:64-72) | One global stream per name: every draw depends on how many draws came before, in list order | Exact mode: an RNG stream is a conflict object (§1.3). Rule D: per-transaction streams (F-MT-7). | +2-5 ns per draw |
| `EventHandler.Find`, `ThinkerIterator` from play code | e.g. [MOD] bar.zs:215 | Reads global lists. Safe if phase A never mutates them (deferral makes that true) | Allowed as reads. A `ThinkerIterator` sees the frozen phase-start list: rule-D semantics. | 0 |
| Machine-local reads in play scope | `players[consoleplayer]` in the bar's Tick ([MOD] bar.zs:226); [A] B.3 | Already a cross-machine divergence in serial | Not made worse. The analyzer flags it per class as a pre-existing hazard. | — |

**Inference:** none of these is deep. Every one is either per-thread scratch, an order-dependent counter that moves to commit, or a
side effect that moves to commit. The hard part is not thread safety. It is **semantics**: list order is part of how the sim
behaves. §1.3 decides what replaces it.

#### 1.2 Option (a): engine-native phases that are pure or partitionable. Verdict: do all of them (exact)

The serial costs below come from [A]'s single-thread plan of record. E is the parallel efficiency for memory-latency-bound walks
on 6 cores + SMT: **E ≈ 4-5** (Inference). Multiple cores overlap misses, but dual-channel DDR4 bandwidth caps the gain; the FR
doc measured parallel copies bandwidth-bound at 12-13 GB/s, [C] B.7.

| Pass | Serial after [A] | Parallel design | Wall after | Saving | Exact? |
|---|---|---|---|---|---|
| Interpolation reset (p_tick.cpp:727-757) | 2.3-3.0 ms (dense array + co-located fields) | Chunks over the dense array; `ShiftSurfacePositions` per model actor in the same chunk | 0.5-0.7 | −1.8 to −2.3 | Yes: each reset touches only its own actor ([A] B.4) |
| Netgame hash (d_net.cpp:1235-1256) | 1.5-2.5 ms (word-wise 64-bit mix, [A] F-SIM-4) | Each chunk produces 64-bit digests per actor in slot order, then one serial fold over 53k u64 (≈30 µs) | 0.35-0.55 | −1.2 to −2.0 | Deterministic. The value differs from today's FNV, which same-build peers accept ([A] F-SIM-4). |
| Recreate-lights scan (dthinker.cpp:318-331) | ≈1.0-1.5 ms on the dense array ([A] B.4) | Parallel scan writes a bitmask of actors needing a light change; serial apply in index order | 0.3-0.4 | −0.7 to −1.1 | Yes (light creation stays serial) |
| `FDynamicLight::Tick` (dthinker.cpp:385-410) | ≈1.5 ms (Inference from [A] B.4: 5.5 − 4 walk) | Per-light parallel animate. The flicker RNG `randLight` (a_dynlight.cpp:42) is a client `FCRandom`: give it a per-light stream | 0.4 | −1.1 | Presentation-exact (client RNG only) |
| Particles (`P_ThinkParticles`, p_tick.cpp:760; no RNG in the step, p_effect.cpp:282ff) | ≈0.7 (with beams) | Chunked step; the free list is rebuilt serially | 0.25 | −0.45 | Yes |
| Render snapshot write ([FR] r_snapshot) | new, ≤1.0 ([A] F-SIM-11) | Chunked copy of the hot prefix | 0.25-0.4 | (avoids +0.6-0.75) | Read-only |
| Sight (F-MT-4) | unmeasured | Re-entrant + prefetch | — | 0-3 SP, more in 4-player | Yes, with full keys |
| Sound update (`S_UpdateSounds`, [C] B.6 G) | small | Stays serial on S after commit | — | — | — |
| **Total** | | | | **≈ −5.3 to −7.0 ms/tic** (sight excluded) | |

#### 1.3 Option (b): the two-phase tick. Verdict: build it; it is the main lever

**Why "frozen world, commit in a fixed order" cannot be exact against stock.**
- Stock semantics are "each thinker sees all writes of thinkers before it in the list".
- A frozen snapshot changes what later thinkers see.
- Exactness would need serializable execution in list order: Block-STM-style optimistic execution with *version tracking on
  every read and write*, including native C++. The playsim writes actor fields directly in hundreds of places (p_mobj.cpp,
  p_map.cpp, p_enemy.cpp, p_interaction.cpp), so object-level write tracking in C++ is not affordable.
- And the RNG kills it: every `random()` advances one global stream (codegen.cpp:8464), so every RNG-using thinker conflicts with
  every earlier one. **Design consequence (Inference):** an exact mode can parallelize only RNG-free work whose reads nobody
  earlier wrote. That is the engine passes (§1.2) and some helpers. It is not the bulk.

**So the plan of record is rule D, a versioned deterministic game rule** (`CVAR_SERVERINFO sim_rule 1`, written to demo and save
headers). One tic under rule D:

```
P0  serial   sleep cycle, STAT_PLAYER and statnums below the bulk lists, as today
P1  build    txs[] = dense snapshot of the bulk lists in list order; tx i carries class category L/M/S
             public-prefix snapshot: memcpy of the first 384 B of every M actor (and of every actor
             that some phase-A class reads as foreign, per the analyzer)
P2  parallel phase A: every L and M tx ticks on a worker
               - self, owned and fresh objects: written in place (self-store undo log kept for M)
               - foreign actors: loads redirected to the snapshot (public prefix) or proven unwritten (cold fields)
               - shared structures (blockmap, sector lists, thinker lists, TID hash, net IDs, sounds,
                 events, line specials, damage, inventory transfers, destroys): appended to the tx's effect queue
               - RNG: the tx's own streams (F-MT-7)
P3  serial   commit in canonical order i = 0..n-1:
               L/M tx: apply its effect queue; for M, first re-validate each committed move against
                       committed positions; on failure undo the tx and execute it serially here
               S tx:   execute it here, live, in order (it sees all effects committed before it)
P4  waves    the fresh thinkers spawned by P2/P3 form the next txs[] (same as the FreshThinkers loop
             dthinker.cpp:174-182), until none are left
```

**Determinism argument (design + code facts).**
- P2 reads only (a) the snapshot, (b) structures nobody mutates during P2 (relinks, spawns and destroys are deferred), and
  (c) the tx's own objects and streams.
- So each tx's P2 result is a function of its inputs, not of scheduling.
- P3 is serial in a fixed order.
- Hence the committed state is independent of thread count. That is testable on one machine (`sim_mt_verify`: run P2 with 1
  thread and N threads, compare per-tic hashes).
- **Design bet:** the analyzer's labels are conservative enough that no L/M tx reads something another P2 tx writes in place.
  The verify mode catches a mislabel as a hash split on the rig before any netplay run.

**How classes are labelled (F-MT-9, static analysis of compiled VM code, closed world).**
1. **Provenance per address register, per function.** Track where each address register came from:
   - `a0` = self in methods;
   - `NEW` result or `Spawn` return = fresh;
   - a load from a field the class declares as an owned part (e.g. the bar's `over/mark/stat/boss/dmark/gemP/champP/mutP/chunkP`,
     [MOD] bar.zs:136-147) = owned. Owned = only stored into by this class, created by this class, and destroyed in its `OnDestroy`;
   - anything else = foreign.

   The store opcodes `SB…SO` ([F] vmops.h:94-108) with a foreign base make the function **S**. `LO`, `LP` and the other loads
   with a foreign base record the field offset as a *foreign read*.
2. **Call graph:**
   - `CALL_K` gives the target directly;
   - `VTBL`+`CALL` resolves to every override below the static receiver class, over all loaded classes (closed world: the mod set
     is fixed and versioned);
   - function-pointer calls (which also block the JIT, codegen.cpp:10010) make the caller S.
3. **Native effect table.** Each native is one of:
   - **pure/read** (math, distance, `CheckClass`, `GetDefaultByType`, `EventHandler.Find`, frozen-structure iterators);
   - **self-write** (`SetState` on self with actions recursively analysed, `A_SetScale`, `SetOrigin` with the relink deferred);
   - **deferred** (`Spawn` with linkage deferred, `Destroy`, `A_StartSound`, `DamageMobj`, `A_GiveInventory` on a foreign target);
   - **transactional** (the movement family, F-MT-6);
   - **serial** (everything unclassified, ACS, level changes, `ThinkerIterator.Create` on mutable statnums, console and cvar writes).
4. **Field write-sets per class.** For each class, the union of self-field offsets it writes in phase A. A foreign read of a
   field outside the 384 B prefix, which some phase-A class writes in place, demotes the *reader* to S.
5. **Output:** L (no transactional natives), M (has some), S. Plus the reason for every S. This is the classifier dry run in F-MT-0.

**Deferred spawns: what a spawning tx sees (design).**
- `Spawn` runs on the worker: `CreateNew`, defaults, `BeginPlay`.
- It returns the real pointer, so script code like `let th = Spawn(...); th.target = self; th.Vel = ...` works unchanged: the new
  object is *fresh*, so it is in the tx's own domain.
- Deferred to commit: thinker list link, sector/blockmap link, net ID, TID hash, `SpawnOrder`, `WorldThingSpawned`.
- `PostBeginPlay` already runs at the first tick ([F] dthinker.cpp:979-985). That is the next wave, as today.

**Deferred destroys (design).**
- `Destroy` in P2 queues the destroy and adds the object to a tx-local "destroyed" set.
- The JIT/AOT read barrier checks that set only when it is non-empty, so the tx itself sees `null` as in serial semantics.
- Other txs keep seeing the object until commit.

**Movement (F-MT-6, design).**
- `P_TryMove`/`P_CheckPosition` get tx variants. They iterate the (frozen) blockmap and lines, read candidate actors from the
  snapshot, record into the effect queue (crossed specials, `spechit`, pushes, touches, telefrags), and write self (position,
  floor/ceiling z, `Sector`).
- At commit, a cheap re-check of the final position against already-committed movers (radius overlap with actors in the
  destination cells) decides whether to apply or to undo-and-re-run.
- **Inference on the failure rate:** a conflict needs two movers to claim overlapping space in one tic.
  - At Doom monster speeds (8-20 units/tic) and radius 20-31, only movers in the same or adjacent 128-unit cells can conflict.
  - In a dense Sunder horde a few % of moving monsters per tic is plausible.
  - Re-running costs one serial tick each. The rig measures it (§D).

**Overhead per phase-A tx (Inference):**

| Item | Cost |
|---|---|
| Tx setup | ≈50 ns |
| Foreign-load redirect (one compare on a per-object snapshot flag + base swap) | ≈1-2 ns per foreign load |
| Self-store undo log (M only) | ≈2-4 ns per store |
| Effect-queue append | ≈10-20 ns per effect |

Against a 2.46 µs average thinker ([A] B.3) that is **o ≈ 1.15-1.30**.

**Commit cost (Inference):**
- relink ≈150-400 ns (two `PointInSector` walks, [A] P9.4);
- deferred spawn linkage ≈0.3-0.5 µs;
- damage = the full serial `P_DamageMobj` chain with its script virtuals and events.

Commit runs serially, so it is modelled as **c ≈ 5-10%** of the phase-A work, plus re-executions.

#### 1.4 Option (c): spatial/island partitioning. Verdict: a scheduler hint, not a correctness boundary

Islands, meaning groups of actors that cannot interact within a tic, look attractive. In this sim they cannot be made sound:
- hitscan and line traces reach 8192 units (e.g. the fork's own laser traces, [A] F-SIM-7);
- `A_Look`/`P_LookForPlayers` couples every awake monster to every player;
- `ThinkerIterator`, `EventHandler` reads and `level.*` globals couple everything;
- 4-player co-op spreads players across the map.

**Inference:** a sound island partition degenerates to "one island" on Sunder-class maps. The useful part is locality: putting
transactions from one blockmap cluster on one worker cuts F-MT-6 re-validation failures and shares cache. F-MT-10 is that hint.
It never changes results.

#### 1.5 Option (d): helpers that only follow a master. Verdict: the first deliverable of (b)

**`RSQTHP_Bar` read against the analyzer ([MOD] bar.zs).**

The bar is `+NOINTERACTION +NOBLOCKMAP` (116-117) with no `Super.Tick()` (164-175). Its `Tick`/`Update`:
- reads `who.*`, `h.*`, `level.maptime`, `players[consoleplayer]` and `EventHandler.Find` (169-226): all reads;
- `Destroy()` itself (deferred);
- `SetOrigin(np)` (221): relink deferred, position written to self;
- `Prev = …` (222): self;
- `Scale = …`, `bBright` (229-230): self;
- `BuildFrame` and `SyncParts` write the owned part actors (declared at 136-147, destroyed in `OnDestroy`): owned.

**Inference:** the bar is **L**, with zero RNG unless `BuildFrame` draws (not read). Its master `who` is a monster: an M or S
actor. Under rule D the bar reads `who` from the snapshot. That is the monster's state *at phase start*, i.e. last tic's
end-state, one tic stale for the HP number and the position.

That staleness is visible: the bar would trail the monster by a tic. Two fixes, both deterministic:
- **(i)** helpers that read their master's public prefix are scheduled in a **second phase-A wave after the M wave commits**.
  They then read post-tick masters, which is what stock gives a bar that ticks after its monster;
- **(ii)** they read `who`'s live object, since M txs of the first wave have already committed.

(i) is the plan of record. It costs one more barrier (≈20-50 µs).

**The other helper families** (from [M] §3; sources not in any tree here, so each label is a **design bet** until the classifier runs):
- `champion_Icon` 4.79 ms and `champion_Leaves` 2.18 ms: probable L;
- crits `csh_critController` + `csh_Brain` 1.57 ms: probable L/M;
- `Shifter` 2.23 ms: sound-pitch per tic, so L with a deferred sound op ([A] F-SIM-2);
- `IA_Sound` 0.89 ms: L with deferred sound ops;
- `champion_*Controller` ≈10.8 ms and `champion_Aggro*` ≈4.3 ms: unknown. Controllers that buff their master write a foreign
  object, so S unless the write becomes a deferred "apply to master" effect.

**Overlap with Phase 10.** The permanent route for presentational helpers is engine overlays/attachments ([A] table "Per-monster
helper thinkers"). Phase 10 *removes* the work. Option (d) *spreads* it over cores. Where Phase 10 lands, (d)'s saving on that
class becomes 0. The combined projection in §C does not count both.

#### 1.6 What fraction of the 130.8 ms becomes parallel, and the wall time on this CPU

**Classes, from [M] §3 (one profiled tic, Sunder MAP15, owner set):**

| Group | ms | Likely label | Basis |
|---|---|---|---|
| `RSQTHP_Bar` | 16.46 | L | Source read (§1.5) |
| `Shifter`, `IA_Sound` | 3.12 | L (deferred sound) | Behaviour per [A]; source not read |
| `champion_Icon`, `champion_Leaves`, crits | 8.54 | probable L/M | No source |
| `champion_*Controller`, `champion_Aggro*` | 15.1 | L/M/S unknown | No source |
| LoB monsters (`SkeletonImp`, `PhantomRevenant`, `SlayerKnight`, `CacoLich`, …) | ≈11 | M (needs F-MT-6) | Monster behaviour |
| Unattributed tail | ≈78 | unknown | ≈17,400 thinkers at ≈4.5 µs each ([A] B.3) |

**Three scenarios for p (the share of thinker time that runs in phase A):**
- **low p = 0.30:** proven and probable L only. (16.46 + 3.12 + 8.54) / 130.8 = 21%, plus a little of the tail.
- **mid p = 0.60:** plus LoB monsters and half the controllers and tail through F-MT-6.
- **high p = 0.85:** plus most of the tail. Only the classes that hit serial natives (ACS, iterators on mutable lists, foreign writes
  without a deferred form) stay serial.

**Wall-time model (Inference):**

`wall = T·(1−p) + T·p·o/E + T·p·c + B`

| Symbol | Meaning | Value |
|---|---|---|
| T | Thinker time at that point in the plan | — |
| o | Phase-A overhead (§1.3) | 1.15-1.30 |
| E | Effective throughput of the pool for this work | 6.0 (range 5.5-7.5) |
| c | Serial commit + re-execution | 0.05-0.10 |
| B | Snapshot + barriers | 0.8 ms |

**Where E comes from.**
- 16 hardware threads, minus:
  - R (the render thread, one core);
  - S (the coordinator, which joins the pool during phase A);
  - the XR runtime and compositor (≈1 core-equivalent, unmeasured);
  - audio and OS.
- That leaves **6 cores + their SMT siblings**. SMT adds roughly +20-35% on memory-latency-bound code, giving 7.2-8.1
  core-equivalents.
- All-core turbo on the 9900KF is lower than single-core turbo (≈−6%).
- The memory-bandwidth ceiling: 53k actors × 2-6 KB touched per tick ≈ 100-320 MB per tic. Over a 10 ms phase that is
  10-32 GB/s, near dual-channel DDR4 limits (RAM speed unknown, §D).
- So E ≈ 5.5-7.5.
- DOD (F-DOD-1/2) shrinks the bytes per actor, which raises that ceiling.

**On today's 130.8 ms, as a pure what-if with T = 130.8, o = 1.25, c = 0.08:**

| p | Serial part | Parallel part | Commit | B | Wall | Cut |
|---|---|---|---|---|---|---|
| 0.30 | 91.6 | 8.2 | 3.1 | 0.8 | **103.7** | −21% |
| 0.60 | 52.3 | 16.4 | 6.3 | 0.8 | **75.8** | −42% |
| 0.85 | 19.6 | 23.2 | 8.9 | 0.8 | **52.5** | −60% |

**Inference:** parallelism alone gives at most ≈2.5× on thinkers, because of the serial share and the commit. It multiplies with
faster script (§2) and smaller actors (§3). §C composes them.

### 2. Faster script execution

#### 2.1 Is the JIT on, what it compiles, what falls back (proven by code)

**Build and switches.**
- `HAVE_VM_JIT` is ON for x86_64 ([F] CMakeLists.txt:304-306, [A] B.8).
- `vm_jit` and `vm_jit_aot` default true (vmframe.cpp:39-53).
- Both are forced false by `-debug` or by the archived `vm_debug` (d_main.cpp:3662, 4188-4195). One debugger session can leave
  the owner's ini interpreting everything.

**When functions are compiled.**
- With `vm_jit_aot`, each function is JIT-compiled **while its own compile unit is built** (vmbuilder.cpp:955-958).
- Otherwise it compiles lazily on first call (`FirstScriptCall`, vmframe.cpp:135, 332). That writes `ScriptCall` on first use,
  which is not thread-safe, so phase A requires the eager mode.

**What falls back to the interpreter, silently or with one log line:**
- `blockJit` functions, i.e. those that call through a function pointer (codegen.cpp:10010; vmframe.cpp:302);
- functions with ≥200 registers (vmframe.cpp:303-309, a log line);
- `-debug`/`vm_debug` (everything).

**The VM and JIT are stock.** The [F] VM/JIT files diff to 0 lines against [U] (`jit/*.cpp`, `jit/*.h`, `vm.h`, `vmexec.h`,
`vmintern.h`, `vmops.h`). `vmexec.cpp` differs in the engine-select fix only; `vmframe.cpp` in log text and a `va_end`.

[A] Q1 settles on/off in one boot. If it is off, enabling it is the single biggest script win (§2.5).

#### 2.2 What a call costs today (anatomy, with code)

| Call kind | Path | Fixed cost (Inference, 4.7 GHz) |
|---|---|---|
| C++ → script (`CallTick`, actions, events) | `IFVIRTUAL` lookup → `VMValidateSignatureSingle`: 3 out-of-line checks reading `func->Proto` (vm.h:690; vmframe.cpp:578-632) → `VMCall`: empty-function test, `VMCycles` `Clock`/`Unclock` = 2 `rdtsc` + 2 FP read-modify-writes of a global double (705-737) → `ScriptCall` indirect | ≈25-45 ns |
| Action call (`SetState` → `CallAction`) | `ActionCycles` 2 `rdtsc`, `CheckCallerType` TArray **copy** (info.cpp:135), `DefaultArgs` **copy** (179), global `actionParams` reserve/clamp, then `VMCallAction` → as above | ≈80-200 ns (two mallocs/frees dominate) |
| Script → script (JIT) | Args stored to the `VMValue` array in memory (jit_call.cpp:129-237); `FillReturns`; indirect `call [ScriptCall]` (:110-122). Callee prologue: global `VMCalls` increment (jit.cpp:460-469), args loaded back from memory (:322-389), **every register zeroed** (:391-398). A function with strings or constructed locals takes a heap frame: `AllocFrame` + `VMFillParams` + all registers loaded from memory (:302-316, 409-431) | ≈8-15 ns simple; ≈40-80 ns with a full frame |
| Script → native with `DirectNativeCall` (1,116 `_NATIVE` natives) | Direct C call with register args (jit_call.cpp:352-520) | ≈2-5 ns + body |
| Script → native without it (798 plain `DEFINE_ACTION_FUNCTION`, incl. 169 in p_actionfunctions.cpp, 44 p_mobj.cpp, 32 p_user.cpp) | `EmitVMCall` → `NativeScriptCall`: `try`, `Unclock`+`Clock` = 2 `rdtsc` (vmframe.cpp:347-356) → `PARAM_*` unpacking from `VMValue` | ≈15-30 ns + body |
| Virtual call | `VTBL`: 3 dependent loads (jit_call.cpp:50-63), then as script→script. A virtual *native* can never go direct ("not implemented", :356-359) | +3-5 ns; natives fall to the slow row |
| Object field load | Inline null check + inline read barrier (jit_load.cpp:229-261) | ≈1-2 ns |
| String register ops | Out-of-line helpers (`CallAssignString`, casts: jit_load.cpp:48-224, jit_move.cpp:41-178) | ≈10-30 ns |

**Inference on where the 2.46 µs average thinker goes.**
- A modded thinker at 2.5-4.5 µs ≈ 12,000-21,000 cycles.
- Pure JIT opcode execution at ~1-2 cycles/op would need 6,000-20,000 ops per tick. Per-actor ticks are much shorter than that.
- So most of the time is in **natives, call overhead and cache misses**, not in arithmetic.
- That puts a ceiling on what any code generator can do. It is why §2.5's ranges are modest, and why the call path matters more
  than codegen quality.

#### 2.3 JIT fixes (F-VM-1..5), exact

| Fix | Detail | Saving |
|---|---|---|
| Per-class `TickEntry` | Stock native → direct C++; script → `ScriptCall`/AOT pointer. Validate once per function. Clocks per thread, only under `vm_profile` | 53,116 × 20-40 ns = **1.1-2.1 ms** (part in [A] P0-F4) |
| Direct entries for the hot plain natives | Plus CHA devirtualization of virtual natives | N_nd calls/tic × 15-25 ns. N_nd is unmeasured; at 3-10 per thinker it is 160k-530k → **2.4-13 ms** |
| Closed-world devirtualization | Compile after all units (vmbuilder.cpp:969) | Per virtual site 3-5 ns + enables direct natives |
| Lighter prologue | Zero only live-in registers, no global counter, strings kept on the native stack | Per script call 3-10 ns |
| `CallAction` thunks | [A] P0-F1/P9.1b | 0.5-2.5 ms |
| **Sum beyond [A]'s rows** | | **≈ −5 to −20 ms/tic (mid −10)** |

#### 2.4 AOT for the fixed, versioned mod set (F-VM-6). Design

The mod set is fixed and versioned (CLAUDE.md goal; netplay requires identical sets). Compiling it ahead of time is therefore
legitimate, and the closed world makes CHA sound.

**Stage 1 (drop-in, 4-6 pw).**
1. A `-dumpaot <dir>` boot runs the normal compile (all 57 mods + engine pk3s) and walks `VMFunction::AllFunctions`
   (vmframe.cpp:64).
2. For each `VMScriptFunction` it emits a C++ function with the **`ScriptCall` ABI** (vm.h:447). Its body translates each VM
   opcode the way `vmexec.h` interprets it:
   - field offsets and constants inlined as literals;
   - `g_sin`/`g_pow`/`g_atan2` called exactly as the VM does (vmexec.h:1492-1522, 2038ff);
   - the same `ThrowAbortException` checks for null and divide-by-zero;
   - the inline read barrier (no write-back, like the JIT).
3. clang-cl compiles everything into `rema_aot_<hash>.dll` with `-ffp-contract=off`, no fast-math, SSE2 scalar doubles.
4. At load, the engine compares each function's bytecode hash plus the field-offset table hash. On a match it sets
   `ScriptCall = aot_symbol`; on a mismatch, that function stays JIT.

**Stage 2 (the speed, 6-10 pw).**
- Emit typed direct C++ calls between AOT functions (no `VMValue` marshalling), and direct calls to every native's C++ body
  (thunks generated from the `PARAM_*` signatures).
- CHA devirtualization; inline accessors and small helpers (the optimizer does this across the DLL with LTO).
- `self` is non-null after entry, so its per-load null checks go.
- Keep `self` fields in registers across native calls that the effect table marks as not writing self.

**Netplay and determinism.**
- The DLL hash joins the mod-set hash in the net handshake. A mismatch refuses to connect, as a mod mismatch should.
- **FP identity is a requirement, not a hope.** clang must not contract `a*b+c` into an FMA, nor reorder FP. The JIT emits plain
  SSE2 (`movsd`/`mulsd`); the C++ interpreter is compiled by MSVC with `/fp:precise`. The rig gate for AOT: identical per-tic
  `WorldStateHash` for interpreter, JIT and AOT over the benchdemo set (`-benchdemo` + `perf_bench_hash`, [A] B.9).

**Debugging.** Off under `-debug`/`vm_debug`, exactly as the JIT is today (d_main.cpp:4188-4195).

**Memory (Inference).**
- Native code per bytecode op ≈ 6-15 B; the JIT already holds a comparable image today.
- For an estimated 30-80k script functions × 50-200 ops, that is **≈ 20-100 MB** of code + unwind data.
- The JIT image (`JitBlocks`, jit_runtime.cpp:49-52) is not built when AOT covers a function, so the net change is roughly
  neutral to +50 MB.
- Measure the function count with `-dumpjit` (vmbuilder.cpp:975) or the new dump.

#### 2.5 Realistic speed-up ranges (Inference, with the basis)

**Notation.**
- s = the script share of thinker + `WorldTick` time: time inside JIT code plus direct natives, versus slow natives and engine C++.
- Today's `stat vm` measures almost exactly that. `VMCycles` is clocked on C++→VM entry (vmframe.cpp:732-736) and *stopped*
  around every slow native (351-353), so it holds JIT code + direct natives + nested script. It is printed per display frame
  (895-911).
- s is unmeasured. **Assume 0.35-0.65.** LoB monsters cost ≈8× a stock monster ([M] §3); bars are mostly natives (`SetOrigin`,
  handler scans).

**AOT speed-up k on that share, versus the JIT:**
- **1.4×** if the time is mostly memory-bound field traffic;
- **2.5×** if it is call-heavy small functions, where marshalling, prologues and virtual dispatch dominate and inlining removes them.

**Saving on the ≈112 ms of thinker + `WorldTick` time left after §2.3:**

| Case | s | k | Saving = 112·s·(1−1/k) | ms |
|---|---|---|---|---|
| Low | 0.35 | 1.4 | 112 × 0.35 × 0.286 | **≈11** |
| Mid | 0.5 | 1.8 | 112 × 0.5 × 0.444 | **≈25** |
| High | 0.65 | 2.5 | 112 × 0.65 × 0.6 | **≈44** |

**If [A] Q1 finds the JIT off:** turning it on gives k_jit ≈ 2-4 on s (interpreter dispatch costs ≈5-15 cycles per op, Inference).
That alone is −30 to −60 ms. It is the first thing to check.

**Not AOT-able:** natives' bodies (`P_TryMove`, `P_DamageMobj`, sound, `SetOrigin`'s relink). These are reduced by [A]'s rows,
§3, and phase A.

### 3. Data-oriented actor storage

#### 3.1 The layout and its constraints (proven by code)

**The layout.**
- `AActor` is 3,104 B. The fields one reset touches span 15 distinct 64 B lines ([A] B.4 offset table).
- `WorldStateHash` reads `Pos`, `Vel`, `Angles`, `health`, `sprite`, `frame`, `tics` (d_net.cpp:1249-1250), again spread out.
- The recreate-lights scan reads `flags8` (336), `effects` and `fountaincolor` ([A] B.4).

**What ZScript compatibility requires.**
- Native fields are exported **by name with the C++ offset** (`DEFINE_FIELD` → `myoffsetof`, vm.h:967-987).
- The JIT bakes offsets in as immediates when it compiles, at load (e.g. `x86::ptr(regA[B], konstd[C])`, jit_load.cpp:244-260).
- Script subclasses append their fields after the parent's `Size` (dobjtype.cpp:377, 637).
- Savegames serialize by name ([F] p_mobj.cpp:456 `A("spawnorder", SpawnOrder)` style).

**Inference: reordering native members inside `AActor` is free for ZScript and saves.** Mods never see offsets; they are resolved
at load.

What is **not** free: moving a script-visible field *out* of the object, because every compiled load is `[obj + const]`.

#### 3.2 Design

**F-DOD-1, the hot prefix (first 384 B = 6 lines).**
- The thinker links and class pointer (line 0, already there).
- The **public** fields other actors read: `Pos`, `Vel`, `Angles`, `radius`, `Height`, `health`, `flags..flags9`, `sprite`/`frame`,
  `target`/`master`/`tracer`, `Sector`, `floorz`/`ceilingz`, `player`.
- The **per-tic engine** fields: `Prev*`, `Scale`, `Alpha`, `CameraFOV`, `PrevPortalGroup`, the fork's
  `FollowBodyYaw*`/`FollowHandRot*` (actorinlines.h:52-56), `effects`, `fountaincolor`, `tics`.
- A `denseIndex` (u32) maintained by `Link`/`Remove`/`ChangeStatNum` ([A] P9.9 Step A).
- The ≈1 KB `ShiftSurfacePositions` copy for model actors ([A] F-SIM-5) copies only slots below a high-water mark.

**What the prefix buys:**
- the reset touches ≈4-5 lines instead of 15;
- the hash ≈2;
- the lights scan 1;
- the phase-A snapshot is a straight `memcpy` of the prefix per M actor;
- foreign-load redirection is a base-pointer swap at the *same* offset: `snapBase + denseIndex*384 + off`.

**F-DOD-2, slabs.**
- `CreateNew` (dobjtype.cpp:483-509) takes a block from a per-thread, per-`Size` slab and keeps the `Defaults` memcpy.
- Actors of one class become contiguous in creation order, which is what the dense array and the class-batched phase A walk.
- Frees go back to the owning slab at commit (serial). Addresses are never sim-visible.

**F-DOD-3, SoA mirrors (engine-only).**
- `Pos` is script-readonly (actor.zs:408) and written in C++ only through the actor.h setters (16 `__Pos` uses). A position SoA
  maintained in the setters is safe.
- It feeds the blockmap/radius queries in phase A and the render snapshot.
- `Vel` and `Prev` are script-writable (actor.zs:409, 424), so they **never** get an authoritative mirror. They stay in the prefix.

#### 3.3 What it is worth (Inference)

**Bytes touched by the engine passes per actor per tic:**
- reset + hash + light scan ≈ 15 + 9 + 3 lines ≈ 1.7 KB today ([A] B.4) → ≈7 lines ≈ 0.45 KB;
- at 53,116 actors: **≈ 90 MB → ≈ 24 MB per tic.**

**Its share of the savings:**
- the bulk of [A]'s reset estimate (9.02 → 2.3-3.0 single-threaded);
- the floor under §1.2's parallel walls, which are bandwidth-bound: 24 MB at 13 GB/s ≈ 1.8 ms of pure transfer across all passes.

**For phase A:**
- the snapshot is 384 B × (M actors + read-as-foreign actors) ≈ 384 B × 10-20k ≈ **4-8 MB, ≈0.3-0.6 ms parallel**;
- copying all 53k actors would be 20 MB, ≈1.5-3 ms. That is why the analyzer limits the set.
- Slabs + prefix also cut the per-tick working set of every thinker, which is the E ceiling in §1.6.

### 4. Using the remaining cores and the GPU

#### 4.1 Today (proven, [C] B.5)

- The main thread runs every tic, all ZScript, prediction, `NetUpdate`, sound, GC, the BSP walk in VR, command recording, fence
  waits and OpenXR calls.
- `ctpl renderPool(8)` exists for wall and scene work (hw_bsp.cpp:87, 401, 1447) and is idle in VR ([C] F-XR-11).
- Texture and model loaders use upload queues (vk_renderdevice.cpp:386-430).

#### 4.2 Core plan for the 9900KF (F-JOB-1, Inference)

| HW threads | Role |
|---|---|
| Core 0 | R: the render thread ([C] C.1, ≈6 ms busy of 11.1). Its SMT sibling: XR runtime calls, OpenAL. |
| Core 1 | S: the sim coordinator. Serial phases P0/P1/P3, `NetUpdate`, GC steps, sound update. Joins the pool during P2. |
| Cores 2-7 + siblings (12 HW threads) | The shared pool. |

SteamVR/compositor processes are not pinned; leave them to the OS.

**Pool priority lanes:**
1. R's frame-critical jobs (scene build per region, model bone evaluation, light binning, sprite sorting);
2. S's phase-A and engine-pass jobs;
3. background (snapshot compression, async asset work).

With frames in flight and a sim thread, R needs pool help only in the first ≈3 ms of each frame. Phase A runs whenever S reaches
P2, so the two interleave by priority rather than by fixed split.

#### 4.3 The job system (F-MT-2): deterministic sim tasks on a work-stealing pool

Rules that make sim jobs deterministic regardless of which worker steals what:
1. A sim job is (range of indices → output slots). It never appends to shared containers and never reduces in completion order.
2. Folds (hash, light bitmask, effect queues) run serially or as a fixed-shape tree in index order. No float atomics.
3. No sim decision reads wall time, worker id or queue depth. No time-budgeted splitting in sim jobs: chunk sizes come from
   indices only.
4. Phase A's per-tx state (RNG streams, effect queue, undo log, scratch) is indexed by tx, not by worker.

**Proven by code:** the per-thread VM stack is already `thread_local` (vmexec.cpp:147), and JIT full frames look it up at run time
(jit.cpp:409-415), not at compile time.

**Implementation choice:** a small C++ pool (Chase-Lev deques) replacing ctpl. 3-4 pw including R's migration.

#### 4.4 GPU and async compute

- **Nothing from the sim may run on the GPU.** Peers have different GPUs and drivers. Float results and ordering differ across
  vendors, and readback adds a frame of latency.
- The GPU's role is to take presentation work off R. That plan is [D]'s: frames in flight, multiview, layered post-process,
  clustered lights, multi-draw indirect, which is required and unused today ([D] §2), async uploads.
- Particles are presentation: the fork already has a GPU particle buffer (common/rendering/hwrenderer/data/hw_gpuparticlebuffer.h).
  Moving `P_ThinkParticles` there removes the 0.25-0.7 ms from S entirely, provided no ZScript reads particles (it cannot today).

### 5. Memory footprint on the owner's 16 GB (coordinator request)

| Item | Size (Inference, 53k actors) | Lifetime |
|---|---|---|
| Dense index | 53k × 4-8 B ≈ 0.4 MB | Persistent |
| Hot prefix | 0 extra (a reorder inside `AActor`) | — |
| Phase-A public-prefix snapshot | 384 B × 10-20k ≈ 4-8 MB (cap at all 53k: 20 MB) | Reused per tic |
| Position SoA mirror (F-DOD-3) | 53k × 24 B ≈ 1.3 MB | Persistent |
| Per-tx state during P2 | Undo log (M only, ≈50 stores × 24 B ≈ 1.2 KB) + effect queue (≈2-8 × 32-64 B) + RNG map (≤8 × 16 B) ≈ 1.5 KB avg × 53k ≈ 80 MB worst | Bounded to **≈ 12 MB** by committing in fixed chunks of 8,192 txs (chunk boundaries by index, so deterministic) |
| Per-thread scratch | VM stack blocks (4 KB growing, vmintern.h:395) + visit stamps (≈320 KB at 60k lines + 20k sectors) + intercepts/`spechit` ≈ 1 MB × 16 | ≈16 MB |
| Slabs | ≤ one partial page per size class per thread; pool cap | ≈16-32 MB |
| AOT DLL | ≈20-100 MB code + unwind (the JIT image it replaces is freed) | Net ≈0 to +50 MB |
| Render snapshot ([FR], not this track) | 312 B × 53k × 2 ≈ 33 MB | Persistent |
| **Total added by this track** | **≈ 50-120 MB** (+33 MB if the render snapshot is counted) | |

**Inference:** under 1% of 16 GB. The binding constraint is memory *bandwidth*, not capacity (§1.6).

---

## 90 Hz contribution

### What this area must deliver

With the sim thread ([C] C.1, P8), S must hold 35 tics/s: **≤20 ms median, ≤28.6 ms p99** ([A] "Budget with a sim thread").

This track's job is to bring the 4-player modlist tic from ≈160 ms into that budget, using the cores that R, the XR runtime and the
OS leave free. Today: ≈160 ms on one core. The gap is ≈ −140 ms (−88%, [A]).

### Combined projection: tic wall time on his machine after each step (Inference)

Baseline B0, 4-player, owner set, Sunder MAP15 ([A] "Today, per tic"):

| Component | ms |
|---|---|
| thinkers | 130.8 |
| reset | 9.02 |
| `WorldTick` | 4.07 |
| particles/beams | 0.72 |
| player ×4 | ≈0.3 |
| client logic | ≈5.5 |
| netgame hash | 8-12 (mid 10) |
| **B0** | **≈160** |

| Step | What lands | Derivation | Tic after (low-mid-high benefit) | Proven / bet |
|---|---|---|---|---|
| **S0** | Measurement only (F-MT-0, [A] Q1) | — | 160 | — |
| **S1** | [A]'s exact single-thread rows | Reset 9.02→2.65; light walk −2.5; hash 10→2; thinker-side rows (F4, F1, 9.1b/c, F2, 9.2, 9.4, 9.5/9.6) −4..−19, mid −11 → thinkers ≈120 | **145 / 133 / 121** | Proven mechanisms; savings Inference ([A]) |
| **S2** | F-MT-1 + F-MT-2 + F-MT-3 (engine passes on the pool), F-DOD-1 prefix | reset 2.65→0.6, hash 2→0.4, client 3.0→1.0, particles 0.72→0.25, +0.3 render snapshot write ⇒ −5.6 | **139 / 127 / 115** | Exact; bandwidth model is Inference |
| **S3** | F-VM-1..5 (call path, direct natives, CHA, prologue) | −5 / −10 / −20 on thinkers + `WorldTick` | **134 / 117 / 95** | Exact; call counts unmeasured |
| **S4** | F-VM-6 AOT (stages 1+2) | 112 × s × (1−1/k): −11 / −25 / −44 (§2.5) | **123 / 92 / 51** (scaled to S3's low/mid/high base) | **Design bet** (s, k unmeasured) |
| **S5** | Phase A: F-MT-5 → F-MT-6 + F-MT-7/8/9 (rule D) | Thinkers at S4 ≈ T = 113 / 86 / 46. wall_th = T(1−p) + T·p·o/E + T·p·c + 0.8, with (p,o,E,c) = (0.30,1.30,5.5,0.10) / (0.60,1.25,6.0,0.08) / (0.85,1.15,7.5,0.05). Low: 79.1+8.0+3.4+0.8 = 91.3. Mid: 34.4+10.7+4.1+0.8 = 50.0. High: 6.9+6.0+2.0+0.8 = 15.7 | **≈101 / ≈56 / ≈21** (non-thinker remainder at S4 ≈10 / 6 / 5) | **Design bet** (p unmeasured, o and E modelled) |
| **S6** | Not this track: Phase 10 helper overlays (bars, icons, leaves → engine attachments), [A] P9.3 Tick LOD opt-in for far controllers and helpers | Removes ≈25 ms of B0 helper work (16.46 + 4.79 + 2.18 + 1.57), scaled by S1-S4 (×0.65-0.9). Partly overlaps S5's L-class gains (net −5 to −15 after S5). LOD −5 to −15 | **≈80 / ≈35-40 / ≈16** | Design bet ([A]) |

**Reading.**
- **Structural track alone (S1-S5): 160 → ≈21-101 ms, median case ≈56 ms (2.9×).**
- **With S6: median ≈35-40 ms, best ≈16 ms (10×).**
- The median case does **not** fit the ≤20 ms sim budget. The high case does.

What separates them is three numbers, each settled by one run (§D):
1. s, from `stat vm`;
2. p, from the classifier dry run weighted by `sim_profile`;
3. whether the JIT is on at all ([A] Q1). If it is off, S4's low case moves to the mid case by itself.

**Answer to the owner's premise.** Engine work does far more than 10-20%: S1-S5 remove 37-87% of the tic, 65% in the median case.
An order of magnitude needs the favorable end *and* S6.

### Order to build (and why)

Lane rule first: default-on sim changes belong after ROADMAP step 3 freezes the suite, or behind off-by-default cvars with the suite
run on and off ([A] B.9; [F] ROADMAP.md:23-29). The VM, playsim and `TryRunTics` touch the netplay lane's code
([F] RENDER_LANE_HANDOFF_2026-11-09.md:197-200). **This track needs the owner to open a sim lane**, or to assign it to the netplay
lane after its freeze.

| # | Work package | Gate | Effort (person-weeks) | Proven / bet |
|---|---|---|---|---|
| 1 | F-MT-0 measurement: `vmstats`, `sim_profile`, classifier dry run (needs a first cut of #6), plus [A] Q1/Q3 | Read-only | 2-3 | Proven tools |
| 2 | [A] S1 rows (exact) | Benchdemo bit-identical where exact; suite | Per [A] | Proven |
| 3 | F-MT-1 groundwork (thread-safety refactor, no behavior change) | **Benchdemo bit-identical** + suite | 6-9 | Proven blockers, mechanical fixes |
| 4 | F-MT-2 job system + F-MT-3 engine passes + F-DOD-1 prefix/dense index (+ F-DOD-2 slabs) | Same hash at 1/N threads; suite | 3-4 + 2-3 + 4-6 | Exact |
| 5 | F-VM-1..5 + F-VM-7 (call path, CHA, prologue, thunks) | Benchdemo identical; `stat vm` | 4-6 | Exact |
| 6 | F-MT-9 analyzer + native effect table (top ~300 natives) | Classifier report reviewed | 6-10 | Design bet on coverage |
| 7 | F-MT-8 runtime + F-MT-7 rule-D RNG + F-MT-5 helper batch (L only) | `sim_mt_verify` on the rig for 10k tics; suite with `sim_mt 0/1` | 8-12 | Design bet |
| 8 | F-VM-6 AOT stage 1 → stage 2 | Interpreter = JIT = AOT hash on benchdemos; suite | 10-16 | Design bet on k |
| 9 | F-MT-6 transactional movement (M classes) + F-MT-10 locality hint | Re-execution ≤5%; verify; suite; then the four-player internet run | 6-10 | Design bet on p and conflict rate |
| 10 | F-MT-4 sight prefetch (any time after #3) | `stat sight` A/B; benchdemo identical | 2-3 | Exact |
| | **Total** | | **≈ 59-92 pw** (≈ 1.2-1.9 person-years); #1-#5 alone ≈ 21-31 pw | |

Why this order:
- #1 decides whether #8 and #9 are worth their cost.
- #3-#5 are exact, so they cannot change gameplay, and they are prerequisites of everything parallel.
- #7 ships L classes first: the largest proven class (`RSQTHP_Bar`, 16.5 ms) and the lowest semantic risk. It proves the
  commit/verify machinery before movers.
- #8 multiplies everything before and after it.
- #9 is last because it carries the highest semantic risk (movement conflicts) and the most new C++.

---

## Open questions (each with the run that settles it)

All runs: bridge `boottest`, the owner's 57-file zdl as for rr342, `-Map MAP15`, `-Tics 700`,
`-ExtraArgs "+vid_vsync 0 +vid_maxfps 0 +cl_capfps 0 -rngseed 1113"`, a new exe name per build (`scripts\build_named.py <name>`).

**G1. Is the JIT on, and what is s?**
- On the current exe: `-Exec` with `wait 35; vm_jit; vm_jit_aot; vm_debug` ([A] Q1).
- Then `+stat vm` at `wait 300`, read from the boottest screenshot. Or, on a port build, `vmstats` printing the same three numbers
  to the log.
- s ≈ (VM ms per tic) / (thinkers + `WorldTick` ms per tic, from `ticprofile` in the same run).
- VM calls per tic gives the script-call count for S3.

**G2. How many calls go through `NativeScriptCall` per tic, and to which natives?**
- A port build with a per-native counter in `NativeScriptCall` (vmframe.cpp:347), dumped by `vmstats natives 30` at `wait 300`.
- Sizes F-VM-2.

**G3. p: the classifier dry run.**
- A port build with F-MT-9's first cut: `sim_mt_classify` at `wait 300` prints, per class, L/M/S, the reason, and `sim_profile` ms
  over 300 tics.
- Σms(L), Σms(M), Σms(S) are p's direct inputs.
- Needs [A] Q3 (the unattributed ≈78 ms) resolved in the same run.

**G4. Memory bandwidth and RAM speed.**
- Owner reports DDR4 speed and channel count (Task Manager → Memory).
- Measure bandwidth during a tic with Intel PCM (`pcm-memory`) or VTune while the Q2 run plays. This sets E.

**G5. Function count and AOT size.**
- `-dumpjit` (vmbuilder.cpp:975) on the owner's set: count functions and bytes of JIT code.
- Sizes the AOT DLL and stage-1 compile time.

**G6. Move-conflict rate.**
- On the Linux rig, once #9 exists: `sim_mt 1 +sim_mt_stats 1` on a horde map (`NOMONSTERS="" bash tools/netplay/run.sh mt4 4 700`).
- Prints re-executions per tic.
- The rig cannot hold 4 × BD22 ([F] tools/netplay/README.md), so use a monster-heavy stock map.

**G7. Thread-count independence.**
- On the rig: `sim_mt_verify 1` with `sim_mt_threads 1` and `12` over the benchdemo set.
- Every tic hash must be equal. This gate comes before the four-machine suite.

**G8. The XR runtime's CPU share in the headset** (sets how many pool threads are really free). Not runnable through the bridge:
the owner wears the headset with `perf_track` per-thread busy columns and Windows Resource Monitor for vrserver and vrcompositor.

**G9. Owner decision.** Is the `sim_rule 1` change of game dice acceptable? Old demos play only with `sim_rule 0`; benchdemo goldens
are re-recorded. Is a sim lane opened before or after ROADMAP step 3?
