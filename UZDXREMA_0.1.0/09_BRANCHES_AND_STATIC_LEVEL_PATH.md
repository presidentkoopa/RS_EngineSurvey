# 09 — The four branches, and the render lane's static level path

Teardown area 09. Trees read: `main` = `/home/claude/uzdx_main` (f182cbcc), `renderer` = `/home/claude/uzdxrema`
(175063a1), `netplay` = `/home/claude/uzdx_netplay` (1c1805f8), `selaco` = `/home/claude/uzdx_selaco` (7e111a3d).
All git comparisons were made in `/home/claude/uzdxrema` against `origin/*`. Unless a path says otherwise, Part B
line numbers are in the **renderer** tree.

Marks: **V** = VERIFIED (I read the code that does it). **I** = INFERRED (deduced; the basis is given). "prev. measured"
= a number copied from a handoff or plan file on `renderer`, not measured by me.

---

## Summary

1. `main` is the engine of record: every other branch lacks some of its work. `netplay` is a strict ancestor of
   `main` (0 commits of its own, 176 behind). `renderer` is `main` as of 77e5ee3d (2026-11-11) plus 116 commits
   (111 render-lane + 5 merges of `main`) at 175063a1, 76 behind `main` [xc: the 116 include the 5 merges; the
   live `origin/renderer` tip abb19af1 has 4 more docs-only teardown commits, 120 ahead, no src/wadsrc change]. `selaco` forked at "UZDXREMA 5.0.3" (2026-09-06): 104 commits of its own, 337 behind.
2. `renderer` changes 49 src/wadsrc files. 12 of them are new (level mesh, wall cache, snapshot, Tracy macros) [xc:
   was 13; `git diff --diff-filter=A origin/main...175063a1` lists 12]. It
   also changes defaults that do not depend on `r_newrenderer`: `sim_maxcatchup 2`, `snd_evictretry 8`,
   `snd_evictbudget 64`, a per-source sound-channel index, and new perf_track CSV columns. It also carries bug fixes
   that `main` lacks (D_Display console-drop guard, demo buffer growth).
3. With `r_newrenderer 0` (the default) the static path costs one cvar test a scene
   (`hw_levelresidency.cpp:4223`) plus one `WallCache_Process` call per processed wall (`hw_wallcache.cpp:315`). It
   also compiles one extra effect shader for every pass at startup and binds two 16-byte storage buffers. The old path
   changes in exactly one place: `HWWall::PutWall` without a draw info (`hw_walls.cpp:894`). That branch runs only
   for the wall workers (`gl_bsp_worker_threads > 1` and `gl_multithread`, and in VR also `vr_scene_multithread`,
   default false, `hw_bsp.cpp:1466-1468`), the wall cache, and the mesh build. Its effect on the worker path is to
   make it match the single-thread old path (the direct path already clears the flag on the shared `HWWall`), not
   to diverge from it [xc: added the multithread conditions and the direction of the change].
4. The static level path (`r_newrenderer 1..3`) builds, on the first frame, lasting GPU buffers for the opaque
   static wall parts (sized for each side's worst case; `BufferUsageType::Static`, GPU-only with a kept staging
   copy, not Vulkan "Persistent") and, at 3, index runs into the existing flat VBO. It also builds one 32-byte
   surface record per slot surface (1 or 2 per side, unused ones included) and per static flat plane, and one
   320-byte light record per distinct light key plus spares [xc: was "per part"; records are per slot surface].
5. Per frame, the static path still runs the whole BSP walk, the clipper and everything else the walk feeds. It
   replaces only `HWWall::Process` and `HWFlat::ProcessSector` for static sides and planes, and their per-part draw
   calls. The mesh draws only what the walk reached (`MakeSceneRanges`, `:4522`), so it depends on the walk.
6. New per-frame CPU work: a change pass at the first scene of each frame. Once a sim tic has passed, this pass
   compares every sector against a 264-byte shadow (`RunTicPass`, `:3681`). It also hashes each reached side's own
   fields once a frame (`:4371`), copies indices into a per-frame buffer, and makes one draw per material group that has reached indices
   [xc: empty groups are skipped, `:4680`].
7. Prev. measured on Sunder MAP15 (HANDOFF_RENDER_2026-11-11.md, 1112y): old path 70.1 fps, new path 60.8 fps.
   Walls CPU fell (2.56 → 1.05 ms) but the bsp clock rose (0.56 → 2.49 ms). The current path costs more than it
   saves; the plan's stage 3 (visibility) is where the saving was expected.
8. Half-built or dead: `R_SnapshotAcquire`/`Release` (the reader side of the snapshot) are never called, so
   `r_snapshot 1` writes snapshots nothing reads. `r_renderthread` is a stub that resets itself. Several exported
   helpers are never called (`WallCache_SectorHash`/`SideHash`/`SectorFastHash`, `HWWall::SetTierLight`,
   `LevelMeshBuffers::StorageCapacityBytes`). The wall cache (`r_wallcache`) is dormant and superseded by the mesh.
9. Memory: walls get worst-case slots per side. Static buffers keep a host staging copy of the same size
   (`vk_hwbuffer.cpp:73-97`). The CPU keeps full copies of the index arrays, and the per-frame index buffer holds 2×
   all static indices. Prev. measured GPU-side totals: 157.7 MB (Abandon) and 180.0 MB (Sunder). The staging copies
   are not counted in those totals.
10. Correctness hazards found in code:
    - Camera-texture scenes are not excluded, so they consume one of the two per-frame index regions. A VR eye can
      then fall back to drawing the whole static mesh (**I**).
    - The light patch reproduces `Process`'s light with duplicated functions and an observed /fp:fast rounding
      form. Only `r_levelmesh_verify` guards that duplication.
    - Eviction flags are frame-global while decisions are made per scene.

---

## Part A — The four branches

### A.1 Topology (V, from `git rev-list`/`merge-base`)

| pair | merge-base | only on left | only on right | src+wadsrc files differing (2-dot) |
|---|---|---|---|---|
| main / renderer | 77e5ee3d ([SYNCGEAR] vr_synclog, 2026-11-11) | 76 | 116 (incl. 5 merges of main) | 78 (46 renderer-only, 29 main-only, 3 both; i.e. 49 renderer-touched, 32 main-touched) [xc: 49+32+3 ≠ 78; the 49 and 32 include the 3] |
| main / netplay | 1c1805f8 = netplay tip | 176 | 0 | 63 (all main-own) |
| main / selaco | 3511a341 ("UZDXREMA 5.0.3", 2026-09-06) | 337 | 104 | 389 (selaco-own 330 touched, main-own 443 touched; many identical cherry-picks) |
| renderer / netplay | 1c1805f8 = netplay tip | 216 | 0 | 93 (42 render-lane, 44 main netplay→77e5ee3d, 7 both) |

- **main**: the engine of record. It is the only branch with `p_vrgrab.cpp`, `p_vrgunfit.cpp`, `p_vrslots.h`, the
  rebuilt holster core, [SLOTLOAD], [NETSNAP], [STANDEYE], [LAGA]/[STICKHIPS] and [CLAMPTIC] (V, file presence
  per branch). `CLAUDE.md` says the same; the code agrees.
- **renderer**: the render lane's work branch (ROADMAP step 4, started early). It is main 77e5ee3d plus the static
  level mesh, wall cache, sim snapshot, Tracy zones, sim/sound fixes from profiling, and many docs/bridge files
  (131 files changed outside src/wadsrc: docs/plans, tools/claude_bridge, libraries/tracy).
- **netplay**: a frozen snapshot of the Oct-31 netplay milestone (tip "Milestone 2026-10-31: netplay…"). It has
  `vr_body`, `vr_rig`, the netplay protocol, and none of the later holsters/torso/grab work. Nothing was committed
  to it after the milestone; it is useful only as a historical reference or regression baseline.
- **selaco**: a separate fork that makes the engine run the commercial game Selaco. It adds `-nostockactors`,
  IWADOVER, pooled particles, DDS, the Globals store, model reach, and level solid queries. It shares older ports of
  REMA render features (smoke, debris, surface damage, bloom), and it lacks all of main's body/netplay work since
  September.

### A.2 main vs renderer

#### A.2.1 What renderer adds (`git diff origin/main...origin/renderer`, 49 files, +7767/−55) — V, diffs read

| subsystem | files | what the change does | touches old path at defaults? |
|---|---|---|---|
| Static level mesh (new) | `rendering/hwrenderer/hw_levelresidency.{h,cpp}` (+5190), `hw_levelstatic.{h,cpp}` (+161), `common/rendering/hwrenderer/data/hw_levelbuffers.{h,cpp}` (+374), `wadsrc/static/shaders/glsl/levelmesh.vp` (+258), `main.fp` `#ifdef LEVEL_MESH` block (+104), `hw_surfacedamage.h` (+11: `ForEachDamagedSide`, a walk of the damage key map used by the change pass) [xc: added; it is one of the 49 files and was not in the table] | Part B | no (gated by `r_newrenderer`, default 0) |
| Wall cache (new, stage 2b) | `scene/hw_wallcache.{h,cpp}` (+466), `scene/hw_walldispatcher.h` (+18/−2: `order`, `recordOrder`, `noDistanceCull`) [xc: was +20] | replays kept `HWWall` parts per side when a signature hash is unchanged | one call per processed wall (`hw_bsp.cpp:349,763`); returns at `hw_wallcache.cpp:316` |
| Hooks in the old path | `scene/hw_bsp.cpp` (+35/−7), `hw_drawinfo.cpp` (+34), `hw_drawinfo.h` (+3), `hw_drawstructs.h` (+9), `hw_walls.cpp` (+55/−1), `hw_drawlist.cpp` (+4), `hw_entrypoint.cpp` (+3), `p_setup.cpp` (+6) | residency/wall-cache hooks; `HWWall::SetSectorLight/SetSideGlow/SetTierLight/SetTierLightLevel` (new, called only by the residency); `noDistanceCull` in `Process`; PutWall white-light rule for the no-draw-info path; opaque.* GPU groups; Tracy zones; `P_FreeLevelData` resets | see A.2.3 |
| Vulkan/backend plumbing | `vk_descriptorset.cpp` (bindings 16/17, pool 13→15 storage per set, vertex-stage count 11→13), `vk_shader.cpp` (compile EFF_LEVELMESH, set `ShaderReady/Failed`), `vk_buffer.{h,cpp}` (two SSO slots), `vk_renderdevice.cpp` (create/delete `LevelMeshBuffers`, Tracy zone on `GpuWait`), `hw_renderstate.h` (`EFF_LEVELMESH`), `hw_shaderpatcher.cpp` (effect entry), `shaderuniforms.h` (binding points 14/15), `v_video.h` (`mLevelMesh`), `gl_shader.cpp`/`gles_shader.cpp` (skip the effect) | always active: one more effect compiled per pass; two placeholder SSBOs bound every frame | yes (startup compile; descriptor layout) |
| Sector plane buffer | `hw_sectorplanebuffer.{h,cpp}` | `mLevelGeneration++` per `BeginLevel` | trivial |
| Sim snapshot (stage 6) | `rendering/r_snapshot.{h,cpp}` (+536), `d_net.cpp` (two `R_SnapshotWrite` calls) | Part B.9 | one int test per `TryRunTics` |
| Profiling | `common/utility/rema_profile.h` (new, Tracy macros, empty without `REMA_TRACY`), `CMakeLists.txt` (+Tracy link), `hw_clock.cpp` (flat-screen fallback for `scene`/`syncwait`), `hw_perftrack.cpp` (5 new CPU columns bsp/clip/walls/flats/sprites), `d_main.cpp` (zones), `d_net.cpp` (`simprofile` cycle_t clocks around TryRunTics stages), `p_tick.cpp` (`ticprofile` cycle_t clocks around P_Ticker stages) | measurement | yes: the clocks always run (rdtsc pairs); the perf_track CSV gets 5 more columns |
| Sim pacing | `d_net.cpp:207` `sim_maxcatchup` (default **2**, ARCHIVE): at most 2 tics a frame outside netgames, dropping the rest of the time | **behaviour change by default** in single player | yes |
| Thinker prefetch | `dthinker.{h,cpp}` `sim_prefetch` (default 0) | prefetch hints in `TickThinkers`/`ProfileThinkers`/`FThinkerIterator::Next` | one int test per thinker |
| Sound | `s_sound.cpp`, `s_soundinternal.h`: per-source channel index (`SourceChannels` TMap + `NextSource` list) used by stop/volume/pitch/relink/is-playing queries; `RestoreEvictedChannel(time)` with `snd_evictretry` (default **8**) and `snd_evictbudget` (default **64**); `s_doomsound.cpp`: `ClearChannelSource`, `NoteChannelSource` after save load, `soundchannels` CCMD, `cycle_t` around every `S_ChangeActorSoundPitch` | O(N channels) walks → O(own channels); evicted restarts staggered | **yes, by default** (restart timing; structure) |
| Bug fixes | `d_main.cpp`: no level frame while `gameaction == ga_fullconsole`; `vk_openxrdevice.cpp`: same test in `IsGameplaySceneActive`; `g_game.cpp`: `G_EnsureDemoSpace` before writing specials (VR demo overflow) | fixes | yes |

#### A.2.2 What main has and renderer lacks (`origin/renderer...origin/main`, 32 files, +5971/−801) — V, tags read

All of it is avatar/netplay lane work. None of it touches the scene renderer:

