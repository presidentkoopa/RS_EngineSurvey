# 01 — The frame loop, timing and threading

Tree of record: `main` at `f182cbcc` (`/home/claude/uzdx_main`). Every path below is relative to that tree's `src/`
unless prefixed `renderer:`, `netplay:` or `selaco:` (trees `/home/claude/uzdxrema`, `/home/claude/uzdx_netplay`,
`/home/claude/uzdx_selaco`). Tags: **V** = VERIFIED (I read the code that does it), **I** = INFERRED (deduced; the basis
is stated). No millisecond figure in this file is mine; the few quoted numbers are marked "previously measured" with
their source file and build.

---

## 1. Summary

1. One thread does everything that matters: input, netcode, every playsim tic, scene build, command recording, every
   GPU wait, OpenXR wait/begin/end, sound update. `D_DoomLoop` (d_main.cpp:1966) is `I_SetFrameTime → TryRunTics →
   I_StartTic/D_ProcessEvents → D_Display → S_UpdateMusic`, strictly serial (V).
2. The tic runs **before** `xrWaitFrame`. `xrWaitFrame/xrBeginFrame/xrLocateViews` sit inside `screen->BeginFrame()`
   in `D_Display` (vk_renderdevice.cpp:1458-1459 → vk_openxrdevice.cpp:6051-6087), and `xrEndFrame` sits inside
   `screen->Update()` (vk_renderdevice.cpp:745 → vk_openxrdevice.cpp:6471) (V).
3. The CPU waits for **all** of a frame's GPU work before the next loop iteration starts: no frame overlap. In VR with
   the default `vr_openxr_sync_mode 1` there are three full CPU←GPU waits per frame (scene, XR blit copy, desktop
   mirror), each a 1 ms-sleep polling loop (`RS_WaitFencesPolled`, vk_commandbuffer.cpp:181-221) (V).
4. On frames with no tic due (≈ half of all frames at 72 Hz vs 35 tics/s) `TicStabilityWait` **busy-spins** for the
   duration of the last tic (`r_ticstability`, default on, d_net.cpp:2251-2271, 2431-2436) (V). Exactly: it runs when
   `totalTics <= 0` (the tic clock did not advance since the last frame; not on network-stall frames, not when
   `pauseext`, which returns at :2387-2388); it spins on unscaled `steady_clock` for `stabilityticduration`, which is the
   wall time of `G_Ticker`+`MakeConsistencies` of the last tic of the most recent batch whose `gameaction` passed
   `ShouldStabilizeTick` (:2286-2292, 2477-2493) — not `P_UnPredictClient`/`P_PredictClient`/client-side logic/sounds —
   capped at 1 s (:2283). Nothing shortens it when the next tic falls due during the spin, and nothing relates it to
   the frame deadline; the only bound is the 1 s cap (and in practice the spin itself makes the next frame owe a tic).
   [xc: precised what is timed, when it runs and what bounds it]
5. Catch-up on `main` is bounded only by the netcode buffers (≤ 35 commands built per call, +1 tic a frame while
   behind), so a tic slower than 28.6 ms spirals to ~35 tics per frame; `sim_maxcatchup` (default 2) exists only on
   `renderer` (mechanism V; the spiral is I from the arithmetic).
6. Single player still serialises and deserialises a prediction rollback snapshot of the local pawn on every frame that
   runs tics (`P_UnPredictClient`/`P_PredictClient`, no netgame guard, p_user.cpp:2226/2345) (V).
