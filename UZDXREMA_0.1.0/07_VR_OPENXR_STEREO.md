# 07 — The VR path: OpenXR, stereo, per-eye cost

Primary tree: `main` (`/home/claude/uzdx_main`, f182cbcc). Paths are relative to `src/` unless they start with another top-level directory. "VK-XR" = `common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp` (8632 lines, read in full). VERIFIED = the code that does it was read. INFERRED = deduced; the basis is given. No millisecond figure in this file is a measurement of mine.

---

## 1. Summary

1. VR is `vr_mode 15` (`VR_OPENXR_MOBILE`, `hw_vrmodes.h:59`), Vulkan only (`hw_vrmodes.cpp:1293-1304`). The only OpenXR implementation compiled is `s3d::VKOpenXRDeviceMode` (VK-XR); the GL OpenVR mode (`rendering/gl/stereo3d/gl_openvr.cpp`, src/CMakeLists.txt:1069, `USE_OPENVR` :396) is also compiled [xc: "the only implementation compiled" was too broad]. `gl/stereo3d/gl_openxrdevice.cpp` is not in any CMake source list: DEAD.
2. **The whole scene is built and drawn once per eye.** `RenderViewpoint` (`rendering/hwrenderer/hw_entrypoint.cpp:359-516`) loops over 2 eyes. Each eye runs `StartDrawInfo` → `ProcessScene` → `DrawScene` → `CreateScene` → `RenderBSP` (the BSP walk, walls, flats, sprites), the draw lists, portals, translucent pass, player sprites, the post-process chain and `DrawEndScene2D`; the `twod` 2D replay runs per eye later, in a second eye loop in `VulkanRenderDevice::Update` (vk_renderdevice.cpp:680-710) [xc: the 2D pass is not inside the RenderViewpoint eye loop]. VERIFIED.
3. Multiview exists but is off. `vr_openxr_multiview` defaults to false (`hw_vrmodes.cpp:643`), and `ShouldUseMultiviewThisFrame` (VK-XR:8608) requires it. With it on, the scene is built once from the left eye's position and drawn into 2 layers. Post-processing still runs per eye: every PP pass has `ViewMask = 0` (`vulkan/renderer/vk_pprenderstate.cpp:112,159`). The cvar alone does not turn it on: `ShouldUseMultiviewThisFrame` also needs `xrMultiviewSupported` (device/runtime probe, VK-XR:8600-8604), `xrViewCount > 1` and `mFrameRenderMode` GameplayEyes or MenuOverWorld, which `SetUp` rewrites (twice per SetUp, VK-XR:3583,3626). The answer is re-evaluated at ~10 sites per frame: hw_entrypoint.cpp:354, hw_viewpointbuffer.cpp:112 (every `SetViewpoint` duplicates into 2 slots while it is true), vk_renderdevice.cpp:675,815,1406,1490-1491,1643, vk_postprocess.cpp:139, VK-XR:1162,1179. BeginFrame sizes the layered buffers from SetUp #1's mode, the eye loop decides from SetUp #2's; the layered paths in vk_renderdevice/vk_postprocess re-check `GetSceneLayers()/GetPipelineLayers() > 1`, hw_entrypoint.cpp:354 does not (INFERRED: a mode change between the two SetUps would give one frame with a postprocess-only eye 1 over single-layer buffers) [xc: added].
4. In VR the BSP walk runs **single-threaded** by default: `multithread = gl_multithread && (!IsVRScene || vr_scene_multithread)` (`rendering/hwrenderer/scene/hw_bsp.cpp:1438-1439`), and `vr_scene_multithread` defaults to false (`hwrenderer/data/hw_cvars.cpp:278`). Flat-screen uses `gl_multithread=true` (`hw_bsp.cpp:46`). So the doubled CPU build also loses its worker thread.
5. **The frame is fully serialized.** The order is: `xrWaitFrame`/`xrBeginFrame`/`xrLocateViews` (`VulkanRenderDevice::BeginFrame`, `vulkan/system/vk_renderdevice.cpp:1458-1459`), then the CPU build of both eyes, then a CPU wait for all GPU work (`Update`, `vk_renderdevice.cpp:735`). After that comes a *second* command buffer that blits the eyes into the OpenXR swapchain, a second CPU fence wait (VK-XR:6342-6379), then `xrEndFrame`. With the default `vr_openxr_sync_mode 1`, a desktop present and a third full wait follow (`vk_renderdevice.cpp:746-750`). The CPU and GPU never overlap across frames.
6. Every fence wait is a poll loop with `sleep_for(1 ms)` (`vulkan/system/vk_commandbuffer.cpp:181-221`). That is 2-3 polled waits per VR frame, each able to overshoot by up to one sleep quantum (the timer period is set to `wPeriodMin`, `common/platform/win32/i_main.cpp:280-287` [xc: path was missing `common/`]; that timer setting is Win32-only). The sleep happens only when a `vkGetFenceStatus` poll finds a fence still unsignalled (vk_commandbuffer.cpp:195-219) [xc: added].
7. Extra fullscreen passes per frame on the default settings: 2× XR present pass into `xrPresentTextures` (recommended size), 1× mirror present pass (the eye `vr_desktop_view 2` shows), 2× blit into the swapchain layers, and 1× blit to the desktop window (`vulkan/textures/vk_framebuffer.cpp:86-111`).
8. `VKOpenXRDeviceMode::SetUp` runs **twice per frame**: in `BeginFrame` before `xrWaitFrame`, and again in `RenderViewpoint`. Each run does `xrSyncActions`, 4× `xrLocateSpace`, key-event posting, a render-rate pawn move and haptics. The first run locates the hands at the *previous* frame's predicted display time. `xrVRFrameIndex`, which is documented as one tick per displayed frame, ticks twice.
9. No foveated rendering, no VRS/shading-rate, no eye tracking, no visibility mask (hidden-area mesh), no depth layer submission. The only "foveation" is `r_effects_foveated`, which lowers ray-march step counts for smoke and flash and has nothing to do with XR (`hwrenderer/postprocessing/hw_postprocess_cvars.cpp:391`). The bench header prints `vr_openxr_foveation=unsupported` as a literal (`hwrenderer/data/hw_clock.cpp:350`).
10. Branches: `renderer` = main plus Tracy zones; its new static level mesh draws inside the same per-eye loop. `netplay` has the same frame structure, with an input-edge bug in the VR device. `selaco` has the same eye loop, double `SetUp` and separate XR blit, but uses blocking `vkWaitForFences` (no 1 ms polling) and lacks spectator, perf_track, the governor and the hand ring.

---

## 2. Entry points and call graph

### 2.1 Mode selection

- `CUSTOM_CVAR(Int, vr_mode, 0)` — `hw_vrmodes.cpp:605`. Values: `hw_vrmodes.h:44-59` (0 mono … 10 OpenVR, 15 OpenXR).
- `VRMode::GetVRMode(toscreen)` — `hw_vrmodes.cpp:1208-1307`. `toscreen=false` (camera textures, save pictures) always returns mono (`:1234`). Case 15 returns `VKOpenXRDeviceMode::getInstance()` only if `V_GetBackend()==BACKEND_VULKAN` and `IsInitialized()` (`:1293-1304`). Case 10 (OpenVR) has **no backend check** (`:1283-1290`); it is GL-only in practice and DORMANT on this Vulkan-only engine.
- `VRMode::GetVRModeCached` — `hw_vrmodes.cpp:561-603`. Thread-local cache keyed on `screen->FrameTime`, `vr_mode`, backend and the texture-filter flag. Worker threads always get a static mono mode (`:563-568`).
- `VKOpenXRDeviceMode::IsInitialized` — VK-XR:2144-2161. Until OpenXR is ready it probes the runtime at most once per `FrameTime`. On failure it sets `vr_mode 0` (`DisableOpenXRModeForCurrentRun`, VK-XR:2135-2142).