- playsim: `p_vrgear.{h,cpp}` (holster core rebuilt to TacticalRig behaviour, +1828/−458), `p_vrgrab.{h,cpp}` (new,
  distance grab), `p_vrgunfit.{h,cpp}` (new, muzzle sizing), `p_vrslots.h` (new), `p_vrtorso.{h,cpp}`
  ([STICKHIPS], [STARTSTEP], seated), `p_vrsynclog.cpp`, `p_user.cpp` ([SLOTLOAD], [VRGRAB], [VRGEAR],
  [GEARPLAY]), `p_map.cpp` ([CLAMPTIC]).
- r_data: `models.cpp` (+156/−10: [GUNFIT], [RIGIDANCHOR], [GUNFITCHECK]), `vr_body.cpp` (+707/−2: [SEATED],
  [LAGDIAG], [FOOTLATCH]), `vr_rig.cpp` ([LAGA], [STANDEYE]), `vr_avatar*.cpp`, `vr_bodyinfer*`, `vr_grasp.cpp`.
- net/save: `d_netinfo.cpp`/`d_netinf.h` ([VRGEAR] userinfo), `g_game.cpp`, `g_level.cpp`, `p_saveg.cpp`
  ([SLOTLOAD]).
- VR device: `vk_openxrdevice.cpp` (+13, [STANDEYE] helper call in `WriteHandCommand`).
- script: `vmthunks.cpp` (+196/−40), `wadsrc/static/zscript/doombase.zs` (+128/−26). [xc: the four sizes in this list
  were added+deleted sums written as "+N" (models +166, vr_body +709, vmthunks +236, doombase +154); numstat values
  given now]

#### A.2.3 Does renderer change the OLD render path (`r_newrenderer 0`, `r_wallcache 0`)?

| change | where | effect at defaults | mark |
|---|---|---|---|
| `mLevelMeshScene = LevelResidency_BeginScene(this)` | `hw_bsp.cpp:1464` → `hw_levelresidency.cpp:4223` | one cvar test a scene, returns false | V |
| `WallCache_Process(...)` before `wall.Process` | `hw_bsp.cpp:763` (single-thread), `:349` (scene worker) | one non-inline call + cvar read per processed wall, returns false | V |
| `experimentalMultiWallWorkers && r_wallcache <= 0 && !mLevelMeshScene` | `hw_bsp.cpp:743` | same decision as before at defaults | V |
| `Process`: `!(di->mh && di->mh->noDistanceCull) && IsDistanceCulled(seg)` | `hw_walls.cpp:2617` | one extra pointer test per `Process` | V |
| `PutWall`: `else if (white light && lightlevel == 255) flags &= ~HWF_GLOW` | `hw_walls.cpp:894` | **changes the parts made without a draw info**: only when `gl_bsp_worker_threads > 1` (non-default; wall workers fill `HWMeshHelper`), the wall cache, or the mesh build. The worker parts later pass through `PutWall(&mergeDispatcher)` (`hw_bsp.cpp:1531`), which applies the same rule per part, so the visible difference is limited to the "DoMidTexture leaves the flag cleared on later parts" case. Workers also need `gl_multithread` (default true) and, in a VR scene, `vr_scene_multithread` (default false, `hw_cvars.cpp:278`), so in VR at defaults the branch never runs on the old path even with `gl_bsp_worker_threads > 1`. Direction: the direct path (`di` set) already clears the flag on the shared `HWWall`, and `DoTexture` restores its saved flags (`hw_walls.cpp:1625,1677`) while `DoMidTexture` does not; on main the worker path kept the glow on parts after such a mid, so the change makes worker output equal the single-thread output rather than creating a new difference. [xc: added the multithread gates and the direction] | V (code); image effect I |
| `HWWall::SetSectorLight/SetSideGlow/SetTierLight/SetTierLightLevel` | `hw_walls.cpp:2431-2461` | not called by `Process` (it keeps its own statements); compiled under /fp:fast | V |
| `HWMeshHelper::order`/`recordOrder` pushes | `hw_walldispatcher.h` | a bool test per part on the mesh-helper path only | V |
| Effect `EFF_LEVELMESH` compiled for every pass | `vk_shader.cpp` (`CompileNextShader`), `hw_shaderpatcher.cpp` | startup shader compile + pipeline cache entries, always | V |
| Set 1 bindings 16/17 written every descriptor update; layout needs 13 vertex-stage storage buffers | `vk_descriptorset.cpp` | devices with `maxPerStageDescriptorStorageBuffers < 13` fail pipeline layout creation (warning printed). Not an issue on the owner's RTX 3080 Ti. | V |
| `opaque.walls/flats/masked/models/decals` GPU groups | `hw_drawinfo.cpp:3824…` | only with r_perflog's `perfGroups` on the main view | V |
| Tracy zones | many files | compile to nothing without `REMA_TRACY` (`rema_profile.h`) | V |

### A.3 main vs netplay (V)

`netplay` is the ancestor. The 63 differing files are main's 176 commits since the milestone, by subsystem
(full list in Appendix B):

- playsim (18): the whole holster/torso/grab/gunfit stack is absent on netplay (`p_vrgear*`, `p_vrgrab*`,
  `p_vrgunfit*`, `p_vrslots.h`, `p_vrtorso*` are all new relative to it); `p_user.cpp` +172; `p_map.cpp` +49.
- r_data (13): `models.cpp` +515 ([MODHAND], [WORLDPSPR], [GUNFIT], [RIGIDANCHOR], [GUNINHAND], [GRASP],
  [LOOKDOWNFADE]), `vr_body.cpp` +715, `vr_grasp.{h,cpp}` (new), `vr_rig.cpp` +332.
