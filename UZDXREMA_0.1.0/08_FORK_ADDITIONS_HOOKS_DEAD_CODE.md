# 08 — Inventory of the fork's additions, hooks and dead code

Primary tree: `main` (`/home/claude/uzdx_main`, f182cbcc). Other trees: `renderer` (`/home/claude/uzdxrema`, 175063a1), `netplay` (`/home/claude/uzdx_netplay`, 1c1805f8), `selaco` (`/home/claude/uzdx_selaco`, 7e111a3d).
All paths below are relative to `src/` of `main` unless a tree or `wadsrc/` is named. "V" = VERIFIED (code read), "I" = INFERRED (reason given).

## 1. Summary

1. Baseline: stock UZDoom tag `5.0.0` (commit 292cf42, shallow-cloned from github.com/UZDoom/UZDoom into /tmp for comparison; not part of any tree). The repo's root commit `68ec2561` ("UZDXR 5.0.0-dxr, pristine", 2026-09-06) already carries a first layer of non-stock code; there is no history before it (V: `git log` root). Layer **P** = present at that import (QuestZDoom/DoomXR VR port + earlier fork work); layer **R** = added on `main` after it.
2. Size (V, file-list diff + `diff -w` line counts, src + wadsrc/static): stock 1,962 files, `main` 2,234. **272 new files, 0 stock files removed; 414 stock files modified (+79,617 / -2,207 lines)**. P alone: 83 new files (~33.4k code lines) and 323 modified stock files (+32,656). R: ~189 new files (~83.6k lines) and ~+47k lines in stock files.
3. CVARs (V, macro parse of all `CVAR*` definitions): stock 958 names, `main` 1,580: **623 fork cvars** (302 P, 321 R), 1 stock cvar removed (`vr_hunits_per_meter`). Of the 623: **6 DEAD**, 6 DORMANT (read only by GL/OpenVR code), 1 callback-only, 7 read only at startup/creation, the rest read at run time (§8.4, full table §8.9).
4. ZScript natives (V, `DEFINE_ACTION_FUNCTION*`/`DEFINE_FIELD*`/`DEFINE_GLOBAL*`): stock 2,075, `main` 2,695: **+459 native functions, +161 fields/globals**. Only ~30 of the 459 have a caller inside the engine pk3; the rest are API for external mods (RS/BD22), whose sources are not in any tree, so their use cannot be verified here (§8.6).
5. Dead code (V, whole-tree identifier index over 2,005 fork-new function definitions): **60 fork functions are defined and never called** (incl. 4 virtual hook methods declared, overridden and never invoked), plus one dead file (`r_data/level_solid_query.cpp`, compiled, never called on `main`), one dead input path (teleport burst), one dead flag bit (`COMPATF2_OLD_RANDOM_GENERATOR`), one ZScript virtual the engine never dispatches (`Actor.OnSweepCrossed`), and one stock cvar the fork made dead (`gl_weapon_purelightlevel`) (§8.1-8.5).
6. Fork code contains almost no `#if 0`: 4 blocks (2 in `rendering/gl/stereo3d/gl_openvr.cpp`, 2 in `hw_vrmodes.cpp`). The P layer carries ~45 QuestZDoom `__MOBILE__`/`__ANDROID__` blocks and 10 never-compiled files (`posix/nosdl/*`, `gl/stereo3d/gl_openxrdevice.cpp`, a stale `xlat_parser.c`) (§8.8).
7. Every fork shader file is referenced from C++ and every fork effect index is bound somewhere; no fork shader is registered-but-never-bound (V, §8.7). But `main.fp` grew from 944 to 5,128 lines and every Vulkan `main.fp` program is compiled with `SURFACE_DAMAGE` + `EFFECT_LIGHTS` and the stamp + outline libraries prepended (`common/rendering/vulkan/shaders/vk_shader.cpp:1171-1192`).
8. Almost every R-layer effect (smoke, debris, level field, surface damage, emissive volumes, exposure impulse, effect lights, heat/shock, drawn lines, GPU/mesh particles) is **demand-driven**: its master cvar defaults on, but it allocates and dispatches nothing until a mod queues an event. Their always-on cost is O(slots) bookkeeping per frame/tic (§3, §5).
9. Always-on costs by default that are NOT demand-gated (V): `vk_gpu_checkpoints 1` (an NV checkpoint per GPU group) and two `std::string` + `std::set` interns on every `PushGroup` even with stat gpu off; soft GPU particles (`r_gpuparticles_soft 8`) break and restart the scene render pass twice per eye once any GPU particle exists; two 8192-unit traces per player per tic for the laser (`P_VRLaserTraceTic`); two engine-registered ZScript event handlers ticking every tic in every game (§7).
10. `selaco` drops the whole avatar/holster/grab/affect/guard/effect-light/emissive/exposure/perf_track layer (113 files) and adds pooled script particles and `model_reach` (which makes `LevelSolidQuery` alive there). `renderer` adds the static level path (12 files, all default-off). `netplay` lacks the holster/torso/grab/grasp/UPnP files (§9).

## 2. Method and limits

- Stock reference: `/tmp/stock_uzdoom/src500` = UZDoom tag `5.0.0`. The fork's `version.h` keeps ENG 5.0.0 (commit 3511a341 message); the exact upstream commit under DoomXR is unknown, so a few "fork" items may be upstream post-5.0.0 code (I: no upstream history in the repo).
- New files: `comm` of `git ls-tree` lists of `src/` and `wadsrc/static/` against the stock tree. Modified-file sizes: `diff -w` counts of `>` / `<` lines.
- CVARs: regex over the `CVAR/CUSTOM_CVAR/CVARD/CUSTOM_CVARD/CVARD_NAMED/DEPR_CVAR` macros (`common/console/c_cvars.h:767-785`), named variants resolved to the console name. A "read" = a non-comment occurrence of the name or variable outside its definition line and outside `EXTERN_CVAR`, in `src` or in ZScript (ZScript reads cvars as globals or via `CVar.FindCVar`), or a `"name"` string lookup. menudef/language references are counted separately and do not count as reads.
- Dead functions: identifier index of every word in `src` (comment tails stripped); a function whose short name occurs nowhere but its definition and pure prototypes is DEAD. Name collisions can only hide dead code, never invent it. Header-only hits were checked by hand (§8.1).
- Natives: callers = non-`native` lines in `wadsrc/static/zscript` matching `Name(`. Name collisions were hand-checked for the bolded entries.
- Cost classes are complexity, not milliseconds. Previously measured numbers are quoted only with their source.

## 3. Entry points and call graph (hooks the fork inserted into stock code)

### 3.1 Frame (render thread = main thread)

`D_Display` → `RenderViewpoint` (`rendering/hwrenderer/hw_entrypoint.cpp`), main view to screen only unless noted:

| Order | Call (V) | Site | Gate / early-out (V) |
|---|---|---|---|
| 1 | `EmissiveVolumes::Get().BeginFrame` | hw_entrypoint.cpp:274 | `r_emissivevolumes` (default 3) and Vulkan; returns after draining the queue when none (`hw_emissivevolumes.cpp:246`) |
| 2 | `BeamLights_HandOver` | hw_entrypoint.cpp:286 → `hw_drawinfo.cpp:326` | loops `MAX_BEAMS` = 128 (`g_levellocals.h:3110`), skips slots with `BeamLightRadius<=0` |
| 3 | `EffectLights::Get().BeginFrame` | hw_entrypoint.cpp:293 → `hw_effectlights.cpp:153` | `r_effectlights` (1) + Vulkan; drains `EffectLightSpawns` |
| 4 | `DynamicLightShadowRowsWanted` | hw_entrypoint.cpp:299 → `hw_dynlightdata.cpp:210` | returns at once unless `gl_light_castshadows`; else walks `Level->lights` |
| 5 | `CollectLights(Level, eye)` + `EffectLights::AssignShadowRows` | hw_entrypoint.cpp:78-153 | only when the shadow-map pass runs |
| 6 | `VRMode::GetVRModeCached` + `vrmode->SetUp()` | hw_entrypoint.cpp:326-327 | always (stock called `GetVRMode`) |
| 7 | `R_UpdatePoseAnchoredLights` | hw_entrypoint.cpp:185, call :330 | loops `Level->PoseAnchoredLights` |
| 8 | `PrepareFrameCompute` | hw_entrypoint.cpp:217 | calls `SectorPlanes::BeginFrame`, `EffectLights::PrepareFrame`, `SmokeVolume::PrepareFrame`, `LevelField::PrepareFrame`, `DebrisPool::PrepareFrame`, `SurfaceDamage::PrepareFrame`, `EmissiveVolumes::PrepareFrame` — each early-outs with no demand (§4) |
| 9 | `screen->RunFrameCompute` | hw_entrypoint.cpp:338 → `vk_renderdevice.cpp:651` → `VkComputeManager::RunFrame` (`vulkan/renderer/vk_compute.cpp:54`) | creates each Vk effect object lazily on first `Active`; `BeginWork` ends the render pass and opens `fx.compute` only when one records work (`vk_compute.cpp:112-123`) |
| 10 | `PinnedLightLive` → `lightmask.SetPinnedLightLive` | hw_entrypoint.cpp:246, :341 | loops 128 beam slots |
| 11 | `ExposureImpulses::Get().BeginFrame` | hw_entrypoint.cpp:345 → `hw_exposureimpulse.cpp:128` | `r_exposureimpulse` default **0** |
| per eye | `HWBillboard_SetEye`, `eye->SetUp`, multiview viewpoints, `PPCustomShaders::SetEye`, `heatrefraction/smokevolume/emissivevolumes.SetEye` | hw_entrypoint.cpp:361-506 | always |

Per eye (per `ProcessScene`, `hw_drawinfo.cpp:4597`, called from hw_entrypoint.cpp:482):

| Call (V) | Line | Gate |
|---|---|---|
| `mMeshParticles->SyncDefinitions`, `mParticleDefinitions->Sync`, `RefreshParticleAtlasLayout`, `SyncAtlasLayers` | 4609-4640 | buffer exists (Vulkan) — generation-checked (I: generation arguments) |
| `mGpuParticles->Sync`, `DebugReport` | 4651 | buffer exists |
| `mMeshParticles->Sync` | 4663 | buffer exists |
| `VRRig_DrawDebug` | 4673 | debug cvars |
| `SyncDrawnLines`, `ReportBeamLines` | 4674-4675 | O(`DrawnLineHigh`) |
| `SyncViewLights` | 4681 | to screen |
| `PPCustomShaders::SetEyeView` ×1-2 | 4693 | to screen |
| `SetupHeatSources`, `SetupShockwaves`, `SetupSmokeVolume`, `SetupEmissiveVolumes` | 4700-4712 | each O(live sources) |
| `SyncBloomOverride` | 4717 | to screen |

`RenderScene` (`hw_drawinfo.cpp:3722`): sweep-band uniforms (only if `Level->SweepMode>0`), `SetupVolumetricBeam` (`:3095`, loops `MAX_VOL_BEAMS`=32), `SetupHeatmap` (`:3233`, returns unless `Level->HeatScale>0`; when active it decays **all 256×256 cells every call** at render rate). `CreateScene` adds `DispatchBillboards` (`:3694`, returns when `Billboards` is empty). `RenderTranslucent`: GPU particles (`:3873`, gated `GpuParticleWritten>0` or debris billboards), soft-particle depth switch (`:3929-3932`, `:3991-3992`), drawn lines (`:4011`). `DrawScene`: debris meshes (`:4528`), `DrawWorldPlayerSprites` (`:4548`), `DrawHitscanTracers`, `DrawLaserSightWorld`, `VRWheel_Draw` (`:4583-4585`).

Post-process, `Postprocess::Pass1` (`common/rendering/hwrenderer/postprocessing/hw_postprocess.cpp:3557`): stock `exposure` → `customShaders("beforebloom")` → **`smokevolume`** (:659, skips unless `r_smoke` and sets) → **`volbeam`** (:88, skips with 0 beams) → **`emissivevolumes`** (:1103) → **`heatmap`** (:160) → **`heatrefraction`** (:254, heat or shock sources) → **`lightmask.RenderDebug`** (`r_lightmask_debug`) → `bloom` (now with pinned-bloom plan) → **`exposureimpulse`** (:3485, `Frame.Live`). Pass2 is stock (:3583).

End of frame (`d_main.cpp`): `EffectsGovernor::EndFrame` every frame (:1540, `r_effects_governor` default 1), `PerfLogEndFrame` if `r_perflog>0` (:1543), `PerfTrackEndFrame` if `perf_track` (:1546), `SolarXR_Tick` (:1572, returns unless `vr_slime_enable`, default 0). Hang watch beats (`d_main.cpp:1983-2012`, `common/platform/win32/i_hangwatch.cpp:267`: two relaxed atomics).

### 3.2 Tic (playsim)

`P_Ticker` (`p_tick.cpp:644`), per level:

| Call (V) | Line | Cost class |
|---|---|---|
| `AdvanceWorldClock` (slow-motion world clock) | 724 | O(1) |
| actor loop skips non-real-time actors when the world does not step | 731-737 | stock loop |
| `TickBillboards` | 762 | O(billboards) |
| surface stamp ageing | ~765 | 64 slots (`MAX_SURFACE_STAMPS`, g_levellocals.h:2597) |
| beam snapshot/claim release | ~775-835 | 128 slots |
| `SnapshotDrawnLines` | 837 | O(`DrawnLineHigh`) |
| `P_StepBeamOutlines` | 856 → `playsim/p_beamoutline.cpp:930` | 8 records (`BEAMOUTLINE_MAX_OUTLINES`) |
| `BeginEffectTic` | 863 → g_levellocals.h:2284 | 9 queue headers |
| `P_PlayerThink` per player (see below) | | |
| `WorldTick` (stock) / **`WorldStep`** (new event) | 896 | per handler |
| `P_TickRealTimeActors` when the world does not step | 902 | **O(all actors)** thinker walk |
| `VRSyncLog_OnTick` | 919 → `playsim/p_vrsynclog.cpp:179` | returns unless `vr_synclog` set; then hashes every actor |

`P_PlayerThink` additions (`playsim/p_user.cpp:1994-2037`), per player per tic: `P_LatchVRHands`, `UpdateCanonicalMainHandPose`, `VRTorso_Update`, `VRGear_Tick` (`p_vrgear.cpp:415`, O(spots)), `VRGear_PlayAfter`, `VRGrab_Tick` (`p_vrgrab.cpp:880`; blockmap box search `p_vrgrab.cpp:682`), **`P_VRLaserTraceTic`** (`p_user.cpp:98`: 2 × `Trace` of 8192 units, unconditional when a hand pose is valid), room-scale move, `VRTelemetry_Tic` (returns unless `vr_telemetry`). `VRGear_PlayTick` at :1891.

Per actor: `Affect_Tick(this)` (`playsim/p_mobj.cpp:4807` → `p_affect.cpp:821`, returns when inactive), `Affect_SweepStep` (:5220, only when watching), `Affect_Impact` (:2780, :3302). Thinker sleep: `FThinkerCollection::RunSleepCycle` every world step (`playsim/dthinker.cpp:162`, `:341`), O(sleepers).

Event handlers registered for every game by the engine pk3: `AddEventHandlers = "OtherPlayerTagHandler", "VRStabilizeSyncHandler"` (`wadsrc/static/mapinfo/common.txt:8`). `VRStabilizeSyncHandler.WorldTick` (`wadsrc/static/zscript/vr_stabilizesync.zs`) loops `MAXPLAYERS` every tic in every game; `OtherPlayerTagHandler.WorldTick` (`otherplayertags.zs:169`) returns at once in single player, in multiplayer it syncs a `VisualThinker` per other player and calls `AimTarget()` per player when the mode is 1.

### 3.3 Startup / level load

Definition lumps parsed once: `LoadParticleDefinitions` (`d_main.cpp:4483`; `gamedata/particledefs.cpp:2073`, lump `PARTICLEDEFS`), `LoadSurfaceDefinitions` (`gamedata/surfacedefs.cpp:357`), `LoadEmissiveVolumeDefinitions` (`gamedata/volumedefs.cpp:386`, `VOLUMEDEFS`), `LoadSurfaceDamageDefinitions` (`gamedata/damagedefs.cpp:657`, `DAMAGEDEFS`), `AFFECTDEF`, `MASSDEF`, `VRAVATAR` lumps (V: `FindLump` strings). `D_RunDataValidators` only with `-norun -validatedata` (`d_main.cpp:4647`). `profileManager.CollectProfiles` (`d_main.cpp:4870`). Level setup clears guard volumes and beam outlines (`p_setup.cpp`), `VRGear_Clear`/`VRGrab_Clear`/`VRTorso_Clear` (`g_level.cpp`, `g_game.cpp`).

## 4. Subsystem inventory (what is added, who calls it, default, status)

Status: ALIVE = runs by default on the owner's config (Vulkan, `vr_mode 15`); DEMAND = alive but does nothing until a mod/event asks; DORMANT = off by default or reachable only by a cvar/command line/other backend; DEAD = unreachable.