### 2.2 One loop iteration in VR (VERIFIED order)

```
D_DoomLoop                                   d_main.cpp:1966
 ├ TryRunTics                                d_main.cpp:2006 → d_net.cpp:2339
 │   doWait=false in VR                      d_net.cpp:2357-2359
 │   NetUpdate: I_StartTic skipped in VR      d_net.cpp:1520-1525; G_BuildTiccmd d_net.cpp:1537
 │     G_BuildTiccmd → vrmode->WriteHandCommand   g_game.cpp:1527 → VK-XR:8317 (reads cached poses)
 ├ I_StartTic; D_ProcessEvents               d_main.cpp:2010-2011  (controller key events posted LAST frame are consumed here)
 └ D_Display                                 d_main.cpp:1560
    ├ screen->BeginFrame                     d_main.cpp:1685 → vk_renderdevice.cpp:1449
    │   ├ vrmode->SetUp()   [SetUp #1]       vk_renderdevice.cpp:1458 → VK-XR:3575
    │   │    ++xrVRFrameIndex, PollXREvents, UpdateControllerState (uses OLD xrFrameState), updateHmdPose (OLD xrViews)
    │   ├ vrmode->BeginXRFrame()             vk_renderdevice.cpp:1459 → VK-XR:6006
    │   │    xrWaitFrame (blocks: runtime pacing) :6051, xrBeginFrame :6075, xrLocateViews :6087, updateHmdPose :6124
    │   └ size buffers to recommended×vr_openxr_render_scale   vk_renderdevice.cpp:1467-1499
    ├ DrawHudToSurface (status bar → VR HUD canvas, VR only)   d_main.cpp:1693 → :1254
    ├ RenderView                             hw_entrypoint.cpp:640
    │   ├ canvases (incl. VR HUD canvas) render pass         :688-715
    │   ├ camera textures (mono RenderViewpoint each)        :719-730
    │   └ RenderViewpoint(main)              :750 → :264
    │       ├ R_SetupFrame                    :268
    │       ├ EmissiveVolumes/BeamLights/EffectLights BeginFrame, shadow map   :273-316  (once)
    │       ├ vrmode->SetUp()  [SetUp #2]     :327  (fresh xrFrameState; second UpdateControllerState)
    │       ├ R_UpdatePoseAnchoredLights      :330  (once)
    │       ├ PrepareFrameCompute/RunFrameCompute, ExposureImpulses   :334-346 (once)
    │       └ for eye_ix in 0..1              :359-516
    │            ++gl_dynlight_viewid (invalidates per-light caches)  :361
    │            eye->SetUp (projection from xrViews[eye].fov)        :368 → VK-XR:1983
    │            SetSceneRenderTarget, StartDrawInfo, Set3DViewport   :412-421
    │            GetRenderFov / GetProjection ×2 / GetViewShift        :435-451
    │            SetupView + ApplyViewpoint (or multiview pair)        :453-479
    │            di->ProcessScene                                      :482 → hw_drawinfo.cpp:4597
    │               particle/mesh/drawn-line/view-light syncs, heat/smoke/emissive setup
    │               DrawScene → CreateScene → RenderBSP (BSP walk)     hw_drawinfo.cpp:4450,3665; hw_bsp.cpp:1422
    │               RenderScene, mesh particles, DrawPlayerSprites (in-scene in VR), DrawWorldPlayerSprites,
    │               SSAO, portals, RenderTranslucent, DrawMountedHud, tracers, laser, VRWheel_Draw  hw_drawinfo.cpp:4496-4587
    │            eye->AdjustHud (extra StartDrawInfo + viewpoint upload)  :483 → VK-XR:2033
    │            EndDrawScene, PostProcessScene (full PP chain), AdjustBlend, V_DrawBlend   :488-507
    │            EndDrawInfo, eye->TearDown, NextEye                  :512-514
    ├ status bar/2D into twod (face HUD only if not mounted)          d_main.cpp:1700-1792
    └ End2DAndUpdate → screen->Update        d_main.cpp:1533 → vk_renderdevice.cpp:657
         for eye 0..1:                                               :680-710
            AdjustBlend, Draw2D(inside), AdjustHud, Draw2D(outside), FinalizeEyeImage
              FinalizeEyeImage → mirror present pass (one eye) + XR present pass   VK-XR:7196-7263
         RenderVirtualScreen (menu frames only)                      :714 → VK-XR:6776
         EndRenderPass, EndFrame
         WaitForCommands(finish=!sync1) — flush + CPU poll-wait all fences     :735 → vk_commandbuffer.cpp:430
         AcquireXRSwapchain                                          :745 → VK-XR:6133
            xrAcquireSwapchainImage, xrWaitSwapchainImage (20 ms timeout)
            own cmd buffer: 2 blits present→swapchain layers, vkQueueSubmit, CPU poll-wait fence
            xrReleaseSwapchainImage, xrEndFrame(projection layer [+ quads])
         [sync_mode 1] AcquireImage (desktop swapchain + mirror blit) + WaitForCommands(true) (3rd wait, present)  :746-750
```

### 2.3 Pose sampling and frame timing (VERIFIED)

- **Head.** Sampled once by `xrLocateViews` at `xrFrameState.predictedDisplayTime`, right after `xrWaitFrame` (VK-XR:6079-6087). `updateHmdPose` (VK-XR:3708) averages eye positions into `hmdPosition` and the centered orientation into `hmdorientation`. It applies the yaw delta to `doomYaw` and `G_AddViewAngle` (:3834-3879), and writes `vp.HWAngles` pitch/roll and `vp.Angles.Yaw` (:3883-3903). The same `xrViews` poses are submitted in `xrProjectionViews` (VK-XR:6399-6410), so the compositor reprojects against the pose the frame was rendered for. There is **no late latch**: nothing re-locates after the CPU build.
- **Hands.** `xrLocateSpace(... xrFrameState.predictedDisplayTime ...)` for aim (VK-XR:4158) and grip (VK-XR:4362). In SetUp #1 this is the previous frame's predicted time, because `xrWaitFrame` has not run yet (`vk_renderdevice.cpp:1458` precedes `:1459`). SetUp #2 uses the current one.
- **Tic vs frame.** `TryRunTics` runs before `xrWaitFrame` in the same iteration (d_main.cpp:2006 vs :1685). Controller buttons are posted as events during `D_Display` (SetUp → `UpdateControllerState` → `PostControllerKeyTransition`, VK-XR:1482). They are consumed by the next iteration's `D_ProcessEvents`/`G_BuildTiccmd`. `WriteHandCommand` (VK-XR:8317) reads the poses cached by the last SetUp. INFERRED latency from button to sim: at least 1 frame plus the wait to the next tic boundary (up to 28.6 ms).
- **Pacing.** `xrWaitFrame` is the only pacer in VR (`TryRunTics` never waits, d_net.cpp:2356-2359). Its blocking time goes into `EffectsGovernor::AddPacingWait` (VK-XR:6053) and `perf_track` `XrWait` (VK-XR:6063).
- **Refresh rate.** `ApplyRefreshRate` (VK-XR:5915) asks `XR_FB_display_refresh_rate` for the rate closest to `vid_refreshrate`, once at `xrBeginSession` (VK-XR:6030) and on `NewRefreshRate` (`vk_renderdevice.cpp:840-847`). It only works if the runtime offers the extension.

