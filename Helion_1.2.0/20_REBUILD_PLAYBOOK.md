# P1 — Rebuild playbook: UZDoom 5.0 → high-performance PC VR engine (Vulkan, OpenXR, stereo 90–120 Hz)

This playbook turns the verified teardown (H1–H6, U1–U3) into eleven workstreams (Phases 0–10), ordered by dependency. Each one names the UZDoom 5.0 code to change, gives implementation steps and says how success is measured. The aim is to take Helion's speed without dropping UZDoom features. Helion is fast for three reasons:

- It does CPU scene work once per tic, not once per frame.
- It keeps the world resident on the GPU.
- It skips expensive features: no portals, no dynamic lights, no models, no ZScript, no 3D-floor occlusion.

The plan copies the first two. For the third, it puts the cost of each feature onto the GPU or into tic-rate work instead of deleting the feature. Nothing here is "impossible"; every item has a price.

Conventions: `[H]` = Helion @ 5edd437f, `[U]` = UZDoom 5.0.0 @ 292cf420, repo-relative `path:lines`. I re-opened every UZDoom location this playbook tells you to edit. Effort figures and performance reasoning are **Inference**. Facts about external APIs (Vulkan, OpenXR, Tracy) are labelled **API fact**. Code sketches are marked *(sketch)*: they show shape, not final code. `[M] §n` = the owner's measurements, `rs_enginesurvey/UZDXREMA_0.1.0/10_MEASUREMENTS.md`, taken on his 5.0-based fork on a flat screen. They are evidence about cost, not about stock 5.0 code.

## Priorities from measured data

The owner measured his UZDoom-5.0-based fork on his PC: RTX 3080 Ti, i9-9900KF, flat screen, no headset [M] rs_enginesurvey/UZDXREMA_0.1.0/10_MEASUREMENTS.md §1–4. These are **measurements, not code facts**. They come from a fork, but the hot paths they hit (`P_Ticker`, `RunThinkers`, `CallTick`, the interpolation reset, the eye loop) are the stock 5.0 code cited throughout this playbook.

| Case | Frame | Sim / script | Scene build | GPU wait | Thinkers |
|---|---|---|---|---|---|
| Sunder MAP15, no mods | 13.9 ms (66.7 fps) | 2.5 / 3.1 ms | 3.8 ms | 3.2 ms | 8,065 |
| Sunder MAP15, owner's 57-file mod set | 315 ms (3.4 fps) | 250 / 258 ms | 4.4 ms | 25.9 ms | 53,116 |
| Abandon MAP13, owner's mod set | — | thinkers 61–87 ms per tic | — | — | — |

Per tic, Sunder MAP15 with mods [M] §2:
- thinkers: 130.8 ms;
- interpolation reset: 9.0 ms;
- `WorldTick` handlers: 4.1 ms;
- particles: 0.7 ms.

The tic budget is 28.6 ms.