7. In VR, `VKOpenXRDeviceMode::SetUp` runs **twice** per frame (BeginFrame and RenderViewpoint): two `xrSyncActions`,
   two controller locates (the first at the previous frame's predicted time), three `updateHmdPose`, and, in single player, two render-side
   pawn moves (`VR_ApplyRenderMove`) and two render-side writes of sim fields (`VRMode::SetUp`) (V). Plus one mono
   `VRMode::SetUp` per camera-texture view (hw_entrypoint.cpp:727 → :327 with `GetVRModeCached(false)` = mono mode,
   hw_vrmodes.cpp:1234), which in single player writes `PlayInVR = false` and the head-height fallback attack pose
   before SetUp #2 writes the VR values back (hw_vrmodes.cpp:1505-1556); `AttackDir`/`OffhandDir` are written on every
   SetUp in multiplayer too (:1506-1507). Every render-side move made while prediction is on is queued and made again at
   the start of the next tic (`PredictRedo`, playsim/p_vrdemo.cpp:509-510, 623-635, via g_game.cpp:1995), so each
   single-player room move costs `P_XYMovement` at render time and again in the tic. [xc: added]
8. Interpolation fraction is taken from the loop-top clock (`I_SetFrameTime`, i_time.cpp:90), not from the headset's
   predicted display time; the HMD pose is from the predicted display time (V; consequence I).
9. Threads started: Windows hang watchdog, 8 ctpl BSP pool threads (idle in VR: `vr_scene_multithread 0`), texture
   loader threads (≤2) + 1 model loader, OpenAL stream thread; optional SolarXR reader, DAP debug server. None of them
   runs sim or VR work (V).
10. Clocks: `perf_track`, `r_perflog`, `stat rendertimes` (glcycle_t), governor pacing, Tracy (renderer, compile-off),
    `simprofile`/`ticprofile` (renderer only). None of them covers `TicStabilityWait`, the 1 ms polling quantum, or
    the loop-top-to-xrWaitFrame interval as such (the spin is inside perf_track `FrameMs`/`CostMs`, inside renderer
    `simprofile` "whole", renderer:src/d_net.cpp:2375-2376, and inside renderer's Tracy `TryRunTics` zone,
    renderer:src/d_main.cpp:2012, but nothing isolates it [xc: added where it is hidden]); `perf_track` CPU columns read 0 unless `r_perflog>0` or
    `stat rendertimes` is on (V).

---

## 2. Entry points and call graph

### 2.1 Process start to the first frame

| Step | Where | Notes | Tag |
|---|---|---|---|
| `wWinMain` | common/platform/win32/i_main.cpp:676 | captures `MainThreadID`, installs `CatchAllExceptions`, calls `DoMain` (:763) | V |
| `DoMain` | i_main.cpp:184 | `timeBeginPeriod(TimerPeriod)` (:286; `wPeriodMin`, normally 1 ms) — this is what makes every `sleep_for(1ms)` below ~1 ms on Windows; creates window; `GameMain()` (:337) | V |
| `main` (Linux/SDL rig) | common/platform/posix/sdl/i_main.cpp:184 | `SDL_Init`, `GameMain()` (:243) | V |
| `GameMain` | d_main.cpp:5190 | `GameTicRate = TICRATE`, `I_InitTime()` (:5206, i_time.cpp:53), `C_InitCVars`, `D_DoomMain_Internal()` | V |
| `D_DoomMain_Internal` | d_main.cpp:4807 | `I_HangWatchStart()` + 90 s timeout (:4815-4817); IWAD pick with watchdog unbounded (:5063-5065); `D_InitGame` (:5117); `D_DoAnonStats` (compiled empty); DAP server if `should_debug` (:5129-5142); `D_DoomLoop()` (:5145) | V |
| `D_InitGame` | d_main.cpp:4184 | file system, `S_Init`, `TexMan.Init`, `PClassActor::StaticInit` (ZScript), `R_Init`, `P_Init`; `D_CheckNetGame()` with watchdog unbounded (:4589-4594, the netgame lobby blocks here); `V_Init2()` (:4624, creates the Vulkan device and its loader threads); `while(!screen->CompileNextShader())` (:4694, synchronous pipeline build); title / `G_InitNew` / demo setup | V |

### 2.2 One loop iteration (= one displayed frame), exact order on `main`

`D_DoomLoop` d_main.cpp:1966-2030. Everything here is the main thread (V).

```
D_DoomLoop                                              d_main.cpp:1966
 ├ (before the loop: I_HangWatchStart again :1983, a no-op after :4815 — g_running guard, i_hangwatch.cpp:261)
 ├ I_HangWatchSetTimeout(12 s), I_HangWatchBeat          :1993-1994   (Win only; 2 relaxed atomics) [xc: line fixed]
 ├ if (gametic > lasttic) I_StartFrame()                 :1999-2002   (joystick poll, only after a tic ran)
 ├ I_SetFrameTime()                                      :2003        → i_time.cpp:90  freezes "now" for the frame
 ├ TryRunTics()                                          :2006        → d_net.cpp:2339  (section 2.3)
 ├ I_StartTic(); D_ProcessEvents()                       :2010-2011   window msgs (SleepEx(0,TRUE) i_input.cpp:583), kbd/mouse
 ├ D_Display()                                           :2013        → d_main.cpp:1560
 │  ├ GC::CheckGC()                                      :1566        (second GC check of the frame)
 │  ├ SetDeltaTime(); SolarXR_Tick()                     :1568-1572
 │  ├ [return if !AppActive && flat]                     :1578-1582
 │  ├ cycles.Clock() (FrameCycles)                       :1586-1589
 │  ├ R_SetFOV, setmode/setsize, wipe decision           :1597-1677
 │  ├ screen->FrameTime = I_msTimeFS(); FrameCount++     :1681-1682
 │  ├ TexAnim.UpdateAnimations; R_UpdateSky(TicFrac)     :1683-1684
 │  ├ screen->BeginFrame()                               :1685 → vk_renderdevice.cpp:1449
 │  │  ├ [VR] vrmode->SetUp()        #1                  :1458 → vk_openxrdevice.cpp:3575
 │  │  │   ├ ++xrVRFrameIndex                            :3581
 │  │  │   ├ PollXREvents()  (ONE event per call)        :3625 → :3657
 │  │  │   ├ UpdateControllerState()                     :3628 → :3906  xrSyncActions (:3922), hand/grip xrLocateSpace
 │  │  │   │     at xrFrameState.predictedDisplayTime of the PREVIOUS xrWaitFrame (:4158, :4362); VR_ApplyRenderMove (:5850)
 │  │  │   └ updateHmdPose(r_viewpoint)                  :3653 (uses the previous frame's xrViews)
 │  │  ├ [VR] BeginXRFrame()                             :1459 → :6006
 │  │  │   ├ xrWaitFrame          ← BLOCKS (headset pacing)        :6051
 │  │  │   ├ xrBeginFrame                                           :6075
 │  │  │   ├ xrLocateViews(predictedDisplayTime)                    :6087
 │  │  │   └ updateHmdPose(r_viewpoint)   #2                        :6124
 │  │  ├ UpdateBackgroundCache(false)  (upload loader output)       :1486
 │  │  └ cmd/texture/screenbuffer/lightmask/renderstate BeginFrame  :1487-1506
 │  ├ DrawHudToSurface(vp)                               :1693 (VR HUD canvas)
 │  ├ D_Render(RenderView, interpolate=true)             :1695 → :690
 │  │  ├ per level: HasDynamicLights; interpolator.DoInterpolations(I_GetTimeFrac()) :700;
 │  │  │            P_FindParticleSubsectors :701 (O(subsectors+particles)); PO_LinkToSubsectors :702
 │  │  ├ RenderView(player)                             rendering/hwrenderer/hw_entrypoint.cpp:640
 │  │  │  ├ CheckBenchActive; ResetProfilingData (All.Clock)        :666-669
 │  │  │  ├ TicFrac = cl_capfps||r_NoInterpolate ? 1 : I_GetTimeFrac() :672
 │  │  │  ├ canvases that changed (RenderTextureView)                :689-715
 │  │  │  ├ camera textures (RenderViewpoint, mono)                  :719-728
 │  │  │  └ RenderViewpoint(main)                                    :747 → :264
 │  │  │     ├ R_SetupFrame (TicFrac, view interpolation)           :268 → rendering/r_utility.cpp:1093
 │  │  │     ├ emissive volumes / beam lights / effect lights / shadow map  :274-312
 │  │  │     ├ vrmode->SetUp()   #2   (UpdateControllerState again, now at THIS frame's predicted time; updateHmdPose #3) :327
 │  │  │     ├ RunFrameCompute, exposure impulses                   :333-345
 │  │  │     └ per eye (or once, multiview): eye->SetUp, StartDrawInfo, SetupView, ProcessScene (BSP, :482), post-process, TearDown :359-516
 │  │  └ per level: interpolator.RestoreInterpolations()           :706-709
 │  ├ 2D: status bar / automap / HUD / CT_Drawer / pause         :1700-1912
 │  └ End2DAndUpdate()  (or PerformWipe)                         :1919 → :1533
 │     ├ twod->End(); screen->Update()                           :1535-1536 → vk_renderdevice.cpp:657 (section 2.4)
 │     ├ CheckBench(); EffectsGovernor::EndFrame()                :1537-1540
 │     ├ if r_perflog>0 PerfLogEndFrame(); if perf_track PerfTrackEndFrame() :1543-1546
 │     └ twod->OnFrameDone()                                      :1547
 ├ M_TickDeferredScreenshot()                            :2016
 └ S_UpdateMusic()                                       :2018
```

### 2.3 TryRunTics and one tic

`TryRunTics` d_net.cpp:2339 (V for every line cited):

```
TryRunTics
 ├ GC::CheckGC()  (inside PerfTrack::Scope REASON_GC)                 :2343-2346
 ├ doWait = cl_capfps || pauseext || (!netgame && r_NoInterpolate && !M_IsAnimated())   :2354
 │   VR active → doWait=false :2357-2359;  vid_dontdowait → false :2361-2362;
 │   flat && !AppActive && vid_lowerinbackground → true :2364-2365
 ├ EnterTic = doWait ? I_WaitForTic(LastEnterTic) : I_GetTime()       :2368-2371
 ├ totalTics = EnterTic - LastEnterTic   (singletics: never true, see §6)  :2374-2376
 ├ NetUpdate(totalTics)                                                :2382 → :1430
 │   ├ GetPackets(); NetState_Pump(); Host_LateJoinTick()              :1432-1434   (every frame)
 │   ├ return if tics<=0                                               :1435-1436
 │   ├ netgame: SendHeartbeat (host), CheckConsistencies               :1438-1455
 │   ├ level-start gate (LST_WAITING/LST_HOST)                         :1460-1504
 │   ├ tics = min(tics, MAXSENDTICS*TicDup) ; cap at BACKUPTICS/2 ahead :1508-1516   (MAXSENDTICS=35, BACKUPTICS=175: common/engine/i_net.h:90-94)
 │   ├ for each tic: [flat only] I_StartTic (:1522-1526); D_ProcessEvents (:1527);
 │   │               G_BuildTiccmd(&LocalCmds[ClientTic++]) (:1537) → g_game.cpp:1106 → VR: WriteHandCommand (g_game.cpp:1527 → vk_openxrdevice.cpp:8317)
 │   ├ pack + send (single player: to LocalNetBuffer)                  :1600-1890
 │   └ GetPackets()   (self-delivery, decode command)                  :1894
 ├ LastEnterTic = EnterTic                                             :2384
 ├ pauseext → return (no spin, no tics, no prediction)                 :2387-2388   [xc: added]
 ├ lowestSequence over NetworkClients; availableTics = lowestSequence - gametic/TicDup + 1   :2392-2416
 ├ runTics = min(totalTics, availableTics); +1 if behind               :2420-2426
 ├ runTics <= 0 (no tic this frame):                                   :2431
 │   ├ totalTics<=0 → TicStabilityWait()   BUSY SPIN                   :2433-2436
 │   ├ else (network stall) → P_ClearLevelInterpolation() O(actors); LagState=WAITING   :2438-2442
 │   ├ ClientTic advanced → P_PredictClient()                          :2446-2450
 │   ├ P_RunClientSideLogic() × totalTics                              :2454-2455
 │   └ totalTics>0 → S_UpdateSounds; VerifyPredictedEntities           :2457-2461
 └ runTics > 0:
     ├ P_UnPredictClient()   (deserialize rollback, even single player) :2474 → playsim/p_user.cpp:2345
     ├ while (runTics--):  TicStabilityBegin; [D_DoAdvanceDemo]; G_Ticker(); MakeConsistencies(); ++gametic; PerfTrack::NoteTic; TicStabilityEnd   :2475-2498
     ├ P_PredictClient()     (serialize rollback, even single player)   :2500 → p_user.cpp:2226
     ├ P_RunClientSideLogic() × totalTics                              :2504-2505 → p_tick.cpp:155
     ├ S_UpdateSounds(...)                                             :2509 → sound/s_doomsound.cpp:988
     └ NetworkEntityManager::VerifyPredictedEntities()                 :2510
```

`G_Ticker` g_game.cpp:1983 (V): `C_TickQueuedInputs` (:1987) → VR demo frame capture/replay (:1995-1999) → reborn /
`G_DoPlayerPop` loop over MAXPLAYERS=64 (:2003-2012) → `gameaction` loop (synchronous load/save/new game/world done,
:2016-2098) → `BotInfo.Main` (:2105) → per client `RunPlayerCommands` + copy `ClientStates[c].Tics[curTic].Command` into
`players[c].cmd` (:2108-2127) → `C_RunDelayedCommands` (:2130) → `P_Ticker()` (:2137) → `VRDemo_SetInTic(false)` (:2160).

`P_Ticker` p_tick.cpp:644, exact stage order (V):

| # | Stage | Line | Per | Notes |
|---|---|---|---|---|
| 1 | `interpolator.UpdateInterpolations()` per level | :648-651 | tic | sector/polyobj interpolation snapshots |
| 2 | `r_NoInterpolate = true` | :652 | | |
| 3 | paused/menu: loop all actors for `MF8_RECREATELIGHTS`, return | :662-679 | tic, O(actors) | runs every paused tic |
| 4 | `DPSprite::NewTick()` | :682 | | |
| 5 | freeze toggle, `S_ResumeSound` | :686-710 | | |
| 6 | `P_ResetSightCounters`, `R_ClearInterpolationPath`, `r_NoInterpolate=false` | :711-715 | | |
| 7 | per level `AdvanceWorldClock()` | :724 | | fork world clock (slow-mo) |
| 8 | **actor interpolation reset**: every actor `ClearInterpolation`, `ClearFOVInterpolation`, `modelData->ShiftSurfacePositions` | :729-757 | tic, O(actors) | one full pointer-chased walk |
| 9 | `P_ThinkParticles` (world step) | :760 | O(particles) | |
| 10 | `TickBillboards`; stamps; beams snapshot (MAX_BEAMS loop); `SnapshotDrawnLines` | :762-838 | O(billboards+beams) | |
| 11 | `P_StepBeamOutlines`, `BeginEffectTic` | :856-863 | | |
| 12 | `P_PlayerThink` per player | :865-867 | | |
| 13 | `localEventManager->WorldTick()` | :872 | | mod ZScript handlers, every real tic |
| 14 | world step: `Level->Tick()`, **`Thinkers.RunThinkers`**, `P_UpdateSpecials`, time++, `WorldStep()` | :878-897 | O(thinkers) | dthinker.cpp:149 |
| 15 | not world step: `P_TickRealTimeActors` (another full iterator walk + inventory chains) | :902 → :79 | O(actors) | only in slow-mo |
| 16 | `realtime++`, world-step-per-second bookkeeping | :908-915 | | |
| 17 | `VRSyncLog_OnTick()` | :919 | O(actors)+fprintf when `vr_synclog` set | dormant by default |

`MakeConsistencies` (d_net.cpp:1286), netgame only: `WorldStateHash` (:1235) walks every sector-linked actor and
FNV-1a-hashes 72 bytes each byte by byte, once per tic (V).

### 2.4 Where the VR frame calls sit relative to tic and render (`main`, `vr_openxr_sync_mode 1`)

`VulkanRenderDevice::Update` vk_renderdevice.cpp:657 (V):

```
Update
 ├ per eye: Draw2D, AdjustHud, FinalizeEyeImage           :679-711   (VREyeComposite clock)
 ├ RenderVirtualScreen                                     :714
 ├ EndRenderPass; EndFrame                                 :725-726
 ├ WaitForCommands(finish=false,...)  ← CPU waits ALL GPU work of the frame (VRRenderSyncWait clock)   :733-736
 │      → vk_commandbuffer.cpp:430: FlushCommands; RS_WaitFencesPolled (1 ms sleeps) :484
 ├ UpdateGpuStats                                          :741
 ├ AcquireXRSwapchain()                                    :745 → vk_openxrdevice.cpp:6133
 │   ├ xrAcquireSwapchainImage                              :6169
 │   ├ xrWaitSwapchainImage (timeout 20 ms) ← BLOCKS       :6186-6189   (VRSubmitWait clock, governor pacing)
 │   ├ record blit of each prepared eye texture into the swapchain layer (vkCmdBlitImage, LINEAR) :6262-6316
 │   ├ vkQueueSubmit; RS_WaitFencesPolled ← CPU waits the copy :6348-6371
 │   ├ xrReleaseSwapchainImage                              :6381
 │   └ xrEndFrame (projection + up to 3 quad layers)        :6471
 ├ desktop: AcquireImage (mirror composition recorded)     :748 → textures/vk_framebuffer.cpp:56-82
 └ WaitForCommands(finish=true,false,false)                :749
        → flush, FPSLimit() if !vsync (vk_commandbuffer.cpp:446-447), QueuePresent (:449), RS_WaitFencesPolled (:484)
```

With `vr_openxr_sync_mode != 1` the desktop acquire/present happen inside the first `WaitForCommands(true,..,true)`
and the XR copy follows (vk_renderdevice.cpp:731-751) (V).

Timeline of one VR frame N, as the code orders it (V for order; I for the timing consequences):

```
loop top: I_SetFrameTime  (TicFrac clock sampled here)
  TryRunTics: build command from controller state written in frame N-1 (SetUp #2, predicted for N-1's display)
              → 0..k tics, or TicStabilityWait spin
  BeginFrame: SetUp#1 (xrSyncActions, hands located at N-1's predicted time)
              xrWaitFrame (blocks) → xrBeginFrame → xrLocateViews(N predicted) → updateHmdPose
  RenderView: R_SetupFrame (TicFrac from loop top) → SetUp#2 (xrSyncActions, hands at N predicted) → eyes
  Update:     GPU wait → xrAcquire/WaitSwapchainImage → blit+GPU wait → xrEndFrame → mirror+present+GPU wait
```

When `BeginXRFrame` returns false (session not running, `xrFrameInProgress`, no camera in a level,
vk_openxrdevice.cpp:6012-6044), nothing paces the loop except `FPSLimit` (`vid_maxfps` default 500, v_video.cpp:75) and a desktop vsync if one is enabled,
because TryRunTics forces `doWait=false` whenever `IsVR()` (d_net.cpp:2357-2359) (V; that the loop then runs up to
500 Hz is I).

### 2.5 Threads the engine starts (`main`)

| Thread | Created | Runs | Idles how | Default | Tag |
|---|---|---|---|---|---|
| Main | process | everything in 2.2-2.4 | blocks in the waits of 2.6 | always | V |
| Hang watchdog | `I_HangWatchStart` common/platform/win32/i_hangwatch.cpp:259-265 | `WatchLoop` :228 — reads an atomic beat; after 12 s (loop) / 90 s (startup) without a beat dumps every thread's stack to `hang_stack.txt` | `sleep_for(500ms)` :235 | Windows only; elsewhere inline no-ops common/engine/i_hangwatch.h:76-79 | V |
| `renderPool` ×8 (ctpl) | static init rendering/hwrenderer/scene/hw_bsp.cpp:87 | BSP scene worker (`WorkerThread`, :268) and optional wall workers (:390-406) | condition variable (common/thirdparty/ctpl.h:227); while a job runs and the queue is empty the worker spins on `_mm_pause` ×8 (hw_bsp.cpp:252-266) | used only when `multithread` (:1439): `gl_multithread` (true) and **not** a VR scene unless `vr_scene_multithread` (false, common/rendering/hwrenderer/data/hw_cvars.cpp:278). So in VR all 8 sit idle; flat uses 1 | V |
| Texture loader `VkTexLoadThread` ×N | common/rendering/vulkan/system/vk_renderdevice.cpp:386-425 | decode + upload on upload queues, `WaitForCommands(false,true)` on their own managers (vk_renderdevice.cpp:220) | `wait_for(3 ms)` poll loop when idle (common/utility/TSQueue.h `BackgroundProc`) | `gl_texture_thread` true, N = min(`gl_texture_thread_workers`=2, hw_threads-1, upload queues≤`vk_max_transfer_threads`=2) (hw_cvars.cpp:74-89) | V |
| Model loader ×1 | vk_renderdevice.cpp:388, 427 | model geometry load | same 3 ms poll | with `gl_texture_thread` | V |
| OpenAL stream thread | first `CreateStream`, common/audio/sound/oalsound.cpp:1332-1333 | `Streams[i]->Process()` | condvar; 100 ms timed wait while streams exist (:983-1000) | when music/streams play | V |
| SolarXR reader | common/engine/i_solarxr.cpp:389 via `SolarXR_Tick` :433-437 | SlimeVR UDP reader | — | `vr_slime_enable` false (:47) | V |
| DAP debug server | d_main.cpp:5129-5142 | ZScript debugger | — | when `vm_debug` (archived cvar, default false, d_main.cpp:3662) is set or the debug command-line switch is given (d_main.cpp:5021-5025) [xc: the archived cvar also starts it, not only the switch] | V |
| SW renderer drawer threads | common/rendering/r_thread.cpp:207/222 | software renderer only | — | not used by the hardware renderer | V |
| Crash minidump writer | common/platform/win32/i_crash.cpp:375 | on crash only | — | — | V |
| Anonymous stats HTTP | d_anonstats.cpp:361 | — | — | compiled out (`NO_SEND_STATS`, :17-27) | V |
| Library threads (OpenXR runtime/loader, Vulkan driver, OpenAL Soft mixer, ZMusic) | outside src/ | — | — | — | I (not in this source) |

What runs serially on the main thread and nowhere else: all tics (`G_Ticker`), all ZScript (`RunThinkers`, event
handlers, status bar), prediction rollback, `NetUpdate`, `S_UpdateSounds`, GC, BSP traversal in VR, all command
recording, all fence waits, all OpenXR calls (V by the absence of any other call site for these).

### 2.6 Every place the main thread blocks or spins

| # | Wait | Where | When | Kind | Counted as governor pacing? | Tag |
|---|---|---|---|---|---|---|
| 1 | `xrWaitFrame` | vk_openxrdevice.cpp:6051 | every VR frame | runtime block | yes (:6053) | V |
| 2 | `xrAcquireSwapchainImage` | :6169 | every VR frame | runtime call (may block) [xc: unconfirmed — OpenXR does not define acquire as the wait; `xrWaitSwapchainImage` is; whether it blocks is runtime-specific] | **no** (perf_track `AcquireMs` only) | V |
| 3 | `xrWaitSwapchainImage`, 20 ms timeout | :6186-6189 | every VR frame | runtime block | yes (:6191) | V |
| 4 | Scene fence wait (`WaitForCommands`) | vk_commandbuffer.cpp:484 via vk_renderdevice.cpp:735/739 | every frame | 1 ms-sleep poll, 5 s deadline | no (it is GPU work) | V |
| 5 | XR copy fence | vk_openxrdevice.cpp:6370 | every VR frame | 1 ms-sleep poll | no | V |
| 6 | Desktop mirror fence (sync mode 1) | vk_renderdevice.cpp:749 → vk_commandbuffer.cpp:484 | every VR frame | 1 ms-sleep poll | no | V |
| 7 | Submit-slot recycle | vk_commandbuffer.cpp:345-375 | when >8 submits since the last full wait (submits happen every `vk_submit_size`=1000 `Apply` calls, vulkan/renderer/vk_renderstate.cpp:43, 228-232) | 1 ms-sleep poll | no | V |
| 8 | Mid-frame full waits `WaitForCommands(false)` | vk_renderstate.cpp:516 (stream buffer full), vk_descriptorset.cpp:90/188 (descriptor pool exhausted), vk_hwbuffer.cpp:185 (buffer grow), vk_renderbuffers.cpp:194, hw_entrypoint.cpp:594 (save picture), vk_renderdevice.cpp:1360 (screenshot); also vk_openxrdevice.cpp:338 (`rig_eyeshot`, rig only) and the upload-only waits past 64 MB of queued staging data, `WaitForCommands(false, true)` at textures/vk_hwtexture.cpp:278, 512 and textures/vk_texture.cpp:695 (on whichever thread uploads) [xc: added] | conditional | 1 ms-sleep poll | no | V |
| 9 | `FPSLimit` | common/rendering/v_framebuffer.cpp:244-282 | every frame with `!vsync`, `vid_maxfps>0`, `!cl_capfps` | `sleep_for(t-2ms)`, then `sleep_for(0)` yield loop | yes (:253-257) | V |
| 10 | Desktop `AcquireImage` / `QueuePresent` | vk_framebuffer.cpp:82, 145 | every frame | FIFO/FIFO_RELAXED if `vid_vsync` (default false, v_video.cpp:203); else IMMEDIATE when the surface offers it, otherwise MAILBOX (exclusive fullscreen only) or FIFO, which then blocks like vsync (libraries/ZVulkan/src/vulkanswapchain.cpp:112-133) [xc: IMMEDIATE is not guaranteed; the fallback is FIFO] | yes | V |
| 10b | Virtual-screen / backdrop / menu-beam swapchains: `xrAcquireSwapchainImage` + `xrWaitSwapchainImage` (100 ms timeouts) | vk_openxrdevice.cpp:6848-6859, 7061-7072, 7115-7120 via `RenderVirtualScreen` (vk_renderdevice.cpp:714) | VR frames that show the screen layer (menus, virtual screen) | runtime block | **no** (only :6053 and :6191 call `AddPacingWait` in the XR file) | V [xc: added] |
| 11 | `TicStabilityWait` | d_net.cpp:2256-2271 | every frame with no tic due, `r_ticstability` on | **pure busy spin** on steady_clock for the last stabilised tic's `G_Ticker`+`MakeConsistencies` time (≤1 s, :2283); not cut short when the next tic falls due [xc: precised] | **no** — the governor counts it as frame cost | V |
| 12 | `I_WaitForTic` | common/utility/i_time.cpp:148-174 | flat with `cl_capfps`, paused/menus (`r_NoInterpolate`), background | `sleep_for(ms-2)` then spin on `I_SetFrameTime` | no | V |
| 13 | BSP worker join `future.wait()` | hw_bsp.cpp:1484, 1487, 1533 | flat frames (multithread) | condvar | no | V |
| 14 | `UpdateBackgroundCache(flush)` | vk_renderdevice.cpp:1123-1240 | level load / precache | `SleepEx(1)` loop, 10 s give-up | — | V |
| 15 | Window message pump `SleepEx(0, TRUE)` | common/platform/win32/i_input.cpp:583 | every `I_StartTic` | yield | no | V |
| 16 | Synchronous save/load in `G_Ticker` | g_game.cpp:2037-2056; netplay resync uses `ga_savegame` (d_net.cpp:4400-4415) and re-reads the file (`NetState_Pump` :4418-4432) | on demand | disk I/O | no | V |
| 17 | Startup: netgame lobby, shader compile loop | d_main.cpp:4589-4594, 4694 | startup | block | — | V |

---

## 3. Per-function table

Cost classes: C = constant; A = O(actors), T = O(thinkers), P = O(particles), S = O(subsectors), Ch = O(sound
channels), E = O(events), K = O(tics this frame), G = waits on GPU, R = waits on the XR runtime.

| Function | File:line | Called from | Per | Cost class | Notes | Tag |
|---|---|---|---|---|---|---|
| `D_DoomLoop` | d_main.cpp:1966 | D_DoomMain_Internal:5145 | process | — | catches `CRecoverableError`, `FileSystemException`, `CVMAbortException` → `D_ErrorCleanup` (:2033-2055) | V |
| `I_HangWatchBeat/SetTimeout` | i_hangwatch.cpp:267-282 | loop :1992-2017, p_setup, g_level, vk_renderdevice | ~6 per frame | C (relaxed atomics) | Windows only | V |
| `I_StartFrame` | win32/i_input.cpp:629; sdl/i_input.cpp:699 | loop :2001 | frame after a tic ran | C (joysticks) | | V |
| `I_SetFrameTime` | i_time.cpp:90 | loop :2003; I_WaitForTic; I_FreezeTime | frame | C | sets `CurrentFrameStartTime`; frozen time reported after 5 s (:105-122) | V |
| `TryRunTics` | d_net.cpp:2339 | loop :2006 | frame | K×tic | see 2.3 | V |
| `GC::CheckGC` | common/objects/dobjgc.cpp:155 | d_net.cpp:2345 and d_main.cpp:1566 | **twice per frame** | step ∝ heap when over threshold | `AllocHistory.AddAlloc` therefore sees half-frames | V |
| `I_WaitForTic` | i_time.cpp:148 | d_net.cpp:2369 | frame (when doWait) | sleep + spin | not reached in VR | V |
| `NetUpdate` | d_net.cpp:1430 | TryRunTics:2382 | frame | K×(BuildTiccmd + pack) + packets | runs `GetPackets` twice (:1432, :1894) | V |
| `GetPackets` / `HGetPacket` | d_net.cpp:903 / :704 | NetUpdate | frame ×2 | packets | sockets are non-blocking (common/engine/i_net.cpp:546-548) | V |
| `NetState_Pump` | d_net.cpp:4418 | NetUpdate:1433 | frame | C; resync: whole save file read + chunk sends | host only | V |
| `Host_LateJoinTick` | common/engine/i_net.cpp:2007 | NetUpdate:1434 | frame | C | host only | V |
| `CheckConsistencies` | d_net.cpp:1161 | NetUpdate:1454 | frame with tics, netgame | O(clients×new tics) | triggers resync on mismatch | V |
| `G_BuildTiccmd` | g_game.cpp:1106 | NetUpdate:1537 | command (≈ tic) | C | VR: aim, `WriteHandCommand` (:1527), `VRReplay_OnBuildTiccmd` (:1533) | V |
| `VKOpenXRDeviceMode::WriteHandCommand` | vk_openxrdevice.cpp:8317 | G_BuildTiccmd:1527 | command | C | reads `weaponoffset/offhandoffset/hmdPosition` globals last written by `UpdateControllerState`/`updateHmdPose` in the previous `D_Display`; calls `VRRig_StandEyeTic` (:8358) | V |
| `TicStabilityWait` | d_net.cpp:2256 | TryRunTics:2436 | frame with no tic | spin = last tic | see §5 #3 | V |
| `P_UnPredictClient` | playsim/p_user.cpp:2345 | TryRunTics:2474 | frame with tics | deserialize pawn + players + rollback set | no `netgame` guard | V |
| `G_Ticker` | g_game.cpp:1983 | TryRunTics:2484 | tic | see 2.3 | gameaction loop can load a level inside a tic | V |
| `P_Ticker` | p_tick.cpp:644 | G_Ticker:2137 | tic | A + T | table in 2.3 | V |
| `FThinkerCollection::RunThinkers` | playsim/dthinker.cpp:149 | P_Ticker:881 | tic (world step) | T | `ThinkCycles` clock (:159-263); `profilethinkers` variant (:183-258) | V |
| `P_TickRealTimeActors` | p_tick.cpp:79 | P_Ticker:902; P_RunClientSideLogic:201 | tic in slow-mo | A + inventory | collects into a static TArray then ticks | V |
| `MakeConsistencies` / `WorldStateHash` | d_net.cpp:1286 / :1235 | TryRunTics:2485 | tic, netgame | A × 72 bytes byte-wise FNV | not in single player (`!netgame` return :1288) | V |
| `VRSyncLog_OnTick` | playsim/p_vrsynclog.cpp:179 | P_Ticker:919 | tic | C off; A + fprintf on | `vr_synclog` "" (:38) | V |
| `P_PredictClient` | p_user.cpp:2226 | TryRunTics:2449, :2500 | frame with tics (and no-tic frames with a new command) | serialize pawn + players + rollback set; re-runs `P_PlayerThink`+`CallTick` for each command between `LastPredictedTic` and `ClientTic` (:2269-2299), which is non-empty only when commands run ahead of `gametic` (netgame latency, `cl_debugprediction`) | no `netgame` guard | V |
| `P_RunClientSideLogic` | p_tick.cpp:155 | TryRunTics:2455, :2505 | real tic (×totalTics) | client-side thinkers + C_Ticker/M_Ticker/UiTick/StatusBar->CallTick/automap | `ClientSideThinkCycles` clock (dthinker.cpp:299-476) | V |
| `S_UpdateSounds` | sound/s_doomsound.cpp:988 | TryRunTics:2459, :2509 | frame with tics or totalTics>0 | Ch + sound sequences×tics | → `SoundEngine::UpdateSounds` common/audio/sound/s_sound.cpp:1366: walks every channel, then `RestoreEvictedChannels()` every call because `RestartEvictionsAt` is 0 unless a caller sets it (:1387-1391; setter s_soundinternal.h:425-428, whose only caller is the savegame-load path, sound/s_doomsound.cpp:1106, `LocalWorldTimer + 2` [xc: added the one caller]) | V |
| `NetworkEntityManager::VerifyPredictedEntities` | common/objects/dobject.cpp:783 | TryRunTics:2460, :2510; `DisablePrediction` :819 | frame with tics | O(non-client-side objects spawned while predicting), normally 0 | walks `s_problemEntities`, marks each predicted, clears the list; `DisablePrediction` (called by `P_UnPredictClient`) then destroys every predicted spawn (:819-830) [xc: body read; was "open question"] | V |
| `I_StartTic` | win32/i_input.cpp:615 | loop :2010; NetUpdate:1525 (flat only) | frame (+tic flat) | E | VR: only once per frame, after TryRunTics | V |
| `D_ProcessEvents` | common/engine/d_event.cpp:61 | loop :2011; NetUpdate:1527 | frame + command | E | | V |
| `D_Display` | d_main.cpp:1560 | loop :2013 | frame | everything below | `FrameCycles` written :1926 | V |
| `SolarXR_Tick` | common/engine/i_solarxr.cpp:433 | D_Display:1572 | frame | C off | | V |
| `R_UpdateSky`, `TexAnim.UpdateAnimations` | d_main.cpp:1683-1684 | D_Display | frame | O(animations) | | V |
| `VulkanRenderDevice::BeginFrame` | vk_renderdevice.cpp:1449 | D_Display:1685 | frame | R (xrWaitFrame) + uploads | | V |
| `VKOpenXRDeviceMode::SetUp` | vk_openxrdevice.cpp:3575 | vk_renderdevice.cpp:1458 **and** hw_entrypoint.cpp:327 | **2 per frame** (camera-texture views call the mono base `VRMode::SetUp` instead, see Summary 7 [xc: added]) | xrSyncActions + ~2-4 xrLocateSpace + input mapping + render-side pawn move | guard only stops re-entry (:3587-3612) | V |
| `PollXREvents` | vk_openxrdevice.cpp:3657 | SetUp:3625 | 2 per frame | C | dequeues **one** event per call (:3662-3665) | V |
| `UpdateControllerState` | vk_openxrdevice.cpp:3906 | SetUp:3628 | 2 per frame | C (OpenXR calls) + `P_XYMovement` via `VR_ApplyRenderMove` (:5850, playsim/p_vrdemo.cpp:494) in single player | first call uses the previous frame's `predictedDisplayTime` (:3913, :4158, :4362) | V |
| `updateHmdPose` | vk_openxrdevice.cpp:3708 | SetUp:3653 (×2), BeginXRFrame:6124 | **3 per frame** | C | | V |
| `VRMode::SetUp` (base) | common/rendering/hwrenderer/data/hw_vrmodes.cpp:1481 | `super::SetUp` :3585 | 2 per frame | C | writes `player->PlayInVR`, `mo->AttackPos/Angle/Pitch`, `OffhandPos...` in single player (:1504-1556): render frame writing sim fields | V |
| `BeginXRFrame` | vk_openxrdevice.cpp:6006 | vk_renderdevice.cpp:1459 | frame | R | | V |
| `UpdateBackgroundCache(false)` | vk_renderdevice.cpp:1123 | BeginFrame:1486 | frame | O(finished loads) | | V |
| `DrawHudToSurface` | d_main.cpp:1254 | D_Display:1693 | frame | 2D HUD into a canvas | | V |
| `D_Render` | d_main.cpp:690 | D_Display:1695 | frame | per level: interpolations + S + P | | V |
| `P_FindParticleSubsectors` | playsim/p_effect.cpp:204 | D_Render:701 | frame | S (clear) + visual thinkers + S (fill) + P | per frame, not per tic | V |
| `RenderView` | rendering/hwrenderer/hw_entrypoint.cpp:640 | D_Render lambda | frame | scene | area 02-06 | V |
| `RenderViewpoint` | hw_entrypoint.cpp:264 | RenderView:747, camera textures:726 | per view | scene | calls `R_SetupFrame` (:268) once per view, `vrmode->SetUp` (:327) | V |
| `R_SetupFrame` | rendering/r_utility.cpp:1093 | RenderViewpoint:268 | per view | C | `TicFrac = I_GetTimeFrac()` (:1095); quakes; view interpolation (`R_InterpolateView` :639) | V |
| `End2DAndUpdate` | d_main.cpp:1533 | D_Display:1804/1919 | frame | | | V |
| `VulkanRenderDevice::Update` | vk_renderdevice.cpp:657 | End2DAndUpdate:1536 | frame | G ×3 + R | see 2.4 | V |
| `VkCommandBufferManager::WaitForCommands` | vk_commandbuffer.cpp:430 | Update; mid-frame sites | 2 per VR frame + conditional | G (polled) | `Finish` clock only when `finish` | V |
| `RS_WaitFencesPolled` | vk_commandbuffer.cpp:181 | WaitForCommands, FlushCommands recycle, XR copy, vk_renderdevice.cpp:254 | per wait | 1 ms quantum | `vkGetFenceStatus` + `sleep_for(1ms)` (:219); deadline `REMA_RIG_FENCE_SECONDS` or 5 s (:58-64) | V |
| `AcquireXRSwapchain` | vk_openxrdevice.cpp:6133 | Update:745 | VR frame | R + G | | V |
| `DFrameBuffer::FPSLimit` | v_framebuffer.cpp:244 | WaitForCommands:447 | frame with finish && !vsync | sleep/yield | `fpsLimitTime` advances only when called | V |
| `EffectsGovernor::EndFrame` | common/rendering/hwrenderer/data/hw_effectsgovernor.cpp:79 | End2DAndUpdate:1540 | frame | C | cost = interval − pacing | V |
| `S_UpdateMusic` | loop :2018 | | frame | C | | V |

---

## 4. Data structures that matter for speed

| Structure | Where | Size / shape | Why it matters | Tag |
|---|---|---|---|---|
| `usercmd_t` | d_protocol.h:168 | 4 + 8 shorts + `vrhandpose_t`×2 (12 each) + `vrhandinput_t` (10) + `vrheadpose_t` (12) + `vrtrackers_t` (36) = 102 bytes, padded to 104 | every controller datum the sim reads enters here; sub-struct sizes are fixed by `static_assert`s :155-162; 16-bit flag word + extension word on the wire | V (sizes); I (padding to 104 from 4-byte alignment) |
| `LocalCmds[LOCALCMDTICS]` | d_net.cpp:123 | 175×3 = 525 commands | local command ring; prediction replays from it | V |
| `ClientStates[MAXPLAYERS]` | d_net.cpp:126, d_net.h:89 | 64 clients × `Tics[175]` (`usercmd_t` + `FDynamicBuffer`) + consistency arrays | ~64×175×(104+buffer) bytes; walked by client loops that iterate `NetworkClients` only | V (shape) |
| Thinker lists (`FThinkerList`, intrusive doubly linked) | playsim/dthinker.cpp | one list per statnum; objects scattered on the heap | every per-tic O(actors) pass (interpolation reset, RunThinkers, real-time pass, world hash, synclog, paused relight loop) is a pointer chase; previously measured: interpolation reset 7.75 ms a tic at 39,500 thinkers (renderer:docs/plans/SIM_AT_MOD_SCALE.md:60-67, `doomxr_rend1112c.exe`, rr136); prefetching the next node did not help (same file :76-79, `doomxr_rend1112d.exe`) | V (structure) |
| Prediction rollback buffer | p_user.cpp:2238-2257 (`PredictionData.RollbackData`) | serialized pawn + all player_t + rollback actors/objects | written every tic-frame and read back the next, single player included | V |
| `SoundEngine::Channels` | common/audio/sound/s_sound.cpp | singly linked list of every channel incl. evicted loops | `UpdateSounds` and `RestoreEvictedChannels` walk it every update; previously measured 2,914 evicted channels and ~35 ms a frame in restarts on Abandon MAP13 with the owner's list (renderer:docs/plans/SIM_AT_MOD_SCALE.md:55-58, build 1110k) | V (walks) |
| Command-buffer ring | vk_commandbuffer.h:128-131 | 8 fences/semaphores (`maxConcurrentSubmitCount`) | >8 flushes between full waits forces a recycle wait; `vk_submit_size` 1000 draws per flush | V |
| `RenderJobQueue::pool[300000]`, `WallWorkQueue` | hw_bsp.cpp:140, 168-171 | ~fixed arrays, single-producer | BSP job hand-off to the pool thread; unused in VR by default | V |
| Background TSQueues | vk_renderdevice (primary/secondary/output tex, patch, model in/out) | mutex-guarded TArrays | dequeued on the main thread every `BeginFrame` | V |
| `CurrentFrameStartTime`, `FirstFrameStartTime` | i_time.cpp:39-41 | ns, scaled by `TimeScale` (`i_timescale`) in `GetClockTimeNS` (:63-68) | the only time base for tics and TicFrac; `I_nsTime` (used by every perf clock and the XR/governor timers) is the same scaled clock | V |
| `glcycle_t` clocks | common/rendering/hwrenderer/data/hw_clock.cpp:50-63, common/engine/stats.h `class glcycle_t` | rdtsc accumulators gated by `glcycle_t::active` | active only if `stat rendertimes` is shown, `bench` prints, or `r_perflog>0` (hw_clock.cpp:444) | V |

---

## 5. What slows it down (ranked for VR on the owner's target)

Ranking is by expected impact on the VR frame budget (13.9 ms at 72 Hz); confidence is mine.

**1. Everything is serial, and each frame ends with the CPU waiting for the GPU.** (code V; impact I, high confidence)
Tic(s) → `xrWaitFrame` → scene build → GPU → CPU waits for that GPU work (vk_renderdevice.cpp:733-739,
vk_commandbuffer.cpp:484) → next loop. The only CPU/GPU overlap is inside one frame, from flushing every 1,000 draws
(vk_renderstate.cpp:228-232); there is no inter-frame pipelining, no render thread, no sim thread. So the budget must
hold `tic_cost×tics_this_frame + scene_CPU + (non-overlapped GPU) + 3 polled waits`. Upstream GZDoom has the same
end-of-frame wait (I, from the inherited code shape); `selaco` keeps it with blocking waits.

**2. Catch-up spiral on `main`.** (V for mechanism; I for dynamics; high confidence)
`totalTics` is unbounded wall time; NetUpdate builds at most 35 commands per call and at most 87 ahead
(d_net.cpp:1508-1516); TryRunTics runs `min(totalTics, availableTics)` and adds one while behind (:2420-2426). A tic
costing more than 1/35 s makes every frame owe more tics than it ran, up to ~35 tics per frame. Previously measured:
"175 ms tic x a 35-tic catch-up loop = 6.5 s frames on Abandon MAP13" (renderer:HANDOFF_RENDER_2026-11-11.md:288-289,
stage 0 baseline). `sim_maxcatchup` (renderer:src/d_net.cpp:207-213, 2413-2416) caps it outside netgames; not on `main`.

**3. `TicStabilityWait` busy-spins on every frame without a tic.** (V; impact I, high confidence for VR)
At 72 Hz and 35 tics/s about 37 of 72 frames a second run no tic; each spins `stabilityticduration` = the last
stabilised tic's wall time (d_net.cpp:2273-2284; only `G_Ticker`+`MakeConsistencies`, d_net.cpp:2477-2493 [xc: precised]),
up to 1 s, on the main thread before `xrWaitFrame`. Nothing cuts it short when a tic falls due mid-spin. In VR it
replaces `xrWaitFrame` blocking time while the frame has slack and adds to the frame only past that (I from the call
order) [xc: added]. Effects:
(a) the CPU budget of a no-tic frame becomes the same as a tic frame, so a load that misses on tic frames now misses on
every frame; (b) one core burns at 100 %; (c) the governor sees the spin as frame cost, not pacing
(hw_effectsgovernor.cpp:111-112; only `AddPacingWait` callers are subtracted), so it trims effects for time the engine
spent doing nothing. No clock isolates it (it is inside `FrameMs`/`CostMs` and renderer `simprofile` "whole"; see 6.1)
[xc: was "not timed by any clock"]. Candidate explanation for "About 6 ms of every frame is
outside `cpu_all` and the GPU groups, on both paths" (previously measured, renderer:HANDOFF_RENDER_2026-11-11.md:125,
`doomxr_rend1112l.exe`, Sunder MAP15 flat) — I; settle with `r_ticstability 0` A/B (section 8).