| Subsystem | Layer | Files (new) | Entry points → callers (V) | Default gate (V) | Cost class when idle / active | Status |
|---|---|---|---|---|---|---|
| OpenXR VR device (Vulkan) | P | `common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp` (8,632 lines), `stereo3d/openxr/oxr_loader.*`, `platform/win32/i_openXR.cpp` | `VRMode::GetVRMode` case `VR_OPENXR_MOBILE` (`hw_vrmodes.cpp:1293`) | `vr_mode 15` + `USE_OPENXR` (CMake `ENABLE_OPENXR`, default NO; `auto-setup-windows-vr.cmd:98` sets ON) | per frame (area 07) | ALIVE |
| OpenVR device (GL) | P | `rendering/gl/stereo3d/gl_openvr.cpp` (3,793), `LSMatrix.h`, `win32/i_openVR.cpp` | `hw_vrmodes.cpp:1283` `case VR_OPENVR` under `#ifdef USE_OPENVR` | `vr_mode 10` + GL backend | — | DORMANT (GL only; engine runs Vulkan) |
| GL OpenXR device | P | `gl/stereo3d/gl_openxrdevice.cpp/.h` | none — not in `src/CMakeLists.txt` (V: grep) | — | — | DEAD (never compiled) |
| QuestZDoom common glue | P | `QzDoom/qzdoom_common.cpp`, `QzDoom/VrCommon.h` | `VR_HapticEvent`, `VR_InputViaCommand`, teleport queues | — | per event | ALIVE (teleport burst producer DEAD, §8.3) |
| QuestZDoom no-SDL platform | P | `posix/nosdl/*` (9 files) | not in CMake | — | — | DEAD (never compiled) |
| VR weapon/inventory wheel | P | `common/rendering/hwrenderer/data/hw_vrwheel.cpp` (2,418) | `VRWheel_Draw` (`hw_drawinfo.cpp:4585`), `VRWheel_IsActive` ×9 in `g_game.cpp` | wheel open | O(1) idle | ALIVE |
| Billboards + SDF fonts | P | `rendering/hwrenderer/scene/hw_sdffont.*`; code in `hw_sprites.cpp:2229-2876`, `g_levellocals.h` | `TickBillboards` (p_tick.cpp:762), `DispatchBillboards` (hw_drawinfo.cpp:3694), ZScript `Level.AddBillboard*` | none until a mod adds | O(billboards) per tic and per scene | DEMAND |
| Sweep bands, surface glow, glow wave, fog slab, darkness, stamps, shapes, desaturation | P (+R) | shader code in `main.fp`, `func_surfacestamps.fp`; uniforms from `RenderScene` (hw_drawinfo.cpp:3730-3790), `FillFogDynLights` | `Level->Sweep*`, sector glow fields set by natives | inert until set | uniform upload O(8 bands); per-fragment branches always compiled in | DEMAND (shader cost always paid, I) |
| Volumetric beam cones | P | `pp/volumetricbeam.fp` | `SetupVolumetricBeam` (:3095) → `PPVolumetricBeam::Render` | 0 beams → return | O(32) per scene | DEMAND |
| Beams / drawn lines / beam lights / beam outlines | P+R | `hw_drawnlinebuffer.*`, `drawnlines.vp/.fp`, `beamglow.glsl`, `playsim/p_beamoutline.*` | tic: beam snapshot, `SnapshotDrawnLines`, `P_StepBeamOutlines`; frame: `BeamLights_HandOver`, `SyncDrawnLines`, drawn-line draw (:4011) | `r_drawnlines 1`, `r_beams_drawn 0` | O(128)+O(lines) per tic and per frame | DEMAND |
| Heatmap (floor heat grid) | P | `pp/heatmap.fp` | `SetupHeatmap` (:3233), natives `HeatmapAdd/At/Clear`, `SetHeatmap` | `HeatScale 0` | when active: 65,536-cell decay per `RenderScene` + grid upload | DEMAND; see §7.8 netplay hazard |
| Heat refraction + shockwaves | R | `pp/heatoffset.fp`, `heatwarp.fp`, `shockwaveoffset.fp`, `hw_shockwavecore.h` | `SetupHeatSources`/`SetupShockwaves` (:4700-4704) → `PPHeatRefraction::Render` | `r_heatrefraction 1`, `r_shockwave 1`, no sources → skip | O(sources ≤64/32) | DEMAND |
| Smoke volume (3D sim + march) | R | `hw_smokevolume.*` (2,370), `vk_smokevolume.*` (1,833), `hw_smoketilecover.h`, 5 compute shaders, `smokemarch/depth/blur/composite/temporal.fp` | `SmokeVolume::PrepareFrame` (hw_smokevolume.cpp:112) via PrepareFrameCompute; `VkSmokeVolume::Run`; `PPSmokeVolume::Render` | `r_smoke 1`, `Active = r_smoke && mHasDemand` (hw_smokevolume.cpp:126,175); allocates nothing until `EmitSmoke`/carve or `r_smoke_computetest` | idle O(queue); active: GPU compute per world step (≤ `MAX_STEPS_PER_FRAME` = 2, `hw_framecompute.h:333`) + per-eye march | DEMAND |
| Level collision field bake | R | `hw_levelfield.*`, `vk_levelfield.*`, `field_bake.comp`, `hw_levelsolidity.*` | `LevelField::PrepareFrame` (hw_levelfield.cpp:90) | `r_particlecollision 1`; demand = a colliding particle def or debris pool wants it | lazy bake | DEMAND |
| Sector plane buffer | R | `hw_sectorplanes.*`, `hw_sectorplanebuffer.*` | `SectorPlanes::BeginFrame` every frame (hw_sectorplanes.cpp:53: counters only); `EnsureLevel` (:68, O(sectors·lines) once per level) on first poll by smoke/levelfield/debris | used by the above | O(1) idle | DEMAND |
| Debris pool (GPU rigid bits) | R | `hw_debrispool.*` (1,351), `hw_debrislanding.*`, `vk_debrispool.*`, `debris_step.comp`, `hw_debrisframe.h` | `DebrisPool::PrepareFrame` (hw_debrispool.cpp:374), draws at hw_drawinfo.cpp:3977, :4528 | `r_gpuparticles && r_debris` (both 1), pool sized `r_debris_pool` (16384 pieces × 176 B, `hw_debrisframe.h:42,98`) | pool allocated when first needed (I: `mCapacity==0` branch) | DEMAND |
| Surface damage paint | R | `hw_surfacedamage.*`, `vk_surfacedamage.*`, `damage_stamp/cool.comp`, `gamedata/damagedefs.*` | `SurfaceDamage::PrepareFrame` (hw_surfacedamage.cpp:168); shader side `#define SURFACE_DAMAGE` in **every** Vulkan main.fp program (vk_shader.cpp:1171) | `r_damage 1`; pages allocated on first paint | per-fragment lookup compiled into all world shaders (I: branch cost) | DEMAND (+ always-compiled shader path) |
| Effect lights | R | `hw_effectlights.*`, `hw_effectlightcore.h`, `hw_effectlightbuffer.*`, `hw_particlelights.h` | BeginFrame/PrepareFrame/AssignShadowRows; `#define EFFECT_LIGHTS` in every Vulkan main.fp program (vk_shader.cpp:1176) | `r_effectlights 1`, `r_effectlights_max 1024` | O(queued lights) | DEMAND (+ shader path) |
| Emissive volumes | R | `hw_emissivevolumes.*`, `hw_emissivevolumecore.h`, `vk_emissivevolumes.*`, `emissivevolume.fp`, `screentilemask.fp`, `sharedmarchwarp.fp`, `gamedata/volumedefs.*` | BeginFrame (:246), PrepareFrame (:429), `PPEmissiveVolumes::Render` | `r_emissivevolumes 3` | O(volumes) | DEMAND |
| Exposure impulse (flash blindness) | R | `hw_exposureimpulse.*`, `pp/exposureimpulse.fp` | hw_entrypoint.cpp:345; `PPExposureImpulse::Render` | `r_exposureimpulse 0` | — | DORMANT |
| Hearing impulse + level ray | R | `sound/s_hearingimpulse.*`, `rendering/r_levelray.*` | natives (`scripting/vmthunks.cpp`), `s_doomsound.cpp` | per event | 8 `LevelRay::Walk` per impulse (s_hearingimpulse.cpp:127-166) | DEMAND |
| GPU particles | R | `hw_gpuparticlebuffer.*`, `hw_gpuparticlewindow.h`, `hw_particledefbuffer.*`, `hw_viewlightbuffer.*`, `gpuparticles.vp/.fp` (1,293+397), `gamedata/particledefs.*` (2,631) | spawn in playsim (`g_levellocals.h:2805-3030`); `Sync` per eye (:4651); draw (:3873) | `r_gpuparticles 1`, ring 65536 (`r_gpuparticles_ringsize`) allocated on first spawn (g_levellocals.h:2701,2737) | O(count) per spawn; per-eye sync + one instanced draw | DEMAND (soft-depth switch cost §7.3) |
| Mesh particles | R | `hw_meshparticles.*`, `meshparticles.vp/.fp` | :4609, :4663, :4513/:4528 | `r_meshparticles 1` | per mesh def | DEMAND |
| Effects governor | R | `hw_effectsgovernor.*` | `EffectsGovernor::EndFrame` (d_main.cpp:1540); readers in hw_drawinfo, emissive, effect lights, particle spawn (g_levellocals.h:3011) | `r_effects_governor 1` | O(1) per frame | ALIVE |
| perf_track / r_perflog / perf bench / bench demo | R | `hw_perftrack.*`, `hw_perflog.*`, `g_perfbench.*`, `g_benchdemo.cpp` | d_main.cpp:1543-1546; `PerfBench::*` in g_game.cpp/d_net.cpp; scene groups gated `perfGroups` | `perf_track 0`, `r_perflog 0`; `perf_bench_hash 1` acts only while recording a single-player demo (`g_perfbench.cpp:230`) | 0 when off | DORMANT |
| VR clocks / stats | R | `hw_clock.cpp` (+31 `glcycle_t`: `VRSceneEyes`, `VRSubsectors`, `VRSubmit`, `WTWallJobs`, `WallMerge`, …), `ADD_STAT(vrstats)`, `bufferstats`, `remotestats` | Clock/Unclock always | — | O(1) per clock | ALIVE |
| GPU checkpoints | R | `vk_commandbuffer.cpp:223-269`, `vk_renderdevice.cpp:306` | every `PushGroup`/`PopGroup` | `vk_gpu_checkpoints 1` | §7.2 | ALIVE |
| Hang watch | R | `platform/win32/i_hangwatch.cpp`, `engine/i_hangwatch.h` | d_main.cpp:1983-2012 (16 beats in src) | always | 2 relaxed atomics per beat + 1 watcher thread | ALIVE |
| Avatar / body IK / rig / arm IK / grasp / hand & joint drive | R | `r_data/vr_avatar*.cpp`, `vr_body.cpp` (3,559), `vr_rig.cpp` (2,501), `vr_armik.cpp`, `vr_armsolver.cpp`, `vr_bodyinfer*.cpp`, `vr_grasp.cpp`, `model_handdrive.cpp`, `model_jointdrive.cpp`, `vr_rig_selftest.cpp` (1,709) | `VRAvatar_Pose` from the model draw (`r_data/models.cpp:4174, 4224`) for actors with `VRRigRole==1` (set by script, `actor.zs:761`); `ModelJointDrive_Apply` (:4168, :4221); OpenXR device calls `VRBodyInfer_*`, `VRRig_StandEyeTic` | role set by the RS_Avatar mod; `vr_body 2`, `vr_avatar_armik 1`, `vr_grasp 1`, `vr_bodyinfer 0` | per avatar model draw (per eye, I: no per-frame cache seen at the entry `vr_avatar_pose.cpp:1422`) | DEMAND (mod-driven); area 04/avatar |
| Holsters / gear, torso, grab, hand latch, gun fit, slots | R | `playsim/p_vrgear.cpp` (2,114), `p_vrtorso.cpp`, `p_vrgrab.cpp` (1,397), `p_vrhand.cpp`, `p_vrgunfit.cpp`, `p_vrslots.h` | `P_PlayerThink` (p_user.cpp:1994-2008); `VRGear_Serialize` etc. in `player_t::Serialize` (:2513-2517) | gear rows parsed lazily (`EnsureParsed`); returns early with no torso | per player per tic O(spots)+blockmap box | ALIVE (cheap with no rig) |
| Laser trace (sim) | R | p_user.cpp:98 | `P_PlayerThink` :2011 | always | 2 traces × 8192 u per player per tic | ALIVE |
| Affect (push/stagger/stun/knockdown/deflect) | R | `playsim/p_affect.*`, `AFFECTDEF` lump | p_mobj.cpp:2780, 3302, 4807, 5220; natives in `vmthunks_actors.cpp` | `FActorAffect::Active()` | O(1) per actor per tic | DEMAND |
| Guard volumes (blocking) | R | `playsim/p_guard.*` | `P_GuardIntercept/BodyAim/RecordBlock` in p_map.cpp, `P_GuardBlocksDamage` in p_interaction.cpp | none set | per attack | DEMAND (`P_GuardBlocksMelee`, `P_GuardBlocks` DEAD §8.1) |
| Hurt volumes / drawn shapes (sweep hit tests) | R | functions in `playsim/p_map.cpp:341-1288` | natives | per call | O(shapes) | DEMAND |
| Floor regions | R | `playsim/p_floorregion.*` | natives `QueryFloorRegion` etc., used by `rs_boutring.zs` | per call | — | DEMAND |
| World clock (slow motion) + `WorldStep` event | R | p_tick.cpp, `events.cpp:1079` | `SetTimeScale` native | scale 1 | O(all actors) per real tic only while slowed | DEMAND |
| Bout director / chase brain (ZScript in engine pk3) | R | `zscript/rs_bout.zs`, `rs_boutring.zs`, `rs_chasebrain.zs` (compiled via `zscript.txt:63-73`) | `RS_BoutDirector` registers only via a mod's `gameinfo { AddEventHandlers = "RS_BoutDirector" }` (zscript.txt:57); no in-tree registration (V: grep) | not registered | 0 | DORMANT (compiled, never registered in-tree) |
| Thinker sleep | R (Selaco port) | `playsim/dthinker.cpp` | `RunSleepCycle` each step | none sleeping | O(sleepers) | DEMAND |
| Netplay: late join, resync, sync log, UPnP, telemetry, replay, ghost, VR demo frames | R | `d_net.cpp:4263-4475`, `engine/i_upnp.*`, `p_vrsynclog.*`, `p_vrtelemetry.*`, `p_vrreplay.*`, `p_vrghost.*`, `p_vrdemo.*`, `i_wsclient.*` | `NetState_*`, `VRSyncLog_OnTick`, `VRReplay_OnBuildTiccmd`/`Ghost_OnBuildTiccmd` (g_game.cpp) | `net_latejoin 1`, `net_autoresync 1`, `net_upnp 1` (host only), `vr_synclog ""`, `vr_telemetry ""`, `ghost_* ""`, `vr_replay_* ""` | idle O(1) | ALIVE (netplay), others DORMANT |
| SolarXR / SlimeVR trackers | R | `engine/i_solarxr.*`, `i_wsclient.*` | `SolarXR_Tick` (d_main.cpp:1572) | `vr_slime_enable 0` | one cvar read per frame | DORMANT |
| Models: Studio (MDL v10+), Quake MDL, VTF textures, DDS quality | R | `common/models/models_studio.cpp` (1,279), `models_mdl.cpp`, `textures/formats/vtftexture.cpp` | model/texture loaders | by file type | load time | DEMAND |
| Hitscan tracers, laser sight | P | `playsim/p_hitscantracer.*`, hw_weapon.cpp:511-1397 | `P_QueueHitscanTracer` (p_map.cpp, events.cpp), draw hw_drawinfo.cpp:4583-4584 | `toggle_laser_sight` | per frame O(tracers) | ALIVE |
| Multiplayer launcher, command-line profiles, cheat/chat/console text menus | P | `engine/multiplayerlaunch.*`, `menu/profiledef.*`, `zscript/engine/ui/menu/*.zs` | menus, `d_main.cpp:2639, 4870` | — | — | ALIVE (`M_HasPendingMultiplayerLaunch`, `I_InitProfiles` DEAD) |
| Data validators | R | `gamedata/datavalidation.*`, `zscript/engine/datavalidator.zs` | `d_main.cpp:4647` | `-norun -validatedata` | — | DORMANT |
| Level solid query | R | `r_data/level_solid_query.*` | **no caller on main** (header included only by its own .cpp) | — | — | DEAD on main (alive on selaco, §9) |
| Removed vs pristine | — | `playsim/p_physics.*` (rigid-body physics, deleted after the import) | — | — | — | gone |

Not fork additions (checked against stock): dithered transparency (`EFF_DITHERTRANS`, `dithertransflag`) is stock UZDoom 5.0.0 (V: present in stock `hw_shaderpatcher.cpp`, `hw_walls.cpp`). "Model swapper" = stock `A_ChangeModel` (no fork subsystem by that name found). "Wall cache", "r_snapshot", static level residency exist only on `renderer` (§9.1).

## 5. Per-function table (fork functions on per-frame / per-tic paths)

| Function | File:line | Called from | Per | Cost class | Notes |
|---|---|---|---|---|---|
| `EmissiveVolumes::BeginFrame` | rendering/hwrenderer/hw_emissivevolumes.cpp:246 | hw_entrypoint.cpp:274 | frame | O(queue + pool) | V; returns with no volumes |
| `BeamLights_HandOver` | rendering/hwrenderer/scene/hw_drawinfo.cpp:326 | hw_entrypoint.cpp:286 | frame | O(128) | V |
| `EffectLights::BeginFrame` | rendering/hwrenderer/hw_effectlights.cpp:153 | hw_entrypoint.cpp:293 | frame | O(queue + pool) | V |
| `EffectLights::PrepareFrame` | hw_effectlights.cpp:386 | PrepareFrameCompute | frame | O(pool) bin + upload | V; returns when not begun/active |
| `EffectLights::AssignShadowRows` | hw_effectlights.cpp | CollectLights (hw_entrypoint.cpp:153) | frame (shadow pass) | O(point lights) | V |
| `DynamicLightShadowRowsWanted` | rendering/hwrenderer/hw_dynlightdata.cpp:210 | hw_entrypoint.cpp:299 | frame | O(1) off; O(lights) on | V |
| `R_UpdatePoseAnchoredLights` | hw_entrypoint.cpp:185 | :330 | frame | O(anchored lights) | V |
| `PrepareFrameCompute` | hw_entrypoint.cpp:217 | :337 | frame | sum of 7 managers | V |
| `SectorPlanes::BeginFrame` | rendering/hwrenderer/hw_sectorplanes.cpp:53 | PrepareFrameCompute | frame | O(1) (O(sectors) on 2^32 wrap) | V |
| `SmokeVolume::PrepareFrame` | hw_smokevolume.cpp:112 | PrepareFrameCompute | frame | O(queue) idle | V |
| `LevelField::PrepareFrame` | hw_levelfield.cpp:90 | PrepareFrameCompute | frame | O(ring scan) | V (`ScanRing`) |
| `DebrisPool::PrepareFrame` | hw_debrispool.cpp:374 | PrepareFrameCompute | frame | O(queue) | V |
| `SurfaceDamage::PrepareFrame` | hw_surfacedamage.cpp:168 | PrepareFrameCompute | frame | O(queue) | V |
| `EmissiveVolumes::PrepareFrame` | hw_emissivevolumes.cpp:429 | PrepareFrameCompute | frame | O(n log n) sort when volumes | V |
| `VkComputeManager::RunFrame` | common/rendering/vulkan/renderer/vk_compute.cpp:54 | vk_renderdevice.cpp:651 | frame | 5 null/`Run` checks idle | V |
| `PinnedLightLive` | hw_entrypoint.cpp:246 | :341 | frame | O(128) | V |
| `ExposureImpulses::BeginFrame` | rendering/hwrenderer/hw_exposureimpulse.cpp:128 | :345 | frame | O(queue) | V |
| `HWBillboard_SetEye` | rendering/hwrenderer/scene/hw_sprites.cpp:2229 | hw_entrypoint.cpp:377 | eye | O(1) | V |
| particle `Sync*` (defs, atlas, ring, mesh) | hw_drawinfo.cpp:4609-4663 | ProcessScene | eye | generation compare / upload of changed slots (I) | V call; cost I |
| `VRRig_DrawDebug` | r_data/vr_rig.cpp | hw_drawinfo.cpp:4673 | eye | O(1) unless debug | V |
| `SyncDrawnLines` / `ReportBeamLines` | hw_drawinfo.cpp:372 / :1889 | :4674-4675 | eye | O(lines) | V |
| `SyncViewLights` | hw_drawinfo.cpp:671 | :4681 | eye | O(view lights) | V |
| `SetupHeatSources` / `SetupShockwaves` / `SetupSmokeVolume` / `SetupEmissiveVolumes` | hw_drawinfo.cpp:866 / 1105 / 1352 / 1517 | :4700-4712 | eye | O(sources) | V |
| `SyncBloomOverride` | hw_drawinfo.cpp:1844 | :4717 | eye | O(1) | V |
| `HWDrawInfo::DispatchBillboards` | hw_drawinfo.cpp:3546 | CreateScene :3694 | scene | O(billboards) | V |
| `HWDrawInfo::SetupVolumetricBeam` | hw_drawinfo.cpp:3095 | RenderScene :3792 | scene | O(32) | V |
| `HWDrawInfo::SetupHeatmap` | hw_drawinfo.cpp:3233 | RenderScene :3793 | scene | O(65,536) when heatmap on | V |
| GPU particle draw + soft-depth switch | hw_drawinfo.cpp:3873-3992 | RenderTranslucent | eye | 1 instanced draw; 2 render-pass restarts | V |
| `DebrisPool::DrawBillboards/DrawMeshes` | hw_debrispool.cpp | hw_drawinfo.cpp:3977, 4528 | eye | per piece batch | V |
| `DrawWorldPlayerSprites`, `DrawHitscanTracers`, `DrawLaserSightWorld` | hw_weapon.cpp:2586, 1333, 1397 | DrawScene :4548, :4583-4584 | eye | O(psprites/tracers) | V |
| `VRWheel_Draw` | common/rendering/hwrenderer/data/hw_vrwheel.cpp:2385 | :4585 | eye | O(1) closed | V |
| PP `smokevolume/volbeam/emissivevolumes/heatmap/heatrefraction/exposureimpulse.Render` | hw_postprocess.cpp:659/88/1103/160/254/3485 | Pass1 :3565-3579 | eye | early-out O(1) idle; full-screen passes active | V |
| `EffectsGovernor::EndFrame` | common/rendering/hwrenderer/data/hw_effectsgovernor.cpp:79 | d_main.cpp:1540 | frame | O(1) | V |
| `SolarXR_Tick` | common/engine/i_solarxr.cpp:433 | d_main.cpp:1572 | frame | 1 cvar read | V |
| `I_HangWatchBeat` | common/platform/win32/i_hangwatch.cpp:267 | d_main.cpp main loop | 3-4 per loop | 2 atomics | V |
| `VkCommandBufferManager::PushGroup/PopGroup` (fork part) | vulkan/system/vk_commandbuffer.cpp:522-556 | ~29 PP sites + compute/effects | group | 2 string allocs + 2 `std::set` lookups + NV checkpoint | V |
| `FLevelLocals::AdvanceWorldClock` | g_levellocals.h:1579 | p_tick.cpp:724 | tic | O(1) | V |
| `FLevelLocals::TickBillboards` | p_tick.cpp (def) | :762 | tic | O(billboards) | V |
| stamp ageing / beam snapshot | p_tick.cpp ~765-835 | P_Ticker | tic | O(64)+O(128) | V |
| `SnapshotDrawnLines` | g_levellocals.h:3737 | p_tick.cpp:837 | tic | O(lines) | V |
| `P_StepBeamOutlines` | playsim/p_beamoutline.cpp:930 | p_tick.cpp:856 | tic | O(8) | V |
| `BeginEffectTic` | g_levellocals.h:2284 | p_tick.cpp:863 | tic | 9 × O(1) | V |
| `P_TickRealTimeActors` | p_tick.cpp (static) | :902 | tic while slowed | O(all actors) walk | V |
| `VRSyncLog_OnTick` | playsim/p_vrsynclog.cpp:179 | p_tick.cpp:919 | tic | O(1) off; O(actors) hash on | V |
| `P_LatchVRHands`, `UpdateCanonicalMainHandPose` | playsim/p_vrhand.cpp; p_user.cpp:125 | p_user.cpp:1994-1995 | player·tic | O(1) | V |
| `VRTorso_Update` | playsim/p_vrtorso.cpp | :1998 | player·tic | O(1) | V |
| `VRGear_Tick` | playsim/p_vrgear.cpp:415 | :2001 | player·tic | O(spots); early return without torso | V |
| `VRGear_PlayAfter`, `VRGear_PlayTick` | p_vrgear.cpp | :2004, :1891 | player·tic | O(1) unless `vr_gear_play` | V |
| `VRGrab_Tick` | playsim/p_vrgrab.cpp:880 | :2008 | player·tic | 2 hands; blockmap box search (:682) while seeking | V |
| `P_VRLaserTraceTic` | playsim/p_user.cpp:98 | :2011 | player·tic | 2 × `Trace` 8192 u | V |
| `VRTelemetry_Tic` | playsim/p_vrtelemetry.cpp:126 | :2037 | player·tic | O(1) off | V |
| `Affect_Tick` | playsim/p_affect.cpp:821 | p_mobj.cpp:4807 | actor·tic | O(1), returns inactive | V |
| `FThinkerCollection::RunSleepCycle` | playsim/dthinker.cpp | :162, :341 | step | O(sleepers) | V |
| `VRStabilizeSyncHandler.WorldTick` | wadsrc/static/zscript/vr_stabilizesync.zs | event dispatch | tic | O(MAXPLAYERS) in VM, every game | V |
| `OtherPlayerTagHandler.WorldTick` | wadsrc/static/zscript/otherplayertags.zs:169 | event dispatch | tic | O(1) SP; O(players) MP + `AimTarget` traces | V |

## 6. Data structures that matter for speed

- `FLevelLocals` (`g_levellocals.h`, +3,378 lines vs stock, V): fixed slot arrays walked every tic and/or frame: beams `MAX_BEAMS`=128 (:3110) with Prev*/Claim*/Light* parallel arrays, `MAX_VOL_BEAMS`=32 (:1981), `MAX_HEAT_SOURCES`=64 (:2057), `MAX_SHOCKWAVES`=32 (:2168), `MAX_SURFACE_STAMPS`=64 (:2597), `MAX_SHAPES`=128 (:2505), `MAX_SWEEP_BANDS`=8 (:2380), `MAX_FOG_DISTURB`=32 (:3920), `BeamOutlines[8]` (:3370), heat grid `HEAT_RES`=256 → 65,536 floats (:4052). Nine `FEffectTicQueue` queues (:2252-2279; 64-512 entries each) are the playsim→renderer hand-off: written in play scope, read by the renderer by serial/cursor (V: `ReadQueue` calls in each manager).
- GPU particle ring: `TArray<GpuParticleRecord> GpuParticles` (g_levellocals.h:2701), sized `r_gpuparticles_ringsize` (65,536) on first spawn (:2737); written by playsim spawn natives, uploaded per eye by `Sync`.
- Debris pool: 16,384 pieces × 176 bytes default (`hw_debrisframe.h:42, 98`) on the GPU.
- Smoke volume: 3D textures sized by `r_smoke_quality` (cvar text says "2 = about 157 MB"; a description string, not a measurement). Allocated only while demand lasts (V: `out.Active` gate, `vk_compute.cpp:65-68`).
- `AActor` (`playsim/actor.h`, +1,811 lines): ~118 more member-declaration lines than stock by a crude count (V count, I effect: larger actor → more cache misses in every thinker walk; settle with `sizeof(AActor)` in both builds). New per-actor data includes `FActorAffect Affect`, laser trace targets/hit positions, `VRRigRole`, model surface/tint/scale fields.
- `main.fp`: 5,128 lines vs 944 stock (V, `wc -l`). Vulkan prepends `func_surfacestamps.fp` (593 lines) and `func_spriteoutline.fp` (153) to every material program and defines `SURFACE_DAMAGE` and `EFFECT_LIGHTS` for every `main.fp` program (`vk_shader.cpp:1171-1192`). Larger SPIR-V, more registers, more uniform-gated branches per fragment (I).
- Checkpoint label set: process-lifetime `std::set<std::string>` capped at 1,024 (`vk_commandbuffer.cpp:237-248`).

## 7. What slows it down (fork additions only, ranked)