---

## 3. Per-function table

Cost class: O(1) = constant; "per eye" = runs 2× per frame; "xr" = an OpenXR runtime call (cost is the runtime's, unmeasured here).

| Function | file:line | Called from | Per | Cost class | Notes |
|---|---|---|---|---|---|
| `VRMode::GetVRModeCached` | hw_vrmodes.cpp:561 | everywhere (≥30 call sites/frame) | call | O(1) hash-free compare | VERIFIED. thread_local cache |
| `VRMode::GetVRMode` | hw_vrmodes.cpp:1208 | TryRunTics (d_net.cpp:2357), NetUpdate (:1522), FVRRig::Get (r_data/vr_rig.cpp:609) | call | O(1) after init | Not cached; `IsInitialized` short-circuits once ready |
| `VKOpenXRDeviceMode::SetUp` | VK-XR:3575 | vk_renderdevice.cpp:1458 **and** hw_entrypoint.cpp:327 | **2× per frame** | xr + O(hands) | ++xrVRFrameIndex, PurgeDeferred, PollXREvents, UpdateControllerState, updateHmdPose |
| `VRMode::SetUp` (base, via `super::SetUp`) | hw_vrmodes.cpp:1481 | VK-XR:3585; also mono SetUp for every camera texture | per SetUp | 2× GetWeaponTransform | In SP writes AttackPos/OffhandPos from controllers. Mono SetUp (camera textures) writes `PlayInVR=false` and the head-height fallback; the main SetUp then overwrites it (INFERRED harmless ordering) |
| `PollXREvents` | VK-XR:3657 | SetUp | 2×/frame | xr | Drains **one** event per call |
| `UpdateControllerState` | VK-XR:3906-5883 | SetUp | 2×/frame | xr: 1 xrSyncActions, 28 GetAction* (14 per hand), 4 xrLocateSpace, 1 xrGetCurrentInteractionProfile+2 xrPathToString+vector alloc (`IsCurrentInteractionProfile`, :3934/1743) | Also: grip arbiter, support latch, key/GUI event posting, menu ray, AttackPos/vel publish, `VR_PushHandSample` ×2, `P_LineTrace` while teleport aim is held (:5807), `VR_ApplyRenderMove`→`P_XYMovement` (roomscale, SP, p_vrdemo.cpp:494), a `PAWNANCHOR` log line once a second ungated (:5864-5876), ProcessHaptics |
| `updateHmdPose` | VK-XR:3708 | SetUp ×2, BeginXRFrame | **3×/frame** | O(views) | Writes hmdPosition/orientation, doomYaw, G_AddViewAngle, r_viewpoint angles, roomscale delta |
| `BeginXRFrame` | VK-XR:6006 | VulkanRenderDevice::BeginFrame | frame | xr (blocking) | xrBeginSession on first READY; xrWaitFrame, xrBeginFrame, xrLocateViews; perf_track/governor timing |
| `ApplyFrameRenderMode` / `DetermineFrameRenderMode` | VK-XR:3558/3532 | SetUp (2× per SetUp) | 4×/frame | O(1) | GameplayEyes / VirtualScreen / MenuOverWorld |
| `VKOpenXRDeviceEyePose::SetUp` | VK-XR:1983 | eye loop :368; multiview second eye :461 | per eye | O(1) | Copies xrViews[eye].pose/fov, builds projection |
| `GetProjection` | VK-XR:1929 | eye loop :441-442 (2 calls) | per eye | O(1) | `BuildOpenXREyeProjection` (VK-XR:1344), asymmetric runtime FOV ± `vr_openxr_fov_adjust_deg` |
| `GetRenderFov` | VK-XR:1942 | :435 | per eye | O(1) | max(h,v) FOV → clipper FOV |
| `GetViewShift` | VK-XR:1951 | :451 | per eye | O(1) | Eye offset = `vr_ipd*0.5*vr_vunits_per_meter` along the view right vector, plus HMD height minus player view height. **Uses cvar IPD (0.064), not the runtime's eye poses** |
| `AdjustViewpointUniforms` | VK-XR:1975 | :456/:477 | per eye | O(1) | CalcDependencies only |
| `HWDrawInfo::ProcessScene` | hw_drawinfo.cpp:4597 | :482 | per eye (eye 0 only with multiview) | heavy | Particle-definition/GPU-ring/mesh-particle syncs, `SyncDrawnLines` (rebuilt every scene, :4674), `SyncViewLights`, heat/shockwave/smoke/emissive setup, DrawScene |
| `HWDrawInfo::CreateScene` | hw_drawinfo.cpp:3665 | DrawScene | per eye | O(visible subsectors+segs+things) | Clocks VRSceneBuild/VRScenePostBSP |
| `HWDrawInfo::RenderBSP` | hw_bsp.cpp:1422 | CreateScene | per eye | O(BSP nodes visited) | **Single-threaded in VR unless vr_scene_multithread** (:1438-1439). `PreparePlayerSprites` per eye (:1558-1563) |
| `RenderScene` / `RenderTranslucent` | hw_drawinfo.cpp:3722/3855 | DrawScene | per eye | O(draw list) | VRSceneDraw clock |
| `DrawPlayerSprites` | rendering/hwrenderer/scene/hw_weapon.cpp:1534 | DrawScene :4539 (`RenderPlayerSpritesInScene()` = true for OpenXR, vk_openxrdevice.h:209) | per eye | O(psprites) + model draws | Weapons/hands drawn in 3D per eye |
| `DrawWorldPlayerSprites` | hw_drawinfo.cpp:4548 | DrawScene | per eye (+ portals except sky) | O(players) | |
| `DrawMountedHud` | VK-XR:6525 | DrawScene :4581 | per eye | 1 quad (+4 border quads on the automap) | |
| `VRWheel_Draw` | hwrenderer/data/hw_vrwheel.cpp:2385 | DrawScene :4585 | per eye | O(entries) while a wheel is open; RefreshEntries+UpdateHover **per eye** (:2259-2260) | UpdateHover can fire haptics (:2135) |
| `DrawLaserSightWorld`, `DrawHitscanTracers` | hw_drawinfo.cpp:4583-4584 | DrawScene | per eye | O(beams) | |
| `models.cpp` follow-hand/body placement | r_data/models.cpp:1980,2072,3185,3198 | model draws | per eye per followed actor | O(1) matrix each | GetWeaponTransform/GetHmdTransform/GetGripTransform |
| `GetGripTransform` | VK-XR:7917 | models.cpp, vr_rig, UpdateControllerState | per call | 2 matrix builds | Builds every grip twice (`GripTransformBefore5f`) and logs `GRIPCHECK` once a second, ungated (:7927-7956) |
| `GetHandTransform` / `GetHmdTransform` / `GetHmdBaseTransform` | VK-XR:7800/7700/7747 | as above | per call | O(1) | Pure math, no xr calls |
| `FVRRig::Get` | r_data/vr_rig.cpp:607 | rig/avatar consumers | per call; Recompute when xrVRFrameIndex changes | Recompute: rig solve | Index ticks twice a frame → up to 2 recomputes/frame (INFERRED from the double SetUp) |
| `eye->AdjustHud` | VK-XR:2033 | eye loop :389/:483; Update :703 | 2× per eye | O(1) + viewpoint buffer slot | StartDrawInfo + SetViewpoint; skipped when the mounted HUD draws |
| `eye->AdjustBlend` | VK-XR:2051 | eye loop :399/:505; Update :695 | 2× per eye | O(1) + viewpoint slot | |
| `VulkanRenderDevice::PostProcessScene` | vk_renderdevice.cpp:805 | eye loop :398/:501 | per eye (also with multiview) | full PP chain at scene size | BlitSceneToPostprocess (resolve/blit), Pass1, 2D-after-bloom, Pass2 |
| `VkPostprocess::BlitSceneToPostprocess` | vulkan/renderer/vk_postprocess.cpp:131 | PostProcessScene | per eye (eye 1 skipped when layered) | 1 resolve/blit (+ light mask) | VRSceneTransfer clock |
| `VulkanRenderDevice::Update` eye composite | vk_renderdevice.cpp:668-716 | End2DAndUpdate | per eye | 2× Draw2D replay of `twod` | `suppressSceneEye2D` only in screen-layer/cinema frames |
| `FinalizeEyeImage` | VK-XR:7196 | Update :709 | per eye | 1 fullscreen pass at recommended size (+1 mirror pass on one eye, +1 spectator pass if `vr_spectator`) | `CreatePresentTextures` check; RigDumpEyeImage only with `rig_eyeshot` |
| `VkPostprocess::DrawPresentTextureToImage` | vk_postprocess.cpp:502 | FinalizeEyeImage, RenderVirtualScreen | 2-3×/frame | fullscreen pass | **Runs `customShaders.Run(...,"screen")` on every call** (:508-509), i.e. 3× a frame with a mirror pass. INFERRED double application on the mirrored eye when a mod has "screen" shaders (gl_custompost defaults true, hw_postprocess.cpp:2862) |
| `RenderSpectatorEye` | VK-XR:7372 | FinalizeEyeImage | 1/frame if vr_spectator>0 | 1 fullscreen pass + ≤20 small CPU iterations | |
| `RenderVirtualScreen` | VK-XR:6776 | Update :714 | menu/cinema frames | 3 swapchain acquire/wait/release, blits, a present pass, clears | Not in gameplay frames (`ShouldRenderVirtualScreen`, :6766) |
| `VkCommandBufferManager::WaitForCommands` | vk_commandbuffer.cpp:430 | Update :735/:739/:749 | 1-2×/frame | flush + CPU poll-wait all fences | Poll with `sleep_for(1ms)` (:219) |
| `AcquireXRSwapchain` | VK-XR:6133 | Update :745 | frame | xr + separate submit + CPU poll-wait | Separate command pool/buffer/fence (VK-XR:2497-2503); `vkResetCommandPool` each frame (:6261) |
| `VkFramebufferManager::AcquireImage` (VR branch) | vulkan/textures/vk_framebuffer.cpp:56-136 | WaitForCommands or Update :748 | frame | desktop swapchain acquire + clear + RenderDesktopMirror blits | Desktop vsync/acquire wait counted as pacing |
| `RenderDesktopMirror` | VK-XR:7501 | vk_framebuffer.cpp:111 | frame | 1-2 blits (+ overlay blits) | |
| `ProcessHaptics` | VK-XR:8066 | end of UpdateControllerState (×2), TearDown, every Vibrate*/menu early return | ≥3×/frame | xr: xrApplyHapticFeedback per active hand per call | |
| `TearDown` (mode) | VK-XR:5885 | hw_entrypoint.cpp:517 | frame | ProcessHaptics | |
| `WriteHandCommand` | VK-XR:8317 | G_BuildTiccmd (g_game.cpp:1527) | per tic | O(1) + `VRRig_StandEyeTic` | Reads cached poses, not live |
| `DrawHudToSurface` | d_main.cpp:1254 | D_Display :1693 | frame (VR, mounted HUD) | full status bar draw (ZScript) into a canvas, then a canvas render pass (hw_entrypoint.cpp:688-715) | Once per frame, shared by both eyes |
| `R_UpdatePoseAnchoredLights` | hw_entrypoint.cpp:185 | RenderViewpoint :330 | frame | O(anchored lights), relink when moved >4 units | Shadow of an anchored light lags one frame (CollectLights runs earlier) |

---

## 4. Data structures that matter for speed

- **Eye images.** Scene buffers: RGBA16F `SceneColor`/depth at `recommended × vr_openxr_render_scale` (clamped 0.25-4.0, VK-XR:2185), sized in `BeginFrame` (`vk_renderdevice.cpp:1467-1499`). Single-layer unless multiview (`eyeLayerCount`, :1489-1494). Pipeline images (PP ping-pong) come in pairs per eye (`SetPipelineImagePair((eye%2)*2, 2)`, :684-686; `NextEye`, :1397-1426).
- **`xrPresentTextures[viewCount]`.** Swapchain-format RGBA8 images at the *recommended* size (not render-scaled), written by the present pass (VK-XR:3019-3109).
- **`xrMirrorPresentTextures[viewCount]`.** The same, allocated for **both** eyes whenever a dedicated mirror is wanted (VK-XR:3062-3063, 3084-3099), though only one eye is drawn for `vr_desktop_view` 1/2 (`ShouldPrepareDesktopMirrorEye`, VK-XR:1133-1147).
- **`xrSwapchain`.** One array swapchain, `arraySize = viewCount`, size = max recommended, usage COLOR_ATTACHMENT|TRANSFER_DST, format = the desktop's UNORM unless `vr_eye_colorspace 1` (VK-XR:2933-3017). One `VulkanImage` and layer view per image per layer (:2993-3008). Nothing renders into it directly; it is only a blit target.
- **Separate XR command infrastructure.** `xrVkCommandPool`, `xrVkCommandBuffer`, `xrVkSubmitFence` (VK-XR:2497-2503; vk_openxrdevice.h:282-284). One submission per frame on the graphics queue, waited on the CPU.
- **Viewpoint buffer.** Per eye: one slot from ApplyViewpoint, one from AdjustHud, one from AdjustBlend; then AdjustBlend and AdjustHud again in Update; plus DrawEndScene2D (`hw_drawinfo.cpp:4396`) [xc: DrawCoronas removed: its only call is commented out (hw_drawinfo.cpp:4356-4359), so it takes no slot]. While multiview is active every single `SetViewpoint` writes 2 slots (`hw_viewpointbuffer.cpp:112-116`) [xc: added]. Multiview writes 2 per `SetViewpoints` (`hw_drawinfo.cpp:3385,3407`).
- **Dynamic-light caches keyed on `gl_dynlight_viewid`.** `mPosRelativeCacheViewId`, `mDistanceCullViewId` (`hwrenderer/data/hw_dynlightdata.h:65-127`) and `mSpotCacheViewId` (`rendering/hwrenderer/hw_dynlightdata.cpp:91-102`). Incremented per eye (`hw_entrypoint.cpp:361`), so every cache entry is recomputed for the second eye.
- **Hand ring.** `vrHandRing[2][48]` (`hw_vrmodes.cpp:1981-1984`), pushed 2× per SetUp (2 hands), i.e. 4 pushes/frame. `VR_HandLeverSelfCheck` runs per push until settled.
- **Controller state.** `xrHandPoses/xrGripPoses/xrGripOffset/xr*State` (vk_openxrdevice.h:329-400): small, per-frame copies.
- **Haptic mixer.** `xrHapticMix[2][4]` (vk_openxrdevice.h:431-439): fixed-cost.

---

## 5. What slows VR down relative to flat-screen (same scene), ranked

Evidence is code paths. Magnitudes are not measured here; §8 gives the measurement for each.

| # | Cause | Evidence | Confidence |
|---|---|---|---|
| 1 | **Full scene CPU build + GPU scene draw done twice.** BSP walk, clipping, wall/flat/sprite processing, list sort, draw-call recording, portals (each portal re-enters DrawScene), translucency, in-scene weapons/hands, SSAO when enabled. | Eye loop hw_entrypoint.cpp:359-516; `renderSceneThisEye = !useMultiviewScene ‖ eye_ix==0` (:372); multiview default off (hw_vrmodes.cpp:643) | VERIFIED (2× work); share of frame INFERRED dominant. The audit doc `docs/plans/RENDERER_AUDIT_2026-11-09.md` (renderer branch) ranks it first too. That is a lead, not evidence. |
| 2 | **BSP build single-threaded in VR.** Flat uses the scene worker thread (and optional wall workers); VR does not unless `vr_scene_multithread 1`. | hw_bsp.cpp:1438-1440; hw_cvars.cpp:278 (default false); gl_multithread default true hw_bsp.cpp:46 | VERIFIED |
| 3 | **Post-process chain twice, at headset eye resolution** (bloom, tonemap, custom PP, heat, smoke, emissive, exposure, lightmask resolve). Not shared even under multiview. | PostProcessScene per eye hw_entrypoint.cpp:398,501; PP passes ViewMask 0 vk_pprenderstate.cpp:112,159,171-173 | VERIFIED; pixel cost INFERRED ∝ eye pixels ×2 |
| 4 | **No CPU/GPU overlap; extra serialized sync points.** CPU waits for the whole frame's GPU work (`WaitForCommands`), then submits a separate blit and CPU-waits its fence, then `xrEndFrame`, then (sync mode 1) desktop acquire + flush + present + a third wait. The next frame's CPU work starts only after all of that. Flat has one wait. | vk_renderdevice.cpp:730-752; VK-XR:6340-6379; vk_commandbuffer.cpp:430-500 | VERIFIED structure; the cost (lost overlap ≈ min(CPU, GPU) per frame) INFERRED |
| 5 | **1 ms poll quantization on each fence wait** (2-3 per VR frame vs 1 flat). | `RS_WaitFencesPolled` sleep_for(1ms) vk_commandbuffer.cpp:219; used at :356, :484, VK-XR:6370 | VERIFIED; overshoot per wait up to one sleep quantum (≈1-2 ms with timeBeginPeriod(wPeriodMin), i_main.cpp:280-287) INFERRED |
| 6 | **Per-light caches invalidated per eye.** Dynamic-light relative position, distance cull and spot direction are recomputed for eye 2. | hw_entrypoint.cpp:361; hw_dynlightdata.h:65-127; hw_dynlightdata.cpp:91-102 | VERIFIED |
| 7 | **Extra fullscreen passes.** XR present pass ×2 (recommended size, even when render scale <1), mirror present pass ×1 (default `vr_desktop_view 2`, `vr_desktop_view_openxr_render 0`), swapchain blit ×2, desktop blit ×1(-2). Spectator +1 if enabled. | VK-XR:7219-7259, 6283-6321; vk_framebuffer.cpp:86-111; VK-XR:1149-1168 | VERIFIED |
| 8 | **Per-eye 2D replay and extra viewpoint setups.** `twod` replayed per eye (inside + outside 2D) in Update; AdjustHud/AdjustBlend ×4 per frame. | vk_renderdevice.cpp:680-710; VK-XR:2033-2074 | VERIFIED; small unless 2D is heavy |
| 9 | **VR CPU extras per frame.** SetUp ×2 (UpdateControllerState ×2 with ~30 xr calls each), status bar drawn into the HUD canvas + its render pass, render-rate `P_XYMovement` ×2 for roomscale (SP), hand ring, haptics ≥3 xr calls, `GRIPCHECK` double matrix per grip call, two ungated once-a-second log lines (`PAWNANCHOR`, `GRIPCHECK`). | VK-XR:3575-3655, 3906-5883, 7927-7956, 5864-5876; d_main.cpp:1254-1500 | VERIFIED presence; INFERRED small (sub-ms) except the status bar draw (ZScript, mod-dependent) |
| 10 | **Per-eye ProcessScene uploads.** `SyncDrawnLines` rebuilds every scene (2×), `SyncViewLights`, heat/smoke/emissive per-eye publish. Particle syncs are generation-gated (INFERRED cheap the second time). | hw_drawinfo.cpp:4617-4717 | VERIFIED calls; cost INFERRED |
| 11 | **Wheel/HUD per eye.** VRWheel RefreshEntries/UpdateHover per eye while open; mounted HUD quad per eye. | hw_vrwheel.cpp:2257-2260, 2411-2417; VK-XR:6525 | VERIFIED |
| 12 | **Missing standard VR savings** (not a regression vs flat, but headroom): no visibility mask (lens corners shaded), no fixed foveation/VRS, no multiview by default, no depth layer for runtime ASW/SSW, render-scale-independent present resolution. | grep: no `XR_KHR_visibility_mask`, `XR_FB_foveation`, `VK_KHR_fragment_shading_rate`, `XR_KHR_composition_layer_depth` in src; DirectXrRender/FinalizeLayered hard-false VK-XR:1182-1183 | VERIFIED absence |

Previously measured (flat-screen only, not VR; `HANDOFF_RENDER_2026-11-11.md` on renderer): "of 14.5 ms, scene build is 4.7, GPU wait 3.6 … playsim 1.7, script 2.0". `docs/plans/RENDERER_REBUILD.md:321` states "9.7 ms is one view at desktop resolution; two eyes at headset resolution break 13.9 ms on every map". That is a projection in the source, not a VR measurement. No VR per-eye measurement was found in those files.

---

## 6. Hooks, cvars and dead code in this area

### 6.1 Cvars read on the VR path

| Cvar | Default | Defined | Read at | Status |
|---|---|---|---|---|
| `vr_mode` | 0 | hw_vrmodes.cpp:605 | GetVRMode :1234 | live; 15 = OpenXR |
| `vr_openxr_multiview` | **false** | hw_vrmodes.cpp:643 | VK-XR:8610, 1177 | DORMANT (menu: menudef.txt:3294,5059,5189) |
| `vr_openxr_multiview_postprocess` | true | hw_vrmodes.cpp:655 | vk_renderdevice.cpp:676,816,1407; vk_postprocess.cpp:140 | only meaningful with multiview |
| `vr_openxr_multiview_mirror_reuse` | true | hw_vrmodes.cpp:654 | VK-XR:1161 | only with multiview |
| `vr_scene_multithread` | **false** | hw_cvars.cpp:278 | hw_bsp.cpp:1438 | DORMANT; flat equivalent on |
| `vr_openxr_render_scale` | 1.0 | hw_vrmodes.cpp:649 | VK-XR:2185, 1194 | live (scene size only, not present/swapchain) |
| `vr_openxr_sync_mode` | 1 | hw_vrmodes.cpp:641 | vk_renderdevice.cpp:731 | live; 1 = defer desktop present after xrEndFrame |
| `vr_desktop_view` | 2 | hw_vrmodes.cpp:628 | VK-XR:1133-1168, 7503-7635; vk_framebuffer.cpp:87 | live; -1 disables mirror work |
| `vr_desktop_view_openxr_render` | false | hw_vrmodes.cpp:653 | VK-XR:1154 | live; true removes the mirror present pass |
| `vr_spectator`, `vr_spectator_fov` | (p_vrdemo.cpp) | — | VK-XR:7239, 7416 | live when >0 |
| `vr_openxr_fov_adjust_deg` | 0 | hw_vrmodes.cpp:638 | VK-XR:1334, 1944 | live |
| `vr_openxr_debug_submit_mode` | 0 | hw_vrmodes.cpp:640 | VK-XR:6285, 6401 | debug |
| `vr_eye_colorspace` | 0 | hw_vrmodes.cpp:693 | VK-XR:2966 | live (swapchain format) |
| `vr_openxr_present_*_bias` (4) | 0.886/0.85/-0.15/1.15 | hw_vrmodes.cpp:661-664 | vk_postprocess.cpp:481-484 | live |
| `vr_menu_keep_world` | true | VK-XR:217 | VK-XR:3544 | live; menu frames still render stereo world |
| `vr_ipd` | 0.064 | hw_vrmodes.cpp:623 | VK-XR:1967, 2235 | live; **overrides the runtime IPD for eye placement**. Per-eye runtime orientation is not used either: `AdjustViewpointUniforms` only runs CalcDependencies (VK-XR:1975-1981), so both eyes look along the centered head axis. The HUD projection (VK-XR:2228-2256) adds the `vr_ipd` shift ×`vr_hud_stereo` (default 1.4, hw_vrmodes.cpp:1089) *and* the runtime eye-to-head offset (`currentEyePose − hmdPosition`, :2249-2255) [xc: added] |
| `vr_posestats`, `vr_grip_debug`(true), `vr_haptic_debug`, `vr_body_debug`, `vr_height_diag`, `vr_handcmd_debug`, `vr_rig_pose_*`, `vr_rig_hmd_*` | — | VK-XR:135,138,155,268,989,8299,282-287 | as named | debug; `vr_grip_debug` defaults **true** (prints on change only) |
| `r_effects_foveated` | true | hw_postprocess_cvars.cpp:391 | hw_drawinfo.cpp:1434,1568 | live; march step reduction, not XR foveation |
| `vid_maxfps` | 500 | v_video.cpp:75 | FPSLimit via WaitForCommands(finish) when vsync off (vk_commandbuffer.cpp:446-447) | DORMANT risk: a low value would throttle VR frames (INFERRED) |

### 6.2 DEAD (with evidence)

| Item | Evidence | Verdict |
|---|---|---|
| `vr_openxr_eye_shift_scale` | hw_vrmodes.cpp:639 (def), VK-XR:110 (EXTERN); no C++ read. Exposed as a slider in the Developer Options menu (wadsrc/static/menudef.txt:2235, inside `ifOption(OpenXR)`) and reset at :2656; no ZScript use | MENU-ONLY: the engine never reads it, so the menu slider does nothing [xc: was DEAD with "only 2 hits"; MENUDEF uses it] |
| `vr_openxr_force_recommended_viewport` | hw_vrmodes.cpp:646 (def); no C++ read; Developer Options option menudef.txt:2236, reset :2656 | MENU-ONLY, no engine effect [xc: was "single hit"; MENUDEF uses it] |
| `vr_openxr_use_screen_viewport_for_submit` | hw_vrmodes.cpp:652 (def, default true); no C++ read; Developer Options option menudef.txt:2237, reset :2656 | MENU-ONLY, no engine effect [xc: was "single hit"; MENUDEF uses it] |
| `xrVirtualScreenBackdropEquirectLayer` | Filled every menu frame when `xrHasEquirectBackdrop` (VK-XR:6748-6762) [xc: condition added] but never put in `layers[]` (VK-XR:6423-6461) | DEAD (built, never submitted) |
| `xrMultiviewProbed`, `xrMultiviewUsesCoreVulkan`, `xrMultiviewMaxInstanceIndex` | Written VK-XR:3436-3440, 8577-8599; no reads (grep) | DEAD fields |
| `VRMode::SupportsMultiview` / override | Declared hw_vrmodes.h:197, vk_openxrdevice.h:177; no caller | DEAD hook |
| `VKOpenXRDeviceMode::Present` | Only caller `gl_postprocess.cpp:170` (GL backend); VK-XR:6482 prints only | DEAD on Vulkan |
| `VRBenchmarkInfo::FinalizeLayered`, `DirectXrRender` | Hard false VK-XR:1182-1183 | DEAD flags (only printed) |
| `gl/stereo3d/gl_openxrdevice.cpp` | Not in any CMake source list; only a `source_group` regex (src/CMakeLists.txt:1818) | DEAD file (header still included by hw_vrmodes.cpp:42) |
| `FOpenXRJoystickManager::AddAxes` | Stub `(void)axes` (common/platform/win32/i_openXR.cpp:259-262 [xc: path was missing `common/`]) | DEAD input path (device registered, contributes no axes) |
| `LogBoundSourcesForAction` | Defined VK-XR:1763; the call was removed (comment VK-XR:2860-2864); grep: no caller | DEAD function |
| `DrawCoronas` loop | `#if 0` hw_drawinfo.cpp:4300-4327; caller commented out :4356-4359 | DEAD |
| `stabilizeGeometryOk` | Constant false VK-XR:4554; the branch at :4698 is reachable only through `supporting` | partially dead |
| `for (uint32_t i...) {}` / empty view-state check | VK-XR:6117-6122 | DEAD code (empty bodies) |

### 6.3 DORMANT (reachable, off by default)

Multiview (whole layered path: hw_entrypoint.cpp:453-473, vk_renderdevice.cpp:1490-1499,1643-1657, vk_postprocess.cpp:136-147), `vr_scene_multithread`, spectator view, `vr_teleport` (per-frame `P_LineTrace`), virtual-screen backdrop/beam swapchains (menu/cinema only), `XrPerfTrack` metrics (only while `perf_track>0`, VK-XR:6059-6070), OpenVR mode 10 (GL).

### 6.4 Not present at all (grep, VERIFIED absence)

XR foveation (`XR_FB_foveation*`), Vulkan fragment shading rate / VRS, eye tracking (`XR_EXT_eye_gaze_interaction`), visibility mask (`XR_KHR_visibility_mask`), depth composition layer, quad views, space warp / motion vectors. The search terms are listed in §5 #12.

---

## 7. Branch differences

| Area | renderer (175063a) | netplay (1c1805f8) | selaco (7e111a3d) |
|---|---|---|---|
| VK-XR | 17 changed lines: `IsGameplaySceneActive` also requires `gameaction != ga_fullconsole` (renderer VK-XR:669-670); STANDEYE call removed | 75 lines: input gates use `multiplayer` instead of `VR_InputViaCommand()`; analog/net values written into **`xrLast*` edge-detector fields** before the edge comparison (netplay VK-XR:3954-3961), the bug main's comment at main VK-XR:3959-3965 describes as having killed the triggers; no `roomdx/roomdy/turn` in the command | 6825 lines (vs 8632): no spectator, no XrPerfTrack, no EffectsGovernor, no GetGripTransform/GRIPCHECK, no PAWNANCHOR, no hand ring push |
| Eye loop (hw_entrypoint) | same + Tracy zones `RenderViewpoint`, `Eye` | identical | same structure (selaco hw_entrypoint.cpp:248-266) |
| Double SetUp | same | same | same (selaco vk_renderdevice.cpp:1289-1290 + hw_entrypoint.cpp:231) |
| GPU waits | same polled waits; `GpuWait` Tracy zone around WaitForCommands | same | **`vkWaitForFences(UINT64_MAX)`**: blocking, no 1 ms poll (selaco vk_commandbuffer.cpp:170,253; XR blit selaco VK-XR:5388) |
| vr_scene_multithread / multiview defaults | false / false | false / false | false (selaco hw_cvars.cpp:253) / false (selaco hw_vrmodes.cpp:668) |
| Static level mesh | `r_newrenderer` (default 0, hw_levelresidency.cpp:137); walls/flats from the mesh are drawn inside `RenderScene` (renderer hw_drawinfo.cpp:3830,3835), i.e. **still per eye**. BSP walk unchanged and still per eye; `levelmesh.vp` reads `gl_ViewIndex` (wadsrc/static/shaders/glsl/levelmesh.vp:118), so it is multiview-ready. Also `r_wallcache` (default 0, renderer hw_wallcache.cpp:40): per-side replay inside the BSP walk (renderer hw_bsp.cpp:344-350, 759-764); the parts are built view-independently, so eye 2 can replay eye 1's work (INFERRED from the header comment and the call sites, not measured); with it on, the wall worker threads are bypassed (renderer hw_bsp.cpp:742-743) [xc: added] | — | — |
| hw_clock summary | flat fallback for Scene/SyncWait when VR clocks are 0 (renderer hw_clock.cpp:163-168) | same as main | — |

---

## 8. Open questions and the measurements that settle them

Counters available (all `glcycle_t` = **CPU** cycle timers, `common/engine/stats.h:260`). They are reset in `ResetProfilingData` (hw_clock.cpp:80), which runs inside `RenderView` (hw_entrypoint.cpp:669) *after* `BeginFrame`. So **xrWaitFrame and SetUp #1 are in none of them**. Print them with `stat rendertimes` (hw_clock.cpp:261) or the `bench` output; the header carries XR flags (hw_clock.cpp:318-350).

| Counter | Covers | Does not cover |
|---|---|---|
| `VRSceneEyes` (hw_entrypoint.cpp:370-515) | CPU of the whole eye-loop body, both eyes summed | SetUp #2, shadow map, compute prep before the loop |
| `VRSceneBuild` / `VRScenePostBSP` (hw_drawinfo.cpp:3683-3710) | CreateScene CPU (BSP + lists + hacks), per eye summed | worker-thread time when multithreaded (see `SceneWorkerElapsed`, `MTWait`) |
| `VRSubsectors`, `VRSubsectorCull/Visible`, `VRLineBuild/Clip/Decide`, `VRThingBuild`, `VRFlatBuild` (hw_bsp.cpp) | BSP sub-phases | |
| `VRSceneDraw` (hw_drawinfo.cpp:3725-4032) | RenderScene + RenderTranslucent command recording | GPU execution |
| `VRPlayerSprites` (hw_bsp.cpp:1560) | PreparePlayerSprites | the in-scene draw of the psprites |
| **Summary `Scene` figure** (`GetRenderTimeSummary`, hw_clock.cpp:157), read by `stat rendertimes`, `r_perflog` (hw_perflog.cpp:791) and `perf_track` (hw_perftrack.cpp:525) | `VRSceneBuild + VRSceneDraw + VRScenePostBSP + VRPlayerSprites` | **double-counts**: `VRScenePostBSP` (hw_drawinfo.cpp:3700-3705) and `VRPlayerSprites` (hw_bsp.cpp:1560-1562, inside RenderBSP) both run inside the `VRSceneBuild` span (hw_drawinfo.cpp:3683-3710). Use the individual counters [xc: added] |
| `VRPostProcessScene` (vk_postprocess.cpp:98), `VRSceneTransfer` (:149) + `VRSceneTransferOps` | PP recording, scene→pipeline transfer recording | GPU |
| `VREyeComposite` (vk_renderdevice.cpp:679-711) | per-eye 2D + FinalizeEyeImage recording | |
| `VRFinalizeEye` (VK-XR:7208), `VRFinalPresent` (VK-XR:7222, 7246) + `VRFinalPresentPasses`, `VRMirrorPreparePasses` | present/mirror pass recording | GPU |
| `VRRenderSyncWait` (vk_renderdevice.cpp:734) | the main CPU-waits-GPU in Update (≈ GPU frame time minus overlap) | the deferred third wait (that one is in `Finish`) |
| `VRSubmit` (VK-XR:6260) | blit record + submit + fence poll + release + xrEndFrame | xrAcquire/xrWaitSwapchainImage (before the timer) |
| `VRSubmitCopy` (VK-XR:6282) + `VRSubmitLayerBlits` | blit recording | |
| `VRSubmitWait` (VK-XR:6187, 6369) | **sum** of xrWaitSwapchainImage and the XR blit fence poll | cannot separate the two |
| `perf_track` columns `XrWait`, `XrAcquire`, `XrImageWait`, `XrEnd`, `XrPeriod`, `XrAhead` (Win32 only), `XrAppGpu`/`XrCompGpu`/`XrCompCpu`/`XrFps`/`XrAppDropped`/`XrCompDropped`/`XrStale` (needs `XR_META_performance_metrics`) | hw_perftrack.h:115-134; filled VK-XR:6059-6066, 6168-6195, 6469-6477 | AheadMs is zero off Windows or without the convert extension |
| `stat gpu` / `r_perflog` GPU groups `scene.opaque`, `scene.psprites`, `scene.portals`, `scene.translucent`, `scene.vrextras` (hw_drawinfo.cpp:4494-4587) | GPU time; DM_MAINVIEW only, so both eyes are summed into the same group names | the present/blit passes |

Questions:

1. **How big is the doubled scene build in a real VR session?** Run a headset session with `stat rendertimes` (or `perf_track 1`). Compare `VRSceneBuild` and `VRSceneDraw` (both eyes) against a flat run of the same demo/spot (`ProcessAll+RenderAll`). Then repeat with `vr_openxr_multiview 1` and with `vr_scene_multithread 1`. That separates "×2" from "single-threaded".
2. **How much frame time is lost to serialization?** `perf_track` `XrWait` plus `VRRenderSyncWait` plus `VRSubmitWait`, against `XrPeriod` (13.9 ms at 72 Hz). If `VRRenderSyncWait` ≈ GPU time while `XrWait` is near 0 and frames are missed, overlap would pay. The poll overshoot cannot be isolated with a cvar or env var (`REMA_RIG_FENCE_SECONDS` only changes the timeout, vk_commandbuffer.cpp:58-63); it needs an A/B build with blocking `vkWaitForFences` (selaco's code is a ready reference). Also A/B the XR blit: per the OpenXR Vulkan binding the app only has to have *submitted* its writes before `xrReleaseSwapchainImage`, so the CPU fence wait at VK-XR:6366-6379 is likely removable (INFERRED from the spec, not from code here).
3. **The real eye resolution.** Read the startup log line `OpenXR: view %u recommended %ux%u` (VK-XR:2888) and the bench header `XR Targets: recommended=… present=…` (hw_clock.cpp:336). The PP and present cost scale with this. Unknown for the owner's Quest 2 over VDXR.
4. **Is multiview correct enough to turn on?** INFERRED risks, read from the code: the scene is culled from the left eye's position and FOV only (hw_entrypoint.cpp:452-482), so the right eye's right edge may lose geometry; stereo billboards fall back to one image (rendering/hwrenderer/scene/hw_sprites.cpp:2587-2600); PP still runs per eye. Settle with a rig eyeshot of both eyes (`rig_eyeshot <path> 1`, VK-XR:294) at `vr_openxr_multiview 0/1`, diffed.
5. **Does SetUp #1 (stale predicted time) cost or harm anything?** It locates hands at last frame's time, posts input and moves the pawn (SP roomscale) before `xrWaitFrame`. Measure by clocking `VulkanRenderDevice::BeginFrame` (no counter exists today). Check the hand latency in `[A0]` stats (`vr_posestats 1`, VK-XR:4400-4501).
6. **Custom "screen" PP shaders running 3× a frame.** INFERRED from `DrawPresentTextureToImage` (vk_postprocess.cpp:508-509) and the mirror pass. Verify with `stat gpu` groups named after the shader (`PushGroup(Desc->Name)`, hw_postprocess.cpp:3187) in a mod that ships one.
7. **IPD.** Eye placement uses `vr_ipd` (0.064 m), not the runtime's eye poses (VK-XR:1967), while the submitted layer uses the runtime poses (VK-XR:6404). This is a comfort/correctness question, not cost. Settle by logging `xrViews[1].pose.position − xrViews[0].pose.position` against `vr_ipd`.
8. **Status-bar-to-canvas cost.** `DrawHudToSurface` (d_main.cpp:1254) is not inside any VR clock. Check `stat gpu` for the canvas pass and the ZScript cost in `perf_track` script columns.

---

## Cross-check (date 2026-11-13)

Checked against the code in main f182cbcc (plus renderer abb19af1 = 175063a + docs, netplay 1c1805f8, selaco 7e111a3d). Greps re-run over all of `src/` and `wadsrc/` (MENUDEF, ZScript, shaders) in each tree.

| Category | Checked | Right | Line/path-fixed | Corrected | Unconfirmed | Added |
|---|---|---|---|---|---|---|
| a. Summary claims | 24 | 21 | 1 | 2 | 0 | 2 |
| b. "What slows it down" items (ranking + evidence) | 12 | 12 | 0 | 0 | 0 | 0 |
| c. DEAD / DORMANT / never-read claims | 19 | 14 | 1 | 4 | 0 | 0 |
| d. Cvar defaults | 24 | 24 | 0 | 0 | 0 | 0 |
| e. Branch differences | 13 | 13 | 0 | 0 | 0 | 1 |
| f. Table rows (§2.2 graph, §3, §4, §6.1, §8) | 62 | 61 | 0 | 1 | 0 | 3 |

Totals: 154 checked; 145 right; 2 line/path-fixed; 7 corrected; 0 unconfirmed; 6 added.

Corrections:
- §6.2 `vr_openxr_eye_shift_scale`: not "only 2 hits"; it is a Developer Options slider (menudef.txt:2235) and in a resetcvar list (:2656). No C++ or ZScript read: menu-only, no engine effect, not DEAD-unreferenced.
- §6.2 `vr_openxr_force_recommended_viewport`: not a single hit; menudef.txt:2236 + :2656. Menu-only, no engine effect.
- §6.2 `vr_openxr_use_screen_viewport_for_submit`: not a single hit; menudef.txt:2237 + :2656. Menu-only, no engine effect.
- §4 viewpoint buffer: DrawCoronas takes no slot; its only call is commented out (hw_drawinfo.cpp:4356-4359). This contradicted §6.2.
- §1.1: "the only implementation compiled" was too broad; the GL OpenVR mode (gl_openvr.cpp) is compiled too. VK-XR is the only OpenXR one.
- §1.2: the 2D replay is not in the RenderViewpoint eye loop; it is the second per-eye loop in `Update` (vk_renderdevice.cpp:680-710).
- §6.2 equirect backdrop layer: filled only when `xrHasEquirectBackdrop` (qualifier added).
- Paths: `platform/win32/i_main.cpp` and `platform/win32/i_openXR.cpp` are under `common/`.

Additions:
- §1.3: all ~10 per-frame multiview decision sites; the conditions besides the cvar; the BeginFrame (SetUp #1) vs eye-loop (SetUp #2) mode split, with hw_entrypoint.cpp:354 not re-checking layer counts (INFERRED one-frame hazard).
- §1.6: the 1 ms sleep only runs when a fence is still unsignalled at the poll.
- §4: under multiview every `SetViewpoint` writes 2 slots (hw_viewpointbuffer.cpp:112).
- §6.1 `vr_ipd`: per-eye runtime orientation is unused too; the HUD projection applies both the `vr_ipd`×`vr_hud_stereo` shift and the runtime eye offset.
- §7 renderer: `r_wallcache` (default 0), a per-side replay that eye 2 can reuse (INFERRED), which bypasses wall workers when on.
- §8: the summary `Scene` figure (hw_clock.cpp:157) used by `stat rendertimes`/`r_perflog`/`perf_track` double-counts `VRScenePostBSP` and `VRPlayerSprites`, which run inside `VRSceneBuild`.

Confirmed as written (selected): `vr_openxr_multiview` default false (hw_vrmodes.cpp:643; selaco :668), menu-exposed (DORMANT, not dead); multiview builds the scene once (eye 1 is post-process-only, hw_entrypoint.cpp:372-407) and PP still runs per eye (ViewMask 0, vk_pprenderstate.cpp:112,159); without multiview the full ProcessScene/CreateScene/RenderBSP runs once per eye; `vr_scene_multithread` default false gating hw_bsp.cpp:1438-1439; SetUp runs twice per frame with SetUp #1 before `xrWaitFrame` (vk_renderdevice.cpp:1458-1459); scene eye offset from `vr_ipd` (VK-XR:1967) while the submitted layer uses runtime poses (VK-XR:6404); 3 polled waits per frame with sync mode 1; selaco uses blocking `vkWaitForFences`; netplay's `xrLast*` edge-field overwrite (netplay VK-XR:3954-3961); no foveation/VRS/visibility-mask/depth-layer/space-warp code anywhere in src.
