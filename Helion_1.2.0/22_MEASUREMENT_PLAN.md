# P3 — Measurement plan: scenarios, metrics, Tracy zones, acceptance targets

Every phase in P1 makes a performance claim, and this plan is how each claim gets accepted or rejected. It defines:
- benchmark scenarios by stress type, including the owner's two real workloads;
- the metrics and how to collect them, in flat-screen and headset runs;
- the Tracy zone and counter list;
- the regression gates (bit-exact sim, pixel diff, savegame round-trip);
- acceptance targets per phase, anchored to the owner's measured baselines.

Conventions:
- `[U]` = UZDoom 5.0.0 @ 292cf420; `[H]` = Helion @ 5edd437f.
- `[M] §n` = the owner's measurements on his 5.0-based fork (RTX 3080 Ti, i9-9900KF, flat screen), from `rs_enginesurvey/UZDXREMA_0.1.0/10_MEASUREMENTS.md`. Those runs are uncapped, in a 3,056 × 1,689 window, with the player standing still at the map start [M] (header); targets anchored to them only hold under the same conditions.
- Targets are **Inference** unless stated otherwise.

---

## 1. Principles

1. **Deterministic input.** Every scenario runs from a recorded demo, or from a scripted camera path for render-only runs. That way builds can be compared tic for tic.
   - UZDoom has `-timedemo` [U] src/d_main.cpp:246, the `timedemo` ccmd [U] src/g_game.cpp:2790, and `-nodraw`/`-noblit` [U] src/g_game.cpp:213-217. `-nodraw` is only read by `G_TimeDemo` [U] src/g_game.cpp:3037-3045 and makes `D_Display` return early [U] src/d_main.cpp:1119-1120. Use `-timedemo` with `-nodraw` for sim-only runs.
   - **Code fact: 5.0's `-timedemo` does not run as fast as possible.** `singletics` [U] src/d_main.cpp:518 is never set to true anywhere in 5.0, and `G_TimeDemo` only sets `nodrawers`, `noblit` and `timingdemo`. So tics stay paced by `I_GetTime` at 35 Hz (catching up when behind) [U] src/d_net.cpp:2206-2265, and the end-of-demo "fps" [U] src/g_game.cpp:3095-3097 cannot report more than real time.
   - **Harness change (P1 §0.3):** in `TryRunTics`, when `timingdemo` is set, force `doWait = false` (2200-2208, so `I_WaitForTic` cannot block) and `totalTics = 1` before `NetUpdate` (line 2221), and set `singletics = true` in `G_TimeDemo`. All three are needed.
     - Setting `singletics` alone only caps tics at one per iteration (line 2214). When no tic is due (`totalTics == 0`), nothing runs and the loop spins in `TicStabilityWait` (2273-2275).
     - Forcing `totalTics` alone leaves the catch-up `++runTics` (2260-2265) active, so some iterations run two tics.
     - With all three changes, exactly one tic runs per loop iteration, back to back, and a `-timedemo -nodraw` run measures true sim throughput. Sim determinism is unaffected.
   - A no-code alternative is `+i_timescale` set high [U] src/d_main.cpp:405-417. **Open risk:** it is unverified as a throughput mode, and it also scales the clock that the end-of-demo report uses, so time the run with the harness's own wall clock.
   - Fix `-rngseed`, as the owner's runs do with 1113 [M] (header).
2. **Fixed configuration per run.** These cvars are logged into the result file: `vid_scalefactor`, `gl_multisample`, `gl_ssao`, `gl_bloom`, `gl_tonemap`, `r_dynlights`, `gl_light_shadowmap`, `vk_raytrace`, `r_portal_recursions`, `gl_multithread`, `r_ticstability`, `vid_vsync`, `vid_maxfps`, `cl_capfps`, `vm_jit`/`vm_jit_aot`, plus every new P1 switch (`vr_singlepass`, `r_gpucull`, `r_residentworld`, `r_oit`, `sv_dormancy`, `sv_ticklod`).
3. **Warm-up, then steady state.**
   - Discard the first 10 s, which is texture upload and pipeline creation until P1 Phase 1 makes those asynchronous.
   - Report medians over the remaining run, as the owner does over "the last three quarters" [M] §1.
   - Report p95, p99 and max as well.
   - Count hitches separately: frames over 2× budget.
4. **One variable at a time.** Every fix has a kill switch (P1 §0.4), so A/B runs differ by exactly one cvar.
5. **CPU pinned and quiet.** Use a high-performance power plan, no background capture apart from Tracy, and Tracy sampling off for timing runs (instrumentation only). Do a second pass with sampling on for diagnosis.
6. **Report the JIT state** (P1 F13). An interpreted run is not comparable with a JIT'd one. The owner's JIT status is currently unknown [M] §5.

---

## 2. Benchmark scenarios by stress type

Each scenario has a **sim run** (`-timedemo -nodraw` with the timedemo fix from §1) and a **render run**. Render runs use `-playdemo` (real-time tic pacing with interpolated frames in between), because under the fixed `-timedemo` every frame carries a tic. The render run is done flat at a fixed 3,056 × 1,689 window, which is the [M] window so baselines and targets share a resolution, and then in the headset at runtime-recommended resolution × 1.0. Choose candidate maps once, freeze their versions and hashes in the harness, and never change them silently.