Top classes in one tic [M] §3:
- `RSQTHP_Bar`, one HP bar per monster (the owner's own mod): 16.5 ms over 3,622 calls, 4.5 µs each, which is more than the monster it belongs to;
- mk-champions helpers: ≈ 25 ms;
- Legion of Bones replacement monsters: ≈ 11 ms, at 2–3 µs each against about 0.3 µs for a stock monster;
- `Shifter`: 7,686 calls, 2.2 ms;
- crit helpers: 11,332 calls, 1.6 ms.

Separately, the `r_ticstability` busy-wait costs 2.3 ms per frame [M] §4. It is stock 5.0 code [U] src/d_net.cpp:2102-2122, called at line 2275.

**Consequences for the plan (Inference):**
1. **With real mods the playsim is the problem, by an order of magnitude.**
   - Nothing on the render side matters until a tic fits in 28.6 ms. A slower tic means the game itself runs slow, and no sim thread fixes that.
   - The sim work therefore moves to the front: Phase 0's profiling and fixes, Phase 9, and the new **Phase 10 (helper-actor explosion)**. They also get the most detailed instructions.
   - The two biggest levers are engine overhead per thinker (VM dispatch, the 9 ms reset pass, three full walks) multiplied by 53k thinkers, and the *number* of thinkers. Most of those thinkers are per-monster helper actors that need no thinker at all.
2. **Without mods, a flat-screen frame is already 13.9 ms** (uncapped, 3056×1689 window; about 2.3 ms of a comparable frame is the F12 busy-wait [M] §4), over the 11.1 ms budget at 90 Hz, before stereo doubles the scene work.
   - The per-eye scene rebuild (Phase 2), the zero-frames-in-flight GPU wait (Phase 1), and F12 (the busy-wait) stay critical.
   - The sim thread (Phase 8) is what keeps a 15–25 ms tic from ever missing a headset frame once tics fit their budget.

---

## Phase 0 — Instrument and baseline (plus the cheap verified fixes)

**Goal.** Before anything else changes, make every later claim measurable, and bank the fixes that cost hours rather than weeks. Exit criterion: one reproducible benchmark run (see P3) gives CPU ms split by subsystem, GPU ms per pass, draw counts and p99 frame time, and the fixes below have landed with A/B numbers.

**Helion evidence.** Helion's own instrumentation is weak. It has stopwatch trees and draw counts [H] Core/Util/Profiling/Profiler.cs:17-22, Core/Util/Profiling/Timers/RenderProfiler.cs:5-17, a spike trigger [H] Core/Util/Profiling/Profiler.cs:45-57, and no GPU timing in use: `GLElapsedTimeQuery` has no callers [H] Core/Render/OpenGL/Queries/GLElapsedTimeQuery.cs:7-37. Its BenchmarkSuite holds BenchmarkDotNet micro-benchmarks of AI movement, LOS, hitscan, entity creation and collision only, never frame time ([H] BenchmarkSuite/Program.cs:5-10 and the `Entity*.cs` files beside it; H5 §8). Copy its spike-trigger idea; nothing else there is worth copying.

**UZDoom starting point.** `glcycle_t` clocks (`All`, `Bsp`, `ProcessAll`, `PostProcess`, `drawcalls`, `MTWait`, `WTTotal`, …) [U] src/common/rendering/hwrenderer/data/hw_clock.cpp:33-42 feed `stat rendertimes` (line 114); the playsim has `stat think`/`sight`/`gc` and `profilethinkers` [U] src/playsim/dthinker.cpp:123-198, 1418, [U] src/common/objects/dobjgc.cpp:737. No Tracy (`grep -rniE tracy` finds nothing). GPU timestamps exist per `PushGroup`/`PopGroup` but are read with `VK_QUERY_RESULT_WAIT_BIT` after the end-of-frame full wait [U] src/common/rendering/vulkan/system/vk_commandbuffer.cpp:208-266, which breaks once Phase 1 adds frames in flight.

### 0.1 Tracy integration

**API fact.** Tracy has CPU zones (`ZoneScopedN`, `ZoneTransientN` for dynamic names), `FrameMark`, `TracyPlot` counters, lock instrumentation, and a Vulkan GPU context (`TracyVkContext`, `TracyVkZone`, `TracyVkCollect`).

Steps:
1. Vendor Tracy under `libraries/tracy`. Add a CMake option `UZ_TRACY` (default OFF in release, ON in the `RelWithDebInfo-VR` preset). Wrap every macro in `src/common/utility/uz_profile.h`, so that a build without Tracy compiles to nothing.
2. Make `glcycle_t` emit a Tracy zone when `UZ_TRACY` is on. Add an optional `const char*` name to the clock; `Clock()` opens a zone and `Unclock()` closes it, **only for named clocks**. Name the per-scene clocks (`All`, `Bsp`, `ProcessAll`, `RenderAll`, `PostProcess`, `MTWait`, `WTTotal`, `Finish`, `Flush3D`, `twoD`; hw_clock.cpp:33-42). Leave the per-item clocks unnamed: `drawcalls` runs on every `Apply()` [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:183, and `SetupWall`/`SetupFlat`/`SetupSprite` on every job [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:149, 188, 197, which would break the no-per-draw-zones rule below.

   **Inference:** a clock can be clocked more than once per frame (per eye, per portal), so the zone must be a manual begin/end pair with a per-thread stack, not RAII. **API fact:** Tracy zones must nest LIFO per thread. Assert LIFO order in the debug wrapper; a `Clock A, Clock B, Unclock A` pattern corrupts the capture.
3. Put `FrameMark` right after `D_Display()` in `D_DoomLoop` [U] src/d_main.cpp:1463-1467. Under OpenXR (Phase 2), put it after `xrEndFrame`.

**Mandatory CPU zones** (names reused in P3):
- *Sim.* `Frame.TryRunTics` ([U] src/d_net.cpp:2190-2347) containing `Sim.GCCheck` (opened only inside `Step()`; condition [U] src/common/objects/dobjgc.cpp:155-163), `Sim.Tic` (each `G_Ticker()`, line 2323) and `Sim.ClientSide` (lines 2294, 2341). In `P_Ticker` [U] src/p_tick.cpp:168-276: `Sim.ClearInterp` (241-252), `Sim.Particles` (254), `Sim.PlayerThink` (256-258), `Sim.WorldTickEvents` (261), `Sim.LevelTick` (262), `Sim.RunThinkers` (263 → [U] src/playsim/dthinker.cpp:93-122), `Sim.Specials` (266-269). In `RunClientSideThinkers`: `Sim.RecreateLights` (around the `recreateLights()` call at [U] src/playsim/dthinker.cpp:308; the lambda body is 250-265) and `Sim.DynLightTick` (309-318).
- *Render.* `Render.Display` (`D_Display`, [U] src/d_main.cpp:1109); `Render.Interp`/`Render.InterpRestore` (`D_Render`, 600-620); `Render.ParticleBucket` ([U] src/playsim/p_effect.cpp:204-239); `Render.PolyLink` ([U] src/d_main.cpp:612); `Render.View` ([U] src/rendering/hwrenderer/hw_entrypoint.cpp:311-407) with `Render.CameraTextures` (375-386); `Render.Eye0`/`Eye1` (`RenderViewpoint`, 97-191, transient names).
- *Scene.* In [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp: `Scene.Create` (470-511), `Scene.RenderHacks` (501-504), `Scene.Opaque` (521-579), `Scene.Translucent` (581-608), `Scene.EndDraw` (908), `Scene.Set3DViewport` (965). In [U] src/rendering/hwrenderer/scene/hw_bsp.cpp: `Scene.BSP` (1008-1062), `Scene.WorkerWait` (1036-1038), `Scene.Worker` on the worker thread (111-216) plus a `worker.idle_spins` plot (118-137). In [U] src/rendering/hwrenderer/scene/hw_portal.cpp: `Portal.EndFrame` (73), `Portal.Render` (158; transient name = portal class).
- *Backend.* `Render.PostProcess` ([U] src/common/rendering/vulkan/renderer/vk_postprocess.cpp:76); `Render.2D` (`Draw2D` in [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:201-222); `VK.WaitForCommands` ([U] src/common/rendering/vulkan/system/vk_commandbuffer.cpp:163-199) with `VK.Acquire`/`VK.Submit`/`VK.FPSLimit`/`VK.Present`/`VK.FenceWait` (182-188); `VK.MidFrameFlush` ([U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:186-190); `VK.TextureUpload` ([U] src/common/rendering/vulkan/textures/vk_hwtexture.cpp:138-187); `VK.PipelineCompile` ([U] src/common/rendering/vulkan/renderer/vk_renderpass.cpp:243).

Do **not** zone per-draw `Apply()` or per-actor `CallTick`: at 10k events per frame the overhead distorts the result (**Inference**). Use per-frame `TracyPlot` counters instead: `draws`, `applies`, `material_binds` (inside `ApplyMaterial`'s `mChanged` branch, [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:431-445), `hwbuffer_binds` (448-465), `pipeline_binds`, `stream_bytes`; the existing `rendered_*` counters [U] src/common/rendering/hwrenderer/data/hw_clock.cpp:45; and `actors_ticked`, `vm_script_calls`, `action_calls`, `sight_checks`, `trymoves`, `gc_step_bytes`.

### 0.2 GPU timestamps

Keep the existing `PushGroup`/`PopGroup` timestamp scopes. Change three things:
1. Readback becomes non-blocking: read frame k−N's pool without `WAIT_BIT` after its fence has signalled. This is needed once Phase 1 adds N frames in flight.
2. Add Tracy GPU zones on the same command buffers (**API fact**: `TracyVkZone`).
3. Name these scopes: `GPU.Clear`, `GPU.SkyFirst`, `GPU.Opaque`, `GPU.Masked`, `GPU.Models`, `GPU.Decals`, `GPU.Portals` (one per portal), `GPU.Translucent`, `GPU.SSAO`, `GPU.PP`, `GPU.2D`, `GPU.Present`. Each is tagged per eye until Phase 2, and per view-pass afterwards.
4. Prepare for multiview (Phase 2). **API fact:** a `vkCmdWriteTimestamp` inside a render pass instance with multiview enabled consumes N consecutive queries, N = bits set in the view mask; only the first is guaranteed to hold the timestamp. `PushGroup`/`PopGroup` (vk_commandbuffer.cpp:208-238) allocate one index each, so they must reserve `popcount(viewMask)` indices inside multiview passes. **Open risk:** Tracy's `TracyVkZone` allocates one query per begin/end; put Tracy GPU zones around multiview render passes, not inside them, or patch its query allocator.

### 0.3 Benchmark map set

P3 holds the full set. In short, there are five stress classes:
- actor count (a slaughter map);
- huge geometry;
- portal-heavy;
- dynamic-light-heavy;
- a ZScript-heavy mod.

Each one runs from a recorded demo with `timedemo`-style playback at a fixed `vid_scalefactor`, with a fixed camera-path variant for render-only runs. Every phase's acceptance targets refer to these runs.

### 0.4 Cheap verified fixes (do them now, measure each)

Each fix is small, local and checked against code. Every fix gets a cvar kill-switch until it has passed the demo-compat suite (P3 §5).

**F1. `FState::CallAction` copies two TArrays per action call.**
- Where: [U] src/gamedata/info.cpp:135 `auto argtypes = ActionFunc->Proto->ArgumentTypes;` (in `FState::CheckCallerType`, which `CallAction` calls at line 172) and :179 `auto defs = ActionFunc->DefaultArgs;`. Both are copies by value. `TArray`'s copy constructor allocates when the source is non-empty [U] src/common/utility/tarray.h:247-250, 848-858.
- Change: both become `const auto&`.
- Why it is safe: `defs` aliases `ActionFunc->DefaultArgs`, while the code writes into the global `actionParams` (line 180, `actionParams.Reserve`). These are different arrays, so a reallocation of `actionParams` cannot invalidate `defs`. `argtypes` is only read (lines 137-147).
- Effect: removes 1–2 malloc/free pairs per action call. `A_Chase` has three optional parameters [U] wadsrc/static/zscript/actors/actor.zs:1287, so it pays 2.
- **Inference:** about 500–700 malloc/free pairs per tic on a 1,000-chaser map.

**F2. `UpdateRenderSectorList` never early-outs (`OldRenderPos` is never updated).**
- Where: [U] src/playsim/p_secnodes.cpp:375-427. The guard is `Pos() != OldRenderPos` (line 378). `OldRenderPos` is written once, at spawn, to `FLT_MAX` [U] src/playsim/p_mobj.cpp:5437, and nowhere else.
- What happens today: the function runs on every tic for every sector-linked actor. It is called from the tail of `AActor::Tick` [U] src/playsim/p_mobj.cpp:5105 and again from `LinkToWorld` [U] src/playsim/p_maputl.cpp:533. Each run clears and rebuilds both portal render lists. It linearly scans all `Level->linePortals` when a portal is near (lines 381-400) and walks sector-portal stacks with `PointInSector` (lines 401-426).
- Five traps when fixing it:
  1. `ClearRenderLineList()` at line 381 runs unconditionally, but the list is rebuilt only if `Pos().XY() != OldRenderPos.XY()` (line 382). With a working cache, a Z-only move would empty the line-portal list and never refill it.
  2. `UnlinkFromWorld` clears both lists unconditionally [U] src/playsim/p_maputl.cpp:317-318. An in-place relink would therefore leave empty lists behind a still-valid cache key.
  3. The sector-portal walk depends on `Top()` (line 408) and on portal plane heights (`GetPortalPlaneZ`, lines 406 and 419). Planes move, so a cache keyed on position alone goes stale.
  4. `PortalBlocksMovement` (lines 404 and 416) tests the plane flags `PLANEF_NOPASS | PLANEF_DISABLED | PLANEF_OBSTRUCTED | PLANEF_LINKED` [U] src/g_levellocals.h:934-937, which change without any actor movement. `CheckPortalPlane` sets `PLANEF_OBSTRUCTED` whenever a moving plane passes the portal height [U] src/playsim/p_sectors.cpp:983-992, and portals can be assigned or cleared at runtime.
  5. `OldRenderPos` is not serialized. After a savegame load the key must be invalid, or an actor whose stale key happens to match keeps empty lists.
- Change *(sketch)*:
  ```cpp
  // actor.h: replace DVector3 OldRenderPos with a cache key
  struct RenderListKey { DVector3 pos; double top; sector_t* sec; double ceilPZ, floorPZ; uint32_t ceilF, floorF, portalEpoch; };
  // p_secnodes.cpp UpdateRenderSectorList():  (planeF = planes[p].Flags & (NOPASS|DISABLED|OBSTRUCTED|LINKED))
  RenderListKey k{Pos(), Top(), Sector, Sector->GetPortalPlaneZ(ceiling), Sector->GetPortalPlaneZ(floor),
                  planeF(Sector, ceiling), planeF(Sector, floor), Level->portalEpoch};
  if (k == OldRenderKey || (flags & MF_NOSECTOR)) return;
  if (k.pos.XY() != OldRenderKey.pos.XY() || k.portalEpoch != OldRenderKey.portalEpoch) { ClearRenderLineList(); /* rebuild lines */ }
  /* sector-portal part unchanged */ ;  OldRenderKey = k;
  // p_maputl.cpp UnlinkFromWorld(), after lines 317-318:
  OldRenderKey.pos = DVector3(FLT_MAX, FLT_MAX, FLT_MAX);
  ```
  `RenderListKey` needs an `operator==`. `Level->portalEpoch` is a new level-wide counter; bump it on every portal-state change the key does not already hold: line-portal creation and `UpdatePortal` [U] src/playsim/portal.cpp:240, polyobject moves of portal lines, and sector-portal assignment. The plane flags of trap 4 are in the key. Deeper stack hops read plane heights of other sectors. If you do not want to track those, keep the key conservative: skip the early-out when `Sector` has any non-blocking portal plane **or** `touching_sectorportallist` is non-empty. The second condition empties a list left over from a portal that has since become blocking. On stacked-portal maps this costs the same as today; everywhere else it saves the work. Set `OldRenderKey.pos` to `FLT_MAX` in the actor's post-load fixup (trap 5).
- Verification: a debug cvar `r_verifyrenderlists` rebuilds the lists into scratch on each skipped call and asserts equality (as sets: see the ordering note under compatibility below).
- Effect (**Inference**): large on portal maps, small but non-zero on portal-free maps (two `P_DelSeclist(nullptr)` calls and two `PortalBlocksMovement` tests per actor per tic).

**F3. Vulkan always rebinds the material.**
- Where: `SetMaterial` sets `mChanged = true` with no compare [U] src/common/rendering/hwrenderer/data/hw_renderstate.h:629-639. `ApplyMaterial` then binds set 0 and set 2 [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:431-445. GL already filters redundant binds [U] src/common/rendering/gl/gl_renderstate.cpp:322-326.
- Change: `VkRenderState` keeps `mLastMaterial {FMaterial*, clamp, translation, overrideshader, NumTextureLayers}` and skips the bind when the key is equal. Three conditions keep it correct:
  1. Reset `mLastMaterial` in `EndRenderPass()` [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:490-504, next to the existing `mLast*` resets, because descriptor bindings do not survive across command buffers. Every mid-frame `FlushCommands` goes through `EndRenderPass` [U] src/common/rendering/vulkan/system/vk_commandbuffer.cpp:133-136. `BeginRenderPass` forces `mMaterial.mChanged = true` (vk_renderstate.cpp:577), so a key compare without this reset would skip the first bind in the new command buffer.
  2. Reset it when the pipeline layout changes, i.e. when `NumTextureLayers` differs. Sets bound under a different layout are disturbed.
  3. Still call `NeedUpdate()` for canvas textures (line 438) even when the bind is skipped.
- Also stop re-binding set 0 in both `ApplyMaterial` and `ApplyHWBufferSet` (lines 442 and 458) unless a new command buffer began or the layout changed. **API fact:** binding under a layout whose set 0 and push-constant ranges are identical keeps set 0 valid, so only a set-0 layout change forces the rebind.
- Effect: cuts 2–4 `vkCmdBindDescriptorSets` per wall draw to roughly 0 in texture-coherent runs. Turning on `gl_sort_textures` (default false [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:45) makes runs coherent.

**F4. Signature validation on every `VMCallVoid`, and `CallTick`'s VM detour.**
- Where: `VMValidateSignatureSingle` runs on every templated call [U] src/common/scripting/vm/vm.h:687-690. `DThinker::CallTick` always enters `VMCallVoid` for script-registered classes [U] src/playsim/dthinker.cpp:1265-1271, even when the virtual is the stock native, whose boxed wrapper just calls `self->Tick()` [U] src/playsim/dthinker.cpp:1259-1263.
- Change:
  - Wrap the validation in `#ifndef NDEBUG`, or validate once per `VMFunction*` with a per-function "validated" bit.
  - In `CallTick`, cache the stock `DThinker.Tick` `VMFunction*` once (the virtual index is already cached by `IFVIRTUAL` [U] src/common/scripting/vm/vm.h:1066-1076). If `func == stockTick`, call `Tick()` directly.
- ZScript compatibility: classes that override `Tick` have a different `func` and keep VM dispatch unchanged.
- Effect (**Inference**): about 5–7 indirect calls and about 10 dependent loads saved per plain actor per tic, roughly 0.2–0.4 ms per tic at 10k actors.

**F5. O(n²) intercept selection in sight checks.**
- Where: `P_SightTraverseIntercepts` rescans every intercept for each step [U] src/playsim/p_sight.cpp:564-583.
- Change: `std::stable_sort` the intercepts by `frac` once, then walk them in order. The original loop takes the *first* minimum (strict `<`), so a stable sort reproduces the order exactly when fracs tie.
- Effect: the sight path's cost becomes linear in the number of crossings. Long, open sightlines on slaughter maps benefit most.

**F6. The scene worker busy-spins forever.**
- Where: [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:118-137. The empty-queue branch spins. On non-x86 the `_mm_pause` block compiles away and leaves a tight loop. In addition, every scene does a `renderPool.push` that heap-allocates (hw_bsp.cpp:1026-1028; [U] src/common/thirdparty/ctpl.h:189-198).
- Change: a persistent worker woken by a `std::binary_semaphore`. On an empty queue it spins for at most ~2 µs, then waits on an atomic (`std::atomic::wait`). `AddJob` calls `notify_one` only when the worker has published a "sleeping" flag; a notify per job is a futex syscall per wall.
- Effect: in VR the compositor and the sim thread need that core.

**F7. `AllocVertices` reads the buffer pointer before the `fetch_add`.**
- Where: [U] src/common/rendering/hwrenderer/data/flatvertices.cpp:138-148. `GetBuffer()` returns `&base[mCurIndex]` from a separate load before the `fetch_add`, so with two allocating threads the pointer and the returned index can disagree. It is correct with one allocating thread and racy with two.
- Change: `auto index = mCurIndex.fetch_add(count); return { GetBuffer(index), index };` (the overflow check stays), and keep `mVertexBuffer` immutable while a scene is being built. This is a prerequisite for the multi-worker scene build in Phase 3.

**F8. Per-DrawInfo memsets and TMap reallocation.**
- Where: `ClearBuffers` memsets three level-sized flag arrays and `TMap::Clear()` frees and reallocates four maps [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:217-254. It runs twice per DrawInfo (once in `StartScene`, once in `Release`). Note one detail: `no_renderflags` is resized to the subsector count (line 242) but memset by `Level->nodes.Size()` (line 246), because `RenderBSPNode` indexes it by *node* index [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:970. That is safe only because a BSP has one node fewer than it has subsectors; add an assert.
- Change: replace the flag arrays with epoch-stamped arrays (`uint32_t stamp[]`, a set is `stamp[i] = epoch`, a test is `stamp[i] == epoch`, a clear is `++epoch`), and add a `TMap::Reset()` that keeps the node storage.
- Effect: removes O(subsectors) work per eye and per portal.

**F9. Particle bucketing every frame.**
- Where: `P_FindParticleSubsectors` clears `sprites` for every subsector and re-buckets all particles and VisualThinkers on every rendered frame [U] src/playsim/p_effect.cpp:204-239, called from `D_Render` [U] src/d_main.cpp:611.
- Change: call it once per tic, at the end of `P_Ticker` and of the client-side logic, and keep a dirty flag. Also set the flag on level start, savegame load and an `r_particles` change (the function's output depends on it, line 227), so the first frame after any of these is bucketed. Positions change only in tics. Any renderer-side interpolation still reads prev/cur, so buckets taken at tic time are the same buckets the per-frame call would produce.
- Effect (**Inference**): 3.4× fewer calls at 120 Hz.

**F10. Behind-view culling disappears above 46° of pitch.**
- Where: `FrustumAngle()` returns `0xffffffff` when `|pitch| > 46°` [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:377-397. `CreateScene` then adds a near-empty range [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:473-474. In an HMD this is the common case.
- Change: compute the exact horizontal yaw extent of the real frustum.
  1. Take the 4 corner rays of the frustum (asymmetric fov tangents, current pitch and roll).
  2. If the frustum contains straight up or straight down, return "full circle". Test it as "±Z lies inside all four side planes". That is exact with head roll; "pitch ± vfov/2 crosses ±90°" is only correct at zero roll.
  3. Otherwise return the min/max yaw of the 4 rays relative to the view yaw. **Inference (geometry):** that is exact. Azimuth is monotonic along each great-circle edge between corners, and a convex cone that excludes both poles spans less than 180° of azimuth, so there is no wrap ambiguity.
  4. Pad by the clipper's angular tolerance.
- The clipper's angles are measured from the clipper's own origin, so each clipper takes the frustum of *its own* eye: its pose and its `XrFovf`. With Phase 2's dual clipper, that means one `FrustumAngle` per eye clipper. Do **not** feed a clipper the combined frustum of Phase 2 step 4. Its apex sits behind the eye midpoint, so near geometry seen from an eye falls outside its yaw range. This restores back-culling for most head poses up to about 90° − vfov/2 of pitch.
- **Inference:** for a 105° vertical FOV that is about 37° of pitch, which is still too low. That is why Phase 5's 3D frustum and occlusion work matters.

**F11. Pipeline-compile hitches on first use.**
- Where: pipelines are created lazily in `GetPipeline` [U] src/common/rendering/vulkan/renderer/vk_renderpass.cpp:235-241. The disk cache is `<cache>/vulkanpipelinecache` (lines 40-81). `gl_precache` defaults to false [U] src/common/rendering/hwrenderer/data/hw_cvars.cpp:138.
- Change: log every `VkPipelineKey` that gets created to a key file next to the pipeline cache. At startup, create all logged keys on a worker thread. Force material precache at map start in VR builds.
- Phase 1 finishes this job with key-space reduction.

**F12. `r_ticstability` busy-wait.**
- Where: `TicStabilityWait` spins on `steady_clock` for the duration of the previous tic whenever no tic is due [U] src/d_net.cpp:2102-2122, called at line 2275. It is on by default.
- Measured cost on the owner's PC: 2.3 ms per frame, +23% FPS when it is off [M] §4.
- What it is for: it pads every tic-less frame by the last tic's run time (`TicStabilityBegin/End`, lines 2124-2135), so that the frame-start times `I_GetTimeFrac` samples are evenly spaced. The padding *is* the cost. Replacing the spin with a sleep frees the core but returns no frame time.
- Change:
  - Under OpenXR, force it off; `xrWaitFrame` paces the frame and `uTicFrac` comes from the predicted display time (Phase 3 D).
  - On desktop, default it off and check judder on the ZScript-heavy P3 scenario; uneven tic cost is the case the cvar exists for (comment at lines 2096-2100).
  - If smoothness regresses there, compute `TicFrac` from a smoothed frame timestamp instead of padding the frame.
  - Where the padding is kept, use sleep-then-short-spin. That saves power and frees the core for other threads, nothing more.

**F13. Verify the JIT at startup.**
- Why: the JIT is the difference between JIT'd and interpreted ZScript, and on the owner's build its status is unknown [M] §5.
- What decides it:
  - the build: `HAVE_VM_JIT`, x86_64 only [U] CMakeLists.txt:274-276;
  - `vm_jit` and `vm_jit_aot` (not saved; command line only) [U] src/common/scripting/vm/vmframe.cpp:39-59;
  - `-debug` or `vm_debug`, either of which forces both off [U] src/d_main.cpp:3411-3418;
  - per function: `CanJit` silently falls back to the interpreter for `blockJit` functions or ≥ 200 registers [U] src/common/scripting/vm/vmframe.cpp:297-311.
- Change: print one startup line, `VM: JIT built=%d vm_jit=%d aot=%d compiled=%u interpreted=%u (blockJit=%u, regs=%u)`, counting in `FirstScriptCall` and the AOT loop. Add a `vmstats` ccmd that lists the interpreted functions by call count. Ship release builds with `vm_jit` archived.

**Data structures.** Only F2's `RenderListKey` and F8's epoch arrays are new.

**ZScript/mod compatibility.**
- F1, F3, F5, F6, F7 and F8 are invisible to mods.
- F4 changes nothing observable: the stock native `Tick` ran the same code.
- F2 must reproduce identical portal render lists; `r_verifyrenderlists` proves it. The membership is identical, but the *order* of each sector's `sectorportal_thinglist` changes: today every actor unlinks and re-prepends its nodes every tic, with the cache it stops. That list is readable from ZScript [U] src/scripting/vmthunks.cpp:3038, so put a portal map in the demo-compat suite.
- F9: VisualThinker positions are only changed in ticks, and it is safe as long as nothing renders between a tic's start and the rebucket. Rebucketing at the end of the tic guarantees that.

**Risks.**
- F2 is the only fix with a correctness risk.
- F3's command-buffer reset rule is easy to miss and shows up as wrong textures after a mid-frame flush. Test with `vk_submit_size 50`.

**Success metric.** Tracy captures for all 5 P3 scenarios exist. Per-fix A/B: F1 + F2 + F4 + F5 together cut `Sim.RunThinkers` by ≥15% on the slaughter scenario, and F3 cuts `material_binds` by ≥80% with `gl_sort_textures 1` (**Inference** targets).

### 0.5 Per-class and per-virtual sim profiling (the tooling the measured workload needs)

The engine already times every thinker by class, but only for a single tic: `profilethinkers N` → `FThinkerList::ProfileThinkers` clocks each `CallTick` into a `TMap<FName, ProfileInfo>` keyed by `GetClass()->TypeName` [U] src/playsim/dthinker.cpp:857-900, then prints and resets [U] src/playsim/dthinker.cpp:123-198. Extend it into continuous tooling:

1. **`sim_profile 1` (continuous).** Keep the per-class map across tics, and emit a `TracyPlot("cls/<Class>")` per class for the top 32 classes each second. Every 35 tics, dump the top 20 by total, by calls and by average µs. Cost: one `cycle_t` pair per thinker.
   - **Inference:** about 40 ns per thinker, so leave it off by default.
   - Accumulate in unsigned 64-bit ticks and convert once at print time. Add a self-test that totals are non-negative and that per-class sums match `Sim.RunThinkers`: the owner's fork's `simprofile` printed negative totals from broken `cycle_t` accumulation [M] (header).
2. **Per-virtual zones.** With `vm_profile 1`, wrap `VMCall` [U] src/common/scripting/vm/vmframe.cpp:705-737 and the event loopers (`DEFINE_EVENT_LOOPER(WorldTick, …)` [U] src/events.cpp:1068; `DStaticEventHandler::WorldTick` [U] src/events.cpp:2068-2076) in `ZoneTransientN` named after `VMFunction::PrintableName`, for example `SkeletonImp.Tick` or `champHandler.WorldTick`. JIT-to-JIT direct calls bypass `VMCall`, so add the zone in the JIT prologue as well: one call to a profiling stub when the cvar is on, emitted at compile time. That attributes the 4.1 ms of `WorldTick` handlers per handler class.
3. **Per-class allocation counter.** Hook `PClass::CreateNew` [U] src/common/objects/dobjtype.cpp:448-474 and `TArray` growth inside VM frames, and attribute them to the class of the thinker being ticked (a thread-local "current thinker"). Report allocations per class per tic. This finds script-side per-tic `new`, `ThinkerIterator.Create` and string building.
4. **Spike trigger**, copied from Helion [H] Core/Util/Profiling/Profiler.cs:45-57: when `Sim.Tic` exceeds `sim_spike_ms` (default 20), log the per-class top 10 for that tic.
5. **Action-call counters.** Count calls per `FState::CallAction` target [U] src/gamedata/info.cpp:148-234 and per `P_CheckSight` / `P_TryMove` caller class.

**Effort (Inference).** Tracy + GPU zones 1.5, benchmark harness (P3) 2, F1–F13 incl. the F2 verification mode 3, sim profiling tooling (0.5) 2: **≈ 8.5 person-weeks.**

---
## Phase 1 — Vulkan backend: frames in flight, state dedup, SSBO draw data, pipelines, bindless, async upload

**Goal.** Turn the GL-state-machine emulation into an explicit backend. Phase 1 is done when four things hold:
- the CPU records frame N+1 while the GPU runs frame N;
- a draw costs one or two `vkCmd*` calls;
- textures are indices;
- nothing compiles during gameplay.

Everything above `FRenderState` keeps working, so this is the safest large win. It is also the base for Phases 2, 4, 6 and 7. **Measured:** 3.2 ms of the no-mod 13.9 ms frame is GPU wait [M] §1.

**Helion evidence.** There is nothing to borrow for the backend. Helion runs GL 3.3 without fences [H] Core/Render/OpenGL/Buffer/ArrayBufferObject.cs:24-36, and its persistently mapped TBO writes are unsynchronized [H] Core/Render/OpenGL/Textures/GLBufferTexture.cs:34-37, 49-54, which is an anti-pattern (H1 §6.2). The lesson worth taking is that per-draw state should be an *index into small tables*:
- sector light, colour and fog are indexed from the vertex [H] Core/Render/Renderer.World.cs:63-72;
- the texture-array layer is baked into the vertex [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/GeometryRenderer.cs:910, 2235.

**UZDoom code to change (verified).**
- *Frame end.* `VulkanRenderDevice::Update` → `WaitForCommands(true)` [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:201-222. It acquires, submits, FPS-limits, presents, then waits on every fence [U] src/common/rendering/vulkan/system/vk_commandbuffer.cpp:163-199.
- *Per-frame buffers have a single copy* [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:186-190. `FFlatVertexBuffer` already supports N pipeline copies [U] src/common/rendering/hwrenderer/data/flatvertices.cpp:37-38, 76-89, [U] src/common/rendering/hwrenderer/data/flatvertices.h:144-149, and GL rotates them every frame [U] src/common/rendering/gl/gl_framebuffer.cpp:267. `UpdatePlaneVertices` writes into the mapped buffer [U] src/rendering/hwrenderer/hw_vertexbuilder.cpp:406-421, but it tracks the last height per pipeline copy (`vboheight[GetPipelinePos()]`, lines 441-453).
- *Apply* [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:181-207:
  - a 384 B `StreamData` copy on every draw (lines 337-353);
  - push constants on every draw (lines 355-399);
  - `ApplyMaterial` (lines 431-445) and `ApplyHWBufferSet` (lines 448-465);
  - 170 records per stream block, so a rebind every 170 draws [U] src/common/rendering/vulkan/shaders/vk_shader.h:49-54;
  - an overflow drains the GPU mid-frame [U] src/common/rendering/vulkan/system/vk_buffer.cpp:125-134, [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:348-352.
- *Pipelines.* A lazy map per render-pass setup [U] src/common/rendering/vulkan/renderer/vk_renderpass.cpp:235-338, topology in the key (lines 300-311), and only viewport, scissor, depth bias and stencil reference as dynamic state (lines 291-294).
- *Descriptors.* One set per (material, clamp, translation), each with at least 11 samplers [U] src/common/rendering/vulkan/textures/vk_hwtexture.cpp:329-386. `OptionalDescriptorIndexing` exists but is never called [U] libraries/ZVulkan/src/vulkanbuilders.cpp:1792-1796, [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:112-121.
- *Uploads.* Texture uploads are synchronous on first use [U] src/common/rendering/vulkan/textures/vk_hwtexture.cpp:138-187. Post-process descriptor sets are allocated per draw [U] src/common/rendering/vulkan/renderer/vk_descriptorset.cpp:197-231.

**Implementation steps.**
1. **Frame contexts.** Use N = 2 for VR; N = 3 adds latency without adding throughput (**Inference**). *(sketch)*
   ```cpp
   struct VkFrameContext { VkFence fence; VkSemaphore acquire, done; std::unique_ptr<VulkanCommandPool> pool;
     std::vector<std::unique_ptr<VulkanCommandBuffer>> freeCmdBufs; DeleteList deletes;
     uint32_t streamBase, matrixBase, lightBase, boneBase, viewpointBase; VkQueryPool timestamps; };
   ```
   `BeginFrame` [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:451-461 waits on `ctx[k].fence`, resets the pool, runs the deletes, reads the timestamps and advances the rings. `Update` submits with that fence and presents, or calls `xrEndFrame` in VR. Mid-frame flushes [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:186-190 never wait.
2. **Ring every per-frame buffer.** Pass `pipelineNbr = N` at vk_renderdevice.cpp:186-190 and call `NextPipelineBuffer()` in `BeginFrame`, as GL does.
   - `FLightBuffer`, `BoneBuffer` and `HWViewpointBuffer` already take a `pipelineNbr` too [U] src/common/rendering/hwrenderer/data/hw_lightbuffer.h:48, hw_bonebuffer.h:44, hw_viewpointbuffer.h:45.
   - `HW_MAX_PIPELINE_BUFFERS` is 2 on desktop [U] src/common/rendering/hwrenderer/data/buffers.h:30-36, which caps `sector_t::vboheight` [U] src/gamedata/r_defs.h:716. N = 2 fits; N = 3 means raising that constant.
   - Each slot's buffers need their own HW-buffer descriptor set (set 1), selected per frame.
   - Give the stream and matrix buffers N slices. Delete `WaitForStreamBuffers` (vk_renderstate.cpp:467-473) and size the rings from Phase 0 captures plus 50%. On overflow *within* a frame, chain a fresh block (new buffer and descriptor set) instead of draining the GPU, then grow the ring for the next frame.
   - Put the per-frame 2D buffers on a persistent ring (U2 §3.1).
3. **Dedup.**
   - `memcmp` the 384 B `StreamData` against the last record and reuse its index when it matches. The setters write `mStreamData` directly [U] src/common/rendering/hwrenderer/data/hw_renderstate.h:341-345, 462-466.
   - Skip push constants that have not changed.
   - Hash `VkPipelineKey` [U] src/common/rendering/vulkan/renderer/vk_renderpass.h:38-61 from dirty bits.
4. **Per-draw data in SSBOs.** Replace the `StreamUBO` [U] src/common/rendering/vulkan/shaders/vk_shader.cpp:251, 329-352 with one `std430` SSBO per frame slice. `uDataIndex` then indexes the whole frame and the rebind every 170 draws goes away. Matrices go into a second SSBO, indexed from `StreamData`. Later indirect draws use `gl_InstanceIndex`/`gl_DrawID`. **API fact:** `gl_DrawID` needs the `shaderDrawParameters` feature (Vulkan 1.1); `vkCmdDrawIndexedIndirectCount` needs `drawIndirectCount` (Vulkan 1.2); passing a per-draw index through a non-zero `firstInstance` in indirect commands needs `drawIndirectFirstInstance`. Require all three in VR builds. `multiDrawIndirect` is already required [U] libraries/ZVulkan/src/vulkanbuilders.cpp:1834-1837.
5. **Shrink the pipeline key, then precompile it.**
   - Convert fans to lists. The fan→list index buffer already exists [U] src/common/rendering/vulkan/system/vk_buffer.cpp:96-108.
   - Make cull, depth and stencil state dynamic. **API fact:** `VK_EXT_extended_dynamic_state`/`_2`, core in 1.3.
   - Drop `NumTextureLayers` from the key (it is forced to ≥ 11 [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:233-234) once step 6 lands.
   - Enumerate the remaining keys: shader variant × NAT × blend × mask × vertex format × pass. The shader table is [U] src/common/rendering/hwrenderer/data/hw_shaderpatcher.cpp:281-308; expect a few hundred pipelines (**Inference**). Compile them on workers behind the loading screen. GLDEFS user shaders compile at parse time.
6. **Bindless.**
   - Call `OptionalDescriptorIndexing()` and make it required for VR builds.
   - Use one global update-after-bind `sampler2D textures[]` plus a sampler array. **API fact:** that needs `runtimeDescriptorArray`, `descriptorBindingPartiallyBound`, `descriptorBindingSampledImageUpdateAfterBind` and `descriptorBindingVariableDescriptorCount` (the builder already enables them when present, [U] libraries/ZVulkan/src/vulkanbuilders.cpp:1864-1868); `UPDATE_AFTER_BIND` flags on the pool and the layout; and an array size within `maxDescriptorSetUpdateAfterBindSampledImages`. An index that is not dynamically uniform within a draw (from a vertex attribute or a cluster id, Phase 4) must be wrapped in `nonuniformEXT` and needs `shaderSampledImageArrayNonUniformIndexing`. An index taken from `StreamData` per draw is uniform and needs neither.
   - Give each `VkHardwareTexture` a slot index, and add a `MaterialGPU { uint layer[11]; uint sampler; uint flags; … }` table that `StreamData` indexes.
   - `VkMaterial::GetDescriptorSet` disappears.
   - **User-shader compatibility:** the preamble [U] src/common/rendering/vulkan/shaders/vk_shader.cpp:182-352 defines `tex`, `brighttexture` and the user sampler names as macros over the bindless array, so user `.fp` files compile unchanged.
7. **Async uploads.** Decode, hq-resize and mip generation run on workers into a staging ring. A transfer queue copies (**API fact:** queue-ownership barriers are required). A fence publishes the slot; until then a 1×1 placeholder stands in. Pre-warm map textures and the sprite frames of spawnable classes through `PrecacheMaterial` [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:278-293.
8. **Smaller items.** Cache post-process descriptor sets. Optionally switch to dynamic rendering (**API fact:** `VK_KHR_dynamic_rendering`, core in 1.3) instead of the 8 render-pass variants [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:533-579; multiview then becomes a `viewMask` field.

**Risks.**
- Frames in flight expose every CPU write into GPU-read memory. Audit `UpdatePlaneVertices`, `FLightBuffer`, `BoneBuffer`, `mViewpoints`, `mSkyData`, canvas textures and shadow-map buffers.
- Synchronization validation must be clean.

**Success metric.**
- Frame time ≈ max(CPU, GPU), with `VK.FenceWait` ≈ 0 when GPU-bound.
- `material_binds` plus `hwbuffer_binds` < 5% of draws.
- No compile or upload zones on the main thread after warm-up.
- No-mod desktop FPS +30–60% on draw-heavy scenes (**Inference**).

**Effort (Inference).** Frame contexts and rings 3, dedup and SSBO 2, pipelines 3, bindless with the compat layer 4, async upload 3, PP and dynamic rendering 2: **≈ 17 person-weeks.**

---

## Phase 2 — OpenXR and multiview: one scene build, both eyes

**Goal.** Get a correct headset image from **one** CPU scene build per frame, submitted **once** with `VK_KHR_multiview`. This is the first playable VR milestone. P2 has the API detail.

**Helion evidence.** Two things transfer:
- a single shared billboard axis, `viewRightNormal` [H] Core/Render/OpenGL/Renderers/Legacy/World/Entities/EntityRenderer.cs:368-369, which keeps a billboard one planar surface in both eyes (H3 transfer note 4);
- the **dual-clipper rule**: a box is culled only when both clippers agree [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.Bsp.cs:73-88. This is the template for stereo occlusion.

**UZDoom code to change (verified).**
- *The eye loop* in `RenderViewpoint` [U] src/rendering/hwrenderer/hw_entrypoint.cpp:126-188. Per eye it creates a new `HWDrawInfo` (line 142), calls `GetProjection` (159), applies `GetViewShift` (163), runs `SetupView` and `ProcessScene` (164-166) and its own post-process (168-181), then `NextEye` (186-187). `FirstEye`/`NextEye` are empty on Vulkan [U] src/common/rendering/v_video.h:258-259, [U] src/common/rendering/vulkan/renderer/vk_postprocess.cpp:358-360.
- *The 3D-TV eye model* [U] src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:136-202.
- *View setup.* `SetViewMatrix`/`SetupView` [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:405-433, and `HWViewpointUniforms` [U] src/common/rendering/hwrenderer/data/hw_viewpointuniforms.h:32-56, which mirrors `ViewpointUBO` [U] src/common/rendering/vulkan/shaders/vk_shader.cpp:191-210.
- *Shader readers.* View matrices are read in [U] wadsrc/static/shaders/glsl/main.vp:72, 114, 119, 128. `uCameraPos` is read in [U] wadsrc/static/shaders/glsl/main.fp:277, 348, 813, 878, [U] wadsrc/static/shaders/glsl/fogboundary.fp:45, [U] wadsrc/static/shaders/glsl/material_specular.fp:56 and [U] wadsrc/static/shaders/glsl/material_pbr.fp:93.
- *`FrustumAngle`* [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:377-397.
- *Render targets* are single-layer [U] src/common/rendering/vulkan/textures/vk_renderbuffers.cpp:64-200.
- *Vulkan creation.* [U] src/common/platform/posix/sdl/sdlglvideo.cpp:389-401, [U] src/common/platform/win32/win32vulkanvideo.h:57-81, [U] libraries/ZVulkan/src/vulkaninstance.cpp:113, [U] libraries/ZVulkan/src/vulkandevice.cpp:127.

**Implementation steps.**
1. **Layered stereo fallback first.** Give the scene, fog, normal, depth-stencil and pipeline images 2 layers, and implement `FirstEye`/`NextEye` on Vulkan. That gives correct stereo at 2× cost, validates OpenXR (P2) before any scene refactor, and stays as `vr_singlepass 0`.
2. **OpenXR integration** (P2 §1–3): `xrCreateVulkanInstanceKHR` and `xrCreateVulkanDeviceKHR`, swapchains, and the frame loop.
3. **`VROpenXRMode`**, alongside the 8 existing `VRMode` tables ([U] src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:49-56). Per frame it supplies two 6-DoF eye poses with their `XrFovf`, a combined frustum, and the **cyclops pose** (the eye midpoint, with head orientation).
4. **Combined frustum** (**Inference**, derivation). Put the eyes at ±h, with the maximum tangents across both eyes. Pull the camera back from the eye midpoint by `b = h / min(tL, tR)`. With those tangents it contains both eye frusta, and its near plane is `zNear + b`. This frustum feeds the per-frame item cull (Phase 3 C) and the GPU cull (Phase 5). It does **not** feed the angular clippers: each eye clipper uses its own eye frustum (F10).
   - The derivation assumes both views share the head's orientation. Some HMDs report canted views, with a per-eye rotation in `XrView.pose`. **Open risk:** for those, build the combined frustum as the hull of the 8 eye-corner rays expressed in the cyclops frame; do not take the max of tangents. Test on a canted-display runtime before relying on it.
5. **Dual clipper.** The angular clipper is positional, so occlusion seen from the cyclops position is not conservative for an eye 3 cm away. Keep one clipper per eye position:
   - the node box test in `RenderBSPNode` (`mClipper->CheckBox`, [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:968; `Clipper::CheckBox` [U] src/rendering/hwrenderer/scene/hw_clipper.cpp:515) and `AddLine` (hw_bsp.cpp:257-403) test both clippers and insert into both;
   - the static clippers [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:123-126 gain a second main clipper.

   **Inference:** the clip work goes up 10–20%, but the scene is built once, for a net saving of about 40–45% of scene CPU.
6. **One `HWDrawInfo`, one `ProcessScene`.** *(sketch)*
   ```cpp
   auto di = HWDrawInfo::StartDrawInfo(mainvp.ViewLevel, nullptr, mainvp, nullptr);
   di->SetStereoViews(xrViews);   // VPUniforms: mViewMatrix[2], mProjectionMatrix[2], mNormalViewMatrix[2], mCameraPos[2], mCyclopsPos
   di->Set3DViewport(RenderState);  di->SetupView(RenderState, cyc.X, cyc.Y, cyc.Z, false, false);
   di->ProcessScene(toscreen);      // recorded once, viewMask = 0b11
   screen->PostProcessScene(...);   di->EndDrawInfo();
   ```
   Order matters. `SetupView` rebuilds `mViewMatrix` from `HWAngles`, recomputes the normal matrix and uploads the uniforms in one call [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:425-433. Change it to write the cyclops matrix to its own field, compute `CalcDependencies` per view, and upload once, after the per-view matrices from `SetStereoViews` are in place; otherwise it overwrites them. Skies call `SetupView(state, 0, 0, 0, …)` [U] src/rendering/hwrenderer/scene/hw_skyportal.cpp:54. Add a `noEyeTranslation` flag so both eyes use rotation-only views there.
7. **Per-view CPU decisions use the cyclops position**, so both eyes agree. The affected code (full table in P2 §5):
   - rotation frame [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:969-1037;
   - billboard corners (1120-1175);
   - anamorphic clip (1190-1238);
   - translucent `ViewDistance` [U] src/rendering/hwrenderer/scene/hw_walls.cpp:592-595;
   - slope facing [U] src/rendering/hwrenderer/scene/hw_flats.cpp:519, 577;
   - missing-texture tests [U] src/rendering/hwrenderer/scene/hw_renderhacks.cpp:543.

   A Z test against the cyclops is not right for both eyes when a plane lies between them. Head roll puts the eyes up to IPD/2 apart in Z, so "the eyes share Z" does not hold. When the cyclops is within IPD/2 of the plane, draw both slope faces and emit both render-hack variants (the Phase 3 mechanism).
8. **Multiview.** Set `viewMask = 0b11` and enable the `multiview` feature next to [U] libraries/ZVulkan/src/vulkanbuilders.cpp:1853-1868. **API fact:** `multiview` is a Vulkan 1.1 core feature. Each render pass needs a `VkRenderPassMultiviewCreateInfo` (or `viewMask` under dynamic rendering). Inside such a pass, clears and draws hit every view in the mask, and timestamps consume one query per view (0.2 step 4). In shaders, add `GL_EXT_multiview` and index the view arrays with `gl_ViewIndex`. Doom light diminishing ([U] wadsrc/static/shaders/glsl/main.fp:342-353) uses the cyclops distance, so the light bands match between eyes (H1 transfer note 9).
9. **Portals.**
   - Stencil is per layer, so `SetupStencil`/`RemoveStencil` [U] src/rendering/hwrenderer/scene/hw_portal.cpp:249-418 work as they are.
   - The portal scene is built once, with per-view `ViewMatrix[i] × M_portal`.
   - The portal window from `ClearClipper` [U] src/rendering/hwrenderer/scene/hw_portal.cpp:427-456 is the union of both eyes' windows (dual clipper).
   - Default `r_portal_recursions` (4 [U] src/common/rendering/hwrenderer/data/hw_cvars.cpp:45) to 2 until Phase 5.

**ZScript/mod compatibility.**
- `RenderOverlay` 2D output becomes a head-locked quad layer (P2 §4).
- Camera textures stay mono [U] src/rendering/hwrenderer/hw_entrypoint.cpp:375-386.

**Risks.**
- A dual-clipper bug shows up as pop-in in one eye only. Ship a debug view that diffs each eye against a no-cull reference.
- Some drivers serialize multiview. Measure it.

**Success metric.**
- One `Scene.Create` per frame.
- Stereo render CPU ≤ 1.15× mono (**Inference**).
- 90 Hz on Doom II MAP01–10 at native per-eye resolution.

**Effort (Inference).** Layered fallback 1, OpenXR 4, frustum and dual clipper 2, single-pass refactor 4, multiview, PP and portals 3: **≈ 14 person-weeks.**

---

## Phase 3 — Tic-rate scene build with GPU interpolation, adapted for VR

**Goal.** Move scene *construction* to tic rate (35 Hz): the BSP walk, item processing, light lists, render hacks and the translucent tree. Each tic produces a `TicScene`. Every frame until the next tic redraws it with a per-frame head pose, a per-frame cull, and interpolation in the vertex shader. Unlike Helion, the view direction is not frozen between tics.

**Helion evidence.**
- *Rebuild gate.* Traversal and buffer refill happen only when `NeedsRenderTickChange` fires [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:143, 310-341, 789-799, and the buffers are uploaded once [H] Core/Render/OpenGL/Buffer/BufferObject.cs:94-101.
- *GPU interpolation.* Vertices carry `prevPos`/`prevUV` [H] Core/Render/OpenGL/Renderers/Legacy/World/DynamicVertex.cs:6-35, mixed by `timeFrac` [H] Core/Render/OpenGL/Renderers/Legacy/World/InterpolationShader.cs:63-68, [H] Core/Render/OpenGL/Renderers/Legacy/World/Entities/EntityProgram.cs:144.
- *Conservative culling over the tic.* Both camera positions are kept [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:180-196, and both 360° clippers must agree [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.Bsp.cs:73-88.
- *What breaks in VR.* Yaw is frozen at the tic's first frame (H3 open question 2). Culling is off above 45° of pitch [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:773-787. A head turn larger than (90° − hFOV/2) within one tic exposes culled geometry (H1 §7).

**UZDoom code to change (verified).**
- *CPU actor interpolation, per frame* [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:840-848.
- *Sector interpolation writes live playsim state.* `D_Render` calls `DoInterpolations`/`RestoreInterpolations` [U] src/d_main.cpp:600-620. `DSectorPlaneInterpolation::Interpolate` sets the plane's D and texZ and calls `P_RecalculateAttached3DFloors` and `CheckPortalPlane` [U] src/r_data/r_interpolate.cpp:470-500. The scroll and polyobject variants are at lines 69, 97, 124 and 609, 709, 818.
- *Moving planes re-patch flat vertices every frame*: `CheckUpdate` [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:794-797 → `UpdatePlaneVertices` [U] src/rendering/hwrenderer/hw_vertexbuilder.cpp:406-421.
- *`TicFrac`* is taken from the frame-start time [U] src/rendering/hwrenderer/hw_entrypoint.cpp:343-344, [U] src/common/utility/i_time.cpp:170-177.
- *Precedent:* models already interpolate in the vertex shader [U] wadsrc/static/shaders/glsl/main.vp:66-67.

**Design.**
- **A. `TicScene`** *(sketch)*: `{ ticIndex; DrawItem opaque[], translucentStatic[]; SpriteInstance sprites[]; PortalRecord portals[]; lightIds[]; envelope; transferHeightsZone; GpuSlice ticState; }`. It is immutable once built and ref-counted by the frames that use it.
- **B. Rebuild triggers:**
  - a new tic;
  - a change of the cyclops heightsec zone (`hw_FakeFlat` [U] src/rendering/hwrenderer/scene/hw_fakeflat.cpp:208ff; Helion likewise rebuilds on this change [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:143);
  - a camera discontinuity;
  - the head leaving the `envelope`.

  The rebuild runs synchronously after the tic until Phase 8 moves it to a worker.
- **C. Conservative visibility over head motion.**
  - Positions: the capsule from the prev to the cur cyclops position (the body position at each tic plus the head offset at build time), inflated by `r = IPD/2 + v_head·(T_tic + T_latency)`. **Inference:** about 8 cm, ≈ 3–4 units at 41 units/m [U] src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:39-45.
  - Orientations: all of them (360°, no near-plane half-space).
  - Occlusion: run clippers from sample points along the capsule axis, both ends included, spaced `s` apart, and cull only when **all** of them agree. Test each sample with radius `ρ = sqrt(r² + (s/2)²)`: shrink each occluder range by `2·asin(ρ/d)` per side and skip occluders closer than `ρ`. With `s = 2r`, `ρ = r·√2`. **Inference (geometry):** parallax within radius `ρ` is ≤ `asin(ρ/d)` for the occluder and less for anything behind it, so each test is conservative for every viewpoint within `ρ` of its sample. Every point of the capsule lies within `ρ` of some sample. Helion's dual rule (the two ends, no inflation) is the 2-sample case, and it is **not** conservative between the ends. A running player moves about 16–23 units per tic (the prev→cur distance `L`) against `r` ≈ 3–4, so the gap between the ends is the normal case. The cheap alternative is one clipper at the midpoint with radius `R = r + L/2`: it costs one walk, but it ignores more near occluders.
  - Gameplay-visible flags. The renderer marks `ML_MAPPED` and the `SECMF_DRAWN`/`SSECMF_DRAWN` flags during the walk [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:310-330, 379, 824. A 360° build must not set them, or the automap reveals what is behind the player. Until Phase 5 replaces them with GPU feedback, set them from the per-frame frustum cull below (item → line, subsector or sector id).
  - Per frame, cull the item AABBs against the current combined frustum: about 0.1–0.3 ms per 10k items on the CPU, and a GPU pass from Phase 5 on.
  - **Honest accounting (Inference).** A 360° build emits 2–4× the items of a frustum build, so 35 such builds cost about the same CPU as 90 frustum builds. The win is that the frame path drops to cull plus submit, and that the build gets a 28 ms deadline on a worker (Phase 8). Phases 4 and 5 then take walls and flats out of the build entirely.
- **D. How prev/cur reaches the shaders.** Tic-state SSBOs live in a 3-slot ring. *(sketch)*
  ```glsl
  struct PlaneTS { vec4 cur; vec4 prev; };   // secplane (a,b,c,d)
  layout(std430, set=3, binding=0) readonly buffer SectorPlanes { PlaneTS planes[]; };
  layout(std430, set=3, binding=1) readonly buffer SideOffsets  { vec4 sideOfs[]; };   // scroll u,v cur+prev
  float planeZ(uint p, vec2 xy, float t) { vec4 a = planes[p].prev, b = planes[p].cur;
    return mix(-(a.w + dot(a.xy, xy)) / a.z, -(b.w + dot(b.xy, xy)) / b.z, t); }
  ```
  - This matches `DSectorPlaneInterpolation`, which lerps D with the normal fixed [U] src/r_data/r_interpolate.cpp:496-497.
  - *Flats.* The load-time flat vertices [U] src/rendering/hwrenderer/hw_vertexbuilder.cpp:299-340 gain a `planeIndex`, so `CheckUpdate` [U] src/rendering/hwrenderer/hw_vertexbuilder.cpp:462-469 leaves the frame path.
  - *Walls on movers* get tic-rate vertices with `zPrev, zCur, vPrev, vCur` (Helion's `DynamicVertex`), replaced by plane indices in Phase 4.
  - *Scroll and polyobject* prev/cur values replace their interpolators.
  - `uTicFrac` comes from the **predicted display time** (P2 §6).
  - With all of this in place, `D_Render` stops calling `DoInterpolations` [U] src/d_main.cpp:610, 616-619. The interpolator objects remain only as the source of the prev values. That is the Phase 8 prerequisite.
  - Behaviour change: anything that reads map state during a frame now sees tic values, not interpolated ones. That covers ZScript `RenderOverlay`/UI code that reads plane heights, CPU sprite clipping against floors (`gl_spriteclip`), and the 3D-floor light lists that `P_RecalculateAttached3DFloors` used to refresh per frame. Each needs a prev/cur read or the GPU path, and the pixel diff (P3 §5.2) must run with moving floors and 3D floors.
- **E. What stays per frame.**
  - Head pose and view matrices.
  - The cull.
  - `uTicFrac`.
  - Sky rotation and scroll (`R_UpdateSky` [U] src/rendering/r_sky.cpp:130-146).
  - The material timer, today computed per draw [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:343-346; it moves to a per-frame UBO.
  - ZScript `Shader.SetUniform*` values.
  - HUD and post-processing.
  - Sprite billboarding and rotation (vertex shader, Phase 7).
  - The sprite translucent key sort.
- **Render hacks.** The test `Planez > Viewpoint.Pos.Z` [U] src/rendering/hwrenderer/scene/hw_renderhacks.cpp:543 tracks head Z. When the envelope straddles the plane, emit both variants and let the vertex shader collapse the wrong one. Helion collapses its flood quads the same way [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Portals/FloodFill/FloodFillProgram.cs:82-86.

**Risks.**
- Envelope violations (lean, teleport) force a synchronous rebuild. Count them.
- Three tic-state slots cost a few MB.

**Success metric.**
- `Scene.Create` runs ≤ 35 times per second at 120 Hz.
- Per-frame render CPU ≤ 2.5 ms on the huge-geometry scenario (≤ 1 ms after Phases 4 and 5).
- Zero misses in the head-shake test (P3).

**Effort (Inference).** `TicScene` 3, envelope clipper 2, tic-state SSBOs and flat VS 3, dynamic walls 2, removing `DoInterpolations` 2, cull 1, hack variants 1: **≈ 14 person-weeks.**

---

## Phase 4 — GPU-resident static world (with 3D floors, slopes, render hacks, decals)

**Goal.** All map geometry lives in device-local buffers built at load. Movers, texture changes, light changes and scrolls patch **tables**, never vertices. The opaque world draws in a few `vkCmdDrawIndexedIndirectCount` calls per pass. Phase 5 culls it on the GPU, unlike Helion. Every UZDoom geometry feature stays.

**Helion evidence.**
- *Whole-map static VBOs* per (type, texture array) [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Static/StaticCacheGeometryRenderer.cs:103-172, 864-892.
- *Event patching* with a free list [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Static/StaticCacheGeometryRenderer.cs:983-1027, 1120-1200, 1360-1402, [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Static/FreeGeometryManager.cs:87-133.
- *Sector-indexed light, colour and fog tables* with dirty lists [H] Core/Render/Renderer.World.cs:63-72, 386-463.
- *Animation through a translation table* [H] Core/Resources/TextureManager.cs:479-494.

What not to copy:
- movers beyond `MaxDistance` vanish (H2 §2.3);
- the double draw of scrollers (H1 §2.1);
- the whole map transformed per eye (H1 §2.6);
- no slopes, no portals, no 3D-floor culling (H2 §7).

**UZDoom code to change (verified).**
- *Walls are streamed at draw time* [U] src/rendering/hwrenderer/scene/hw_walls_vertex.cpp:279-288, [U] src/rendering/hwrenderer/scene/hw_walls.cpp:364-373, with per-wall state and one draw per 3D-floor light slice [U] src/rendering/hwrenderer/scene/hw_walls.cpp:205-330.
- *Flats are already static and indexed* [U] src/rendering/hwrenderer/hw_vertexbuilder.cpp:30-110, 299-340, [U] src/rendering/hwrenderer/scene/hw_flats.cpp:192-204.
- *Render hacks* [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:501-504, [U] src/rendering/hwrenderer/scene/hw_renderhacks.cpp:42-75, 143-184, 535-600, and the 3-pass flood stencil [U] src/rendering/hwrenderer/scene/hw_flats.cpp:241-295.
- *Decals:* one item plus 4 streamed vertices each [U] src/rendering/hwrenderer/scene/hw_decal.cpp:384-421.
- *`DoomLevelMesh`* [U] src/rendering/hwrenderer/doom_levelmesh.h:37-54 is a ray-tracing mesh. Reuse its surface classification, not its mesh.

**Implementation steps.**
1. **Cluster mesh built at load**, after `CreateVBO` [U] src/maploader/maploader.cpp:3232. A cluster is one wall tier of one sidedef, one plane of one section, or one decal batch. *(sketch)*
   ```cpp
   struct WorldVertex { float x, y; uint32_t zPlane; float u; uint32_t vAnchor; uint32_t clusterId; };   // Z and V from plane tables
   struct ClusterGPU  { AABB box; uint32_t firstIndex, indexCount, materialSlot, lightPart, sideOrPlane, flags; };
   ```
   The vertex shader recomputes Z and V from the plane table (Phase 3 D), reproducing the pegging rules of `SetWallCoordinates` [U] src/rendering/hwrenderer/scene/hw_walls.cpp:1099-1251. Keep a CPU reference implementation and unit-test the two against each other on every benchmark map.
2. **T-junctions** (`gl_seamless`, today per frame [U] src/rendering/hwrenderer/scene/hw_walls_vertex.cpp:31-203). Insert a vertex at every potential split height at load, each with its own `zPlane`, so cracks stay closed while planes move.
3. **Material slots.** `ClusterGPU.materialSlot` → slot table → `MaterialGPU`. ANIMDEFS, switches and `SetTexture` each rewrite 4 bytes. Helion instead moves the geometry to another VBO [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Static/StaticCacheGeometryRenderer.cs:1265-1307.
4. **Light tables.**
   - Per sector part: light, `FColormap`, special and additive colours, glow, gradient.
   - Per side tier: `rellight` and the per-tier colours.
   - Write the dirty entries once per tic from a `SectorUpdates`-style set [H] Core/Render/SectorUpdates.cs:27-43, into the frame ring.
5. **3D floors without per-slice draws.** Each cluster gets a `lightListOffset` into `{planeIndex, lightPart}` entries, sorted top-down. The fragment shader picks the entry `i` with `plane[i](xy) ≥ z > plane[i+1](xy)`: the region whose top plane is the nearest plane *above* the fragment. That is the band today's per-slice draw clips to with `uSplitTopPlane`/`uSplitBottomPlane` [U] src/rendering/hwrenderer/scene/hw_walls.cpp:293-320. That replaces the slice loops for walls, decals and sprites ([U] src/rendering/hwrenderer/scene/hw_walls.cpp:293-320, [U] src/rendering/hwrenderer/scene/hw_decal.cpp:87-117, [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:243-298). **Inference:** the lists are short and uniform per cluster.
6. **Slopes** are free: the plane table already stores `(a, b, c, d)`. Keep the rule that disables mirrors on slopes [U] src/maploader/maploader.cpp:3215-3220, and match the loader's vertex-height slopes [U] src/maploader/slopes.cpp:145, 367.
7. **Render hacks become resident.**
   - Pre-classify the candidate gaps at load with the same flood search [U] src/rendering/hwrenderer/scene/hw_renderhacks.cpp:535-600.
   - Emit "eye above" and "eye below" variants, and let the vertex shader collapse the wrong one by cyclops Z (Helion's flood collapse [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Portals/FloodFill/FloodFillProgram.cs:82-86).
   - Flood planes become a per-fragment ray-plane shader [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Portals/FloodFill/FloodFillProgram.cs:121-149, using the **per-view** `uCameraPos[gl_ViewIndex]` so the parallax is correct.
   - Re-classify the sectors whose planes changed, with a per-tic budget.
8. **Decals** become resident quads anchored with `zPlane`, allocated on spawn and freed on expiry. The arena is bounded by `cl_maxdecals` (1024 [U] src/g_cvars.cpp:90). They draw in one indirect pass.
9. **Patches are written once per tic:**
   - a plane move writes 1 `PlaneTS`;
   - light, colour or fog writes 1 entry;
   - a texture change writes 1 slot;
   - a scroll writes 1 offset;
   - a render-affecting line flag writes 1 cluster flags word;
   - a polyobject move rewrites its segs in a dynamic arena with prev/cur;
   - a runtime portal change writes flags plus the portal table.
10. **Submission.** Buckets (opaque, masked, translucent, sky-stencil, portal-stencil, fog-boundary, decal) each get one `vkCmdDrawIndexedIndirectCount` per view pass. Before Phase 5, the CPU fills the indirect buffer from the `TicScene`.

**ZScript/mod compatibility.**
- Every native map setter behind `Sector.*`, `Line.*` and `Side.*` (`SetPlaneTexZ`, `SetTexture`, `SetLightLevel`, `SetColor`, `SetFade`, `SetGlowColor`, `SetSpecialColor`, …) marks the dirty sets.
- Setters are not enough. ZScript can also write plain native fields directly: `Sector.lightlevel`, `Side.Flags` and `Line.flags` are writable [U] wadsrc/static/zscript/mapdata.zs:379, 138, 232, and ACS and native code write fields too. Once per tic, also compare the render-relevant fields of all sectors, sides and lines against a shadow copy. That is linear and cache-friendly, a few bytes per element. The setter marks keep the common case cheap; the compare catches the rest.
- A debug mode diffs the tables against a fresh CPU computation every tic.
- GLDEFS shaders work through the Phase 1 preamble.

**Risks.**
- Pegging parity: `DoMidTexture` (from [U] src/rendering/hwrenderer/scene/hw_walls.cpp:1412) and its `GetAreas` split (1665-1712), `wrapmidtex`, 3D midtex. Gate on a per-map pixel diff (P3 §5).
- Hack pre-classification misses. Keep the legacy tic-rate hack path as a fallback.

**Success metric.**
- 0 wall or flat vertices allocated per frame.
- ≤ 50 world draws per view pass.
- Mover and light updates < 1 µs each.
- Pixel diff clean.

**Effort (Inference).** Mesh 4, pegging 5, slots 1, tables 3, 3D lists 3, hacks 4, decals 2, setter audit 2: **≈ 24 person-weeks.**

---

## Phase 5 — Visibility rebuild: GPU culling, portals against the resident world, CPU traversal only for gameplay flags

**Goal.** Rendering visibility moves to the GPU, per view pass and per frame: frustum culling against the combined frustum, Hi-Z occlusion, and portals. The CPU BSP walk leaves the frame path. What gameplay reads (automap marks, `DRAWN` flags) is fed back from GPU visibility.

**Helion evidence.** None to copy: no GPU culling, queries, indirect draws or Hi-Z (H2 "Absent"), an unculled static cache (H1 §2.6), and an island test as the only PVS-like mechanism [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:217-247. Its tic-frozen cull is unsafe for fast yaw (H3 §1). Here UZDoom has to go beyond Helion.

**UZDoom code to change (verified).**
- *The CPU pipeline:* `RenderBSP` [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:1008-1062, `RenderBSPNode`/`CheckBox` (947-985), `AddLine` (257-403), and the clipper [U] src/rendering/hwrenderer/scene/hw_clipper.h:31-97.
- *Portals re-run all of it* [U] src/rendering/hwrenderer/scene/hw_portal.h:178-186, [U] src/rendering/hwrenderer/scene/hw_portal.cpp:73-103, and sector stacks walk every node (779-813).
- *Renderer writes that gameplay sees:*
  - `ML_MAPPED` [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:379;
  - `SSECMF_DRAWN` (310-330, read back by the radar clipper at 723);
  - `SECMF_DRAWN` (824);
  - `RF_MAYBEINVISIBLE` [U] src/rendering/hwrenderer/scene/hw_portal.cpp:655;
  - dither targets through `P_CheckSight` [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:1049-1055.

**Implementation steps.**
1. **Cull compute pass.** One pass serves both eyes under multiview. It reads `ClusterGPU[]`; mover AABBs take the union of their prev and cur planes. Per cluster:
   - (a) flags;
   - (b) the 6 planes of the combined frustum;
   - (c) a GPU copy of `CurrentMapSections` ([U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:262-268, [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:696-697; portals add destination sections, [U] src/rendering/hwrenderer/scene/hw_portal.cpp:668);
   - (d) Hi-Z;
   - (e) an atomic append to the bucket's indirect array.

   Sprite instances and models use the same kernel.
2. **Two-phase Hi-Z** *(technique; Inference on fit)*.
   - (i) Draw last frame's visible clusters (persistent bits).
   - (ii) Build one depth pyramid per view layer. A cluster counts as occluded only if it is occluded in **both** eyes.
   - (iii) Test the rest and draw the newly visible ones.

   There is no readback and no pop on a head turn.
3. **Portals on the resident world.**
   - The kernel writes a visibility flag per portal.
   - For each candidate portal, the CPU records: stencil-mark as today [U] src/rendering/hwrenderer/scene/hw_portal.cpp:249-418, `ViewMatrix[i] × M_portal`, the clip plane, a cull with the portal frustum, the indirect buckets, and stencil removal.
   - Wrap each portal pass in **`VK_EXT_conditional_rendering`** keyed on its flag (**API fact**: it predicates draws, dispatches and `vkCmdClearAttachments`; begin and end must sit in the same subpass; the flag buffer needs a barrier to `VK_PIPELINE_STAGE_CONDITIONAL_RENDERING_BIT_EXT` / `VK_ACCESS_CONDITIONAL_RENDERING_READ_BIT_EXT`).
   - **API fact:** `vkCmdDispatch` is not allowed inside a render pass instance. Record every candidate portal's cull dispatch (per recursion level; the portal matrices are known on the CPU) *before* the scene render pass, not interleaved with the portal draws.
   - Recursion stays CPU-driven and needs no BSP walk. **Inference:** a portal then costs one dispatch plus a few draws, which makes `r_portal_recursions` 4 affordable again.
   - `SetCoverage` becomes a per-portal cluster mask computed at load.
4. **Skybox viewpoints → cubemap** (*beyond Helion*). Render skybox areas [U] src/rendering/hwrenderer/scene/hw_portal.cpp:703-744 into a cubemap with multiview `viewMask = 0b111111` (**API fact:** `maxMultiviewViewCount` ≥ 6). Do it at tic rate when static and per frame otherwise, then sample it in the sky stencil. Parallax inside the skybox is lost; keep a per-map override.
5. **Gameplay flags from GPU visibility.**
   - `ML_MAPPED` and the `DRAWN` flags must keep meaning "seen", or the automap reveals what lies behind the player.
   - The kernel sets a "drawn this frame" bit per cluster. An async readback, 2 frames later, maps the bits to line, subsector and sector ids and hands a delta to the sim thread, which applies it at the start of the next tic. Section granularity is an automap-only difference.
   - These flags are render-dependent today too (frame rate, resolution, view), so they are not demo- or net-synced state. Applying them at tic start only keeps them from changing mid-tic. P3's hash log hashes line flags, so it must mask `ML_MAPPED`.
   - Before Phase 5 lands, Phase 8 feeds the same delta list from the CPU walk (Phase 8 step 1b).
   - Never write map data from a worker. Helion's lock-free AutomapMarker loses bits [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:262-266.
   - Dither sight checks move to the sim thread.
   - `RF_MAYBEINVISIBLE` becomes a render-side flag.
   - Radar and ortho paths [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:714-780 stay CPU, at tic rate.
6. **Retire the per-frame BSP** once the pixel diffs pass. Only the Phase 3 `TicScene` walk remains.

**Risks.**
- Keep `r_gpucull 0`.
- Masked geometry makes Hi-Z conservative.
- `conditionalRendering` support must be verified on the target GPUs (**Inference**). Fallback without it: the cull kernel writes zero draw counts for invisible portals, consumed by `vkCmdDrawIndexedIndirectCount` (core in Vulkan 1.2). The stencil-mark and stencil-removal draws then also need GPU-written counts.

**Success metric.**
- ≤ 1.5× the triangles of a perfect-visibility reference.
- ≤ 0.05 ms CPU per visible portal.
- No per-frame `Scene.BSP`.

**Effort (Inference).** Kernel and indirect 4, Hi-Z 3, portals 5, cubemap 2, flag feedback 2: **≈ 16 person-weeks.**

---

## Phase 6 — Lighting: GPU light culling replaces per-surface CPU light lists; shadowmaps

**Goal.** Remove the O(surfaces × lights) CPU work and the silent 80k-entry overflow. Keep UZDoom's light *semantics*: section-scoped linking, portals, subtractive and additive lights, and the light flags. Helion has no dynamic lights at all (H6), so this is a feature that is kept and moved to the GPU.

**UZDoom code to change (verified).**
- *Linking is playsim work:* `CollectWithinRadius` fills `flat_dlist`/`wall_dlist` [U] src/playsim/a_dynlight.cpp:496-634, [U] src/g_levellocals.h:56-57. Positions are tic positions with no interpolation ([U] src/rendering/hwrenderer/hw_dynlightdata.cpp:46, 71; [U] src/playsim/a_dynlight.cpp:364).
- *Draw-time list building:*
  - walls: `HWWall::SetupLights` [U] src/rendering/hwrenderer/scene/hw_walls.cpp:405-555;
  - flats: `HWFlat::SetupLights` [U] src/rendering/hwrenderer/scene/hw_flats.cpp:142-184;
  - `UploadLights` returns −1 silently on overflow [U] src/common/rendering/hwrenderer/data/hw_lightbuffer.cpp:75-123.
- *Sprites:* one RGB per draw, plus CPU AABB-tree shadow rays [U] src/rendering/hwrenderer/scene/hw_spritelight.cpp:100-207, [U] src/common/rendering/hwrenderer/data/hw_shadowmap.cpp:84-90.
- *Models:* `BSPWalkCircle` [U] src/rendering/hwrenderer/scene/hw_spritelight.cpp:225-278.
- *Shader:* three list loops [U] wadsrc/static/shaders/glsl/material_normal.fp:18-112, then shadowmap PCF [U] wadsrc/static/shaders/glsl/main.fp:526-590 or ray queries (380-453).
- *Shadowmap update:* once per frame [U] src/rendering/hwrenderer/hw_entrypoint.cpp:103-110.

**Steps.**
- **Tier A (semantics-exact, do first).**
  1. A `LightGPU[]` array (prev and cur position, radius, colour, type, flags, cone, shadow index), uploaded per tic.
  2. Section and side light-index lists built **per tic** from the existing `flat_dlist`/`wall_dlist`; polyobject sides go through their subsector section, as `SetupLights` does today [U] src/rendering/hwrenderer/scene/hw_walls.cpp:427-490. Rebuild only sets that changed (`UpdateLocation` [U] src/playsim/a_dynlight.cpp:343).
  3. The fragment loop reads the cluster's section or side list and performs the plane and radius tests per fragment (today's CPU tests, [U] src/rendering/hwrenderer/scene/hw_walls.cpp:493-553).
  4. Delete `SetupLights`/`UploadLights` from the draw path.
  5. Optional `r_interpolatelights` (prev ≠ cur) for smoother moving lights.
- **Tier B (accelerator).**
  6. A froxel grid in **combined-frustum** space: one grid for both eyes (**Inference**).
  7. The fragment iterates `sectionMask AND froxelMask` over a per-tic compact light index space (≤ 4,096 lights = 128 words; the z-bin word-mask technique). Section linking still decides which lights may reach a fragment, so today's leak rules hold.
  8. Sprites and particles get per-pixel light through the same lookup. A per-instance compute "flat" mode keeps `gl_light_sprites` semantics, with GPU shadow tests instead of CPU rays.
- **Shadows.**
  9. Update the 1-D shadowmap (`gl_light_shadowmap`, off [U] src/common/rendering/hwrenderer/data/hw_shadowmap.cpp:57) only when lights or geometry change.
  10. Optional cube shadows for the top-K lights: rendered from the resident world with `viewMask = 0b111111`, cached, a few re-renders per frame.
  11. Ray-query shadows (`vk_raytrace` [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:63): 1 ray with temporal accumulation, or off, in VR.

**Compatibility.** Lights behave the same, because linking is unchanged. Above 4,096 active lights, Tier A runs without the froxel mask. There is no overflow.

**Success metric.**
- The light-heavy scenario's CPU light work drops to ≤ 0.2 ms per tic.
- GPU time ≤ 1.3× the no-light time at 200 visible lights.

**Effort (Inference).** Tier A 4, Tier B 4, sprite light 2, shadowmap 1, (cube shadows 4): **≈ 11–15 person-weeks.**

---

## Phase 7 — Sprites, particles, models and overlays: instanced sprites, per-view billboarding, OIT or sorted, GPU skinning, instanced models

**Goal.** A sprite becomes one instance record per tic, with billboarding, rotation and anamorphic clip done in the vertex shader from the cyclops position. Opaque sprites leave the translucent list. Models are instanced and skinned once per frame. The same instance path renders Phase 10's native overlays.

**Helion evidence.**
- A 40 B instance per sprite [H] Core/Render/OpenGL/Renderers/Legacy/World/Entities/EntityVertex.cs:7-28.
- `gl_VertexID` corners with instanced draws [H] Core/Render/OpenGL/Renderers/Legacy/World/Entities/EntityProgram.cs:142-159, [H] Core/Render/OpenGL/Renderers/Legacy/World/Data/RenderData.cs:61-63.
- Buckets keyed by (style, array) [H] Core/Render/OpenGL/Renderers/Legacy/World/Entities/EntityRenderer.cs:278-280.
- WBOIT [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:586-695 (its composite re-rasterises, H3 §6).
- BAM rotation at tic rate [H] Core/Render/OpenGL/Renderers/Legacy/World/Entities/EntityRenderer.cs:115-137.

Avoid: Helion's 8-rotation limit, missing scale and wall/flat/roll sprites, missing models (H3 §5, §11), its fuzz feedback loop [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:609-610, and its brightmap array exclusion [H] Core/Render/OpenGL/Texture/Legacy/GLTextureArrayBuilder.cs:201-205.

**UZDoom code to change (verified).**
- *`HWSprite::Process`* [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:798-1461: interpolation (840-848), rotation (969-1037), corners (1120-1175), anamorphic clip (1190-1238), light (1240-1411), slices (1438-1457). It writes into actors: `RF_XFLIP` and `isoscale*` (1055-1056, 1077-1095).
- *`DrawSprite`* (97ff): per-draw light, vertices per slice (243-287), fuzz (132-150).
- *All non-model, non-wall sprites are routed translucent* [U] src/rendering/hwrenderer/scene/hw_drawlistadd.cpp:138-153 and BSP-sorted [U] src/rendering/hwrenderer/scene/hw_drawlist.cpp:629-948.
- *Particles:* [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:1470-1657; `particle_t` is 136 B [U] src/playsim/p_effect.h:78-99.
- *Models:* a per-draw `AnimateBones` VM call [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:308-314, `CalculateBones`/`UploadBones` [U] src/r_data/models.cpp:570-652, and frame pairs [U] src/rendering/hwrenderer/hw_models.cpp:108-141, [U] wadsrc/static/shaders/glsl/main.vp:66.

**Steps.**
1. **The instance record** *(sketch, 64 B)*: `{ pos[3], prevPos[3]; angle, prevAngle; frameSet (→ SpriteFrameGPU{tex[16], flipMask, numRot}); scale (half2); offset (short2); styleAlpha|flags; lightPart; translation; roll }`. `Process` keeps all of its game logic and emits instances. The actor writes move into instance bits, so the worker never writes to `AActor`.
2. **Vertex shader.**
   - `mix(prevPos, pos, uTicFrac)`.
   - Rotation from `atan(p − cyclops) − mix(angles)`, with 8 or 16 rotations (UZDoom's `GetSpriteAngle` semantics [U] src/r_data/sprites.h:40).
   - Position-facing yaw billboards by default; `gl_billboard_mode` variants [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:82-83; flat, wall and roll sprites from the actor angles.
   - Anamorphic pull.
   - 3D-floor lists through the Phase 4 function.

   Everything is cyclops-relative, so both eyes see the same frame on one planar quad.
3. **Opaque routing.** Send `STYLEHW_Solid` sprites to an alpha-tested, instanced, early-Z bucket. Fix `AddSprite` (hw_drawlistadd.cpp:138-153), except when `gl_sprite_blend` is on [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:75. **Inference:** the translucent list shrinks by an order of magnitude on monster maps.
4. **Translucency.**
   - Additive, subtractive and add-colour styles commute: draw them unsorted. This holds on the float (RGBA16F) scene target; on a UNORM target, per-blend clamping breaks commutation.
   - True alpha blending: a per-frame GPU radix sort on cyclops depth, then one sorted instanced draw.
   - Translucent walls and flats keep a CPU tic-rate sort (`DoSort`/`Sort` [U] src/rendering/hwrenderer/scene/hw_drawlist.cpp:629-713, drawn by `DrawSorted` 871-948), merged by depth with the sprites.
   - Optional WBOIT (`r_oit`), at +12 B per pixel per eye, with a full-screen composite.
5. **Fuzz.** `fuzz_software.fp` uses `gl_FragCoord` and `uViewHeight` [U] wadsrc/static/shaders/glsl/fuzz_software.fp:55-58, which differ per eye. Use angular, cyclops-relative noise instead. Refraction reads a copied colour buffer, never the bound attachment.
6. **GPU translations.** A palette-remap texture plus R8 index textures replaces one `FHardwareTexture` per translation [U] src/common/textures/hw_texcontainer.h:79-141, so translated sprites batch.
7. **Particles.** At tic end, compact them into 32–40 B instances. Simulation stays in `P_ThinkParticles` [U] src/playsim/p_effect.cpp:278-351, with a cached subsector. GPU-only cosmetic particles are optional, after a ZScript API audit. VisualThinkers emit instances like actors do.
8. **Models.**
   - (a) Run `AnimateBones` and `CalculateBones` once per frame per visible actor in a pre-pass, cached in the bone SSBO ([U] src/rendering/hwrenderer/scene/hw_sprites.cpp:308-318, [U] src/r_data/models.cpp:570-652). This is a documented frequency change: from per eye or portal to per frame. **Open risk (Phase 8):** `AnimateBones` is a `ui virtual` that reads the actor (wadsrc/static/zscript/actors/actor.zs:1651). Once the sim runs on its own thread it cannot run on the render thread. The options are to call it on the sim thread at snapshot time for the tic endpoints and interpolate bones on the GPU (an approximation), or to hold a reader lock that stalls the sim (no decoupling on maps with such models).
   - (b) Compute pre-skinning for IQM.
   - (c) Instanced MD2, MD3, OBJ, UE1 and voxel draws with vertex pulling (`frameOffset`), one indirect draw per mesh surface.
   - (d) Phase 6 per-pixel light replaces `BSPWalkCircle`.
9. **Overlays (Phase 10).** `ActorOverlay` slots in the snapshot record emit 1–2 instances each (frame and fill), with fill computed in the shader from `health / SpawnHealth`.
10. **VR weapon.** It becomes a world-space model or a sprite quad on the hand pose, replacing `DrawPlayerSprites` [U] src/rendering/hwrenderer/scene/hw_weapon.cpp:114 and the HUD-model depth clear [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:917-924.

**Compatibility.**
- Every sprite property maps to instance fields.
- Override shaders get their own bucket.
- `AnimateBones` changes frequency.

**Risks.**
- Sort differences against the BSP splitter (keep it as an option).
- Alpha-to-coverage needs MSAA.

**Success metric.**
- 4,000 visible monsters: sprite CPU ≤ 0.3 ms per tic and about 0 per frame.
- ≤ 10 sprite draws per pass.
- Identical rotation frames in both eyes.
- ≤ 5 µs per model per frame.

**Effort (Inference).** Instances 4, routing 0.5, translucency 3, fuzz 1, translations 3, particles 2, models 6, overlays 1, weapon 2: **≈ 22.5 person-weeks.**

---

## Phase 8 — Decoupled sim thread: render snapshot, interpolation state, late-latched head pose

**Goal.** A 5–15 ms slaughter-map tic (U3 §5, Inference) must never land inside an 8–11 ms frame. Run `G_Ticker` on a dedicated sim thread at 35 Hz. The render thread runs at display rate and consumes an immutable per-tic **render snapshot**. The head pose is sampled as late as possible. The simulation stays single-threaded and deterministic.

**Helion evidence.**
- Helion is single-threaded too: the sim and all GL work run on one thread [H] Client/Client.cs:388-406.
- Its one background reader of world state, AutomapMarker, races the main thread on line flags [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:262-266. That is the cautionary example: any off-thread consumer needs a snapshot.
- Helion caps catch-up at one tic per frame [H] Core/Util/Timing/Ticker.cs:81-104. That policy is irrelevant once the sim has its own thread.

**UZDoom code to change (verified).**
- *Serial loop.* `TryRunTics()` then `D_Display()` [U] src/d_main.cpp:1463-1467. `TryRunTics` catches up with several tics per frame [U] src/d_net.cpp:2312-2335.
- *The renderer mutates or reads live sim state.* Each of these must stop:
  - (a) `DoInterpolations` writes interpolated plane heights into sectors [U] src/r_data/r_interpolate.cpp:470-500, from [U] src/d_main.cpp:610. Removed by Phase 3.
  - (b) `P_FindParticleSubsectors` and `PO_LinkToSubsectors` per frame [U] src/d_main.cpp:611-612. F9 moves the particle part to the sim. `PO_LinkToSubsectors` lazily rebuilds the render subsector links of moved polyobjects [U] src/playsim/po_man.cpp:1703-1712; it must also move to tic end.
  - (c) Sprite processing writes `RF_XFLIP`/`isoscale*` into actors [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:1055-1056, 1077-1095. Removed by Phase 7.
  - (d) The renderer writes `ML_MAPPED`/`SECMF_DRAWN`/`SSECMF_DRAWN` [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:310-330, 379, 824. Step 1b turns these into a delta list, fed by the CPU walk; Phase 5 feeds it from the GPU later.
  - (e) Dither targets call `P_CheckSight` from the renderer [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:1049-1055.
  - (f) The renderer and the playsim share the global `validcount` [U] src/rendering/r_utility.cpp:176, and the renderer stamps `line_t::validcount`/`sector_t::validcount` (for example the sector check at [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:794).
  - (g) `hw_FakeFlat` reads live heightsec sectors [U] src/rendering/hwrenderer/scene/hw_fakeflat.cpp:208ff.
  - (h) Both `CheckGC` calls run on the main loop [U] src/d_net.cpp:2192, [U] src/d_main.cpp:1115.
  - (i) The sound update reads `actor->Pos()` and `Sector->PortalGroup` for every actor-attached channel [U] src/sound/s_doomsound.cpp:1063-1071, from `S_UpdateSounds` in `TryRunTics` [U] src/d_net.cpp:2298, 2345.
- *UI-scope ZScript at frame rate.* Status bar `Draw` and event-handler `RenderOverlay`/`RenderUnderlay` read play data during rendering. That is legal today only because everything runs on one thread.

**Implementation steps.**
1. **Render-private visibility stamps.** Give the renderer its own `render_validcount` and epoch arrays indexed by line, sector and subsector id (F8's arrays). It stops writing `validcount` fields in map structs.

   **1b. Remove the other render→play accesses before the split** (this does not wait for Phase 5):
   - (d) the CPU BSP walk records `ML_MAPPED`/`DRAWN` marks into a render-private delta list, which the sim thread applies at tic start (the Phase 5 step 5 mechanism, fed by the CPU walk);
   - (e) the dither `P_CheckSight` runs on the sim thread against the targets of the last published frame;
   - (g) the Phase 3 `TicScene` build, which calls `hw_FakeFlat` and reads live sectors, lines and actors, runs on the sim side after the tic and is published with the snapshot. The render thread never rebuilds from live data. An envelope violation (Phase 3 B) requests a rebuild from the sim side and draws the old `TicScene` with an unculled fallback for that frame.
   - `RF_MAYBEINVISIBLE` becomes a render-side flag (Phase 5 step 5).
2. **Snapshot builder on the sim thread**, at the end of each tic. *(sketch)*
   ```cpp
   struct RenderSnapshot {               // immutable after publish; ref-counted; triple-buffered
     uint32_t tic; uint64_t ticStartNs;
     TArray<SpriteInstance> sprites;     // Phase 7 records (prev+cur already inside)
     TArray<ParticleInstance> particles;
     DirtyTables tables;                 // sector planes, light parts, material slots, offsets (dirty ranges + data)
     TArray<LightGPU> lights; SectionLightLists lightLists;     // Phase 6
     TArray<PolyVertex> polyVerts;       // polyobject seg endpoints prev/cur
     TArray<DecalOp> decalOps;           // add/remove
     CameraState camera;                 // body pos prev/cur, viewheight/bob, angles, player flags, fixedcolormap, extralight
     HUDState hud;                       // psprite states/offsets prev/cur, for the VR weapon
     TArray<VisDelta> visDeltaAck;       // gameplay-visibility deltas already applied
   };
   ```
   - The cost is proportional to the *changed* state plus the actor count (**Inference:** 10k actors × 64 B ≈ 0.6 MB, about 0.2–0.4 ms, on the sim thread).
   - Publish it with an atomic pointer swap. The render thread acquires the newest snapshot at frame start and keeps the previous one for interpolation.
   - The snapshot holds plain data and ids, never `DObject` pointers, so a GC step on the sim thread can never free something the render thread reads.
3. **Run the sim thread.**
   - `D_DoomLoop` becomes two threads. The sim thread runs `TryRunTics` with its own pacing: sleep until the next tic boundary on the engine clock, and run at most 2 catch-up tics per wake, logging when that happens. That cap applies to single-player only (game time slows instead of spiralling). Netgames and demo playback keep today's catch-up, because the peers' tic stream must be consumed. `NetUpdate` is only called from `TryRunTics` [U] src/d_net.cpp:2221, so all netcode stays on the sim thread.
   - The render thread runs the OpenXR frame loop (P2 §6).
   - Input: the render thread pushes events, poses and controller states into a lock-free queue. The sim thread builds tic commands from them (`G_BuildTiccmd`) at tic time, using the newest pose. Aim direction comes from the controller pose, as Helion folds per-frame mouse look into the tic command [H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:396-407.
4. **GC on the sim thread.** Move both `CheckGC` calls into the sim thread, after the tic and after the snapshot publish. Total GC work tracks the allocation rate, not the call count [U] src/common/objects/dobjgc.cpp:155-163, 331-335. A time budget per step (`Step()` with a deadline, finishing the remaining work next tic) keeps the worst case bounded. GC pacing already depends on frame rate today (the `D_Display` call), so a deadline adds no new non-determinism. But a sweep calls `Destroy()`, and thus ZScript `OnDestroy`, on unreachable objects that were not destroyed explicitly [U] src/common/objects/dobjgc.cpp:224-228. That is another reason the GC may only run on the sim thread.
5. **UI-scope ZScript on the sim thread.**
   - Run the status bar `Draw`, `RenderOverlay`/`RenderUnderlay` and `UiTick` on the sim thread right after the snapshot. They record into a second, sim-owned `F2DDrawer` instance, double-buffered with the snapshot (`F2DDrawer` merges commands [U] src/common/2d/v_2ddrawer.cpp:211-224). The global drawer stays with the render thread for menus and the console.
   - Publish the recorded 2D command list with the snapshot. The render thread replays it into the HUD quad layer (P2 §4).
   - The HUD now updates at 35 Hz. In VR it is a compositor-reprojected quad, so this is invisible except for interpolated HUD animation. `RenderEvent.FracTic`/`ViewPos` then describe the tic, not the frame.
   - Menus and the console run on the render thread only while the sim thread is **blocked** at a barrier. A "paused" game state is not enough: DObject allocation and the GC (`GC::Root`, `AllocBytes`, barriers) are single-threaded, and menu code allocates DObjects. The VM frame stack is already `thread_local` [U] src/common/scripting/vm/vmexec.cpp:147. In netgames the menu does not pause the sim, so there the menu ticks and draws on the sim thread too (35 Hz).
6. **Late-latched head pose.**
   - The render thread culls with the pose predicted at `xrWaitFrame` time plus an angular margin.
   - Right before `vkQueueSubmit`, it re-queries `xrLocateViews` for the same `predictedDisplayTime`, which gives a newer estimate. It writes the final view and projection matrices into the frame's host-visible `ViewpointUBO` slot. The command buffer references the slot, not the values.
   - It submits the same final poses in `XrCompositionLayerProjectionView`.
   - The cull margin is `ω_max × (t_submit − t_cull)`. **Inference:** at 300°/s over 3 ms, that is about 1°.
7. **Audio.** `S_UpdateSounds` reads live actor positions (item (i)), so it stays on the sim thread at tic rate, as today. The render thread only updates the listener orientation from the head pose, through a backend call that touches no play data. Alternatively, resolve channel positions from the snapshot. Sound starts stay on the sim thread.

**ZScript/mod compatibility.** Play-scope scripts run exactly as before, on one thread, in the same order: demos and netplay are unaffected. UI-scope scripts change thread and run at 35 Hz in game, between tics against a consistent world; mods animating `Screen.*` drawing in `RenderOverlay` at frame rate get 35 Hz animation inside the HUD layer (document it). Make the CVar store thread-safe (mutex on write, atomic reads for hot cvars).

**Risks.**
- **Data races are the main risk.** Run ThreadSanitizer builds in CI. A "no render access to play objects" rule can be enforced in debug builds by poisoning `AActor`/`sector_t` reads from the render thread: an assert on thread id in `TObjPtr` access and the map accessors.
- Hidden renderer→play reads, for example `players[consoleplayer].camera` inside hw code and `Level->` accessors throughout `hw_*`. Audit them with `grep -rn "Level->\|players\[" src/rendering/hwrenderer` and convert each to a snapshot read.

**Success metric.**
- In the slaughter scenario, render-thread frame time is independent of tic cost: p99 render frame ≤ budget while `Sim.Tic` p99 is 15 ms.
- Missed frames caused by sim stalls = 0.
- TSAN clean on all 5 scenarios.

**Effort (Inference).** Render-private stamps and step 1b 2, snapshot builder 4, thread split + input 3, UI-scope move + 2D list publishing + menu barrier 3, late latch 1, race audit + TSAN 4: **≈ 17 person-weeks.**

---

## Phase 9 — Playsim: native fast paths, dormancy, optional slow tick, spatial acceleration, allocation and GC

**Goal.** Bring the slaughter-map tic down to a budget the sim thread can hold at 35 Hz with headroom. Target ≤ 10 ms p99 at 5,000 awake monsters (**Inference**). Every default-on item must be bit-exact for demos. Approximations are opt-in only.

**Helion evidence.** Take: monster closets (island classification plus a cheap `A_Look` substitute) [H] Core/World/Geometry/Islands/ClosetClassifier.cs:30-122, [H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:1595-1601, 3406-3427, default on [H] Core/Util/Configs/Components/ConfigGame.cs:163-164; the 16-unit BSP point grid [H] Core/World/WorldBase.BspBlockMap.cs:36-49, [H] Core/World/BspBlockmapBuilder.cs:43-115; denormalised per-block line arrays [H] Core/World/Blockmap/BlockLine.cs:8-17, [H] Core/World/Blockmap/BlockMap.cs:33-37, 376-393; the allocation idea behind pooled entities [H] Core/Util/DataCache.cs:96-132. Leave: the inverted (no-op) closet wake-up loop [H] Core/World/WorldBase.cs:4202-4219; approximate closet behaviour (16-tic polls, no sight, no attacks, 40-unit steps [H] Core/World/Entities/Enemy.cs:327-342); approximate slow tick (tics × (multiplier + offset), offset −1..+2, off by default [H] Core/World/Entities/Definition/States/FrameState.cs:226-261, [H] Core/Util/Configs/Components/ConfigSlowTick.cs:12-34) that reads renderer visibility and keeps stale flags after Dehacked [H] Core/Dehacked/DehackedApplier.cs:460-482; and Helion's entity-list iteration, which skips the rest of the list when an entity destroys its successor [H] Core/World/WorldBase.cs:1173-1206, [H] Core/World/Entities/EntityManager.cs:133-151, [H] Core/World/Entities/Entity.cs:1304-1323. UZDoom's `NextToThink` advance in `DThinker::Remove` is correct [U] src/playsim/dthinker.cpp:956-961; every new iterator below must preserve it.

**UZDoom code to change (verified).** `RunThinkers`/`TickThinkers` [U] src/playsim/dthinker.cpp:93-122, 812-850 and `CallTick` (1265-1271); three full actor passes per tic (interpolation reset [U] src/p_tick.cpp:241-252, ticking, recreate-lights [U] src/playsim/dthinker.cpp:250-265); `AActor::Tick`'s always-run tail [U] src/playsim/p_mobj.cpp:5076-5122; `CallAction` [U] src/gamedata/info.cpp:148-234; `PointInSubsector` walking from `HeadGamenode()` in fixed point [U] src/playsim/p_maputl.cpp:1946-1962; blockmap line lists [U] src/gamedata/p_blockmap.h:87-93; `CollectConnectedGroups` scanning all linked portals [U] src/playsim/portal.cpp:1049-1099; `CreateNew` mallocing each object [U] src/common/objects/dobjtype.cpp:448-474; no dormancy anywhere (U3 §6).

**Implementation steps.**

1. **Native fast paths** (extends F4).
   - (a) `CallTick`: the stock-native shortcut (F4).
   - (b) `CallAction`: when `ActionFunc` has a direct-native pointer (registered with `DEFINE_ACTION_FUNCTION_NATIVE`; the JIT already uses this [U] src/common/scripting/jit/jit_call.cpp:65-128), call it through the platform ABI with the default arguments pre-unpacked once per function. That skips the `VMValue` array build, the `actionParams` push and the `try` frame [U] src/gamedata/info.cpp:177-200. A C++ caller cannot call an arbitrary native signature generically: generate one small thunk per signature at load (with the JIT's native-call emitter when the JIT is on), or cover the common signatures with templates and leave the rest on the VM path.
   - (c) Optionally native-ize the engine-side `Inventory.Tick` body [U] wadsrc/static/zscript/actors/inventory/inventory.zs:115-149, keeping it `virtual` so mod overrides still win. Every pickup on the map runs it every tic.
2. **Dormancy, exact by construction** (*beyond Helion*: a generalized, demo-safe closet).

   An actor is **dormant-eligible** when its class has the stock `Tick` (F4's check), it has no `Behaviors`, zero velocity with `Z == floorz` and no float bob, its sector has no damage, terrain, `SECF_KILLMONSTERS` or scrolling, it carries no inventory (carried items get a VM `DoEffect` call per tic), it has no poison, `MF8_RECREATELIGHTS`, `effects` or `fountaincolor`, and its current state is `tics == -1` or a loop whose only actions are `A_Look`/`A_LookEx` (checked per class at load, like Helion's spawn-loop check [H] Core/World/Entities/Definition/Composer/DefinitionStateApplier.cs:123-140). The line-by-line read of `AActor::Tick` [U] src/playsim/p_mobj.cpp:4504-5122 adds more exclusions: `freezetics`, a morph `alternative`, model data (`CalcBones` runs per tic, lines 4550 and 5121), `MF_STEALTH`, `MF4_VFRICTION`, a non-null `BlockingMobj`, `MF_SOLID` on a steep slope, `MF6_TOUCHY`/`MF2_BLASTED`, `MF8_INSCROLLSEC`, and bots in the level. It also excludes **monsters in a `tics == -1` or `CanRaise` state on a respawn skill, or with `MF5_ALWAYSRESPAWN`**: the nightmare-respawn block increments `movecount` and draws `pr_nightmarerespawn` every tic [U] src/playsim/p_mobj.cpp:5124-5156. Under these conditions the always-run tail [U] src/playsim/p_mobj.cpp:5076-5156 does nothing observable except decrement `tics` and call the look action on transitions (`UpdateRenderSectorList` there is render-only).

   Implementation: dormant actors stay in their thinker list, so iteration and RNG order are unchanged. In `TickThinkers` (dthinker.cpp:812-850, and the same in `ProfileThinkers`), the fast path applies only when the actor is not frozen; a frozen actor's stock `Tick` returns without decrementing `tics`, so it takes the full path. On the fast path, a node with `tics == -1` is skipped, and one with `tics > 1` just gets `--tics` inline (fields stay exact for ZScript readers). Otherwise the full `CallTick` runs, which decrements, transitions and calls `A_Look` exactly as before. **Inference:** about 20–40 ns per dormant actor against 100+ ns for `CallTick` plus the tail.

   **Re-check the predicate natively every tic; do not rely on wake hooks.** ZScript field writes (`Vel`, `bFlags`, `Line.flags`, sector fields) are raw VM stores with no setter: `DEFINE_FIELD` only registers an offset. A hook list can never be complete. The per-class part (stock `Tick`, look-loop states) is cached at load. The per-actor part (velocity, `Z == floorz`, the flag words, `Inventory`, `Behaviors`, `freezetics`, sector damage and flags, `state`/`tics`) is about 3 cache lines of loads per actor, so it is cheap, and it makes dormancy exact by construction. Start with `tics == -1` decorations plus look-loop monsters and widen.

   **Closets fall out for free:** a monster in a closet runs an exact `A_Look` once per look period, and its sight check fails cheaply (REJECT, then the first one-sided wall found). Optional `sv_closetlook 1` (off, recorded in demos and savegames) skips `P_LookForPlayers`' sight path but keeps its RNG draws. It is exact only when sight is impossible: no player in the monster's `mapsection` (connectivity through two-sided segs, [U] src/maploader/renderinfo.cpp:77-102) **and** no line or sector portal linking that section to a player's section, because `P_CheckSight` crosses portals. Enable it by default only if the demo suite stays bit-exact.
3. **Tick LOD: Helion's slow tick, generalised and made deterministic.**
   - Helion multiplies the tic count of chase, look and tracer frames by (multiplier + offset), and it reads renderer visibility. That is off by default and not demo-safe [H] Core/World/Entities/Definition/States/FrameState.cs:226-261.
   - The engine version (`sv_ticklod`, off by default, a `CVAR_SERVERINFO` cvar so every netgame peer uses the host's value, recorded in demo and savegame headers):
     - (a) **Criterion: distance only.** Use the minimum distance to any player's body plus "not targeting or targeted by a player within R". Both are sim state, so the result is deterministic and independent of the headset FOV. Never use renderer visibility.
     - (b) **Stock AI.** Helion-style frame-duration scaling on `A_Chase`/`A_Look`/tracer states, with step-size compensation, applied only to classes with the stock `Tick`. Recompute each state's action class after DEHACKED and ZScript load.
     - (c) **Script-heavy classes.** A class property `TickLOD "near, far, maxInterval"` lets a mod declare that its actor may skip its *whole* `Tick` on k−1 of k tics beyond `far`, staggered by the actor's thinker-order index against `Level->maptime` (saved with the level, unlike wall-clock or session counters), so the work spreads evenly across tics. Skipped tics accumulate `tics` decrements, applied when the actor next ticks.
     - (d) **Opt-out.** A new flag `+NOTICKLOD`, plus an automatic exemption for actors with `MF_SPECIAL`, players, bosses, `MF8_` "always-tick"-style flags, and anything with a scripted special pending.
     - (e) **A per-tic budget watchdog** (optional). If `Sim.RunThinkers` exceeds `sim_tick_budget_ms`, raise the LOD distance thresholds by one step next tic and lower them again when back under budget. It is driven by measured wall time, so it is **non-deterministic**. Allow it only in single-player non-demo sessions, as a VR comfort mode.
4. **BSP point-lookup grid.**
   - Build after nodes load ([U] src/maploader/maploader.cpp:3160-3161) over **gamenodes**. Each cell stores the deepest node whose ancestors' splitters do not cut the cell, or a subsector directly.
   - The cell classification must use the *same fixed-point `R_PointOnSide`* as `PointInSubsector` [U] src/rendering/r_utility.h:126-129: `DMulScale(...) > 0` thresholds a linear function computed exactly in 64 bits, so its extremes over a box lie at the corners. Test the cell's four **extreme fixed-point lattice corners** (`x0<<16` and `((x0+16)<<16) − 1`, same for y). The cell counts as uncut only if all four give the same side. Then a lookup from the cell's start node returns exactly what the root walk returns.
   - Derive the cell index from the fixed-point coordinates the walk uses (`FloatToFixed(x) >> 20` for 16-unit cells), never from the double. Otherwise rounding in `FloatToFixed` can put a point outside the corner range its cell was tested with.
   - Bounds-check x and y separately; Helion checks only the flat index [H] Core/World/WorldBase.BspBlockMap.cs:36-49.
   - Build a second grid for render nodes (`PointInRenderSubsector`).
   - Memory: use 16-unit cells up to 16k×16k maps (4 MB) and 32-unit cells above.
5. **Blockmap with inline line data.**
   - A per-block contiguous array of `{lineIndex, v1, delta, bbox}` (~48 B) **in blockmap-lump order**, not sorted. `FBlockLinesIterator` returns polyobject links first, then the lump list, deduplicated by `validcount` [U] src/playsim/p_maputl.cpp:622-667. That order decides which line wins in `PIT_CheckLine` and the spechit order, so it must stay identical.
   - Inline only immutable geometry. `Line.flags` is a writable ZScript field (raw store, wadsrc/static/zscript/mapdata.zs:232), so flags and sectors are read from `line_t` after the geometric reject passes. Polyobject lines keep their existing dynamic `polyLink` path.
   - `FBlockLinesIterator` returns entries. `PIT_CheckLine`'s early box and side reject [U] src/playsim/p_map.cpp:824-829 runs on the inline data before touching `line_t`.
6. **Sight caching (within-tic, exact).**
   - Key: `(t1, t2, t1->Pos, t2->Pos, t1 and t2 heights, flags, Level->geometryEpoch)`, where `geometryEpoch` is bumped by any plane move, native line-flag change or polyobject move.
   - Cache only the deterministic trace part of `P_CheckSight` [U] src/playsim/p_sight.cpp:840-950: the portal-aware traverse from line 914 on. Always re-run the early part, including the random invisibility reject (lines 871-882), so the RNG sequence stays identical.
   - Clear the cache every tic.
   - **Open risk:** a ZScript write to `Line.flags` (for example `ML_BLOCKSIGHT`) between two identical checks in one tic cannot bump the epoch. Rare, but not provably exact: gate on the demo suite.
   - Add the cache only if a counter shows repeated `(t1, t2)` pairs within a tic (**Inference**: `A_Chase` paths re-check the same target).
7. **Portal link acceleration.** Precompute per blockmap cell the list of linked portals within reach, extending UZDoom's existing `PortalBlockmap` with its `containsLines`/`neighborContainsLines` flags [U] src/playsim/p_secnodes.cpp:381-388. `CollectConnectedGroups` then iterates only those portals instead of all `linkedPortals` (portal.cpp:1070-1081).
8. **Actor arenas and layout.**
   - `PClass::CreateNew` [U] src/common/objects/dobjtype.cpp:448-474 allocates from per-size-class slabs (actor classes cluster into a few sizes). GC sweep returns objects to their slab, replacing the `M_Free` in `SweepObjects` [U] src/common/objects/dobjgc.cpp:234-235. Addresses already vary run to run under ASLR, so no sim code can depend on them; slabs do not affect determinism.
   - Reorder native `AActor` fields so the hot ones share 2–3 cache lines: `Pos`, `Vel`, flags, `tics`/`state`, `Sector`/`floorz`, `Prev*`, `target`. Today they are spread from actor.h:1142 to 1401 (U3 §8). `DEFINE_FIELD` uses offsets, savegames serialise by name, and script fields follow the native size, so reordering is ABI-safe for ZScript. Verify with the savegame round-trip test.
9. **The interpolation-reset pass: 9.0 ms per tic measured [M] §2. Make it cheap and exact, then lazy.**
   - *Today.* A separate full walk over every actor, `GetThinkerIterator<AActor>()`, before any thinking [U] src/p_tick.cpp:241-252. The iterator does an `IsKindOf` per thinker [U] src/playsim/dthinker.cpp:1364-1410. With the default statnum it covers only `STAT_FIRST_THINKING..MAX_STATNUM` (32–127) [U] src/playsim/dthinker.cpp:1298-1313, so **parked actors (statnum < 32) are never reset**. Phase 10 relies on this. `ClearInterpolation` copies Pos, Angles, Scale and Alpha into `Prev*`, and it loads `Sector->PortalGroup` [U] src/playsim/actorinlines.h:46-54. `Prev` sits near the end of a ~1.6 KB object, far from `__Pos` (U3 §8).
   - *Why it is a separate pass.* So that an actor moved by *another* thinker before its own tick still interpolates from its start-of-tic position (the comment at p_tick.cpp:241).
   - *Helion folds the copy into the tick walk:* `entity.PrevPosition = entity.Position` right before `Move` [H] Core/World/WorldBase.cs:1167-1181. Pushes by entities earlier in the list then snap instead of interpolating. Do not copy that literally.
   - **Step A (exact, cheap):**
     - iterate a dense `TArray<AActor*>` maintained on spawn, destroy and `ChangeStatNum`, holding exactly the actors in statnums ≥ 32 (the set the iterator visits today), instead of `FThinkerIterator` (no list chase, no `IsKindOf`). Reset order does not matter, because each reset touches only its own actor;
     - co-locate `Angles/PrevAngles`, `Scale/PrevScale`, `Alpha/PrevAlpha` and `__Pos/Prev` (field reorder, step 8) so a reset touches three cache lines instead of five or more (about 150 B);
     - cache `PortalGroup` in the actor at `LinkToWorld` so the dependent `Sector->` load disappears;
     - prefetch 8 actors ahead.

     **Inference:** 3–4× faster, with no semantic change.
   - **Step B (lazy, exact for position):**
     - `Pos` is `readonly` to ZScript [U] wadsrc/static/zscript/actors/actor.zs:193, and every native write goes through five inline setters (`SetZ`, `AddZ`, `SetXY`, two `SetXYZ`) [U] src/playsim/actor.h:1606-1628, plus savegame load.
     - Give those setters a copy-on-first-write: `if (PrevStamp != Level->ticStamp) { Prev = Pos(); PrevPortalGroup = …; PrevStamp = Level->ticStamp; }`.
     - Keep a `moved[N]` list of actors that took that branch. At the start of tic N+1, reset only the actors in `moved[N]`. Everyone else already has `Prev == Pos`, by induction.
     - The induction also breaks wherever `Prev` is set to something other than `Pos`, so those actors must enter `moved[]` too:
       - the native `Prev` writes [U] src/playsim/p_things.cpp:785, 793 (`A_Warp` interpolation), [U] src/playsim/p_user.cpp:1879 (prediction), [U] src/playsim/p_mobj.cpp:4241, 4258, [U] src/playsim/p_map.cpp:2605, 2790;
       - spawn, a `ChangeStatNum` from below 32 into the thinking range, and savegame load (do a full reset after load);
       - **ZScript writes to `Prev`**: `Prev` is a writable field (actor.zs:194), and the store is a raw VM store that cannot be hooked at run time. Detect it at compile time instead: when the code generator resolves a store to `Actor.Prev` anywhere in the load order, set a global flag and run Step A only (the full exact pass) for that session. Mods that cancel interpolation therefore still work, at Step A cost.
     - Angles, Scale and Alpha are freely writable from ZScript (for example `native double Angle`, actor.zs:204) and cannot be hooked. They keep a full pre-tic pass, but over the co-located block. Folding it into `RunThinkers` right before each thinker's own tick is **not** exact: an angle set on that actor by an earlier thinker would snap, which is Helion's flaw again.
   - **Step C (Phase 8):** the renderer takes prev from the *previous snapshot*, so render interpolation no longer depends on `Prev*` at all. The fields remain only for ZScript readers.
   - Also move the recreate-lights full pass [U] src/playsim/dthinker.cpp:250-265 onto the dense array, with `flags8`, `effects` and `fountaincolor` co-located in the hot block. The pass also runs `P_RunEffect` (dthinker.cpp:211-224), and ZScript can set those flag bits and fields with raw stores, so a natively maintained "flagged list" would miss some actors.
10. **GC pacing.** On the sim thread (Phase 8), call `Step` with a deadline, and tune `gc stepmul`/`pause` per scenario [U] src/common/objects/dobjgc.cpp:44-76, 811-851. Slab allocation (step 8) also lowers the per-object free cost.

**ZScript/mod compatibility.**
- 1, 4, 5, 7, 8 and 9 have no observable change, and 6 has none barring its open risk. Use the demo suite (P3 §5) as the gate.
- 2 is exact by construction, because the predicate is re-checked every tic. Its remaining risk is an incomplete predicate, a tail path the line-by-line read missed. Ship `sv_dormancy` with a debug mode (`sv_dormancy_verify`, P3 §5.1) that shadow-ticks dormant actors and asserts that nothing changed.
- 9 Step B falls back to Step A when any loaded script stores to `Actor.Prev`.
- 3 is opt-in and approximate.

**Risks.**
- The completeness of the dormancy predicate.
- Slab allocation interacting with DObject destruction order. `OF_EuthanizeMe` objects are freed in sweep, so slabs free at sweep too.
- Field reordering interacting with hard-coded offsets in the JIT. The JIT uses `DEFINE_FIELD` offsets, so it is safe, but test it.

**Success metric (P3 §7 holds the numbers).**
- Interpolation reset (step 9 A + B): 9.0 → ≤ 2.0 ms per tic with the owner's mods, 1.3 → ≤ 0.3 ms without.
- `Sim.RunThinkers` −40% on the no-mod slaughter scenario (dormancy, fast paths, grid).
- `sight_checks` time −50% (cache plus F5).
- Allocations per tic −90% (F1, slabs, and Phase 10's `TickBehaviors` fix).
- Demo suite bit-exact with all default-on items enabled.

**Effort (Inference).** Fast paths 2, dormancy predicate + audit 6, tick LOD (b)–(e) 3, BSP grid 2, inline blockmap 3, sight cache 1, portal cells 1, arenas + layout 3, pass folding incl. the `Prev`-store detection 2, GC 1: **≈ 24 person-weeks.**

---

## Phase 10 — Helper-actor explosion: engine mechanisms that let mods stop spawning thinkers

**Goal.** With the owner's mod set, the thinker count rises from 8,065 to 53,116 [M] §1. Most of the extra thinkers are *per-monster helper actors*: HP bars, champion icons, controllers, aggro trackers and leaves, crit controllers and brains, shifters. Each one pays the full per-thinker engine overhead (`CallTick` VM entry, the interpolation reset, the three walks, `AActor::Tick`'s tail, blockmap and sector links), plus its script. This phase gives mods cheaper ways to express the same features, and it rewrites the owner's own mod against them.

- **Inference:** removing about 40k helpers from Sunder MAP15 is worth more than every other sim optimisation combined.
- Helion offers no evidence here (no ZScript). Its closet idea, a cheaper representation chosen at load, is the spirit.

**UZDoom building blocks that already exist (verified).**
- **Behaviors.** These are per-actor component objects with `Owner`, `Initialize` and a virtual `Tick` [U] wadsrc/static/zscript/actors/actor.zs:101-110. They are managed by `AddBehavior`/`FindBehavior`/`RemoveBehavior` (lines 608-612). The owner's native `AActor::Tick` calls `TickBehaviors` every tic unless frozen [U] src/playsim/p_mobj.cpp:4565-4566, and that calls a behavior's `Tick` only when the behavior overrides it (lines 564-603). Behaviors therefore only run while their owner ticks.
  - They are not in the thinker list or the blockmap.
  - They need no interpolation reset and no rendering.
  - Defect: `TickBehaviors` builds two local `TArray`s per call. `toTick` gets one `Push` per behavior, so there is a heap allocation per actor per tic whenever behaviors exist, even data-only ones (lines 566-567, 580). Fix it with stack arrays that have inline capacity (for example 8 entries). Do not use one thread-local scratch array: `TickBehaviors` is script-callable (actor.zs:611) and can re-enter.
- **Parking.** Statnums below `STAT_FIRST_THINKING` (32) are never ticked [U] src/playsim/statnums.h:34-42. `ChangeStatNum` lets scripts move any thinker there, except `STAT_STATIC` for actors and `STAT_TRAVELLING` for any thinker [U] src/playsim/dthinker.cpp:1054-1076. Park in 6–31, which the engine never walks. Avoid 0–3: for example `STAT_AUTODECAL` is trimmed by destroying its oldest entries [U] src/playsim/a_decals.cpp:663. A parked actor still renders and collides (U3 §6). Three side effects follow from the default iterator range (32–127, [U] src/playsim/dthinker.cpp:1298-1313):
  - it skips the interpolation reset, so whoever moves it must set `Prev`;
  - it skips the recreate-lights pass;
  - default `ThinkerIterator.Create` calls no longer find it; a mod must pass the statnum.
- **Event handlers.** Handlers get `WorldThingSpawned`/`Died`/`Damaged`/`Destroyed` [U] src/events.h:326-331 and `WorldTick` [U] src/events.cpp:2068-2076. `WorldTick` runs **before** `RunThinkers` [U] src/p_tick.cpp:261-263, so it sees positions from the end of the previous tic.
- **Slow iteration.** `ThinkerIterator.Create`/`Reinit` [U] wadsrc/static/zscript/doombase.zs:291-296 walk *every* thinker with an `IsKindOf` each [U] src/playsim/dthinker.cpp:1364-1410, and each `Create` allocates a DObject.

**Mechanisms to add (engine side; ZScript API sketches are marked *(sketch)*).**

1. **Native per-actor overlays** (HP bars, icons, markers) rendered without any actor. *(sketch)*

   Binding pattern: copy `FLineTraceData`. Mods can declare locals of it because it is a plain (non-`native`) struct whose fields are `native` [U] wadsrc/static/zscript/actors/actor.zs:61-87. The engine pre-registers its size with `NewStruct` [U] src/scripting/thingdef_data.cpp:846-848 and binds each field with `DEFINE_FIELD_X` [U] src/scripting/vmthunks_actors.cpp:2183ff. Do **not** use `struct … native`: a mod cannot instantiate a native struct [U] src/common/scripting/frontend/zcc_compile.cpp:2292-2295. A struct parameter is passed by pointer [U] src/common/scripting/frontend/zcc_compile.cpp:2670-2674, so the native side reads it with `PARAM_POINTER(ov, FActorOverlay)` and copies it.
   `version("5.1")` stands for the first engine version that ships the API (the stock base is `version "5.0.0"`, wadsrc/static/zscript.txt:1).
   ```zscript
   struct ActorOverlay version("5.1")    // C++ FActorOverlay (POD); NewStruct("ActorOverlay") with its sizeof/alignof
   {
       native TextureID Tex, FillTex;    // frame + fill texture (FillTex optional)
       native Vector2 Size;              // world units
       native double ZOffset;            // above Height (or below if negative)
       native Color Tint, FillTint;
       native double Fill;               // 0..1 when not bound by flags
       native int FillMax;               // OVF_FILL_HEALTH denominator; 0 = SpawnHealth() (mods that buff health set it)
       native int Flags;                 // OVF_FILL_HEALTH (fill = health/FillMax, read natively at snapshot time),
                                         // OVF_HIDE_FULL, OVF_HIDE_DEAD, OVF_FACE_VIEWER, OVF_FULLBRIGHT, OVF_DEPTHTEST_OFF
       native int ViewerMask;            // bit per player; 0 = everyone (filtered at render time by consoleplayer)
       native double MaxDist;            // culled beyond this distance from the viewer
   }
   extend class Actor
   {
       native version("5.1") void SetOverlay(int slot, ActorOverlay ov);   // PARAM_POINTER → copy into the slot
       native version("5.1") void ClearOverlay(int slot);
       native version("5.1") void SetOverlayFill(int slot, double f);
   }
   ```
   - Storage: a lazily allocated slot block (max 4 slots) behind one pointer in `AActor`. Do not put inline slots in the hot object: 4 × ~64 B on all 53k actors would bloat every walk.
   - Overlays are play state, set from play scope. They serialize with the actor (`AActor::Serialize`, key `"overlays"`), so a loaded savegame keeps them, because `WorldThingSpawned` does not re-run on load. They never affect the sim, so they are demo- and net-safe.
   - The renderer emits overlay instances from the actor's snapshot record (Phase 7 instance path, 2 instances per bar). The cost is zero per tic and one instance per visible bar per frame.
   - **Before Phase 7 (needed for the recommended order):** emit them from the existing sprite path. Where `RenderThings` processes each actor [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:566-604, add one `HWSprite` per active slot: a billboard quad, plus a fill quad clipped by UV and width. That costs one sorted translucent sprite per visible bar per frame, the same as today's bar actor, and nothing per tic. **Inference:** about 1 person-week.
   - Health binding is native, so damage needs no script.
   - This replaces 3,622 `RSQTHP_Bar` thinkers (16.5 ms per tic [M] §3) with nothing.
   - In VR this is also the only correct way to draw world-anchored bars: `RenderOverlay` draws into a head-locked HUD layer (Phase 8 step 5).
2. **Attachments: actors that follow a master without ticking.**
   - Add `Actor.AttachTo(Actor master, Vector3 ofs, int flags)` *(sketch)*.
   - The engine repositions attached actors in one native pass right after `RunThinkers` [U] src/p_tick.cpp:263, in attachment-creation order. For each attachment whose master moved this tic, it does one `SetXYZ` plus a relink, no VM. Every attachment gets `Prev = master->Prev + ofs` each tic, so it interpolates in lockstep with its master: parked actors are skipped by the engine's reset. Relinking from inside the master's `LinkToWorld` is not recommended: that runs in the middle of `P_TryMove` and re-enters the link code.
   - Mid-tic readers see the attachment's position from the previous tic, the same as a helper that ticks before its master today.
   - The engine destroys or detaches attachments when the master dies or is destroyed (`ATF_DIE_WITH_MASTER`).
   - Attached actors are **parked** (a statnum in 6–31): they never tick unless they ask to.
   - The attachment record (master, offset, flags) serializes with the actor.
   - This covers champion icons and leaves, and any "`A_Warp(AAPTR_MASTER)` every tic" helper.
   - Render-only attachments (`ATF_NOWORLD`) are not linked into the blockmap or sectors at all. The current renderer finds sprites only through sector thing lists (`RenderThings`, hw_bsp.cpp:566ff), so these need the renderer to emit them from the master: the Phase 7 snapshot record, or the same `RenderThings` hook as the interim overlay path.
3. **Tickless actors with event wake-up.**
   - Add a flag `+TICKLESS` and the methods `SetTickless(bool)` / `WakeFor(int tics)` *(sketch)*.
   - A tickless actor lives in `STAT_TICKLESS` (a new statnum in 6–31). It is moved back to its thinking statnum for `n` tics by `WakeFor`, by `SetState`/`SetStateLabel` with a state whose tics are not −1, by damage, or by `Thing_*` specials. These are native entry points; raw field writes (`tics = …`) do not wake it, so document that. It drops back once it reaches a `tics == -1` state or the wake window ends.
   - Every move between lists appends at the tail, which changes thinker order. That is deterministic, but it differs from the unported mod, so old demos of that mod will not replay.
   - The engine dormancy of Phase 9 step 2 handles the stock case automatically. `+TICKLESS` is the mod-declared, broader version.
4. **Throttled statnums.**
   - Add `Level.SetStatTickInterval(int statnum, int interval)` *(sketch)*, valid for `STAT_USER..STAT_USER_MAX` [U] src/playsim/statnums.h:55-56.
   - A thinker in such a list ticks when `(Level->maptime + orderIndex) % interval == 0`, where `orderIndex` is its position in the walk, so the work is spread evenly. Use `maptime`, not `gametic`: it is saved with the level. Thinkers still in the fresh list tick on their first tic.
   - It is deterministic, because it depends only on list order and saved state, and therefore demo-safe. Serialize the per-statnum intervals with `FLevelLocals`.
   - Mods move periodic helpers (aggro scans, AI directors, ambience emitters such as `IA_Sound`: 2,544 calls [M] §3) into a user statnum and pick an interval.
5. **Per-class actor lists and batch handlers.**
   - Add `Level.GetActors(class<Actor> cls, out Array<Actor> result)` and `ActorList` iterators *(sketch)*, backed by native per-exact-class intrusive lists maintained at spawn and destroy. A query for `cls` walks `cls` and its descendants.
   - Order must be deterministic and survive a save: keep a serialized per-actor spawn serial, append on spawn, and sort by serial after load. Do not rebuild the lists from thinker order, which `ChangeStatNum` permutes.
   - That removes "iterate 53k thinkers with `IsKindOf`" from mod handlers. One `WorldTick` handler looping over a compact array of N controllers replaces N controller thinkers: one VM frame plus a loop, instead of N `CallTick`s, N resets and N `AActor::Tick` tails.
   - The 122 `champion_CyanController`s cost 7.16 ms, about **59 µs each** [M] §3. That is script body, not per-thinker overhead (≈ 0.3 µs). Merging them into a handler saves little by itself. First use `vm_profile` to find what the body does, typically a `ThinkerIterator` or radius scan per tic. Then replace that scan with per-class lists or a `BlockThingsIterator`, or throttle it (mechanism 4).
   - Add a non-allocating iterator variant (opt-in, for example `ThinkerIterator.CreateTransient`, documented as invalidated by the next call of the same kind). Do **not** make `Create` return a pooled instance: scripts nest two iterators of the same type (pair loops), and a shared cursor would silently break them.
6. **Damage-reactive helpers become events.**
   - "Two thinkers per actor" crit systems (`csh_critController` plus `csh_Brain`: 11,332 calls [M] §3) usually only react to damage.
   - `WorldThingDamaged` (events.h:330), combined with per-actor state kept in a data-only `Behavior` (no `Tick` override, so `TickBehaviors` makes no VM call for it), removes both thinkers. Until the `TickBehaviors` fix above lands, each owner with a behavior still pays one heap allocation per tic. That is still far cheaper than two thinkers.
   - These helpers cost only 0.14 µs per call [M] §3, so the gain is mostly their share of the per-actor walks. 11,332 of 53,116 actors is about 21% of the 9 ms interpolation reset.
7. **Shifter-style "one helper per N actors"** (7,686 calls [M] §3): the same pattern, a `Behavior` or `ActorOverlay` plus a handler.
8. **Script allocation hygiene**, supported by the 0.5 allocation counter:
   - per-tic `new` of arrays or objects in `Tick`;
   - string formatting for names or CVars every tic;
   - `ThinkerIterator.Create` per tic.

   Each one is visible per class in `sim_profile`.
9. **Monster replacements at 8× stock cost** (Legion of Bones: 2–3 µs against 0.3 µs [M] §3). Use `vm_profile` per-virtual zones to find which overrides cost (typically a `Tick` override that calls `Super.Tick()` plus extra traces). Where the work is purely cosmetic (smoke, glow), move it to `TickLOD` (Phase 9 step 3c; opt-in and approximate) or to client-side actors (`SpawnClientSide`, actor.zs:876) ticked in the client-side thinker lists (`P_RunClientSideLogic` [U] src/p_tick.cpp:54-99), which are off the deterministic path.

**How to rewrite `RSQTHP_Bar` (the owner's mod) against the new API.**

*Step 0: find out what the bar actor does.* The measurement shows only one 4.5 µs tick per bar [M] §3. That is about 15× a stock monster's whole tick, so it is script work: a warp, health reads, maybe a trace or string building. Whether the bar draws as its own sprite or only feeds a `RenderOverlay` drawer is not recorded: the owner's survey leaves it open, and the mod source is not in the surveyed repos (rs_enginesurvey/UZDXREMA_0.1.0/04_SPRITES_MODELS_PARTICLES_WEAPONS.md §5, "Mod-driven costs"). Read its `Tick` and states first, or run `vm_profile`.
- If the actor only tracks a monster for a HUD or `RenderOverlay` drawer, delete it. Keep an `Array<Actor>` of tracked monsters in the handler instead; nothing per monster ticks. In VR, move the drawing to overlays anyway (mechanism 1).
- If the bar draws as its own sprite, use the overlay rewrite below, or the stock-5.0 transition until the API exists.

*Overlay rewrite (needs mechanism 1; renders through the interim sprite path until Phase 7):*
- Delete the bar actor class and add one `EventHandler` (MAPINFO `EventHandlers`). It holds no state, because overlays serialize with the actor.
- In `WorldThingSpawned(e)`: `if (e.Thing.bIsMonster && !e.Thing.bFriendly) { ActorOverlay bar; bar.Tex = …; bar.FillTex = …; bar.Size = (32, 4); bar.ZOffset = 8; bar.Flags = OVF_FILL_HEALTH | OVF_HIDE_FULL | OVF_HIDE_DEAD | OVF_FACE_VIEWER; bar.MaxDist = 2048; e.Thing.SetOverlay(OVS_HPBAR, bar); }`. Leave `FillMax` at 0 unless a mod in the load order (LegenDoom, champions) raises monster health above spawn health. In that case set it where the health is raised, or bind it to a "max health seen" field.
- No `WorldTick` and no per-monster thinker. The bar follows the monster and reads its health natively.
- A "top" variant that shows only the crosshair target: in `WorldTick`, compute `players[i].mo.AimTarget()` for **every** player in game, and set or clear one overlay per target with `ViewerMask = 1 << i`. Never branch on `consoleplayer` in play scope; that differs per peer and desyncs netgames. The renderer filters by `consoleplayer`.

*Transitional rewrite on stock 5.0 (no engine change):*
1. Park each bar: `ChangeStatNum(20)`, a free number in 6–31. It never ticks, and it drops out of the engine's interpolation reset and recreate-lights walks.
2. Keep `(bar, monster, lastPos, lastHealth)` records in an `EventHandler` (not a `StaticEventHandler`, so the records are saved with the game), filled in `WorldThingSpawned`.
3. Do the update **after** the monsters move, not in `WorldTick`: `WorldTick` runs before `RunThinkers` [U] src/p_tick.cpp:261-263, so bars placed there trail by one tic. Create one updater `Thinker` and `ChangeStatNum(Thinker.MAX_STATNUM)` (127). Statnums tick in ascending order [U] src/playsim/dthinker.cpp:106-120, so it runs after every monster (`STAT_DEFAULT` = 100). In its `Tick`, for each record:
   - if `monster == null` (object loads pass through `GC::ReadBarrier`, so destroyed objects read as null [U] src/common/scripting/vm/vmexec.h:238-241), destroy the bar and swap-remove the record;
   - if `monster.Pos != lastPos`, call `bar.SetOrigin(monster.Pos + (0, 0, monster.Height + gap), true)`;
   - **always** set `bar.Prev = monster.Prev + (0, 0, monster.Height + gap)`. A parked bar is never reset, so a `Prev` left stale would make it interpolate from an old position on every frame;
   - if `monster.health != lastHealth`, update the bar's frame or scale.

That is one VM loop iteration per bar plus a native `SetOrigin` per *moved* bar, instead of a full thinker tick per bar per tic. **Inference:** about 5–10× cheaper on 3,622 bars, more when most monsters idle. The bars also leave the 9 ms reset walk. Bit-exactness against the old mod is not expected; it is a different mod. It is deterministic, so demos recorded with the new version replay.

**ZScript/mod compatibility.**
- Everything here is additive and opt-in. Existing mods run unchanged, at today's cost.
- Throttled statnums, attachments, tickless actors and overlays are deterministic, so they are demo- and net-safe as long as the mod is part of the demo's load order. Each new piece of state (overlay slots, attachment records, stat intervals, spawn serials) must serialize; P3 §5.4's savegame round-trip gates them.
- Ported mods that park or attach actors must pass the statnum to `ThinkerIterator.Create` to find them (the default range starts at 32).

**Risks.**
- API design churn. Prototype it against the owner's three heaviest mod families first (RSQ HP bar, mk-champions, crits), then freeze it.
- Native per-class lists must handle class replacement and actors that travel between levels (`STAT_TRAVELLING`).
- The interim overlay renderer and `ATF_NOWORLD` add a renderer dependency to a sim-track phase. Keep both behind a cvar until Phase 7 replaces them.

**Success metric** (Sunder MAP15 with the owner's mod set; P3 §7 holds the numbers).
- The owner's own port: `RSQTHP_Bar` 16.5 → 0 ms and 3,622 fewer thinkers. The transitional port alone should take most of the 16.5 ms.
- With the mk-champions and crit patterns also ported (third-party mods: patches the owner maintains, or demonstrated on the generator maps): thinkers 53,116 → ≤ 20k, and `Sim.RunThinkers` ≤ 40 ms per tic from this phase alone (**Inference**).
- Combined with Phase 9: thinkers ≤ 20 ms per tic on Sunder MAP15 with the owner's mods (S2), with the whole tic at median ≤ 25 ms and p95 ≤ 28.6 ms, and ≥ 35 tics per second in real time; thinkers ≤ 12 ms on Abandon MAP13 (S3). These match the measurement plan's acceptance targets. Thinkers at 25 ms would not fit: the other per-tic work (interpolation reset, event handlers, particles) adds several ms on top.

**Effort (Inference).** Overlays 3 (instanced rendering reuses Phase 7), interim sprite-path rendering 1, attachments 2, tickless + wake 2, throttled statnums 1, per-class lists + transient iterators 2, Behavior fixes 0.5, serialization of the new state 0.5, owner mod ports (RSQ bar plus a template for third-party mods) 2: **≈ 14 person-weeks.**

---

## Dependency summary and recommended order

| Phase | Hard dependencies | Soft dependencies | Effort, person-weeks (Inference) |
|---|---|---|---|
| 0 Instrument, fixes, sim profiling | — | — | 8.5 |
| 1 Vulkan backend | 0 | — | 17 |
| 2 OpenXR + multiview | 1 (frame contexts, layered targets) | 0 (F10) | 14 |
| 3 Tic-rate scene + GPU interpolation | 1 (SSBOs), 2 (cyclops/combined frustum) | — | 14 |
| 4 Resident world | 1 (bindless, indirect), 3 (plane tables) | — | 24 |
| 5 GPU visibility + portals | 4 | 2 | 16 |
| 6 Lighting | 1 | 4, 3 | 11–15 |
| 7 Sprites / particles / models / overlays | 1, 3 | 6, 5 | 22.5 |
| 8 Sim thread | 3 (no render writes into sectors; `TicScene` built sim-side), 7 step 1 (no actor writes) | 9, 5 (its step 5 later feeds the step 1b delta list from the GPU) | 17 |
| 9 Playsim engine overhead | 0 (0.5 tooling) | 8 (GC placement) | 24 |
| 10 Helper-actor explosion | 0 (0.5 tooling) | 7 (instanced overlays; until then the interim sprite path), 9 | 14 |

**Total about 185 person-weeks (Inference)**, about 11 months for a team of 4.

**Recommended order for the measured workload.** The owner's mod set spends about 80% of a 315 ms frame in the playsim and about 2% in the renderer [M] §1, so the **sim track leads**.

1. **Phase 0, with 0.5 and F12/F13 first.** That covers per-class and per-virtual profiling, the JIT check and the busy-wait. Then the sim fixes F1, F2, F4 and F5. These are days of work, and they measure and cut the dominant cost.
2. **Phase 9 steps 1, 2 and 9** (native fast paths, dormancy, the interpolation-reset pass, 9.0 ms measured), **and Phase 10** (overlays with the interim sprite-path renderer, attachments, tickless, throttled statnums, per-class lists), with the owner porting `RSQTHP_Bar` and pressing the mk-champions and crit patterns. The owner can ship the transitional stock-5.0 `RSQTHP_Bar` rewrite on day one, before any engine change. **Target:** Sunder MAP15 with mods under the 28.6 ms tic. Until then the game cannot run at full speed on any display.
3. **In parallel on the render track: Phase 1, then Phase 2.** The no-mod flat frame is already 13.9 ms, with 3.2 ms of GPU wait from zero frames in flight and 3.8 ms of scene build that stereo doubles. Phase 2 gives the first VR build at one scene build per frame.
4. **The rest of Phase 9** (grid, inline blockmap, sight cache, portal cells, arenas, GC pacing), driven by `sim_profile` captures.
5. **Phase 3 and Phase 7 step 1, then Phase 8.** Once tics fit their budget, the sim thread keeps a 15–25 ms tic from ever costing a headset frame. Phase 8 step 1b removes the remaining render→play accesses with a CPU-fed delta list, so Phase 8 does not wait for Phase 5. Phase 7's instance path then replaces the interim overlay renderer.
6. **Phase 4, then 5, then 6.** Resident world, GPU visibility and portals, lighting: GPU-scaling work for heavy maps and light-heavy mods.

On a no-mod or light-mod workload, swap steps 2 and 3: the render track first.

**Fallbacks to keep forever:** `vr_singlepass 0`, `r_gpucull 0`, `r_residentworld 0`, `r_oit 0/1`, `sv_dormancy 0`, `sv_ticklod 0`. Each one is a debugging lifeline and a pixel-diff or demo-diff reference.