- net/protocol: `d_protocol.h` (head pose grows from 6 to 12 bytes: `roomdx`, `roomdy`, `turn`;
  `DEM_NETREMOVEPAWN = 212`), `d_net.cpp` +92 (pawn removal on a tic of the host's choosing),
  `common/engine/i_upnp.{h,cpp}` (new, UPnP host port mapping), `d_netinfo.cpp`, `g_game.cpp`, `p_saveg.cpp`,
  `version.h` (DEMOGAMEVERSION/MINDEMOVERSION 0x221 → 0x222, so netplay demos do not play on main).
- render: `hw_weapon.cpp` +91 and `hw_drawinfo.cpp` +9 ([WORLDPSPR]: `DrawWorldPlayerSprites` after psprites in
  every non-sky-portal scene), `hw_models.cpp` and `modelrenderer.h` (`hudWorldDepth`), `main.fp` +7
  ([LOOKDOWNFADE]: negative `uEyeFadeNear` is a dither floor), `r_utility.cpp` (roomscale offsets keyed on
  `VR_InputViaCommand()` instead of `multiplayer`), `texture.cpp`/`textures.h` (`FCanvasTexture` destructor
  clears the canvas back-pointer: a quit crash fix).
- VR device: `vk_openxrdevice.{h,cpp}` (separate `xrNet*` controller state for the command path).

### A.4 main vs selaco (V for the counts and the files read; the per-file descriptions are condensed)

Fork point: 5.0.3 (2026-09-06). Of the 389 differing files, 3 groups by subsystem (S = only selaco changed it since
the fork, M = only main, SM = both; full list in Appendix C):

| subsystem | files | S/M/SM | what differs |
|---|---|---|---|
| playsim | 55 | 3/38/14 | main: VR gear/torso/grab/synclog, netplay; selaco: `p_pooledparticles.{h,cpp}` (+2453, new), thinker sleep, ABSDAMAGE, block bits |
| top-level src | 42 | 12/14/16 | selaco: `g_levellocals.h` +1441, IWADOVER, `-nostockactors`; main: netplay `d_net`/`d_protocol`, `g_game` |
| hwrenderer common data | 34 | 0/12/22 | both carry REMA effect buffers; main adds perftrack, effect lights, mesh particles changes |
| engine/platform/utility | 33 | 14/18/1 | selaco: Globals store, joystick acceleration; main: UPnP, hang watch, etc. |
| r_data | 30 | 3/23/4 | selaco: `model_reach*` (new), `level_solid_query*` (new); main: `vr_*` body/rig/grasp (absent on selaco) |
| vulkan backend | 28 | 0/4/24 | `vk_openxrdevice.cpp` 2255 lines apart; main: OpenXR netplay/command input, sync modes |
| shaders | 24 | 0/11/13 | `main.fp` 1175 lines apart; main: effect lights, G6 glow seed, LOOKDOWNFADE |
| zscript | 23 | 7/6/10 | selaco: `particledefinition.zs`, Selaco natives; main: VR gear/grab hooks |
| hwrenderer effects | 21 | 0/12/9 | smoke/debris/surface-damage/level-field: both have them (ported to selaco in one build each), main has later revisions |
| scene | 16 | 2/5/9 | `hw_bsp.cpp`: selaco adds `ParticlePoolJob` + `RenderDefinedParticles` per subsector (main's `hw_bsp.cpp` is unchanged since the fork); `hw_flats.cpp`: main has [EFFECTLIGHTS] and [G6] `SetGlowPlanes`, selaco inlines glow planes; `hw_drawinfo.cpp` 1034 lines apart |
| others | gamedata 14, model formats 13, sound 10, textures 9, GL 8, scripting VM 8, post-process 5, misc | — | selaco: DDS compressed upload, damagedefs/surfacedefs/defblocks, ambient-sound range; main: hearing impulse |

Hot-path consequence (V): selaco's walk does more per subsector (pooled-particle job); its flats/glow differ from
main's. Selaco has none of: `hw_effectlights.cpp`, `hw_perftrack.cpp`, `p_vrsynclog.cpp`, `vr_body.cpp`,
`vr_rig.cpp`, the level mesh, or the wall cache.

### A.5 renderer vs netplay (V)

`renderer` = `netplay` + main's 100 commits from netplay to 77e5ee3d (44 files: holster core v1 `p_vrgear.cpp`
+744, torso +251, [SYNCGEAR] synclog, [WORLDPSPR], grasp, UPnP, the protocol growth, LOOKDOWNFADE) + the render
lane's 116 commits (42 files) + 7 files changed by both. Appendix D lists the 93 files with R/M/RM attribution.

---

## Part B — The static level path (renderer tree)

### B.1 Entry points and call graph

```
D_Display (d_main.cpp:1683 screen->FrameCount++)
 └ RenderView → [camera textures: RenderViewpoint(..., mainview=false) hw_entrypoint.cpp:730]
              → RenderViewpoint(main)  hw_entrypoint.cpp:753 → eye loop :361 (1 scene if multiview, else 1 per eye)
    └ ProcessScene → CreateScene → HWDrawInfo::RenderBSP  hw_bsp.cpp:1445
        Bsp.Clock()                                                           (glcycle_t Bsp)
        LevelResidency_BeginScene(di)                    hw_levelresidency.cpp:4221   [main thread]
          ├ gates (cvar, buffers/shader, portal/fullbright/OOB/ortho/skybox/mirror, gl_render_walls, gl_texture)
          ├ global hash moved / other level / other switches → Build() (≤ once per 35 frames)      :2088
          ├ first scene of a frame: ClearEvictions :4167, RunChangePass :3930
          │     ├ sim moved: RunTicPass :3681 → sectors, vertex lists, FoldBack :3637, dyn-light marks, RunLightPatch :2994, RunFlatLightPatch :3109
          │     ├ else RefreshMovingSectors :3892
          │     └ damage marks (SurfaceDamage::ForEachDamagedSide), optional side scan
          │   WriteAllFlatPlaneRecords if plane buffer was zeroed; WriteAllLights if the level-wide light key moved
          └ reserve this scene's per-frame index region; clear ReachedSides/ReachedFlatRuns
        RenderBSPNode → DoSubsector → ProcessVisibleSubsector hw_bsp.cpp:1059
            CheckUpdate(flat VBO heights) :1063  — unchanged
            AddLines :861 (ClipWall clock) → AddLine :530
               clipper / backface / hw_FakeFlat / hw_CheckClip / SafeAddClipRange :581-731 — unchanged
               → LevelResidency_SkipWall :4410 → SkipWallDecision :4327   (instead of HWWall::Process)
                    [multithread: on the scene worker, hw_bsp.cpp:346]
            flats: LevelResidency_FlatPlanes :4464 → ProcessSector(which) or nothing   hw_bsp.cpp:1157 / :368
        Bsp.Unclock()
    └ DrawScene → RenderScene  hw_drawinfo.cpp:3730
        LevelResidency_Draw :4632       (before GLDL_PLAINWALLS)   hw_drawinfo.cpp:3830
           MakeSceneRanges :4522 → per material: SetMaterial + DrawIndexed
        LevelResidency_DrawFlats :4864  (before GLDL_PLAINFLATS)   hw_drawinfo.cpp:3835
           ObserveOldFlats :4705, MakeSceneFlatRanges :4776 → per flat group: SetMaterial (+tex matrix) + DrawIndexed
P_FreeLevelData (p_setup.cpp:641-643): R_SnapshotInvalidate, WallCache_Reset, LevelResidency_Reset :4207
TryRunTics (d_net.cpp:2493, :2557): R_SnapshotWrite (r_snapshot.cpp:278)
```

### B.2 Cvars and commands (all V)

| cvar | def | file:line | read at | effect |
|---|---|---|---|---|
| `r_newrenderer` | 0 | `hw_levelresidency.cpp:137` | `:4223`, `:2103-2104`, `:4241` | 0 off. 1 static walls of non-sky sectors. 2 + sky-ceiling sides (portal walls kept, replayed through PutPortal). 3 + static flats. |
| `r_levelmesh_verify` | 0 | `:140` | many | dev check: rebuilds every skipped side through `Process` and compares field by field (`VerifySide :4088`). Every light-moved side goes through `CheckSideParts`. Flats are drawn once by the old path and compared. Costs `Process` per reached side per frame. |
| `r_levelmesh_sidescan` | 0 | `:144` | `:3971` | 1: hash every static side's own fields every frame in the pass; 0: hash each reached side in the walk. |
| `r_levelmesh_clock` | 0 | `:147` | `:4412`, `:4390` | clocks (cycle_t `WalkClock`, `ReplayClock`) around each reached-side decision |
| `r_levelmesh_drawall` | 0 | `:150` | `:4530`, `:4783` | 1: draw the whole static index buffer (no walk culling) |
| `r_levelmesh_foldback` | **1** | `:155` | `:3698` | 2d fold-back on |
| `r_wallcache` | 0 | `hw_wallcache.cpp:40` | `:315`, `hw_bsp.cpp:743`, global hash | stage 2b wall cache |
| `r_snapshot` | 0 | `r_snapshot.cpp:28` | `:280`, `:323`, `:373` | 1 compact records; 2 records + byte images; 3 records + parallel measurement, **without** 2's serial byte images (`:323` tests `== 2`) [xc: was "3 + parallel measurement" on top of 2; read line :371 → :373] |
| `r_snapshot_threads` | 6 | `:29` | `:191` | r_snapshot 3 only [xc: read at :191, not :197] |
| `r_renderthread` | false | `:33` | its own callback | **stub**: prints "not built yet" and resets to false |

CCMDs: `levelmeshstats` (`:4943`, prints and resets `FrameStats`), `levelstats` (`hw_levelstatic.cpp:103`),
`wallcachestats` (`hw_wallcache.cpp:412`), `snapshotstats` (`r_snapshot.cpp:390`).

### B.3 What it builds at load (`Build`, `:2088-2596`) — V

Build is called from `BeginScene` on the first accepted scene after a level loads or a switch/global hash moves
(`:4242-4247`). It is rate-limited to once per 35 frames for global-hash and switch (`otherBuild`) changes; only a
new level bypasses the limit (`:4245`) [xc: the limit covers switch changes too] (`REBUILD_FRAMES`, `:410`) and runs
inside `RenderBSP`'s `Bsp` clock, so the build frame shows a spike in the perf_track `bsp` column. It returns false:
that frame and the next (`IsDrawable` needs `FrameCount > mAllocFrame + 1`, `hw_levelbuffers.cpp:172`) draw
entirely on the old path.

1. **Classify** every side (`LevelStatic_Classify`, `hw_levelstatic.cpp:93`). A side is refused if it has: no segs,
   polyobject, line portal/horizon/mirror, translucent mid, heightsec, 3D floors/light list, sector portal, sky
   plane (accepted at ≥2 for a sky ceiling), an animated base pic (`TexAnim` base pics), a warped/canvas/translucent
   texture. The classifier also marks a heightsec/3D-floor neighbour at either vertex (`LSR_NEIGHBOUR`), but `Build`
   accepts that reason like `LSR_ELIGIBLE` (`:2180`, and `FoldSide` `:3225`), so such sides are not refused
   [xc: the doc listed the neighbour test as a refusal]. O(sides × 3 tiers).
2. **Shadow**: per sector, `MakeSectorShadow` (264 B, `:905`) plus three hashes (`SectorBuildHash`, `SectorLightHash`,
   `SectorPlaneBuildHash`, `:2138-2147`), computed for every sector at every build but read only under
   `r_levelmesh_verify` (`:3709`) [xc: was "for r_levelmesh_verify only"; the computation is unconditional].
   Floor/ceiling texture pointers go into `SectorFlatTex`. Per vertex, `RecalcVertexHeights` if dirty and a hash of
   its height list (`:2150-2164`). Note: the build *writes* `vertex_t` state through `RecalcVertexHeights`, as
   `Process` itself does.
3. **Per eligible side** (`:2177-2398`): run `HWWall::Process` into an `HWMeshHelper` with `noDistanceCull`
   (view-independent). Then collect every refusal reason:
   - lightmap;
   - sector not cacheable;
   - translucent/missing-texture lists;
   - portal walls other than a plain deferred sky on a sky side;
   - part type other than TOP/M1S/BOTTOM;
   - light list, style/alpha, texture flags, material shader ≠ Default (`PartRefusals :1648`);
   - the slot bound.

   An accepted side gets an **FSideEntry** (`:209`) and a slot:
   - Surfaces: 2 if the seg has a back sector, else 1.
   - Vertices per part: `4 + E(v1) + E(v2) + 2·(numsegs−1)`, where `E(v) = 2·numsectors − 2` (level planes) or
     `2·numsectors` (`SideVertexBound :1703`), or 4 without `gl_seamless`.
   - Vertices are copied from `CreateVertices` (`MakePartVertices :1723`), with the surface id written into `lu`.
   - Light record by key (`FindLightInput :1455`, which runs the old path's setters against a draw-nothing
     `FLightProbeState` `:170`).
   - The /fp:fast rounding of `RescaleLightLevel` is observed per part type (`ObserveRescale :1372`).
4. **Indices**: parts are sorted by (has triangles, texture index, texture ptr, clamp); per surface the slot is
   `(vpp−2)·3` indices, a fan as a triangle list with a degenerate tail (`WriteSlotIndices :1762`). One
   `FMaterialGroup` (range) per (texture, clamp).
5. **Fold-back tables**: vertex→static-sides CSR (`VertexSideFirst/List`), stamps, `SectorFoldPrev` sized to all
   sectors. **Spare light records**: one per key that a side's TOP/BOTTOM or M1S slot could ask for but no record
   has yet (`:2506-2525`).
6. **Flats** (`r_newrenderer 3`, `BuildFlatPart :1819`):
   - Per sector plane: `FlatPlaneRefusal :1784` (not cacheable, skybox, lightmaps, sky, plane-buffer capacity,
     flat-edge glow, texture fx, no texture, animated, warped/canvas/translucent, material).
   - Per section of the sector: the existing CreateVBO index range `sec->iboindex[p] + section.vertexindex`,
     `vertexcount`. It is refused for DONTRENDER, hacked, render_sector mismatch, empty, or out of range.
   - Material group per (texture, texture matrix from `hw_SetPlaneTextureRotation`).
   - One surface record and one light record per plane (stale if the old path's rounding for that level is
     undecided).
   - Indices are **copied** from `fvb->ibo_data` into `FlatStaticIndices`, sorted by group.
   - `FinishFlats :2059` writes the surface id into `lu` of every flat-VBO vertex those runs use. This touches the
     CPU shadow and every mapped pipeline copy, then calls `Upload`. It then writes every static flat's sector into
     the sector-plane SSBO (binding 12).
7. **Allocate** (`hw_levelbuffers.cpp:77`):
   - static VB/IB and flat IB as `BufferUsageType::Static`: GPU-only, with a CPU_ONLY staging buffer of equal size
     that is kept (`vk_hwbuffer.cpp:73-97`);
   - surface and light SSBOs as Persistent; they grow through `Resize`, which waits for the GPU, and never shrink;
   - per-frame index buffer of `(wallIndices + flatIndices) × FRAME_INDEX_SCENES(2)`, Persistent, created or grown
     by `SetData` only.
   - Then the surface records are memcpy'd and all light records written (`WriteAllLights :1151`).
8. Prints `levelmesh: slots …` and `levelmesh: built … %.1f ms` (`BuildStats.BuildMS`, a `cycle_t` around the whole
   build).

Build cost: O(eligible sides × `HWWall::Process` + `CreateVertices`) + O(sectors + vertices + sections) + sorts. Its
time is in the console line; I have no measured value to quote.

### B.4 What it does per frame — V

**BeginScene (`:4221`)**, every scene:
- `WallCache_GlobalHash` (≈10 mixes) and `ComputeLightKey` (≈12 mixes).
- At the first scene of a frame (`screen->FrameCount` changed):
  - `ClearEvictions`: O(sides and flats evicted last frame); clears `LEVELMESH_SURFACE_EVICTED` bits in the mapped
    surface SSBO.
  - `RunChangePass`.
  - A plane-buffer generation check.
  - `WriteAllLights` if the light key moved.
- A later scene with a different light key keeps the old path (`:4277-4283`).
- Each accepted scene reserves `StaticIndices.Size() + FlatStaticIndices.Size()` indices of the per-frame buffer
  (`:4292-4307`). With no room left, it draws the whole static buffer, or, if any material group was changed by a
  fold-back, it does not use the mesh at all.

**RunChangePass (`:3930`)**:
- **Tic pass** (`RunTicPass :3681`). It runs when `gametic`, `Level->LocalTimer` or `HasDynamicLights` changed since
  the last pass (`:3946`). At 72-90 Hz with 35 tics/s that is roughly every second frame (**I**), which alternates
  frame cost. Its steps:
  1. **All sectors** (`:3713-3805`):
     - `SectorPlanesDiffer` (80 B memcmp) and `GatherSectorRest` + 104 B memcmp. This reads `sec->e->XFloor`, a
       pointer chase.
     - Fold-candidate bookkeeping for changed sectors.
     - `GatherSectorLight` + 80 B memcmp.
     - **two `TexMan.GetGameTexture` lookups**.
     - For sectors with static flats: 28 B flat-light compare, plus a plane-record write while moving.
     - `interpolations[].ForceGet()` ×2 (read barrier).
     - With `r_levelmesh_verify`, the three old hashes too.

     Cost: O(numsectors) every pass, whatever is visible.
  2. Vertex height lists: only around sectors whose planes changed (`LookAtVertex :1036`, which may call
     `RecalcVertexHeights`). Finding them is a second loop over all sectors' `SectorPlaneChanged` bytes whenever any
     sector differs and `gl_seamless` is on (`:3814-3827`) [xc: added].
  3. `FoldBack :3637`: candidates still for one tic. At most `FOLD_SIDES_PER_PASS = 500` sides through `Process` a
     pass (`:837`); the first closure always runs, even over the cap.
  4. Dynamic lights: O(lights × (wall_tlist + flat_tlist)) when `Level->HasDynamicLights`; marks sides/flats
     `FB_DYNLIGHTS`.
  5. `RunLightPatch :2994`: pending sectors; then each moved sector's records (`PatchSectorLights :2744` →
     `RecomputeLight :2706`: `LightOwner` (may walk the sector's lines), `DirectLightInput` (SetSectorLight /
     SetSideGlow / `PatchRescale` / SetTierLightLevel / MakeLightInput with probe state runs), `WriteLightRecord`
     only if changed). Then the static sides on the moved sectors' lines (`EvaluateSideLight :2952`): `CheckFog` for
     two-sided; `CheckSideParts :2821` (= `HWWall::Process`) only for a sky side whose fade colour changed, a side
     next to a stepping flat-texture, or verify. Then the waiting sides. `RunFlatLightPatch :3109` likewise for flats.
- **Frames between passes**: `RefreshMovingSectors :3892`. Sectors with a plane interpolation at the last pass are
  compared again, and their vertices re-hashed (interpolated heights change every frame).
- Every frame: `SurfaceDamage::ForEachDamagedSide` (O(damage map entries)) → `MarkSide`; optional side scan
  (O(static sides) `WallCache_SideFastHash`).
- Clocked (cycle_t, always on): `FrameStats.PassMS/PassMaxMS` (whole pass), `PassSectorMS`, `PassVertexMS`,
  `PassLightMS`, `PassPatchMS`, `PassMovingMS`, `PassDamageMS`, `PassSideScanMS`, `FoldMS/FoldMaxMS`,
  `LightRecordMS` (WriteAllLights). Not covered: `ClearEvictions`, the hashes in BeginScene, and the per-side walk
  decision (only with `r_levelmesh_clock 1`). All of it is inside the engine's `Bsp` glcycle_t, so it lands in the
  perf_track `bsp` column (`Bsp − ClipWall`, `hw_perftrack.cpp`).

**Walk hook — SkipWallDecision (`:4327`)**, per reached seg of every side, per scene:
1. `FSideEntry` lookup → not static → old path.
2. `EvictedNow` → old path.
3. Front/back sector equal to the build's (else EV_SECTOR).
4. `OkStamp == SceneStamp` → true (other segs of the side).
5. Dither flags, `AttachedDecals`, `FB_DYNLIGHTS`, `FB_DAMAGE` → evict.
6. `FoldRefused`.
7. `SectorChanged[front|back] | VertexChanged[v1|v2]`.
8. **Once a frame, `WallCache_SideFastHash(side, line)`** over `side_t::textures … Flags` plus glow and line fields
   (`hw_wallcache.cpp:212-228`) (`OwnChecksInWalk`).
9. `LightState`/`SectorLightStale`.
10. Verify.
11. **Sky sides: copy each kept portal `HWWall` and `PutPortal` it, every scene** (`:4388-4401`).
12. Push to `ReachedSides`.

Evict writes `LEVELMESH_SURFACE_EVICTED` into the mapped surface SSBO for every surface of the side (`:4155`).

**FlatPlanes (`:4464`)**, per reached section: up to two `FlatDrawableNow` checks. These are once a scene per plane:
- `SectorChanged`;
- xform memcmp;
- a `TexMan.GetGameTexture`;
- glow/texfx/dither;
- `FB_DYNLIGHTS`;
- **`SurfaceDamageFlatKey`** (a hash lookup per flat per scene);
- a stale check.

Each accepted plane pushes a run to `ReachedFlatRuns`.

**Draw (`LevelResidency_Draw :4632`)**:
- `MakeSceneRanges :4522`: `std::sort(ReachedSides)` (O(n log n)), two passes over reached surfaces, and a
  `memcpy` of each run's live indices (`run.Count`; the degenerate slot tail is not copied) into
  the scene's region of the persistently mapped per-frame index buffer.