| ID | Stress type | Content | What it isolates |
|---|---|---|---|
| **S1** | Actor count, stock | Sunder MAP15, no mods, from the map start, moving the same 60 s route each run (demo). Variants **S1s** (and **S2s** for S2): standing still at the start for 700 tics with no input, which reproduces the [M] runs | Engine per-thinker overhead at about 8k thinkers [M] §1; slaughter AI; sight; sprites |
| **S2** | Actor count + ZScript helpers (owner workload) | Sunder MAP15 with the owner's 57-file load order | The measured worst case: about 53k thinkers, 130.8 ms of thinkers per tic [M] §1–2 |
| **S3** | ZScript-heavy mod, second map | Abandon MAP13 with the owner's load order | Same mod mix on different geometry: 61–87 ms per tic [M] §2 |
| **S4** | Huge geometry | A large UDMF map with tens of thousands of sidedefs and open sightlines (candidate: a "Frozen Time"-class map; freeze one) | BSP walk, clipper, wall setup, static-world GPU cost, Hi-Z value |
| **S5** | Portal-heavy | A map with line-to-line portals, sector stacks, a skybox viewpoint and mirrors in view together; plus a synthetic generator map (below) | Portal recursion cost per eye; stencil; Phase 5 portal passes |
| **S6** | Dynamic-light-heavy | A map with 200+ visible dynamic lights (GLDEFS attached lights on monsters and projectiles), with and without `gl_light_shadowmap` | Per-surface light lists, light buffer overflow, Tier A/B lighting |
| **S7** | Models | A model-replacement pack (MD3 / IQM skeletal) on S1's map | Model draw, bones, `AnimateBones` VM calls |
| **S8** | Render-only VR | S1, S4 and S5 with a scripted head-motion path: yaw sweeps at 300°/s, pitch ±80°, lateral lean 20 cm, played back through a fake-pose provider so that it repeats | Stereo culling correctness (head-shake test), per-eye cost, pop-in |

**Inference (S8):** OpenXR has no standard way to inject poses into a real runtime. The fake-pose provider is engine-side: it replaces the `xrLocateViews` results, keeps the runtime's FOVs, and still runs the real frame loop. The compositor then reprojects against the real head pose, so S8 frames are valid for culling and cost, not for comfort judgements. A headless alternative is a simulated-HMD runtime such as Monado's; **Open risk:** check that its multiview and depth-layer support match the target runtime before relying on it.

**Synthetic generator maps** (a small script writing UDMF), for clean scaling curves:
- `gen_actors_N[_aA]`: N ∈ {1k, 4k, 16k, 64k} idle monsters in closets, plus A awake (default N/4; `gen_actors_16k_a5000` is the P1 Phase 9 goal case). Variants: (a) stock monsters, (b) stock monsters each with one "HP-bar-style" ZScript helper actor that `A_Warp`s to its master every tic, (c) the same helper rewritten with P1 Phase 10 mechanisms (overlay, attachment, throttled statnum).
- `gen_lights_N`: N point lights in a grid room.
- `gen_portals_N`: N nested line portals and mirrors in view.
- `gen_sectors_N`: N moving sectors (doors and lifts in constant motion), the mover and patching path.

**Mod-family micro-scenarios** from the owner's load order, one map each, to attribute cost per family [M] §3:
- RSQ HP bar only;
- mk-champions only;
- Legion of Bones only;
- crit helpers only.

Each one is measured on S1's map, with and without the family.

---

## 3. Metrics and how to collect them

### 3.1 Per-frame metrics (CSV, one row per frame)

