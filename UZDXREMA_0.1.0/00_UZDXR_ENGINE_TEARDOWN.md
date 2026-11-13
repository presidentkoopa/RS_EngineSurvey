# 00 — UZDXREMA engine teardown: summary reference

Teardown 0.1.0, written 2026-11-13 from sections 01-10 of this folder, including every section's appended
cross-check. Where a cross-check corrected a claim, the corrected claim is used here. This file describes the
engine as it is. It holds no plans and no recommendations.

**Conventions**
- Pointers: `§03 6.1` = file `03_…md`, its section 6.1. `M10 §2` = `10_MEASUREMENTS.md` section 2 (the only
  measured numbers in this file). Code: `file:line` in `main` (`/home/claude/uzdx_main/src`) unless prefixed
  `renderer:`, `netplay:` or `selaco:`.
- Short paths: `ep` = `rendering/hwrenderer/hw_entrypoint.cpp`; `bsp` = `rendering/hwrenderer/scene/hw_bsp.cpp`;
  `di` = `rendering/hwrenderer/scene/hw_drawinfo.cpp`; `walls`, `flats`, `portal`, `sky` = `hw_walls.cpp`,
  `hw_flats.cpp`, `hw_portal.cpp/.h`, `hw_sky.cpp` in the same folder; `VK-XR` =
  `common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp`; `vk_renderdevice.cpp`, `vk_commandbuffer.cpp` are in
  `common/rendering/vulkan/system/`; `vk_renderstate.cpp`, `vk_postprocess.cpp`, `vk_pprenderstate.cpp` in
  `common/rendering/vulkan/renderer/`; `hw_vrmodes.cpp`, `hw_cvars.cpp`, `hw_clock.cpp` in
  `common/rendering/hwrenderer/data/`.
- Marks: a code statement with no mark was VERIFIED (code read) by the cited section. **(inferred)** = deduced in
  the cited section; the basis is there. **measured** = from `10_MEASUREMENTS.md` only. **doc figure** = a number a
  section quoted from a handoff or plan file on `renderer`; it was not re-measured by this teardown and is not a
  measurement here.

---

## 1. What UZDXREMA is