- Then ~15 render-state sets and, for **every** material group (also those with count 0, skipped inside the loop),
  `SetMaterial` + `DrawIndexed` (`:4675-4685`).
- Cull_CW. Effect-light mode as RenderTexturedWall.
- `IndexBuildMS` is clocked (cycle_t). The draw itself is inside the engine's `RenderAll` (not `RenderWall`), so its
  CPU cost is in perf_track `scene` and **not** in the `walls` column. On the GPU it is inside the `opaque.walls`
  group (`hw_drawinfo.cpp:3828-3832`).

**DrawFlats (`:4864`)**:
- `ObserveOldFlats` walks the scene's plain-flats list while either rounding is undecided or verify is on.
- `MakeSceneFlatRanges`, then per flat group `SetMaterial` (+ texture matrix) + `DrawIndexed`, with the flat VBO
  bound. No culling: the plane-side test is in the shader. GPU inside `opaque.flats`.

**Shaders**:
- `levelmesh.vp`:
  - per vertex: two clamped SSBO reads (surface 32 B, light record fields ~7 vec4) and a sector-plane SSBO read for
    flats;
  - collapse to (2,2,2,0) when evicted or on a flat's closed side;
  - otherwise main.vp's maths.
- `main.fp` under `LEVEL_MESH`: ~25 uniform names are `#define`d to `levelMeshLights[vLmLight].*`, so every use is
  an SSBO read per fragment (cached by the GPU; **I**). Index buffer and records live in Persistent (HOST_VISIBLE,
  DEVICE_LOCAL preferred) memory. Whether VMA placed them in device-local BAR or system memory is not determined by
  the code (**I**; see open questions).

### B.5 Fold-back (stage 2d) — V

A sector that is not as built becomes a candidate when two consecutive passes, with a tic between them, see the same
planes and rest (`:3726-3738`). `FoldClosure :3549` gathers:
- the static sides on its lines whose other sector is as built;
- with `gl_seamless`, every static side touching a vertex whose height list settled (all of its other sectors are as
  built).

`FoldSide :3201`:
- re-classifies;
- runs `Process`;
- re-applies every build test, including the slot bound and light-record room;
- writes the slot's vertices and indices through `SetSubData` (staging + a copy in this frame's transfer commands,
  `vk_hwbuffer.cpp:141-158`);
- rewrites surface records;
- may create light records (spares) or material groups.

The material groups are new `FMaterialGroup`s with no build range (`WallGroupOf :3171`). Once any part changes group
(`WallRegrouped > 0`), the whole-buffer fallback can no longer be used: every no-room/drawall/foreign-list scene must
rebuild ranges from all runs, or the scene leaves the mesh (`:4305`, `:4531-4546`). A folded side is evicted for its
fold frame (EV_FOLD). `FoldFlats :3464` re-groups flats (no slots; their vertices are the old flat VBO).

### B.6 Memory — V for formulas, prev. measured for totals