| Metric | Source | Notes |
|---|---|---|
| `frame_ms` | `FrameMark`-to-`FrameMark`; in VR, the `xrWaitFrame`-return to `xrWaitFrame`-return interval | Report median, p95, p99, max |
| `cpu_sim_ms` | Sum of `Sim.Tic` zones run on this frame (single-thread form) or 0 (sim thread) | The owner's "think" [M] §1 |
| `cpu_script_ms` | Stock `VMCycles[0]` per frame: clocked around script `VMCall`s and paused during native calls [U] src/common/scripting/vm/vmframe.cpp:732-736, 351-353, and reported by `stat vm` (895-911). The harness must read and reset it itself, because `stat vm` is what shifts it | Similar in intent to the owner's "script" [M] §1, but not identical: his figure is the fork's effects-governor attribution. Compare deltas, not absolutes |
| `cpu_scene_ms` | `Scene.Create` (+ `Scene.TicBuild` after P1 Phase 3) | Split into BSP, clip, walls, flats, sprites, as in [M] §1 |
| `cpu_submit_ms` | Command recording: `Scene.Opaque` + `Scene.Translucent` + `Portal.*` + `Render.2D` minus nested build zones | |
| `cpu_wait_ms` | `VK.FenceWait` (end of frame [U] src/common/rendering/vulkan/system/vk_commandbuffer.cpp:186, plus the submit-ring fence wait in `FlushCommands`, line 107), plus `VR.WaitFrame` blocking time | The owner's "GPU wait" [M] §1 |
| `gpu_ms_total`, `gpu_ms_eye0/1` (2-pass) or `gpu_ms_view` (multiview) | Vulkan timestamps per P1 §0.2 scope | Per pass: `GPU.Opaque`, `GPU.Masked`, `GPU.Translucent`, `GPU.Portals`, `GPU.PP`, `GPU.LightCull`, `GPU.Cull`, `GPU.HiZ` |
| `draws`, `applies`, `material_binds`, `hwbuffer_binds`, `pipeline_binds`, `stream_bytes` | `TracyPlot` counters (P1 §0.1) | Per frame |
| `walls`, `flats`, `sprites`, `portals`, `decals` | existing `rendered_*` counters [U] src/common/rendering/hwrenderer/data/hw_clock.cpp:45 | Per frame |
| `tris_rendered` | `VK_QUERY_TYPE_PIPELINE_STATISTICS` (clipping primitives, fragment-shader invocations). **API fact:** this needs the `pipelineStatisticsQuery` device feature. Inside a multiview render pass a query uses one index per view, and how the counts are split across those indices is implementation-dependent; only their sum is defined | Visibility efficiency. Report per view only in 2-pass mode; under multiview, report the sum |
| `light_uploads`, `light_overflow` | `FLightBuffer::UploadLights` [U] src/common/rendering/hwrenderer/data/hw_lightbuffer.cpp:75-123. Count as overflow (a) the silent truncation branch, where lights are dropped to fit `mMaxUploadSize` (83-99), and (b) the buffer-full `return -1` (121). Do **not** count the `totalsize <= 1` "no lights" `-1` (104) | Overflow must be 0 |
| `alloc_vertices` | `FFlatVertexBuffer::AllocVertices` count [U] src/common/rendering/hwrenderer/data/flatvertices.cpp:138-148 | P1 Phase 4 target: ≈ 0 for walls and flats |

### 3.2 Per-tic metrics (CSV, one row per tic)

| Metric | Source |
|---|---|
| `tic_ms` | `Sim.Tic` |
| `interp_reset_ms` | `Sim.ClearInterp` [U] src/p_tick.cpp:241-252 (measured 9.0 ms in S2 [M] §2) |
| `thinkers_ms`, `thinkers_count`, `actors_ticked`, `dormant_skipped`, `lod_skipped` | `Sim.RunThinkers` [U] src/playsim/dthinker.cpp:93-201 plus new counters |
| `worldtick_handlers_ms` | `Sim.WorldTickEvents` [U] src/p_tick.cpp:261 (4.1 ms in S2 [M] §2), split per handler class (P1 §0.5) |
| `particles_ms` | `Sim.Particles` |
| `clientside_ms` | `Sim.ClientSide`, `Sim.RecreateLights`, `Sim.DynLightTick` |
| `top_classes` | `sim_profile` per-class top 20: ms, calls, µs per call (the format of `profilethinkers` [U] src/playsim/dthinker.cpp:123-198) |
| `vm_script_calls`, `vm_interp_calls`, `action_calls`, `sight_checks`, `trymoves` | P1 §0.1 and §0.5 counters |
| `allocs_per_tic`, `alloc_bytes_per_tic`, top allocating classes | P1 §0.5 allocation counter |
| `gc_step_ms`, `gc_state` | `Sim.GCCheck` zone around the `Step()` call in `GC::CheckGC` [U] src/common/objects/dobjgc.cpp:155-163. `CheckGC` runs once per frame in `TryRunTics` [U] src/d_net.cpp:2192 and again in `D_Display` [U] src/d_main.cpp:1115, outside any tic, so it is reported per frame and not added to `tic_ms` |
| `snapshot_ms`, `snapshot_bytes` | P1 Phase 8 |
| **`tics_per_sec_under_load`** | Tics completed per wall-clock second in a `-timedemo -nodraw` sim run **with the timedemo fix (§1)**. Without it, 5.0 paces tics at 35 Hz and the figure saturates at real time. **≥ 35.0 means real time**, and the excess is headroom. Below 35, the game runs slow regardless of display. S2 today is roughly 6–7 sim-only (**Inference** from [M] §2: about 145 ms per tic summed across `ticprofile`), and about 6 with rendering (think 249.88 ms per frame ÷ ≈ 145 ms per tic ≈ 1.7 tics per frame, × 3.4 fps [M] §1). Measure it; do not quote the 3.4 fps as tics per second |

### 3.3 VR-specific metrics (headset runs)