**4. Per-tic O(actors) walks and prediction serialisation.** (V; impact from previous measurement)
Per tic: interpolation reset (p_tick.cpp:729-757), RunThinkers, `P_TickRealTimeActors` in slow-mo, `WorldStateHash`
in a netgame (byte-wise FNV over 72 bytes/actor, d_net.cpp:1235-1256), the paused-tic relight loop (:662-679). Per
tic-frame: `P_UnPredictClient` + `P_PredictClient` serialise/deserialise the pawn and all `player_t` in single player
too (p_user.cpp:2232-2257, 2345-2375). Previously measured: thinkers ~57 ms and interpolation reset 7.75 ms a tic on
Abandon MAP13 with the owner's list (renderer:docs/plans/SIM_AT_MOD_SCALE.md:60-67).

**5. Three polled CPU←GPU waits per VR frame, plus an extra copy pass.** (V; impact I, medium confidence)
`RS_WaitFencesPolled` checks fences then sleeps 1 ms (vk_commandbuffer.cpp:203-219). Right after a submit the fence is
almost never signalled, so each wait costs `GPU_remaining` rounded up to the next ~1 ms wake (timer period from
`timeBeginPeriod`, i_main.cpp:286). Three such waits per VR frame in sync mode 1: scene (vk_renderdevice.cpp:735),
XR copy (vk_openxrdevice.cpp:6370), desktop mirror (vk_renderdevice.cpp:749). The XR copy itself is a separate submit
that blits both eye textures into the OpenXR swapchain (:6262-6316) instead of rendering into it, and it cannot start
until the scene wait returned. Recycle waits (vk_commandbuffer.cpp:345-375) add a mid-frame stall once a frame has
more than 8 flushes (>8,000 `Apply` calls). `selaco` uses blocking `vkWaitForFences` (selaco:src/common/rendering/
vulkan/system/vk_commandbuffer.cpp:170, 253; vk_openxrdevice.cpp:5388).