1. **Uber-shader growth of `main.fp`** (V size/defines; I cost). Every wall/flat/sprite/model fragment on Vulkan runs a program compiled with surface damage, effect lights, stamps, sprite outline, sweep bands, glow wave, fog slab and darkness code; features are uniform-gated, so idle cost is branches + register pressure, not work. Confidence: medium. Settle: `stat gpu` / `r_perflog` `scene.opaque` and `gpu.opaque.*` on a scene with no fork effects, fork build vs a build with `main.fp`/`vk_shader.cpp:1171-1192` reverted to stock; also check SPIR-V register count with `spirv-cross`/vendor tools.
2. **`vk_gpu_checkpoints 1` by default + PushGroup interning** (V `vk_commandbuffer.cpp:237-269, 522-556`, `vk_renderdevice.cpp:306`). On the owner's NVIDIA GPU the device is created with `VK_NV_device_diagnostic_checkpoints` and every Push/Pop records `vkCmdSetCheckpointNV`; independent of that, every `PushGroup` now builds two `std::string`s and does two `std::set` lookups before the stock `gpuStatActive` early return (stock returned at once: stock `vk_commandbuffer.cpp:208-211`). ~29 post-process group sites plus compute/effect groups per eye. Confidence: high that it costs something, low on size (CPU µs; GPU checkpoint overhead is driver-defined). Settle: A/B `vk_gpu_checkpoints 0` (read at device creation, restart needed) with `perf_track` `frame_ms`/`cost_ms` and `stat gpu`.
3. **Soft GPU particles break the scene pass** (V `hw_drawinfo.cpp:3873-3932, 3991`; `hw_cvars.cpp:311` default 8.0; the comment at :3919 "At 0, the default" is stale). Once `Level->GpuParticleWritten>0` (any GPU particle spawned on the level, I: counter never resets mid-level) each eye ends and restarts the scene render pass twice (`SetSceneDepthReadable(true/false)`), every frame. Confidence: high that it happens; cost = 2 pass breaks per eye (load/store of colour+depth, I). Settle: `r_perflog` group `fx.depthread` (exists, :3930) and A/B `r_gpuparticles_soft 0`.
4. **Per-tic VR sim additions** (V): `P_VRLaserTraceTic` = 2 × 8192-unit `Trace` per player per tic, unconditional (`p_user.cpp:98-123`); `VRGrab_Tick` blockmap search (`p_vrgrab.cpp:682`); gear/torso O(spots). Plus `VRStabilizeSyncHandler` and `OtherPlayerTagHandler` registered for every game (`mapinfo/common.txt:8`), i.e. two extra VM `WorldTick` dispatches per tic. Previously measured (HANDOFF_RENDER_2026-11-11.md:269, renderer-branch `ticprofile` build): "`WorldTick` handlers 2.5-3.7" ms per tic — that total includes the mods' handlers; the two engine handlers' share is unknown. Settle: `ticprofile` (renderer) or `perf_track` `think_ms` with the two handlers removed from `common.txt`; count `Trace` calls per tic.
5. **Per-frame effect bookkeeping** (V): ~20 fork calls per frame plus ~12 per eye (§3.1), most O(1)-O(128) loops with early returns; per-eye particle definition/atlas/ring `Sync` calls even when nothing changed (I: generation compare). Confidence: low-medium (each small). Settle: `r_perflog` CPU sample `fx.compute` (vk_compute.cpp:108) and a `perf_track` A/B with `r_gpuparticles 0 r_meshparticles 0 r_effectlights 0 r_drawnlines 0`.
6. **Bigger `AActor` / `FLevelLocals`** (V growth, I effect on cache). Every stock per-actor walk (thinkers, interpolation reset — previously measured 6.5-7.8 ms per tic on the owner's mod list, HANDOFF_RENDER_2026-11-11.md:269) touches larger objects. Settle: `sizeof(AActor)` stock vs fork; `perf stat` cache misses on a 30k-thinker map.
7. **Slow-motion real-time actor walk** (V `p_tick.cpp:902`): O(all actors) thinker iteration per real tic while the world is slowed. Dormant until `SetTimeScale<1` (only `rs_bout.zs` calls it in-tree).
8. **Heatmap decay on the render side** (V `hw_drawinfo.cpp:3233-3260`): 65,536 cells per `RenderScene` (per eye, per portal scene) while a mod has the heatmap on, plus the grid re-upload when dirty. Also a **netplay hazard**: the decay is driven by `screen->FrameTime` and writes `Level->HeatIntensity`, which play code can read through the `clearscope` native `HeatmapAt` (`wadsrc/static/zscript/doombase.zs:2098`, impl `scripting/vmthunks.cpp:5618-5631`); the comment at doombase.zs:2092 proposes a spawn director reading it. Frame-rate-dependent state read by play code would desync (I: no in-tree play caller).
9. **Demand-driven GPU effects when active** (smoke 5 compute programs + march/temporal/blur/composite per eye, debris step, field bake, surface damage stamp/cool, emissive march): heavy but gated; costed in areas 04/05/06. `r_effects_governor` trims them under load.
10. **Hang watch, governor, VR clocks**: O(1) per frame; negligible (I).

## 8. Hooks, cvars and dead code

### 8.1 Dead functions (defined, never called) — main

Evidence for every row: `grep -rnw <name> src wadsrc/static/zscript` returns only the definition and, where present, its prototype (V, whole-tree index; header-only hits checked by hand). Layer P = present at the pristine import; R = added later.

| Function | Def | Layer | Note |
|---|---|---|---|
| `VR_QueueTeleportCommandBurst` | QzDoom/qzdoom_common.cpp:84 | P | sole writer of the burst; see §8.3 |
| `VR_QueueMultiplayerRoomscaleTeleportTarget` | QzDoom/qzdoom_common.cpp:113 | P | |
| `M_HasPendingMultiplayerLaunch` | common/engine/multiplayerlaunch.cpp:163 | P | |
| `PerfTrack::ReasonName` | common/rendering/hwrenderer/data/hw_perftrack.cpp:340 | R | |
| `VRWheel_ShouldSuppressGameplayInput` | common/rendering/hwrenderer/data/hw_vrwheel.cpp:2186 | P | |
| `VRWheel_GetTransform` | hw_vrwheel.cpp:2226 | P | |
| `VkSmokeVolume::GetDensityHeatView` | common/rendering/vulkan/renderer/vk_smokevolume.cpp:138 | R | named only in a comment, vk_smokevolume.h:38 |
| `VkSmokeVolume::GetVelocityView` | vk_smokevolume.cpp:144 | R | |
| `FrameRenderModeName`, `PostControllerAxisTransitions`, `NonLinearFilter`, `LogBoundSourcesForAction`, `IsSRGBSwapchainFormat` (all `static`) | common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:1279, 1514, 1629, 1763, 1913 | P | unused statics |
| `VKOpenXRDeviceMode::SubmitFrame` (virtual, base `VRMode::SubmitFrame` hw_vrmodes.h:225) | vk_openxrdevice.cpp:5997 | P | dead virtual hook: no call to `SubmitFrame(` anywhere |
| `VKOpenXRDeviceMode::GetControllerProfile` (virtual, base hw_vrmodes.h:270) | vk_openxrdevice.cpp:7981 | R | dead virtual hook |
| `VulkanRenderDevice::CacheProgress` (virtual, base `common/rendering/v_video.h:278`) | common/rendering/vulkan/system/vk_renderdevice.cpp:1026 | P | dead virtual hook |
| `VkRenderState::DrawInstanced` (virtual, base `hw_renderstate.h:1219`) | common/rendering/vulkan/renderer/vk_renderstate.cpp:80 | R | dead virtual hook; mesh particles do not use it |
| `PerfBench::HashGroupName` | g_perfbench.cpp:122 | R | |
| `EmissiveVolumeDefinitionCount` | gamedata/volumedefs.cpp:481 | R | |
| `I_InitProfiles` | menu/profiledef.cpp:152 | P | wrapper; `profileManager.CollectProfiles()` is called directly at d_main.cpp:4870 |
| `Affect_NoteMeleeWindup` | playsim/p_affect.cpp:688 | R | |
| `P_BeamOutlineDescribe` | playsim/p_beamoutline.cpp:389 | R | |
| `P_GuardBlocksMelee` | playsim/p_guard.cpp:434 | R | also makes `P_GuardBlocks` (p_guard.cpp:411) dead: its only call is p_guard.cpp:444 inside `P_GuardBlocksMelee` |
| `VRDemo_GetControllerTransform` | playsim/p_vrdemo.cpp:857 | R | |
| `P_ClearVRHands` | playsim/p_vrhand.cpp:186 | R | |
| `VRTorso_SeatedEye` | playsim/p_vrtorso.cpp:447 | R | (main only) |
| `LevelSolidQuery` | r_data/level_solid_query.cpp:115 | R | whole file dead on main: `level_solid_query.h` is included only by its own .cpp (V grep); its header names `model_reach.cpp` as first caller, which exists only on `selaco` |
| `VRAvatar_Clear` | r_data/vr_avatar.cpp:473 | R | |
| `AxisRotation`, `StripScale` (static) | r_data/vr_avatar_pose.cpp:706, 1048 | R | |
| `VRBody_SetFit`, `VRBody_GetFit` | r_data/vr_body.cpp:333, 339 | R | |
| `VRRig_TestIQMRoundTrip` | r_data/vr_rig.cpp:1675 | R | |
| `LevelSolidity::IsPointOpen` | rendering/hwrenderer/hw_levelsolidity.cpp:259 | R | |
| `I_OpenVRGetPitch`, `I_OpenVRGetDirectionalMove` | win32/i_openVR.cpp:528, 537 | P | file compiled only with `ENABLE_OPENVR` |
| `VR_GetUIScale` | common/2d/v_draw.cpp:129 | P | |
| `C_DoCommandC` | common/console/c_dispatch.cpp:184 | P | `extern "C"` decl c_dispatch.h:70, no C caller |
| `I_GetNetWaitBackend`, `I_GetNetWaitSourceContext` | common/engine/i_net.cpp:737, 752 | P | |
| `Host_LateJoinActive` | common/engine/i_net.cpp:1965 | R | |
| `getMenuState` | common/menu/menu.cpp:531 | P | QuestZDoom JNI remnant |
| `main_android` | common/platform/posix/sdl/i_main.cpp:182 | P | inside `#ifdef __ANDROID__` (i_main.cpp:181-183) |
| `VR_GetScriptLaserRange` | common/rendering/hwrenderer/data/hw_vrmodes.cpp:1928 | P | |
| `VR_ClearHandWorldPos`, `VR_ClearHandSamples` | hw_vrmodes.cpp:2046, 2061 | R | |
| `PP_ProjectWorldForView`, `PP_GetViewFocalY` | common/rendering/hwrenderer/postprocessing/hw_postprocess.cpp:2909, 2915 | R | |
| `PP_NoteWorldToView`, `PP_NoteDepthToViewScale` | postprocessing/hw_postprocessshader.cpp:52, 59 | R | |
| `D_GetStartupQueryIWad`, `D_GetStartupDefaultIWad`, `D_GetStartupLoadSupportWad` | d_main.cpp:619, 624, 629 | P | |
| `VR_DoomMain` | d_main.cpp:5249 | P | QuestZDoom entry remnant |
| `NetState_InProgress` | d_net.cpp:4395 | R | |
| `getGameState` | g_game.cpp:443 | P | QuestZDoom JNI remnant |
| `DThinker::CallSleep` | playsim/dthinker.cpp:1265 | R | |
| `SetModelBoneLengthShared` (static) | playsim/p_actionfunctions.cpp:6013 | R | |
| `HudAnchor_GetOffset` | r_data/models.cpp:124 | P | |
| `TrackedPoseSourceName` | rendering/r_utility.cpp:1727 | R | |

Total 60 definitions (+ `P_GuardBlocks` transitively). Branch check (V, same grep): `LevelSolidQuery`, `DrawInstanced`, `CacheProgress`, `SubmitFrame`, `VR_QueueTeleportCommandBurst` are equally dead on renderer and netplay; on selaco `LevelSolidQuery` is alive (3 refs, `r_data/model_reach.cpp`), and `P_GuardBlocksMelee`, `VRAvatar_Clear`, `GetControllerProfile`, `PerfTrack::ReasonName` do not exist.

### 8.2 Hooks: ZScript virtuals and events the fork added

| Hook | Declared | Engine dispatch (V) | Status |
|---|---|---|---|
| `StaticEventHandler.WorldStep()` | wadsrc/static/zscript/events.zs:197-201, events.h:342,538 | `DEFINE_EVENT_LOOPER(WorldStep)` events.cpp:1079, `DStaticEventHandler::WorldStep` events.cpp:2109, called p_tick.cpp:896 | ALIVE (fires every world step; empty base) |
| `Actor.HurtVolumeMiss` | actors/actor.zs | `IFVIRTUALPTR` playsim/p_map.cpp:623 | ALIVE |
| `Actor.PuffSplash/PuffHit/PuffThrough` (virtual native) | actor.zs | `IFOVERRIDENVIRTUALPTRNAME` p_map.cpp:6162, 6174, 6226, 6286, 7798 | ALIVE (only when overridden) |
| `PlayerPawn.PickWeapon/PickPrevWeapon/PickNextWeapon/MoveWeaponToHand/GetVRWheelInfo/SwitchWeaponHand/FireWeapon(Alt)` (hand parameter) | actors/player/player.zs | g_game.cpp:696, 871; d_net.cpp:2674, 2684, 3382; hw_vrwheel.cpp:262, 1065 | ALIVE |
| `Thinker.Sleep/ShouldWake/Wake/SleepIndefinite` | doombase.zs | dthinker.cpp (natives + `RunSleepCycle`) | ALIVE |
| `DataValidator.Validate` | zscript/engine/datavalidator.zs | `D_RunDataValidators` (d_main.cpp:4647) | DORMANT (`-norun -validatedata`) |
| `Actor.OnSweepCrossed` | actors/actor.zs:937 | **none** (V: `grep -rnw OnSweepCrossed src` empty; comment at actor.zs:924 says the engine never calls it) | DEAD as an engine hook; a mod-to-mod contract only |
| `RS_ChaseBrain.Decide`, `RS_BoutRules.*`, `RS_BoutRingBase.*` | rs_chasebrain.zs, rs_bout.zs, rs_boutring.zs | script-only (called from those files and `Actor.ChaseBrain`, actor.zs:596) | DORMANT unless a mod sets `ChaseBrain` / registers `RS_BoutDirector` |
| `EventHandler.SendNetworkEvent` now returns `bool` | events.zs:249-275 | stock native already returned it | ALIVE (signature change) |

Event handlers registered by the engine for every game: `OtherPlayerTagHandler`, `VRStabilizeSyncHandler` (`wadsrc/static/mapinfo/common.txt:8`) — ALIVE every tic (§3.2). `RS_BoutDirector` (`rs_bout.zs:227`) — DORMANT, opt-in by a mod's GAMEINFO.

### 8.3 Dead / unreachable paths

| What | Evidence (V) | Status |
|---|---|---|
| Teleport command burst | writer `VR_QueueTeleportCommandBurst` (qzdoom_common.cpp:84) never called; `VR_ClearTeleportCommandBurst` only zeroes; reader `VR_ConsumeTeleportCommandStep` (:219) returns false at length ≤ 0.001 (:229), so `VR_ApplyTeleportBurstMove` (g_game.cpp:141) always returns at :146 | DEAD path (called per command build, no effect) |
| `COMPATF2_OLD_RANDOM_GENERATOR` / `compat_oldrandom` | bit defined doomdef.h:244, set in compat presets d_main.cpp:918, 928, cvar d_main.cpp:1024; no `& COMPATF2_OLD_RANDOM_GENERATOR` test anywhere | DEAD flag (not in stock 5.0.0) |
| `gl_lights` (stock `DEPR_CVAR`) | fork changed the default true→false (g_cvars.cpp:63; stock true); only read in a debug printout hw_clock.cpp:353 | DEAD in both (deprecated) |
| `gl_weapon_purelightlevel` (stock cvar) | read twice in stock (`hw_weapon.cpp:691, 773` of 5.0.0), zero reads in main | DEAD — made dead by the fork |
| `r_renderthread` (renderer branch) | `CUSTOM_CVAR` that prints "not built yet" and resets itself (renderer `rendering/r_snapshot.cpp:33-40`); value never read | DEAD stub (renderer only) |
| OpenVR mode | `VR_OPENVR` returns the GL `OpenVRMode` (hw_vrmodes.cpp:1283-1291); `vrmi_openvr` objects are in `#if 0 //def USE_OPENVR` (:1218-1220, :1230-1232) | DORMANT (GL backend only) |
| `__WIN32__` blocks in `common/engine/i_net.cpp:172, 202, 234` | MSVC predefines `_WIN32`, not `__WIN32__` (I: MSVC documentation; no `#define __WIN32__` in the tree, V grep) — the extra `WSAStartup/WSACleanup` in the local-address lookup never compile on the Windows MSVC build | DORMANT on MSVC (harmless: winsock is already started) |

### 8.4 CVARs: summary

- Stock 5.0.0: 958 cvar names. `main`: 1,580. Fork-added: 623 (302 P, 321 R). Removed: `vr_hunits_per_meter`.
- Stock cvars in main with no run-time reader: 26 by the scanner, of which almost all were already so in stock (userinfo cvars read through the userinfo system, `CUSTOM` callbacks, platform-specific); exactly one changed state because of the fork: `gl_weapon_purelightlevel` (§8.3).
- Fork cvars by status (V, scanner + hand checks):
  - **DEAD (6)**: `compat_oldrandom`, `r_transparentPlayerSprites` (hw_weapon.cpp:124, no other reference), `vr_missile_haptic_level` (hw_vrmodes.cpp:1085; only `EXTERN_CVAR` p_mobj.cpp:100), `vr_openxr_eye_shift_scale` (hw_vrmodes.cpp:639; `EXTERN` vk_openxrdevice.cpp:110 + menudef.txt:2235), `vr_openxr_force_recommended_viewport` (:646; menudef.txt:2236 only), `vr_openxr_use_screen_viewport_for_submit` (:652; menudef.txt:2237 only). The last three are exposed in the developer menu and change nothing.
  - **DORMANT, GL/OpenVR code only (6)**: `vr_two_handed_weapons`, `vr_openvr_menu_pointer_tip_offset`, `vr_openvr_menu_pointer_pitch_bias` (read only in `rendering/gl/stereo3d/gl_openvr.cpp`), `gl_no_clip_planes`, `gl_no_ssbo`, `gl_no_persistent_buffer` (read only in `common/rendering/gl_load/gl_interface.cpp`).
  - **Callback-only (1)**: `r_smoke_preset` (hw_postprocess_cvars.cpp:287) — value never read; its body sets `r_smoke`, `r_smoke_quality`, `r_smoke_steps`, `r_smoke_light_quality`, `r_smoke_temporal`.
  - **Read only at startup / object creation (7, by reading function)**: `gl_texture_thread_workers`, `gl_texture_thread_upload`, `vk_max_transfer_threads`, `vk_robust_buffer_access` (all in `VulkanRenderDevice::VulkanRenderDevice`), `vk_gpu_checkpoints` (ctor :306 + first `GpuCheckpoint`), `vr_eye_colorspace` (`VKOpenXRDeviceMode::CreateSwapchain`), `r_gpuparticles_ringsize` (its description: "takes effect on restart").
  - `ext_haptic_level_*` (8, p_actionfunctions.cpp:81-88) look unread by name but are ALIVE: read through a runtime-built name `"ext_haptic_level_%s"` in `C_GetExternalHapticLevelValue` (common/console/c_cvars.cpp:1440-1450) for `damage_projectile, fire_weapon, healstation, heartbeat, pickup, pickup_weapon, poison, rumble` (V: call sites grep).
  - Debug/diagnostic cvars: 84 fork cvars whose names contain `debug`, `diag`, `_test` or `fake` (72 of them end in `_debug/_diag/_test/_fake*` and are in R) (V: name count on the table). They are read (ALIVE) but only switch prints/tests/fake poses; candidates for removal in a rebuild (I: by name and the reader functions listed in the table).
- Full table: §8.9.

### 8.5 Console commands and stats added

76 fork `CCMD`s (V, set difference vs stock): affect_reload affect_test bb_clear bb_neon bb_spawn bb_text bloomlook cinemamode damagebrushes disablerendercull emissivevolumes fade_debug fade_density_down fade_density_up fade_gradient_down fade_gradient_up fade_toggle gl_bloom_pin_capture gl_bloom_pin_reset idbehold idchoppers idclip iddt idfa idkfa modelsurfaces mp_launch_host mp_launch_join mp_reset_defaults netplay_load netplay_ready netplay_resync particles perf_track_flush perflog_label qzd_restart rig_eyeshot rig_handreport rs_remap_dump screenshot_in stamp surface surfaces switchhand toggle_laser_sight togglecheatmenu togglefps toggleportablehud usercmd_selftest vr_avatar_dump vr_avatar_handscale vr_avatar_setheight vr_avatars vr_body_source vr_body_tunables vr_bodyinfer_fit vr_bodyinfer_mark vr_bodyinfer_neckfit vr_bodyinfer_record vr_bodyinfer_replay vr_bodyinfer_synth vr_bodyinfer_take vr_debug_roommove vr_gear_play vr_gear_selftest vr_grab_selftest vr_gun_dump vr_gunfit vr_gunfit_check vr_handpose vr_haptictest vr_rig_selftest vr_slime_reset vr_slime_status vr_stand_eye_measure worldclock. (`id*` are console commands that send the classic cheats over the network, `console/c_cmds.cpp:1277-1345`, used by the fork's cheat menu.) Stats: `ADD_STAT(vrstats)`, `bufferstats`, `remotestats`. Clocks: +31 `glcycle_t` in `common/rendering/hwrenderer/data/hw_clock.cpp:55-63` (`VRSceneEyes, VRSceneBuild, VRSubsectors, VRSubsectorCull, VRSubsectorVisible, VRLineBuild, VRLineClip, VRLineDecide, VRThingBuild, VRFlatBuild, VRScenePostBSP, VRPlayerSprites, VRSceneDraw, VREyeComposite, VRFinalizeEye, VRSubmit, VRPostProcessScene, VRSceneTransfer, VRFinalPresent, VRSubmitCopy, VRSubmitWait, VRRenderSyncWait, WTWallJobs, WTFlatJobs, WTThingJobs, WallWorkersElapsed, WallMerge, SceneWorkerElapsed`, …). perf_track columns include `frame_ms, cost_ms, pacing_ms, think_ms, csthink_ms, xr.wait, xr.acquire, xr.imagewait, xr.end, xr.app_gpu, xr.comp_gpu, xr.comp_cpu` (hw_perftrack.cpp:730-770); r_perflog GPU groups `fx.depthread, fx.drawnlines, fx.emissive, fx.gpuparticles, fx.heatsources, fx.meshparticles, fx.shockwaves, fx.smokedraw, fx.viewlights, pp.customdepth, pp.emissive, pp.exposureimpulse, pp.heatoffset, pp.heatwarp, pp.lightmaskcarry, pp.lightmaskdebug, pp.scenemask, pp.shockwave, pp.smoke, pp.smokebeams, scene.opaque, scene.portals, scene.psprites, scene.translucent, scene.vrextras, scene.worldpsprites` and CPU sample `fx.compute` (vk_compute.cpp:108). Scene groups exist only while `r_perflog>0` (`perfGroups`, hw_drawinfo.cpp:4499-4578).

### 8.6 ZScript natives the fork added

459 native functions + 161 fields/globals (V). Native→declaration check: every one of the 459 has a `native` declaration line in `wadsrc/static/zscript` matching its name (name-level check). In-tree callers (V, then hand-disambiguated): `AActor.StartSound` (rs_bout.zs:748), `AffectResistOf` (actor.zs:1964-1966), `TexMan.SetCanvasTextureTranslucent` (otherplayertags.zs), `SetTimeScale/GetTimeScale/IsWorldSlowed`, `ClockFor` (rs_bout.zs, player.zs:1709, 1736), `SetBeam/ClaimBeam/ReleaseBeam/IsBeamClaimed/SetBeamStyle` (rs_boutring.zs), `StrikeTicsLeft`, `IncomingStrike` (rs_chasebrain.zs), `SuppressVRInput`, `GetRawStickMove/Turn` (rs_bout.zs), `VRGunFit.Size` (rs_boutring.zs), `QueryFloorRegion`, `FloorRegionOutlineCount/Point`, `FloorRegionRelease` (rs_boutring.zs), the four menu `DoCommand`/chat natives (engine menus), `SetModelSurfaceOffset` (actor.zs:2669), `ClearModelSurfaces` (player.zs:3330), `AttackDir`/`OffhandDir` (actions.zs:106-114). Every other fork native (~425) has no caller inside the engine pk3; whether the external RS/BD22 mods call them cannot be checked from these trees. They cost nothing unless called (registration only). Grouped list (bold = in-tree caller; `^P` = present at the pristine import):

- `FLevelLocals` (230; impl in scripting/vmthunks.cpp): AddBillboard^P, AddBillboardGroup^P, AddBillboardPersistent^P, AddShape^P, AimBillboard^P, AnimateBillboardGroup^P, AttachBillboard^P, BeamOutlineAlive, BeamOutlineBudget, BeamOutlineDegraded, BeamOutlineDrop, BeamOutlineLight, BeamOutlineMat, BeamOutlineOpen, BeamOutlinePhase, BeamOutlinePoint, BeamOutlinePosts, BeamOutlineRaise, BeamOutlineRelease, BeamOutlineSlots, BeamOutlineStanding, BeamOutlineStrandHeights, BeamOutlineStyle, BeamOutlineStyleScroll, BeamOutlineTiming, BeamWakeDraws, BillboardFontCount^P, BillboardFontName^P, CarveSmoke, **ClaimBeam**, ClearBeamStyle, ClearBeams^P, ClearBloomLook, ClearBloomOverride, ClearDarkness^P, ClearDrawnLine, ClearDrawnLines, ClearFogDisturb^P, ClearFogGradientOverride, ClearFogIgniteColor, ClearFogSlab^P, ClearFogSlabOverride, ClearGlowWave^P, ClearGpuParticles, ClearHeatSource, ClearShapes^P, ClearSmoke, ClearSurfaceStamps^P, ClearSweep^P, ClearVolumetricBeam^P, **ClockFor**, CollectMissiles, CollectStrikes, CountMissiles, CountStrikes, DeclareHandStrike, DeclareStrike, DrawnLineCapacity, EmissiveVolumeDefinition, EmissiveVolumeEnabled, EmitSmoke, ExposureImpulse, FieldAt^P, FieldCount^P, **FloorRegionOutlineCount**, **FloorRegionOutlinePoint**, **FloorRegionRelease**, FogDisturb^P, ForceVRLaser^P, ForgetStrike, ForgetStrikesFrom, FreeBeamSlots, GetActorModelClass^P, GetFieldBool^P, GetFieldFloat^P, GetFieldInt^P, GetFieldIntArray^P, GetFieldName^P, GetFieldObject^P, GetFieldString^P, GetGuardPlate, GetModelBoundsHint^P, GetModelOffsetHint^P, GetModelOrientationHint^P, GetModelWorldOffset^P, **GetRawStickMove**^P, **GetRawStickTurn**, GetStrike, **GetTimeScale**, HandPeakAgeMs, HandPos, HandVelAtPoint, HasField^P, HearingImpulse, HeatSourceCapacity, HeatmapAdd^P, HeatmapAt^P, HeatmapClear^P, **IsBeamClaimed**, IsVRInputSuppressed^P, **IsWorldSlowed**, JSONProfileBegin^P, JSONProfileGetDouble^P, JSONProfileLoad^P, JSONProfileSave^P, JSONProfileSetDouble^P, LinkShape^P, MeasureBillboardText^P, MeasureBillboardTextBlock^P, MoveBillboard^P, MoveShape^P, OrientBillboard^P, PaintSurfaceDamage, ParticleDefinition, PushEffectImpulse, **QueryFloorRegion**, RealSeconds, **ReleaseBeam**, RemoveBillboard^P, RemoveBillboardGroup^P, RemoveShape^P, ResizeBillboard^P, RollBillboard^P, RollBillboardFonts^P, **SetBeam**^P, SetBeamAnchor, SetBeamCount^P, SetBeamLight, SetBeamLook^P, SetBeamOnModel, **SetBeamStyle**, SetBeamStyleScroll, SetBeamWakeFrom, SetBillboardAlpha^P, SetBillboardFont^P, SetBillboardGlow^P, SetBillboardGradient^P, SetBillboardGroup^P, SetBillboardGroupOrigin^P, SetBillboardGroupScale^P, SetBillboardProgress^P, SetBillboardStereoRight, SetBillboardText^P, SetBloomLook, SetBloomOverride, SetDarkness^P, SetDarknessActors, SetDarknessHeightFollow, SetDarknessSpace^P, SetDesatGlobal^P, SetDesatKeep^P, SetDrawnLine, SetDrawnLineAnchor, SetDrawnLineGradient, SetDrawnLineLight, SetDrawnLineLook, SetDrawnLineTurbulence, SetFogBottom^P, SetFogBow^P, SetFogDynLights, SetFogFollow^P, SetFogGradient^P, SetFogGradientOverride, SetFogIgniteColor, SetFogNoise^P, SetFogPickup^P, SetFogSlab^P, SetFogSlabOverride, SetFogSurface^P, SetFogTendrils^P, SetFogWake^P, SetFogWakeMotion^P, SetFogWakePos, SetFogWakeShape, SetFogZones, SetGlowCells^P, SetGlowFlow^P, SetGlowReact^P, SetGlowTexture^P, SetGlowWave^P, SetGlowWaveDepth^P, SetGlowWaveOrigin^P, SetGlowWavePhase^P, SetHeatSource, SetHeatSourceAnchor, SetHeatmap^P, SetShapeLook^P, SetShapeMotion^P, SetShapeOrient^P, SetShapeRepeat^P, SetSmokeLook, SetSmokeWind, SetSweepBand^P, SetSweepBandAt^P, SetSweepBandDraw^P, SetSweepBandFill^P, SetSweepBandPassed, SetSweepCount^P, SetSweepFill^P, SetSweepFillAir^P, SetSweepFillMotion^P, SetSweepOrigin^P, SetSweepPassedLook, SetSweepRoom^P, SetSweepTrail^P, **SetTimeScale**, SetTornado^P, SetTornadoLook^P, SetTornadoMotion^P, SetTrackedPoseSmoothing, SetVRLaserRange^P, SetVolumetricBeam^P, SetVolumetricBeamAnchor, SpawnEffectLight, SpawnEmissiveVolume, SpawnGpuParticles, SpawnParticles, SpawnShockwave, SpawnSurfaceStamp^P, **StrikeTicsLeft**, **SuppressVRInput**^P, SweepBillboard^P, SweepDrawnShapes, TouchBillboard^P, UpdateBillboard^P, VRHaptic^P, VRHapticEx, VRHapticFreq, VRHapticHold, WorldSeconds
- `AActor` (91; impl in playsim/a_dynlight.cpp, playsim/p_actionfunctions.cpp, playsim/p_enemy.cpp, playsim/p_mobj.cpp, r_data/model_jointdrive.cpp, r_data/models.cpp, scripting/vmthunks_actors.cpp): A_SetUserVarName^P, AffectDownTics, AffectHit, AffectHold, AffectHoldTics, AffectHoldsAI, AffectInMotionFromHit, AffectKnockTics, AffectMeter, AffectPush, **AffectResistOf**, AffectStunTics, AffectThrownTics, AttackChainTics, **AttackDir**^P, CalculateMissileDamage, CaptureHandHold, ClearHurtVolumes, ClearModelJointDrawPose, ClearModelJointDrive, ClearModelJointOffset, ClearModelStateFrames^P, ClearModelSurfaceDrive, ClearModelSurfaceOffset, **ClearModelSurfaces**, CountStateLabels^P, Deflect, DeflectedTics, FindBoneIndex^P, FindModelJointIndex, FindModelSurfaceIndex, FollowHandLocal, FreezeFollowHand, GetClassModelFile, GetClassModelJointCount, GetClassModelJointName, GetClassModelSurfaceCount, GetClassModelSurfaceName, GetDrawnShape, GetModelFrameCount, GetModelJointCount, GetModelJointDrawnValue, GetModelJointName, GetModelPlacementPrefix, GetModelSurfaceCount, GetModelSurfaceDrawnValue, GetModelSurfaceName, GetSpriteTextureID^P, GetStateLabelAt^P, HasHandHold, HasModelFrame, HasVoxelFrame, HurtVolumeCount, HurtVolumeEnd, HurtVolumeNearest, HurtVolumeRadius, HurtVolumeStart, HurtVolumeZone, HurtVolumesGovern, **IncomingStrike**, ModelFollowFrameToWorld, ModelPointToWorld^P, **OffhandDir**^P, PlaysimHeadPos, PlaysimHeadSource, PuffHit, PuffSplash, PuffThrough, RegisterModelStateFrame^P, ReleaseHandHold, SetAttachedLightAnchor, SetHurtVolume, SetModelJointDrawOffset, SetModelJointDrawPose, SetModelJointDrive, SetModelJointDriveHinge, SetModelJointDriveRotation, SetModelJointDriveStage, SetModelJointOffset, SetModelSurfaceBlend, SetModelSurfaceDrive, SetModelSurfaceDriveHinge, SetModelSurfaceDriveRotation, SetModelSurfaceDriveStage, SetModelSurfaceHidden, **SetModelSurfaceOffset**, SetModelSurfacePos, **StartSound**, SweepDrawnShape, SweepDrawnShapeSimple, WorldToModelPoint
- `_VRGear` (46; impl in scripting/vmthunks.cpp): Assigned, BodyScale, Content, DrawClass, DrawTic, EmptyHandAt, EmptyHandCount, Event, EventSeq, FeelLength, FeelPulse, FiledAs, FindSpot, HandAt, HandBusy, HandInSpot, HandKind, HandTics, IsEmptyHand, MainLeft, OwnHome, PopAction, Post, PrevWeapon, RoomMetres, Seated, SetAssigned, SetContent, SetDraw, SetPrevWeapon, SetSpotBits, SoundOf, SpotCount, SpotHang, SpotHoldAmmo, SpotHoldCount, SpotHoldWeapon, SpotHolds, SpotLength, SpotLive, SpotName, SpotPose, SpotRadius, SpotRule, SpotShow, SpotType
- `_Sector` (18; impl in scripting/vmthunks.cpp): GetFlatGlowColor^P, GetFlatGlowHeight^P, GetGlowFalloff^P, GetGlowIntensity^P, GetLightTrimBase, GetLightTrimDim, GetLightTrimOffset, SetFlatGlowColor^P, SetFlatGlowColorFar^P, SetFlatGlowFalloff^P, SetFlatGlowHeight^P, SetFlatGlowInset, SetFlatGlowIntensity^P, SetGlowColorFar^P, SetGlowFalloff^P, SetGlowInset, SetGlowIntensity^P, SetLightTrim
- `_VRAvatarTable` (18; impl in scripting/vmthunks.cpp): ActorName, BodySocketKey, BodyYaw, Count, EyeAboveFloor, GetBodyData, GetGunData, GetWeaponFit, GunMeshKey, Measured, NameAt, RigHandPos, RigHandValid, SetBodyData, SetGunData, SetWeaponFit, SocketInForce, SocketWorld
- `_VRGrab` (10; impl in scripting/vmthunks.cpp): Abort, Event, EventSeq, HandState, HandVel, Mass, Palm, Policy, PopAction, Tune
- `_VRTorso` (8; impl in scripting/vmthunks.cpp): Anchor, Freeze, Get, RecenterCount, SeatedEye, StandEye, Valid, Yaw
- `_PlayerInfo` (7; impl in playsim/p_guard.cpp, playsim/p_vrhand.cpp): ClearGuardVolume, GetGuardBlock, GetVRHand, GetVRHandInput, GetVRHead, SetGuardVolume, VRHandTic
- `DObject` (4; impl in sound/s_doomsound.cpp): S_GetWorldPitch, S_SetWorldPitch, S_SoundPitch, S_StopSound
- `FSoundHandleStruct` (4; impl in sound/s_doomsound.cpp): IsPlaying, SetPitch, SetVolume, StopSound
- `_Side` (4; impl in scripting/vmthunks.cpp): GetGlowColor^P, GetGlowHeight^P, SetGlowColor^P, SetGlowHeight^P
- `DThinker` (4; impl in playsim/dthinker.cpp): ShouldWake, Sleep, SleepIndefinite, Wake
- `_VRGunFit` (3; impl in scripting/vmthunks.cpp): **Size**, Sized, Socket
- `DChatTextEnterMenu` (3; impl in common/scripting/interface/vmnatives.cpp): **CancelChatMessage**^P, **IsTeamChat**^P, **SubmitChatMessage**^P
- `_DataValidation` (2; impl in gamedata/datavalidation.cpp): Refuse, Running
- `_TexMan` (2; impl in common/scripting/interface/vmnatives.cpp, scripting/vmthunks.cpp): GetSurface, **SetCanvasTextureTranslucent**^P
- `DPSprite` (2; impl in playsim/p_pspr.cpp): ClearModelSurfaceOffset, **SetModelSurfaceOffset**
- `DOptionMenuItemCommandInput` (1; impl in common/scripting/interface/vmnatives.cpp): **DoCommand**^P
- `DConsoleTextEnterMenu` (1; impl in common/scripting/interface/vmnatives.cpp): **DoCommand**^P
- `DCheatMenu` (1; impl in common/scripting/interface/vmnatives.cpp): **DoCommand**^P

### 8.7 Shaders: registered vs bound

Every fork shader file is referenced from C++ (V: `grep -F <file> src` for each of the 45 fork shader files; the only shader files with no C++ reference are include-only libraries `beamglow.glsl`, `valuenoise.glsl` and the stock `gamma*.fp`, `func_warp3.fp`, `func_wavex.fp`). Binding:

| Kind | Fork entries | Bound at (V) | Notes |
|---|---|---|---|
| Material shaders (`defaultshaders[]`, `common/rendering/hwrenderer/data/hw_shaderpatcher.cpp:298-327`, enum `common/textures/textures.h:78-87`) | `SDF Text`, `Segment`, `Seam`, `WG13`, `SDF Panel`, `SDF Hex`, `SDF Star`, `Neon` (all P) | `OverrideShader = SHADER_*` in `rendering/hwrenderer/scene/hw_sprites.cpp:2632-2841` (billboard payloads) | compiled for every pass type at startup (`hw_shaderpatcher.cpp:374` counts them into the total); bound only when a billboard of that payload draws |
| Effect shaders (`effectshaders[]`, hw_shaderpatcher.cpp:345-365; enum `hw_renderstate.h:41-68`) | `gpuparticles`, `drawnlines`, `meshparticles`, `eyefade`, `portalair` (R) | `SetEffect(EFF_GPUPARTICLES)` hw_drawinfo.cpp:3936; `EFF_DRAWNLINES` :4012; `EFF_MESHPARTICLES` hw_debrispool.cpp:1295, hw_meshparticles.cpp:499; `EFF_EYEFADE` hw_models.cpp:89; `EFF_PORTALAIR` hw_portal.cpp:482, 544 | GL/GLES skip the first three (`gl_shader.cpp:1235`, `gles_shader.cpp:857`) |
| Post-process `PPShader` (`hw_postprocess.h`) | 92 `PPShader` members in the file; all fork ones (volumetricbeam, heatmap, heatoffset/heatwarp/shockwave, smoke depth/march/blur/composite/temporal, emissivevolume, screentilemask, sharedmarchwarp, customdepth, scenemask, lightmaskdebug, tonemapfilmic, exposureimpulse; `present.Spectator`) | each referenced in `hw_postprocess.cpp` (V script: 0 unreferenced except `PPPresent` members, which are bound in `vulkan/renderer/vk_postprocess.cpp`, e.g. `present.Spectator` :540) | reachability per pass = §3.1 gates |
| Compute (`shaders/compute/*.comp`) | 9 files | `VkComputeManager::CreateProgram` by lump name: vk_smokevolume.cpp:573-588, 1473; vk_levelfield.cpp:448; vk_debrispool.cpp:352; vk_surfacedamage.cpp:520-535 | built lazily when the effect first becomes active (vk_compute.cpp:65-100) |
| Always-prepended libraries | `func_surfacestamps.fp`, `func_spriteoutline.fp` | vk_shader.cpp:1189-1192 (every material program), gl_shader.cpp:684-693 | not optional: always compiled in |

Conclusion (V): no fork shader is registered and never bound. The cost item is the opposite one: code compiled into every program (§7.1).

### 8.8 Code that never compiles in the Windows Vulkan build

Defines on the owner's build (V: `CMakeLists.txt:308-330`, `src/CMakeLists.txt:394-512`, `auto-setup-windows-vr.cmd:98-104`): `HAVE_VULKAN`, `HAVE_GLES2` (non-Apple default ON), `USE_OPENXR`+`DYN_OPENXR`, `USE_OPENVR`+`DYN_OPENVR` (if the owner's `build-dxr` was configured by that script; I), MSVC `_WIN32`; never `__MOBILE__`, `__ANDROID__`, `__WIN32__`.

- Files present but never compiled (V: not listed in `src/CMakeLists.txt`, not `#include`d): `posix/nosdl/` (9 files: `crashcatcher.c, glvideo.cpp/.h, gl_sysfb.h, hardware.cpp, i_gui.cpp, i_input.cpp, i_joystick.cpp, i_system.cpp, st_start.cpp`), `gl/stereo3d/gl_openxrdevice.cpp/.h`, `gamedata/xlat/xlat_parser.c/.h/.out` (stale generated copies; CMake regenerates `xlat_parser.c` into the build dir, `src/CMakeLists.txt:632-634`), `win32/QZDoomVR.bat`, `openvr_include.h` (its only include is commented out, `gl_openvr.cpp:70`).
- `#if 0` in fork code: 4 blocks only — `rendering/gl/stereo3d/gl_openvr.cpp:2357-2362, 3413-3424`; `hw_vrmodes.cpp:1218-1220, 1230-1232` (`#if 0 //def USE_OPENVR`). R-layer code has none (V: script over all `#if 0/#if false` blocks, diffed against stock).
- QuestZDoom `__MOBILE__`/`__ANDROID__` blocks added by the P layer in compiled files: 42 blocks, 261 lines, in 22 files (V, same script), mostly GL backend (`gl_renderbuffers.cpp:315-512`, `gl_hwtexture.cpp`, `gl_shaderprogram.cpp`, `gl_load/gl_load.c:61-126`), `oalsound.cpp:1175-1756`, `playsim/p_interaction.cpp:1482-1542` (59 lines), posix SDL platform files.
- `__WIN32__` blocks in `common/engine/i_net.cpp:172-174, 202-204, 234-236` (§8.3).
- Constant-false conditions added by the fork: none besides the above (V: `if (0)`, `if (false)`, `&& false` scan vs stock).
- GL/GLES backends: compiled (stock), but the engine runs Vulkan (`vid_preferbackend 1`); fork additions inside GL files (OpenVR, `__MOBILE__` blocks) are DORMANT on this configuration.

### 8.9 Full table: the 623 fork cvars (main)

Columns: name | layer (P = at pristine import, R = later) | definition | type and default | flags (A archive, G globalconfig, N nosave, S serverinfo, U userinfo, I noinitcall, C cheat, L latch) | reads `src/string-lookup/zscript/menudef` (V, counts from the scanner; strings = `"name"` lookups in C++) | first reading functions (heuristic enclosing-function name) | status. Sorted by defining file.

| cvar | L | definition | type default | flags | reads s/str/zs/menu | read in | status |
|---|---|---|---|---|---|---|---|
| vr_haptic_event_scale | R | src/QzDoom/qzdoom_common.cpp:338 | Float 1.0f | AG | 1/0/0/0 | VR_HapticEvent | ALIVE |
| vr_input_in_command | R | src/QzDoom/qzdoom_common.cpp:144 | Bool false | N | 1/0/0/0 | VR_InputViaCommand | ALIVE |
| cl_custombinds | P | src/common/console/c_bind.cpp:44 | Bool false | AG | 1/0/0/2 | C_SetDefaultKeys | ALIVE |
| cl_custombinds_override | P | src/common/console/c_bind.cpp:45 | Bool false | AG | 1/0/0/1 | C_SetDefaultKeys | ALIVE |
| net_latejoin | R | src/common/engine/i_net.cpp:402 | Bool true | NS | 3/0/0/0 | GetPacket, HostGame | ALIVE |
| net_latejoin_debug | R | src/common/engine/i_net.cpp:403 | Bool false | N | 7/0/0/0 | GetPacket, HGetPacket | ALIVE |
| vr_slime_debug | R | src/common/engine/i_solarxr.cpp:49 | Bool false | - | 1/0/0/0 | SolarXR_Tick | ALIVE |
| vr_slime_enable | R | src/common/engine/i_solarxr.cpp:47 | Bool false | A | 3/0/0/0 | PrintSlimeStatus, SolarXR_Tick | ALIVE |
| vr_slime_port | R | src/common/engine/i_solarxr.cpp:50 | Int 21110 | A | 2/0/0/0 | SolarXR_Tick | ALIVE |
| net_upnp | R | src/common/engine/i_upnp.cpp:39 | Bool true | AG | 1/0/0/0 | I_UPnPMapHostPort | ALIVE |
| gl_lite_shader | P | src/common/rendering/gl/gl_shader.cpp:45 | Bool false | AG | 2/0/0/2 | FShaderCollection::Compile, getRealLightmode | ALIVE |
| debug_precache_actor | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:247 | Bool false | N | 4/0/0/0 | PrecacheLevel, hw_PrecacheTexture | ALIVE |
| gl_background_flush_count | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:93 | Int 100 | AG | 1/0/0/2 | VulkanRenderDevice::UploadLoadedTextures | ALIVE |
| gl_global_fade | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:255 | Bool false | A | 4/0/0/4 | CCMD, PPAmbientOcclusion::Render | ALIVE |
| gl_global_fade_color | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:267 | Color 0x3f3f3f | A | 1/0/0/4 |  | ALIVE |
| gl_global_fade_debug | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:268 | Bool false | - | 2/0/0/0 | CCMD | ALIVE |
| gl_global_fade_density | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:257 | Float 0.001f | A | 4/0/0/4 | CCMD, PPAmbientOcclusion::Render | ALIVE |
| gl_global_fade_gradient | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:262 | Float 1.5f | A | 4/0/0/4 | CCMD, PPAmbientOcclusion::Render | ALIVE |
| gl_mirror_player | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:104 | Bool true | AG | 2/0/0/2 | HWMirrorPortal::Setup, HWPlaneMirrorPortal::Setup | ALIVE |
| gl_no_clip_planes | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:276 | Bool false | - | 1/0/0/0 | gl_LoadExtensions | DORMANT (GL/OpenVR code only) |
| gl_no_persistent_buffer | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:275 | Bool false | - | 1/0/0/0 | gl_LoadExtensions | DORMANT (GL/OpenVR code only) |
| gl_no_ssbo | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:277 | Bool false | - | 1/0/0/2 | gl_LoadExtensions | DORMANT (GL/OpenVR code only) |
| gl_precache_actors | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:246 | Bool true | A | 1/0/0/0 | PrecacheLevel | ALIVE |
| gl_storage_buffer_type | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:270 | Int 1 | AGI | 1/0/0/3 | EXTERN_CVAR | ALIVE |
| gl_texture_quality | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:224 | Int 0 | AGI | 1/0/0/2 | VkHardwareTexture::CreateCompressedTexture | ALIVE |
| gl_texture_thread | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:74 | Bool true | AG | 11/0/0/3 | HUDSprite::GetWeaponRect, HWSprite::Process | ALIVE |
| gl_texture_thread_models | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:75 | Bool true | AG | 4/0/0/2 | HWSprite::Process, VulkanRenderDevice::BackgroundLoadModel | ALIVE |
| gl_texture_thread_upload | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:76 | Bool true | AG | 2/0/0/2 | VulkanRenderDevice::VulkanRenderDevice | ALIVE, read at startup/creation only |
| gl_texture_thread_workers | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:79 | Int 2 | AG | 1/0/0/2 | VulkanRenderDevice::VulkanRenderDevice | ALIVE, read at startup/creation only |
| r_beams_debug | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:480 | Bool false | - | 1/0/0/0 | ReportBeamLines | ALIVE |
| r_beams_drawn | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:462 | Bool false | GI | 11/0/0/1 | BeamsRouteToDrawnLines, CUSTOM_CVARD | ALIVE |
| r_beams_drawn_surfacelight | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:469 | Bool true | AG | 2/0/0/0 | DrawnLinesLogToggle, HWDrawInfo::StartScene | ALIVE |
| r_drawnlines | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:472 | Bool true | AG | 4/0/0/1 | BeamWakesCanDraw, BeamsRouteToDrawnLines | ALIVE |
| r_drawnlines_depthbias | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:477 | Float -1.0f | AG | 1/0/0/0 | SyncDrawnLines | ALIVE |
| r_gpuparticles | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:291 | Bool true | AG | 10/0/0/0 | CUSTOM_CVARD, DebrisPool::PrepareFrame | ALIVE |
| r_gpuparticles_atlas_compressed | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:380 | Bool true | AG | 2/0/0/1 | GpuParticleAtlasPolicy, static_assert | ALIVE |
| r_gpuparticles_atlas_compressed_size | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:381 | Int 512 | AG | 1/0/0/1 | GpuParticleAtlasCompressedSide | ALIVE |
| r_gpuparticles_atlas_layers | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:382 | Int 2048 | AG | 1/0/0/1 | GpuParticleAtlasLayersPerAtlas | ALIVE |
| r_gpuparticles_atlas_overflow | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:383 | Int 1 | AG | 1/0/0/1 | GpuParticleAtlasPolicy | ALIVE |
| r_gpuparticles_atlas_size | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:352 | Int 256 | AG | 2/0/0/1 | GpuParticleAtlasLayerSize, VkTextureManager::CreateParticleAtlas | ALIVE |
| r_gpuparticles_cull | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:429 | Bool true | G | 2/0/0/0 | CUSTOM_CVARD, HWDrawInfo::StartScene | ALIVE |
| r_gpuparticles_debug | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:318 | Bool false | - | 1/0/0/0 | GpuParticleBuffer::DebugReport | ALIVE |
| r_gpuparticles_intensity | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:299 | Float 1.0f | AG | 1/0/0/0 | HWDrawInfo::StartScene | ALIVE |
| r_gpuparticles_legacy | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:334 | Bool false | G | 3/1/0/1 | CCMD, CUSTOM_CVARD | ALIVE |
| r_gpuparticles_light_lod_distance | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:434 | Float 0.f | AG | 2/0/0/1 | CUSTOM_CVARD, HWDrawInfo::StartScene | ALIVE |
| r_gpuparticles_lights | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:419 | Int 32 | AG | 2/0/0/1 | CUSTOM_CVARD, SyncViewLights | ALIVE |
| r_gpuparticles_maxsize | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:297 | Float 64.0f | AG | 2/0/0/1 | GpuParticleBuffer::SpawnRecordLights, HWDrawInfo::StartScene | ALIVE |
| r_gpuparticles_ringsize | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:312 | Int 65536 | AGI | 1/0/0/0 | GpuParticleRingCapacity | ALIVE, read at startup/creation only |
| r_gpuparticles_sizescale | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:292 | Float 1.0f | AG | 5/0/0/0 | DebrisPool::ExpandBurst, DebrisPool::PrepareFrame | ALIVE |
| r_gpuparticles_soft | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:311 | Float 8.0f | AG | 4/0/0/1 | CUSTOM_CVARD, HWDrawInfo::RenderTranslucent | ALIVE |
| r_gpuparticles_stretch | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:298 | Float 1.0f | AG | 3/0/0/0 | EXTERN_CVAR, GpuParticleBuffer::SpawnRecordLights | ALIVE |
| r_heatrefraction | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:511 | Bool true | AG | 4/0/0/1 | CUSTOM_CVARD, PPHeatRefraction::Render | ALIVE |
| r_heatrefraction_scale | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:515 | Float 1.0f | AG | 1/0/0/1 | SetupHeatSources | ALIVE |
| r_heatrefraction_test | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:520 | Bool false | G | 3/0/0/1 | SetupHeatSources, static_assert | ALIVE |
| r_portalair | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:486 | Bool true | AG | 1/0/0/0 | HWPortal::WantsAirInFront | ALIVE |
| r_shockwave | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:531 | Bool true | AG | 5/0/0/1 | CUSTOM_CVARD, PPHeatRefraction::Render | ALIVE |
| r_shockwave_chroma | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:542 | Int 1 | AG | 3/0/0/1 | CUSTOM_CVARD, SetupShockwaves | ALIVE |
| r_shockwave_look | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:536 | Int 0 | AG | 3/0/0/1 | CUSTOM_CVARD, SetupShockwaves | ALIVE |
| r_shockwave_max | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:544 | Int 16 | AG | 3/0/0/1 | CUSTOM_CVARD, SetupShockwaves | ALIVE |
| r_shockwave_near | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:539 | Int 0 | AG | 3/0/0/1 | CUSTOM_CVARD, SetupShockwaves | ALIVE |
| r_shockwave_scale | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:533 | Float 1.0f | AG | 3/0/0/1 | CUSTOM_CVARD, SetupShockwaves | ALIVE |
| r_shockwave_test | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:548 | Int 0 | G | 2/0/0/1 | SetupShockwaves, static_assert | ALIVE |
| r_smoke | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:561 | Bool true | AG | 8/0/0/1 | CUSTOM_CVARD, PPSmokeVolume::Render | ALIVE |
| r_smoke_computetest | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:577 | Bool false | G | 3/0/0/1 | CUSTOM_CVARD, SmokeVolume::PrepareFrame | ALIVE |
| r_smoke_dissipation_scale | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:584 | Float 1.0f | AG | 3/0/0/1 | CUSTOM_CVARD, SmokeVolume::FillSimSettings | ALIVE |
| r_smoke_quality | R | src/common/rendering/hwrenderer/data/hw_cvars.cpp:566 | Int 2 | AG | 5/0/0/1 | CUSTOM_CVARD, SmokeVolume::PrepareFrame | ALIVE |
| vid_brightness | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:149 | Float 0.05f | AG | 2/0/0/2 | MakeXrPresentUniforms, VkPostprocess::DrawPresentTexture | ALIVE |
| vk_max_transfer_threads | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:85 | Int 2 | AGI | 1/0/0/2 | VulkanRenderDevice::VulkanRenderDevice | ALIVE, read at startup/creation only |
| vr_scene_multithread | P | src/common/rendering/hwrenderer/data/hw_cvars.cpp:278 | Bool false | AG | 1/0/0/3 | HWDrawInfo::RenderBSP | ALIVE |
| r_effects_governor | R | src/common/rendering/hwrenderer/data/hw_effectsgovernor.cpp:35 | Bool true | AG | 5/0/0/2 | CUSTOM_CVARD, EffectsGovernor::ActiveRung | ALIVE |
| r_effects_governor_floor | R | src/common/rendering/hwrenderer/data/hw_effectsgovernor.cpp:42 | Int EffectsGovernorCor.. | AG | 7/0/0/2 | CUSTOM_CVARD, EffectsGovernor::ActiveRung | ALIVE |
| r_effects_governor_target | R | src/common/rendering/hwrenderer/data/hw_effectsgovernor.cpp:50 | Int EffectsGovernorCor.. | AG | 4/0/0/2 | CUSTOM_CVARD, EffectsGovernor::EndFrame | ALIVE |
| r_gpuparticles_window | R | src/common/rendering/hwrenderer/data/hw_gpuparticlebuffer.cpp:40 | Bool true | G | 5/0/0/0 | CUSTOM_CVARD, GpuParticleBuffer::DebugReport | ALIVE |
| r_particlelights_test | R | src/common/rendering/hwrenderer/data/hw_gpuparticlebuffer.cpp:36 | Bool false | G | 5/1/0/1 | CCMD, CUSTOM_CVARD | ALIVE |
| gl_max_lights | P | src/common/rendering/hwrenderer/data/hw_lightbuffer.cpp:23 | Int 80000 | - | 1/2/0/0 | FLightBuffer::FLightBuffer | ALIVE |
| r_meshparticles | R | src/common/rendering/hwrenderer/data/hw_meshparticles.cpp:45 | Bool true | AG | 6/1/0/1 | CCMD, CUSTOM_CVARD | ALIVE |
| r_gpuparticles_looks | R | src/common/rendering/hwrenderer/data/hw_particledefbuffer.cpp:39 | Int 2 | AG | 4/1/0/1 | CCMD, CUSTOM_CVARD | ALIVE |
| r_perflog | R | src/common/rendering/hwrenderer/data/hw_perflog.cpp:122 | Int 0 | AG | 7/0/0/1 | CUSTOM_CVARD, CheckBenchActive | ALIVE |
| perf_track | R | src/common/rendering/hwrenderer/data/hw_perftrack.cpp:61 | Int 0 | AG | 15/0/0/2 | BenchDemo::Begin, CUSTOM_CVARD | ALIVE |
| perf_track_dir | R | src/common/rendering/hwrenderer/data/hw_perftrack.cpp:70 | String "" | AG | 1/0/0/0 | PerfTrack::OutputDir | ALIVE |
| perf_track_frames | R | src/common/rendering/hwrenderer/data/hw_perftrack.cpp:78 | Int 40000 | AG | 3/0/0/2 | PerfTrack::EndFrame | ALIVE |
| perf_track_hitch_ms | R | src/common/rendering/hwrenderer/data/hw_perftrack.cpp:74 | Float 0.0f | AG | 1/0/0/2 | PerfTrack::EndFrame | ALIVE |
| gl_light_castshadows | R | src/common/rendering/hwrenderer/data/hw_shadowmap.cpp:87 | Int 0 | AG | 6/0/0/3 | CUSTOM_CVARD, IShadowMap::CastShadowsOn | ALIVE |
| gl_light_shadowmap_lights | R | src/common/rendering/hwrenderer/data/hw_shadowmap.cpp:92 | Int 0 | AG | 4/0/0/3 | CUSTOM_CVARD, IShadowMap::CastShadowsReachAllLights | ALIVE |
| portablehud | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1105 | Bool false | AG | 3/1/0/1 | CCMD, VR_UsePortableHud | ALIVE |
| use_action_spawn_yzoffset | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1055 | Bool false | AG | 4/0/1/1 | P_LineAttack, P_LineTrace | ALIVE |
| vr_2dweaponOffsetX | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:702 | Float 0.f | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_2dweaponOffsetY | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:703 | Float 0.f | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_2dweaponOffsetZ | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:704 | Float 0.f | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_2dweaponScale | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:705 | Float 1.0f | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_3dweaponOffsetX | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:699 | Float 0.f | AG | 1/0/0/2 | RenderHUDModel | ALIVE |
| vr_3dweaponOffsetY | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:700 | Float 0.f | AG | 1/0/0/2 | RenderHUDModel | ALIVE |
| vr_3dweaponOffsetZ | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:701 | Float 0.f | AG | 1/0/0/2 | RenderHUDModel | ALIVE |
| vr_automap_border | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1124 | Int 1 | AG | 4/0/0/2 | PostGuiWheelEvent, VKOpenXRDeviceMode::DrawMountedHud | ALIVE |
| vr_automap_border_color | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1125 | Color 0x636363 | AG | 2/0/0/2 | PostGuiWheelEvent, VKOpenXRDeviceMode::DrawMountedHud | ALIVE |
| vr_automap_distance | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1111 | Float 1.f | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_automap_fixed_pitch | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1113 | Bool true | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_automap_fixed_roll | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1114 | Bool true | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_automap_mount | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1115 | Bool true | AG | 9/0/0/12 | D_Display, DrawHudToSurface | ALIVE |
| vr_automap_mount_pitch | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1121 | Float 45.0f | AG | 1/0/0/2 | VR_GetMountedHudTransform | ALIVE |
| vr_automap_mount_pos | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1116 | Int 1 | AG | 3/0/0/2 | PostGuiWheelEvent, VKOpenXRDeviceMode::DrawMountedHud | ALIVE |
| vr_automap_mount_roll | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1123 | Bool true | AG | 1/0/0/2 | VR_GetMountedHudTransform | ALIVE |
| vr_automap_mount_scale | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1117 | Float 0.15f | AG | 2/0/0/2 | PostGuiWheelEvent, VKOpenXRDeviceMode::DrawMountedHud | ALIVE |
| vr_automap_mount_xoffset | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1118 | Float 0.0f | AG | 1/0/0/2 | VR_GetMountedHudTransform | ALIVE |
| vr_automap_mount_yaw | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1122 | Float 0.0f | AG | 1/0/0/2 | VR_GetMountedHudTransform | ALIVE |
| vr_automap_mount_yoffset | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1119 | Float 0.15f | AG | 1/0/0/2 | VR_GetMountedHudTransform | ALIVE |
| vr_automap_mount_zoffset | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1120 | Float -0.05f | AG | 1/0/0/2 | VR_GetMountedHudTransform | ALIVE |
| vr_automap_rotate | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1112 | Float 13.f | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_automap_scale | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1109 | Float 0.4f | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_automap_stereo | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1110 | Float 1.0f | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_automap_use_hud | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1108 | Bool false | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_control_scheme | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:665 | Int 0 | AGI | 29/0/0/2 | AngleVectors, FSpriteModelFrame::ObjectToWorldMatrix | ALIVE |
| vr_crouch_use_button | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1054 | Bool true | AG | 9/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_desktop_view | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:628 | Int 2 | AG | 20/0/0/2 | PostGuiWheelEvent, ShouldPrepareDesktopMirrorEye | ALIVE |
| vr_desktop_view_openxr_render | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:653 | Bool false | AG | 1/0/0/2 | ShouldReuseSubmittedPresentForDesktopMirror | ALIVE |
| vr_enable_haptics | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1082 | Bool true | AG | 8/0/0/0 | CCMD, PostGuiWheelEvent | ALIVE |
| vr_eye_colorspace | R | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:693 | Int 0 | AG | 1/0/0/0 | VKOpenXRDeviceMode::CreateSwapchain | ALIVE, read at startup/creation only |
| vr_hand_lever_sign | R | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:2005 | Float 1.0f | AG | 5/0/0/0 | VRHandPointVel, VR_HandLeverSelfCheck | ALIVE |
| vr_hand_ofs_x | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1066 | Float 0.0f | AG | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_hand_ofs_y | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1067 | Float 0.0f | AG | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_hand_ofs_z | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1068 | Float 0.0f | AG | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_hand_pitch | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1070 | Float 0.0f | AG | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_hand_roll | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1071 | Float 0.0f | AG | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_hand_throw_window_ms | R | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1987 | Float 40.0f | AG | 1/0/0/0 | VR_HandVelAtPoint | ALIVE |
| vr_hand_window_ms | R | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1986 | Float 250.0f | AG | 1/0/0/0 | VRHandPeakIndex | ALIVE |
| vr_hand_yaw | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1069 | Float 0.0f | AG | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_height_adjust | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:637 | Float 0.0f | AG | 21/0/0/2 | AngleVectors, GetRawHmdHeightInMapUnit | ALIVE |
| vr_hitscan_ricochet | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:959 | Bool false | AG | 1/0/0/2 | ShouldSpawnRicochet | ALIVE |
| vr_hitscan_ricochet_chance | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:960 | Float 20.0f | AG | 2/0/0/2 | ShouldSpawnRicochet | ALIVE |
| vr_hitscan_tracer | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:942 | Int 0 | AG | 4/0/0/2 | P_QueueHitscanRicochet, P_QueueHitscanTracer | ALIVE |
| vr_hitscan_tracer_alpha | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:954 | Float 0.75f | AG | 1/0/0/2 | DrawHitscanTracerGeometry | ALIVE |
| vr_hitscan_tracer_color | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:953 | Color 0xffc040 | AG | 1/0/0/2 | DrawHitscanTracerGeometry | ALIVE |
| vr_hitscan_tracer_length | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:955 | Float 50.0f | AG | 1/0/0/2 | DrawHitscanTracers | ALIVE |
| vr_hitscan_tracer_offset | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:958 | Float 8.0f | AG | 1/0/0/2 | P_QueueHitscanTracer | ALIVE |
| vr_hitscan_tracer_speed | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:957 | Float 26.0f | AG | 3/0/0/2 | DrawHitscanTracers, P_QueueHitscanRicochet | ALIVE |
| vr_hitscan_tracer_width | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:956 | Float 0.25f | AG | 1/0/0/2 | DrawHitscanTracerGeometry | ALIVE |
| vr_holster_use_grip | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1051 | Bool true | AG | 1/0/0/0 | VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_hud_distance | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1090 | Float 1.f | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_hud_fixed_pitch | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1092 | Bool false | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_hud_fixed_roll | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1093 | Bool true | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_hud_mount | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1094 | Bool true | AG | 8/0/0/10 | D_Display, DrawHudToSurface | ALIVE |
| vr_hud_mount_pitch | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1100 | Float 0.0f | AG | 1/0/0/2 | VR_GetMountedHudTransform | ALIVE |
| vr_hud_mount_pos | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1095 | Int 1 | AG | 3/0/0/2 | PostGuiWheelEvent, VKOpenXRDeviceMode::DrawMountedHud | ALIVE |
| vr_hud_mount_roll | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1102 | Bool true | AG | 1/0/0/2 | VR_GetMountedHudTransform | ALIVE |
| vr_hud_mount_scale | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1099 | Float 0.15f | AG | 2/0/0/2 | PostGuiWheelEvent, VKOpenXRDeviceMode::DrawMountedHud | ALIVE |
| vr_hud_mount_xoffset | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1096 | Float 0.0f | AG | 1/0/0/2 | VR_GetMountedHudTransform | ALIVE |
| vr_hud_mount_yaw | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1101 | Float 0.0f | AG | 1/0/0/2 | VR_GetMountedHudTransform | ALIVE |
| vr_hud_mount_yoffset | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1097 | Float 0.0f | AG | 1/0/0/2 | VR_GetMountedHudTransform | ALIVE |
| vr_hud_mount_zoffset | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1098 | Float -0.20f | AG | 1/0/0/2 | VR_GetMountedHudTransform | ALIVE |
| vr_hud_rotate | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1091 | Float 10.f | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_hud_scale | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1088 | Float 0.25f | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_hud_stereo | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1089 | Float 1.4f | AG | 3/0/0/2 | AngleVectors, PostGuiWheelEvent | ALIVE |
| vr_joy_mode | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:669 | Int 1 | AGI | 4/0/0/3 | PostGuiWheelEvent, VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_laser_beam | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:710 | Bool false | AG | 3/0/0/2 | DrawLaserSightWorld | ALIVE |
| vr_laser_beam_alpha | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:713 | Float 0.3f | AG | 1/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_beam_emissive | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:768 | Float 1.8f | AG | 2/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_beam_fade | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:794 | Float 100.0f | AG | 1/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_beam_glow | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:745 | Float 1.0f | AG | 1/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_beam_length | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:720 | Int 0 | AG | 1/0/0/2 | GetLaserBeamEndpoints | ALIVE |
| vr_laser_beam_taper | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:773 | Float 0.45f | AG | 1/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_beam_width | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:714 | Float 0.15f | AG | 1/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_color | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:707 | Color 0xff0000 | AG | 3/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_color_cycle | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:902 | Bool false | AG | 1/0/0/2 | DrawLaserSightWorld | ALIVE |
| vr_laser_color_cycle_dot | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:913 | Bool false | AG | 1/0/0/2 | DrawLaserSightWorld | ALIVE |
| vr_laser_color_cycle_speed | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:908 | Float 60.0f | AG | 1/0/0/2 | CycleHue | ALIVE |
| vr_laser_color_mode | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:856 | Int 0 | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_color_offhand | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:860 | Color 0x40a0ff | AG | 2/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_color_per_slot | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:866 | Bool false | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_color_slot0 | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:876 | Color 0x909090 | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_color_slot1 | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:867 | Color 0xff4040 | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_color_slot2 | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:868 | Color 0xffa040 | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_color_slot3 | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:869 | Color 0xffe040 | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_color_slot4 | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:870 | Color 0x60ff60 | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_color_slot5 | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:871 | Color 0x40ffd0 | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_color_slot6 | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:872 | Color 0x40a0ff | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_color_slot7 | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:873 | Color 0xa060ff | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_color_slot8 | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:874 | Color 0xff60c0 | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_color_slot9 | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:875 | Color 0xffffff | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_dot_color | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:861 | Color 0xff2020 | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_dot_color_offhand | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:862 | Color 0x60c0ff | AG | 1/0/0/2 | GetLaserBeamColorFor | ALIVE |
| vr_laser_dot_range | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:817 | Float 320.0f | AG | 1/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_fixed_length | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:721 | Int 100 | AG | 1/0/0/2 | GetLaserBeamEndpoints | ALIVE |
| vr_laser_headshot_color | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:931 | Color 0xff2020 | AG | 2/0/0/2 | DrawLaserSightWorld | ALIVE |
| vr_laser_headshot_pulse | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:936 | Bool true | AG | 1/0/0/2 | DrawLaserSightWorld | ALIVE |
| vr_laser_headshot_pulse_speed | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:940 | Float 6.0f | AG | 1/0/0/2 | DrawLaserSightWorld | ALIVE |
| vr_laser_headshot_react | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:926 | Bool true | AG | 1/0/0/2 | DrawLaserSightWorld | ALIVE |
| vr_laser_hide_on_wheel | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:709 | Bool true | AG | 1/0/0/2 | DrawLaserSightWorld | ALIVE |
| vr_laser_lock | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:797 | Bool true | AG | 1/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_lock_rate | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:806 | Float 0.42f | AG | 1/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_lock_tighten | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:802 | Float 0.40f | AG | 1/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_other_players_beam | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:711 | Bool false | AG | 3/0/0/2 | DrawLaserSightWorld | ALIVE |
| vr_laser_other_players_pointer | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:712 | Bool false | AG | 3/0/0/2 | DrawLaserSightWorld | ALIVE |
| vr_laser_pointer_alpha | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:716 | Float 0.9f | AG | 1/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_pointer_glow | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:717 | Int 1 | AG | 2/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_pointer_glow_intensity | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:719 | Float 1.0f | AG | 1/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_pointer_glow_scale | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:718 | Float 1.5f | AG | 1/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_pointer_scale | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:715 | Float 0.1f | AG | 1/0/0/2 | DrawLaserBeamGeometry | ALIVE |
| vr_laser_show_melee | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:708 | Bool false | AG | 2/0/0/2 | DrawLaserSightWorld | ALIVE |
| vr_laser_sight | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:706 | Bool false | AG | 5/0/0/2 | CCMD, DrawLaserSightWorld | ALIVE |
| vr_laser_source_offset_x | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:722 | Float 0.0f | AG | 1/0/0/2 | GetLaserBeamEndpoints | ALIVE |
| vr_laser_source_offset_y | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:723 | Float 0.0f | AG | 1/0/0/2 | GetLaserBeamEndpoints | ALIVE |
| vr_laser_source_offset_z | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:724 | Float 0.0f | AG | 1/0/0/2 | GetLaserBeamEndpoints | ALIVE |
| vr_missile_haptic_level | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1085 | Float 0.6f | AG | 0/0/0/0 |  | DEAD: only EXTERN_CVAR p_mobj.cpp:100 |
| vr_momentum | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1052 | Bool false | AG | 0/0/1/2 | zscript | ALIVE |
| vr_momentum_threshold | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1053 | Float 1.f | AG | 0/0/1/1 | zscript | ALIVE |
| vr_move_speed | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1147 | Int 19 | AGI | 2/0/0/2 | CUSTOM_CVAR | ALIVE |
| vr_move_use_offhand | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:695 | Bool false | AG | 1/1/0/2 | PostGuiWheelEvent | ALIVE |
| vr_offhand_ofs_x | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1075 | Float 0.0f | AG | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_offhand_ofs_y | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1076 | Float 0.0f | AG | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_offhand_ofs_z | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1077 | Float 0.0f | AG | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_offhand_pitch | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1079 | Float 0.0f | AG | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_offhand_roll | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1080 | Float 0.0f | AG | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_offhand_yaw | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1078 | Float 0.0f | AG | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_openxr_debug_submit_mode | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:640 | Int 0 | AG | 2/0/0/2 | VKOpenXRDeviceMode::AcquireXRSwapchain | ALIVE |
| vr_openxr_eye_shift_scale | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:639 | Float 1.0f | AG | 0/0/0/2 |  | DEAD: EXTERN vk_openxrdevice.cpp:110 + menudef.txt:2235, never read |
| vr_openxr_force_recommended_viewport | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:646 | Bool false | AG | 0/0/0/2 |  | DEAD: menudef.txt:2236 only |
| vr_openxr_fov_adjust_deg | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:638 | Float 0.0f | AG | 2/0/0/2 | AdjustOpenXREyeFov, VKOpenXRDeviceEyePose::GetRenderFov | ALIVE |
| vr_openxr_multiview | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:643 | Bool false | AG | 3/0/0/5 | HWSprite::EmitBillboardPayload, VKOpenXRDeviceMode::GetBenchmarkInfo | ALIVE |
| vr_openxr_multiview_mirror_reuse | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:654 | Bool true | AG | 1/0/0/4 | ShouldReuseSubmittedPresentForDesktopMirror | ALIVE |
| vr_openxr_multiview_postprocess | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:655 | Bool true | AG | 5/0/0/4 | VKOpenXRDeviceMode::GetBenchmarkInfo, VkPostprocess::BlitSceneToPostprocess | ALIVE |
| vr_openxr_present_brightness_bias | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:663 | Float -0.15f | AG | 1/0/0/2 | MakeXrPresentUniforms | ALIVE |
| vr_openxr_present_contrast_bias | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:662 | Float 0.85f | AG | 1/0/0/2 | MakeXrPresentUniforms | ALIVE |
| vr_openxr_present_gamma_bias | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:661 | Float 0.886f | AG | 1/0/0/2 | MakeXrPresentUniforms | ALIVE |
| vr_openxr_present_saturation_bias | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:664 | Float 1.15f | AG | 1/0/0/2 | MakeXrPresentUniforms | ALIVE |
| vr_openxr_render_scale | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:649 | Float 1.0f | AG | 2/0/0/4 | VKOpenXRDeviceMode::GetBenchmarkInfo, VKOpenXRDeviceMode::GetRecommendedRenderSize | ALIVE |
| vr_openxr_sync_mode | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:641 | Int 1 | AG | 3/0/0/2 | AppendBenchmarkHeader, VKOpenXRDeviceMode::GetBenchmarkInfo | ALIVE |
| vr_openxr_use_screen_viewport_for_submit | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:652 | Bool true | AG | 0/0/0/2 |  | DEAD: menudef.txt:2237 only |
| vr_overlayscreen | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:629 | Int 2 | AG | 14/0/0/6 | PostGuiWheelEvent, VKOpenXRDeviceMode::RenderVirtualScreen | ALIVE |
| vr_overlayscreen_always | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:630 | Bool false | AG | 10/0/0/2 | PostGuiWheelEvent, VKOpenXRDeviceMode::DetermineFrameRenderMode | ALIVE |
| vr_overlayscreen_bg | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:634 | Int 0 | AG | 6/0/0/2 | GetVirtualScreenBackgroundColor, PostGuiWheelEvent | ALIVE |
| vr_overlayscreen_dist | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:632 | Float 0. | AG | 4/0/0/2 | BuildControllerFollowOverlayAnchor, BuildOverlayRelativeTransform | ALIVE |
| vr_overlayscreen_size | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:631 | Float 1. | AG | 3/0/0/2 | GetOpenXRVirtualScreenMeters, PostGuiWheelEvent | ALIVE |
| vr_overlayscreen_vpos | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:633 | Float 0. | AG | 6/0/0/2 | BuildControllerFollowOverlayAnchor, BuildOverlayRelativeTransform | ALIVE |
| vr_override_weap_pos | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1141 | Bool false | - | 2/0/0/0 | UpdateCanonicalMainHandPose, VRMode::SetUp | ALIVE |
| vr_pickup_haptic_level | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1083 | Float 0.2 | AG | 6/0/0/2 | FMugShot::UpdateState, V_AddPlayerBlend | ALIVE |
| vr_quake_haptic_level | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1084 | Float 0.8 | AG | 3/0/0/2 | R_SetupFrame | ALIVE |
| vr_render_weap_in_scene | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1142 | Bool false | - | 1/0/0/0 | VRMode::RenderPlayerSpritesInScene | ALIVE |
| vr_run_multiplier | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1151 | Float 1.0 | AGI | 2/0/0/2 | CUSTOM_CVAR | ALIVE |
| vr_secondary_button_mappings | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:963 | Bool true | AG | 10/0/0/2 | PostGuiWheelEvent, VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_snapTurn | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:961 | Float 45.0f | AG | 14/0/0/2 | PostGuiWheelEvent, VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_stabilize_distance_inches | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1031 | Float 8.0f | AG | 1/0/0/0 | VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_stabilize_requires_grab | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1043 | Bool false | AG | 1/0/0/0 | VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_support_cone_enter | R | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1003 | Float 0.955f | AG | 1/0/0/0 | VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_support_cone_exit | R | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1004 | Float 0.900f | AG | 1/0/0/0 | VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_support_dist_max | R | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1009 | Float 0.90f | AG | 1/0/0/0 | VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_support_dist_min | R | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1008 | Float 0.15f | AG | 1/0/0/0 | VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_support_mode | R | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1022 | Int 3 | AG | 1/0/0/2 | VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_support_slide_deadband | R | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1028 | Float 0.02f | AG | 1/0/0/2 | VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_switch_sticks | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:962 | Bool false | AG | 3/0/0/2 | PostGuiWheelEvent, VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_teleport | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:696 | Bool false | AG | 11/0/0/2 | AngleVectors, G_BuildTiccmd | ALIVE |
| vr_two_handed_weapons | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:978 | Bool false | AG | 3/0/0/0 | PostGuiWheelEvent | DORMANT (GL/OpenVR code only) |
| vr_vunits_per_meter | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:636 | Float 34.0f | AG | 87/0/0/2 | AddPlayerLengths, AngleVectors | ALIVE |
| vr_walk_multiplier | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:1155 | Float 1.0 | AGI | 1/0/0/2 | CUSTOM_CVAR | ALIVE |
| vr_weaponRotate | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:697 | Float -30.f | AG | 5/0/0/2 | ADD_STAT, PostGuiWheelEvent | ALIVE |
| vr_weaponScale | P | src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:698 | Float 1.02f | AG | 8/0/0/2 | AngleVectors, CCMD | ALIVE |
| vr_wheel_auto_split | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:45 | Bool true | AG | 1/0/0/1 | VRWheel_Available | ALIVE |
| vr_wheel_deadzone | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:79 | Float 0.30f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_desktop | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:101 | Bool false | AG | 2/0/0/0 | VRWheel_Available | ALIVE |
| vr_wheel_desktop_range | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:103 | Float 22.0f | AG | 1/0/0/0 | VRWheel_Available | ALIVE |
| vr_wheel_distance | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:49 | Float 0.05f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_event_close | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:66 | String "vrwheel_close" | AG | 1/0/0/0 | VRWheel_Available | ALIVE |
| vr_wheel_event_open | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:65 | String "vrwheel_open" | AG | 1/0/0/0 | VRWheel_Available | ALIVE |
| vr_wheel_hide_hand_weapon | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:41 | Bool true | AG | 1/0/0/2 | VRWheel_ShouldSuppressWeaponHand | ALIVE |
| vr_wheel_hide_other_class_weapons | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:42 | Bool true | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_icon_bg_color | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:46 | Color (int)MAKEARGB(128,.. | AG | 3/0/0/2 | DrawOneWheel | ALIVE |
| vr_wheel_icon_disable_color | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:48 | Color (int)MAKEARGB(160,.. | AG | 3/0/0/0 | DrawOneWheel | ALIVE |
| vr_wheel_icon_load_model | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:44 | Bool false | AG | 1/0/0/6 | VRWheel_Available | ALIVE |
| vr_wheel_icon_model_scale | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:81 | Float 0.8f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_icon_model_xoffset | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:83 | Float -40.0f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_icon_model_yaw | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:82 | Float -135.0f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_icon_model_zoffset | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:84 | Float 0.0f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_icon_scale | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:80 | Float 1.2f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_icon_select_color | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:47 | Color (int)MAKEARGB(160,.. | AG | 3/0/0/2 | DrawOneWheel | ALIVE |
| vr_wheel_info | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:150 | Int 1 | AG | 2/0/0/3 | VRWheel_Available | ALIVE |
| vr_wheel_info_bg_color | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:152 | Color (int)MAKEARGB(190,.. | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_info_scale | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:151 | Float 1.0f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_leash | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:135 | Float 0.0f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_pointer_dot | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:141 | Bool true | AG | 1/0/0/2 | DrawOneWheel | ALIVE |
| vr_wheel_radius | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:78 | Float 8.0f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_select_angle | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:85 | Float 30.0f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_selection_type | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:115 | Int 3 | AG | 4/0/0/4 | DrawOneWheel, VRWheel_Available | ALIVE |
| vr_wheel_sound | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:43 | Bool true | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_stick_deadzone | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:119 | Float 0.45f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_switch_hands | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:40 | Bool false | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_time_event | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:74 | String "" | AG | 1/0/0/0 | VRWheel_Available | ALIVE |
| vr_wheel_time_event_off | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:75 | String "" | AG | 1/0/0/0 | VRWheel_Available | ALIVE |
| vr_wheel_weapon_all | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:39 | Bool false | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_xoffset | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:76 | Float 0.0f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| vr_wheel_yoffset | P | src/common/rendering/hwrenderer/data/hw_vrwheel.cpp:77 | Float 0.0f | AG | 1/0/0/2 | VRWheel_Available | ALIVE |
| gl_bloom_anamorphic | P | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:68 | Bool false | A | 2/0/0/0 | PPBloom::RestLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_anamorphic_ratio | P | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:69 | Float 3.0f | A | 2/0/0/0 | PPBloom::RestLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_chromatic | P | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:82 | Float 0.0f | A | 2/0/0/0 | PPBloom::RestLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_knee | P | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:58 | Float 0.5f | A | 3/0/0/0 | CCMD, PPBloom::RestLook | ALIVE |
| gl_bloom_override_strength | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:107 | Float 1.0f | A | 1/0/0/0 | BloomOverrideStrength | ALIVE |
| gl_bloom_pin_amount | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:156 | Float 1.4f | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_pin_anamorphic | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:170 | Bool false | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_pin_anamorphic_ratio | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:171 | Float 3.0f | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_pin_beams | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:139 | Bool false | AI | 8/0/0/0 | CUSTOM_CVARD, PPBloom::RenderBloom | ALIVE |
| gl_bloom_pin_captured | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:148 | Bool false | A | 3/0/0/0 | CUSTOM_CVAR, PPBloom::RenderBloom | ALIVE |
| gl_bloom_pin_chromatic | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:179 | Float 0.0f | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_pin_exposure_base | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:191 | Float 0.35f | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_pin_exposure_min | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:190 | Float 0.35f | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_pin_exposure_scale | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:189 | Float 1.3f | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_pin_exposure_speed | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:192 | Float 0.05f | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_pin_knee | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:165 | Float 0.5f | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_pin_step | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:184 | Float 1.0f | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_pin_threshold | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:160 | Float 1.0f | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_pin_tint_b | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:178 | Float 1.0f | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_pin_tint_g | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:177 | Float 1.0f | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_pin_tint_r | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:176 | Float 1.0f | A | 3/0/0/0 | PPBloom::PinnedLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_step | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:95 | Float 1.0f | A | 2/0/0/0 | PPBloom::RestLook, SetPinnedBloomLook | ALIVE |
| gl_bloom_threshold | P | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:48 | Float 1.0f | A | 3/0/0/0 | CCMD, PPBloom::RestLook | ALIVE |
| gl_bloom_tint_b | P | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:81 | Float 1.0f | A | 3/0/0/0 | CCMD, PPBloom::RestLook | ALIVE |
| gl_bloom_tint_g | P | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:80 | Float 1.0f | A | 3/0/0/0 | CCMD, PPBloom::RestLook | ALIVE |
| gl_bloom_tint_r | P | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:79 | Float 1.0f | A | 3/0/0/0 | CCMD, PPBloom::RestLook | ALIVE |
| gl_tonemap_desaturate | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:466 | Float 0.0f | AG | 1/0/0/2 | PPTonemap::FilmicUniforms | ALIVE |
| gl_tonemap_exposure | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:457 | Float 1.0f | AG | 1/0/0/2 | PPTonemap::FilmicUniforms | ALIVE |
| gl_tonemap_knee | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:434 | Float 0.85f | AG | 1/0/0/2 | PPTonemap::FilmicUniforms | ALIVE |
| gl_tonemap_white | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:447 | Float 1.10f | AG | 1/0/0/2 | PPTonemap::FilmicUniforms | ALIVE |
| r_effects_foveated | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:391 | Bool true | AG | 3/0/0/1 | CUSTOM_CVARD, SetupEmissiveVolumes | ALIVE |
| r_effects_sharedmarch | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:357 | Bool true | AG | 3/0/0/1 | CUSTOM_CVARD, PPEmissiveVolumes::Render | ALIVE |
| r_emissivevolumes_resolution | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:394 | Int 2 | AG | 3/0/0/1 | CUSTOM_CVARD, SetupEmissiveVolumes | ALIVE |
| r_emissivevolumes_steps | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:366 | Int 32 | AG | 3/0/0/1 | CUSTOM_CVARD, SetupEmissiveVolumes | ALIVE |
| r_emissivevolumes_tiles | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:376 | Bool true | G | 5/0/0/0 | CUSTOM_CVARD, PPEmissiveVolumes::Render | ALIVE |
| r_lightmask_debug | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:252 | Int 0 | - | 7/0/0/1 | CUSTOM_CVARD, PPLightMask::BeginFrame | ALIVE |
| r_smoke_beams | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:327 | Bool true | AG | 3/0/0/1 | CUSTOM_CVARD, SetupSmokeVolume | ALIVE |
| r_smoke_beams_depth | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:332 | Bool true | AG | 3/0/0/1 | CUSTOM_CVARD, SetupSmokeVolume | ALIVE |
| r_smoke_cones_depth | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:336 | Bool true | AG | 3/0/0/1 | CUSTOM_CVARD, SetupSmokeVolume | ALIVE |
| r_smoke_debugslice | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:317 | Bool false | - | 3/0/0/1 | CUSTOM_CVARD, SetupSmokeVolume | ALIVE |
| r_smoke_density_scale | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:311 | Float 1.0f | AG | 3/0/0/1 | CUSTOM_CVARD, SetupSmokeVolume | ALIVE |
| r_smoke_preset | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:287 | Int 3 | AGI | 0/0/0/0 |  | CALLBACK-ONLY (CUSTOM body acts; value never read) |
| r_smoke_steps | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:268 | Int 24 | AG | 5/0/0/1 | CUSTOM_CVARD, SetupSmokeVolume | ALIVE |
| r_smoke_temporal | R | src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:346 | Bool true | AG | 4/0/0/1 | CUSTOM_CVARD, PPSmokeVolume::Render | ALIVE |
| vid_refreshrate | P | src/common/rendering/v_video.cpp:211 | Int 72 | AG | 8/0/0/3 | AppendBenchmarkHeader, PostGuiWheelEvent | ALIVE |
| r_smoke_light_cull | R | src/common/rendering/vulkan/renderer/vk_smokevolume.cpp:44 | Bool true | G | 4/1/0/0 | CUSTOM_CVARD, VkSmokeVolume::Run | ALIVE |
| vr_body_debug | R | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:268 | Bool false | - | 1/0/0/0 | VKOpenXRDeviceMode::GetHmdTransform | ALIVE |
| vr_grip_debug | P | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:138 | Bool true | AG | 1/0/0/0 | VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_handcmd_debug | R | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:8299 | Int 0 | - | 1/0/0/0 | VKOpenXRDeviceMode::WriteHandCommand | ALIVE |
| vr_haptic_debug | P | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:155 | Bool false | AG | 3/0/0/0 | VKOpenXRDeviceMode::ProcessHaptics, VKOpenXRDeviceMode::Vibrate | ALIVE |
| vr_height_diag | R | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:989 | Bool false | A | 1/0/0/0 | VKOpenXRDeviceMode::GetHmdBaseTransform | ALIVE |
| vr_menu_keep_world | R | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:217 | Bool true | AG | 2/1/0/2 | CUSTOM_CVARD, VKOpenXRDeviceMode::DetermineFrameRenderMode | ALIVE |
| vr_menu_pointer | P | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:207 | Bool true | AG | 3/0/1/0 | PostGuiWheelEvent, VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_menu_pointer_color | P | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:208 | Color 0xffffff | AG | 4/0/0/2 | PostGuiWheelEvent, VKOpenXRDeviceMode::RenderVirtualScreen | ALIVE |
| vr_mouse_in_menu | P | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:209 | Bool false | AG | 4/0/0/2 | PostGuiWheelEvent, VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_posestats | P | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:135 | Bool false | AG | 1/0/0/0 | VKOpenXRDeviceMode::UpdateControllerState | ALIVE |
| vr_rig_hmd_pitch | R | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:287 | Float 0.f | - | 2/0/0/0 | VKOpenXRDeviceMode::BeginXRFrame | ALIVE |
| vr_rig_hmd_yaw | R | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:286 | Float 0.f | - | 2/0/0/0 | VKOpenXRDeviceMode::BeginXRFrame | ALIVE |
| vr_rig_pose_0 | R | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:282 | String "" | - | 1/0/0/0 | RigPoseOverride | ALIVE |
| vr_rig_pose_1 | R | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:283 | String "" | - | 1/0/0/0 | RigPoseOverride | ALIVE |
| vr_rig_pose_yaw | R | src/common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp:284 | Float 0.f | - | 1/0/0/0 | RigPoseOverride | ALIVE |
| vk_gpu_checkpoints | R | src/common/rendering/vulkan/system/vk_renderdevice.cpp:114 | Bool true | AG | 3/0/0/0 | VkCommandBufferManager::GpuCheckpoint, VulkanRenderDevice::VulkanRenderDevice | ALIVE, read at startup/creation only |
| vk_robust_buffer_access | R | src/common/rendering/vulkan/system/vk_renderdevice.cpp:115 | Bool false | AG | 1/0/0/0 | VulkanRenderDevice::VulkanRenderDevice | ALIVE, read at startup/creation only |
| compat_oldrandom | P | src/d_main.cpp:1024 | Flag compatflags2 | - | 0/0/0/1 |  | DEAD: flag bit COMPATF2_OLD_RANDOM_GENERATOR set (d_main.cpp:918,928) but never tested |
| vr_hud_debug_border | P | src/d_main.cpp:650 | Bool false | A | 1/0/0/2 | DrawVRHudDebugBorder | ALIVE |
| net_autoresync | R | src/d_net.cpp:4134 | Bool true | NS | 2/0/0/0 | NetState_HandlePacket | ALIVE |
| net_statechunk | R | src/d_net.cpp:4135 | Int 4000 | N | 1/0/0/0 | NetState_Pump | ALIVE |
| cl_otherplayerhealth | P | src/d_netinfo.cpp:92 | Int 2 | A | 0/0/1/2 | zscript | ALIVE |
| cl_otherplayernames | P | src/d_netinfo.cpp:91 | Int 2 | A | 0/0/1/2 | zscript | ALIVE |
| vr_handaim | R | src/d_netinfo.cpp:78 | Bool false | U | 3/1/0/0 | EXTERN_CVAR, G_BuildTiccmd | ALIVE |
| vr_holster_reach | R | src/d_netinfo.cpp:90 | Float 1.f | AU | 0/1/0/0 |  | ALIVE |
| vr_main_left | R | src/d_netinfo.cpp:89 | Bool false | AU | 0/2/0/0 |  | ALIVE |
| vr_seated | R | src/d_netinfo.cpp:88 | Int -1 | AU | 1/1/0/0 | CCMD | ALIVE |
| vr_stand_eye | R | src/d_netinfo.cpp:87 | Float 0.f | AU | 5/1/0/0 | CCMD, VRRig_StandEyeTic | ALIVE |
| netplay_handtest | R | src/g_game.cpp:120 | Int 0 | N | 3/0/0/0 | G_BuildTiccmd | ALIVE |
| netplay_roomtest | R | src/g_game.cpp:119 | Int 0 | N | 4/0/0/0 | G_BuildTiccmd | ALIVE |
| puristmode | P | src/g_game.cpp:401 | Bool false | AG | 3/0/0/1 | UpdateCanonicalMainHandPose, VRDemo_BeginTic | ALIVE |
| use_walk_multiplier | P | src/g_game.cpp:403 | Bool false | - | 2/0/0/1 | CUSTOM_CVAR | ALIVE |
| vanilla_melee_attack | P | src/g_game.cpp:402 | Bool false | AG | 0/0/10/1 | zscript | ALIVE |
| perf_bench_hash | R | src/g_perfbench.cpp:55 | Bool true | AG | 4/0/0/2 | FARG, PerfBench::EndPlayback | ALIVE |
| menu_showdoublebindings | P | src/menu/doommenu.cpp:391 | Bool false | I | 1/0/0/1 | CheckSkipGameOptionBlock | ALIVE |
| menu_showexperimental | P | src/menu/doommenu.cpp:380 | Bool false | AI | 1/0/0/2 | CheckSkipGameOptionBlock | ALIVE |
| mp_host_gamemode | P | src/menu/doommenu.cpp:99 | Int 0 | AG | 3/1/0/1 | CCMD, M_BuildMultiplayerOptionGroups | ALIVE |
| mp_host_map | P | src/menu/doommenu.cpp:102 | String "" | AG | 3/1/0/1 | CCMD, M_BuildMultiplayerOptionGroups | ALIVE |
| mp_host_netmode | P | src/menu/doommenu.cpp:100 | Int 0 | AG | 3/1/0/1 | CCMD, M_BuildMultiplayerOptionGroups | ALIVE |
| mp_host_players | P | src/menu/doommenu.cpp:98 | Int 2 | AG | 3/1/0/1 | CCMD, M_BuildMultiplayerOptionGroups | ALIVE |
| mp_host_skill | P | src/menu/doommenu.cpp:101 | Int 3 | AG | 3/1/0/1 | CCMD, M_BuildMultiplayerOptionGroups | ALIVE |
| mp_join_address | P | src/menu/doommenu.cpp:103 | String "" | AG | 1/1/0/1 | CCMD | ALIVE |
| cmdlineprofile | P | src/menu/profiledef.cpp:27 | String "" | AGI | 0/2/0/1 |  | ALIVE |
| worldclock_log | R | src/p_tick.cpp:77 | Bool false | - | 1/0/0/0 | Level_SetTimeScale | ALIVE |
| gl_light_distance_cull | P | src/playsim/a_dynlight.cpp:45 | Float 2000.0 | AG | 3/0/0/5 | AppendBenchmarkHeader, gl_IsDistanceCulled | ALIVE |
| gl_light_flat_candidate_budget | P | src/playsim/a_dynlight.cpp:49 | Int 16 | AG | 3/0/0/4 | AppendBenchmarkHeader, HWDrawInfo::SetupLightsForOtherPlane | ALIVE |
| gl_light_flat_max_lights | P | src/playsim/a_dynlight.cpp:47 | Int 1000 | AG | 3/0/0/5 | AppendBenchmarkHeader, HWDrawInfo::SetupLightsForOtherPlane | ALIVE |
| gl_light_max_collected_subsectors | P | src/playsim/a_dynlight.cpp:46 | Int 1000 | AG | 2/0/0/4 | AppendBenchmarkHeader, FDynamicLight::CollectWithinRadius | ALIVE |
| gl_light_max_intensity | P | src/playsim/a_dynlight.cpp:44 | Float 1000.0 | AG | 3/0/0/4 | AppendBenchmarkHeader, FDynamicLight::Activate | ALIVE |
| gl_light_range_limit | P | src/playsim/a_dynlight.cpp:51 | Int 64 | AG | 2/0/0/4 | AppendBenchmarkHeader | ALIVE |
| gl_light_wall_candidate_budget | P | src/playsim/a_dynlight.cpp:50 | Int 8 | AG | 2/0/0/4 | AppendBenchmarkHeader, HWWall::SetupLights | ALIVE |
| gl_light_wall_max_lights | P | src/playsim/a_dynlight.cpp:48 | Int 1000 | AG | 2/0/0/5 | AppendBenchmarkHeader, HWWall::SetupLights | ALIVE |
| ext_haptic_level_damage_projectile | P | src/playsim/p_actionfunctions.cpp:81 | Float 1.0 | AG | 0/0/0/0 |  | ALIVE (read by runtime-built name, c_cvars.cpp:1443 C_GetExternalHapticLevelValue) |
| ext_haptic_level_fire_weapon | P | src/playsim/p_actionfunctions.cpp:84 | Float 1.0 | AG | 0/0/0/0 |  | ALIVE (read by runtime-built name, c_cvars.cpp:1443 C_GetExternalHapticLevelValue) |
| ext_haptic_level_global_intensity | P | src/playsim/p_actionfunctions.cpp:80 | Float 1.0 | AG | 2/0/0/2 | C_GetExternalHapticLevelValue | ALIVE |
| ext_haptic_level_healstation | P | src/playsim/p_actionfunctions.cpp:86 | Float 1.0 | AG | 0/0/0/0 |  | ALIVE (read by runtime-built name, c_cvars.cpp:1443 C_GetExternalHapticLevelValue) |
| ext_haptic_level_heartbeat | P | src/playsim/p_actionfunctions.cpp:87 | Float 1.0 | AG | 0/0/0/0 |  | ALIVE (read by runtime-built name, c_cvars.cpp:1443 C_GetExternalHapticLevelValue) |
| ext_haptic_level_pickup | P | src/playsim/p_actionfunctions.cpp:83 | Float 1.0 | AG | 0/0/0/0 |  | ALIVE (read by runtime-built name, c_cvars.cpp:1443 C_GetExternalHapticLevelValue) |
| ext_haptic_level_pickup_weapon | P | src/playsim/p_actionfunctions.cpp:82 | Float 1.0 | AG | 0/0/0/0 |  | ALIVE (read by runtime-built name, c_cvars.cpp:1443 C_GetExternalHapticLevelValue) |
| ext_haptic_level_poison | P | src/playsim/p_actionfunctions.cpp:85 | Float 1.0 | AG | 0/0/0/0 |  | ALIVE (read by runtime-built name, c_cvars.cpp:1443 C_GetExternalHapticLevelValue) |
| ext_haptic_level_rumble | P | src/playsim/p_actionfunctions.cpp:88 | Float 1.0 | AG | 0/0/0/0 |  | ALIVE (read by runtime-built name, c_cvars.cpp:1443 C_GetExternalHapticLevelValue) |
| sv_checklofreachable | R | src/playsim/p_actionfunctions.cpp:2163 | Bool false | S | 1/0/0/0 | DEFINE_ACTION_FUNCTION | ALIVE |
| vr_recoil | P | src/playsim/p_actionfunctions.cpp:77 | Bool false | AG | 1/0/1/2 | DEFINE_ACTION_FUNCTION | ALIVE |
| affect_damage_cap | R | src/playsim/p_affect.cpp:50 | Int 200 | AS | 2/0/0/0 | Affect_Hit, Affect_Impact | ALIVE |
| affect_damage_per_impulse | R | src/playsim/p_affect.cpp:49 | Float 0.25f | AS | 1/0/0/0 | Affect_Hit | ALIVE |
| affect_debug | R | src/playsim/p_affect.cpp:77 | Int 0 | - | 7/0/0/0 | Affect_Deflect, Affect_Hit | ALIVE |
| affect_deflect_mark_tics | R | src/playsim/p_affect.cpp:76 | Int 35 | AS | 1/0/0/0 | Affect_Deflect | ALIVE |
| affect_deflect_retarget | R | src/playsim/p_affect.cpp:63 | Bool true | AS | 1/0/0/0 | Affect_Deflect | ALIVE |
| affect_deflect_speed_max | R | src/playsim/p_affect.cpp:72 | Float 2.0f | AS | 1/0/0/0 | Affect_Deflect | ALIVE |
| affect_deflect_speed_min | R | src/playsim/p_affect.cpp:71 | Float 0.5f | AS | 1/0/0/0 | Affect_Deflect | ALIVE |
| affect_impact_damage_scale | R | src/playsim/p_affect.cpp:55 | Float 0.35f | AS | 1/0/0/0 | Affect_Impact | ALIVE |
| affect_impact_min_speed | R | src/playsim/p_affect.cpp:56 | Float 4.0f | AS | 2/0/0/0 | Affect_Impact, Affect_Tick | ALIVE |
| affect_impulse_scale | R | src/playsim/p_affect.cpp:47 | Float 1.0f | AS | 1/0/0/0 | Affect_Hit | ALIVE |
| affect_knockback_scale | R | src/playsim/p_affect.cpp:48 | Float 1.0f | AS | 7/0/0/0 | Affect_Hit, Affect_Push | ALIVE |
| affect_knockdown_tics | R | src/playsim/p_affect.cpp:53 | Int 50 | AS | 1/0/0/0 | Affect_Hit | ALIVE |
| affect_meter_drain | R | src/playsim/p_affect.cpp:51 | Float 6.0f | AS | 1/0/0/0 | Affect_Tick | ALIVE |
| affect_stagger_tics | R | src/playsim/p_affect.cpp:52 | Int 9 | AS | 1/0/0/0 | Affect_Hit | ALIVE |
| affect_stun_tics | R | src/playsim/p_affect.cpp:54 | Int 50 | AS | 1/0/0/0 | Affect_Hit | ALIVE |
| vr_guard_debug | R | src/playsim/p_guard.cpp:37 | Int 1 | A | 4/0/0/0 | P_GuardBlocksDamage, P_GuardRecordBlock | ALIVE |
| vr_guard_deflect | R | src/playsim/p_guard.cpp:47 | Bool false | AS | 1/0/0/0 | PIT_CheckThing | ALIVE |
| drawnshape_debug | R | src/playsim/p_map.cpp:665 | Int 0 | - | 1/0/0/0 | P_SweepDrawnShape | ALIVE |
| hurtvolume_debug | R | src/playsim/p_map.cpp:264 | Int 0 | - | 1/0/0/0 | P_HurtVolumeStruck | ALIVE |
| sv_vr_shotclamp | R | src/playsim/p_map.cpp:308 | Bool true | S | 1/0/0/0 | P_ShotOrigin | ALIVE |
| use_mode | P | src/playsim/p_map.cpp:94 | Int 0 | AG | 4/0/0/1 | P_UseLines, P_UsePuzzleItem | ALIVE |
| vr_shotclamp_debug | R | src/playsim/p_map.cpp:309 | Bool false | N | 1/0/0/0 | P_ShotOrigin | ALIVE |
| sv_traceexact3dwater | R | src/playsim/p_trace.cpp:47 | Bool false | S | 1/0/0/0 | Trace | ALIVE |
| netplay_desynctest | R | src/playsim/p_user.cpp:291 | Bool false | N | 3/0/0/0 | P_PlayerThink | ALIVE |
| vr_slotload_debug | R | src/playsim/p_user.cpp:2137 | Bool false | - | 2/0/0/0 | VRSlots_DebugSaved, VRSlots_MoveStaged | ALIVE |
| vr_demo_debug | R | src/playsim/p_vrdemo.cpp:66 | Bool false | - | 2/0/0/0 | VRDemo_BeginTic, VRDemo_ReadFrame | ALIVE |
| vr_replay_camera | R | src/playsim/p_vrdemo.cpp:64 | Bool true | AG | 1/0/0/2 | VRDemo_AdjustReplayView | ALIVE |
| vr_spectator | R | src/playsim/p_vrdemo.cpp:51 | Int 0 | AG | 2/0/0/2 | VKOpenXRDeviceMode::FinalizeEyeImage, VKOpenXRDeviceMode::RenderDesktopMirror | ALIVE |
| vr_spectator_cutangle | R | src/playsim/p_vrdemo.cpp:62 | Float 30.f | AG | 1/0/0/2 | FVRViewStabilizer::Update | ALIVE |
| vr_spectator_fov | R | src/playsim/p_vrdemo.cpp:56 | Float 70.f | AG | 1/0/0/2 | VKOpenXRDeviceMode::RenderSpectatorEye | ALIVE |
| vr_spectator_pitchfollow | R | src/playsim/p_vrdemo.cpp:60 | Float 0.85f | AG | 1/0/0/2 | FVRViewStabilizer::Update | ALIVE |
| vr_spectator_rolllock | R | src/playsim/p_vrdemo.cpp:58 | Float 1.f | AG | 1/0/0/2 | FVRViewStabilizer::Update | ALIVE |
| vr_spectator_smooth | R | src/playsim/p_vrdemo.cpp:53 | Float 0.35f | AG | 1/0/0/2 | FVRViewStabilizer::Update | ALIVE |
| vr_gear_debug | R | src/playsim/p_vrgear.cpp:38 | Int 0 | - | 4/0/0/0 | CCMD, CVAR | ALIVE |
| ghost_loop | R | src/playsim/p_vrghost.cpp:58 | Bool false | - | 1/0/0/0 | Ghost_OnBuildTiccmd | ALIVE |
| ghost_play | R | src/playsim/p_vrghost.cpp:57 | String "" | - | 3/0/0/0 | EnsureLoaded, Ghost_OnBuildTiccmd | ALIVE |
| ghost_record | R | src/playsim/p_vrghost.cpp:56 | String "" | - | 5/0/0/0 | EXTERN_CVAR, Ghost_OnBuildTiccmd | ALIVE |
| vr_grab_debug | R | src/playsim/p_vrgrab.cpp:37 | Int 0 | - | 1/0/0/0 | VRGrab_Tick | ALIVE |
| vr_replay_loop | R | src/playsim/p_vrreplay.cpp:31 | Bool false | - | 2/0/0/0 | CVAR, VRReplay_OnBuildTiccmd | ALIVE |
| vr_replay_play | R | src/playsim/p_vrreplay.cpp:29 | String "" | - | 5/0/0/0 | CVAR, VRReplay_IsPlaying | ALIVE |
| vr_replay_record | R | src/playsim/p_vrreplay.cpp:30 | String "" | - | 3/0/0/0 | VRReplay_OnBuildTiccmd | ALIVE |
| vr_synclog | R | src/playsim/p_vrsynclog.cpp:38 | String "" | - | 4/0/0/0 | CVAR, VRSyncLog_OnTick | ALIVE |
| vr_synclog_actors | R | src/playsim/p_vrsynclog.cpp:39 | Bool false | - | 1/0/0/0 | VRSyncLog_OnTick | ALIVE |
| vr_telemetry | R | src/playsim/p_vrtelemetry.cpp:32 | String "" | - | 5/0/0/0 | CVAR, VRTelemetry_Tic | ALIVE |
| vr_torso_diag | R | src/playsim/p_vrtorso.cpp:30 | Bool false | - | 1/0/0/0 | VRTorso_Update | ALIVE |
| r_jointpose_test | R | src/r_data/model_jointdrive.cpp:93 | String "" | - | 1/0/0/0 | ApplyTestChannel | ALIVE |
| r_jointpose_test_axis | R | src/r_data/model_jointdrive.cpp:99 | Int 0 | - | 1/0/0/0 | ApplyTestChannel | ALIVE |
| r_jointpose_test_deg | R | src/r_data/model_jointdrive.cpp:98 | Float 0.f | - | 1/0/0/0 | ApplyTestChannel | ALIVE |
| vr_avatar_mod_hands_ghost | R | src/r_data/models.cpp:2780 | Bool false | - | 2/1/0/0 | ModHandCapture, ModHandTakeOver | ALIVE |
| vr_avatar_mod_hands_scale | R | src/r_data/models.cpp:2765 | Float 0.72f | A | 1/0/0/0 | ModHandScale | ALIVE |
| vr_avatar_mod_hands_size | R | src/r_data/models.cpp:2760 | Bool true | A | 2/0/0/0 | ModHandBaseAndSize, ModHandSizeOnly | ALIVE |
| vr_followhand_sim | R | src/r_data/models.cpp:432 | Bool true | A | 1/0/0/0 | VR_SimHandFrame | ALIVE |
| vr_gun_anchor | R | src/r_data/models.cpp:303 | Bool false | A | 3/0/0/0 | VKOpenXRDeviceMode::UpdateControllerState, VR_GripAnchorFrame | ALIVE |
| vr_gun_size | R | src/r_data/models.cpp:2684 | Float 1.0f | A | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_gunfit_debug | R | src/r_data/models.cpp:2777 | Bool false | - | 2/0/0/0 | ModHandCapture, VRModHand_DebugWrist | ALIVE |
| vr_place_debug | R | src/r_data/models.cpp:608 | Bool false | - | 1/0/0/0 | FSpriteModelFrame::ObjectToWorldMatrix | ALIVE |
| vr_pose_debug | P | src/r_data/models.cpp:275 | Bool false | - | 2/0/0/0 | CalcModelOverrides, ProcessModelFrame | ALIVE |
| vr_spatialreport | P | src/r_data/models.cpp:518 | Bool false | - | 1/0/0/0 | RenderHUDModel | ALIVE |
| vr_surf_debug | R | src/r_data/models.cpp:285 | Bool false | - | 4/0/0/0 | RenderModelFrame | ALIVE |
| vr_validate | P | src/r_data/models.cpp:517 | Bool false | - | 1/0/0/0 | ValidateHudModel | ALIVE |
| vr_voxel_debug | P | src/r_data/models.cpp:600 | Bool true | AG | 2/0/0/0 | FSpriteModelFrame::ObjectToWorldMatrix, FindModelFrame | ALIVE |
| vr_voxel_rollaxis | P | src/r_data/models.cpp:595 | Float -1.f | AG | 2/0/0/2 | FSpriteModelFrame::ObjectToWorldMatrix | ALIVE |
| vr_weapons_world | R | src/r_data/models.cpp:2674 | Bool true | A | 2/0/0/0 | HWDrawInfo::DrawWorldPlayerSprites, RenderHUDModel | ALIVE |
| vr_arm_solver | R | src/r_data/vr_armsolver.cpp:40 | Int 0 | A | 1/0/0/0 | VRArmSolver_Current | ALIVE |
| vr_arm_clear_torso | R | src/r_data/vr_avatar_pose.cpp:231 | Bool true | A | 1/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_avatar_armik | R | src/r_data/vr_avatar_pose.cpp:227 | Bool true | A | 1/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_avatar_armk_ref | R | src/r_data/vr_avatar_pose.cpp:140 | Bool false | A | 1/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_avatar_bone_diag | R | src/r_data/vr_avatar_pose.cpp:118 | Bool false | A | 3/0/0/0 | ApplyTwistLeaves, VRAvatar_Pose | ALIVE |
| vr_avatar_hand_diag | R | src/r_data/vr_avatar_pose.cpp:76 | Bool false | A | 1/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_avatar_head_invert_pitch | R | src/r_data/vr_avatar_pose.cpp:216 | Bool false | A | 1/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_avatar_head_invert_roll | R | src/r_data/vr_avatar_pose.cpp:217 | Bool false | A | 1/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_avatar_keep_clavicle | R | src/r_data/vr_avatar_pose.cpp:201 | Bool true | A | 1/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_avatar_ownbody | R | src/r_data/vr_avatar_pose.cpp:156 | Int 1 | A | 4/0/0/0 | RenderHUDModel, VRAvatar_Pose | ALIVE |
| vr_avatar_reach_max | R | src/r_data/vr_avatar_pose.cpp:87 | Float 1.0f | A | 1/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_body_ab | R | src/r_data/vr_avatar_pose.cpp:178 | Bool false | - | 1/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_body_diag | R | src/r_data/vr_avatar_pose.cpp:129 | Bool false | - | 2/0/0/0 | VRAvatar_Pose, VRBody_LogLevel | ALIVE |
| vr_body_hide_point | R | src/r_data/vr_avatar_pose.cpp:163 | Int 0 | A | 1/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_gun_anchor_diag | R | src/r_data/vr_avatar_pose.cpp:113 | Bool false | - | 1/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_gun_anchor_hand | R | src/r_data/vr_avatar_pose.cpp:108 | Bool false | - | 3/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_gun_anchor_qw | R | src/r_data/vr_avatar_pose.cpp:1119 | Float 1.0f | A | 2/0/0/0 | HandFromGripMatrix, VRAvatar_Pose | ALIVE |
| vr_gun_anchor_qx | R | src/r_data/vr_avatar_pose.cpp:1116 | Float 0.0f | A | 2/0/0/0 | HandFromGripMatrix, VRAvatar_Pose | ALIVE |
| vr_gun_anchor_qy | R | src/r_data/vr_avatar_pose.cpp:1117 | Float 0.0f | A | 2/0/0/0 | HandFromGripMatrix, VRAvatar_Pose | ALIVE |
| vr_gun_anchor_qz | R | src/r_data/vr_avatar_pose.cpp:1118 | Float 0.0f | A | 2/0/0/0 | HandFromGripMatrix, VRAvatar_Pose | ALIVE |
| vr_gun_full_reach | R | src/r_data/vr_avatar_pose.cpp:238 | Bool true | A | 1/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_hand_wall | R | src/r_data/vr_avatar_pose.cpp:236 | Bool true | A | 1/0/0/0 | VRAvatar_Pose | ALIVE |
| vr_body | R | src/r_data/vr_body.cpp:102 | Int 2 | A | 8/0/0/0 | VRBody_RegisterSource, VRBody_Solve | ALIVE |
| vr_body_fade_s | R | src/r_data/vr_body.cpp:110 | Float 0.2f | - | 1/0/0/0 | VRBody_GetSource | ALIVE |
| vr_body_fake | R | src/r_data/vr_body.cpp:112 | Bool false | - | 2/0/0/0 | CCMD, CVAR | ALIVE |
| vr_body_fake_chest_pitch | R | src/r_data/vr_body.cpp:121 | Float 0.f | - | 1/0/0/0 | CVAR | ALIVE |
| vr_body_fake_chest_yaw | R | src/r_data/vr_body.cpp:122 | Float 0.f | - | 1/0/0/0 | CVAR | ALIVE |
| vr_body_fake_foot_l_x | R | src/r_data/vr_body.cpp:123 | Float 0.f | - | 1/0/0/0 | CVAR | ALIVE |
| vr_body_fake_foot_l_y | R | src/r_data/vr_body.cpp:124 | Float 0.f | - | 1/0/0/0 | CVAR | ALIVE |
| vr_body_fake_foot_r_x | R | src/r_data/vr_body.cpp:125 | Float 0.f | - | 1/0/0/0 | CVAR | ALIVE |
| vr_body_fake_foot_r_y | R | src/r_data/vr_body.cpp:126 | Float 0.f | - | 1/0/0/0 | CVAR | ALIVE |
| vr_body_fake_pelvis | R | src/r_data/vr_body.cpp:117 | Float 0.55f | - | 1/0/0/0 | CVAR | ALIVE |
| vr_body_fake_pitch | R | src/r_data/vr_body.cpp:118 | Float 0.f | - | 1/0/0/0 | CVAR | ALIVE |
| vr_body_fake_roll | R | src/r_data/vr_body.cpp:119 | Float 0.f | - | 1/0/0/0 | CVAR | ALIVE |
| vr_body_fake_yaw | R | src/r_data/vr_body.cpp:120 | Float 0.f | - | 1/0/0/0 | CVAR | ALIVE |
| vr_body_lagdiag | R | src/r_data/vr_body.cpp:131 | Int 0 | - | 2/0/0/0 | VRBody_LagDiag, VRBody_SolveLegs | ALIVE |
| vr_seat_debug | R | src/r_data/vr_body.cpp:71 | Bool false | - | 1/0/0/0 | VRBody_SolveLegs | ALIVE |
| vr_seat_fake | R | src/r_data/vr_body.cpp:72 | Bool false | - | 2/0/0/0 | VRBody_LagDiag, VRBody_SolveLegs | ALIVE |
| vr_seat_pose | R | src/r_data/vr_body.cpp:70 | Bool true | A | 2/0/0/0 | VRBody_LagDiag | ALIVE |
| vr_body_anchor | R | src/r_data/vr_bodyinfer_input.cpp:1038 | Int 0 | A | 1/0/0/0 | VRBodyInfer_AnchorActive | ALIVE |
| vr_body_trackers | R | src/r_data/vr_bodyinfer_input.cpp:230 | Bool false | A | 2/0/0/0 | TrackerSource, VRBodyInfer_Frame | ALIVE |
| vr_bodyinfer | R | src/r_data/vr_bodyinfer_input.cpp:723 | Bool false | A | 17/0/0/0 | InferredSource, VRBodyInfer_AnchorActive | ALIVE |
| vr_bodyinfer_diag | R | src/r_data/vr_bodyinfer_input.cpp:55 | Bool false | - | 5/0/0/0 | CCMD, VRBodyInfer_Frame | ALIVE |
| vr_bodyinfer_fakedrop | R | src/r_data/vr_bodyinfer_input.cpp:727 | Float 0.f | - | 1/0/0/0 | InferredSource | ALIVE |
| vr_bodyinfer_fakelean | R | src/r_data/vr_bodyinfer_input.cpp:732 | Float 0.f | - | 1/0/0/0 | InferredSource | ALIVE |
| vr_bodyinfer_yawcheck | R | src/r_data/vr_bodyinfer_input.cpp:128 | Bool false | - | 1/0/0/0 | VRBodyInfer_Frame | ALIVE |
| vr_fit_neck | R | src/r_data/vr_bodyinfer_input.cpp:60 | String "" | A | 4/2/0/0 | CCMD, VRBodyInfer_FittedPivot | ALIVE |
| vr_grasp | R | src/r_data/vr_grasp.cpp:27 | Bool true | A | 5/0/0/0 | VRGrasp_NoteHeldPart, VRGrasp_SolveHand | ALIVE |
| vr_grasp_diag | R | src/r_data/vr_grasp.cpp:29 | Bool false | - | 2/0/0/0 | VRGrasp_SolveHand | ALIVE |
| vr_grasp_sink | R | src/r_data/vr_grasp.cpp:28 | Float 0.15f | A | 1/0/0/0 | VRGrasp_SolveHand | ALIVE |
| vr_avatar_mod_hands | R | src/r_data/vr_rig.cpp:872 | Bool true | A | 6/0/0/0 | ModHandAnchor, ModHandBaseAndSize | ALIVE |
| vr_avatar_scale_diag | R | src/r_data/vr_rig.cpp:1640 | Bool false | A | 1/0/0/0 | BuildBodyFrameMatrix | ALIVE |
| vr_body_simtorso | R | src/r_data/vr_rig.cpp:1372 | Bool true | A | 1/0/0/0 | VRRig_BodyYaw | ALIVE |
| vr_body_turnahead | R | src/r_data/vr_rig.cpp:1373 | Bool true | - | 1/0/0/0 | VRRig_BodyYaw | ALIVE |
| vr_body_yaw_deadzone | R | src/r_data/vr_rig.cpp:1369 | Float 45.f | A | 2/0/0/0 | AddPlayerLengths, VRRig_BodyYaw | ALIVE |
| vr_body_yaw_follow | R | src/r_data/vr_rig.cpp:1370 | Float 6.f | A | 2/0/0/0 | AddPlayerLengths, VRRig_BodyYaw | ALIVE |
| vr_body_yaw_maxneck | R | src/r_data/vr_rig.cpp:1371 | Float 100.f | A | 3/0/0/0 | AddPlayerLengths, VRRig_BodyYaw | ALIVE |
| vr_gun_anchor_rake | R | src/r_data/vr_rig.cpp:836 | Bool true | A | 1/0/0/0 | VRRig_GripFrame | ALIVE |
| vr_rig_debug | R | src/r_data/vr_rig.cpp:539 | Int 0 | - | 2/0/0/0 | VRRig_DrawDebug | ALIVE |
| vr_rig_fake | R | src/r_data/vr_rig.cpp:551 | Bool false | - | 4/0/0/0 | FVRRig::Recompute, VRRig_DescribeNoGrip | ALIVE |
| vr_rig_fake_l_p | R | src/r_data/vr_rig.cpp:555 | Float 0.f | - | 1/0/0/0 | FVRRig::VirtualGrip | ALIVE |
| vr_rig_fake_l_r | R | src/r_data/vr_rig.cpp:557 | Float 0.f | - | 1/0/0/0 | FVRRig::VirtualGrip | ALIVE |
| vr_rig_fake_l_x | R | src/r_data/vr_rig.cpp:552 | Float -10.f | - | 1/0/0/0 | FVRRig::VirtualGrip | ALIVE |
| vr_rig_fake_l_y | R | src/r_data/vr_rig.cpp:553 | Float 20.f | - | 1/0/0/0 | FVRRig::VirtualGrip | ALIVE |
| vr_rig_fake_l_yw | R | src/r_data/vr_rig.cpp:556 | Float 0.f | - | 1/0/0/0 | FVRRig::VirtualGrip | ALIVE |
| vr_rig_fake_l_z | R | src/r_data/vr_rig.cpp:554 | Float -8.f | - | 1/0/0/0 | FVRRig::VirtualGrip | ALIVE |
| vr_rig_fake_r_p | R | src/r_data/vr_rig.cpp:561 | Float 0.f | - | 1/0/0/0 | FVRRig::VirtualGrip | ALIVE |
| vr_rig_fake_r_r | R | src/r_data/vr_rig.cpp:563 | Float 0.f | - | 1/0/0/0 | FVRRig::VirtualGrip | ALIVE |
| vr_rig_fake_r_x | R | src/r_data/vr_rig.cpp:558 | Float 10.f | - | 1/0/0/0 | FVRRig::VirtualGrip | ALIVE |
| vr_rig_fake_r_y | R | src/r_data/vr_rig.cpp:559 | Float 20.f | - | 1/0/0/0 | FVRRig::VirtualGrip | ALIVE |
| vr_rig_fake_r_yw | R | src/r_data/vr_rig.cpp:562 | Float 0.f | - | 1/0/0/0 | FVRRig::VirtualGrip | ALIVE |
| vr_rig_fake_r_z | R | src/r_data/vr_rig.cpp:560 | Float -8.f | - | 1/0/0/0 | FVRRig::VirtualGrip | ALIVE |
| vr_rig_hold_weapons | R | src/r_data/vr_rig.cpp:1303 | Bool false | A | 1/0/0/0 | VRRig_ObjectToWorld | ALIVE |
| vr_slime_markers | R | src/r_data/vr_rig.cpp:541 | Bool false | - | 1/0/0/0 | VRRig_DrawDebug | ALIVE |
| vr_stand_eye_auto | R | src/r_data/vr_rig.cpp:425 | Bool true | A | 1/0/0/0 | VRRig_StandEyeTic | ALIVE |
| vr_openvr_menu_pointer_pitch_bias | P | src/rendering/gl/stereo3d/gl_openvr.cpp:231 | Float 0.0f | AG | 1/0/0/2 | PostGuiWheelEvent | DORMANT (GL/OpenVR code only) |
| vr_openvr_menu_pointer_tip_offset | P | src/rendering/gl/stereo3d/gl_openvr.cpp:232 | Float 0.035f | AG | 1/0/0/2 | PostGuiWheelEvent | DORMANT (GL/OpenVR code only) |
| r_debris_sounds | R | src/rendering/hwrenderer/hw_debrislanding.cpp:39 | Bool true | AG | 6/0/0/1 | CUSTOM_CVARD, DebrisLandingSounds::BeginBurst | ALIVE |
| r_debris_sounds_volume | R | src/rendering/hwrenderer/hw_debrislanding.cpp:41 | Float 1.f | AG | 4/0/0/1 | CUSTOM_CVARD, DebrisLandingSounds::Report | ALIVE |
| r_debris | R | src/rendering/hwrenderer/hw_debrispool.cpp:55 | Bool true | AG | 7/0/0/1 | CUSTOM_CVARD, DebrisPool::PrepareFrame | ALIVE |
| r_debris_life | R | src/rendering/hwrenderer/hw_debrispool.cpp:59 | Float 60.f | AG | 6/0/0/1 | CCMD, CUSTOM_CVARD | ALIVE |
| r_debris_pool | R | src/rendering/hwrenderer/hw_debrispool.cpp:66 | Int DEBRIS_POOL_DEFAUL.. | AG | 6/0/0/1 | CUSTOM_CVARD, DebrisPool::PrepareFrame | ALIVE |
| r_debris_test | R | src/rendering/hwrenderer/hw_debrispool.cpp:74 | Bool false | G | 6/0/0/1 | CUSTOM_CVARD, DebrisPool::PrepareFrame | ALIVE |
| gl_light_distance_cull_cache | P | src/rendering/hwrenderer/hw_dynlightdata.cpp:41 | Bool false | AG | 2/0/0/2 | AppendBenchmarkHeader, gl_IsDistanceCulled | ALIVE |
| gl_light_model_dedupe_cache | P | src/rendering/hwrenderer/hw_dynlightdata.cpp:42 | Bool false | AG | 4/0/0/2 | AppendBenchmarkHeader, hw_GetDynModelLight | ALIVE |
| gl_light_pos_relative_cache | P | src/rendering/hwrenderer/hw_dynlightdata.cpp:44 | Bool false | AG | 2/0/0/2 | AppendBenchmarkHeader, gl_GetLightPosRelative | ALIVE |
| gl_light_spot_cache | P | src/rendering/hwrenderer/hw_dynlightdata.cpp:43 | Bool false | AG | 2/0/0/2 | AppendBenchmarkHeader, GetSpotlightShaderParams | ALIVE |
| gl_light_weapons | P | src/rendering/hwrenderer/hw_dynlightdata.cpp:40 | Bool false | AG | 4/0/0/4 | AppendBenchmarkHeader, HWDrawInfo::DrawPSprite | ALIVE |
| r_effectlights | R | src/rendering/hwrenderer/hw_effectlights.cpp:42 | Bool true | AG | 6/0/0/1 | CUSTOM_CVARD, DebrisPool::ExpandBurst | ALIVE |
| r_effectlights_distance | R | src/rendering/hwrenderer/hw_effectlights.cpp:65 | Float 1536.f | AG | 4/0/0/1 | CUSTOM_CVARD, EffectLights::BeginFrame | ALIVE |
| r_effectlights_max | R | src/rendering/hwrenderer/hw_effectlights.cpp:44 | Int 1024 | AG | 3/0/0/1 | CUSTOM_CVARD, EffectLights::PoolCap | ALIVE |
| r_effectlights_perbin | R | src/rendering/hwrenderer/hw_effectlights.cpp:59 | Int 16 | AG | 3/0/0/1 | CUSTOM_CVARD, EffectLights::PrepareFrame | ALIVE |
| r_effectlights_quality | R | src/rendering/hwrenderer/hw_effectlights.cpp:53 | Int 2 | AG | 3/0/0/1 | CUSTOM_CVARD, EffectLights::PrepareFrame | ALIVE |
| r_effectlights_test | R | src/rendering/hwrenderer/hw_effectlights.cpp:74 | Int 0 | G | 4/0/0/1 | CUSTOM_CVARD, EffectLights::BeginFrame | ALIVE |
| r_effectlights_walls | R | src/rendering/hwrenderer/hw_effectlights.cpp:71 | Bool true | AG | 3/0/0/1 | CUSTOM_CVARD, EffectLights::BeginFrame | ALIVE |
| r_emissivevolumes | R | src/rendering/hwrenderer/hw_emissivevolumes.cpp:47 | Int 3 | AG | 5/0/0/1 | CUSTOM_CVARD, EmissiveVolumeDefinitionEnabled | ALIVE |
| r_emissivevolumes_brightness | R | src/rendering/hwrenderer/hw_emissivevolumes.cpp:78 | Float 1.f | AG | 3/0/0/1 | CUSTOM_CVARD, EmissiveVolumes::CurrentSettings | ALIVE |
| r_emissivevolumes_length | R | src/rendering/hwrenderer/hw_emissivevolumes.cpp:55 | Int 0 | AG | 3/0/0/1 | CUSTOM_CVARD, EmissiveVolumes::CurrentSettings | ALIVE |
| r_emissivevolumes_light | R | src/rendering/hwrenderer/hw_emissivevolumes.cpp:76 | Bool true | AG | 3/0/0/1 | CUSTOM_CVARD, EmissiveVolumes::BeginFrame | ALIVE |
| r_emissivevolumes_max | R | src/rendering/hwrenderer/hw_emissivevolumes.cpp:69 | Int 128 | AG | 3/0/0/1 | CUSTOM_CVARD, EmissiveVolumes::BeginFrame | ALIVE |
| r_emissivevolumes_motion | R | src/rendering/hwrenderer/hw_emissivevolumes.cpp:63 | Int 0 | AG | 3/0/0/1 | CUSTOM_CVARD, EmissiveVolumes::CurrentSettings | ALIVE |
| r_emissivevolumes_test | R | src/rendering/hwrenderer/hw_emissivevolumes.cpp:85 | Int 0 | G | 8/0/0/1 | CUSTOM_CVARD, EmissiveVolumes::BeginFrame | ALIVE |
| r_exposureimpulse | R | src/rendering/hwrenderer/hw_exposureimpulse.cpp:43 | Bool false | AG | 3/0/0/1 | CUSTOM_CVARD, ExposureImpulses::BeginFrame | ALIVE |
| r_exposureimpulse_cap | R | src/rendering/hwrenderer/hw_exposureimpulse.cpp:64 | Float 0.6f | AG | 3/0/0/1 | CUSTOM_CVARD, ExposureImpulses::BeginFrame | ALIVE |
| r_exposureimpulse_comfort | R | src/rendering/hwrenderer/hw_exposureimpulse.cpp:55 | Bool false | AG | 3/0/0/1 | CUSTOM_CVARD, ExposureImpulses::BeginFrame | ALIVE |
| r_exposureimpulse_debug | R | src/rendering/hwrenderer/hw_exposureimpulse.cpp:87 | Bool false | G | 2/0/0/0 | ExposureImpulses::BeginFrame | ALIVE |
| r_exposureimpulse_holdbeams | R | src/rendering/hwrenderer/hw_exposureimpulse.cpp:78 | Bool true | AG | 3/0/0/1 | CUSTOM_CVARD, ExposureImpulses::BeginFrame | ALIVE |
| r_exposureimpulse_look | R | src/rendering/hwrenderer/hw_exposureimpulse.cpp:47 | Int 0 | AG | 3/0/0/1 | CUSTOM_CVARD, ExposureImpulses::BeginFrame | ALIVE |
| r_exposureimpulse_recovery_scale | R | src/rendering/hwrenderer/hw_exposureimpulse.cpp:70 | Float 1.f | AG | 3/0/0/1 | CUSTOM_CVARD, ExposureImpulses::BeginFrame | ALIVE |
| r_exposureimpulse_strength | R | src/rendering/hwrenderer/hw_exposureimpulse.cpp:57 | Float 1.f | AG | 3/0/0/1 | CUSTOM_CVARD, ExposureImpulses::BeginFrame | ALIVE |
| r_exposureimpulse_test | R | src/rendering/hwrenderer/hw_exposureimpulse.cpp:81 | Int 0 | G | 3/0/0/1 | CUSTOM_CVARD, ExposureImpulses::BeginFrame | ALIVE |
| r_particlecollision | R | src/rendering/hwrenderer/hw_levelfield.cpp:43 | Bool true | AG | 5/0/0/1 | CUSTOM_CVARD, DebrisLandingSounds::BeginBurst | ALIVE |
| r_particlecollision_quality | R | src/rendering/hwrenderer/hw_levelfield.cpp:46 | Int LEVEL_FIELD_QUALIT.. | AG | 3/0/0/1 | CUSTOM_CVARD, LevelField::PrepareFrame | ALIVE |
| r_particlecollision_test | R | src/rendering/hwrenderer/hw_levelfield.cpp:53 | Bool false | G | 3/0/0/1 | CUSTOM_CVARD, LevelField::PrepareFrame | ALIVE |
| r_smoke_darkness | R | src/rendering/hwrenderer/hw_smokevolume.cpp:53 | Bool true | AG | 2/0/0/1 | CUSTOM_CVARD, SmokeVolume::UpdateSurfaceLight | ALIVE |
| r_smoke_light_quality | R | src/rendering/hwrenderer/hw_smokevolume.cpp:45 | Int SMOKE_LIGHT_QUALIT.. | AG | 5/0/0/1 | CUSTOM_CVARD, SmokeVolume::PrepareLight | ALIVE |
| r_smoke_surfaceglow | R | src/rendering/hwrenderer/hw_smokevolume.cpp:50 | Bool true | AG | 2/0/0/1 | CUSTOM_CVARD, SmokeVolume::UpdateSurfaceLight | ALIVE |
| r_damage | R | src/rendering/hwrenderer/hw_surfacedamage.cpp:45 | Bool true | AG | 5/0/0/1 | CUSTOM_CVARD, SurfaceDamage::PrepareFrame | ALIVE |
| r_damage_debug | R | src/rendering/hwrenderer/hw_surfacedamage.cpp:73 | Bool false | G | 3/0/0/1 | SurfaceDamage::PrepareFrame, SurfaceDamage::Report | ALIVE |
| r_damage_depth_scale | R | src/rendering/hwrenderer/hw_surfacedamage.cpp:62 | Float 1.f | AG | 4/0/0/1 | CUSTOM_CVARD, SurfaceDamage::PrepareFrame | ALIVE |
| r_damage_heat | R | src/rendering/hwrenderer/hw_surfacedamage.cpp:54 | Bool true | AG | 5/0/0/1 | CUSTOM_CVARD, SurfaceDamage::PrepareFrame | ALIVE |
| r_damage_heat_scale | R | src/rendering/hwrenderer/hw_surfacedamage.cpp:67 | Float 1.f | AG | 4/0/0/1 | CUSTOM_CVARD, SurfaceDamage::PrepareFrame | ALIVE |
| r_damage_memory | R | src/rendering/hwrenderer/hw_surfacedamage.cpp:48 | Int SURFACE_DAMAGE_MEM.. | AG | 5/0/0/1 | CUSTOM_CVARD, SurfaceDamage::PrepareFrame | ALIVE |
| r_damage_soot_scale | R | src/rendering/hwrenderer/hw_surfacedamage.cpp:57 | Float 1.f | AG | 4/0/0/1 | CUSTOM_CVARD, SurfaceDamage::PrepareFrame | ALIVE |
| r_damage_test | R | src/rendering/hwrenderer/hw_surfacedamage.cpp:78 | Bool false | G | 3/0/0/1 | CUSTOM_CVARD, SurfaceDamage::PrepareFrame | ALIVE |
| gl_bsp_wall_batch_size | P | src/rendering/hwrenderer/scene/hw_bsp.cpp:55 | Int 96 | AG | 2/0/0/4 | GetWallBatchSize | ALIVE |
| gl_bsp_worker_sky_mainthread | P | src/rendering/hwrenderer/scene/hw_bsp.cpp:54 | Bool false | AG | 1/0/0/4 | ComputeSectorMainThreadWallPath | ALIVE |
| gl_bsp_worker_threads | P | src/rendering/hwrenderer/scene/hw_bsp.cpp:47 | Int 1 | AGI | 3/0/0/4 | HWDrawInfo::RenderBSP, HWDrawInfo::StartWallWorkersIfNeeded | ALIVE |
| gl_line_distance_cull | P | src/rendering/hwrenderer/scene/hw_bsp.cpp:449 | Float 4000.0 | AG | 3/1/0/5 | AppendBenchmarkHeader, CCMD | ALIVE |
| fog_dynlights_max | R | src/rendering/hwrenderer/scene/hw_drawinfo.cpp:1781 | Int 8 | AG | 1/0/0/0 | FillFogDynLights | ALIVE |
| r_beam_interpolate | P | src/rendering/hwrenderer/scene/hw_drawinfo.cpp:123 | Bool true | AG | 4/0/0/0 | HWDrawInfo::StartScene, ResolveBeamLine | ALIVE |
| rs_bb_cullradius | P | src/rendering/hwrenderer/scene/hw_drawinfo.cpp:3544 | Float 0.0f | AG | 1/0/0/0 | HWDrawInfo::DispatchBillboards | ALIVE |
| rs_bb_maxpanels | P | src/rendering/hwrenderer/scene/hw_drawinfo.cpp:3543 | Int 0 | AG | 1/0/0/0 | HWDrawInfo::DispatchBillboards | ALIVE |
| vol_beam_axisfade | P | src/rendering/hwrenderer/scene/hw_drawinfo.cpp:98 | Float 0.85f | AG | 1/0/0/0 | HWDrawInfo::SetupVolumetricBeam | ALIVE |
| vol_beam_axisfade_reach | R | src/rendering/hwrenderer/scene/hw_drawinfo.cpp:107 | Float 12.0f | AG | 1/0/0/0 | HWDrawInfo::SetupVolumetricBeam | ALIVE |
| vol_beam_debug | R | src/rendering/hwrenderer/scene/hw_drawinfo.cpp:113 | Bool false | - | 2/0/0/0 | HWDrawInfo::SetupVolumetricBeam, HWDrawInfo::StartScene | ALIVE |
| vol_beam_quality | P | src/rendering/hwrenderer/scene/hw_drawinfo.cpp:87 | Int 24 | AG | 1/0/0/0 | HWDrawInfo::SetupVolumetricBeam | ALIVE |
| gl_max_vertices | P | src/rendering/hwrenderer/scene/hw_flats.cpp:53 | Int 0 | A | 2/0/0/2 | HWFlat::DrawSubsectors, HWWall::RenderWall | ALIVE |
| gl_max_portals | P | src/rendering/hwrenderer/scene/hw_portal.cpp:35 | Int -1 | A | 1/0/0/2 | HWDrawInfo::RenderPortal | ALIVE |
| gl_skydome | P | src/rendering/hwrenderer/scene/hw_sky.cpp:35 | Bool true | AG | 2/0/0/5 | HWSkyPortal::DrawContents | ALIVE |
| bb_flipu | P | src/rendering/hwrenderer/scene/hw_sprites.cpp:90 | Bool false | AG | 4/0/0/0 | HWSprite::ProcessBillboard | ALIVE |
| bb_scale | P | src/rendering/hwrenderer/scene/hw_sprites.cpp:96 | Float 1.0f | AG | 3/0/0/0 | BillboardQueryScale, HWSprite::ProcessBillboard | ALIVE |
| bb_sdffont | P | src/rendering/hwrenderer/scene/hw_sprites.cpp:107 | String "sdfmono" | AG | 6/0/0/0 | FSDFFontRoster::Roll, FSDFFontRoster::Slot | ALIVE |
| bb_tiltbias | P | src/rendering/hwrenderer/scene/hw_sprites.cpp:97 | Float 0.0f | AG | 4/0/0/0 | AimBillboard, HWSprite::ProcessBillboard | ALIVE |
| gl_sprite_decor_distance_cull | P | src/rendering/hwrenderer/scene/hw_sprites.cpp:1042 | Float 1200.0 | AG | 3/0/0/4 | AppendBenchmarkHeader, ShouldCullDecorRenderThing | ALIVE |
| gl_sprite_distance_cull | P | src/rendering/hwrenderer/scene/hw_sprites.cpp:1041 | Float 2000.0 | AG | 3/1/0/4 | AppendBenchmarkHeader, CCMD | ALIVE |
| r_transparentPlayerSprites | P | src/rendering/hwrenderer/scene/hw_weapon.cpp:124 | Bool true | A | 0/0/0/0 |  | DEAD: no reference besides definition |
| gl_fatItemWidth | P | src/rendering/r_utility.cpp:154 | Float 0.5f | A | 2/0/0/2 | HWDrawInfo::DrawPSprite | ALIVE |
| r_PlayerSprites3DMode | P | src/rendering/r_utility.cpp:153 | Int 1 | A | 4/0/0/2 | HWDrawInfo::DrawPSprite | ALIVE |
| r_hudflatoverlay | P | src/rendering/r_utility.cpp:158 | Float 1.0f | A | 3/0/0/0 | HWDrawInfo::PreparePlayerSprites2D | ALIVE |
| r_voxeldistance | P | src/rendering/r_utility.cpp:117 | Float 0.f | AG | 2/0/0/2 | HWSprite::Process | ALIVE |
| r_voxeldistance_band | R | src/rendering/r_utility.cpp:142 | Float 32.f | AG | 1/0/0/2 | HWSprite::Process | ALIVE |
| r_voxelpack_loaded | R | src/rendering/r_utility.cpp:151 | Bool false | X | 3/0/0/3 | R_DetectVoxelPack, VoxelsEffectiveMode | ALIVE |
| r_voxels_mode | R | src/rendering/r_utility.cpp:131 | Int -1 | AG | 3/0/0/2 | FindModelFrame, KeepVoxelWithoutSprite | ALIVE |
| r_visualstate_log | R | src/scripting/vmthunks.cpp:4794 | Bool false | - | 18/0/0/0 | ClearBloomLook, ClearBloomOverride | ALIVE |
| snd_hearingimpulse | R | src/sound/s_hearingimpulse.cpp:34 | Bool false | AG | 1/0/0/2 | HearingImpulses::Update | ALIVE |
| snd_hearingimpulse_debug | R | src/sound/s_hearingimpulse.cpp:57 | Bool false | G | 1/0/0/0 | HearingImpulses::Update | ALIVE |
| snd_hearingimpulse_muffle | R | src/sound/s_hearingimpulse.cpp:35 | Int 0 | AG | 1/0/0/2 | HearingImpulses::Update | ALIVE |
| snd_hearingimpulse_recovery_scale | R | src/sound/s_hearingimpulse.cpp:50 | Float 1.f | AG | 1/0/0/2 | HearingImpulses::Update | ALIVE |
| snd_hearingimpulse_ring | R | src/sound/s_hearingimpulse.cpp:40 | Float 0.35f | AG | 1/0/0/2 | HearingImpulses::Update | ALIVE |
| snd_hearingimpulse_strength | R | src/sound/s_hearingimpulse.cpp:45 | Float 1.f | AG | 1/0/0/2 | HearingImpulses::Update | ALIVE |
| snd_hearingimpulse_test | R | src/sound/s_hearingimpulse.cpp:55 | Bool false | G | 1/0/0/1 | HearingImpulses::Update | ALIVE |

## 9. Branch differences

### 9.1 renderer (`/home/claude/uzdxrema`, 175063a1)

Merge base with main `77e5ee3d`; 116 commits only on renderer, 76 only on main (V: `git log`). File-level (V, `ls-tree` diff):

- **Added (12)**: `rendering/hwrenderer/hw_levelresidency.cpp/.h` (5,152 lines), `hw_levelstatic.cpp/.h`, `common/rendering/hwrenderer/data/hw_levelbuffers.cpp/.h`, `rendering/hwrenderer/scene/hw_wallcache.cpp/.h`, `rendering/r_snapshot.cpp/.h`, `common/utility/rema_profile.h`, `wadsrc/static/shaders/glsl/levelmesh.vp`.
- **Missing vs main (5)**: `playsim/p_vrgrab.*`, `p_vrgunfit.*`, `p_vrslots.h` (avatar-lane work after the split) and 13 cvars (`vr_stand_eye, vr_seated, vr_main_left, vr_holster_reach, vr_seat_pose, vr_seat_debug, vr_seat_fake, vr_body_lagdiag, vr_stand_eye_auto, vr_body_turnahead, vr_gunfit_debug, vr_grab_debug, vr_slotload_debug`).
- **Added cvars (14)** with status (V, renderer tree):

| cvar | def (renderer) | default | readers | status |
|---|---|---|---|---|
| `r_newrenderer` | hw_levelresidency.cpp:137 | 0 | `LevelResidency_BeginScene` :4223 returns false at 0; build options :2103-2104 | DORMANT (static level path off) |
| `r_levelmesh_verify/sidescan/clock/drawall/foldback` | hw_levelresidency.cpp:140-155 | 0/0/0/0/1 | inside residency code only | DORMANT (only with r_newrenderer) |
| `r_wallcache` | scene/hw_wallcache.cpp:40 | 0 | hw_wallcache.cpp:315; hw_bsp.cpp:743 | DORMANT |
| `r_snapshot`, `r_snapshot_threads` | r_snapshot.cpp:28-29 | 0, 6 | `R_SnapshotWrite` :280 returns at 0 (called d_net.cpp:2493, 2557) | DORMANT |
| `r_renderthread` | r_snapshot.cpp:33 | false | callback refuses and resets; value never read | DEAD stub |
| `sim_prefetch` | playsim/dthinker.cpp:73 | 0 | dthinker.cpp:998, 1044, 1723 | DORMANT (previously measured "no effect at 8/24/48 lines", HANDOFF_RENDER_2026-11-11.md:270) |
| `sim_maxcatchup` | d_net.cpp:207 | **2** | d_net.cpp:2415: single player caps the tics run per frame to 2 | ALIVE behaviour change vs main (sim time dropped when a frame is slow) |
| `snd_evictretry`, `snd_evictbudget` | common/audio/sound/s_sound.cpp:1341, 1345 | 8, 64 | s_sound.cpp | ALIVE |

- Renderer-only CCMDs: `levelmeshstats, levelstats, simprofile, snapshotstats, soundchannels, ticprofile, wallcachestats`; missing: `vr_gear_play, vr_grab_selftest, vr_gunfit, vr_gunfit_check, vr_stand_eye_measure`.
- Entry hooks (V): `LevelResidency_BeginScene` (hw_bsp.cpp:1464), `LevelResidency_SkipWall` / `LevelResidency_FlatPlanes` in the BSP walk (hw_bsp.cpp:346-368, 661, 760, 1157-1158), `WallCache_Process` (hw_bsp.cpp:349, 763), `LevelResidency_Reset`/`WallCache_Reset`/`R_SnapshotInvalidate` at level setup (p_setup.cpp:641-643). Details: teardown area 09.
- Same as main on renderer for every finding in §7 (V: `InternCheckpointLabel` present, `vk_gpu_checkpoints` default true, `r_gpuparticles_soft` default 8, both engine event handlers registered, `P_VRLaserTraceTic` present) and for the dead functions checked in §8.1.

### 9.2 netplay (`/home/claude/uzdx_netplay`, 1c1805f8, older than main)

- Missing vs main (V): `common/engine/i_upnp.*`, `playsim/p_vrgear.*`, `p_vrgrab.*`, `p_vrgunfit.*`, `p_vrslots.h`, `p_vrtorso.*`, `r_data/vr_grasp.*`; no netplay-only files.
- 33 fewer cvars: `net_upnp, netplay_handtest, netplay_roomtest, sv_vr_shotclamp, vr_avatar_mod_hands*, vr_body_lagdiag, vr_body_simtorso, vr_body_turnahead, vr_bodyinfer_yawcheck, vr_gear_debug, vr_grab_debug, vr_grasp*, vr_gun_full_reach, vr_gun_size, vr_gunfit_debug, vr_holster_reach, vr_input_in_command, vr_main_left, vr_seat_*, vr_seated, vr_shotclamp_debug, vr_slotload_debug, vr_stand_eye*, vr_torso_diag, vr_weapons_world`.
- No `P_VRLaserTraceTic` (V: 0 hits in `playsim/p_user.cpp`). Checkpoint interning, `vk_gpu_checkpoints` default, soft particles default 8, the two engine event handlers: same as main (V).

### 9.3 selaco (`/home/claude/uzdx_selaco`, 7e111a3d; separate fork from `3511a341` "UZDXREMA 5.0.3", 111 commits of its own)

- **Removed relative to main (113 files, V)**: the avatar/IK/rig stack (`r_data/vr_*`, `model_handdrive`, `model_jointdrive`), all VR sim gear (`p_vrgear/grab/torso/hand/gunfit/slots/demo/replay/ghost/telemetry/synclog`), `p_affect`, `p_guard`, `p_beamoutline`, `p_floorregion`, effect lights, emissive volumes, exposure impulse, hearing impulse + `r_levelray`, shockwave core, effects governor, perf_track, perf bench/bench demo, hang watch, SolarXR/UPnP/websocket client, Studio models, VTF textures, data validators, `rs_*.zs`, and shaders `beamglow.glsl, func_neon.fp, emissivevolume.fp, exposureimpulse.fp, scenemask.fp, screentilemask.fp, sharedmarchwarp.fp, shockwaveoffset.fp, smoketemporal.fp, tonemapfilmic.fp`. 230 main cvars absent (largest groups: `vr_body*` 24, `vr_rig*` 20, `vr_avatar*` 14, `r_emissivevolumes*` 10, `vr_gun*` 10, `r_exposureimpulse*` 9, `r_effectlights*` 7, `r_gpuparticles*` 7, `r_shockwave*` 7, `snd_hearingimpulse*` 7).
- **Kept**: smoke volume, debris pool, level field, surface damage, GPU/mesh particles, drawn lines, sector planes, r_perflog, OpenXR device, billboards/sweep (P layer).
- **Added (V)**: `playsim/p_pooledparticles.cpp/.h` (2,157 lines; Selaco `ParticleDefinition` pool, thought in `P_Ticker` p_tick.cpp:576, drawn per subsector `RenderDefinedParticles` hw_bsp.cpp:387, 1050; "costs nothing" with no definition loaded per its header — I), `r_data/model_reach.cpp/.h/_math.h` (1,349 lines; `ModelDrawPose_Apply` called from models.cpp:3334, 3354 — this makes `LevelSolidQuery` ALIVE on selaco), `wadsrc/static/iwadover.txt`, `zscript/actors/standalone/stockstandins.zs`, `zscript/particledefinition.zs`; 15 cvars (`am_playerScale, g_viewbob, r_shadowquality, gl_link_playerlights, g_leveltilting, gl_selflighting, r_reachchain_debug, r_mirror_recursions, vr_hud_sway, vr_hud_sway_portable, vr_hud_sway_scale, vr_hud_sway_max, vr_two_handed_swap_inches, crosshairfilter, crosshairalpha`).
- Same as main for: checkpoint interning, `vk_gpu_checkpoints 1`, `r_gpuparticles_soft 8`, `OtherPlayerTagHandler`/`VRStabilizeSyncHandler` registration (V). No `P_VRLaserTraceTic`.

## 10. Alive additions ranked by default per-frame / per-tic cost (main, owner config, no mod effects active)

| Rank | Addition | Per | Cost class | Confidence |
|---|---|---|---|---|
| 1 | `main.fp` uber-shader (all fork features compiled into every world program) | fragment | branches + registers on every world fragment | I, medium |
| 2 | soft-particle scene pass restarts (once any GPU particle exists) | eye | 2 render-pass end/begin | V path, I cost |
| 3 | `vk_gpu_checkpoints` + PushGroup string interning | GPU group | NV checkpoint + 2 allocs + 2 set lookups | V path, I size |
| 4 | `P_VRLaserTraceTic` | player·tic | 2 × 8192-unit traces | V |
| 5 | engine event handlers `VRStabilizeSyncHandler`, `OtherPlayerTagHandler` | tic | 2 VM dispatches, MAXPLAYERS loop | V |
| 6 | VR sim latch/torso/gear/grab | player·tic | O(spots) + blockmap box | V |
| 7 | per-frame effect bookkeeping (§3.1, ~20 calls/frame + ~12/eye) | frame/eye | O(1)-O(128) each | V |
| 8 | per-tic effect bookkeeping (billboards, 64 stamps, 128 beams, drawn lines, 8 outlines, 9 queues) | tic | O(≤ a few hundred) | V |
| 9 | larger `AActor` / `FLevelLocals` | every actor walk | cache footprint | I |
| 10 | governor, VR clocks, hang watch | frame | O(1) | V |

## 11. Open questions and the measurements that settle them

1. **Size of the uber-shader tax.** Build a variant of `vk_shader.cpp` without the `SURFACE_DAMAGE`/`EFFECT_LIGHTS` defines and without the two prepended libraries (and a stock `main.fp`), same map/pose; compare `stat gpu` `scene.opaque` / `gpu.opaque.*` (with `r_perflog 1`) and `perf_track` `cost_ms`. Also dump SPIR-V and compare register counts.
2. **Checkpoint overhead.** `vk_gpu_checkpoints 0` vs 1 (restart between), `perf_track` `frame_ms`, `cost_ms`, `xr.app_gpu`; and a CPU profile of `PushGroup` (expect µs).
3. **Soft-particle pass breaks.** `r_perflog 1`, read `fx.depthread` per eye with any GPU particle alive; A/B `r_gpuparticles_soft 0`. Also confirm whether `Level->GpuParticleWritten` ever returns to 0 within a level (`g_levellocals.h` writers) — if not, the switch runs for the rest of the level after the first spawn.
4. **Laser trace cost and use.** Count `Trace` calls per tic (`P_VRLaserTraceTic`) and time with `ticprofile` (renderer) or a `cycle_t` around p_user.cpp:2011; check whether the mods read `LaserTraceTargetMain/Off` (not verifiable here: mod sources absent).
5. **Engine event handlers.** Time `WorldTick` dispatch with and without `OtherPlayerTagHandler`/`VRStabilizeSyncHandler` in `mapinfo/common.txt` (previously measured total for all handlers 2.5-3.7 ms/tic, HANDOFF_RENDER_2026-11-11.md:269).
6. **Avatar pose per eye.** Does `VRAvatar_Pose` (vr_avatar_pose.cpp:1422) re-solve for each eye and each draw of the avatar model? Count calls per frame with a counter; area 04 owns the answer.
7. **`AActor` growth.** Print `sizeof(AActor)` in stock 5.0.0 and main builds; `perf stat -e cache-misses` on a 30k-thinker map (owner's list).
8. **External use of the ~425 natives without in-tree callers.** Grep the RS/BD22 mod sources (not in any tree) for each name in §8.6 to find the truly dead API.
9. **Heatmap determinism.** If any play-scope script calls `Level.HeatmapAt`, the render-rate decay (hw_drawinfo.cpp:3233-3260) makes it frame-rate dependent; check mods, or move the decay to the tic.
10. **Build defines.** Confirm the owner's `build-dxr` CMake cache (`ENABLE_OPENVR`, `ENABLE_OPENXR`, `HAVE_GLES2`) — this document assumed `auto-setup-windows-vr.cmd`.