| Metric | How |
|---|---|
| **Missed frames** | Per frame, compare `XrFrameState.predictedDisplayTime` with the previous one. A delta above 1.5 × the *nominal* refresh period counts as a missed or reprojected frame. Take the nominal period from the first frames or from `XR_FB_display_refresh_rate`, not from the current `predictedDisplayPeriod`: a runtime that throttles the app to half rate (for example ASW) reports a doubled period, and the misses would then hide. Report the count per minute, the longest run, and the time spent throttled. **API fact:** Meta runtimes expose `XR_META_performance_metrics` counters; log them where the extension exists, labelled by runtime. SteamVR's frame timing is an OpenVR interface (`IVRCompositor::GetFrameTiming`) or its UI. **Open risk:** whether an OpenXR app can read it in-process is unverified, so treat it as an external cross-check. |
| `xr_wait_ms`, `xr_begin_to_submit_ms`, `xr_submit_to_end_ms` | `VR.WaitFrame`, `VR.BeginFrame`, `VR.EndFrame` zones |
| Pose-to-photon age | `predictedDisplayTime − time(xrLocateViews call used for the submitted matrices)`. Read the second term in `XrTime` by converting the CPU clock with `XR_KHR_win32_convert_performance_counter_time` (Windows) or `XR_KHR_convert_timespec_time` (Linux). After the P1 Phase 8 late latch, this measures the latch's benefit |
| Envelope violations | Count of synchronous `TicScene` rebuilds caused by the head leaving the envelope (P1 Phase 3) |
| One-eye pop-in | The S8 head-shake test (§5.3) |
| Comfort flags | Runtime warnings (dropped-frame overlay, reprojection active %) |

### 3.4 Build and environment record (one per run)

Commit hash, compiler, LTO on/off ([U] CMakeLists.txt:15-28, `FORCE_NO_LTO`), the JIT line (P1 F13), GPU/driver/runtime versions, headset model, recommended render size, the load order with file hashes, and the cvar dump of §1, principle 2.

---

## 4. Tracy zone and counter list

These are the names used throughout P1, so captures from every phase can be compared. Every zone on an existing code path is anchored to the 5.0 function it wraps. Zones marked *new* wrap code that P1 adds; 5.0 has no OpenXR path, and its stereo is the `mEyeCount` loop in `RenderViewpoint`.

**Frame and VR**
- `FrameMark` after `D_Display` [U] src/d_main.cpp:1463-1467 (desktop), or after `xrEndFrame` (VR).
- *New (P1 Phase 2):* `VR.PollEvents`, `VR.WaitFrame`, `VR.BeginFrame`, `VR.LocateViews`, `VR.LateLatch` (Phase 8), `VR.AcquireSwapchain`, `VR.EndFrame`, `VR.MirrorBlit`.

**Sim** (P1 §0.1, §0.5)
- `Frame.TryRunTics` [U] src/d_net.cpp:2190-2347, with:
  - `Sim.GCCheck` (around `Step()` only; `GC::CheckGC` is called at line 2192 and in `D_Display`, see §3.2);
  - `Sim.TicStabilityWait` [U] src/d_net.cpp:2107-2122, called at 2275. This zone must disappear in VR builds (F12). The owner measured 2.3 ms per frame [M] §4;
  - `Sim.Tic` (`G_Ticker`, line 2323);
  - `Sim.ClientSide` (`P_RunClientSideLogic`, lines 2294 and 2341).
- Inside `P_Ticker` [U] src/p_tick.cpp:168-276: `Sim.ClearInterp` (the `ClearInterpolation` loop, 241-252), `Sim.Particles` (`P_ThinkParticles`, 254), `Sim.PlayerThink` (256-258), `Sim.WorldTickEvents` (261), `Sim.LevelTick` (`Level->Tick`, 262), `Sim.RunThinkers` (263), `Sim.Specials` (`P_UpdateSpecials`, 266-269).
- In `RunClientSideThinkers` [U] src/playsim/dthinker.cpp:226-397: `Sim.RecreateLights` (the lambda at 250-265, called at 308) and `Sim.DynLightTick` (309-318).
- Per class, sampled (`sim_profile 1`): `TracyPlot("cls/<Class>")`.
- Per virtual (`vm_profile 1`): transient zones named by `VMFunction::PrintableName` around `VMCall` [U] src/common/scripting/vm/vmframe.cpp:705-740, the event loopers [U] src/events.cpp:1068, 2068-2076, and JIT prologues (*new*).
- *New, after P1 Phase 8:* `Sim.SnapshotBuild`, `Sim.SnapshotPublish`, `Sim.UiScripts` (status bar and `RenderOverlay` on the sim thread), `Sim.VisDeltaApply`.