**6. Double VR SetUp per frame.** (V; impact I, low-medium)
Two `xrSyncActions`, two sets of hand/grip `xrLocateSpace`, three `updateHmdPose`, two `VR_ApplyRenderMove`
(`P_XYMovement` on the pawn, single player), two `VRMode::SetUp` sim writes, `xrVRFrameIndex` += 2 per frame
(vk_openxrdevice.cpp:3581) although `FVRRig::Get` keys its once-per-frame cache on it (r_data/vr_rig.cpp:610-630). The
first pass samples controllers at the previous frame's predicted display time (:3913-3922, :4158). Each of those
render-side moves made while prediction is on (single player: always, between tic batches) is also queued and made
again after the rollback at the start of the next tic (`PredictRedo`, playsim/p_vrdemo.cpp:509-510, 623-635, called from
`VRDemo_SetInTic(true)`, g_game.cpp:1995), so the queue holds two entries a frame and the tic pays `P_XYMovement` for
each again. Camera-texture views add a mono `VRMode::SetUp` each, which in single player overwrites `PlayInVR` and the
attack pose with flat values until SetUp #2 restores them (hw_vrmodes.cpp:1505-1556). [xc: added]

**7. Sound update walks.** (V; measured on renderer before its fix)
`SoundEngine::UpdateSounds` walks every channel and calls `RestoreEvictedChannels` on every call (s_sound.cpp:1370-1391).
Previously measured ~35 ms a frame of restart tries with 2,914 evicted loops (SIM_AT_MOD_SCALE.md:55-58); `renderer`
throttles it with `snd_evictretry`/`snd_evictbudget` (renderer:src/common/audio/sound/s_sound.cpp:1335-1414); `main`
does not.