| Item | Fact | Pointer |
|---|---|---|
| Base | Stock UZDoom tag `5.0.0` (commit 292cf42, used only as the comparison baseline). The repo's root commit `68ec2561` ("UZDXR 5.0.0-dxr, pristine", 2026-09-06) already carries layer **P** (QuestZDoom/DoomXR VR port and earlier fork work). Layer **R** = everything added on `main` since. The exact upstream commit under DoomXR is unknown. | §08 1.1, 2 |
| Size of the fork | vs stock 5.0.0: 272 new files, 0 removed, 414 stock files modified (+79,617/−2,207 lines). 623 fork cvars (302 P, 321 R). +459 ZScript native functions and +161 native fields/globals. `main.fp` 944 → 5,128 lines. | §08 1.2-1.4, 6 |
| Renderer | Vulkan. VR = `vr_mode 15` (OpenXR, `VKOpenXRDeviceMode`, VK-XR, 8,632 lines). The GL and GLES backends and the GL OpenVR mode (`vr_mode 10`) are still compiled. `vid_preferbackend` defaults to `BACKEND_OPENGL` (`v_video.cpp:92`), so Vulkan is chosen by the config. | §01 6; §07 1.1, 2.1; §08 8.8 |
| Engine of record | `main` at f182cbcc. Every other branch lacks some of its work. `CLAUDE.md` says the same and the code agrees. | §09 Summary 1, A.1 |
| Trees read | `main` f182cbcc (`/home/claude/uzdx_main`); `renderer` 175063a1 (`/home/claude/uzdxrema`; its tip abb19af1 adds only teardown docs); `netplay` 1c1805f8 (`/home/claude/uzdx_netplay`); `selaco` 7e111a3d (`/home/claude/uzdx_selaco`); Helion 5edd437 (for comparison in §04 only). | §01 header; §04 cross-check; §09 cross-check |
| Measured build | `doomxr_rend1113a.exe` = `renderer` at b3fc177, run with `r_newrenderer 0` (old path). Checked for this summary: `git diff --stat b3fc177 175063a1` touches only `HANDOFF_RENDER_2026-11-11.md` and `docs/plans/STAGE3_VISIBILITY_PLAN.md`, so the measured source equals the read `renderer` source. The old path on `renderer` differs from `main` in its defaults `sim_maxcatchup 2`, `snd_evictretry 8`, `snd_evictbudget 64` and the per-source sound-channel index (§7 below). | M10 header; §09 A.2.1 |
| Hardware measured on | RTX 3080 Ti, i9-9900KF, 16 GB, flat screen, window 3056×1689, through the bridge's `boottest`. The headset is a Quest 2 over VDXR; no headset run exists. | M10 header, §5; §07 8 Q3 |
| Build defines assumed | `HAVE_VULKAN`, `HAVE_GLES2`, `USE_OPENXR`+`DYN_OPENXR`, `USE_OPENVR`+`DYN_OPENVR` (the CMake options `ENABLE_OPENXR`/`ENABLE_OPENVR` default to NO, `src/CMakeLists.txt:394-396, 464-466`; `auto-setup-windows-vr.cmd:98-104` turns them on; that the owner's `build-dxr` was configured by that script is (inferred)). MSVC `_WIN32`; never `__MOBILE__`, `__ANDROID__`, `__WIN32__`. ZScript JIT compiled in on x86-64 (`CMakeLists.txt:304-306`), `vm_jit` default true (`vmframe.cpp:43`); whether the owner's build has it is open. Struct sizes are for `NDEBUG` (release) builds. | §01 6; §02 1.4, cross-check; §08 8.8 |

---

## 2. How one frame runs (main; VR at defaults: `vr_openxr_sync_mode 1`, multiview off, `vr_scene_multithread` off)

Everything below runs on the main thread, in this order, serially (§01 1.1, 2.2-2.5). Flat-screen differences are in
brackets. "Per tic-frame" = per frame that runs at least one tic.

| # | Step | What it does | Where | How often | Detail |
|---|---|---|---|---|---|
| 1 | Loop top | Hang-watch timeout + beats (Windows); `I_StartFrame` (joysticks) only after a tic ran; `I_SetFrameTime` freezes "now" for the frame. That time is the base of `TicFrac` (world interpolation). | `d_main.cpp:1966, 1993-2003` → `i_time.cpp:90` | frame | §01 2.2 |
| 2 | `TryRunTics` head | `GC::CheckGC` (first of two GC checks a frame); `doWait` forced false in VR [flat: may `I_WaitForTic` = sleep then spin, with `cl_capfps`, paused, menus]; `totalTics` = clock difference, unbounded on `main`. | `d_net.cpp:2339, 2343-2376`; `i_time.cpp:148-174` | frame | §01 2.3; §02 2.1 |
| 3 | `NetUpdate` | `GetPackets`, `NetState_Pump`, `Host_LateJoinTick`; per new command `G_BuildTiccmd` → in VR `WriteHandCommand`, which reads hand/head poses cached by the previous frame's SetUp #2; pack, send (single player: local buffer), `GetPackets` again. ≤35 commands per call, ≤87 ahead. | `d_net.cpp:1430-1894`; `g_game.cpp:1106, 1527`; VK-XR:8317 | frame (× commands) | §01 2.3; §07 2.3 |
| 4 | Tic count | `runTics = min(totalTics, availableTics)`, +1 while behind. No cap on `main` (`renderer`: `sim_maxcatchup 2` in single player). | `d_net.cpp:2420-2426`; `renderer:d_net.cpp:207, 2415` | frame | §01 5.2; §02 5.8 |
| 5a | No tic due | If `totalTics<=0`: `TicStabilityWait` **busy-spins** on `steady_clock` for the wall time of the last stabilised tic's `G_Ticker`+`MakeConsistencies` (cap 1 s; not cut short when a tic falls due). Network stall instead: `P_ClearLevelInterpolation` (full actor walk). Then `P_PredictClient` if a command was added, client-side logic × `totalTics`, sounds. About 37 of 72 frames a second at 72 Hz have no tic (inferred). | `d_net.cpp:2251-2292, 2431-2463` | frame without tic | §01 1.4, 2.3, 5.3 |
| 5b | Tic(s) | `P_UnPredictClient` (deserialise the prediction rollback; no netgame guard, so also single player); per tic `G_Ticker` → `P_Ticker`: sector interpolation snapshot, **actor interpolation reset walk** over every actor, `P_ThinkParticles`, billboards/stamps/beams snapshots, `P_PlayerThink` per player (VR chain incl. two 8192-unit laser traces), `WorldTick` handlers, `Level->Tick`, **`RunThinkers`** (every thinker's `Tick` through the VM call layer), `P_UpdateSpecials`, `WorldStep`, `VRSyncLog_OnTick`; `MakeConsistencies` (netgame only: byte-wise FNV of 72 B per actor); then `P_PredictClient` (serialise rollback; re-runs predicted tics only when commands run ahead, i.e. netgame). | `p_user.cpp:2345, 2226`; `g_game.cpp:1983`; `p_tick.cpp:644-919`; `dthinker.cpp:149`; `d_net.cpp:1235-1288` | tic (prediction: per tic-frame) | §01 2.3; §02 2.2-2.4 |
| 6 | Client-side logic | `P_RunClientSideLogic` × `totalTics`: console/menu/`UiTick`, client-side thinkers, the light-recreate scan over **all** world and client-side actors, `FDynamicLight::Tick` for every light (relink on X/Y or link-radius change), status-bar `CallTick`, automap. | `p_tick.cpp:155-218` → `dthinker.cpp:294-477`; `a_dynlight.cpp:253-727` | real tic | §02 2.6; §05 2.1 |
| 7 | Sounds | `S_UpdateSounds` → walks every channel and, on `main`, restores evicted channels on every call (recursive over the channel list); `VerifyPredictedEntities`. | `d_net.cpp:2509-2510` → `sound/s_doomsound.cpp:988` → `common/audio/sound/s_sound.cpp:1316-1392` | frame with tics | §01 3; §02 2.7, 5.7 |
| 8 | Input | `I_StartTic` + `D_ProcessEvents`: consumes the controller key events that the previous frame's SetUp posted. | `d_main.cpp:2010-2011` | frame | §01 2.2; §07 2.3 |
| 9 | `D_Display` head | Second `GC::CheckGC`; `SolarXR_Tick`; FOV/size/wipe; `R_UpdateSky`, `TexAnim.UpdateAnimations`. | `d_main.cpp:1560-1684` | frame | §01 2.2 |
| 10 | `BeginFrame` + XR wait | **[VR] SetUp #1**: `++xrVRFrameIndex`, `PollXREvents` (one event), `UpdateControllerState` (`xrSyncActions`, hands located at the **previous** frame's predicted display time, single-player render-side pawn move `VR_ApplyRenderMove` → `P_XYMovement`), `updateHmdPose`. **[VR] `BeginXRFrame`**: `xrWaitFrame` **blocks** (the only pacer in VR), `xrBeginFrame`, `xrLocateViews(predictedDisplayTime)`, `updateHmdPose`. Then buffer sizing, `UpdateBackgroundCache` (≤100 texture creations; on the main thread when the upload family is transfer-only), command/texture/render-buffer/render-state BeginFrame, descriptor sets 0+1 rewritten. | `vk_renderdevice.cpp:1449-1507`; VK-XR:3575-3655, 6006-6131 (`xrWaitFrame` :6051) | frame | §01 2.2; §06 2.2; §07 2.2 |
| 11 | VR HUD canvas | `DrawHudToSurface`: status bar (ZScript) drawn into a canvas, once a frame, shared by both eyes. | `d_main.cpp:1254, 1693` | frame | §04 3.2; §07 3 |
| 12 | `D_Render` | `DoInterpolations` (writes sector planes etc., restored after the render), `P_FindParticleSubsectors` (O(subsectors + particles), rebuilt every frame), `PO_LinkToSubsectors`. | `d_main.cpp:690-709` | frame | §01 2.2; §04 2.1 |
| 13 | `RenderView` | Fake-flat arena cleared, clocks reset, bone and light buffer cursors reset (shared by everything drawn this frame), changed canvases drawn, each camera texture = one mono `RenderViewpoint` (its own full scene). | `ep:640-730` | frame; per camera texture | §03 2.1; §04 2.1 |
| 14 | `RenderViewpoint` head | `R_SetupFrame` (`TicFrac` from the loop-top clock); emissive/beam/effect-light BeginFrame; shadow-map pass when wanted; **SetUp #2** (fresh predicted time, second `UpdateControllerState`, third `updateHmdPose`); `R_UpdatePoseAnchoredLights`; `PrepareFrameCompute` + `RunFrameCompute` (fork compute recorded on the graphics queue into the draw command buffer). | `ep:264-346` | frame (main view) | §01 2.2; §05 2.2; §06 8 |
| 15 | **Eye loop** | 2 iterations in VR (with multiview: 1 scene + post-process for eye 1). Per eye: `++gl_dynlight_viewid` (invalidates per-light caches), `eye->SetUp` (projection from the runtime FOV), `StartDrawInfo`/`StartScene` (~800 lines of fork uniform fill), `SetupView`/`ApplyViewpoint` (26,944 B uniform block), `ProcessScene` (particle/atlas/drawn-line/view-light syncs, heat/smoke/emissive setup) → `DrawScene`. | `ep:359-516` (scene decision :372); `di:2066, 3336, 4597, 4450` | eye | §03 2.1; §07 2.2 |
| 15a | Scene build (BSP walk) | `CreateScene` → `RenderBSP`: `validcount++`; **single-threaded in VR**; `RenderBSPNode` + `Clipper::CheckBox` (1-D angular occlusion) → `DoSubsector` → `ProcessVisibleSubsector`: flat-VBO height check per sector, particles per subsector, `AddLine` per seg (backface, range test, fork distance cull → impostor, occlusion add, `ML_MAPPED`, per-linedef dedupe) → `HWWall::Process` per linedef side → `PutWall` → copy into a draw list; once per sector `RenderThings` → `HWSprite::Process` per actor → `PutSprite` (per-pixel-lit models gather lights with a BSP circle walk); once per section `HWFlat::ProcessSector`; portals found by `PutPortal`; then `PreparePlayerSprites`, `DispatchBillboards`, render hacks. [flat: one scene worker runs walls/flats/sprites; the walk itself is single-threaded in every mode] | `di:3665`; `bsp:1422-1563` (thread gate :1438-1439); `walls:2419, 827, 906`; `flats:672`; `hw_sprites.cpp:1057, 840` | eye (+ per scene portal + per camera texture) | §03 2.2-2.4, 2.7; §04 2.2 |
| 15b | Sky | `RenderFirstSkyPortal` (largest sky portal, dome, no walk). | `portal:123-168` | eye | §03 2.6 |
| 15c | Opaque lists | `RenderScene`: heat grid decay, then plain walls (`DrawWall`: on Vulkan **light list and vertices built here**, ~19 state setters, one `Draw` per wall part), plain flats (`SetupLights`, one `DrawIndexed` per section plane), masked walls/flats, `GLDL_MODELS` (solid models: `RenderModel`, CPU bone evaluation per draw), decals (one draw each). Every draw = `VkRenderState::Apply` (2,000 B `StreamData` copy, push constants, binds). | `di:3722-3837`; `walls:559-592, 333-525`; `flats:403, 279-297`; `models.cpp:724, 4122`; `vk_renderstate.cpp:223-248` | draw | §03 2.5, 4; §04 3.2; §06 2.3 |
| 15d | In-scene extras | Mesh particles and debris (instanced), player sprites drawn inside the scene in VR (`RenderHUDModel` per layer), other players' world weapons, SSAO if on. | `di:4510-4556`; `hw_weapon.cpp:1534, 2586` | eye | §04 2.1, 3.2 |
| 15e | Portals | `EndFrame`: each mirror/line/skybox/stacked-sector portal = stencil passes + a **full `CreateScene`/`RenderBSP` from the BSP root**; sky lines one draw per line per pass. | `di:4562`; `portal.h:187-195`; `portal:252-255` | portal × eye | §03 2.6 |
| 15f | Translucent | `RenderTranslucent`: BSP-style sort of all translucent items (all non-model sprites except solid wall sprites, particles, translucent walls/flats, translucent models); sprites get a **CPU-summed light per draw**; GPU particles (soft particles end and restart the render pass twice); drawn lines; then mounted HUD, hitscan tracers, laser sight (`Trace` per hand per eye), VR wheel. | `di:3855-4031, 4579-4586`; `hw_drawlist.cpp:931`; `hw_spritelight.cpp:101-210` | eye | §04 3.2; §05 2.3 |
| 15g | Post-process | `AdjustHud`, `EndDrawScene`, `PostProcessScene`: scene resolve/blit, `Pass1` (exposure mip chain, custom "beforebloom", fork passes when content exists, bloom 22 draws), `Pass2` (tonemap/colormap/lens/fxaa/custom "scene"). About 34 passes per eye at defaults (inferred), each its own render pass; output targets one array layer; **runs per eye even with multiview**. | `ep:488-507`; `vk_renderdevice.cpp:805`; `common/rendering/hwrenderer/postprocessing/hw_postprocess.cpp:3557-3593`; `vk_pprenderstate.cpp:112, 159` | eye | §05 2.5, 4.6; §07 5 #3 |
| 16 | 2D | Status bar/automap/HUD/console into `twod` (face HUD only when not mounted). | `d_main.cpp:1700-1912` | frame | §01 2.2 |
| 17 | Eye composite | `Update`: per eye `Draw2D` up to twice (each clears stencil = ends the render pass), `AdjustHud`, `FinalizeEyeImage` (XR present pass at recommended size + mirror pass on one eye); `RenderVirtualScreen` on menu frames; end pass. | `vk_renderdevice.cpp:657-726`; VK-XR:7196-7263 | eye | §04 3.2; §07 2.2 |
| 18 | **Fence wait #1** | `WaitForCommands`: final submit, then the CPU waits for **all** GPU work of the frame, polling `vkGetFenceStatus` with `sleep_for(1 ms)` between polls (5 s deadline). [flat: `WaitForCommands(true)` = desktop acquire + `FPSLimit` + present + this wait] | `vk_renderdevice.cpp:733-739` → `vk_commandbuffer.cpp:430-513, 181-221` | frame | §01 2.4, 2.6; §06 1.1 |
| 19 | XR submit | `AcquireXRSwapchain`: `xrAcquireSwapchainImage`, `xrWaitSwapchainImage` (20 ms timeout), own command buffer blits each eye into the swapchain layer, `vkQueueSubmit`, **fence wait #2 (polled)**, `xrReleaseSwapchainImage`, `xrEndFrame` (projection layer, same poses as rendered). | VK-XR:6133-6479 (wait :6348-6371; `xrEndFrame` :6471) | VR frame | §06 2.2; §07 2.2 |
| 20 | Desktop mirror | `AcquireImage` (mirror composition), `WaitForCommands(true)`: `FPSLimit` when vsync is off (no-op at 72 Hz with `vid_maxfps 500`), present, **fence wait #3 (polled)**. | `vk_renderdevice.cpp:745-750`; `textures/vk_framebuffer.cpp:56-145`; `vk_commandbuffer.cpp:446-449` | VR frame | §01 2.4, 5.11 |
| 21 | Frame end | `UpdateGpuStats`, bench check, `EffectsGovernor::EndFrame`, perf logs if on, deferred screenshot, `S_UpdateMusic`. | `vk_renderdevice.cpp:741`; `d_main.cpp:1537-1547, 2016-2018` | frame | §01 2.2 |
| — | Inside 15c-15g | A submit every 1,000 `Apply` calls (no CPU wait; ends and re-begins the render pass); the 9th submit of a frame waits on the fence of the frame's oldest submit; capacity walls force mid-frame full drains (stream buffer ~9,536 Applies, matrix buffer ~49,998 changes, viewpoint buffer at the 99th upload, descriptor-pool exhaustion); >64 MB of queued staging waits on the transfer submits; flat-vertex overflow past 1,999,500 vertices is `I_FatalError`. | `vk_renderstate.cpp:43, 227-232, 390-394, 514-520`; `vk_commandbuffer.cpp:343-372`; `hw_viewpointbuffer.cpp:53-70`; `flatvertices.cpp:139-149` | conditional | §06 4.2, 5 #4-5 |

**Order of the VR calls relative to tic and render** (§01 2.4, §07 2.3): tics run **before** `xrWaitFrame`; the
command a tic consumes carries poses written by the previous frame's SetUp #2; world interpolation uses the
loop-top clock while the head uses `predictedDisplayTime`; there is no late latch. When `BeginXRFrame` returns
false (session not running, no camera), nothing paces the loop except `FPSLimit` (`vid_maxfps 500`) and a desktop
vsync, because `doWait` is false in VR (`d_net.cpp:2357-2359`; the free-run is (inferred)).

**Threads** (§01 2.5). The main thread runs all tics and ZScript, prediction, `NetUpdate`, sound updates, both GC
checks, the scene build in VR, all command recording, all fence waits and all OpenXR calls. Other threads: a
Windows hang watchdog (`common/platform/win32/i_hangwatch.cpp:228-265`); a static 8-thread ctpl `renderPool`
(`bsp:87`) of which flat screen uses one scene worker (spinning on `_mm_pause` while it waits, `bsp:251-266`) and VR
uses none at defaults (`hw_cvars.cpp:278`); up to 2 texture-loader threads and 1 model loader, waking every 3 ms
when idle (`vk_renderdevice.cpp:386-434`); the OpenAL stream thread; optional SolarXR reader and DAP debug server;
library threads of the OpenXR runtime, driver and OpenAL (outside `src/`, (inferred)). There is no sim thread and no
render thread (`renderer:` `r_renderthread` is a stub that resets itself, `renderer:src/rendering/r_snapshot.cpp:33-40`).

---

## 3. What slows it down

### 3.1 Measured costs (M10; flat screen, `renderer` old path, owner's 57-file mod set unless stated)

| Rank | Cost | Measured value | Share | Source |
|---|---|---|---|---|
| 1 | Playsim, whole frame (`think_ms`, Sunder MAP15) | 249.88 ms of a 315.29 ms frame (3.4 fps) | 79.3 % | M10 §1 rr342 |
| 2 | `RunThinkers` stage per tic (`ticprofile`) | Sunder MAP15 130.80 ms/tic; Abandon MAP13 61.38-87.17 ms/tic (rr343), 75.24 (rr348) | Sunder 4.6× the 28.6 ms tic budget; Abandon 2.1-3.0× | M10 §2 |
| 3 | GPU wait (`cpu_syncwait_ms`) | 25.87 ms | 8.2 % | M10 §1 |
| 4 | Actor interpolation reset walk per tic | Sunder 9.02 ms; Abandon 6.49-6.81 ms | — | M10 §2 |
| 5 | `WorldTick` event handlers per tic | Sunder 4.07 ms; Abandon 2.28-3.41 ms | — | M10 §2 |
| 6 | CPU scene build (`cpu_scene_ms`) | 4.35 ms (walls 2.56, clip 1.24, draw-call state 0.93, flats 0.75, sprites 0.62, BSP 0.43) | 1.4 % | M10 §1 |
| 7 | GPU post-process | bloom 3.23 + BloomBoost mod 1.60 + 1.64, exposure 1.38 ms | — | M10 §1 |
| 8 | GPU opaque | 6.15 ms (walls 3.09, flats 1.21, models 0.84) | — | M10 §1 |
| 9 | Busy-wait (`r_ticstability`), **no mods** | 2.30 ms a frame: 12.31 → 10.01 ms, 81.2 → 99.9 fps (+23 %) | 18.7 % of the 12.31 ms frame | M10 §4 rr345/rr346 |
| 10 | Particles/beams stage per tic | Sunder 0.72; Abandon 0.65-0.73 ms | — | M10 §2 |
| 11 | `P_PlayerThink` per tic | 0.06-0.07 ms | — | M10 §2 |

Reading notes. `script_ms` (258.24 ms) and `think_ms` together exceed the frame, so the two clocks overlap and must
not be added (inferred from the numbers). The scene-build split clocks sum to more than `cpu_scene_ms` (6.53 vs 4.35
ms with mods, 5.46 vs 3.80 without); the likely reason is that the flat-screen scene worker's time is clocked in the
split but runs beside the main thread (inferred; §03 2.7, 6.2). On `renderer` flat runs, the sync-wait column falls
back to the flat-mode wait (`renderer` `hw_clock.cpp:163-168`), which covers acquire + `FPSLimit` + present +
fence poll (§06 12), so 25.87 ms is not GPU time alone (inferred). The GPU groups listed in M10 sum to 14.0 ms;
M10 does not list the rest of the GPU work (open, §8 #12). The measured runs used `sim_maxcatchup 2` (at most 2 tics
a frame): `130.80 × 2 ≈ 262 ms` matches the 250 ms of sim per frame (inferred). `main` has no such cap (§2 step 4).

### 3.2 Ranked for the owner's setup: playsim first, then the rest

Each row: cost (measured or code-derived class), mechanism, evidence, section, confidence. "Code-derived" = no
measurement exists; the class is complexity, not milliseconds.

**A. Playsim — the per-thinker work and the actor count**

| # | Item | Cost | Mechanism | Evidence | Section | Conf. |
|---|---|---|---|---|---|---|
| A1 | Mod thinkers by family (Sunder MAP15, one tic, `profilethinkers -t 60`) | **measured**: mk-champions ≈ 25 ms (`champion_CyanController` 7.16 ms/122, `champion_Icon` 4.79/1,646, `champion_Leaves` 2.18/976, `champion_DarkRedController` 2.11/102, `champion_SilverController` 1.53/21, `champion_Aggro*` ≈ 4.3/≈2,900); `RSQTHP_Bar` 16.46 ms/3,622 (one HP-bar actor per monster, 4.5 µs each); Legion of Bones ≈ 11 ms (`SkeletonImp` 4.24/1,724, `PhantomRevenant` 1.88/1,545, `SlayerKnight` 1.86/972, `CacoLich` 1.31/489); `Shifter` 2.23/7,686; crits (`csh_critController`+`csh_Brain`, two thinkers per actor) 1.57/11,332; ambience `IA_Sound` 0.89/2,544. Abandon MAP13 same order: `RSQTHP_Bar` 7.97 ms/1,891, champions controllers+aggro ≈ 12 ms, LoB ≈ 6 ms, `Shifter` 1.96/7,040. The listed Sunder entries sum to ≈ 52 ms; M10 does not itemise the rest of the 130.8 ms stage. | Each mod actor runs a ZScript `Tick` plus its state actions; the mods multiply actors 8,065 → 53,116 thinkers (helpers per monster: HP bar, champion icon/leaves/aggro, two crit thinkers). | M10 §3; `dthinker.cpp:149-183` | M10; §02 5.0, 5.2 | measured |
| A2 | Stock vs mod per-actor tick | **measured**: stock monster/item 0.27-0.47 µs; LoB replacement 2-3 µs (≈ 8×); HP bar 4.5 µs. Whole stock thinker list on Sunder MAP15 ≈ 2.4 ms. | ZScript `Tick` overrides and actions vs native `Tick`. | M10 §3 | M10 | measured |
| A3 | Per-thinker VM call layer | code-derived: T (every thinker, every tic) | `DThinker::CallTick` always dispatches through `VMCallVoid` → `VMValidateSignatureSingle` (three out-of-line checks) → `VMCall` → native thunk, even for native-only thinkers, because `Thinker.Tick` is `virtual native`. Script overrides add 2 `rdtsc`, a JIT prologue and `super.Tick()`. Every `Inventory` actor ticks through the ZScript `Inventory.Tick`; each carried item gets a VM `DoEffect` call per carrier tic. The layer's share of the per-thinker time is not measured. | `dthinker.cpp:1584-1590`; `vm.h:688-714, 1066-1076`; `vmframe.cpp:578-632, 705-752`; `inventory.zs:115-150`; `p_mobj.cpp:4872-4882` | §02 1.3, 5.2 | M (share I) |
| A4 | Three full actor walks per real tic (+1 in netgames) | **measured** for the reset: 9.02 ms/tic at 53,116 thinkers vs 1.25-1.39 ms at 8,065 — ≈ 155-172 ns per thinker in both, i.e. linear in the count (inferred from the two runs). Others code-derived A. | (1) interpolation reset (`ClearInterpolation`, `ClearFOVInterpolation`, `ShiftSurfacePositions`, `IsKindOf` per thinker); (2) `RunThinkers`; (3) the light-recreate scan over all world and client-side actors in client-side logic, `totalTics` times a frame; (4) netgame `WorldStateHash`, 72 B per actor byte-wise FNV-1a. `AActor` is 3,104 B (49 cache lines, `NDEBUG`); the reset touches ~11 lines per actor (inferred); 40,000 actors ≈ 124 MB ≫ the 16 MB L3 (inferred). Doc figure: `sim_prefetch` 0/8/24/48 lines changed nothing. | `p_tick.cpp:727-757`; `dthinker.cpp:318-333, 1617-1729`; `d_net.cpp:1235-1256, 1288`; `actorinlines.h:46-59` | §02 1.2, 4, 5.1, 5.14; M10 §2 | measured + I |
| A5 | Base `AActor::Tick` path, even standing still | code-derived: A | `Affect_Tick` test, `TickBehaviors` TMap walk, per-item `DoEffect` VM call, `P_XYMovement` entry, `CheckPortalTransition`, `UpdateWaterLevel`, `P_ActorOnSpecial3DFloor`, **`UpdateRenderSectorList` every tic for every actor** (its `Pos() != OldRenderPos` gate never closes: `OldRenderPos` is written only at spawn), state countdown. Every `TObjPtr` read goes through a read barrier that loads the pointee and can write the field. | `p_mobj.cpp:4741-5432, 5377, 5716`; `p_secnodes.cpp:375-398`; `dobjgc.h:137-143` | §02 2.4, 4, 5.0, 5.9 | V |
| A6 | State changes and actions | code-derived: S | `SetState` loops 0-tic states, `FState::CallAction` per state; actions with default arguments (`A_Chase`, `A_FaceTarget`, `A_Wander`) heap-copy `DefaultArgs` per call (malloc+free); 2 `rdtsc` per action, 2 per script `VMCall`, unconditional on Windows. A state change of a class with GLDEFS frame lights sets `MF8_RECREATELIGHTS`. | `p_mobj.cpp:1000-1105, 1099-1102`; `info.cpp:148, 177-197`; `stats.h:214-237` | §02 5.10, 5.12 | V (per-state count I) |
| A7 | Monster movement and collision | code-derived: M × steps, B, N² | A blocked chasing monster runs `P_SmartMove` + up to 8 `P_TryWalk` retries, each a `P_TryMove` → `P_CheckPosition` scan of its crowded cells (corpses included, rejected after loading flags); fast small projectiles do `1+speed/(radius−1)` full moves a tic (speed = larger axis component), each with relink (two BSP walks + `P_CreateSecNodeList` twice: gameplay and render radius); `P_ChangeSector` restarts its node loop after each thing (O(n²) per mover tic). | `p_enemy.cpp:535-640, 778-965, 3180-3183`; `p_mobj.cpp:2707-2734`; `p_map.cpp:2617, 3170-3368, 9445-9464`; `p_maputl.cpp:412-534` | §02 5.0, 5.3-5.5 | V code, I frequency |
| A8 | Sight checks | code-derived: per call, ∝ distance | No cache across monsters or tics. In multiplayer `A_DoChase` adds a `P_CheckSight` per chase call and `P_LookForPlayers` tests every player: up to 4× sight work in 4-player co-op (inferred). | `p_enemy.cpp:1845-1989, 3125-3147`; `p_sight.cpp:840-945` | §02 5.6 | I |
| A9 | Catch-up spiral (`main` only) | doc figure: "175 ms tic × a 35-tic catch-up loop = 6.5 s frames on Abandon MAP13" (before `sim_maxcatchup`) | A tic slower than 28.6 ms makes every frame owe more tics than it ran, up to ~35 tics a frame. | `d_net.cpp:1508-1516, 2374-2426` | §01 5.2; §02 5.8 | V mechanism, I dynamics |
| A10 | Sound channel walks (`main` only) | doc figures (on `renderer` before its fix): Universal Entropy `Shifter` × 4,752 = 174 ms/tic; restore walk ~35 ms/frame with 2,914 evicted loops | `CheckSoundLimit` on every `StartSound` and every per-source query walk the global channel list; `UpdateSounds` restores evicted channels on every call. `renderer` has a per-source index and `snd_evictretry`/`snd_evictbudget`; `main` has none. | `s_sound.cpp:843-875, 926-1100, 1316-1392`; `renderer:s_sound.cpp:74, 1335-1414` | §01 5.7; §02 5.7 | V code, doc figures |
| A11 | Prediction serialisation in single player | code-derived: per tic-frame, ∝ pawn + `player_t` + VR side tables | Serialise and deserialise the local pawn, `ViewPos`, `modelData`, all `players[]` and RNG state on every frame that runs tics; no `netgame` test. | `p_user.cpp:2226-2257, 2345-2381` | §01 1.6; §02 5.15 | V code, cost I |
| A12 | `WorldTick` handlers incl. two engine handlers | **measured** total 2.3-4.07 ms/tic (all handlers) | `VRStabilizeSyncHandler` (MAXPLAYERS loop in the VM) and `OtherPlayerTagHandler` are registered for every game; their share is not measured. | `wadsrc/static/mapinfo/common.txt:8` | §08 3.2, 7.4; M10 §2 | measured total |
| A13 | VR player chain | code-derived: per player per tic | `P_LatchVRHands`, `UpdateCanonicalMainHandPose`, torso, gear O(spots), grab (blockmap box + sight per candidate), **two 8192-unit traces** (one per hand with a valid latched pose, no cvar), telemetry; in a netgame all of it re-runs per predicted tic. | `p_user.cpp:98-123, 1994-2037`; `p_vrgrab.cpp:674-702` | §02 5.13; §08 7.4 | V |

**B. The busy-wait** — `TicStabilityWait` (`d_net.cpp:2251-2292, 2431-2436`; §01 1.4, 5.3). **Measured** 2.30 ms a
frame flat, no mods (M10 §4). Mechanism: on every frame with no tic due it spins one core for the last stabilised
tic's `G_Ticker`+`MakeConsistencies` time (≤1 s), so a no-tic frame costs as much CPU as a tic frame; the governor
counts it as frame cost, not pacing, and trims effects for it (`hw_effectsgovernor.cpp:111-112`); no clock isolates it
(it is inside perf_track `FrameMs`/`CostMs`). In VR it uses time `xrWaitFrame` would otherwise block on and adds to
the frame only past the slack (inferred, §01 cross-check). With the owner's mods every measured frame owed tics
(`totalTics > 0`), so the spin does not run there (inferred from `d_net.cpp:2433`). Confidence: V mechanism,
measured flat, VR effect unmeasured.

**C. VR doubling, waits, no CPU/GPU overlap** (no VR measurement exists; M10 §5)

| # | Item | Cost class | Mechanism | Evidence | Section | Conf. |
|---|---|---|---|---|---|---|
| C1 | Whole scene built and drawn once per eye | ×2 of all of §3.2 D-G | `vr_openxr_multiview` defaults false, so `renderSceneThisEye` is true for both eyes: walk, clipping, wall/flat/sprite processing, sort, draw recording, portals, translucency, in-hand weapons, bones, HUD models, laser traces, per-eye light caches (`++gl_dynlight_viewid`) all twice. With multiview on, the scene is built once from the left eye (right-eye-only edge geometry may be clipped, (inferred)). | `hw_vrmodes.cpp:643`; `ep:354, 361, 372`; VK-XR:8608 | §03 6 #1; §04 5 #1; §07 1.2-1.3, 5 #1 | V (2× work), share I |
| C2 | Scene build single-threaded in VR | serial main thread | `multithread = gl_multithread && (!IsVRScene or vr_scene_multithread)`; `vr_scene_multithread` false. The 8 pool threads idle. | `bsp:1438-1439`; `hw_cvars.cpp:278`; `bsp:87` | §03 6 #2; §07 5 #2 | V |
| C3 | Post-process twice at eye resolution, also under multiview | ~34 passes × 2 (inferred count) | `PostProcessScene` per eye; every PP pass has `ViewMask = 0`; exposure adapts twice a frame into one camera texture (inferred). | `ep:398, 501`; `vk_pprenderstate.cpp:112, 159, 171-173`; `hw_postprocess.cpp:2344-2361` | §05 5 GPU 2; §07 5 #3 | V |
| C4 | No frames in flight; three full CPU waits per VR frame | frame ≈ CPU + GPU + 2 extra submit round trips (inferred) | CPU waits all GPU work at the end of every frame; then a separate XR blit submit + CPU fence wait; then (sync mode 1) desktop mirror + present + third wait. The next frame's CPU work starts after all of it. All per-frame CPU→GPU buffers are single-buffered persistent mappings, which is what the full sync keeps correct. | `vk_commandbuffer.cpp:430-513`; `vk_renderdevice.cpp:607-611, 730-751`; VK-XR:6342-6379 | §01 5.1, 5.5; §06 1.1-1.3, 5 #1-2; §07 5 #4 | V structure, I cost |
| C5 | 1 ms poll quantum on each wait | ≥ 1 ms per failed poll, typically 1-2 ms with the 1 ms Windows timer period (inferred) | `RS_WaitFencesPolled`: `vkGetFenceStatus`, then `sleep_for(1 ms)` while unsignalled. `selaco` uses blocking `vkWaitForFences`. | `vk_commandbuffer.cpp:181-221`; `common/platform/win32/i_main.cpp:286`; `selaco:…/vk_commandbuffer.cpp:170, 253` | §01 5.5; §06 5 #12; §07 5 #5 | V code, I size |
| C6 | Extra fullscreen passes | per frame: XR present pass ×2 (recommended size, not render-scaled), mirror present pass ×1, swapchain blit ×2, desktop blit ×1-2; custom "screen" shaders run on every present pass (3× a frame with the mirror, (inferred) double application on the mirrored eye) | Nothing renders into the OpenXR swapchain directly; it is a blit target. | VK-XR:6262-6321, 7196-7263; `vk_postprocess.cpp:502-509`; `vk_framebuffer.cpp:86-111` | §07 1.7, 4, 5 #7 | V |
| C7 | Double SetUp | per frame: 2 `xrSyncActions`, 2 hand-locate sets (first at last frame's time), 3 `updateHmdPose`, 2 single-player render-side `P_XYMovement`s, each re-run in the next tic by `PredictRedo`; ≥3 haptics passes; `GRIPCHECK` double grip matrix and two ungated once-a-second log lines | SetUp is called from `BeginFrame` and again from `RenderViewpoint`. | VK-XR:3575-3655, 3906-5883, 5864-5876, 7927-7956; `vk_renderdevice.cpp:1458`; `ep:327`; `playsim/p_vrdemo.cpp:494-548, 509-510, 623-635` | §01 1.7, 5.6; §07 1.8, 5 #9 | V presence, I size |

**D. Scene build** (measured flat: 3.80 ms no mods, 4.35 ms with mods, M10 §1)

| # | Item | Cost | Mechanism | Evidence | Section | Conf. |
|---|---|---|---|---|---|---|
| D1 | Walls | **measured** 2.30 / 2.56 ms flat (largest split) | No persistent raster wall geometry on `main`: every scene emits each visible linedef side as fresh `HWWall`s (~340 B copied into the arena, again into portal lists); on Vulkan the vertices and light lists are built at **draw** time on the main thread. Doc figure (renderer): "the cost of a wall is the draw-list hand-off, not `Process`". The map-load `DoomLevelMesh` feeds only the ray tracer. | `walls:2419, 827-898, 559-568`; `hw_walls_vertex.cpp:279`; `hw_drawlistadd.cpp:33-58`; `maploader/maploader.cpp:3251` | §03 1.2, 6 #3 | measured + V |
| D2 | Clipper and walk | **measured** clip 1.27 / 1.24, BSP 0.48 / 0.43 ms flat | 1-D angular range list, linear scans per node/seg; walk single-threaded in every mode; in VR (serial) the `Clip` clock also contains `Process`. Per-subsector and per-seg VR clock calls (6 per visible subsector, 2 per seg). | `bsp:517-749, 1193-1399`; `hw_clipper.cpp:121-146` | §03 3, 6.2 | measured + V |
| D3 | Draw-list submission state | **measured** draw-call state (`Apply`) 0.67 / 0.93 ms flat; doc figure 5.0-6.0 ms on Abandon MAP13 with the owner's list | One draw per wall part, section plane, sprite, particle, decal and model mesh, each with ~15-25 setters and a full `Apply`; walk order (texture sort off) → material change on most draws (inferred). | `vk_renderstate.cpp:223-248`; `di:74, 3798` | §03 4, 6 #3; §06 5 #3 | measured + I |
| D4 | Flats, sprites | **measured** flats 0.60 / 0.75, sprites 0.14 / 0.62 ms flat | Flats from a static VBO (z patched when a reached plane moved), one `DrawIndexed` per section plane; sprites: visibility = "sector reached" + distance culls (2000 all, 1200 decor), no per-sprite frustum test, no batching, all non-model sprites in the sorted translucent list. | `flats:672`; `hw_vertexbuilder.cpp:406-477`; `bsp:914-1036`; `hw_drawlistadd.cpp:136-151` | §03 1.2; §04 1.1-1.4 | measured + V |
| D5 | Portals | code-derived: full walk per scene portal per eye | Each portal runs `CreateScene`/`RenderBSP` from the root, clipped by a silhouette; sky lines one draw per line per pass. | `portal.h:187-195`; `portal:252-255` | §03 6 #5 | V |
| D6 | Uniform blocks per scene | code-derived | ~27 KB `HWViewpointUniforms` copied 2-3× per `SetupView` and uploaded per `ApplyViewpoint`; each translucent flat uploads twice; ~800 lines of fork uniform fill per `StartScene` (fixed 128/64/32 loops; fog-light walk when the level asks). | `di:2066-2433, 3336-3372`; `flats:541-545`; `hw_viewpointbuffer.cpp:109-118` | §03 6 #6-7; §06 4.1 | V code, I size |
| D7 | Models and bones | code-derived: M × bones × eyes | IQM bones evaluated on the CPU per model per draw (no frame cache, one shared scratch palette per model), `VRAvatar_Pose` per avatar draw (per eye, (inferred)), placement/fade cvars looked up by formatted name per draw, VM `AnimateBones` and `ModifyBobLayer*` from the draw. 80,000-bone buffer; overflow draws unskinned. | `models.cpp:662-722, 4122-4241`; `models_iqm.cpp:823-965`; `hw_bonebuffer.cpp:27, 71-100`; `hw_sprites.cpp:501-507`; `hw_weapon.cpp:2412, 2518, 2531` | §04 5 #5, #8 | V (avatar I) |
| D8 | Translucent sort | code-derived: O(n log n), O(n·k) with k translucent planes/walls | On the render thread, not separately clocked on `main`. | `hw_drawlist.cpp:629-708, 931-948` | §04 5 #6 | V |

**E. Lights** (code-derived; measured counts only: 56 dynamic lights without mods, 122 with, M10 §1)

| # | Item | Mechanism | Evidence | Section | Conf. |
|---|---|---|---|---|---|
| E1 | Per-tic relink of moving lights | A light relinks when X/Y or its link radius changes (point/sector lights use 2 × current intensity, so a sector light relinks on every light-level change; yaw-rotated offsets; pose-anchored hand lights every tic); relink = unlink from every side/section map + 2-D flood with `DistToSeg` per seg/side + one heap `FLightNode` per touched side/section. A new light links twice in its first tic. Cost ∝ moved lights × touched area. | `a_dynlight.cpp:377-439, 455-727` | §05 1.1, 5 CPU 1 | V |
| E2 | Per-eye draw-time light lists for walls and flats | Every lit surface, every eye: iterate the side/section TMap (deactivated lights included), two distance-cull calls per light, plane test, insertion sort to budget (8/16), 64 B per light into one 80,000-light SSBO. Walls with decals do it twice. | `walls:600-806, 879-884`; `flats:157-271`; `hw_lightbuffer.cpp:81-126` | §05 1.2, 5 CPU 2 | V |
| E3 | Sprites, particles, models | One CPU-summed light per sprite/particle draw (probe search + section map walk; CPU shadow `RayTest` only when the light's shadow switch is on — none at defaults). Models: BSP circle walk, the section revisit guard is never armed, `std::find` dedupe, no candidate budget. | `hw_spritelight.cpp:101-321, 253, 259`; `hw_shadowmap.cpp:154-160` | §04 5 #4; §05 5 CPU 3 | V |
| E4 | Whole-list O(L) walks per frame | `DynamicLightShadowRowsWanted`, `CollectLights`, `SyncViewLights` (per eye), `FillFogDynLights` (per draw info), `SmokeVolume::GatherLights`; no spatial structure. | `hw_dynlightdata.cpp:210`; `ep:78`; `di:671, 1783`; `hw_smokevolume.cpp:1348` | §05 5 CPU 4 | V |
| E5 | Fragment light loops, twice in VR | Three loops over the surface's light ranges per pixel; normal material capped at 17 lights per class by `gl_light_range_limit`, PBR/specular uncapped. | `material_normal.fp:18-146` | §05 5 GPU 1 | V code, I size |

**F. Submission** (§06)

| # | Item | Mechanism | Evidence | Section | Conf. |
|---|---|---|---|---|---|
| F1 | Per-draw CPU work in `Apply` | 2,000 B `StreamData` memcpy into host-visible memory every draw (32 draws per 64 KB block), 64 B push constants every draw, pipeline key build + memcmp, two descriptor binds per `SetMaterial` (no same-material check), two every 32 draws; 2-13 vkCmd calls per draw. | `vk_renderstate.cpp:223-512`; `hw_renderstate.h:1095-1105`; `vk_streambuffer.cpp:34-48` | §06 1.4, 5 #3, 10 | V |
| F2 | Submit chopping and capacity walls | Submit every 1,000 Applies (render-pass end + LOAD re-begin); fence wait at the 9th submit; forced full drains at the stream/matrix/viewpoint/descriptor walls (§2 table, last row). No counter exists for any of them. | `vk_renderstate.cpp:43, 227-232, 514-520`; `vk_commandbuffer.cpp:343-372` | §06 5 #4-5, 12 | V code, I whether hit |
| F3 | Lazy pipelines and textures inside the frame | Pipelines compiled on first use; any change of scene size/samples/layers drops every pipeline and render pass (VR menu transitions change size, (inferred)); wall/flat textures created synchronously on first draw. | `vk_renderpass.cpp:120-124, 352-363`; `vk_renderbuffers.cpp:192-196`; `vk_hwtexture.cpp:93-175, 643-699` | §06 5 #7-8 | V |
| F4 | Fork diagnostics on every GPU group | `vk_gpu_checkpoints 1`: `vkCmdSetCheckpointNV` per Push/PopGroup; two `std::string` builds + two `std::set` lookups per `PushGroup` even with `stat gpu` off. | `vk_commandbuffer.cpp:237-269, 522-556`; `vk_renderdevice.cpp:114, 306` | §06 5 #12; §08 7.2 | V path, I size |

**G. Post-process and fork shader cost**

| # | Item | Cost | Mechanism | Evidence | Section | Conf. |
|---|---|---|---|---|---|---|
| G1 | Bloom + exposure chain | **measured** flat with mods: bloom 3.23, exposure 1.38, BloomBoost mod 1.60 + 1.64 ms | ~34 passes per eye at defaults (inferred count: exposure ≈ 12 at ~11 mip levels + bloom 22), each its own render pass with barriers and a new descriptor set. | `hw_postprocess.cpp:1851-1912, 2286-2429`; `vk_pprenderstate.cpp:54-121` | §05 4.6, 5 GPU 2 | measured + I |
| G2 | `main.fp` uber-shader | code-derived: branches + register pressure on every world fragment | 5,128 lines; every Vulkan `main.fp` program compiled with `SURFACE_DAMAGE` + `EFFECT_LIGHTS` and the stamp and outline libraries prepended; fork features uniform-gated. | `vk_shader.cpp:1171-1192` | §08 7.1; §05 5 GPU 6 | I |
| G3 | Soft GPU particles | code-derived: 2 render-pass breaks per eye per frame | `r_gpuparticles_soft` defaults to 8.0; once any GPU particle has been written on the level (`GpuParticleWritten` is only zeroed at map change/load), each main-view scene ends and restarts its pass twice. | `di:3890-3933, 3989-3994`; `hw_cvars.cpp:311`; `g_levellocals.h:3092-3101` | §04 5 #7; §08 7.3 | V |

### 3.3 Vanilla frame (no mods) — the second view (M10 §1 rr341, Sunder MAP15, flat, uncapped)

| Part | ms | % of 13.89 ms frame | Notes |
|---|---|---|---|
| Frame | 13.89 (66.7 fps) | 100 | 8,065 thinkers, 52 sprites, 56 dynamic lights |
| Playsim (`think_ms`) | 2.52 | 18.1 | per tic: thinkers 3.24-3.79 ms, interpolation reset 1.25-1.39 ms, WorldTick 0.01, player 0.03 (M10 §2 rr344/rr349) |
| Script (`r_script_ms`) | 3.14 | 22.6 | governor attribution; overlaps other clocks (inferred) |
| CPU scene build (`cpu_scene_ms`) | 3.80 | 27.4 | walls 2.30, clip 1.27, draw-call state 0.67, flats 0.60, BSP 0.48, sprites 0.14 (splits include worker time, (inferred)) |
| GPU wait | 3.24 | 23.3 | includes present-side waits on the flat path (inferred, §3.1 notes) |
| Busy-wait | 2.30 (other run) | — | M10 §4: 12.31 → 10.01 ms with `r_ticstability 0` |

In the vanilla frame the four buckets are of the same order; with the owner's mods the playsim is ~79 % and the
scene build ~1.4 % (§3.1).

---

## 4. What the engine does not do that a modern VR engine does

| Not done | Evidence (file:line) | Section |
|---|---|---|
| Multiview is off by default; when on, post-process still runs per eye (no layered PP), the scene is culled from the left eye only, and ~10 sites re-decide it per frame | `hw_vrmodes.cpp:643`; VK-XR:8600-8616; `vk_pprenderstate.cpp:112, 159`; `ep:354, 452-482` | §07 1.3 |
| No frames in flight: CPU frame N+1 never overlaps GPU frame N | `vk_commandbuffer.cpp:430-513`; `vk_renderdevice.cpp:735, 739` | §06 1.1 |
| Per-frame CPU→GPU data single-buffered (stream, matrices, viewpoints, lights, bones, flat vertices, compute host lists) | `vk_renderdevice.cpp:607-611`; `hw_viewpointbuffer.h:45`; `flatvertices.h:115`; `hw_lightbuffer.h:49`; `hw_bonebuffer.h:44` | §06 1.2 |
| No indirect draws: `multiDrawIndirect` is required but never used; the ZVulkan `drawIndirect`/`drawIndexedIndirect` wrappers have no caller; `drawIndirectCount` and `shaderDrawParameters` not enabled | `vulkanbuilders.cpp:1917-1925`; `vulkanobjects.h:391-392, 780-788` | §06 6.1, 7 |
| No bindless / descriptor indexing: `OptionalDescriptorIndexing` never called (enabled only if the OpenXR runtime's required extension list names it, (inferred)); per-material set 2 | `vulkanbuilders.cpp:1834, 1958-1962`; `vulkandevice.cpp:167-171`; `vk_descriptorset.cpp:453-470` | §06 1.6, 4.4 |
| Per-draw data through a dynamic-offset UBO + push-constant index (2,000 B per draw); no per-draw SSBO, no `gl_DrawID` | `vk_renderstate.cpp:379-395`; `vk_shader.cpp:984-1040` | §06 11 |
| No GPU culling; occlusion is the CPU 1-D angular clipper; no per-sprite frustum test | `bsp:1361-1399`; `hw_clipper.cpp:121`; `bsp:914-1003` | §03 1.1; §04 1.1 |
| No persistent raster wall geometry on `main`; walls rebuilt per scene | `hw_walls_vertex.cpp:279`; `walls:559-568` | §03 1.2 |
| Scene build single-threaded in VR; the walk itself single-threaded in every mode | `bsp:1438-1439`; `hw_cvars.cpp:278` | §03 2.7, 6 #2 |
| No sim thread, no render thread; sim and render serial in one loop | `d_main.cpp:1966-2018`; `renderer:src/rendering/r_snapshot.cpp:33-40` | §01 1.1; §09 B.9 |
| No batching or instancing of sprites, particles, decals, models (only GPU particle ring, debris and mesh particles are instanced/one-draw) | `hw_sprites.cpp:476`; `hw_decal.cpp:106`; `models_iqm.cpp:567`; `hw_meshparticles.cpp:529` | §04 1.3 |
| No clustered/tiled lighting for `FDynamicLight`: per-surface CPU lists per eye (the fork's effect lights are binned, ≤32 per pixel) | `walls:600`; `flats:157`; `hw_effectlightcore.h:440-636` | §05 1.2, 1.6, 4.5 |
| No foveated rendering, no VRS/fragment shading rate, no eye tracking, no visibility mask, no depth composition layer, no space warp/motion vectors, no quad views | grep absence over `src/` (§07 6.4); `hw_clock.cpp:350` prints `vr_openxr_foveation=unsupported` as a literal | §07 1.9, 5 #12 |
| Nothing renders into the OpenXR swapchain; eyes are blitted in a second submit that the CPU waits on | VK-XR:6262-6379 | §07 4, 5 #4 |
| No late latch; world interpolation from the loop-top clock, not the predicted display time | VK-XR:6079-6087; `i_time.cpp:90`; `r_utility.cpp:1095`; `ep:672` | §01 1.8; §07 2.3 |
| Eye placement from `vr_ipd` (0.064), not the runtime eye poses; per-eye runtime orientation unused in the scene | VK-XR:1951, 1967, 1975-1981 | §07 3, 6.1 |
| Blocking/timeline-semaphore waits not used; fence waits are 1 ms sleep polls | `vk_commandbuffer.cpp:181-221` | §06 5 #12 |
| No async compute queue; all rendering, compute and the XR blit on the graphics queue | `vulkanbuilders.cpp:2003-2022`; §06 6.2 | §06 6.2 |
| No SPIR-V cache; scene shaders compiled serially on the main thread at startup (164 + 4U glslang compiles, (inferred)); PP/compute shaders and pipelines compiled lazily inside frames | `d_main.cpp:4694`; `vulkanbuilders.cpp:251-311`; `vk_renderpass.cpp:352-363` | §06 1.7-1.8, 9 |
| MSAA resolve is a separate transfer per eye; all four scene attachments always allocated | `vk_postprocess.cpp:162-181`; `vk_renderbuffers.cpp:274-302` | §06 4.5 |
| No texture/material sort of opaque draws by default | `di:74, 3798` | §03 4 |

---

## 5. Bugs found during the teardown (correctness, not cost)

Branch: M = `main`, R = `renderer`, N = `netplay`, S = `selaco` (as checked by the section). "V code / I effect" =
the code path was read, the visible or gameplay effect is (inferred).

### 5.1 Rendering and VR

| # | Bug | Effect | file:line | Section | Confidence | Branch |
|---|---|---|---|---|---|---|
| R1 | Fork line distance cull (`gl_line_distance_cull 4000`) measures from the global `r_viewpoint.Pos`, not the scene's viewpoint | Skybox and camera-texture scenes cull by the player's position (camera textures by the previous frame's) | `bsp:457-458` | §03 B1 | V code / I effect | M R N S |
| R2 | Distance-culled segs bypass the per-linedef dedupe | One full-linedef impostor per far seg, coplanar over the real wall when some segs are near (z-fight) | `bsp:632-655` vs `:716-721` | §03 B2 | V code / I effect | M R N S |
| R3 | Impostors are opaque full-height walls with the front sector's **ceiling** flat (or sky parts), add a clip range, get no `ML_MAPPED`; `gl_render_walls 0` does not stop them | Wrong texture; geometry behind hidden | `walls:2563-2580`; `bsp:632, 653, 711, 723` | §03 B3-B4 | V | M R N S |
| R4 | Impostors draw with an uninitialised light level (`HWWall` has no constructor; the culled branch never sets `lightlevel`/`rellight`) | Indeterminate brightness | `walls:2563-2580, 854, 336, 469-470`; `hw_drawstructs.h:143-235` | §03 B12 | V code / I effect | M (R N S share the cull code, §03 10) |
| R5 | `IsDistanceCulled` tests only the seg's two endpoints | A long seg passing near the viewer is culled and its clip range hides near geometry | `bsp:451-462` | §03 B13 | V code / I frequency | M R N S |
| R6 | Heat grid decays by `screen->FrameTime` (ms since the first frame, not the frame delta), once per scene; ZScript reads it back via clearscope `HeatmapAt` | Heat empties almost at once after long uptime; value depends on uptime, eye count, portals, frame rate; render→script path (netplay hazard) | `di:3247-3261`; `scripting/vmthunks.cpp:5618-5629`; `doombase.zs:2098` | §03 B5; §08 7.8 | V code / I effect | M |
| R7 | Billboard `drawPos` written per scene (portals, second eye) and read by ZScript hit-tests for view/hand-locked billboards | Script sees the last scene's value; client-view-dependent | `di:3644`; `scripting/vmthunks.cpp:3076` | §03 B6 | V code / I consequence | M |
| R8 | Dither targets are never collected in single-thread mode (VR default): `RenderThings` stamps `validcount` first; in MT the same field races with the worker | Dormant feature broken (needs `r_dithertransparency` + out-of-bounds camera) | `bsp:923-924, 1088-1097` | §03 B11; §04 6 | V | M R N S |
| R9 | `RenderJobQueue`/`WallWorkQueue` (300,000 entries) have no bounds check | Overflow writes past static arrays | `bsp:140-147, 174-181` | §03 B9 | V | M R N |
| R10 | `AllocVertices` reads `GetBuffer()` before `fetch_add` | Stale pointer if two threads allocate (portal/sky parts on the scene worker) | `flatvertices.cpp:141-142` | §03 B10 | I | M R N |
| R11 | Wall workers can run `RecalcVertexHeights` / lazy texture data for shared vertices concurrently | Data race (dormant: `gl_bsp_worker_threads` 1) | `walls:1951, 2504-2505`; `bsp:280-311` | §03 B8 | I | M R N |
| R12 | `HWFlat::Process` writes `dithertransflag` into model sector planes from the scene worker | Render writes level state | `flats:592-593` | §03 B7 | V | M R N |
| R13 | `hw_GetDynModelLight`'s section revisit guard is never armed (`section->validcount` never set) | Section light map re-walked per subsector; duplicates caught only by `std::find` | `hw_spritelight.cpp:253, 259` | §04 3.1; §05 6 | V | M R N |
| R14 | Sprite distance cull and light distance cull use the global `r_viewpoint.Pos` | Wrong distances inside portals/mirrors; light cull not per eye | `hw_sprites.cpp:1050`; `hw_dynlightdata.h:103-128` | §04 5 #11; §05 3 | V code / I effect | M R N |
| R15 | Light buffer overflow (80,000 lights a frame): the cursor has already advanced, so every later upload that frame fails | All later surfaces that frame drawn unlit | `hw_lightbuffer.cpp:81` | §05 3 | V | M R N |
| R16 | Deactivated lights stay linked (`Deactivate` only clears `m_active`; `Tick` returns before `UpdateLocation`) | Every reader iterates them; on `renderer` they still evict static surfaces | `a_dynlight.h:293`; `a_dynlight.cpp:279, 792-795` | §05 4.1, 1.10 | V | M R N |
| R17 | `vr_voxel_debug` defaults **true**; its throttle is one static last-actor per site | With two held voxel actors (two hands) it prints on every draw, per eye | `r_data/models.cpp:600, 1778-1803, 5609-5625` | §04 6 | V | M |
| R18 | Scene eye offset from `vr_ipd` and the centred head axis, while the submitted layer uses the runtime eye poses | Stereo mismatch when the user's IPD ≠ 0.064 m (comfort) | VK-XR:1967, 1975-1981, 6399-6410 | §07 3, 6.1, 8 Q7 | V code / I effect | M R N S |
| R19 | `DrawPresentTextureToImage` runs custom "screen" PP shaders on every call (XR present ×2 + mirror) | A mod's "screen" shader applied twice on the mirrored eye | `vk_postprocess.cpp:502-509` | §07 3 | I | M |
| R20 | Multiview decision split: buffers sized from SetUp #1's mode, eye loop decides from SetUp #2's; `ep:354` does not re-check layer counts | One frame with a postprocess-only eye 1 over single-layer buffers on a mode change | `ep:354`; `vk_renderdevice.cpp:1490-1491` | §07 1.3 | I | M |
| R21 | `xrVRFrameIndex` advances twice a frame while `FVRRig::Get` keys its once-per-frame cache on it | Up to 2 rig recomputes a frame | VK-XR:3581; `r_data/vr_rig.cpp:607-630` | §01 5.6; §07 3 | V code / I count | M R N |
| R22 | SetUp #1 locates hands at the previous frame's predicted display time, posts input and moves the pawn before `xrWaitFrame` | Stale hand pose in that pass | VK-XR:3913, 4158, 4362 | §01 1.7; §07 2.3 | V | M R N S |
| R23 | `PollXREvents` dequeues one event per call | Event backlog after bursts | VK-XR:3662-3665 | §01 3 | V code / I effect | M |
| R24 | Single-player render-side room-scale `P_XYMovement` per SetUp is queued and made again in the next tic (`PredictRedo`) | Whether room motion is applied once or twice is open (§8 #13) | `playsim/p_vrdemo.cpp:494-548, 509-510, 623-635`; `g_game.cpp:1995` | §01 1.7, 8 Q5 | V code / effect open | M |
| R25 | Each camera texture's mono `VRMode::SetUp` writes `PlayInVR=false` and the head-height attack pose in single player until SetUp #2 restores them | Render writes sim fields; harmless by ordering (inferred) | `hw_vrmodes.cpp:1481-1556`; `ep:727` | §01 1.7; §07 3 | V code / I effect | M |
| R26 | Equirect backdrop layer built every menu frame (when available) but never put in `layers[]` | Wasted work; backdrop never shown | VK-XR:6423-6461, 6748-6762 | §07 6.2 | V | M |
| R27 | Main loop free-runs at `vid_maxfps` when `BeginXRFrame` returns false in VR | Up to 500 Hz loop without a session | `d_net.cpp:2357-2359`; `v_video.cpp:75` | §01 2.4 | I | M |
| R28 | Renderer's set-1 layout needs 13 vertex-stage storage buffers | Pipeline layout creation fails on devices with fewer (not the owner's GPU) | `renderer` `vk_descriptorset.cpp` | §09 A.2.3 | V | R |

### 5.2 Playsim and netplay

| # | Bug | Effect | file:line | Section | Confidence | Branch |
|---|---|---|---|---|---|---|
| P1 | `UpdateRenderSectorList` gate never closes: `OldRenderPos` is written only at spawn (FLT_MAX) | Every actor rebuilds its render line/sector-portal lists every tic; near line portals it loops all `linePortals` and allocates | `p_secnodes.cpp:378-398`; `p_mobj.cpp:5716` | §02 5.9; §04 4 | V | M R N S (upstream) |
| P2 | `static previous_health` in `P_PlayerThink` shared by all players | Haptics fire for every player's health in multiplayer, and again inside prediction | `p_user.cpp:1933-1946` | §02 6 | V | M |
| P3 | Renderer calls `P_CheckSight`, which can draw `pr_checksight` | Frame-rate RNG draw from a playsim stream (dormant: dither + out-of-bounds view only) | `bsp:1546-1552`; `p_sight.cpp:873-883` | §02 5.B | V | M R N |
| P4 | Teleport command burst: the writer is never called, the reader always returns at the length test | `VR_ApplyTeleportBurstMove` runs per command build with no effect | `QzDoom/qzdoom_common.cpp:84, 219-229`; `g_game.cpp:141-146` | §08 8.3 | V | M R N |
| P5 | `compat_oldrandom` sets `COMPATF2_OLD_RANDOM_GENERATOR`, which nothing tests | Compatibility-menu option does nothing | `doomdef.h:244`; `d_main.cpp:918, 928, 1024`; `menudef.txt:4196` | §08 8.3-8.4 | V | M |
| P6 | `netplay`: analog/net values written into the `xrLast*` edge-detector fields before the edge comparison | Trigger edges lost (the bug `main` fixed) | `netplay:` VK-XR:3954-3961 | §07 7; §01 7 | V | N |
| P7 | `netplay`: `DrawLaserSightWorld` stores laser trace targets into the player actor from the render trace | Render writes sim state | `netplay` `hw_weapon.cpp` diff; main moved it to the sim (`hw_weapon.cpp:1458-1460`) | §04 7 | V | N |
| P8 | Single-player VR frame code writes pawn fields and runs `P_XYMovement`; `UpdateCanonicalMainHandPose` reads live device poses inside the tic | Sim input does not all enter through `usercmd_t` in single player | `vk_openxrdevice.cpp:5603-5617, 5831-5850`; `p_user.cpp:170-191` → `hw_vrmodes.cpp:1644-1682` | §02 2.7, 5.B | V | M |

### 5.3 Instruments (wrong numbers, not wrong game behaviour)

| # | Bug | file:line | Section | Branch |
|---|---|---|---|---|
| I1 | `simprofile` prints negative `whole`/`ticker` totals (its `cycle_t` accumulation is broken for those two; clocks are open while the CCMD prints) | `renderer:src/d_net.cpp:191-205, 2375-2573` | M10 header; §01 6.1 | R |
| I2 | Summary `Scene` figure (used by `stat rendertimes`, `r_perflog`, `perf_track`) double-counts `VRScenePostBSP` and `VRPlayerSprites`, which run inside `VRSceneBuild` | `hw_clock.cpp:157`; `di:3683-3710`; `bsp:1560` | §03 6.2; §07 8 | M R N |
| I3 | `perf_track` CPU columns read 0 unless `glcycle_t::active` (`stat rendertimes`, `printstats` or `r_perflog`); perf_track `scene` is 0 in flat runs on `main` | `hw_clock.cpp:153-163, 444`; `hw_perftrack.cpp:526` | §01 6.1; §03 6.2 | M N |
| I4 | `stat sight`: early returns skip `SightCycles.Unclock()`; render-path calls land in the next tic | `p_sight.cpp:842-853` | §02 6 | M R N S |
| I5 | `stat ACS` reset in every `DACSThinker::Tick` (shows whichever ACS thinker ticked last); `ActionCycles` double-counts nested actions and is not unclocked on exceptions; `stat VM` rotates per frame, nested calls double-count | `p_acs.cpp:3454`; `info.cpp:205-227`; `vmframe.cpp:896-912` | §02 6 | M R N S |
| I6 | `SetupSprite` double-counts the particle path (nested clock) | `bsp:374-377, 1008, 1054` | §04 5 #11 | M R N |
| I7 | `stat lightstats` link counters reset after the tics ran, so they show only render-time relinks | `ep:669`; `hw_clock.cpp:139-143` | §05 5 CPU 1 | M R N |
| I8 | `render_vertexsplit` never incremented; `WallWorkersElapsed` never clocked (read as an always-0 fallback); `rendered_lines` skips distance-culled walls in single-thread mode | `hw_clock.cpp:70, 124, 174`; `bsp:648-652` | §03 6.3 | M R N |
| I9 | `VRSubmitWait` sums `xrWaitSwapchainImage` and the XR blit fence poll; the third (mirror) wait is only in `Finish`, not in `VRRenderSyncWait`/`cpu_syncwait`; `xrAcquireSwapchainImage` and `TicStabilityWait` are counted as frame cost, not pacing | VK-XR:6187, 6369; `vk_renderdevice.cpp:747-749`; `hw_clock.cpp:162` | §01 6.1; §06 12; §07 8 | M |

### 5.4 Static path on `renderer` (default off)

| # | Hazard | file:line | Section | Confidence |
|---|---|---|---|---|
| S1 | Camera textures are not excluded from `LevelResidency_BeginScene`; they can take one of the two per-frame index regions (a VR eye then draws the whole mesh or none) and set the frame's light key (a main view with a different key is refused) | `renderer` `hw_levelresidency.cpp:4221-4283`; `hw_levelbuffers.h:78` | §09 B.8.1 | I from V code |
| S2 | Eviction flags are frame-global while decisions are per scene | `:4155-4187` | §09 B.8.2 | I (low) |
| S3 | Light patch duplicates `Process`'s light statements with an observed /fp:fast rounding form; only `r_levelmesh_verify` detects drift | `hw_walls.cpp:2416-2461`; `hw_levelresidency.cpp:1358-1404` | §09 B.8.3 | V |
| S4 | Flat-VBO `lu` write relies on the old path writing only `z`; surface id rides in a float (`MAX_SURFACES = 1<<24`); `FoldBack` uses a function-static scratch | `:2059-2086, 409, 3653` | §09 B.8.4-7 | V / I |

---

## 6. Dead and dormant code

Long tables stay in §08 8.1-8.9 (all 623 fork cvars with status, all 58 dead functions, all 459 natives). Counts
and the items that matter are here.

### 6.1 Counts (main, §08 1.3-1.6, 8.4)

| Kind | Count | Notes |
|---|---|---|
| Fork functions defined and never called | 58 (+ `P_GuardBlocks` transitively) | incl. 4 virtual hooks declared, overridden, never invoked |
| Fork cvars DEAD | 6 | no reader anywhere: `r_transparentPlayerSprites` (`hw_weapon.cpp:124`), `vr_missile_haptic_level` (`hw_vrmodes.cpp:1085`); **menu-only** (a menu row, no engine or script read): `compat_oldrandom`, `vr_openxr_eye_shift_scale` (`hw_vrmodes.cpp:639`), `vr_openxr_force_recommended_viewport` (:646), `vr_openxr_use_screen_viewport_for_submit` (:652), the last three in Developer Options (`menudef.txt:2235-2237`) |
| Fork cvars DORMANT (read only by GL/OpenVR code) | 7 | `vr_two_handed_weapons`, `vr_openvr_menu_pointer_tip_offset`, `vr_openvr_menu_pointer_pitch_bias`, `gl_no_clip_planes`, `gl_no_ssbo`, `gl_no_persistent_buffer`, `gl_storage_buffer_type` (after `IsVulkan() ||`, `common/rendering/v_video.h:228`) |
| Callback-only / startup-only fork cvars | 1 / 7 | `r_smoke_preset`; `gl_texture_thread_workers`, `gl_texture_thread_upload`, `vk_max_transfer_threads`, `vk_robust_buffer_access`, `vk_gpu_checkpoints`, `vr_eye_colorspace`, `r_gpuparticles_ringsize` |
| Debug/diag/test/fake fork cvars | 84 (72 in R) | alive, switch prints/tests only |
| Fork natives without an in-engine-pk3 caller | ~429 of 459 | external mods (RS/BD22) cannot be checked from these trees |
| Never-compiled source files | 12 | `posix/nosdl/*` (10), `gl/stereo3d/gl_openxrdevice.cpp/.h` (+ `xlat_parser.out`, `win32/QZDoomVR.bat`) |
| `#if 0` blocks in fork code | 4 | `gl_openvr.cpp:2357-2362, 3413-3424`; `hw_vrmodes.cpp:1218-1220, 1230-1232` |
| QuestZDoom `__MOBILE__`/`__ANDROID__` blocks in compiled files | 42 (unconfirmed; a directive count gives 57 in 28 files) | mostly GL backend, `oalsound.cpp`, `p_interaction.cpp:1482-1542` |

### 6.2 Dead stock or engine items that matter

| Item | Status | Evidence | Section |
|---|---|---|---|
| Coronas: `DrawCorona`, `DrawCoronas`, the `Coronas` member, `gl_coronas` (default true) | DEAD | `di:4060-4125, 4300-4327` `#if 0`; call commented `di:4356-4359`; `hw_sprites.cpp:1079-1085`; `hw_drawinfo.h:186`; menu row commented `menudef.txt:3198` | §03 9; §04 6; §05 6 |
| `gl_weapon_purelightlevel` | DEAD (made dead by the fork; stock read it twice) | `hw_weapon.cpp:141` | §04 6; §08 8.3 |
| `gl_lights` (deprecated, default changed to false), `gl_render_segs` (`EXTERN_CVAR` only), `cl_noprediction` (no reader; only an `EXTERN` in a never-compiled file), `singletics` (never assigned) | DEAD | `g_cvars.cpp:63`; `bsp:413`; `p_user.cpp:297`; `d_main.cpp:587` | §05 6; §03 9; §02 6; §01 6 |
| `RenderFrame` event and its call | DEAD (both commented out) | `events.cpp:2132-2144`; `d_main.cpp:1689-1690` | §01 6; §02 6 |
| `D_DoAnonStats` / stats thread | DEAD (empty under `NO_SEND_STATS`) | `d_anonstats.cpp:17-27` | §01 6 |
| `VKOpenXRDeviceMode::SubmitFrame`, `VKOpenXRDeviceMode::Present` (Vulkan), `GetControllerProfile`, `VulkanRenderDevice::CacheProgress`, `VRMode::SupportsMultiview`, `VkRenderState::DrawInstanced` (the indexed instanced draw is live: `hw_meshparticles.cpp:529`, `hw_debrispool.cpp:1324`) | DEAD hooks | VK-XR:5997, 6482-6486, 7981; `vk_renderdevice.cpp:1026`; `hw_vrmodes.h:197`; `vk_renderstate.cpp:80` | §01 6; §07 6.2; §08 8.1 |
| `LevelSolidQuery` (whole `r_data/level_solid_query.cpp`) | DEAD on M/R/N (alive on S via `model_reach.cpp:593`) | `level_solid_query.cpp:115` | §08 8.1 |
| QuestZDoom remnants: `VR_DoomMain`, `getMenuState`, `getGameState`, `main_android`, `I_InitProfiles`, Quest JNI | DEAD | `d_main.cpp:5249`; `common/menu/menu.cpp:531`; `g_game.cpp:443` | §08 8.1 |
| `P_GuardBlocksMelee` → `P_GuardBlocks`; `Actor.OnSweepCrossed` (engine never dispatches) | DEAD | `p_guard.cpp:411-444`; `actor.zs:924-937` | §08 8.1-8.2 |
| `OptionalDescriptorIndexing`, ZVulkan indirect wrappers, bindless allocate overloads and the non-zero `flags` path of `AddBinding` | DEAD | `vulkanbuilders.cpp:780-800, 1834`; `vulkanobjects.h:391-392, 1049-1087` | §06 7 |
| Empty on Vulkan: `EnableClipDistance/EnableMultisampling/EnableLineSmooth`; unreachable on Vulkan: `SplitWallComplex`, `SplitSprite`, `SortSpriteIntoPlane`/`SortWallIntoPlane` CPU fallbacks, non-persistent `PutWall`/`PutFlat`/`PutSprite` branches, `SHADER_LITE`, `FLightBuffer` UBO path | DEAD on Vulkan | `vk_renderstate.cpp:159-161, 215-221`; `walls:1069, 1213, 859-866`; `hw_sprites.cpp:892-933, 854`; `flats:572-578`; `material_normal.fp:26-50`; `hw_lightbuffer.cpp:42-49, 128-136` | §03 9; §04 6; §05 6; §06 7 |
| `IShadowMap::ShadowTest` 2-argument overload; `#if 0` ColorFlicker light types; `DummyLight` has no `Tick` case | DEAD / no effect | `hw_shadowmap.cpp:145`; `a_dynlight.cpp:319-344`; `a_dynlight.h:42-43` | §05 6 |
| NPOT emulation code (`hw_npottest`), `bool inited`, `DoHorizon` mesh-builder branch (dead on M) | DEAD | `walls:329-331, 392-407, 1350-1354`; `bsp:88` | §03 9 |
| `MBF_FRIENDS` block (would not compile), `VMException` try/catch wrappers `#if 0` | DEAD | `p_enemy.cpp:1867-1898`; `vmframe.cpp:707-709, 741-751` | §02 6 |
| VR device leftovers: `xrMultiviewProbed/UsesCoreVulkan/MaxInstanceIndex` (written, never read), `FinalizeLayered`/`DirectXrRender` (hard false), `LogBoundSourcesForAction`, 5 unused statics, `stabilizeGeometryOk` (constant false), empty loop, `FOpenXRJoystickManager::AddAxes` stub, both-eye `xrMirrorPresentTextures` when one is drawn | DEAD | VK-XR:1182-1183, 1279-1913, 3436-3440, 4554, 6117-6122, 8577-8599; `common/platform/win32/i_openXR.cpp:259-262`; VK-XR:3062-3099 | §07 4, 6.2; §08 8.1 |
| `__WIN32__` blocks (MSVC defines `_WIN32`) | never compiled on MSVC (harmless) | `common/engine/i_net.cpp:172, 202, 234` | §08 8.3 |
| `FrameCycles` | written every `D_Display`, read only by the software renderer | `d_main.cpp:609, 1586-1926` | §01 6 |

### 6.3 Dead shader bindings and payload

| Item | Status | Evidence | Section |
|---|---|---|---|
| Set 1 binding 12 (`SectorPlaneSSO`) | DEAD on the GPU on M and N: written every frame, filled by the CPU, declared by no GLSL. LIVE on R (`levelmesh.vp:109, 229-230`). | `vk_descriptorset.cpp:101-159`; `hw_sectorplanes.cpp:111, 228` | §06 4.4, 7 |
| Light record floats 14/15 | DEAD payload, 8 B per light per surface | `hw_dynlightdata.cpp:187-188` | §05 6 |
| Set 0 binding 3 (scene depth) | **not** dormant: active by default with `r_gpuparticles_soft 8`; the "At 0, the default" comment is stale | `hw_cvars.cpp:311`; `di:3919, 3928-3932` | §06 7 |
| Set 0 binding 2 (TLAS) | DORMANT (`vk_raytrace` or `gl_light_castshadows 2` on an RT GPU, decided at device creation) | `vk_renderdevice.cpp:100, 365-368` | §06 7 |
| No fork shader is registered and never bound; the cost is the opposite: libraries and defines compiled into every `main.fp` program | — | `vk_shader.cpp:1171-1192` | §08 8.7 |

### 6.4 Half-built paths on `renderer`

| Item | Status | Evidence | Section |
|---|---|---|---|
| Snapshot writer `R_SnapshotWrite` (`r_snapshot` 0 = one int test; 1 writes records every frame; 3 spawns and joins up to 16 threads per write) | **nothing reads it**: `R_SnapshotAcquire`/`R_SnapshotRelease` and `FWorldSnapshot::ActorImage/Sectors/Sides/Lines` have no callers. Doc figure: 5.5 ms a write for 17,481 records. | `renderer:src/rendering/r_snapshot.cpp:181-186, 278, 373-383`; `renderer:src/d_net.cpp:2493, 2557` | §09 B.9; §04 4 |
| `r_renderthread` | stub: prints "not built yet" and resets | `renderer:src/rendering/r_snapshot.cpp:33-40` | §09 B.11 |
| `WallCache_SectorHash/SideHash/SectorFastHash`, `WallCache_NoBackSector`, `HWWall::SetTierLight`, `LevelMeshBuffers::StorageCapacityBytes` | DEAD | `renderer` `hw_wallcache.cpp:302-307`; `hw_walls.cpp:2448` | §09 B.11 |
| `r_wallcache` (stage 2b) | DORMANT, superseded by the mesh, still called per processed wall | `renderer` `hw_bsp.cpp:349, 763`; `hw_wallcache.cpp:313-316` | §09 B.11 |
| Verify hashes (`SectorBuildHash`, `SectorLightHash`, `SectorPlaneBuildHash`) | computed for every sector at every build; read only under `r_levelmesh_verify` | `renderer` `hw_levelresidency.cpp:2138-2147, 3605-3606` | §09 B.11 |

### 6.5 Dormant at defaults (reachable)

Multiview and its layered path (`vr_openxr_multiview` 0, menu-exposed); `vr_scene_multithread` 0 (menu-exposed);
`gl_bsp_worker_threads` 1 (its message says "after restart", but it is read every scene; the pool is fixed at 8
threads); `perf_track` 0 (menu-exposed; `-benchdemo` sets it), `r_perflog` 0; `vr_synclog`, `vr_telemetry`,
`vr_slime_enable`, `cl_debugprediction`, `vr_posestats`; `gl_light_shadowmap`/`gl_light_castshadows` 0; `vk_raytrace`;
`gl_ssao`, `gl_fxaa`, `gl_lens` 0, `gl_tonemap` 0 (a level's `level_tonemap` overrides it); `gl_sort_textures` 0,
`gl_max_portals` −1, `gl_max_vertices` 0; `r_dithertransparency`, `r_radarclipper` (cheat, need an out-of-bounds
camera); `gl_spriteclip` −1, `gl_sprite_blend`, `gl_billboard_mode` 0, `r_actorspriteshadow` 0, `r_voxeldistance` 0,
`gl_light_weapons` 0, the four `gl_light_*_cache` cvars; `r_exposureimpulse` 0; `nomonsterinterpolation`; the
slow-motion world clock (`P_TickRealTimeActors` only below scale 1.0); chase brain (per-actor null check) and
`RS_BoutDirector` (compiled, registered by no in-tree GAMEINFO); data validators (`-norun -validatedata`); the DAP
per-instruction hook (interpreter only); OpenVR mode 10 and the GL/GLES backends. On `renderer`: `r_newrenderer` 0,
`r_wallcache` 0, `r_snapshot` 0, `sim_prefetch` 0. Evidence: §01 6, §02 6, §03 9, §04 6, §05 6, §06 7, §07 6.1-6.3,
§08 4, §09 B.11.

**Always-on diagnostics** (live, not dead): `[PAWNANCHORLOG]` once a second in single-player VR (VK-XR:5864-5876),
`GRIPCHECK` once a second plus a double grip matrix per call (VK-XR:7927-7956), `vr_voxel_debug` default true (§5 R17),
`vr_grip_debug` default true (prints on change), `vk_gpu_checkpoints 1`, the RS submit journal (O(1) per submit,
`vk_commandbuffer.cpp:103-176`), +31 VR `glcycle_t` clocks (tick only when active), unconditional `rdtsc` in
`cycle_t` on Windows; on `renderer` the `simprofile`/`ticprofile` clocks run always. Evidence: §01 6, §04 6, §06 7,
§07 6.1, §08 4, 8.5, §09 B.11.

---

## 7. The four branches

| | `main` | `renderer` | `netplay` | `selaco` |
|---|---|---|---|---|
| Commit read | f182cbcc | 175063a1 (tip abb19af1 = + 4 docs commits) | 1c1805f8 | 7e111a3d |
| Relation to `main` | engine of record | merge-base 77e5ee3d (2026-11-11); 116 own commits (111 render-lane + 5 merges of main), 76 behind | strict ancestor: 0 own, 176 behind (frozen "Milestone 2026-10-31: netplay") | forked at 3511a341 "UZDXREMA 5.0.3" (2026-09-06); 104 own, 337 behind |
| Files differing from `main` (src+wadsrc) | — | 78 (46 renderer-only, 29 main-only, 3 both) | 63 | 389 |
| Has that others lack | `p_vrgrab`, `p_vrgunfit`, `p_vrslots.h`, rebuilt holster core, [SLOTLOAD], [NETSNAP], [STANDEYE], [LAGA]/[STICKHIPS], [CLAMPTIC] | static level mesh (`r_newrenderer`), wall cache, snapshot writer, Tracy zones (compiled out without `REMA_TRACY`), `ticprofile`/`simprofile`, sound per-source index, `sim_maxcatchup`, D_Display `ga_fullconsole` guard, demo buffer growth fix | — | `-nostockactors`, IWADOVER, pooled particles, `model_reach` (makes `LevelSolidQuery` live), DDS, Globals store |
| Lacks | — | main's 76 commits: grab/gunfit/slots, newer holster core, STANDEYE, 13 cvars | holster/torso/grab/grasp/UPnP files, `P_VRLaserTraceTic`, `DrawWorldPlayerSprites`, 33 cvars | 113 files: avatar/IK/rig, VR sim gear, affect, guard, effect lights, emissive volumes, exposure impulse, governor, perf_track, hang watch, 230 cvars |
| `usercmd_t` head | 6 shorts (z, pitch, roll, roomdx, roomdy, turn) | identical to main | 3 shorts (no room travel, no turn) | own protocol |
| Fence waits | polled 1 ms | polled 1 ms | polled 1 ms | blocking `vkWaitForFences(UINT64_MAX)` |
| `sizeof(AActor)` (`NDEBUG`) | 3,104 B | 3,104 B | 3,096 B | 2,464 B |
| Known branch bugs | §5 | I1, R28, S1-S4 | P6, P7 | — |

Sources: §09 A.1-A.5, appendices B-D; §01 7; §02 7; §06 13; §07 7; §08 9. The selaco commit count is 104 (re-run
for this summary: `git rev-list --count origin/main..origin/selaco` = 104); §08 9.3 said 111.

**What `renderer` changes by default** (with `r_newrenderer 0`, `r_wallcache 0`; §09 Summary 2-3, A.2.1, A.2.3, B.11):
- Behaviour: `sim_maxcatchup 2` — single player runs at most 2 tics a frame and drops the rest of the time
  (`renderer:src/d_net.cpp:207, 2415`); `snd_evictretry 8` / `snd_evictbudget 64` stagger evicted-sound restarts and a
  per-source channel index replaces O(channels) walks (`renderer` `s_sound.cpp:74, 1335-1414`); no level frame while
  `gameaction == ga_fullconsole` (`renderer` `d_main.cpp:1692`, VK-XR:669-670); `G_EnsureDemoSpace` before VR demo
  specials.
- Old render path image: exactly one change, `HWWall::PutWall` without a draw info clears `HWF_GLOW` at white light
  255 (`renderer` `hw_walls.cpp:894`); it runs only on the wall-worker path (`gl_bsp_worker_threads > 1`, and in VR
  also `vr_scene_multithread`), the wall cache and the mesh build, and makes worker output match the single-thread
  output.
- Per-frame overhead: one cvar test a scene (`LevelResidency_BeginScene`, `hw_levelresidency.cpp:4223`), one
  `WallCache_Process` call per processed wall, one int test per `TryRunTics` (`R_SnapshotWrite`), one int test per
  thinker (`sim_prefetch`), `simprofile`/`ticprofile` `rdtsc` pairs always, 5 more perf_track CSV columns and a
  flat-screen fallback for `scene`/`syncwait` (`renderer` `hw_clock.cpp:163-168`).
- Startup/backend: `EFF_LEVELMESH` compiled for every pass (`MAX_EFFECTS` 11, +4 glslang compiles); set 1 bindings
  16/17 bound every frame; set 1 binding 12 becomes live (`levelmesh.vp`).
- With `r_newrenderer 1..3` (not default): the walk, clipper and everything it feeds still run in full; only
  `HWWall::Process`/`HWFlat::ProcessSector` and per-part draws are replaced for static, non-evicted surfaces; every
  side with a dynamic light is evicted to the old path. Doc figures (flat, Sunder MAP15): old 70.1 / new 60.8 fps
  (1112y), old 76.2 / new 67.2 (1112x); walls CPU 2.56 → 1.05 ms but bsp 0.56 → 2.49 ms (sector comparison 1.42 ms
  a pass). §09 Summary 4-7, B.7, B.13.

---

## 8. Open measurements (merged, de-duplicated, ordered by how much they would change the picture)

**H** = needs the owner with the headset (the bridge cannot run one, M10 §5). **B** = needs a new build. Otherwise a
bridge `boottest` or a file read settles it.

| # | Question | Run that settles it | Needs | From |
|---|---|---|---|---|
| 1 | Does the owner's VR config already run multiview, the VR scene worker, Vulkan? (decides whether every §3.2 C/D cost is ×2 and serial) | Read `vr_openxr_multiview`, `vr_scene_multithread`, `vid_preferbackend` from the owner's ini through the bridge; startup log lines `Selecting Vulkan backend...` (`v_video.cpp:115`) and the bench header `XR: multiview=… active=…` (`hw_clock.cpp:325-334`) | — | §01 Q10; §03 Q1; §04 Q1; §06 Q8 |
| 2 | Real VR frame split: doubled scene build, waits, pacing | Headset session, `perf_track 1` + `stat rendertimes` (or `r_perflog 1`): `VRSceneBuild`, `VRSceneDraw`, `VRRenderSyncWait`, `VRSubmitWait`, `Finish`, `XrWait` vs `XrPeriod` (13.9 ms at 72 Hz); same spot with `vr_openxr_multiview 0/1` and `vr_scene_multithread 0/1`; compare with a flat run of the same spot | H | §07 Q1-Q2; §03 Q1-Q3; §04 Q1; §06 Q2 |
| 3 | The owner's mod set on `main` itself (no `sim_maxcatchup`, O(channels) sound): how much worse than the measured `renderer` runs | Same M10 run (Sunder MAP15, Abandon MAP13, `-rngseed 1113`, uncapped, `perf_track 1`) on a `main` exe; and on `renderer` A/B `sim_maxcatchup 0`, `snd_evictretry 0`, `snd_evictbudget 0` vs defaults | B (main exe) | §01 5.2, 5.7; §02 5.7-5.8; §09 Q8 |
| 4 | Is the ZScript JIT on in the owner's build? (off would make the VM interpreter and DAP hook dominant) | `HAVE_VM_JIT` in `E:\DOOMWork\UZDXREMA\build-dxr\CMakeCache.txt`; `vm_debug` in the ini; `vm_jit` in the console; startup log | — | §02 Q1; M10 §5 |
| 5 | Per-thinker engine overhead split: VM call layer share, state changes, movement, sight, sounds | Build with `CallTick` calling the native `Tick` directly when the slot is native and without `VMValidateSignatureSingle`, compare `stat think`; per-tic counters of `SetState`, `CallAction` (DefaultArgs branch), `P_TryMove` by caller, `P_CheckSight`, `S_StartSound`, `FBlockThingsIterator::Next` visits, `P_ChangeSector` node visits, on Sunder MAP15 and Abandon MAP13 with the owner's list | B | §02 5.0, 5.2-5.5, Q3-Q7 |
| 6 | Busy-wait in VR | Headset, same spot, `r_ticstability 1/0`; `FrameMs`, `CostMs` on rows with `Tics=0`; `XrWait` should rise by the spin when off | H | §01 Q1; M10 §4 |
| 7 | Cost of the polled waits and of the XR blit wait | `VRRenderSyncWait`, `VRSubmitWait`, `Finish` against the GPU group sum; A/B with a build using blocking `vkWaitForFences` (selaco's code); A/B the CPU fence wait before `xrReleaseSwapchainImage` | B + H | §01 Q2; §06 Q6; §07 Q2 |
| 8 | 4-player netgame per-tic cost (the goal): `WorldStateHash`, prediction re-runs, sight in co-op, haptics in MP | 4-player rig session vs single player on the same spot: clock `MakeConsistencies`, `P_PredictClient`/`P_UnPredictClient`; `stat sight` after fixing its early returns; check `VR_HapticEvent` for remote players' health | B (clocks) | §01 Q7, Q12; §02 Q6, Q8, Q12 |
| 9 | Do frames hit the capacity walls (stream ~9,536 Applies, matrix, viewpoint at 99 uploads, descriptor pools, recycle waits)? | Add counters for Applies/frame, `WaitForStreamBuffers`, `VkHardwareBuffer::Resize`, descriptor-pool waits, 64 MB waits, recycle waits; Abandon MAP13 owner list, two eyes | B (+H for VR) | §01 Q8; §06 Q1 |
| 10 | Where `drawcalls` goes (2,000 B memcpy vs binds vs pipeline misses), draw count and material churn per scene | `stat renderstats`; RenderDoc capture of one eye (draws, descriptor binds); A/B `gl_sort_textures 1`; profile `Apply` sub-functions | — / B | §03 Q5; §06 Q3 |
| 11 | Light CPU cost under real fire | `profilecsthinkers` (lights as `InternalDynamicLight`) and `r_perflog` relinks during a firefight; `RenderWall`/`RenderFlat` minus `drawcalls` with `r_dynlights 1` vs `0`; A/B `gl_light_sprites 0`, `gl_light_particles 0`; `stat lightstats` pairs | — | §05 9; §04 Q4 |
| 12 | Why GPU wait is 25.87 ms with mods while the listed GPU groups sum to 14.0 ms | Same run with `r_perflog 1`: all GPU groups (`scene.translucent`, `fx.*`, `pp.*`), and `Finish` vs fence-only time | — | M10 §1; §06 12 |
| 13 | Pose staleness, interpolation jitter, room-scale double application | Per-frame log of `I_nsTime` at `I_SetFrameTime`, at `xrWaitFrame` return, `predictedDisplayTime` of each `UpdateControllerState` (shared clock, VK-XR:651-665) and the `gametic` built; `[PAWNANCHORLOG]` "asked to move" vs real head travel; `vr_posestats 1` `[A0]` | H | §01 Q3-Q5; §07 Q5 |
| 14 | Fork always-on costs: uber-shader, checkpoints, soft particles, engine event handlers, laser traces | Build without `SURFACE_DAMAGE`/`EFFECT_LIGHTS` defines and prepended libraries, compare `scene.opaque` + SPIR-V register counts; `vk_gpu_checkpoints 0` (restart); `fx.depthread` and `r_gpuparticles_soft 0`; `WorldTick` with the two handlers removed from `mapinfo/common.txt`; `Trace` count per tic | B / — | §08 11 Q1-Q5; §05 9 |
| 15 | Prediction serialisation cost in single player with BD22-class pawns | `simprofile` "predict" on `renderer`, or a `PerfTrack::Scope` around the two calls on `main` | — / B | §01 Q7; §02 5.15 |
| 16 | Memory-bound walks | Hardware counters (L3/DRAM misses) over the reset loop and the recreate-lights scan; `perf stat -e cache-misses` on a 30k-thinker map; `sizeof(AActor)` stock 5.0.0 vs main | B | §02 Q9; §08 Q7 |
| 17 | Eye resolution and present cost | Startup log `OpenXR: view %u recommended %ux%u` (VK-XR:2888); bench header `XR Targets:` (`hw_clock.cpp:336`) | H | §07 Q3 |
| 18 | Exposure adapting twice per VR frame; custom "screen" shaders 3× a frame | RenderDoc capture of the 1×1 camera texture per eye; `stat gpu` groups named after a mod's screen shader | H | §05 9; §07 Q6 |
| 19 | Multiview correctness (left-eye culling, stereo billboards) | `rig_eyeshot <path> 1` at `vr_openxr_multiview 0/1`, diff both eyes | rig | §03 Q9; §07 Q4 |
| 20 | Upload queue family on the 3080 Ti (main-thread texture creation if transfer-only) | Startup log `Upload Queue Family: #N (...)` (`vk_renderdevice.cpp:1585`) and the "do not support graphics" message | — | §06 Q5; M10 §5 |
| 21 | `FVRRig` recomputes and `VRAvatar_Pose` calls per frame | Counters per displayed frame | B | §01 Q6; §04 Q7; §08 Q6 |
| 22 | Bone and light buffer pressure; draw count by actor class | `stat bufferstats` "bonebuffer="; a per-class `PutSprite` counter (gore vs HP bars vs champion effects) | — / B | §04 Q2, Q5, Q9 |
| 23 | GC step cost and frequency | perf_track `REASON_GC` (add the `D_Display` call) | B | §01 Q9 |
| 24 | Static path in VR with dynamic lights; camera-texture region cap; fold-back spikes; build time; memory | Two-eye non-multiview VR, `r_dynlights 1`, `r_newrenderer 0` vs 3: perf_track `bsp/clip/walls/flats/scene/drawcalls`, `levelmeshstats` (`ScenesNoRoom`, `DrawScenesWhole`, `FoldMaxMS`), `levelmesh: built … ms`, RSS/VRAM | H | §09 Q1-Q7 |
| 25 | Determinism leaks used by the mod set (`HeatmapAt`, `ML_MAPPED`, `SECMF_DRAWN`, billboard `drawPos`) and real use of the ~429 uncalled natives | grep the owner's fixed mod set's ZScript and the RS/BD22 sources | — | §03 Q10; §08 Q8-Q9 |
| 26 | Distance-cull impostor count and the skybox bug in play | Counter of `Process(isculled)` calls vs unique linedefs, or `rendered_lines` with `gl_line_distance_cull 0`; screenshots of a skybox map at 4000 vs 0 | — | §03 Q4 |
| 27 | Pipeline churn on VR menu transitions; live pipelines per render-pass key | perf_track `REASON_PIPELINE` count entering/leaving menus; `Pipelines.size()` at level end | H / B | §06 Q4, Q7 |
| 28 | IPD mismatch | Log `xrViews[1].pose.position − xrViews[0].pose.position` against `vr_ipd` | H | §07 Q7 |
| 29 | `main.fp` register pressure; `StreamData` real size | Driver shader statistics for the default program; `static_assert(sizeof(StreamData)==2000)` or print `MAX_STREAM_DATA` (§06 compiled 2,000 B; §05 kept it inferred) | B | §05 9; §06 4.1 |
| 30 | Build defines of the owner's exe | `ENABLE_OPENXR`, `ENABLE_OPENVR`, `HAVE_GLES2` in the CMake cache | — | §08 Q10 |

---

## 9. Index

| Section | What it covers | Key findings |
|---|---|---|
| [01](01_FRAME_LOOP_TIMING_THREADS.md) | Frame loop, tic timing, threads, every wait | One thread does everything; tics run before `xrWaitFrame`; three polled CPU←GPU waits per VR frame, no frame overlap; `TicStabilityWait` busy-spins on no-tic frames; no catch-up cap on `main`; single-player prediction serialisation every tic-frame; SetUp twice a frame; interpolation from the loop-top clock; 8 pool threads idle in VR. |
| [02](02_PLAYSIM_SCRIPTING.md) | Playsim and ZScript/ACS per tic; what blocks a parallel sim | `RunThinkers` `Tick` bodies dominate (re-ranked by M10); every thinker goes through the VM call layer; three full walks over 3,104 B actors per real tic (+netgame hash); `UpdateRenderSectorList` never skips; `CallAction` heap copy; sound O(channels) on `main`; shared mutable state and render↔sim writes listed (§5.B). |
| [03](03_SCENE_BUILD_BSP_WALLS_FLATS.md) | CPU scene build: BSP walk, clipper, walls, flats, portals, sky | Whole visible world rebuilt per eye/portal/camera texture; walls have no raster geometry and build vertices + light lists at draw time; walk serial in VR; fork distance cull bugs B1-B13; 14 outputs only the walk produces (§7); render→script leaks (heat grid, billboard `drawPos`). |
| [04](04_SPRITES_MODELS_PARTICLES_WEAPONS.md) | Sprites, models, particles, decals, weapons, HUD; finding actors without the walk | Actors reach the renderer only via reached sectors; one draw + full `Apply` per sprite/particle/decal/mesh, no batching; CPU light sum per sprite per draw; CPU bones per model per draw, no cache; VM calls from the draw; walk-free enumerators and what each misses; Helion comparison. |
| [05](05_LIGHTS_SHADOWS_FOG_POSTPROCESS.md) | Dynamic lights, shadows, fog, post-process | Lights relink with a 2-D flood and heap nodes per tic; per-surface light lists per eye at draw time; fork effect lights are already binned; ~34 PP passes per eye, each its own render pass; dead coronas and light-path leftovers; what a clustered design would replace. |
| [06](06_VULKAN_BACKEND_SUBMISSION.md) | Vulkan backend and draw submission | Single queue, single frame, full sync each frame; all per-frame buffers single-buffered; 2,000 B `StreamData` per draw, 32 per block; capacity walls force mid-frame drains; `multiDrawIndirect` required but unused; no descriptor indexing; lazy pipelines; serial startup shader compile; fork compute on the graphics queue. |
| [07](07_VR_OPENXR_STEREO.md) | OpenXR path, stereo, per-eye cost | Scene and PP per eye (multiview off; PP per eye even with it); double SetUp, stale first pass; second submit + wait for the swapchain blit; extra present passes; scene IPD from a cvar; no foveation/VRS/visibility mask/depth layer; menu-only XR cvars. |
| [08](08_FORK_ADDITIONS_HOOKS_DEAD_CODE.md) | Fork inventory vs stock 5.0.0, every cvar and hook, dead code | 272 new files, 623 fork cvars (6 dead, 7 GL-only), +459 natives (~429 uncalled in-tree), 58 dead functions, 12 never-compiled files; most R-layer effects are demand-driven; always-on costs: uber-shader, checkpoints, soft-particle pass breaks, laser traces, two engine event handlers. |
| [09](09_BRANCHES_AND_STATIC_LEVEL_PATH.md) | The four branches; the render lane's static level path | `main` is the engine of record; `renderer` changes defaults (`sim_maxcatchup 2`, sound eviction) and one old-path image rule; the static path keeps the whole walk, evicts lit surfaces, is slower in every measured pair (doc figures), and has a snapshot writer nothing reads. |
| [10](10_MEASUREMENTS.md) | Measurements on the owner's PC (flat, `renderer` old path) | Vanilla 66.7 fps / 13.9 ms; owner's mod set 3.4 fps / 315 ms with the sim at ~79 % and the scene build at ~1.4 %; thinkers 4.6× the tic budget on Sunder MAP15; per-class costs by mod family; busy-wait costs 2.3 ms a frame flat. |
| [README](README.md) | Folder index and status | Lists the sections and the remaining work items. |

**Disagreements between sections, and how this file settles them**
- `selaco` own commits: §08 9.3 says 111, §09 A.1 says 104. `git rev-list --count origin/main..origin/selaco` = 104.
- `StreamData` size: §05 4.3 keeps 2,000 B as inferred (field replica); §06 4.1 obtained 2,000 B by compiling the real
  header. Used as 2,000 B (compiled); the `static_assert` in §8 #29 would close it.
- `VkRenderState::DrawInstanced`: §06 3 lists it with the indexed variant as "the only instanced path"; §08 8.1 lists
  it as dead. Grep for this file: `DrawInstanced(` has no caller; `DrawIndexedInstanced(` is called at
  `hw_meshparticles.cpp:529` and `hw_debrispool.cpp:1324`.
- Cvar lines: §03 2.7/9 put `gl_multithread`/`gl_bsp_worker_threads` at `bsp:45/46`; §01/§04 at `:46/:47`. Source:
  `:46` and `:47`.
- VR waits per frame: §07 says "2-3", §06 "two or three", §01 "three". At the default `vr_openxr_sync_mode 1` there are
  three (§01 2.4); two with another sync mode.
- Poll overshoot: §01 "rounded up to the next ~1 ms", §06 cross-check "1-2 ms per failed poll". Used ≥1 ms, typically
  1-2 ms, (inferred) for the Windows sleep granularity.
- The three Developer Options XR cvars: §07 calls them MENU-ONLY, §08 counts them among the 6 DEAD. Both are true:
  dead to the engine, present as menu rows (§6.1).
- Thinker and `WorldTick` costs: §01/§02/§08 quote doc figures (thinkers ~46-57 ms, WorldTick 2.5-3.7 ms); M10
  measured 61-131 ms and 2.3-4.1 ms per tic. M10 numbers are used; doc figures are labelled as such.
- Measured build vs read source: M10 ran `renderer` b3fc177, the sections read 175063a1; the only commit between
  them changes two docs files (checked for this file).