**Render**
- `Render.Display` (`D_Display` [U] src/d_main.cpp:1109).
- In `D_Render` [U] src/d_main.cpp:600-620: `Render.Interp` (`DoInterpolations`, 610), `Render.ParticleBucket` (`P_FindParticleSubsectors`, 611 → [U] src/playsim/p_effect.cpp:204-239), `Render.PolyLink` (`PO_LinkToSubsectors`, 612), and `Render.InterpRestore` (`RestoreInterpolations`, 618). The two interpolation zones disappear after P1 Phase 3.
- In [U] src/rendering/hwrenderer/hw_entrypoint.cpp: `Render.View` (`RenderView`, 311-407), `Render.CameraTextures` (canvases and camera textures, 358-386), and `Render.Eye0`/`Eye1` (the eye loop in `RenderViewpoint`, 126-188; removed by single-pass).
- In [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp: `Scene.Create` (`CreateScene`, 470-511), `Scene.RenderHacks` (501-504), `Scene.Opaque` (`RenderScene`, 521), `Scene.Translucent` (`RenderTranslucent`, 581), and `Scene.EndDraw` (`EndDrawScene`, 908).
- In [U] src/rendering/hwrenderer/scene/hw_bsp.cpp: `Scene.BSP` (`RenderBSP`, 1008-1062), `Scene.Worker` (`WorkerThread`, 111-216), and `Scene.WorkerWait` (`future.wait()` under the `MTWait` clock, 1036-1038).
- `Portal.EndFrame`, `Portal.Render.<Class>` [U] src/rendering/hwrenderer/scene/hw_portal.cpp:73, 158.
- *New in P1 Phases 3–7:* `Scene.TicBuild`, `Scene.FrameCull`, `Scene.IndirectBuild`, `Scene.SpriteInstances`, `Scene.TranslucentSort`, `Scene.ModelPose`.

**Backend**
- `Render.PostProcess` (`VkPostprocess::PostProcessScene` [U] src/common/rendering/vulkan/renderer/vk_postprocess.cpp:76).
- `Render.2D` (`Draw2D` in `VulkanRenderDevice::Update` [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:201-222).
- `VK.WaitForCommands` [U] src/common/rendering/vulkan/system/vk_commandbuffer.cpp:163-199, containing `VK.Acquire` (170), `VK.Submit` (`FlushCommands`, 173), `VK.FPSLimit` (178), `VK.Present` (179) and `VK.FenceWait` (186).
- `VK.MidFrameFlush` (`vk_submit_size` flush in `Apply` [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:186-190).
- `VK.TextureUpload` (`VkHardwareTexture::CreateTexture` [U] src/common/rendering/vulkan/textures/vk_hwtexture.cpp:138-187).
- `VK.PipelineCompile` (`VkRenderPassSetup::CreatePipeline` [U] src/common/rendering/vulkan/renderer/vk_renderpass.cpp:243).
- *New, after P1 Phase 1:* `VK.FrameBegin` (fence wait for context k), `VK.UploadWorker`, `VK.PipelineWorker`.

**GPU zones (TracyVk)**
- `GPU.Clear`, `GPU.SkyFirst`, `GPU.Opaque`, `GPU.Masked`, `GPU.Models`, `GPU.Decals`, `GPU.Portals.<n>`, `GPU.Translucent`, `GPU.SSAO`, `GPU.PP`, `GPU.2D`, `GPU.Present`.
- Later: `GPU.Cull`, `GPU.HiZ`, `GPU.LightCull`, `GPU.ShadowCube`, `GPU.Skinning`, `GPU.SkyboxCube`, `GPU.XRBlit`.

**Counters (`TracyPlot`)**
- `draws`, `applies`, `material_binds`, `hwbuffer_binds`, `pipeline_binds`, `stream_bytes`;
- `walls`, `flats`, `sprites`, `portals`, `decals` (from the stock `rendered_*` counters);
- `actors_ticked`, `dormant_skipped`, `lod_skipped`;
- `vm_script_calls`, `vm_interp_calls`, `action_calls`, `sight_checks`, `trymoves`;
- `allocs_per_tic`, `gc_step_bytes`;
- `worker.idle_spins`, `light_overflow`, `envelope_violations`, `xr_missed_frames`.

---

## 5. Regression gates (all phases)

### 5.1 Bit-exact simulation

The built-in netgame consistency check is weak and inactive for this purpose. It covers only each player's position, yaw, pitch and health plus the sum of four RNG seeds, and it runs only in netgames and never during demo playback [U] src/d_net.cpp:1146-1183. Add `sim_hashlog 1`, which writes one 64-bit hash per tic over:
- **every** play RNG state, by walking `FRandom::RNGList` [U] src/common/engine/m_random.h:218 and skipping the client-side `CRNGList`. `StaticSumSeeds` covers only four RNGs (d_net.cpp:1146-1153), so it is not enough;
- every actor in **`SpawnOrder`** order [U] src/playsim/actor.h:1408: class, position, velocity, angles, health, `state` pointer index, `tics`, `flags*`, `special` and `args`. Use a canonical order, not list order, because dormancy and throttled statnums (P1 Phases 9–10) legitimately move actors between lists. A change in tick order that alters behaviour still shows up as a state difference a few tics later. Hash other thinkers by class, then list order;
- sector plane D values and light levels, polyobject positions and angles;
- line flags and specials, with `ML_MAPPED` masked out: the renderer sets it (automap reveal), so including it would fail the demo diff whenever a render change alters what gets drawn.

Then:
- **Gate:** every default-on sim change (F1, F2, F4, F5, P1 Phase 9 steps 1, 2, 4–10, Phase 10 engine mechanisms used by an unchanged mod) must produce **identical hash logs** against the baseline build, over S1–S3 and the generator maps, for at least 10,000 tics each.
- Opt-in approximate features (`sv_ticklod`, `sv_closetlook`) are tested for *stability*, not equality: no desync between two runs of the same build.
- **Dormancy shadow mode:** with `sv_dormancy_verify 1`, dormant actors are also shadow-ticked and asserted unchanged (P1 Phase 9).
- **Render-list verification:** `r_verifyrenderlists 1` for F2 (P1 §0.4).
- **Ported mods are out of scope for equality.** Once the owner's mods are ported (Phase 10), S2 and S3 demos may desync, because helper actors no longer spawn and so `SpawnOrder` and RNG call sequences change. Phase 10 performance runs therefore use the input-free S2s variant (§2) with a fixed tic count. They are compared as workloads, not tic for tic.

### 5.2 Pixel diff

Use fixed camera positions (about 50 per map) on S1, S4, S5, S6 and a set of mapping-feature test maps: 3D floors with light lists, slopes, Boom 242, missing-texture tricks, scrollers, polyobjects, decals, fog boundaries, translucent mid-textures, wall/flat/roll sprites, translations.
- Render the legacy path and the new path, then compare with a per-pixel tolerance (for example ΔE < 2) plus a count of changed pixels.
- **Gate:** no change beyond tolerance, except entries on an "intended differences" list (for example the improved fuzz, or Phase 6's per-pixel sprite light). This gates P1 Phases 1 (SSBO draw data and bindless must be pixel-identical), 2 (multiview against two-pass, per eye), 3, 4, 5, 6 and 7.
- Baselines are per runner and driver: a lavapipe reference image is not comparable with an NVIDIA one.

### 5.3 Head-shake visibility test (VR culling)

On S8, render every frame twice:
- (a) with the production cull (tic-rate envelope plus per-frame frustum, or GPU cull);
- (b) with culling disabled.

Diff the depth buffers per eye. Any pixel whose depth is nearer in (b) than in (a) means geometry was missed.
- Pass (b) must not write the Hi-Z pyramid, the visibility history or any other temporal state that (a) reads in the next frame.
- Compare with a small depth epsilon (a few ULP). The GPU-driven path and the reference may use different vertex shaders, and depth is bit-identical only if both declare `invariant gl_Position` and share the transform code. Require that invariance for the resident-world shaders (P1 Phase 4), so that the epsilon can stay tiny.
- Only opaque and alpha-tested geometry writes depth. Missed translucent surfaces and portal interiors need the colour diff of §5.2 on the same frames.
- **Gate:** 0 pixels on 100% of frames for P1 Phases 2, 3 and 5. Report one-eye-only misses separately: those are the uncomfortable ones.

### 5.4 Savegame round-trip

Save, load and continue for 2,000 tics, then compare hash logs with an uninterrupted run. This gates field reordering, arenas (P1 Phase 9 step 8), dormancy, attachments and overlays (Phase 10).

---

## 6. Baselines (record before any change)

Re-measure the owner's numbers on **stock 5.0** with the harness, using S1s/S2s at 3,056 × 1,689, uncapped, with `-rngseed 1113`, so that all targets refer to one build. Pending that, the fork numbers [M] are the reference. They come from `doomxr_rend1113a.exe` with `r_newrenderer 0`, which is the old render path [M] (header).

The `r_ticstability` A/B pair (10.01 against 12.31 ms) comes from a different session (700 tics) from the 13.89 ms run [M] §1, §4. Compare F12 against the A/B pair, not against 13.89.

| Scenario | Baseline (owner, fork, flat) |
|---|---|
| S1 Sunder MAP15, no mods | 13.89 ms frame (66.7 fps); think 2.52, script 3.14, scene build 3.80 (BSP 0.48, clip 1.27, walls 2.30, flats 0.60, sprites 0.14, draw state 0.67), GPU wait 3.24; 8,065 thinkers; per tic: interp reset 1.25–1.39, thinkers 3.24–3.79 [M] §1–2 |
| S1 with `r_ticstability 0` | 10.01 ms against 12.31 ms with it on [M] §4 |
| S2 Sunder MAP15, owner mods | 315.29 ms frame (3.4 fps); think 249.88, script 258.24, scene build 4.35, GPU wait 25.87; 53,116 thinkers; per tic: interp reset 9.02, particles 0.72, WorldTick handlers 4.07, thinkers 130.80. Top classes: `RSQTHP_Bar` 16.46 ms over 3,622 calls; mk-champions ≈ 25; Legion of Bones ≈ 11; `Shifter` 2.23 over 7,686; crits 1.57 over 11,332 [M] §1–3. The listed classes sum to ≈ 52 ms of the 130.8 ms: about 78 ms per tic is not yet attributed to a class |
| S3 Abandon MAP13, owner mods | Thinkers 61.4–87.2 ms per tic; interp reset 6.5–6.8; WorldTick handlers 2.3–3.4; `RSQTHP_Bar` 7.97 ms over 1,891 bars [M] §2–3 |
| S2 GPU (flat) | Opaque 6.15 (walls 3.09, flats 1.21, models 0.84), exposure 1.38, bloom 3.23, BloomBoost 1.60 + 1.64 [M] §1 |
| S4–S8 | Measure with the harness before Phase 1 starts |

---

## 7. Acceptance targets per phase

All targets are **Inference**, set against the baselines in §6 and the P2 §7 budgets. "Flat" means the 3,056 × 1,689 window on the owner-class PC (3080 Ti / 9900KF). "VR" means headset at recommended resolution × 1.0, 90 Hz unless stated. Phases are listed in the P1 recommended order (sim track first), not in number order.

### Phase 0: instrumentation and cheap fixes
- Tracy captures and CSVs exist for S1–S8, and the JIT line is in every log.
- The harness timedemo fix (§1) is in, and S2's `tics_per_sec_under_load` baseline is measured.
- F12 (sleep-based wait): S1s frame within 3% of `r_ticstability 0`, so ≤ 10.3 ms against 10.01 ms [M] §4. `Sim.TicStabilityWait` burns ≤ 0.1 ms of CPU per frame.
- F1 + F2 + F4 + F5 together: S1 `Sim.RunThinkers` −15%; S2 `thinkers_ms` −10% (engine overhead is a smaller share there; the scripts dominate).
- F3 with `gl_sort_textures 1`: `material_binds` −80%.
- `sim_profile` reproduces the owner's top-class list for S2 within ±20% per class, **and attributes ≥ 90% of `thinkers_ms`** to named classes, closing the ≈ 78 ms gap in §6. The Phase 10 target depends on knowing what that residue is.
- Bit-exact gate (§5.1) passes.

### Phase 9: playsim engine overhead (**first on the owner's workload**)
- Interpolation reset (step 9 A + B): S2 9.0 ms → ≤ 2.0 ms per tic; S1 1.3 → ≤ 0.3 ms.
- Native fast paths and dormancy: S1 `thinkers_ms` 3.2–3.8 → ≤ 2.0 ms, which is P1's −40% or better; stock-monster µs per call 0.27–0.47 [M] §3 → ≤ 0.2.
- Together, the two items above put S1 `tic_ms` p99 ≤ 3 ms. That is the P2 §7 single-thread budget at 90 Hz, and the Phase 2 VR target needs it.
- P1 Phase 9 goal: `gen_actors_16k_a5000` `tic_ms` p99 ≤ 10 ms.
- Allocations per tic in S2 −90% (F1 plus slabs plus the `TickBehaviors` fix).
- Grid, inline blockmap and sight cache: `sight_checks` time −50% and `trymoves` time −30% on S1.
- **Gate:** bit-exact across S1–S3.

### Phase 10: helper-actor explosion (with the owner's mod ports)
- RSQ HP bar on the overlay API: `RSQTHP_Bar` 16.46 ms → 0, and 3,622 fewer thinkers in S2.
- mk-champions and crit patterns ported (or demonstrated on the generator maps): S2 thinkers 53,116 → ≤ 20,000.
- **S2s `thinkers_ms` ≤ 20 ms per tic** (P1 Phase 10's combined target, from 130.8), with `tic_ms` median ≤ 25 ms and p95 ≤ 28.6 ms. Then **S2 `tics_per_sec_under_load` ≥ 35** (real time) holds in a `-timedemo -nodraw` sim run.
  - The non-thinker part of the tic must fit in about 5 ms: interpolation reset ≤ 2 (Phase 9), `WorldTick` handlers ≤ 2.5 (from 4.07), and particles 0.7.
  - A looser 25 ms thinker target would not give real time: 25 + 9.0 + 4.1 + 0.7 = 38.8 ms per tic today, and about 25 + 7 even after Phase 9.
- S3 `thinkers_ms` ≤ 12 ms per tic (61–87 today, which scales with S2's ≈ 0.55×).
- **Open risk:** P2 §7's ≤ 20 ms *whole-tic* target is a stretch here. ≤ 20 ms of thinkers requires the ≈ 78 ms that no class yet accounts for (§6) to shrink by about 6× like the rest. If Phase 0 attribution shows that residue is genuine per-monster AI (Legion of Bones costs 2–3 µs per tick [M] §3), only mod-side changes close the gap, and the target becomes "real time on S3, best effort on S2".
- Generator `gen_actors_16k` variant (c) (helpers on the new API) costs ≤ 1.2× variant (a) (no helpers).

### Phase 1: Vulkan backend
- S1s flat with F12: `cpu_wait_ms` 3.2 → ≤ 0.5 ms; frame ≤ max(CPU ms, GPU ms) × 1.05 (P1: frame ≈ max(CPU, GPU)) and ≤ 8.0 ms (≥ 125 fps, +25% over 10.01 ms).
  - Why ≤ 8.0: removing the 3.24 ms wait from 10.01 ms leaves ≈ 6.8 ms of CPU [M] §1, §4.
  - Before setting the number, measure S1's GPU time per frame (§6, Phase 0). If the GPU time is above 7.6 ms, the cap becomes GPU ms × 1.05.
- `material_binds` + `hwbuffer_binds` < 5% of draws.
- Zero `VK.PipelineCompile` and `VK.TextureUpload` on the main thread after warm-up.
- S2 flat: `cpu_wait_ms` 25.9 → ≤ 2 ms. At least 14 ms of GPU work per frame ([M] §1: opaque 6.15 plus 7.85 of exposure, bloom and BloomBoost) hides completely behind a 300 ms CPU frame once frames are in flight.
- Pixel-diff gate (§5.2) passes.

### Phase 2: OpenXR and multiview
- VR S1 (no mods) at 90 Hz: one `Scene.Create` per frame; `draws` ≈ the mono count; render-thread CPU ≤ 1.15× mono; GPU ≤ 9.5 ms; ≤ 1 missed frame per minute in single-thread form with F12.
  - **Precondition:** S1 `tic_ms` p99 ≤ 3 ms (Phase 9 steps 1, 2 and 9; P2 §7).
  - Today a frame that runs a tic carries about 5 ms of sim [M] §2 on top of about 3.8 ms of scene build, which overruns 11.1 ms.
  - Until the precondition holds, report misses on tic-carrying frames separately and do not fail the phase on them.
- Doom II MAP01–10 at 90 Hz and native per-eye resolution with zero missed frames (P1). Stretch: 120 Hz with ≤ 1 missed frame per minute.
- Head-shake gate (§5.3): 0 one-eye misses. Pixel diff, multiview against two-pass: clean.

### Phase 3: tic-rate scene build
- `Scene.Create` + `Scene.TicBuild` ≤ 35 per second at both 90 and 120 Hz.
- Render-thread per-frame CPU on S4 ≤ 2.5 ms (≤ 1 ms after Phases 4 and 5, P1).
- Envelope violations ≤ 1 per minute on S8.
- Head-shake gate (§5.3) and pixel-diff gate (§5.2) pass.
- `Render.Interp` zones gone.

### Phase 7: sprites, particles, models, overlays
- 4,000 visible sprites: sprite CPU ≤ 0.3 ms per tic and ≤ 0.05 ms per frame; ≤ 10 sprite draws per pass. Use an S1 camera point that sees ≥ 4,000 actors, or `gen_actors_16k`. The S1 start shows only 52 sprites [M] §1.
- Identical rotation frames in both eyes (P1). Check this by comparing per-instance frame indices between views on S8.
- Pixel-diff gate passes, with intended differences listed (fuzz, OIT).
- S7: model CPU ≤ 5 µs per model per frame; the `AnimateBones` call count equals visible decoupled models per frame (not × eyes × portals).
- Phase 10 overlays: 3,622 bars cost ≤ 0.2 ms GPU, both eyes.

### Phase 8: sim thread
- S2 and S3 after Phases 9 and 10: VR render p99 frame ≤ 11.1 ms while `Sim.Tic` p99 is up to 25 ms.
- Missed frames caused by sim stalls = 0.
- `tics_per_sec_under_load` within 3% of single-thread, so the split does not slow the sim. `snapshot_ms` is ≤ 1 ms per tic on S2.
- TSAN clean on S1–S7 (P1: all scenarios).
- Pose-to-photon age reduced by ≥ 3 ms with the late latch.

### Phase 4: resident world
- `alloc_vertices` for walls and flats = 0 per frame.
- ≤ 50 world draws per view pass on S1 and S4.
- Mover and light update < 1 µs CPU each (`gen_sectors_N`).
- Pixel-diff gate passes.

### Phase 5: GPU visibility and portals
- S4: `tris_rendered` ≤ 1.5× a perfect-visibility reference; no `Scene.BSP` per frame.
- S5 and `gen_portals_N`: CPU ≤ 0.05 ms per visible portal; GPU scales with visible portal pixels; `r_portal_recursions 4` at 90 Hz VR on S5.
- Head-shake gate passes with Hi-Z on.

### Phase 6: lighting
- S6: CPU light work ≤ 0.2 ms per tic, `light_overflow` = 0, GPU ≤ 1.3× the S6-with-`r_dynlights 0` time at 200 visible lights.
- Pixel-diff gate passes against an intended-differences list (per-pixel sprite light).
- `gen_lights_N`: GPU light cost is sub-linear in N beyond 64 lights per view (Tier B).

### End-to-end (the project goal)
- **VR 90 Hz, no mods:** S1, S4, S5, S6 with ≤ 1 missed frame per minute, GPU ≤ 9.5 ms and render-thread CPU ≤ 5 ms.
- **VR 90 Hz, owner's mod set:** S2 and S3 in real time (`tics_per_sec_under_load` ≥ 35) with ≤ 2 missed frames per minute, post-process mods reduced or off per §8 of P2. For S2 this is subject to the Phase 10 open risk.
- **VR 120 Hz, no mods:** Doom II and S1 with ≤ 1 missed frame per minute.

---

## 8. Cadence and reporting

- **Per pull request (CI, headless):**
  - the sim runs of S1 and the generator maps (`-timedemo -nodraw`) for 10,000 tics, with hash-log comparison and per-tic CSV;
  - the pixel-diff suite on a software-rasterised runner (lavapipe) or a fixed-GPU runner;
  - a summary table: median and p99 `tic_ms`, `thinkers_ms`, `interp_reset_ms`, allocations per tic, top-10 class deltas.
- **Nightly (owner's PC or a private runner):** the S2 and S3 sim runs with hash logs.
  - At about 145 ms per tic today, 10,000 S2 tics take about 25 minutes per build, so the §5.1 gate runs nightly rather than per PR.
  - **Open risk:** the owner's 57-file load order includes third-party mods that may not be redistributable to a shared CI host. Keep them on the owner's machine and store only their hashes in the harness.
- **Weekly (owner's PC, flat):** S1–S7 render runs, with CSVs and Tracy captures archived next to the owner's existing `render_bench` reports [M] (header).
- **Per milestone (headset):** S1, S2 and S8 in VR with missed-frame and pose-age metrics, plus a short comfort note (judder, pop-in, shimmer).
- **Spike triage:** every `sim_spike_ms` trigger (P1 §0.5) and every frame over 2× budget dumps the per-class top 10 and the open zones to the log. The CI summary lists the five most frequent spike sources.