**8. Interpolation clock not tied to display time.** (V for the clocks; impact I, medium for perceived smoothness)
`TicFrac` = `I_GetTimeFrac()` of the time captured at loop top (i_time.cpp:90, 208-215; r_utility.cpp:1095;
hw_entrypoint.cpp:672). The world is interpolated for "loop top", the head for `predictedDisplayTime`. The gap
(tic + optional spin + `xrWaitFrame` + scene + runtime latency) varies frame to frame with tic cost and spin, so actor
motion gets jitter that the head does not. Command poses are a frame older still (2.4 timeline).

**9. Parallel hardware left idle.** (V)
8 pool threads idle in VR (`vr_scene_multithread` false, hw_cvars.cpp:278; hw_bsp.cpp:1438-1439); wall workers off
(`gl_bsp_worker_threads` 1, hw_bsp.cpp:47); loader threads wake every 3 ms when idle (TSQueue.h `wait_for(3ms)`). On
the flat path the scene worker spins on `_mm_pause` while it waits for jobs (hw_bsp.cpp:252-266, 316-322) and the main
thread blocks on `future.wait()` (:1533).

**10. GC twice a frame, on the main thread.** (V; impact unmeasured) `GC::CheckGC` at d_net.cpp:2345 and
d_main.cpp:1566; a step runs whenever the threshold is exceeded (common/objects/dobjgc.cpp:155-163).