| item | size | where | notes |
|---|---|---|---|
| static wall VB | Σ slots × vpp × 32 B (`FFlatVertex` = 8 floats) | GPU + equal CPU_ONLY staging | worst-case slots, degenerate tails |
| static wall IB | Σ slots × (vpp−2)×3 × 4 B | GPU + staging + **`StaticIndices` CPU copy** | |
| static flat IB | Σ section runs × 4 B | GPU + staging + **`FlatStaticIndices` CPU copy** | duplicates CreateVBO's `ibo_data` ranges |
| per-frame IB | 2 × (wall IB + flat IB) indices × 4 B | Persistent mapped | never shrinks across levels |
| surface SSBO | 32 B × surfaces (walls' slots + flats) | Persistent mapped | grows, never shrinks |
| light SSBO | 320 B × (records + spares) | Persistent mapped | grows, never shrinks |
| `FSideEntry` | ~72 B × **all** sides (**I**, from field list `:209-238` + padding) | CPU | |
| `FSectorShadow` | 264 B × sectors (`static_assert`s `:808-809`) | CPU | the pass reads it all every tic |
| `SectorFoldPrev` | ~192 B × sectors (**I**) | CPU | |
| `FLightInput` | several hundred B × records (**I**: FColormap, 2×FGlowInput-ish arrays, FColorInput 128 B, FLightKey) | CPU | |
| `KeptPortals` | `sizeof(HWWall)` × kept sky portal walls | CPU | printed by levelmeshstats |
| verify only | `KeptParts` (HWWall per surface), `VerifyVertices` (copy of the VB) | CPU | only when built with verify |

Prev. measured (HANDOFF_RENDER_2026-11-11.md, stage 2d build 1113a): Abandon 157.7 MB and Sunder 180.0 MB of
level-mesh GPU memory (static 109-125 MB, per-frame index buffer 47-53 MB). `levelmeshstats` sums
`vb+ib+sb+lb+fib+frameBytes` (`:4993-4996`). It does **not** count the staging copies of the three Static buffers,
nor the CPU arrays. On a 12 GB card and 16 GB host this is a real share (**I**).

### B.7 What it removes and what it leaves (the key question)

**Removed, per static side that is reached and not evicted** (V, by where the hook sits):
- `HWWall::Process` (`hw_walls.cpp:2472`): texture coordinates, light, glow, `PutWall`, the draw-list push
  (`hw_bsp.cpp:760` replaces `:764`).
- At draw time on Vulkan: `HWWall::DrawWall` → `SetupLights` + `MakeVertices` (`hw_walls.cpp:559-567`),
  `RenderTexturedWall` state setup, and **one `Draw(DT_TriangleFan)` per part** (`hw_walls.cpp:222`). These are
  replaced by one `DrawIndexed` per material group.
- For static flats: `HWFlat::ProcessSector` and its plain-flat `DrawFlat`/`DrawSubsectors` draws.

Prev. measured (1112r, Sunder MAP15): walls CPU 2.56 → 1.05 ms, draw calls 0.76 → 0.37 ms.

**Left, unchanged, every scene** (V):
- `RenderBSPNode` recursion (`hw_bsp.cpp:1384`) and `DoSubsector`'s per-subsector clipper visibility tests
  (`hw_bsp.cpp:1296-1335`).
- `DoSubsector` and `ProcessVisibleSubsector` (`:1059`), including:
  - `CheckUpdate` of flat-VBO heights (`:1061-1064`) — still needed, the static flats draw from those vertices;
  - particle/sprite jobs (`:1067-1101`);
  - `SECMF_DRAWN` (`:1091`);
  - dither targets;
  - section/subsector render flags;
  - hacked subsectors;
  - portal coverage.
- `AddLines` / `AddLine`:
  - `ClipPortal::ClipSeg`;
  - `GetClipAngle` ×2;
  - radar clipper;
  - backface rejection;
  - `SafeCheckRange`;
  - `IsDistanceCulled`;
  - `hw_FakeFlat` + `hw_CheckClip` for two-sided lines;
  - `SafeAddClipRange` occlusion;
  - `ML_MAPPED` (automap);
  - `validcount`.

  All of this runs **before** the hook, and the hook sits exactly where `Process` was (`:581-731`, then `:760`).

Why the walk cannot go (V): the mesh draws only `ReachedSides` / `ReachedFlatRuns`, which only the walk fills
(`:4406`, `:4491`). Without it the only option is `r_levelmesh_drawall`. Prev. measured (1112n): 491,998 vertices a
frame against 10,511, GPU `opaque.walls` 0.58 → 2.18 ms. Sprites, particles, portals, automap bits, flat-VBO height
updates and every non-static or evicted surface also come only from the walk. `docs/plans/STAGE3_VISIBILITY_PLAN.md`
Q1 reaches the same conclusion; it is a lead, and the code above confirms it.

**Added** (V): the tic pass (O(sectors) per tic), the per-reached-side decision and own-field hash, the sky-portal
`HWWall` copies + `PutPortal` per scene, the per-frame index copy and sort, light-record recomputes, fold-back
`Process` runs, and `SurfaceDamageFlatKey` per flat. Prev. measured (1112r): bsp 0.56 → 2.49 ms, of which the sector
comparison was 1.42 ms (later 1.43 ms a pass on 13,028 sectors after the shadow rewrite).

**Evicted surfaces fall back to the old path in full.** That includes every side with a dynamic light. Prev. measured
(1112x): with dynamic lights on, Abandon had 288,577 wall and 431,575 flat-plane evictions in a run. So with lights
on, the saving shrinks toward zero while the added costs stay (**I**).

**Net**, prev. measured: Sunder MAP15 `r_dynlights 0` gave old 70.1 / new 60.8 fps (1112y) and old 76.2 / new 67.2
(1112x). The path is slower than the old one in every measured pair.

### B.8 Correctness and design hazards found in the code

1. **Camera textures are not excluded** (I, from V code). `BeginScene` has no `mainview` test (`:4221-4250`). Camera
   textures render before the main view (`hw_entrypoint.cpp:726-730` vs `:753`) and are not portals. They can
   therefore be accepted, run the frame's change pass, and take one of the `FRAME_INDEX_SCENES = 2` regions
   (`hw_levelbuffers.h:78`). In non-multiview VR the second eye then finds no room. It draws the whole static index
   buffer, or no mesh at all once a group changed (`:4302-4306`). The comments at `:57` and `:4227` ("main view
   only") are not enforced [xc: line was :58]. A second consequence of the same gap: the camera scene, being the
   frame's first accepted scene, sets the frame's light key; a later main-view scene whose `ComputeLightKey`
   (`:1053`, includes `getExtraLight()`) differs is refused whole for that frame (`ScenesOtherLight`, `:4277-4283`)
   [xc: added; I — not traced whether `r_viewpoint.extralight` differs between the camera and main scenes].
2. **Eviction flags are frame-global** (`:4155-4187`): the GPU reads them when the frame's commands run. If two scenes
   in one frame decide differently for a side (mesh in scene A, evicted in scene B), A loses the side. The per-frame
   decisions (dither, decals, dynlights, damage, own fields once a frame) make this unlikely. EV_SECTOR depends on the
   sectors passed per scene (**I**, low).
3. **Duplicated light logic**. `HWWall::SetSectorLight/SetSideGlow/SetTierLight*` duplicate statements that
   `Process` keeps separately (`hw_walls.cpp:2416-2461`, comment: calling them from `Process` changed the image). The
   rounding form is learned from observed parts (`RescaleForm :1358-1404`). Only `r_levelmesh_verify` detects drift.
   The flat rounding is learned only by observing the old path's own draw lists at runtime (`ObserveOldFlats`); until
   then those flats stay on the old path.
4. **The flat-VBO `lu` write** (`FinishFlats :2059-2086`) relies on the old path never reading `lu` for a vertex
   whose `lindex` is −1 and on `UpdatePlaneVertices` writing only `z` (V: `hw_vertexbuilder.cpp:406-421`). Any
   future rewrite of whole flat vertices silently breaks the surface id (**I**).
5. `MAX_SURFACES = 1<<24` because the id rides in a float (`:409`).
6. The tic pass reads `sec->interpolations[].ForceGet()` (a read barrier that can write the pointer, per its own
   comment `:3803`) for every sector, every pass.
7. `FoldBack` uses a function-static scratch (`:3653`). It is main-thread only, which holds today.

### B.9 Stage-6 snapshot writer (`r_snapshot.cpp`) — V

- `R_SnapshotWrite :278` is called at the end of every `TryRunTics` (`d_net.cpp:2557`). It is also called in the
  "no tic ran, predicted" branch (`:2493`) — so at most one write per frame, per frame, not per tic.
- `r_snapshot 0`: one int test.
- `r_snapshot 1` does the following, with a mutex held only around index selection:
  - O(sectors) over the `touching_renderthings` and `sectorportal_thinglist` lists;
  - an open-addressing pointer hash per node;
  - an `FSnapActorRec` per unique actor (312 B prev. measured; RENDER_THREAD_SHADOW_WORLD.md);
  - CSR index arrays.
- `r_snapshot 2`: also memcpy of every actor in `thinglist` (class size) and whole `sectors`/`sides`/`lines` arrays.
- `r_snapshot 3`: the `r_snapshot 1` records plus `MeasureParallel :189` (not the `r_snapshot 2` serial byte images:
  `:323` tests `== 2`) [xc: was "also", implying 3 includes 2], which **creates and joins `std::thread`s on every write**
  (`RunParallel :181`) for actor images and a 4 KB page diff of the level arrays against a shadow. Measurement only.
- Prev. measured (RENDER_THREAD_SHADOW_WORLD.md, Abandon MAP13, `doomxr_rend1111b`): 17,481 records, 5.5 ms a
  write (max 8.4). Byte images (r_snapshot 2) 24.3 ms. r_snapshot 3: actor images 11.8 ms on 6 workers.
- **Nothing reads it**: `R_SnapshotAcquire :376` / `R_SnapshotRelease :383` have no callers (grep: only their
  definitions). At `r_snapshot 1`, therefore, every frame pays the write for nothing. `FWorldSnapshot::ActorImage/
  Sectors()/Sides()/Lines()` have no callers. `r_renderthread` refuses to turn on.

### B.10 Per-function table (static path)

Cost class: C = per call constant; S = per sector; Sd = per side; V = per vertex; L = per light; R = per record.

| function | file:line | called from | per | cost class | notes |
|---|---|---|---|---|---|
| `LevelResidency_BeginScene` | hw_levelresidency.cpp:4221 | `RenderBSP` hw_bsp.cpp:1464 | scene | C (+pass at frame's first scene) | inside `Bsp` clock |
| `Build` | :2088 | BeginScene :4246 | level / global change | Sd×Process + S + V + sort | `BuildStats.BuildMS` |
| `LevelStatic_Classify` | hw_levelstatic.cpp:93 | Build :2112, `levelstats` | build | Sd | |
| `LevelStatic_ClassifySide` | hw_levelstatic.cpp:60 | Classify, `FoldSide` :3224 | side | C (3 tiers, TMap lookups) | |
| `PartRefusals` | :1648 | Build, FoldSide, CheckSideParts | part | C (+`FMaterial::ValidateTexture`) | |
| `SideVertexBound` | :1703 | Build | side | C | |
| `MakePartVertices` | :1723 | Build, FoldSide, VerifySide | part | V | `HWWall::CreateVertices` |
| `FindLightInput` / `MakeLightInput` / `ReadPartGlow` / `ReadPartColors` / `MakeLightKey` | :1455/:1410/:1171/:1278/:1235 | Build, patch, fold | part | C (probe state setters) | |
| `ObserveRescale` / `DecideRescaleForms` / `PatchRescale` | :1372/:1382/:1395 | Build / patch | part | C | /fp:fast reproduction |
| `BuildFlatPart` | :1819 | Build | build | S×sections + sort | r_newrenderer 3 |
| `FinishFlats` | :2059 | Build | build | flat-run vertices + uploads | writes the shared flat VBO |
| `WriteAllLights` / `WriteLightRecord` | :1151/:1077 | Build, BeginScene | light-key change / record | R × (SetFog+SetColor on probe) | `LightRecordMS` |
| `RunChangePass` | :3930 | BeginScene :4261 | frame | see below | `PassMS`, `PassMaxMS` |
| `RunTicPass` | :3681 | RunChangePass :3952 | sim tic (≈ every 2nd VR frame, I) | **S (all sectors)** + changed + L | `PassSectorMS` etc. |
| `LookAtVertex` | :1036 | tic pass, moving sectors | vertex | C (+`RecalcVertexHeights`) | |
| `FoldBack` / `FoldClosure` / `FoldSector` / `FoldSide` / `FoldFlats` | :3637/:3549/:3593/:3201/:3464 | tic pass | candidate sector | ≤500 sides × Process a pass | `FoldMS`, `FoldMaxMS` |
| `RunLightPatch` / `PatchSectorLights` / `RecomputeLight` / `LightOwner` / `DirectLightInput` | :2994/:2744/:2706/:2622/:2688 | tic pass | moved sector / record | R; LightOwner may walk the sector's lines | `PassPatchMS` |
| `EvaluateSideLight` / `CheckSideParts` | :2952/:2821 | RunLightPatch | side on moved sector | C, or Process | |
| `RunFlatLightPatch` / `PatchSectorFlatLights` / `RecomputeFlatLight` / `FlatLightInputNow` | :3109/:3089/:3061/:1572 | tic pass | moved sector | C | |
| `RefreshMovingSectors` | :3892 | RunChangePass | frame between passes | moving sectors × lines | `PassMovingMS` |
| `WriteFlatPlaneRecord` / `WriteAllFlatPlaneRecords` | :1625/:1634 | tic pass, moving, fold, BeginScene, DrawFlats | sector | C | |
| `ClearEvictions` | :4167 | BeginScene | frame | evicted count | not clocked |
| `LevelResidency_SkipWall` → `SkipWallDecision` | :4410/:4327 | AddLine hw_bsp.cpp:661,760; worker :346 | reached seg, per scene | C + one side hash a frame + sky PutPortal replays | inside `ClipWall`+`SetupWall` (single-thread); `WalkClock` opt. |
| `VerifySide` | :4088 | SkipWallDecision | reached side (verify only) | Process | dormant |
| `Evict` / `EvictFlat` | :4155/:4190 | decisions, fold | event | surfaces of side | writes mapped SSBO |
| `LevelResidency_FlatPlanes` / `FlatDrawableNow` | :4464/:4427 | ProcessVisibleSubsector hw_bsp.cpp:1158; worker :368 | reached section | C (+TexMan lookup, damage key) | inside `SetupFlat` |
| `SceneRegion` | :4508 | ranges | scene | O(scenes) | |
| `MakeSceneRanges` | :4522 | Draw | scene | sort + reached surfaces + memcpy | `IndexBuildMS` |
| `LevelResidency_Draw` | :4632 | RenderScene hw_drawinfo.cpp:3830 | scene | groups × (SetMaterial+DrawIndexed) | in `RenderAll`, GPU `opaque.walls` |
| `ObserveOldFlats` | :4705 | DrawFlats | scene while undecided/verify | plain flats in list | |
| `MakeSceneFlatRanges` | :4776 | DrawFlats | scene | sort + runs + memcpy | `FlatIndexBuildMS` |
| `LevelResidency_DrawFlats` | :4864 | RenderScene hw_drawinfo.cpp:3835 | scene | flat groups × draw | GPU `opaque.flats` |
| `LevelResidency_Reset` | :4207 | P_FreeLevelData p_setup.cpp:643 | level | free | |
| `LevelMeshBuffers::Allocate/Free/WriteVertices/WriteIndices/IsDrawable` | hw_levelbuffers.cpp:77/155/181/190/172 | Build/Reset/FoldSide/BeginScene | — | C / SetSubData copy | |
| `WallCache_Process` | hw_wallcache.cpp:313 | AddLine :763, worker :349 | processed wall | C at 0; hashes + replay at ≥1 | dormant |
| `WallCache_SideFastHash` | hw_wallcache.cpp:308 | SkipWallDecision, sidescan, fold, build | side | ~byte range of side_t + line | |
| `R_SnapshotWrite` | r_snapshot.cpp:278 | TryRunTics d_net.cpp:2493,2557 | frame | C at 0; actors+nodes at 1 | `snapshotstats` |

### B.11 Hooks, cvars and dead code in this area

| item | status | evidence |
|---|---|---|
| `R_SnapshotAcquire`, `R_SnapshotRelease` | **DEAD** (defined, never called) | grep over src: only `r_snapshot.{h,cpp}` |
| `FWorldSnapshot::ActorImage/Sectors()/Sides()/Lines()` | **DEAD** | no callers outside `r_snapshot.h` |
| `r_renderthread` | **DEAD/stub**: callback resets it to false | `r_snapshot.cpp:33-40` |
| `r_snapshot` 1/2/3 | DORMANT; 1 writes for no reader | B.9 |
| `WallCache_SectorHash`, `WallCache_SideHash`, `WallCache_SectorFastHash` | **DEAD** (exported, never called) | grep: definitions only (`hw_wallcache.cpp:302,303,307`) |
| `WallCache_NoBackSector` enum | **DEAD** | 0 references outside its header |
| `HWWall::SetTierLight` | **DEAD** (only `SetTierLightLevel` is used) | `hw_walls.cpp:2448`; no callers |
| `LevelMeshBuffers::StorageCapacityBytes` | **DEAD** | no callers |
| `r_wallcache` + `WallCache_Process` replay | DORMANT, superseded by the mesh, still called per wall | `hw_bsp.cpp:349,763` |
| `r_levelmesh_verify`, `_sidescan`, `_clock`, `_drawall` | DORMANT dev switches | B.2 |
| `r_levelmesh_foldback` | ON by default | `:155` |
| `EFF_LEVELMESH` on GL/GLES | never compiled (skipped) | `gl_shader.cpp:1236`, `gles_shader.cpp:858` |
| `SectorBuildHash`, `SectorLightHash`, `SectorPlaneBuildHash` arrays | computed at build for **every** sector and at fold for the folded sector, read only under verify [xc: fold recomputes one sector, not all] | `:2138-2147`, `:3605-3606`, `:3709-3762` — dead weight at verify 0 |
| `HWMeshHelper::recordOrder/order` | used by the wall cache only | `hw_wallcache.cpp:369,396` |
| `sim_prefetch` | DORMANT (0) | `dthinker.cpp` |
| `sim_maxcatchup`, `snd_evictretry`, `snd_evictbudget` | ACTIVE by default (non-zero) | A.2.1 |
| `simprofile`, `ticprofile`, `soundchannels`, `levelstats`, `levelmeshstats`, `wallcachestats`, `snapshotstats` | CCMDs; their clocks run always (`d_net.cpp`, `p_tick.cpp`, `s_doomsound.cpp`) | |

### B.12 Data structures that matter for speed

- `FSideEntry` array, indexed by side (all sides, static or not): touched per reached seg; ~72 B (**I**).
- `SectorShadow` (264 B) and `SectorChanged`/`SectorPlaneChanged`/`SectorLightStale` (1 B each), by sector. The tic
  pass streams the whole shadow plus three or more cache lines of each live `sector_t`. Prev. measured: "bound by
  `sector_t`'s layout (about ten cache lines a sector)" (HANDOFF_RENDER_2026-11-11.md).
- `VertexChanged`/`VertexSeen`/`VertexBuildHash` by vertex.
- `SurfaceRuns` (24 B) by surface id, `StaticIndices` CPU copy: the per-scene index copy reads both.
- `Groups`/`FlatGroups`: at most one draw each per scene; groups with no reached indices are skipped
  (`if (count == 0) continue;`, `:4680`, `:4914`), so the number of groups is the ceiling, not the floor, on mesh
  draw calls [xc: was "floor"].
- Mapped `LevelMeshSurfaceRecord[]` (32 B) and `LevelMeshLightRecord[]` (320 B): read per vertex (and, for lights,
  per fragment) by the GPU; written by the CPU for evictions, patches and folds.
- Per-frame index buffer: written sequentially by the CPU each scene, read by the GPU as an index buffer.

### B.13 What slows it down (ranked)

| # | cost | evidence | confidence |
|---|---|---|---|
| 1 | The BSP walk, clipper and everything around them still run in full; the mesh only replaces `Process` + per-part draws | `hw_bsp.cpp:581-731` before the hook at `:760`; ranges from `ReachedSides` | high (V) |
| 2 | Tic pass over all sectors, every tic, whatever is visible; prev. measured 1.42-1.43 ms a pass on Sunder; alternating frame cost in VR | `RunTicPass :3713-3805` | high (V + prev. measured) |
| 3 | Evictions send whole sides back to the old path (dynamic lights, damage, decals, dither, movers); with lights on they dominate | `:4358-4380`; prev. measured eviction counts | high |
| 4 | Fold-back `HWWall::Process` bursts (≤500 sides a pass, first closure uncapped) | `:3654-3668` | medium (no measured ms quoted) |
| 5 | Per reached side, once a frame: `WallCache_SideFastHash` over several hundred bytes of `side_t` | `:4371-4376` | medium (I on size) |
| 6 | GPU: per-vertex and per-fragment SSBO indirection; record and index memory placement unknown | `levelmesh.vp`, `main.fp` LEVEL_MESH | medium (I) |
| 7 | Sky sides: `HWWall` copy + `PutPortal` per side per scene | `:4388-4401` | medium |
| 8 | Memory: worst-case slots, staging duplicates, CPU index copies, 2× per-frame buffer | B.6 | high (V) |
| 9 | Per-frame index sort + copy (prev. measured 0.16 ms walls, 0.07 ms flats) | `MakeSceneRanges`, `MakeSceneFlatRanges` | high |
| 10 | Build on the first frame of a level / global change, inside the frame | `:4242-4247` | high (V); ms unmeasured |

---

## Open questions and the measurements that would settle them

1. **Does the static path ever win on CPU in VR?** Run a two-eye non-multiview VR scene, `r_dynlights 1` (the
   owner's real setting), `r_newrenderer 0` vs 3. Compare perf_track `bsp`, `clip`, `walls`, `flats`, `scene`,
   `drawcalls`, and `levelmeshstats` (`PassMS/PassMaxMS`, `Skipped/Evicted per frame`, `FoldMS`). All existing
   pairs are flat-screen with `r_dynlights 0`.
2. **Alternating frame cost**: per-frame `bsp` column frame by frame at 72 Hz. Pass frames vs reuse frames
   (`FrameStats.Passes` vs `PassesReused`).
3. **Camera textures and the 2-region cap**: a map with a camera texture in VR non-multiview. Watch `levelmeshstats`
   `ScenesNoRoom`, `DrawScenesWhole`, "scenes found no room". Expect >0 if hazard B.8.1 is real.
4. **Memory placement of the Persistent SSBOs and the per-frame index buffer**: VMA statistics or a
   `vkGetBufferMemoryRequirements` / memory-type dump. Then the GPU `opaque.walls` group mesh vs old at equal
   indices.
5. **True memory footprint**: process RSS and VRAM (OS tools) with `r_newrenderer 0` vs 3 on Abandon. `levelmeshstats`
   misses staging and CPU copies.
6. **Fold-back spikes**: `FoldMaxMS` on a map with many doors/lifts (Sunder), and the frame-time histogram around
   door closes.
7. **Build time**: the `levelmesh: built … ms` console line on Abandon/Sunder. It is a hitch on the first frame and
   every 35 frames while a global input (e.g. `r_fakecontrast`) is moving.
8. **Old-path regressions from renderer at defaults**: compare pacing and audio with `sim_maxcatchup 0`,
   `snd_evictretry 0`, `snd_evictbudget 0` (stock behaviour) against the defaults on renderer, before any merge to
   main.
9. **Merge**: renderer carries fixes main lacks (D_Display ga_fullconsole guard, demo buffer) and main carries 76
   commits renderer lacks. The two only meet in `vk_openxrdevice.cpp`, `g_game.cpp` and `CMakeLists.txt` (V: the
   changed-file lists intersect only there).

---

## Appendix B — main vs netplay, all 63 files (all changed on main since netplay)

**playsim** (18)
`playsim/actor.h` +8/-0, `playsim/p_local.h` +3/-0, `playsim/p_map.cpp` +49/-7, `playsim/p_mobj.cpp` +5/-4,
`playsim/p_user.cpp` +172/-4, `playsim/p_vrdemo.cpp` +5/-0, `playsim/p_vrdemo.h` +5/-0, `playsim/p_vrgear.cpp`
+2114/-0, `playsim/p_vrgear.h` +118/-0, `playsim/p_vrgrab.cpp` +1397/-0, `playsim/p_vrgrab.h` +102/-0,
`playsim/p_vrgunfit.cpp` +122/-0, `playsim/p_vrgunfit.h` +37/-0, `playsim/p_vrreplay.cpp` +5/-1,
`playsim/p_vrslots.h` +40/-0, `playsim/p_vrsynclog.cpp` +49/-3, `playsim/p_vrtorso.cpp` +589/-0,
`playsim/p_vrtorso.h` +94/-0

**r_data** (13)
`r_data/models.cpp` +515/-18, `r_data/models.h` +1/-1, `r_data/vr_avatar.cpp` +41/-0, `r_data/vr_avatar.h` +4/-0,
`r_data/vr_avatar_pose.cpp` +120/-2, `r_data/vr_body.cpp` +715/-5, `r_data/vr_body.h` +7/-0,
`r_data/vr_bodyinfer.h` +3/-0, `r_data/vr_bodyinfer_input.cpp` +95/-8, `r_data/vr_grasp.cpp` +351/-0,
`r_data/vr_grasp.h` +35/-0, `r_data/vr_rig.cpp` +332/-4, `r_data/vr_rig.h` +26/-0

**top-level src** (10)
`CMakeLists.txt` +6/-0, `d_net.cpp` +92/-4, `d_netinf.h` +1/-0, `d_netinfo.cpp` +38/-0, `d_protocol.cpp` +11/-2,
`d_protocol.h` +26/-3, `g_game.cpp` +66/-42, `g_level.cpp` +9/-0, `p_saveg.cpp` +34/-1, `version.h` +2/-2

**scene** (4)
`rendering/hwrenderer/scene/hw_drawinfo.cpp` +9/-0, `hw_drawinfo.h` +1/-0, `hw_weapon.cpp` +91/-14,
`hw_weapon.h` +5/-0

**engine** (3) `common/engine/i_net.cpp` +3/-0, `common/engine/i_upnp.cpp` +232/-0, `common/engine/i_upnp.h` +4/-0
· **scripting** (3) `scripting/vmthunks.cpp` +257/-0, `scripting/vmthunks_actors.cpp` +1/-0,
`scripting/zscript/zcc_compile_doom.cpp` +7/-2 · **QzDoom** (2) `QzDoom/VrCommon.h` +17/-0,
`QzDoom/qzdoom_common.cpp` +64/-3 · **vulkan** (2) `vk_openxrdevice.cpp` +50/-25, `vk_openxrdevice.h` +9/-4 ·
**textures** (2) `texture.cpp` +18/-0, `textures.h` +4/-8 · **rendering other** (2) `hw_models.cpp` +5/-2,
`r_utility.cpp` +2/-2 · **wadsrc** (3) `zscript/actors/actor.zs` +3/-0, `zscript/doombase.zs` +146/-0,
`shaders/glsl/main.fp` +7/-1 · **model formats** (1) `common/models/modelrenderer.h` +6/-0

## Appendix C — main vs selaco, all 389 files (S = only selaco changed it since 5.0.3, M = only main, SM = both)

**playsim** (55 files: S 3, M 38, SM 14)  
`playsim/a_dynlight.cpp` SM +66/-6, `playsim/a_dynlight.h` SM +1/-28, `playsim/actor.h` SM +106/-977, `playsim/bots/b_bot.cpp` S +16/-1, `playsim/d_player.h` SM +8/-49, `playsim/dthinker.cpp` SM +2/-14, `playsim/dthinker.h` SM +1/-6, `playsim/fragglescript/t_func.cpp` M +3/-6, `playsim/mapthinkers/a_lightning.cpp` M +5/-10, `playsim/mapthinkers/a_lights.cpp` M +28/-64, `playsim/p_actionfunctions.cpp` SM +305/-381, `playsim/p_affect.cpp` M +0/-946, `playsim/p_affect.h` M +0/-293, `playsim/p_beamoutline.cpp` M +0/-1028, `playsim/p_beamoutline.h` M +0/-300, `playsim/p_effect.cpp` M +2/-11, `playsim/p_enemy.cpp` M +25/-583, `playsim/p_enemy.h` M +1/-3, `playsim/p_floorregion.cpp` M +0/-1242, `playsim/p_floorregion.h` M +0/-213, `playsim/p_guard.cpp` M +0/-884, `playsim/p_guard.h` M +0/-171, `playsim/p_interaction.cpp` M +0/-50, `playsim/p_local.h` SM +0/-158, `playsim/p_map.cpp` SM +34/-1534, `playsim/p_mobj.cpp` SM +19/-282, `playsim/p_pooledparticles.cpp` S +2157/-0, `playsim/p_pooledparticles.h` S +296/-0, `playsim/p_pspr.cpp` SM +28/-80, `playsim/p_pspr.h` SM +14/-42, `playsim/p_sectors.cpp` SM +4/-16, `playsim/p_spec.cpp` M +3/-12, `playsim/p_user.cpp` SM +128/-373, `playsim/p_visualthinker.h` M +0/-1, `playsim/p_vrdemo.cpp` M +0/-913, `playsim/p_vrdemo.h` M +0/-138, `playsim/p_vrgear.cpp` M +0/-2114, `playsim/p_vrgear.h` M +0/-118, `playsim/p_vrghost.cpp` M +0/-338, `playsim/p_vrghost.h` M +0/-6, `playsim/p_vrgrab.cpp` M +0/-1397, `playsim/p_vrgrab.h` M +0/-102, `playsim/p_vrgunfit.cpp` M +0/-122, `playsim/p_vrgunfit.h` M +0/-37, `playsim/p_vrhand.cpp` M +0/-265, `playsim/p_vrhand.h` M +0/-100, `playsim/p_vrreplay.cpp` M +0/-294, `playsim/p_vrreplay.h` M +0/-46, `playsim/p_vrslots.h` M +0/-40, `playsim/p_vrsynclog.cpp` M +0/-220, `playsim/p_vrsynclog.h` M +0/-3, `playsim/p_vrtelemetry.cpp` M +0/-192, `playsim/p_vrtelemetry.h` M +0/-31, `playsim/p_vrtorso.cpp` M +0/-589, `playsim/p_vrtorso.h` M +0/-94

**top-level src (game, net, setup, menus...)** (42 files: S 12, M 14, SM 16)  
`CMakeLists.txt` SM +4/-61, `am_map.cpp` S +59/-0, `common/2d/wipe.cpp` M +0/-31, `common/console/c_console.cpp` M +1/-23, `common/menu/joystickmenu.cpp` S +67/-0, `common/menu/menu.cpp` S +18/-3, `common/menu/menu.h` S +4/-0, `common/menu/resolutionmenu.cpp` M +1/-1, `common/menu/savegamemanager.cpp` S +32/-8, `common/menu/savegamemanager.h` S +5/-4, `common/statusbar/base_sbar.cpp` S +4/-0, `console/c_cmds.cpp` S +7/-0, `d_iwad.cpp` SM +89/-34, `d_main.cpp` SM +60/-205, `d_main.h` SM +4/-0, `d_net.cpp` SM +12/-747, `d_net.h` M +0/-1, `d_netinf.h` M +0/-1, `d_netinfo.cpp` SM +1/-38, `d_protocol.cpp` M +6/-327, `d_protocol.h` M +2/-177, `events.cpp` SM +248/-13, `events.h` SM +50/-6, `g_benchdemo.cpp` M +0/-106, `g_cvars.cpp` M +1/-3, `g_game.cpp` SM +147/-232, `g_game.h` S +2/-2, `g_level.cpp` SM +71/-317, `g_levellocals.h` SM +21/-1052, `g_perfbench.cpp` M +0/-412, `g_perfbench.h` M +0/-134, `g_statusbar/sbarinfo.cpp` S +9/-1, `m_misc.cpp` SM +187/-70, `menu/doommenu.h` S +1/-1, `menu/loadsavemenu.cpp` S +26/-2, `menu/profiledef.cpp` M +0/-3, `p_saveg.cpp` SM +13/-52, `p_setup.cpp` SM +1/-54, `p_tick.cpp` SM +13/-344, `version.h` SM +2/-2, `widgets/settingspage.cpp` M +1/-4, `widgets/settingspage.h` M +0/-6

**hwrenderer common data** (34 files: S 0, M 12, SM 22)  
`common/rendering/hwrenderer/data/hw_clock.cpp` SM +0/-1, `common/rendering/hwrenderer/data/hw_clock.h` SM +0/-5, `common/rendering/hwrenderer/data/hw_cvars.cpp` SM +14/-141, `common/rendering/hwrenderer/data/hw_cvars.h` SM +1/-20, `common/rendering/hwrenderer/data/hw_drawnlinebuffer.cpp` SM +1/-20, `common/rendering/hwrenderer/data/hw_drawnlinebuffer.h` SM +1/-18, `common/rendering/hwrenderer/data/hw_effectlightbuffer.cpp` M +0/-160, `common/rendering/hwrenderer/data/hw_effectlightbuffer.h` M +0/-159, `common/rendering/hwrenderer/data/hw_effectsgovernor.cpp` M +0/-162, `common/rendering/hwrenderer/data/hw_effectsgovernor.h` M +0/-348, `common/rendering/hwrenderer/data/hw_emissivevolumeframe.h` M +0/-137, `common/rendering/hwrenderer/data/hw_framecompute.h` SM +0/-65, `common/rendering/hwrenderer/data/hw_gpuparticlebuffer.cpp` SM +0/-135, `common/rendering/hwrenderer/data/hw_gpuparticlebuffer.h` SM +0/-26, `common/rendering/hwrenderer/data/hw_gpuparticlewindow.h` M +0/-241, `common/rendering/hwrenderer/data/hw_meshparticles.cpp` SM +1/-8, `common/rendering/hwrenderer/data/hw_meshparticles.h` SM +1/-12, `common/rendering/hwrenderer/data/hw_particledefbuffer.cpp` SM +16/-40, `common/rendering/hwrenderer/data/hw_particledefbuffer.h` SM +8/-84, `common/rendering/hwrenderer/data/hw_perflog.cpp` SM +6/-357, `common/rendering/hwrenderer/data/hw_perflog.h` SM +1/-12, `common/rendering/hwrenderer/data/hw_perftrack.cpp` M +0/-846, `common/rendering/hwrenderer/data/hw_perftrack.h` M +0/-169, `common/rendering/hwrenderer/data/hw_renderstate.h` SM +0/-99, `common/rendering/hwrenderer/data/hw_screentiles.h` M +0/-241, `common/rendering/hwrenderer/data/hw_shaderpatcher.cpp` SM +0/-13, `common/rendering/hwrenderer/data/hw_shadowmap.cpp` M +1/-73, `common/rendering/hwrenderer/data/hw_shadowmap.h` M +1/-28, `common/rendering/hwrenderer/data/hw_smoketilecover.h` M +0/-610, `common/rendering/hwrenderer/data/hw_viewpointbuffer.cpp` SM +1/-10, `common/rendering/hwrenderer/data/hw_viewpointuniforms.h` SM +2/-112, `common/rendering/hwrenderer/data/hw_vrmodes.cpp` SM +46/-765, `common/rendering/hwrenderer/data/hw_vrmodes.h` SM +4/-286, `common/rendering/hwrenderer/data/hw_vrwheel.cpp` SM +2/-6

**engine/platform/utility** (33 files: S 14, M 18, SM 1)  
`common/engine/d_gui.h` S +4/-0, `common/engine/i_hangwatch.h` M +0/-81, `common/engine/i_interface.cpp` M +34/-96, `common/engine/i_interface.h` M +0/-9, `common/engine/i_net.cpp` M +8/-197, `common/engine/i_net.h` M +0/-39, `common/engine/i_solarxr.cpp` M +0/-624, `common/engine/i_solarxr.h` M +0/-68, `common/engine/i_upnp.cpp` M +0/-232, `common/engine/i_upnp.h` M +0/-4, `common/engine/i_wsclient.cpp` M +0/-264, `common/engine/i_wsclient.h` M +0/-66, `common/engine/m_haptics.cpp` S +46/-0, `common/engine/m_haptics.h` S +5/-0, `common/engine/m_joy.cpp` S +41/-0, `common/engine/m_joy.h` S +14/-0, `common/engine/sc_man.h` M +0/-6, `common/platform/posix/sdl/i_input.cpp` S +7/-0, `common/platform/posix/sdl/i_joystick.cpp` S +52/-16, `common/platform/posix/sdl/i_system.cpp` M +1/-10, `common/platform/posix/sdl/sdlglvideo.cpp` S +13/-0, `common/platform/win32/i_crash.cpp` M +0/-95, `common/platform/win32/i_dijoy.cpp` SM +19/-0, `common/platform/win32/i_hangwatch.cpp` M +0/-289, `common/platform/win32/i_input.cpp` S +4/-0, `common/platform/win32/i_main.cpp` M +1/-8, `common/platform/win32/i_xinput.cpp` S +59/-10, `common/platform/win32/win32basevideo.cpp` S +25/-4, `common/platform/win32/win32basevideo.h` S +1/-0, `common/utility/TSQueue.h` M +1/-46, `common/utility/engineerrors.cpp` S +14/-0, `common/utility/engineerrors.h` S +12/-0, `common/utility/i_time.cpp` M +0/-38

**r_data (models, VR body/rig)** (30 files: S 3, M 23, SM 4)  
`r_data/a_dynlightdata.cpp` SM +0/-25, `r_data/gldefs.cpp` SM +0/-17, `r_data/model_handdrive.cpp` M +0/-669, `r_data/model_handdrive.h` M +0/-178, `r_data/model_jointdrive.cpp` M +0/-1079, `r_data/model_jointdrive.h` M +0/-81, `r_data/model_jointdrive_math.h` M +0/-415, `r_data/model_reach.cpp` S +1349/-0, `r_data/model_reach.h` S +102/-0, `r_data/model_reach_math.h` S +1105/-0, `r_data/models.cpp` SM +752/-1778, `r_data/models.h` SM +1/-11, `r_data/vr_armik.cpp` M +0/-943, `r_data/vr_armik.h` M +0/-250, `r_data/vr_armsolver.cpp` M +0/-398, `r_data/vr_armsolver.h` M +0/-70, `r_data/vr_avatar.cpp` M +0/-1341, `r_data/vr_avatar.h` M +0/-386, `r_data/vr_avatar_pose.cpp` M +0/-3470, `r_data/vr_body.cpp` M +0/-3559, `r_data/vr_body.h` M +0/-289, `r_data/vr_bodyinfer.cpp` M +0/-546, `r_data/vr_bodyinfer.h` M +0/-362, `r_data/vr_bodyinfer_input.cpp` M +0/-1317, `r_data/vr_bodyinfer_record.cpp` M +0/-1273, `r_data/vr_grasp.cpp` M +0/-351, `r_data/vr_grasp.h` M +0/-35, `r_data/vr_rig.cpp` M +0/-2501, `r_data/vr_rig.h` M +0/-298, `r_data/vr_rig_selftest.cpp` M +0/-1709

**vulkan backend** (28 files: S 0, M 4, SM 24)  
`common/rendering/vulkan/renderer/vk_compute.cpp` SM +0/-8, `common/rendering/vulkan/renderer/vk_compute.h` SM +0/-6, `common/rendering/vulkan/renderer/vk_descriptorset.cpp` SM +9/-37, `common/rendering/vulkan/renderer/vk_emissivevolumes.cpp` M +0/-222, `common/rendering/vulkan/renderer/vk_emissivevolumes.h` M +0/-73, `common/rendering/vulkan/renderer/vk_postprocess.cpp` SM +10/-59, `common/rendering/vulkan/renderer/vk_postprocess.h` SM +1/-2, `common/rendering/vulkan/renderer/vk_renderpass.cpp` SM +10/-71, `common/rendering/vulkan/renderer/vk_renderpass.h` SM +2/-11, `common/rendering/vulkan/renderer/vk_renderstate.cpp` SM +2/-23, `common/rendering/vulkan/renderer/vk_renderstate.h` SM +1/-9, `common/rendering/vulkan/renderer/vk_smokevolume.cpp` SM +8/-513, `common/rendering/vulkan/renderer/vk_smokevolume.h` SM +1/-68, `common/rendering/vulkan/shaders/vk_shader.cpp` SM +38/-114, `common/rendering/vulkan/shaders/vk_shader.h` SM +24/-41, `common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp` SM +224/-2031, `common/rendering/vulkan/stereo3d/vk_openxrdevice.h` SM +0/-96, `common/rendering/vulkan/system/vk_commandbuffer.cpp` SM +3/-250, `common/rendering/vulkan/system/vk_commandbuffer.h` SM +0/-20, `common/rendering/vulkan/system/vk_renderdevice.cpp` SM +21/-267, `common/rendering/vulkan/system/vk_renderdevice.h` SM +1/-16, `common/rendering/vulkan/textures/vk_framebuffer.cpp` M +0/-10, `common/rendering/vulkan/textures/vk_hwtexture.cpp` SM +0/-4, `common/rendering/vulkan/textures/vk_pptexture.cpp` M +0/-1, `common/rendering/vulkan/textures/vk_renderbuffers.cpp` SM +0/-74, `common/rendering/vulkan/textures/vk_renderbuffers.h` SM +0/-19, `common/rendering/vulkan/textures/vk_texture.cpp` SM +73/-293, `common/rendering/vulkan/textures/vk_texture.h` SM +0/-11

**shaders** (24 files: S 0, M 11, SM 13)  
`wadsrc/static/shaders/compute/smoke_light.comp` SM +0/-464, `wadsrc/static/shaders/glsl/beamglow.glsl` M +0/-499, `wadsrc/static/shaders/glsl/burn.fp` SM +0/-8, `wadsrc/static/shaders/glsl/drawnlines.fp` SM +70/-117, `wadsrc/static/shaders/glsl/fogboundary.fp` SM +0/-8, `wadsrc/static/shaders/glsl/func_neon.fp` M +0/-551, `wadsrc/static/shaders/glsl/gpuparticles.fp` SM +4/-88, `wadsrc/static/shaders/glsl/gpuparticles.vp` SM +0/-242, `wadsrc/static/shaders/glsl/main.fp` SM +226/-949, `wadsrc/static/shaders/glsl/meshparticles.fp` SM +0/-8, `wadsrc/static/shaders/glsl/meshparticles.vp` SM +2/-94, `wadsrc/static/shaders/glsl/stencil.fp` SM +0/-11, `wadsrc/static/shaders/pp/emissivevolume.fp` M +0/-688, `wadsrc/static/shaders/pp/exposureimpulse.fp` M +0/-125, `wadsrc/static/shaders/pp/heatwarp.fp` SM +0/-12, `wadsrc/static/shaders/pp/present.fp` M +0/-34, `wadsrc/static/shaders/pp/scenemask.fp` M +0/-47, `wadsrc/static/shaders/pp/screentilemask.fp` M +0/-38, `wadsrc/static/shaders/pp/sharedmarchwarp.fp` M +0/-324, `wadsrc/static/shaders/pp/shockwaveoffset.fp` M +0/-175, `wadsrc/static/shaders/pp/smokecomposite.fp` SM +43/-25, `wadsrc/static/shaders/pp/smokemarch.fp` SM +10/-340, `wadsrc/static/shaders/pp/smoketemporal.fp` M +0/-256, `wadsrc/static/shaders/pp/tonemapfilmic.fp` M +0/-143

**zscript** (23 files: S 7, M 6, SM 10)  
`wadsrc/static/zscript.txt` SM +3/-23, `wadsrc/static/zscript/actors/actor.zs` SM +75/-865, `wadsrc/static/zscript/actors/attacks.zs` M +0/-8, `wadsrc/static/zscript/actors/inventory/stateprovider.zs` SM +1/-2, `wadsrc/static/zscript/actors/inventory/weapons.zs` M +2/-4, `wadsrc/static/zscript/actors/player/player.zs` SM +29/-218, `wadsrc/static/zscript/actors/shared/dynlights.zs` SM +13/-10, `wadsrc/static/zscript/actors/shared/soundsequence.zs` S +219/-4, `wadsrc/static/zscript/actors/standalone/stockstandins.zs` S +157/-0, `wadsrc/static/zscript/constants.zs` SM +0/-313, `wadsrc/static/zscript/doombase.zs` SM +9/-1319, `wadsrc/static/zscript/engine/base.zs` SM +29/-0, `wadsrc/static/zscript/engine/datavalidator.zs` M +0/-25, `wadsrc/static/zscript/engine/inputevents.zs` S +4/-0, `wadsrc/static/zscript/engine/ui/menu/loadsavemenu.zs` S +4/-2, `wadsrc/static/zscript/engine/ui/menu/menu.zs` S +33/-0, `wadsrc/static/zscript/engine/ui/menu/optionmenuitems.zs` S +11/-0, `wadsrc/static/zscript/events.zs` SM +41/-32, `wadsrc/static/zscript/mapdata.zs` SM +102/-38, `wadsrc/static/zscript/particledefinition.zs` S +372/-0, `wadsrc/static/zscript/rs_bout.zs` M +0/-982, `wadsrc/static/zscript/rs_boutring.zs` M +0/-614, `wadsrc/static/zscript/rs_chasebrain.zs` M +0/-347

**hwrenderer effects (smoke, debris, damage, fields)** (21 files: S 0, M 12, SM 9)  
`rendering/hwrenderer/hw_debrislanding.cpp` SM +1/-4, `rendering/hwrenderer/hw_debrispool.cpp` SM +0/-231, `rendering/hwrenderer/hw_debrispool.h` SM +0/-13, `rendering/hwrenderer/hw_dynlightdata.cpp` M +0/-48, `rendering/hwrenderer/hw_effectlightcore.h` M +0/-834, `rendering/hwrenderer/hw_effectlights.cpp` M +0/-568, `rendering/hwrenderer/hw_effectlights.h` M +0/-155, `rendering/hwrenderer/hw_emissivevolumecore.h` M +0/-810, `rendering/hwrenderer/hw_emissivevolumes.cpp` M +0/-559, `rendering/hwrenderer/hw_emissivevolumes.h` M +0/-118, `rendering/hwrenderer/hw_entrypoint.cpp` SM +4/-121, `rendering/hwrenderer/hw_exposureimpulse.cpp` M +0/-277, `rendering/hwrenderer/hw_exposureimpulse.h` M +0/-78, `rendering/hwrenderer/hw_exposureimpulsecore.h` M +0/-334, `rendering/hwrenderer/hw_models.cpp` SM +2/-63, `rendering/hwrenderer/hw_models.h` SM +0/-12, `rendering/hwrenderer/hw_particlelights.h` M +0/-554, `rendering/hwrenderer/hw_precache.cpp` SM +28/-10, `rendering/hwrenderer/hw_shockwavecore.h` M +0/-399, `rendering/hwrenderer/hw_smokevolume.cpp` SM +1/-821, `rendering/hwrenderer/hw_smokevolume.h` SM +0/-51

**scene (BSP/walls/flats/sprites/drawinfo)** (16 files: S 2, M 5, SM 9)  
`rendering/hwrenderer/scene/hw_bsp.cpp` S +51/-0, `rendering/hwrenderer/scene/hw_decal.cpp` SM +0/-9, `rendering/hwrenderer/scene/hw_drawinfo.cpp` SM +26/-1008, `rendering/hwrenderer/scene/hw_drawinfo.h` SM +1/-13, `rendering/hwrenderer/scene/hw_drawlist.cpp` S +1/-1, `rendering/hwrenderer/scene/hw_drawstructs.h` SM +6/-8, `rendering/hwrenderer/scene/hw_flats.cpp` SM +7/-20, `rendering/hwrenderer/scene/hw_portal.cpp` M +1/-109, `rendering/hwrenderer/scene/hw_portal.h` M +0/-10, `rendering/hwrenderer/scene/hw_sdffont.cpp` M +1/-9, `rendering/hwrenderer/scene/hw_skyportal.cpp` M +0/-22, `rendering/hwrenderer/scene/hw_spritelight.cpp` M +1/-3, `rendering/hwrenderer/scene/hw_sprites.cpp` SM +294/-226, `rendering/hwrenderer/scene/hw_walls.cpp` SM +2/-52, `rendering/hwrenderer/scene/hw_weapon.cpp` SM +20/-144, `rendering/hwrenderer/scene/hw_weapon.h` SM +0/-5

**gamedata (defs)** (14 files: S 5, M 4, SM 5)  
`gamedata/a_keys.cpp` S +7/-0, `gamedata/datavalidation.cpp` M +0/-116, `gamedata/datavalidation.h` M +0/-13, `gamedata/decallib.cpp` S +8/-0, `gamedata/g_doomedmap.cpp` S +31/-0, `gamedata/g_mapinfo.cpp` SM +0/-6, `gamedata/g_mapinfo.h` SM +1/-1, `gamedata/g_skill.cpp` S +29/-0, `gamedata/info.cpp` S +7/-1, `gamedata/particledefs.cpp` SM +42/-755, `gamedata/particledefs.h` SM +0/-125, `gamedata/r_defs.h` SM +4/-114, `gamedata/volumedefs.cpp` M +0/-503, `gamedata/volumedefs.h` M +0/-41

**model formats** (13 files: S 0, M 3, SM 10)  
`common/models/bonecomponents.h` M +0/-9, `common/models/model.cpp` SM +1/-31, `common/models/model.h` SM +1/-97, `common/models/model_iqm.h` SM +0/-39, `common/models/model_md2.h` SM +1/-10, `common/models/model_mdl.h` SM +0/-2, `common/models/model_studio.h` M +0/-190, `common/models/modelrenderer.h` SM +0/-16, `common/models/models_iqm.cpp` SM +0/-95, `common/models/models_md2.cpp` SM +0/-57, `common/models/models_md3.cpp` SM +0/-14, `common/models/models_mdl.cpp` SM +0/-53, `common/models/models_studio.cpp` M +0/-1279

**sound** (10 files: S 1, M 6, SM 3)  
`common/audio/sound/i_sound.h` M +0/-5, `common/audio/sound/i_soundinternal.h` M +0/-4, `common/audio/sound/oalsound.cpp` M +13/-221, `common/audio/sound/oalsound.h` M +0/-17, `common/audio/sound/s_sound.cpp` SM +5/-34, `common/audio/sound/s_soundinternal.h` SM +0/-30, `sound/s_advsound.cpp` S +216/-46, `sound/s_doomsound.cpp` SM +0/-37, `sound/s_hearingimpulse.cpp` M +0/-348, `sound/s_hearingimpulse.h` M +0/-252

**textures** (9 files: S 0, M 3, SM 6)  
`common/textures/formats/ddstexture.cpp` SM +0/-6, `common/textures/formats/vtftexture.cpp` M +0/-508, `common/textures/gametexture.cpp` SM +1/-4, `common/textures/image.cpp` SM +79/-170, `common/textures/image.h` SM +0/-23, `common/textures/texture.cpp` M +1/-22, `common/textures/texturemanager.cpp` SM +51/-0, `common/textures/texturemanager.h` SM +2/-0, `common/textures/textures.h` M +8/-7

**GL/GLES backend** (8 files: S 4, M 1, SM 3)  
`common/rendering/gl/gl_framebuffer.cpp` S +2/-2, `common/rendering/gl/gl_framebuffer.h` S +1/-1, `common/rendering/gl/gl_postprocess.cpp` S +2/-2, `common/rendering/gl/gl_renderbuffers.cpp` M +0/-1, `common/rendering/gl/gl_renderer.h` S +1/-1, `common/rendering/gl/gl_renderstate.cpp` SM +3/-12, `common/rendering/gl/gl_shader.cpp` SM +0/-17, `common/rendering/gl/gl_shader.h` SM +0/-1

**scripting VM** (8 files: S 0, M 0, SM 8)  
`common/scripting/frontend/zcc_parser.cpp` SM +28/-0, `common/scripting/frontend/zcc_parser.h` SM +1/-0, `common/scripting/interface/vmnatives.cpp` SM +140/-0, `scripting/thingdef_data.cpp` SM +0/-43, `scripting/thingdef_properties.cpp` SM +5/-8, `scripting/vmthunks.cpp` SM +15/-2451, `scripting/vmthunks_actors.cpp` SM +4/-713, `scripting/zscript/zcc_compile_doom.cpp` SM +2/-7

**post-processing** (5 files: S 0, M 0, SM 5)  
`common/rendering/hwrenderer/postprocessing/hw_postprocess.cpp` SM +34/-1035, `common/rendering/hwrenderer/postprocessing/hw_postprocess.h` SM +6/-1170, `common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp` SM +3/-161, `common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.h` SM +0/-22, `common/rendering/hwrenderer/postprocessing/hw_postprocessshader.h` SM +1/-2

**rendering other** (5 files: S 0, M 3, SM 2)  
`rendering/gl/stereo3d/gl_openvr.h` M +0/-2, `rendering/r_levelray.cpp` M +0/-314, `rendering/r_levelray.h` M +0/-55, `rendering/r_utility.cpp` SM +60/-215, `rendering/r_utility.h` SM +0/-4

**wadsrc other** (5 files: S 1, M 2, SM 2)  
`wadsrc/static/iwadover.txt` S +16/-0, `wadsrc/static/language.0` SM +4/-0, `wadsrc/static/menudef.txt` SM +19/-493, `wadsrc/static/vrhandcurl.txt` M +0/-87, `wadsrc/static/vrhandpose.txt` M +0/-1682

**common rendering other** (4 files: S 2, M 0, SM 2)  
`common/rendering/i_video.h` S +4/-0, `common/rendering/v_framebuffer.cpp` SM +13/-8, `common/rendering/v_video.cpp` S +10/-0, `common/rendering/v_video.h` SM +1/-7

**QzDoom VR glue** (2 files: S 0, M 2, SM 0)  
`QzDoom/VrCommon.h` M +0/-17, `QzDoom/qzdoom_common.cpp` M +3/-64

## Appendix D — renderer vs netplay, all 93 files (R = render lane, M = main between netplay and 77e5ee3d, RM = both)

**playsim** (15)  
`playsim/actor.h` M +8/-0, `playsim/dthinker.cpp` R +25/-0, `playsim/dthinker.h` R +1/-0, `playsim/p_local.h` M +3/-0, `playsim/p_map.cpp` M +49/-7, `playsim/p_mobj.cpp` M +5/-4, `playsim/p_user.cpp` M +69/-4, `playsim/p_vrdemo.cpp` M +5/-0, `playsim/p_vrdemo.h` M +5/-0, `playsim/p_vrgear.cpp` M +744/-0, `playsim/p_vrgear.h` M +88/-0, `playsim/p_vrreplay.cpp` M +5/-1, `playsim/p_vrsynclog.cpp` M +45/-3, `playsim/p_vrtorso.cpp` M +251/-0, `playsim/p_vrtorso.h` M +65/-0

**scene** (11)  
`rendering/hwrenderer/scene/hw_bsp.cpp` R +35/-7, `rendering/hwrenderer/scene/hw_drawinfo.cpp` RM +43/-0, `rendering/hwrenderer/scene/hw_drawinfo.h` RM +4/-0, `rendering/hwrenderer/scene/hw_drawlist.cpp` R +4/-0, `rendering/hwrenderer/scene/hw_drawstructs.h` R +9/-0, `rendering/hwrenderer/scene/hw_wallcache.cpp` R +427/-0, `rendering/hwrenderer/scene/hw_wallcache.h` R +39/-0, `rendering/hwrenderer/scene/hw_walldispatcher.h` R +18/-2, `rendering/hwrenderer/scene/hw_walls.cpp` R +55/-1, `rendering/hwrenderer/scene/hw_weapon.cpp` M +91/-14, `rendering/hwrenderer/scene/hw_weapon.h` M +5/-0

**top-level src** (10)  
`CMakeLists.txt` RM +13/-0, `d_main.cpp` R +15/-3, `d_net.cpp` RM +155/-4, `d_protocol.cpp` M +11/-2, `d_protocol.h` M +26/-3, `g_game.cpp` RM +63/-44, `p_saveg.cpp` M +21/-0, `p_setup.cpp` R +6/-0, `p_tick.cpp` R +34/-0, `version.h` M +2/-2

**rendering other** (10)  
`rendering/hwrenderer/hw_entrypoint.cpp` R +3/-0, `rendering/hwrenderer/hw_levelresidency.cpp` R +5152/-0, `rendering/hwrenderer/hw_levelresidency.h` R +38/-0, `rendering/hwrenderer/hw_levelstatic.cpp` R +129/-0, `rendering/hwrenderer/hw_levelstatic.h` R +32/-0, `rendering/hwrenderer/hw_models.cpp` M +5/-2, `rendering/hwrenderer/hw_surfacedamage.h` R +11/-0, `rendering/r_snapshot.cpp` R +426/-0, `rendering/r_snapshot.h` R +110/-0, `rendering/r_utility.cpp` M +2/-2

**hwrenderer common data** (9)  
`common/rendering/hwrenderer/data/hw_clock.cpp` R +6/-0, `common/rendering/hwrenderer/data/hw_levelbuffers.cpp` R +218/-0, `common/rendering/hwrenderer/data/hw_levelbuffers.h` R +156/-0, `common/rendering/hwrenderer/data/hw_perftrack.cpp` R +12/-2, `common/rendering/hwrenderer/data/hw_renderstate.h` R +3/-0, `common/rendering/hwrenderer/data/hw_sectorplanebuffer.cpp` R +1/-0, `common/rendering/hwrenderer/data/hw_sectorplanebuffer.h` R +5/-0, `common/rendering/hwrenderer/data/hw_shaderpatcher.cpp` R +5/-0, `common/rendering/hwrenderer/data/shaderuniforms.h` R +5/-1

**r_data (models, VR body/rig)** (9)  
`r_data/models.cpp` M +369/-18, `r_data/models.h` M +1/-1, `r_data/vr_avatar_pose.cpp` M +52/-2, `r_data/vr_body.cpp` M +8/-3, `r_data/vr_bodyinfer_input.cpp` M +35/-6, `r_data/vr_grasp.cpp` M +330/-0, `r_data/vr_grasp.h` M +35/-0, `r_data/vr_rig.cpp` M +142/-4, `r_data/vr_rig.h` M +17/-0

**vulkan backend** (7)  
`common/rendering/vulkan/renderer/vk_descriptorset.cpp` R +19/-3, `common/rendering/vulkan/shaders/vk_shader.cpp` R +21/-3, `common/rendering/vulkan/stereo3d/vk_openxrdevice.cpp` RM +40/-26, `common/rendering/vulkan/stereo3d/vk_openxrdevice.h` M +9/-4, `common/rendering/vulkan/system/vk_buffer.cpp` R +4/-1, `common/rendering/vulkan/system/vk_buffer.h` R +3/-0, `common/rendering/vulkan/system/vk_renderdevice.cpp` R +19/-7

**engine/utility** (4)  
`common/engine/i_net.cpp` M +3/-0, `common/engine/i_upnp.cpp` M +232/-0, `common/engine/i_upnp.h` M +4/-0, `common/utility/rema_profile.h` R +60/-0

**sound** (3)  
`common/audio/sound/s_sound.cpp` R +75/-18, `common/audio/sound/s_soundinternal.h` R +66/-2, `sound/s_doomsound.cpp` R +29/-1

**common rendering other** (3)  
`common/rendering/gl/gl_shader.cpp` R +2/-1, `common/rendering/gles/gles_shader.cpp` R +2/-1, `common/rendering/v_video.h` R +4/-0

**scripting** (3)  
`scripting/vmthunks.cpp` M +101/-0, `scripting/vmthunks_actors.cpp` M +1/-0, `scripting/zscript/zcc_compile_doom.cpp` M +7/-2

**QzDoom VR glue** (2)  
`QzDoom/VrCommon.h` M +17/-0, `QzDoom/qzdoom_common.cpp` M +64/-3

**textures** (2)  
`common/textures/texture.cpp` M +18/-0, `common/textures/textures.h` M +4/-8

**shaders** (2)  
`wadsrc/static/shaders/glsl/levelmesh.vp` R +258/-0, `wadsrc/static/shaders/glsl/main.fp` RM +111/-1

**wadsrc** (2)  
`wadsrc/static/zscript/actors/actor.zs` M +3/-0, `wadsrc/static/zscript/doombase.zs` M +44/-0

**model formats** (1)  
`common/models/modelrenderer.h` M +6/-0

## Cross-check (date 2026-11-13)

Checked against the code in `/home/claude/uzdxrema` (src/wadsrc identical between 175063a1 and the current tip
abb19af1), `/home/claude/uzdx_main`, and read-only `git log/rev-list/merge-base/diff/numstat` against
`origin/main`, `origin/netplay`, `origin/selaco`, 175063a1. Cvar/dead-code greps ran over all of `src/` and `wadsrc/`
(menudef, zscript, txt lumps included); none of the area's cvars or helpers is read from wadsrc.

| category | checked | right | line-fixed | corrected | unconfirmed | added |
|---|---|---|---|---|---|---|
| a. Summary claims | 24 | 19 | 0 | 5 | 0 | 0 |
| b. "What slows it down" items (rank + evidence) | 10 | 10 | 0 | 0 | 0 | 0 |
| c. DEAD / DORMANT / never-read claims | 15 | 14 | 0 | 1 | 0 | 0 |
| d. cvar defaults (both branches where relevant) | 15 | 15 | 2 (read sites) | 1 (r_snapshot 3 semantics) | 0 | 0 |
| e. branch-difference claims | 38 | 31 | 0 | 6 | 0 | 2 |
| f. per-function / call-graph / appendix rows | 46 | 46 | 0 | 0 | 0 | 0 |
| other Part B body claims | 30 | 23 | 1 | 4 | 0 | 2 (one I) |

Confirmed and worth stating: `sim_maxcatchup` (2, `d_net.cpp:207`, used `:2415` outside netgames), `snd_evictretry`
(8, `s_sound.cpp:1341`), `snd_evictbudget` (64, `:1345`) exist only on renderer and are active at `r_newrenderer 0`;
`sim_prefetch` 0; main has none of the four. The PutWall change at `hw_walls.cpp:894` is real and is the only
old-path image change; R_SnapshotAcquire/Release, the three WallCache hash exports, `WallCache_NoBackSector`,
`HWWall::SetTierLight` and `StorageCapacityBytes` have no callers anywhere in src/wadsrc. All commit counts and
merge-bases in A.1 reproduce exactly at 175063a1.

Corrections:
- Summary 1: the 116 renderer commits include 5 merges of main; origin/renderer is now 120 ahead (4 docs-only teardown commits).
- Summary 2: 12 new files, not 13.
- Summary 3 / A.2.3 PutWall: workers also need `gl_multithread` and, in VR, `vr_scene_multithread` (default false); the change makes worker output match the single-thread path instead of creating a difference.
- Summary 4: surface records are per slot surface (unused included) and per static flat plane, not per part; static VB/IB are Static usage, not Persistent.
- Summary 6: one draw per non-empty material group.
- A.1 main/renderer: 46 renderer-only + 29 main-only + 3 both = 78 (49/32 included the 3).
- A.2.1: `hw_walldispatcher.h` is +18/−2, not +20; added the missing `hw_surfacedamage.h` (+11).
- A.2.2: models +156/−10, vr_body +707/−2, vmthunks +196/−40, doombase +128/−26 (the doc gave add+delete sums as "+N").
- B.2/B.9: `r_snapshot 3` = records + parallel measurement, without the `r_snapshot 2` byte images; read lines :373 and :191.
- B.3 classify: `LSR_NEIGHBOUR` (heightsec/3D-floor neighbour at a vertex) is accepted by Build and FoldSide, not refused.
- B.3 shadow: the three verify hashes are computed for every sector at every build, regardless of verify.
- B.3: the 35-frame rebuild limit covers switch changes too; only a new level bypasses it.
- B.8.1: comment line :57 (not :58).
- B.11: fold recomputes the verify hashes for the folded sector only.
- B.12: group count is the ceiling, not the floor, on mesh draw calls.
- Added (B.4): the vertex step loops over all sectors' change bytes whenever any sector changed.
- Added (B.8.1, I): a camera-texture scene sets the frame's light key, so a main view with a different key is refused whole.