**11. FPSLimit after xrEndFrame.** (V; impact I, low) In VR sync mode 1 the final `WaitForCommands(true)` calls
`FPSLimit` (vk_commandbuffer.cpp:446-447). With the default `vid_maxfps 500` it is a no-op at 72 Hz; a cap set near the
headset rate would sleep after `xrEndFrame` and beat against the runtime's pacing.

---

## 6. Hooks, cvars and dead code in this area

| Item | Kind | Default | Read where | Status | Evidence | Tag |
|---|---|---|---|---|---|---|
| `r_ticstability` | cvar | true | d_net.cpp:2262 | LIVE (busy spin) | d_net.cpp:2251 | V |
| `cl_capfps` | cvar | false | d_net.cpp:2354; r_utility.cpp:653, 1096; hw_entrypoint.cpp:672; v_framebuffer.cpp:249 | LIVE flat; ignored for waiting in VR | v_framebuffer.cpp:52 | V |
| `vid_maxfps` | cvar | 500 (35..1000) | v_framebuffer.cpp:249-259; governor budget | LIVE | v_video.cpp:75-86 | V |
| `vid_vsync` | cvar | false | swapchain present mode | LIVE | v_video.cpp:203 | V |
| `vid_dontdowait`, `vid_lowerinbackground` | cvars | false / true | d_net.cpp:2361-2365 | LIVE flat | d_net.cpp:175-176 | V |
| `singletics` | global flag | false | d_net.cpp:2375, 2421 | **DEAD**: never assigned anywhere | only `d_main.cpp:587` defines it; grep finds no writer | V |
| `sim_maxcatchup` | cvar | — | — | absent on `main`; renderer default 2 | renderer:src/d_net.cpp:207 | V |
| `cl_debugprediction` | cvar CHEAT | 0 | d_net.cpp:2401 | DORMANT | d_net.cpp:216 | V |
| `vr_openxr_sync_mode` | cvar | 1 | vk_renderdevice.cpp:731 | LIVE | hw_vrmodes.cpp:641 | V |
| `vr_scene_multithread` | cvar | false | hw_bsp.cpp:1438; menu option wadsrc/static/menudef.txt:5053, 5174 | DORMANT by default, menu-exposed (VR BSP single-threaded) [xc: menu exposure added] | hw_cvars.cpp:278 | V |
| `gl_multithread` | cvar | true | hw_bsp.cpp:1439 | LIVE flat only | hw_bsp.cpp:46 | V |
| `gl_bsp_worker_threads` | cvar | 1 (1..8, restart) | hw_bsp.cpp:395, 1440, 1452; menu slider menudef.txt:2243, 5274 | DORMANT by default, menu-exposed (wall workers need >1) [xc: menu exposure and :395 added] | hw_bsp.cpp:47 | V |
| `vk_submit_size` | cvar | 1000 | vk_renderstate.cpp:228 | LIVE | :43 | V |
| `REMA_RIG_FENCE_SECONDS` | env var | unset → 5 s | vk_commandbuffer.cpp:58-64 | rig only | | V |
| `gl_texture_thread`, `_workers`, `_upload`, `vk_max_transfer_threads` | cvars | true, 2, true, 2 | vk_renderdevice.cpp:340, 386-430 | LIVE | hw_cvars.cpp:74-89 | V |
| `perf_track` | cvar | 0 | `PerfTrack::Active()` hw_perftrack.h:53 [xc: line fixed]; menu option menudef.txt:2058 | DORMANT by default, menu-exposed; `-benchdemo` sets it (g_benchdemo.cpp:86) | hw_perftrack.cpp:61 | V |
| `r_perflog` | cvar | 0 | d_main.cpp:1543; hw_clock.cpp:444 | DORMANT | hw_perflog.cpp:122 | V |
| `r_effects_governor` | cvar | true | hw_effectsgovernor.cpp:121 | LIVE | :35 | V |
| `vr_synclog` | cvar string | "" | p_vrsynclog.cpp:181 | DORMANT | :38 | V |
| `vr_slime_enable` | cvar | false | i_solarxr.cpp:435 | DORMANT | :47 | V |
| `vr_posestats` (`[A0]` report) | cvar | off | vk_openxrdevice.cpp:4400 | DORMANT | | V |
| `[PAWNANCHORLOG]` once-a-second log line | diagnostic | always | vk_openxrdevice.cpp:5864-5876 | LIVE in single-player VR: one `Printf(PRINT_LOG)` a second, no switch | | V |
| `worldclock_log` | cvar | false | scripting/vmthunks.cpp:2822 (defined p_tick.cpp:77) [xc: p_tick.cpp:77 is the definition, the read is in vmthunks] | DORMANT | | V |
| `VKOpenXRDeviceMode::SubmitFrame` / `VRMode::SubmitFrame` | virtual | — | none | **DEAD**: no caller | defined vk_openxrdevice.cpp:5997, hw_vrmodes.h:225; grep for `SubmitFrame()` finds only these | V |
| `VKOpenXRDeviceMode::Present` | override | — | only `gl_postprocess.cpp:170` (`vrmode->Present()`, GL backend) | **DEAD on Vulkan** | vk_openxrdevice.cpp:6482-6486 | V |
| `FrameCycles` | cycle_t written every D_Display | — | rendering/swrenderer/scene/r_scene.cpp:371, 380 only | DORMANT under the hardware renderer (written, never read) | d_main.cpp:609, 1926 | V |
| `E_RenderFrame` call | hook | — | commented out | DEAD | d_main.cpp:1689-1690 | V |
| `D_DoAnonStats` / anonstats thread | function | — | d_main.cpp:5126 | **DEAD** (empty under `NO_SEND_STATS`) | d_anonstats.cpp:17-27 | V |
| `src/gl/stereo3d/gl_openxrdevice.cpp` | source file | — | — | **DEAD**: not in src/CMakeLists.txt (only `rendering/gl/stereo3d/gl_openvr.cpp`, :1069) | grep of CMakeLists | V |
| `vid_preferbackend` default | cvar | **BACKEND_OPENGL** | v_video.cpp:92 | the GL backend and `FPSLimit` paths in gl_framebuffer.cpp are compiled; Vulkan must be selected by config | v_video.cpp:92-122 | V |
| `USE_OPENXR` / `USE_OPENVR` | defines set by the CMake options `ENABLE_OPENXR` / `ENABLE_OPENVR`, both `NO` by default (src/CMakeLists.txt:464-466, 394-396) [xc: the options are ENABLE_*, in src/CMakeLists.txt, not CMakeLists.txt] | — | the Windows setup script turns both ON (auto-setup-windows-vr.cmd:98-100); the Linux rig ON for OpenXR (tools/linux_rig/setup.sh:28) | | V (files) / I (that the owner's exe is built that way) |
| `I_HangWatch*` | hooks | — | loop, load, precache | LIVE on Windows; no-ops elsewhere | i_hangwatch.h:57-79 | V |
| `I_FPSTime` | — | — | — | does not exist; the frame clock is `I_msTimeFS` (i_time.cpp:193) | grep | V |

### 6.1 Clocks and counters around the loop — what each covers and does not

| Clock | Defined / taken | Covers | Does NOT cover | Tag |
|---|---|---|---|---|
| `perf_track` row `FrameMs` | hw_perftrack.cpp:506 (`I_nsTime` between `EndFrame` calls) | full loop period | — (but `I_nsTime` is scaled by `i_timescale`, i_time.cpp:63-68) | V |
| `perf_track` `CostMs/PacingMs/BudgetMs` | from `EffectsGovernor::LastFrame` (hw_perftrack.cpp:514-519) | pacing = xrWaitFrame + xrWaitSwapchainImage + FPSLimit + desktop acquire/present | xrAcquireSwapchainImage, fence polling, **TicStabilityWait**, I_WaitForTic (all counted as cost) | V |
| `perf_track` `Tics`, `ThinkMs`, `CsThinkMs`, `Thinkers` | d_net.cpp:2489; dthinker.cpp:263-268, 472-476 via `PerfLog::AddThinkSample` (hw_perflog.cpp:704-707) | number of tics; `RunThinkers` and `RunClientSideThinkers` only | the rest of P_Ticker (interpolation reset, WorldTick handlers, particles…), G_Ticker, prediction, NetUpdate, sounds, GC | V |
| `perf_track` `Xr*` | vk_openxrdevice.cpp:6058-6066, 6167-6171, 6193-6194, 6468-6476 | xrWaitFrame, acquire, image wait, xrEndFrame; display-ahead via XR_KHR_win32_convert_performance_counter_time (Windows only, :505-541); XR_META metrics if offered | the XR copy's fence wait (that is `VRSubmitWait`) | V |
| `perf_track` CPU columns (`scene, post, finalize, submit, composite, syncwait, all, drawcalls`) | `GetRenderTimeSummary` hw_clock.cpp:153-163 | glcycle_t clocks | **read 0 unless `glcycle_t::active`**, which `perf_track` does not set (hw_clock.cpp:444 tests only `stat rendertimes`, `printstats`, `r_perflog`) | V |
| `PerfTrack::Scope` reasons | GC d_net.cpp:2344; pipeline vk_renderpass.cpp:359; texture vk_hwtexture.cpp:227 | those regions | anything else | V |
| `perf_track` `FrameMs`/`CostMs`, renderer `simprofile` "whole", renderer Tracy `TryRunTics` zone | as above | **contain** `TicStabilityWait` | do not separate it from the rest of the frame / of TryRunTics | V [xc: added] |
| `All` (`stat rendertimes` "All") | `ResetProfilingData` hw_clock.cpp:83-84 → `All.Unclock` hw_entrypoint.cpp:752; plus `Finish` | RenderView, plus the finishing `WaitForCommands` | TryRunTics, BeginFrame (incl. xrWaitFrame), HUD/2D, eye composite, XR submit | V |
| `Finish` | vk_commandbuffer.cpp:434-435, 509 | finishing WaitForCommands: acquire + flush + FPSLimit + present + fence wait | the VR scene wait (finish=false in sync mode 1) | V |
| `VRRenderSyncWait` | vk_renderdevice.cpp:734 | VR scene fence wait | — | V |
| `VRSubmitWait` | vk_openxrdevice.cpp:6187, 6369 | xrWaitSwapchainImage + XR copy fence wait | — | V |
| `VRSubmit`, `VRSubmitCopy` | :6260, 6282 | XR copy record/submit/wait | — | V |
| `VREyeComposite` | vk_renderdevice.cpp:679-711 | per-eye 2D + finalize | — | V |
| `ThinkCycles`, `ClientSideThinkCycles` (`stat think`) | dthinker.cpp:44, 159-263, 299-472 | RunThinkers / RunClientSideThinkers | — | V |
| `FrameCycles` | d_main.cpp:1586-1926 (plain cycle_t) | D_Display after its early returns | read only by the software renderer | V |
| r_perflog frame interval | hw_perflog.cpp:786 | full loop period | — | V |
| Governor | hw_effectsgovernor.cpp:79-123 | interval, pacing | see perf_track Cost | V |
| `simprofile` (renderer only) | renderer:src/d_net.cpp:191-205, 2375-2573 | whole TryRunTics, GC, NetUpdate, G_Ticker+MakeConsistencies, P_Un/PredictClient, client-side logic, sounds, verify | the clocks run across frames until the command; the CCMD runs inside a tic, so "whole"/"ticker" are open while it prints (renderer:HANDOFF_RENDER_2026-11-11.md:303 reports they read negative) | V |
| `ticprofile` (renderer only) | renderer:src/p_tick.cpp:644-660, 743-930 | per level: interpolation reset loop, particles…effect tic, player think, WorldTick, level tick, RunThinkers, specials, WorldStep | P_Ticker head (UpdateInterpolations, NewTick, sight counters), `P_TickRealTimeActors`, paused path, `VRSyncLog_OnTick`, all of G_Ticker outside P_Ticker; `TicProfileCount` counts level iterations, not tics | V |
| Tracy zones (renderer only) | renderer:src/common/utility/rema_profile.h; zones at d_main.cpp:2012/2022/2025, hw_entrypoint.cpp:267/363, vk_renderdevice.cpp:667/742, hw_bsp.cpp:1447, hw_drawinfo.cpp, hw_drawlist.cpp | TryRunTics, D_Display, RenderViewpoint, Eye, VkUpdate, GpuWait, scene stages | compiled out unless CMake `REMA_TRACY` ON (renderer:CMakeLists.txt:430, default OFF); no zone around xrWaitFrame/BeginFrame or TicStabilityWait | V |
| `stat fps` / `vid_fps` | d_main.cpp:1105-1150 | ms between `screen->FrameTime` (ms resolution) | — | V |

---

## 7. Branch differences (this area)

| Item | main | renderer (175063a) | netplay (1c1805f8) | selaco (7e111a3d) |
|---|---|---|---|---|
| `D_DoomLoop` order | as 2.2 | same + Tracy zones and frame mark (renderer:src/d_main.cpp:2011-2025) | identical to main (no diff in d_main.cpp) | same order, no hang-watch beats (selaco:src/d_main.cpp:1917-1950) |
| D_Display level gate | `gametic != 0` | also `gameaction != ga_fullconsole` (renderer d_main.cpp:1692) | as main | — |
| Catch-up cap | none | `sim_maxcatchup` 2 (SP only) | none | none |
| `simprofile`, `ticprofile` | absent | present | absent | absent |
| `R_SnapshotWrite` after `P_PredictClient` | absent | two call sites (renderer d_net.cpp:2493, 2557); `r_snapshot` 0 = one int test; `r_snapshot 3` spawns and joins up to 16 `std::thread`s per write (renderer:src/rendering/r_snapshot.cpp:181-186, 373); `r_renderthread` refuses to turn on (:33-40) | absent | absent |
| `TicStabilityWait` busy spin | yes | yes | yes | yes (selaco d_net.cpp:2121-2136) |
| Prediction in single player | yes | yes | yes | yes |
| Fence waits | polled 1 ms | polled 1 ms (+`GpuWait` zone) | polled 1 ms (file identical) | blocking `vkWaitForFences(UINT64_MAX)` |
| `FPSLimit` reports pacing to governor | yes | yes | yes | no governor (file absent) |
| Hang watchdog / frozen-clock report | yes | yes | yes | absent (i_hangwatch.cpp absent; i_time.cpp diff removes the report) |
| `perf_track` | yes | yes + bsp/clip/walls/flats/sprites columns; flat fallback for scene/syncwait in `GetRenderTimeSummary` (renderer hw_clock.cpp) | yes | absent (hw_perftrack.cpp absent), smaller r_perflog |
| Sound eviction restarts | every update | throttled (`snd_evictretry` 8, `snd_evictbudget` 64) + per-source channel index | every update | every update |
| `sim_prefetch` in thinker walks | absent | cvar, default 0 | absent | absent |
| `usercmd_t.head` | 6 shorts (z, pitch, roll, roomdx, roomdy, turn) | identical to main (`git diff origin/main origin/renderer -- src/d_protocol.h src/d_protocol.cpp` is empty) | 3 shorts (no room travel, no turn) (netplay d_protocol.h diff) | own protocol (179-line diff) |
| Netgame roomscale | in the command (`roomdx/roomdy`), applied by every machine in `P_PlayerThink` | same as main | local teleport queue when drift ≥ 0.35 m (netplay g_game.cpp diff, `VR_QueueMultiplayerRoomscaleTeleportTarget`) | — |
| Controller net copies | `xrNet*` separate from edge trackers (vk_openxrdevice.cpp:3966-3972) | same | writes `xrLast*` edge trackers in the input loop (the trigger-edge bug main fixed) | — |
| `DEM_NETREMOVEPAWN`, `netplay_ready`, `netplay_load` | present | present | absent | absent |
| `VRRig_StandEyeTic` in WriteHandCommand | yes | removed (renderer is based on an older main) | absent | — |
| Double VR SetUp per frame | yes | yes | yes | yes (selaco vk_renderdevice.cpp:1289, hw_entrypoint.cpp:231) |
| `renderPool(8)`, `vr_scene_multithread` false | yes | yes | yes | yes (+ particle-pool job type, selaco hw_bsp.cpp) |

All rows V by diff (`git -C /home/claude/uzdxrema diff origin/main origin/<branch> -- <file>`) or direct reads.
Files in this area with **no** diff between main and netplay: d_main.cpp, p_tick.cpp, i_time.cpp, vk_commandbuffer.cpp,
vk_renderdevice.cpp, v_framebuffer.cpp, hw_bsp.cpp, hw_entrypoint.cpp, i_main.cpp, i_hangwatch.cpp, dthinker.cpp (V).

---

## 8. Open questions and the measurements that would settle them

| # | Question | Measurement |
|---|---|---|
| 1 | How much frame time does `TicStabilityWait` cost, and is it the "~6 ms outside `cpu_all`" (renderer:HANDOFF_RENDER_2026-11-11.md:125)? | Same spot, same seed, `perf_track 1`, A/B `r_ticstability 1/0`; compare `FrameMs` and `CostMs` on rows with `Tics=0`. In VR also compare `XrWait` (should rise by the spin when off). |
| 2 | What does the 1 ms polling quantum cost per VR frame? | Read `VRRenderSyncWait` and `VRSubmitWait` (`r_perflog 1` or `stat rendertimes`, not perf_track alone) against the GPU group sum for the same frames; A/B with a build using blocking `vkWaitForFences` (as selaco does). |
| 3 | How stale is a VR command's hand pose relative to the tic it drives, and how much does the first SetUp pass matter? | Log, per frame, `I_nsTime` at `I_SetFrameTime`, at xrWaitFrame return, the predictedDisplayTime used by each `UpdateControllerState` call (converted via the existing shared clock, vk_openxrdevice.cpp:651-665) and the `gametic` built. |
| 4 | Does the loop-top TicFrac clock cause visible world judder in VR? | Per frame: `predictedDisplayTime − loop-top time` (shared clock). Its standard deviation × actor speed = jitter amplitude. |
| 5 | Are room-scale deltas applied once or split/duplicated across the 2 `UpdateControllerState` + 3 `updateHmdPose` calls per frame? (area 07 owns the internals) | Existing `[PAWNANCHORLOG]` line: compare "asked to move" against real head travel over a measured walk; or log `positional_movement*` per call with the call index. |
| 6 | Does `FVRRig::Get` solve twice per frame because `xrVRFrameIndex` advances twice? | Count `FVRRig` recomputes per displayed frame (one counter, printed by an existing stat). |
| 7 | Cost of single-player `P_UnPredictClient`/`P_PredictClient` serialisation with BD22-class pawns/inventories. | `simprofile` "predict" column on renderer (already exists), or a `PerfTrack::Scope` around the two calls on main. |
| 8 | How often do mid-frame full waits (#7-8 of 2.6) fire in VR on big maps? | Count calls of `WaitForCommands(false)` and recycle waits per frame (the RS submit journal already records each submit, vk_commandbuffer.cpp:122-136; dump it with a counter rather than on stall). |
| 9 | GC step cost and frequency. | `perf_track` `REASON_GC` time per frame (covers only the TryRunTics call; add the D_Display call) |
| 10 | Is the owner's exe built with `USE_OPENXR`/`USE_OPENVR` and `vid_preferbackend 1` in his ini? | Startup log lines `Selecting Vulkan backend...` (v_video.cpp:115) and `perf_track: OpenXR shared clock ...` (vk_openxrdevice.cpp:554). |
| 11 | With the headset session not running (`BeginXRFrame` false) does the loop free-run at `vid_maxfps`? | `stat fps` / perf_track `FrameMs` with the headset idle. |
| 12 | Netgame per-tic cost of `WorldStateHash` at 20-40k actors. | Wrap `MakeConsistencies` in a cycle_t in a 4-player rig session; compare with the same spot single player. |

---

## Cross-check (date 2026-11-13)

Checked against the code in `main` f182cbcc (plus `renderer` src = 175063a, `netplay` 1c1805f8, `selaco` 7e111a3d).
Greps for dead/dormant claims covered `src/` and `wadsrc/` (incl. menudef.txt, ZScript, CVARINFO-style lumps).

| Category | Checked | Right | Line-fixed | Corrected | Unconfirmed | Added |
|---|---|---|---|---|---|---|
| a. Summary items | 10 | 7 | 0 | 2 (#4, #10) | 0 | 1 (#7) |
| b. "What slows it down" items (ranking kept) | 11 | 9 | 0 | 1 (#3) | 0 | 2 (#3 VR note, #6) |
| c. DEAD / DORMANT / never-read claims | 16 | 16 status right | 1 (perf_track) | 1 (worldclock_log read site) | 0 | 3 menu-exposure notes |
| d. Cvar defaults | 26 | 26 | 0 | 0 | 0 | 0 |
| e. Branch-difference claims | 20 | 20 | 0 | 0 | 0 | 0 |
| f. Table rows / tree cites (≈1 in 2 of §3, all of 2.1, 2.3, 2.4, 2.6, 7 of 17 P_Ticker rows, 9 of 11 thread rows) | ~120 | ~112 | 1 (loop :1993-1994) | 3 (2.6 #10, DAP thread, VerifyPredictedEntities) | 1 (2.6 #2) | 6 |

Key claims confirmed as written: the 1 ms-sleep fence poll (`RS_WaitFencesPolled`, vk_commandbuffer.cpp:181-221; checks
first, sleeps 1 ms, 5 s deadline) at all three VR waits; two `VKOpenXRDeviceMode::SetUp` per VR frame with the first at
the previous frame's `predictedDisplayTime`; single-player prediction serialise/deserialise with no `netgame` guard
(p_user.cpp:2226-2257, 2345-2375); `TicStabilityWait` is a pure busy spin with a 1 s cap.

Corrections and additions, one line each:
1. Summary 4, §5 #3, 2.6 #11: the spin length is `G_Ticker`+`MakeConsistencies` of the last stabilised tic only; it runs only when `totalTics<=0`, is skipped on `pauseext`, is never cut short when a tic falls due; the 1 s cap is its only bound.
2. Summary 10, §5 #3, 6.1: "not timed by any clock" was wrong; the spin is inside perf_track `FrameMs`/`CostMs`, renderer `simprofile` "whole" and renderer Tracy `TryRunTics`, just never isolated.
3. Summary 7, §5 #6, §3 SetUp row: added the mono `VRMode::SetUp` per camera-texture view (writes `PlayInVR=false` and flat attack pose in SP), `AttackDir`/`OffhandDir` written in MP too, and `PredictRedo` re-running every render-side move in the next tic.
4. 2.6 #10: without vsync the present mode is IMMEDIATE only if the surface offers it; otherwise FIFO (or MAILBOX in exclusive fullscreen).
5. 2.6 #2: `xrAcquireSwapchainImage` as a "runtime block" marked unconfirmed.
6. 2.6 #8: added the rig eyeshot wait and the 64 MB upload-only waits; 2.6 #10b: added virtual-screen/backdrop/beam swapchain waits (100 ms timeouts, not counted as governor pacing).
7. 2.5: the DAP server also starts from the archived `vm_debug` cvar, not only the command-line switch.
8. §3: `VerifyPredictedEntities` body read (dobject.cpp:783), no longer an open question.
9. §3 `S_UpdateSounds`: the only `SetRestartTime` caller is the savegame-load path (s_doomsound.cpp:1106).
10. §6: `worldclock_log` is read in vmthunks.cpp:2822 (p_tick.cpp:77 is its definition); `PerfTrack::Active` at hw_perftrack.h:53; `vr_scene_multithread`, `gl_bsp_worker_threads`, `perf_track` are menu-exposed; the build options are `ENABLE_OPENXR`/`ENABLE_OPENVR` in src/CMakeLists.txt.
11. 2.2: loop hang-watch lines :1993-1994; noted the second `I_HangWatchStart` (:1983), a no-op by guard.
12. 2.3: added the `pauseext` early return (:2387-2388).

Not checked: §7 "controller net copies" row, selaco protocol diff size, renderer perf_track column additions, §4 sizes
beyond `usercmd_t` and the command ring, every quoted "previously measured" number (sources are docs, not code).
