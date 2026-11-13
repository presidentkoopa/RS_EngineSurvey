# 03 — The CPU scene build (world geometry): BSP walk, clipper, walls, flats, portals, draw lists

Conventions.
- Trees: **M** = main `/home/claude/uzdx_main/src` (f182cbcc, the engine of record, primary tree). **R** = renderer
  `/home/claude/uzdxrema/src` (175063a). **N** = netplay `/home/claude/uzdx_netplay/src` (1c1805f8). **S** = selaco
  `/home/claude/uzdx_selaco/src` (7e111a3d). Paths are relative to `src/`. A citation without a tree letter is **M**.
- Short paths: `bsp` = `rendering/hwrenderer/scene/hw_bsp.cpp`, `di` = `rendering/hwrenderer/scene/hw_drawinfo.cpp`,
  `walls` = `rendering/hwrenderer/scene/hw_walls.cpp`, `wv` = `rendering/hwrenderer/scene/hw_walls_vertex.cpp`,
  `flats` = `rendering/hwrenderer/scene/hw_flats.cpp`, `clip` = `rendering/hwrenderer/scene/hw_clipper.cpp/.h`,
  `ff` = `rendering/hwrenderer/scene/hw_fakeflat.cpp`, `rh` = `rendering/hwrenderer/scene/hw_renderhacks.cpp`,
  `portal` = `rendering/hwrenderer/scene/hw_portal.cpp/.h`, `sky` = `rendering/hwrenderer/scene/hw_sky.cpp`,
  `dl` = `rendering/hwrenderer/scene/hw_drawlist.cpp`, `dla` = `rendering/hwrenderer/scene/hw_drawlistadd.cpp`,
  `vb` = `rendering/hwrenderer/hw_vertexbuilder.cpp`, `ep` = `rendering/hwrenderer/hw_entrypoint.cpp`,
  `wd` = `rendering/hwrenderer/scene/hw_walldispatcher.h`, `clock` = `common/rendering/hwrenderer/data/hw_clock.cpp`.
- **[V]** = VERIFIED (I read the code that does it). **[I]** = INFERRED (deduced; the basis is stated). Comments in the
  code were used as leads only. No milliseconds here are mine; numbers quoted from the owner's PC say where they came from.

---

## 1. Summary

1. Every frame, per eye, per portal view and per camera texture, the CPU rebuilds the whole visible world: a front-to-back
   BSP walk with a 1-D angular occlusion clipper (`HWDrawInfo::RenderBSP`, `bsp:1422`) emits every wall part as a fresh
   `HWWall` (`HWWall::Process`, `walls:2419`) and every visible sector plane as an `HWFlat` (`flats:672`) into 8 draw lists.
2. Walls have no persistent GPU geometry on M: their vertices are generated each scene (`HWWall::MakeVertices`, `wv:279`);
   on Vulkan (persistent buffers) this happens at **draw** time on the main thread, together with per-wall dynamic-light
   gathering (`HWWall::DrawWall`, `walls:559-568`) [V]. Flats draw from a static vertex + index buffer built at map load (`CreateVBO`,
   `vb:477`); the vertex z values (shadow copy + mapped buffer, then `Upload`) are patched in place when a reached plane moved
   (`CheckUpdate` `vb:462` -> `UpdatePlaneVertices` `vb:406-421`); the index buffer is never patched [V] [xc: it is the vertex
   buffer, not the index buffer, that is patched]. (M does build a static per-side wall mesh at map load, `DoomLevelMesh`,
   `maploader/maploader.cpp:3251`, `rendering/hwrenderer/doom_levelmesh.cpp:22-28`, but it only feeds the ray tracer
   (`ep:318`, `common/rendering/vulkan/renderer/vk_raytrace.cpp:49-62`, GPU copy only when `RaytracingEnabled`), never the raster draw [xc: added].)
3. VR defaults make it worse: `vr_scene_multithread` is false (`common/rendering/hwrenderer/data/hw_cvars.cpp:278`), so the
   whole walk + wall/flat/sprite setup runs single-threaded on the main thread in VR (`bsp:1438-1439`); `vr_openxr_multiview`
   is false (`common/rendering/hwrenderer/data/hw_vrmodes.cpp:643`), so the scene is built **twice per frame**, once per eye
   (`ep:372`) [V].
4. Draw cost per surface: one `Draw`/`DrawIndexed` per wall part and per section-plane, each preceded by ~15-25 render-state
   setters and one `VkRenderState::Apply` that writes a ~2 KB `StreamData` block (`common/rendering/vulkan/renderer/vk_renderstate.cpp:223-247,379-395`);
   lists are drawn in walk order, texture sorting is off (`gl_sort_textures` false, `di:74,3798`) [V]/[I size].
5. The fork's distance cull (`gl_line_distance_cull` 4000, `bsp:449-462`) is not a cull: a far line is still processed and
   replaced by a full-height wall textured with the front sector's **ceiling flat** (`walls:2563-2580`), and it adds a clip
   range that hides everything behind it (`bsp:653`) [V]. It runs per **seg**, before the per-linedef de-duplication, so a
   linedef split into N segs yields N identical impostors (`bsp:632-655` vs `bsp:716-721`) [V]. It measures from the global
   `r_viewpoint.Pos`, not the scene's viewpoint, so skybox and camera-texture scenes cull by the player's position (`bsp:457`) [V] (bug [I]).
   It tests only the seg's two endpoints (`bsp:457-459`), so a long seg passing close to the viewer is still culled, and the
   impostor path never assigns `lightlevel`/`rellight` before `DoTexture` (`walls:2563-2580`; `HWWall` has no constructor,
   `hw_drawstructs.h:197,200`; fresh stack `HWWall` at `bsp:648` (ST), `bsp:335` (scene worker), `bsp:304` (wall workers)), so impostors draw with an indeterminate light level
   [V code, I effect] [xc: added, B12/B13].
6. Portals (sky, skybox, mirror, line-to-line, stacked sector, plane mirror, horizon) are discovered only by the walk
   (`HWWall::PutPortal`, `walls:906`); every scene-type portal runs another full `CreateScene`/`RenderBSP` from the BSP root
   (`portal.h:187-195`, `di:4479`) [V]. Sky lines become one draw call per line (`portal:252-255`) [V].
7. The walk writes level state that other systems read: `ML_MAPPED`, `SSECMF_DRAWN`, `SECMF_DRAWN` (automap, savegames,
   ZScript-readable), the global `validcount`, vertex clip-angle caches, sector VBO heights; plus dormant dither flags, and
   per scene the billboard `drawPos` that ZScript hit-tests read (`di:3644`, `scripting/vmthunks.cpp:3076`) [V]. Outside the
   walk but in the same per-scene path, `RenderScene` -> `SetupHeatmap` decays `Level->HeatIntensity` (`di:3247-3261`), which
   ZScript reads back through the clearscope native `HeatmapAt` (`scripting/vmthunks.cpp:5618-5629`, `wadsrc/static/zscript/doombase.zs:2098`) [V] [xc: added].
8. Fork per-surface hooks on the draw path: surface damage (`unordered_map` lookup per wall/flat draw when active,
   `rendering/hwrenderer/hw_surfacedamage.cpp:146-160`), effect lights mode, glow seed/planes (every wall and flat), flat-edge
   glow (copies up to 64 sector lines per flat draw, `flats:478-487`), dither (dormant: needs an out-of-bounds camera) [V].
9. The walk is the only producer of: visibility, all wall geometry, the flat/section visible set and moving-plane updates,
   portal lists, thing/particle submission, decals, render-hack fills, Boom heightsec view state (section 7); it is the only
   *view-driven* producer of the automap bits (other writers exist, section 7 item 11) [xc: automap bits are not walk-only].
10. Renderer branch: adds an opt-in static level mesh and wall cache, entered from inside this walk (`R:bsp:346,349,368,661,760,763,1158,1464`,
    `R:di:3830,3835`); both default off (`r_newrenderer` 0, `r_wallcache` 0). The walk itself is unchanged on R, N and S except as listed in section 10 [xc: cross-reference said section 9].

---

## 2. Entry points and call graph

### 2.1 Frame -> view -> eye -> scene

```
D_Display                                   d_main.cpp:1681 FrameTime=I_msTimeFS(); 1697 RenderView(&players[consoleplayer])
 RenderView(player)                          ep:640
  hw_ClearFakeFlat()                         ep:662 -> ff:412   (fake-sector arena freed ONCE per frame)
  ResetProfilingData()                       ep:669 -> clock:80 (all glcycle_t and counters reset once per frame)
  for each changed FCanvas: Draw2D           ep:688-715
  for each Level: canvasTextureInfo.UpdateAll -> RenderViewpoint(texvp,...,mainview=false,toscreen=false)  ep:719-730
  RenderViewpoint(r_viewpoint, camera, ..., mainview=true, toscreen=true)      ep:750
   R_SetupFrame                              ep:268
   (main view only) EmissiveVolumes/BeamLights/EffectLights BeginFrame, shadow map, PrepareFrameCompute   ep:273-346  [other areas]
   vrmode->SetUp(); R_UpdatePoseAnchoredLights                               ep:327-330
   for eye_ix in 0..eyeCount-1                                               ep:359
     renderSceneThisEye = !useMultiviewScene || eye_ix==0                    ep:372
     (multiview, eye 1) post-process only, continue                          ep:386-408
     di = HWDrawInfo::StartDrawInfo(level, nullptr, mainvp, nullptr)         ep:417 -> di:2046 -> StartScene di:2066
     di->SetViewArea()                                                       ep:422 -> di:2952 (in_area for Boom heightsecs)
     vp.Pos = base + eye->GetViewShift()                                     ep:452
     di->SetupView(...) [x2 + ApplyMultiviewViewpoints when multiview]       ep:453-479 -> di:3336, 3378/3394
     di->ProcessScene(toscreen)                                              ep:482 -> di:4597
       ... particle/drawn-line/view-light syncs, heat/shock/smoke/emissive setup   di:4599-4717 [other areas]
       DrawScene(DM_MAINVIEW|DM_OFFSCREEN)                                   di:4719 -> di:4450
         (r_dithertransparency && bDoOob) SetDitherTransFlags                di:4461-4464 (dormant)
         CreateScene(drawpsprites)                                          di:4479 -> di:3665   <== THE CPU SCENE BUILD
         portalState.RenderFirstSkyPortal                                   di:4497 -> portal:123
         RenderScene  (opaque lists)                                        di:4500 -> di:3722
         mesh particles, debris, player sprites, world psprites, SSAO       di:4510-4556 [other areas]
         portalState.EndFrame  (all remaining portals, recursive)           di:4562 -> portal:83
         RenderTranslucent (sorted translucent list, GPU particles, lines)   di:4570 -> di:3855
     di->EndDrawScene / PostProcessScene / EndDrawInfo                       ep:488-512
```

### 2.2 CreateScene: one scene build

```
HWDrawInfo::CreateScene                         di:3665
  FrustumAngle(); mClipper->SafeAddClipRangeRealAngles(yaw+a1, yaw-a1)   di:3668-3669  (back-of-view clip; none when |pitch|>46°, di:3063)
  (bDoOob) vClipper pitch range                  di:3671-3678
  portalState.StartFrame()                       di:3681 -> portal:51
  screen->mVertexData->Map(); screen->mLights->Map()     di:3687-3688
  RenderBSP(Level->HeadNode(), drawpsprites)     di:3690 -> bsp:1422
  DispatchBillboards()                           di:3694 -> di:3546 [billboards; writes bb.drawPos]
  HandleMissingTextures(in_area)                 di:3701 -> rh:602
  HandleHackedSubsectors()                       di:3702 -> rh:1190
  PrepareUnhandledMissingTextures()              di:3703 -> rh:903
  DispatchRenderHacks()                          di:3704 -> rh:52
  mLights->Unmap(); mVertexData->Unmap()         di:3706-3707
```

### 2.3 RenderBSP: the walk

```
HWDrawInfo::RenderBSP(node, drawpsprites)       bsp:1422
  ClearDitherTargets()                           bsp:1424 (20 slots)
  viewx/viewy = FLOAT2FIXED(Viewpoint.Pos)       bsp:1428-1434
  validcount++                                   bsp:1436 (GLOBAL counter shared with the playsim)
  multithread = gl_multithread && (!IsVRScene || vr_scene_multithread)   bsp:1438-1439
  experimentalMultiWallWorkers = multithread && gl_bsp_worker_threads>1  bsp:1440
  [MT]  renderPool.push(WorkerThread())  (1 scene worker)                 bsp:1447-1449
  RenderBSPNode(root)  or RenderOrthoNoFog()     bsp:1471/1527/1539-1540
    RenderBSPNode: R_PointOnSide; recurse front; CheckBox(back bbox) [+CheckBoxOrthoPitch]  bsp:1361-1399
      DoSubsector(sub)                          bsp:1193
        mapsection / POLYORG / SSRF_SEEN->UnclipSubsector / IsBlocked / OoB secret   bsp:1216-1251
        (bDoOob) per-seg angle+pitch+radar visibility   bsp:1254-1325
        (mClipPortal) ClipSubsector -> AddSpecialPortalLines   bsp:1327-1340
        fakesector = hw_FakeFlat(sector, in_area, false)        bsp:1341 -> ff:200
        ProcessVisibleSubsector(sub, sector, fakesector)        bsp:1038
          CheckUpdate(mVertexData, sector)   once per sector per scene   bsp:1040-1043 -> vb:462
          particles: ParticleJob | RenderParticles                       bsp:1046-1058
          AddLines(sub, fakesector)                                       bsp:1060 -> bsp:840
            polys ? AddPolyobjs (BuildPolyBSP if dirty; RenderPolyBSPNode) : for each seg AddLine   bsp:847-868
              AddLine(seg)                                                 bsp:517  (see 2.4)
          once per sector: validcount, SECMF_DRAWN, SpriteJob|RenderThings, dither target scan   bsp:1066-1098
          flats: once per SECTION: FlatJob | HWFlat::ProcessSector; ss_renderflags; AddHackedSubsector;
                 PortalJob|AddSubsectorToPortal for ceiling/floor portal groups   bsp:1101-1180
  [MT] TerminateJob; wait scene future; [wall workers] wait + merge (PutWall/PutPortal/missing tex)   bsp:1474-1523
  dither: P_CheckSight + SetDitherTransFlags for up to 20 targets                 bsp:1547-1553
  mCurrentPortal->RenderAttached(this)                                             bsp:1556
  (drawpsprites) PreparePlayerSprites(...)                                         bsp:1558-1563 [weapon area]
```

### 2.4 AddLine (per seg) -> HWWall::Process (per linedef side)

```
AddLine(seg, portalclip)                         bsp:517
  (portalclip) mClipPortal->ClipSeg              bsp:561-565
  startAngle/endAngle = GetClipAngle(v2/v1)      bsp:569-570 (cached per vertex per scene, clip.h:187)
  (radar, dormant) rClipper ...                  bsp:575-594
  backface: startAngle-endAngle < ANGLE_180 -> return     bsp:597
  miniseg (no sidedef): SafeCheckRange -> SSECMF_DRAWN    bsp:602-616
  !SafeCheckRange -> return                               bsp:618
  SSECMF_DRAWN |= ...                                     bsp:623-624
  IsDistanceCulled(seg)  -> Process(...,isculled=true) [impostor], SafeAddClipRange, return   bsp:632-655
  one-sided / WALLF_BLOCKRENDERING -> SafeAddClipRange    bsp:661-668
  two-sided, other sector: hw_CheckViewArea, hw_FakeFlat(back), hw_CheckClip -> SafeAddClipRange   bsp:688-703
  ML_MAPPED                                               bsp:711-714
  per-linedef dedupe via linedef->validcount              bsp:716-721
  gl_render_walls -> [MT] sceneJobQueue WallJob | [wall workers] wallJobQueue | [ST] HWWall::Process inline   bsp:723-747

HWWall::Process(di, seg, front, back, isculled)  walls:2419
  read ~20 sector/side/line fields; gl_seamless RecalcVertexHeights if dirty     walls:2466-2506
  glow: GetWallGlowFar, GetWallGlow, ApplyWallOwnGlow                              walls:2547-2549
  4x ZatPoint front planes                                                         walls:2551-2554
  Line_Horizon -> SkyNormal + DoHorizon                                            walls:2556-2561
  isculled||IsDistanceCulled -> SkyNormal | DoTexture(TOP, ceiling flat, fch..ffh) walls:2563-2580
  one-sided: SkyNormal; portal -> PutPortal(LINETOLINE) | SkyLine | DoTexture(M1S)  walls:2586-2622
  two-sided: 4x ZatPoint back; SkyTop+SkyBottom; upper DoTexture|AddUpperMissingTexture;
             mid DoMidTexture (+fog boundary); DoFFloorBlocks if any 3D floor; lower DoTexture|AddLowerMissingTexture   walls:2623-2885
DoTexture -> SetWallCoordinates, CheckTexturePosition, PutWall | SplitWall(3D light list) | PutPortal(MIRROR)   walls:1604-1669
PutWall -> (non-persistent only: SetupLights+MakeVertices); decals: [SetupLights]+ProcessDecals; di->AddWall   walls:827-898
AddWall -> drawlists[...].NewWall() (RenderDataAllocator) + full HWWall copy                                   dla:33-58, dl:752
```

### 2.5 Draw side (main thread, per scene)

```
RenderScene                                      di:3722
  SetSweep..., SetupVolumetricBeam, SetupHeatmap di:3748-3793
  (gl_sort_textures) SortWalls/SortFlats         di:3798-3805 (off by default)
  PLAINWALLS.DrawWalls -> HWWall::DrawWall per item     di:3814 -> dl:840 -> walls:559
       [persistent] SetupLights (if Level->HasDynamicLights) + MakeVertices          walls:561-568
       RenderTexturedWall -> SetFog/SetColor/... -> RenderWall -> state.Draw(DT_TriangleFan)   walls:333,202
  PLAINFLATS.DrawFlats -> HWFlat::DrawFlat       di:3815 -> dl:856 -> flats:403
       DrawSubsectors -> [persistent] SetupLights -> state.DrawIndexed(section range)   flats:279-297
  MASKEDWALLS, MASKEDFLATS, MASKEDWALLSOFS, MODELS, Decals[0]      di:3819-3837
RenderTranslucent: TRANSLUCENTBORDER.Draw; TRANSLUCENT.DrawSorted (BSP-style sort, dl:931)  di:3868-3871
```

### 2.6 Portals

- Discovery: `HWWall::PutPortal` (`walls:906`) creates/looks up an `HWPortal` (linear `FindPortal`, `di:3504`) and copies the
  wall into `portal->lines` (`portal.h:127-132`); sky walls via `SkyPlane/SkyLine/SkyNormal/SkyTop/SkyBottom` (`sky:133-456`);
  stacked-sector coverage via `AddSubsectorToPortal` (`di:4729`) [V].
- Rendering: `RenderFirstSkyPortal` (largest sky portal, no stencil, `portal:123-168`) then `EndFrame` pops every portal
  (`portal:83-113`) -> `HWDrawInfo::RenderPortal` (`di:4042`): `SetupStencil` (`portal:281`), new `StartDrawInfo` with
  inherited uniforms (`di:4047`), `DrawContents`; scene portals: `Setup` (new viewpoint, `ClearClipper` silhouette from the
  portal lines, `portal:568-597`) then `DrawScene(DM_PORTAL)` = a full `CreateScene` -> `RenderBSP(root)` (`portal.h:187-195`);
  sky portals draw the dome without a walk (`rendering/hwrenderer/scene/hw_skyportal.cpp:37`) [V].
- Budgets: `gl_max_portals` (-1 = none, `portal:35`, read `di:4044`), `r_portal_recursions` (`portal:659`), skybox
  recursion 3 (`portal:851`) [V].

### 2.7 Threading modes of the walk [V]

| Mode | When | Who runs `HWWall::Process` / `HWFlat::ProcessSector` / sprites |
|---|---|---|
| ST (single thread) | `IsVRScene && !vr_scene_multithread` (**VR default**), or `gl_multithread` 0 | main thread, inline in `AddLine` / `ProcessVisibleSubsector` (`bsp:737-746,1129-1138,1078-1083`) |
| MT (one scene worker) | flat screen default (`gl_multithread` true, `bsp:45`) | one `renderPool` thread consumes `sceneJobQueue` (`bsp:268-388`); main thread walks and enqueues; join at `bsp:1530-1534` |
| MT + wall workers | `gl_bsp_worker_threads` > 1 (default 1, `bsp:46`) | N wall workers process walls into private `HWMeshHelper`s with `di==nullptr` (`bsp:280-311`), merged on the main thread (`bsp:1490-1522`); portal/sky/horizon walls stay on the scene worker (`bsp:464-515`) |

The scene worker busy-waits with 8x `_mm_pause` per empty poll (`bsp:251-265`) [V]. `renderPool` is a static 8-thread
`ctpl::thread_pool` created at startup regardless of mode (`bsp:87`) [V].

---

## 3. Per-function table

Cost classes: per frame (F), per eye (E), per scene (Sc = per eye + per scene-portal + per camera texture), per BSP node (Nd),
per subsector reached (Ss), per seg tested (Sg), per linedef side processed (Ln), per wall part emitted (Wp), per visible
section-plane (Fp), per sector reached (Sec), per draw (Dr), per light candidate (Lt).

| Function | File:line | Called from | Per | Cost class / work | Notes |
|---|---|---|---|---|---|
| `RenderView` | ep:640 | d_main.cpp:1697 | F | O(canvases + camera textures) + main view | resets clocks, fake-flat arena [V] |
| `RenderViewpoint` | ep:264 | ep:750, ep:727, ep:610 | view | eye loop | builds scene per eye unless multiview [V] |
| `HWDrawInfo::StartDrawInfo` / `StartScene` | di:2046 / di:2066 | ep:417, di:4047 | Sc | O(MAX_BEAMS 128 + stamps 64 + fog disturb 32 + shapes 128 + sweep bands) uniform fill; `FillFogDynLights` walks all level lights when the fog slab is on **and** the level asks for fog lights (`Level->FogDynLights` > 0, capped by `fog_dynlights_max` 8; `di:1781-1843,2433`), measuring from the global `r_viewpoint.Pos` (`di:2433`) [xc: the walk also needs FogDynLights > 0, not only the slab]; `ClearBuffers` memsets | ~800 lines of fork uniform setup per scene [V]; `HWViewpointUniforms` ≈ 26.7 KB [I: summed declared fields of `common/rendering/hwrenderer/data/hw_viewpointuniforms.h`] |
| `HWDrawInfo::ClearBuffers` | di:2892 | StartScene di:2849, `FDrawInfoList::Release` di:2035 | Sc ×2 | memset section/subsector/node flag arrays, clear TArrays/TMaps | twice per scene [V] |
| `HWDrawInfo::SetViewArea` | di:2952 | ep:422, portal Setup | Sc | 1-2 `PointInRenderSubsector` | sets `in_area` [V] |
| `HWDrawInfo::SetupView` | di:3336 | ep:455/468/476, portals | Sc | matrix build + copies of ~27 KB uniform structs (`previousLeft`, `previousRight`, `nextRight`) | upload when `upload` true [V] |
| `HWDrawInfo::ApplyViewpoint` | di:3378 | ep:478, portals, flats:542,545 | Sc + translucent flat ×2 | `HWViewpointBuffer::SetViewpoint` memcpy of 1 (or 2 under multiview) uniform blocks | `common/rendering/hwrenderer/data/hw_viewpointbuffer.cpp:109-118` [V] |
| `HWDrawInfo::ProcessScene` | di:4597 | ep:482 | E | particle/defs/atlas/drawn-line/view-light syncs, heat/shock/smoke/emissive setup | not geometry; other areas [V] |
| `HWDrawInfo::DrawScene` | di:4450 | di:4719, portal.h:191 | Sc | orchestrates build + opaque + portals + translucent | [V] |
| `HWDrawInfo::CreateScene` | di:3665 | di:4479/4484 | Sc | frustum clip, map buffers, walk, billboards, render hacks | clocks `ProcessAll`, `VRSceneBuild` [V] |
| `Clipper::Clear` / `SetViewpoint` | clip:67, clip.h:99 | StartScene di:2068-2070,2845-2847; `ClearClipper` | Sc | free node list; `starttime++` invalidates all vertex angle caches | 3 static clippers shared by all scenes (`di:2061-2063`) [V] |
| `FrustumAngle` | di:3052 | CreateScene, ClearClipper | Sc | constant | returns "no clip" when |pitch|>46° (`di:3063`) [V] |
| `HWDrawInfo::RenderBSP` | bsp:1422 | CreateScene di:3690 | Sc | walk + job orchestration + join + merge | clocks `Bsp`, `MTWait`, `WallMerge` [V] |
| `RenderBSPNode` | bsp:1361 | RenderBSP | Nd | `R_PointOnSide` + `CheckBox` (2 `PointToPseudoAngle` + clip-list scan) | recursion depth = BSP depth [V] |
| `RenderOrthoNoFog` | bsp:1402 | RenderBSP | Sc | O(all subsectors) bbox overlap | ortho + no fog-of-war only [V] |
| `Clipper::CheckBox` | clip:515 | RenderBSPNode | Nd | 2 pseudo-angles + `IsRangeVisible` O(clip nodes) | [V] |
| `Clipper::IsRangeVisible` / `SafeCheckRange` | clip:121 / clip.h:106 | CheckBox, AddLine, DoSubsector | Sg, Nd | linear scan of sorted range list | list length grows with occluders [V] |
| `Clipper::AddClipRange` / `SafeAddClipRange` | clip:146 / clip.h:116 | AddLine (occluding segs) | Sg | O(clip nodes), merges | node arena + freelist [V] |
| `Clipper::GetClipAngle` | clip.h:187 | AddLine, UnclipSubsector, DoSubsector | Sg | cached in `vertex_t::angletime/viewangle` keyed on `starttime` | writes level vertices [V] |
| `DoSubsector` | bsp:1193 | RenderBSPNode | Ss | flag tests, `hw_FakeFlat`, optional OoB per-seg pitch test | 6 `VRSubsector*` clock calls per visible subsector in VR, 4 per culled one (`bsp:1195-1196,1342-1346`) [V] [xc: was "10"] |
| `UnclipSubsector` | bsp:420 | DoSubsector | Ss (portal) | per seg `SafeRemoveClipRange` | portals only [V] |
| `hw_FakeFlat` | ff:200 | DoSubsector, AddLine, flats, render hacks, workers | Ss/Sg | returns `sec` unless heightsec / overlapping back sector; else copies a full `sector_t` once per frame per sector into an arena | cache cleared once per frame (`ep:662`), shared by both eyes and all portals [V] |
| `hw_CheckViewArea` | ff:153 | AddLine | Sg (heightsec maps) | 4 ZatPoint | [V] |
| `hw_CheckClip` | ff:38 | AddLine | Sg (two-sided, different sector) | 4 portal flag reads + up to 8 ZatPoint/fD | decides occlusion of two-sided lines [V] |
| `ProcessVisibleSubsector` | bsp:1038 | DoSubsector | Ss | `CheckUpdate` (once/sector), particles, `AddLines`, things (once/sector), flats (once/section), portal groups | [V] |
| `CheckUpdate` / `CheckPlanes` / `UpdatePlaneVertices` | vb:462 / vb:441 / vb:406 | ProcessVisibleSubsector | Sec | compare `GetPlaneTexZ` with `vboheight[pipelinePos]`; on change rewrite z of all of that plane's vertices (shadow copy + mapped buffer) | heightsec and every 3D-floor model too (`vb:465-468`) [V] |
| `AddLines` | bsp:840 | ProcessVisibleSubsector | Ss | loop segs | clocks `ClipWall`, `VRLineBuild` [V] |
| `AddPolyobjs` / `PolySubsector` / `RenderPolyBSPNode` | bsp:817 / 760 / 784 | AddLines | Ss with polyobjects | `BuildPolyBSP` when dirty | [V] |
| `AddLine` | bsp:517 | AddLines, PolySubsector, AddSpecialPortalLines | Sg | backface, range test, distance cull, occlusion add, `ML_MAPPED`, dedupe, dispatch | `ScopedVRLineTimer` 2 clock switches per seg in VR (`bsp:526-552,567,627`) [V] |
| `IsDistanceCulled` | bsp:451 | AddLine, `HWWall::Process` | Sg + Ln | 2 squared distances vs `r_viewpoint.Pos` | evaluated twice per processed wall [V] |
| `NeedsMainThreadWallPath` | bsp:503 | AddLine (wall workers only) | Sg | portal/horizon/sky tests | [V] |
| `HWDrawInfo::WorkerThread` | bsp:268 | renderPool | Sc | job switch: Wall/Flat/Sprite/Particle/Portal | spins when empty [V] |
| `HWWall::Process` | walls:2419 | AddLine, WorkerThread | Ln | ~20 field reads, 4-8 `ZatPoint`, glow lookups, 1-3 tier emissions + sky + 3D floors | wall = whole linedef side (fracleft 0, fracright 1), except polyobjects (`walls:2498-2523`) [V] |
| `HWWall::DoTexture` | walls:1604 | Process | Wp | `GetTexCoordInfo`, `SetWallCoordinates`, `CheckTexturePosition`, `PutWall`/`SplitWall`/mirror `PutPortal` | saves/restores `glseg` [V] |
| `HWWall::DoMidTexture` | walls:1678 | Process | Wp | plane intersection logic, fog boundary part, translucency, `GetAreas` vertical slicing into several walls | each slice = separate draw [V] |
| `HWWall::SetWallCoordinates` | walls:1365 | DoTexture, DoMidTexture, BuildFFBlock | Wp | tex/lightmap UVs, ceiling-below-floor clipping | [V] |
| `HWWall::SplitWall` / `Put3DWall` | walls:1163 / 1049 | DoTexture, DoMidTexture, 3D floors | Wp in sectors with a 3D light list | O(lightlist) per part; on Vulkan uses `lightlist` + per-slice draws (clip planes) | `SplitWallComplex` unreachable on Vulkan (section 6) [V] |
| `DoFFloorBlocks` / `ClipFFloors` / `InverseFloors` / `BuildFFBlock` | walls:2307 / 2233 / 2182 / 2028 | Process | Ln with 3D floors | O(back ffloors × front ffloors) | [V] |
| `HWWall::SkyNormal/SkyTop/SkyBottom/SkyPlane/SkyLine` | sky:262/281/381/133/217 | Process | Ln | `HWSkyInfo::init` (memset + texture lookups, sky:59) per sky part, then `PutPortal` | [V] |
| `HWWall::DoHorizon` | walls:1278 | Process | Ln (Line_Horizon) | `PutPortal(HORIZON)` | [V] |
| `HWWall::PutWall` | walls:827 | DoTexture etc. | Wp | translucency decision, `ViewDistance` for translucent, [non-persistent: lights+verts], decals, `AddWall` | [V] |
| `HWWall::PutPortal` | walls:906 | Process, Sky*, DoHorizon, DoTexture | Wp (portal parts) | `MakeVertices` **at walk time**, `UniqueSkies/Horizons.Get`, linear `FindPortal`, `lines.Push(*this)` | [V] |
| `HWDrawInfo::AddWall` | dla:33 | PutWall via `HWWallDispatcher::AddWall` (wd:89) | Wp | list select + arena alloc + ~340-byte copy | [I size: field sum of `hw_drawstructs.h:178-235`] |
| `HWDrawInfo::AddMirrorSurface` | dla:64 | PutPortal(MIRROR) when `gl_mirror_envmap` | mirror part | extra wall in TRANSLUCENTBORDER + decals | [V] |
| `HWDrawInfo::AddUpper/LowerMissingTexture` | rh:258 / rh:310 | Process | missing-texture parts | linear search of `MissingUpper/LowerTextures` → O(n²) | [V] |
| `HWWall::ProcessDecals` | (hw_decal.cpp) | PutWall walls:886 | wall part with decals | per decal | decals area [V call] |
| `HWFlat::ProcessSector` | flats:672 | ProcessVisibleSubsector / FlatJob / DispatchRenderHacks | per section (first subsector of it) | floor + ceiling + 3D planes; `Process` per plane | [V] |
| `HWFlat::Process` / `PutFlat` / `AddFlat` | flats:589 / 566 / dla:97 | ProcessSector | Fp | `plane.GetFromSector`, resets plane dither flags, skybox verts, arena alloc + copy | [V] |
| `HWDrawInfo::RenderThings` / `RenderParticles` | bsp:914 / bsp:1005 | ProcessVisibleSubsector / jobs | Sec / Ss | per thing `HWSprite::Process` | sprites area [V] |
| `AddSubsectorToPortal` | di:4729 | ProcessVisibleSubsector / PortalJob | Ss with portal plane | linear `FindPortal` | [V] |
| `HandleMissingTextures` | rh:602 | CreateScene | Sc | per missing entry: `validcount++`, flood search (`DoOneSector*`, `DoFakeBridge`), `AddOther*Plane` (lights + vertices) | [V] |
| `HandleHackedSubsectors` | rh:1190 | CreateScene | Sc | per hacked subsector: anchor search + recursive collection | deep-water hacks; skipped for Hexen-format maps (`rh:962`) [V] |
| `PrepareUnhandledMissingTextures` / `PrepareUpper/LowerGap` | rh:903 / 790 / 846 | CreateScene | Sc | flood polys (8 verts each) | unless `Level->notexturefill` [V] |
| `DispatchRenderHacks` | rh:52 | CreateScene | Sc | `ProcessSector` for each hack plane | [V] |
| `HWDrawInfo::DispatchBillboards` | di:3546 | CreateScene | Sc | O(billboards) (+ sort under budget) | writes `bb.drawPos` [V] |
| `HWDrawInfo::RenderScene` | di:3722 | DrawScene | Sc | sweep uniforms, `SetupVolumetricBeam`, `SetupHeatmap`, 6 list draws, decals | clocks `RenderAll`, `VRSceneDraw` [V] |
| `HWDrawList::DrawWalls` / `DrawFlats` / `Draw` / `DoDraw` | dl:840 / 856 / 827 / 788 | RenderScene, RenderTranslucent | Wp / Fp | one `DrawWall`/`DrawFlat` per item | clocks `RenderWall`/`RenderFlat` [V] |
| `HWWall::DrawWall` | walls:559 | DrawWalls/DoDraw | Wp | [persistent] `SetupLights` + `MakeVertices`; then `RenderTexturedWall` / translucent / fog boundary / mirror | the deferred vertex path [V] |
| `HWWall::SetupLights` | walls:600 | DrawWall (persistent), PutWall | Wp × Lt | iterate the side's `wall_dlist` TMap; plane distance; budgeted best-N (`gl_light_wall_candidate_budget` 8); quad overlap test; `UploadLights` | `playsim/a_dynlight.cpp:47-50` [V] |
| `HWWall::MakeVertices` / `CountVertices` / `CreateVertices` / `Split*Edge` | wv:279 / 262 / 187 / 31-179 | DrawWall, PutPortal, AddMirrorSurface | Wp | atomic `AllocVertices` (4 + seamless T-junction splits), write `FFlatVertex` | once per HWWall (`vertcount==0`, wv:281) [V] |
| `HWWall::RenderTexturedWall` | walls:333 | DrawWall, RenderTranslucentWall | Wp | ~15-25 `FRenderState` setters, surface-damage map lookup, effect-light gate, ext colours, light-list slices | [V] |
| `HWWall::RenderWall` | walls:202 | RenderTexturedWall etc. | Dr | dither effect toggle, `SetLightIndex`, `state.Draw(DT_TriangleFan)` | clears dither flags on the side (`walls:225-239`) [V] |
| `HWFlat::DrawFlat` | flats:403 | DrawFlats/DoDraw | Fp | ~18 setters, flat-edge glow copies ≤64 lines, translucent: 2× `ApplyViewpoint` | [V] |
| `HWFlat::DrawSubsectors` | flats:279 | DrawFlat | Fp | [persistent] `SetupLights`; one `DrawIndexed` of the section range | [V] |
| `HWFlat::SetupLights` | flats:157 | DrawSubsectors, PutFlat | Fp × Lt | iterate `flat_dlist[section]` TMap; budget 16 | [V] |
| `HWFlat::DrawOtherPlanes` / `DrawFloodPlanes` | flats:306 / 335 | DrawFlat | hack planes | 1 fan per subsector / 3 strips + 9 stencil/state changes per flood seg | [V] |
| `HWDrawList::DrawSorted` / `Sort` | dl:931 / 702 | RenderTranslucent | Sc | BSP-style split sort of translucent walls/flats/sprites; maps vertex buffer for split vertices | [V] |
| `HWDrawInfo::SetupHeatmap` | di:3233 | RenderScene | Sc | O(HEAT_RES²) decay loop when decay>0 (`HEAT_RES` 256 -> 65,536 cells, `g_levellocals.h:4052`); whenever a cell moved, two fresh 256 KB heap arrays + 2 memcpy + `SetGrid` per scene (`di:3264-3276`) [xc: added] | writes `Level->HeatIntensity`, which ZScript `HeatmapAt` reads (bug B5, section 6) [V] |
| `FPortalSceneState::RenderFirstSkyPortal` / `EndFrame` / `RenderPortal` | portal:123 / 83 / 171 | DrawScene | Sc | per portal: stencil passes + nested scene | [V] |
| `HWPortal::DrawPortalStencil` | portal:212 | Setup/RemoveStencil | portal × pass | **one `Draw` per portal line** (`portal:252-255`) + caps | [V] |
| `HWScenePortalBase::ClearClipper` | portal:568 | portal Setup | portal | O(portal lines) range removal, silhouette | [V] |
| `HWDrawInfo::SetDitherTransFlags` | di:4229 | RenderBSP bsp:1551, DrawScene di:4463 | target | 6 `Trace` calls | needs `r_dithertransparency` + OoB camera [V] |

---

## 4. Work per frame: counts, draws, state changes

Let E = scenes built per displayed frame for the main view: **2 in VR with `vr_openxr_multiview` 0 (default), 1 with it on,
1 flat** (`ep:353-372`) [V]. Add one scene per visible scene-type portal per E (`portal.h:187-195`) and one per updated camera
texture (`ep:719-730`) [V]. Per scene:

| Unit | Work | Evidence |
|---|---|---|
| BSP node reached | `R_PointOnSide` + `CheckBox` (2 pseudo-angles, range scan) | bsp:1373,1382 [V] |
| Subsector reached | ~8 flag/array tests, `hw_FakeFlat`, `ProcessVisibleSubsector` | bsp:1193-1347 [V] |
| Seg tested | 2 cached angles, 1-2 range scans, 2 distance computations, possibly `hw_CheckClip` + 1 range add | bsp:517-749 [V] |
| Linedef side processed | `HWWall::Process`: 0-3 normal tiers + up to 2 sky portal parts + fog boundary + 3D-floor blocks + `GetAreas` slices | walls:2419-2886 [V] |
| Distance-culled seg (> 4000 units) | one `Process(isculled)` **per seg**, one impostor wall or sky portal part per seg | bsp:632-655, walls:2563-2580 [V] |
| Wall part (opaque) | arena alloc + ~340 B copy at walk; at draw: lights (TMap walk), vertex alloc + write (4..4+splits), ~15-25 setters, 1 `Draw` (or 1 per 3D light slice) | dla:33-58, walls:559-592,333-525 [V]/[I size] |
| Section plane | 1 `HWFlat` copy at walk; at draw: lights, ~18 setters, 1 `DrawIndexed` | flats:672-918, 403-556 [V] |
| Translucent flat | + 2 `ApplyViewpoint` (each a uniform-block upload, 2 blocks under multiview) | flats:541-545, hw_viewpointbuffer.cpp:109-118 [V] |
| Sky/portal line | vertices at walk time, 1 `Draw` per line per stencil pass | walls:913, portal:252-255 [V] |
| Every draw | `VkRenderState::Apply`: stream-data write (~2 KB [I: sum of `StreamData` fields in `common/rendering/hwrenderer/data/hw_renderstate.h:213+`]), push constants, material/descriptor apply, flush every `vk_submit_size` applies | vk_renderstate.cpp:223-247,379-395 [V] |

Wall state-setter count per opaque part, read off `RenderTexturedWall` with no light list, no `WALLF_EXTCOLOR`, no glow
(`walls:333-525`, `walls:202-240`, `walls:570`): `SetNormal`, `SetFogDensityScale`, `SetGlowSeedLine`, `SetGlowPlanes`,
[`SetSurfaceDamageKey`×2], [`SetEffectLightMode`×2], `SetMaterial`, `SetFog`, `SetColor`, `SetLightIndex`, **Draw**,
`SetNpotEmulation`, `SetObjectColor`, `SetObjectColor2`, `SetAddColor`, `SetTextureMode`, `SetTextureClamp`, `EnableGlow`,
`EnableGradient`, `ClearFlatGlow`, `ApplyTextureManipulation` ≈ 19 calls + 1 draw; glow adds 5; ext colours add 4-6 [V].
Flat: `SetFogDensityScale`, [damage ×2], [effect ×2], `SetGlowSeedLine`, `SetGlowPlanes`, `SetNormal`, `SetColor`, `SetFog`,
`SetObjectColor`, `SetAddColor`, `ApplyTextureManipulation`, [dither `SetEffect` ×2], `SetFlatGlowParams`+`SetGlowInset` or
`ClearFlatGlow`, `SetMaterial`, [texture matrix], `SetLightIndex`, **DrawIndexed**, 3 resets ≈ 17-20 + 1 draw (`flats:403-556`) [V].
Draw order is the walk's emission order (front to back, roughly) because `gl_sort_textures` is false (`di:74,3798`), so the
material changes on most consecutive walls [I: walk order is spatial, not by texture].

Counters that already exist for these: `rendered_lines`, `rendered_flats`, `vertexcount`, `flatvertices`, `flatprimitives`,
`rendered_portals`, `render_texsplit` (`stat renderstats`, `clock:234-240,275`); `iter_dlight`/`draw_dlight`,
`iter_dlightf`/`draw_dlightf` (`stat lightstats`, `clock:242-254`) [V]. Caveats in section 6.3.

---

## 5. Data structures that matter for speed

| Structure | Where | What matters |
|---|---|---|
| `HWDrawInfo` (pooled by `FDrawInfoList`) | di.h:115-215, di:2001-2038 | 8 `HWDrawList`s, `Portals`, missing-texture arrays, 4 TMaps, per-section/subsector/node flag arrays memset twice per scene (di:2919-2921), 3 copies of `HWViewpointUniforms` (VPUniforms + 2 multiview) ≈ 80 KB [I] |
| `HWViewpointUniforms` | common/rendering/hwrenderer/data/hw_viewpointuniforms.h | ≈ 26.7 KB (beams 5×128 `mBeamA/B/Col/Look/Scroll`, shapes 5×128, stamps 4×64, fog disturb 2×32, fog lights 2×8 FVector4 arrays; `hw_viewpointuniforms.h:196-198,570,645`) [I: field sum] [xc: beams are 5×128, not 6×128]; copied by value in `SetupView` (2-3 copies, di:3340-3372), `StartScene` (di:2083), `RenderViewpoint` (`viewpoints[2]` on the stack, ep:457), and uploaded whole per `SetViewpoint` |
| `Clipper` (3 statics) | clip.h:45-197, di:2061-2063 | doubly linked sorted angle ranges in an `FMemArena` + freelist; all tests are linear scans; the range list is what makes the walk occlusion-culled |
| `vertex_t::angletime/viewangle` (+radar pair) | clip.h:187-195 | per-scene angle cache stored **in level vertices**; invalidated by `Clipper::starttime++` (clip:90) |
| `RenderJobQueue` / `WallWorkQueue` | bsp:134-247 | fixed static arrays: 300,000 `RenderJob` (+ 300,000 `WallWorkItem` + 100,000 batches), **no overflow check** (bsp:140-147,174-181); tens of MB of BSS [I: element sizes ×counts] |
| `HWWall` | hw_drawstructs.h:178-235 | ≈ 340 bytes [I]; built on the stack, copied into the arena by `AddWall`, again into `portal->lines`, `HWMeshHelper` TArrays, `SplitWall*` copies |
| `HWFlat` | hw_drawstructs.h (HWFlat) | copied into the arena by `AddFlat` (dla:131-132) |
| `RenderDataAllocator` | dl:30 | 1 MB-block `FMemArena`, not thread safe; freed when the outermost draw info ends (di:2880-2881). Safe today only because exactly one thread allocates from it at a time [I: MT mode routes every alloc to the scene worker; wall workers write to private arrays] |
| `FFlatVertexBuffer` | common/rendering/hwrenderer/data/flatvertices.h/.cpp | static flat vertices + IBO from `CreateVBO`; dynamic per-frame area from `mIndex` via atomic `AllocVertices` (flatvertices.cpp:139-150; `GetBuffer()` is read before the `fetch_add`, a race if two threads allocate [I]); `mPipelineNbr` buffers and `sector_t::vboheight[pipeline][plane]` (vb:443-450) |
| Fake sectors | ff:178-191 | per-frame arena of full `sector_t` copies for heightsec / overlapping sectors, indexed by `fakesectorbuffer[sectornum]` |
| Light maps | `Level->lightlists.wall_dlist[side]`, `flat_dlist[section]` | `TMap<FDynamicLight*, unique_ptr<FLightNode>>` iterated per surface per draw (walls:627-646, flats:180) |
| Section/subsector flag arrays | di.h:207-209 | `section_renderflags` (flats once per section), `ss_renderflags` (PROCESSED for render hacks; SEEN for portals), `no_renderflags` (node coverage for stacked portals) |
| Draw lists | dl:42-78, hw_drawlist.h:81-84 | `walls/flats/sprites` pointer arrays + `drawitems`; `SortNodes` static array for the translucent sort |

---

## 6. What slows it down (ranked)

Ranking is by expected share of CPU frame time in VR on the default settings; confidence given per item. Previously measured
numbers (flat screen, renderer branch builds, owner's PC): `HANDOFF_RENDER_2026-11-11.md` on R reports for the **old path**
(build `doomxr_rend1112q.exe`, Sunder MAP15, `r_dynlights 0`) "bsp 0.56, walls 2.47, draw calls 0.74 ms", and for
`render_vantage13_300.cfg` (build `1112f`, MAP13) "5,300-6,000 walls, scene build 5-11 ms (walls 3.4-5.6, sprites 0.8-3.5,
clip 1-2, flats ~1, draw calls 1-2.2)". It also reports that the wall cache (skipping `Process`) was **not** faster: "the
cost of a wall is the draw-list hand-off, not `Process`". These were flat-screen MT runs; in VR the same work is serial and doubled.

1. **Two full scene builds per VR frame** (high confidence, structural). Multiview is off by default
   (`hw_vrmodes.cpp:643`) so `renderSceneThisEye` is true for both eyes (`ep:372`) and each eye runs `StartScene` +
   `CreateScene` + walk + draw-time vertex/light work + all draws. The second eye's walk differs only by a ~64 mm shift.
   With multiview on, the walk runs once from the left eye's position (`ep:453-472`; clipper reads `di->Viewpoint`,
   `clip.h:99`, which is the left eye), so geometry visible only to the right eye near the frustum edge can be clipped [I].
2. **Serial walk in VR** (high). `vr_scene_multithread` false (`hw_cvars.cpp:278`) → `multithread` false (`bsp:1439`):
   clipping, `Process`, flats, things and particles all on the main thread, in sequence, with per-subsector and per-seg VR
   clock calls (`bsp:1195-1346`, `bsp:567,627`). Flat screen gets one worker for the producers, but the walk itself
   (`RenderBSPNode`/`AddLine`/clipper) is single-threaded in every mode [V].
3. **Per-part draw submission on the main thread** (high). Every wall part and section plane is its own draw with ~20
   state setters and an `Apply` that writes a ~2 KB stream block; on Vulkan the wall vertices and the dynamic-light lists are
   built at draw time too (`walls:561-568`, `flats:287-290`). The R measurement above ("cost of a wall is the draw-list
   hand-off") points here. Texture sort off → material changes per draw [V code / I share].
4. **Dynamic lights per surface per scene** (medium-high, map/mod dependent). Every drawn wall and flat walks its light TMap,
   recomputes plane distances, partial-sorts candidates and uploads a light block (`walls:600-806`, `flats:157-271`), per eye.
   Budgets cap the lights kept, not the candidates examined [V].
5. **Portals re-walk from the root** (medium, map dependent). Each mirror/line portal/skybox/stacked sector runs a full
   `CreateScene` (clipped by a silhouette), per eye; sky portal lines cost one draw per line per pass (`portal:252-255`) [V].
6. **Uniform-block copies and uploads** (medium) [I size]. ~27 KB `HWViewpointUniforms` copied 2-3× per `SetupView`
   (`di:3340-3372`), uploaded per `ApplyViewpoint`; each translucent flat uploads twice (`flats:541-545`); under multiview
   `SetViewpoint` builds `{*vp, *vp}` (54 KB) for a single view (`hw_viewpointbuffer.cpp:112-115`) [V code].
7. **`StartScene` fork uniform work per scene** (low-medium). ~800 lines, fixed-size loops (128 beams, 64 stamps, 128 shapes,
   32 fog disturbs) and, with the fog slab and a level that asks for fog lights (`Level->FogDynLights` > 0), a walk over all
   level lights (`di:2433`, `di:1781-1843`) — per eye and per portal [V] [xc: fog-light walk also needs FogDynLights > 0].
8. **Distance-cull impostors** (low CPU, visible artefacts). N identical impostors per N-seg linedef; impostors drawn with
   ceiling textures and an uninitialised light level (B12); endpoint-only test (B13); not counted in `rendered_lines` in ST
   mode (section 6.3) [V] [xc: B12/B13 added].
9. **Render hacks** (low, map dependent): O(n²) missing-texture bookkeeping (`rh:278-294,345-361,629-638`), recursive
   collections, each with `validcount++` [V].
10. **Heightsec maps**: whole-`sector_t` copies (`ff:243`) per affected sector per frame (cached across eyes) [V].

### 6.1 Bugs and hazards found in this path

| # | Finding | Evidence | Conf. |
|---|---|---|---|
| B1 | Distance cull uses the global `r_viewpoint.Pos`, not the scene's viewpoint. Skybox scenes (viewpoint = skybox actor, `portal:866`) and camera textures (`texvp`, `ep:725-727`) cull against the player's position; a skybox room more than 4000 units from the player has its lines replaced by impostors and its view occluded. Camera textures are rendered before the main view's `R_SetupFrame(r_viewpoint, ...)` (`ep:719-730` before `ep:750` -> `ep:268`), so they cull against the previous frame's main-view position [xc: refined]. | bsp:457-458 | [V] code, [I] visible effect |
| B2 | Distance-culled segs bypass the linedef de-duplication: one impostor per seg, each spanning the whole linedef (Process uses linedef v1/v2 with fracleft 0/fracright 1, walls:2487-2501). On a linedef with some near and some far segs, the near seg emits the normal full-linedef wall and each far seg emits a coplanar full-linedef opaque impostor over it (z-fight / ceiling texture showing through) [xc: added, V code / I effect]. | bsp:632-655 before bsp:716-721 | [V] |
| B3 | Impostors are textured with the front sector's ceiling flat (or become sky portal parts) and are opaque full-height walls; they also add a clip range. Culled lines do not get `ML_MAPPED`. | walls:2563-2580, bsp:653, bsp:711 | [V] |
| B4 | `gl_render_walls 0` does not stop impostors (checked only after the cull branch). | bsp:632 vs bsp:723 | [V] |
| B5 | `SetupHeatmap` decays `Level->HeatIntensity` by `HeatDecay * screen->FrameTime * 0.001` where `FrameTime` is milliseconds since the first frame (`d_main.cpp:1681`, `common/utility/i_time.cpp:193-196`), not the frame delta; and it runs per scene (per eye, per portal). Heat empties almost at once once uptime is large, faster with more scenes. The decayed array is the one ZScript reads back through the clearscope native `HeatmapAt` (`scripting/vmthunks.cpp:5618-5629`, `wadsrc/static/zscript/doombase.zs:2098`; written by `HeatmapAdd`, `vmthunks.cpp:5473-5514`), so script sees a value that depends on uptime, eye count, portals and frame rate: a render→script path, view- and machine-dependent (netplay hazard) [xc: added; section 8 said "read only by the renderer"]. | di:3247-3261 | [V] code, [I] effect |
| B6 | Billboard `drawPos` is written per scene, including portal scenes (no `mCurrentPortal` gate) and the second eye; ZScript hit-tests read it (`scripting/vmthunks.cpp:3076`), but only for view-locked and hand-locked billboards with a non-zero `drawPos` (`BBFL_VIEWLOCKED|BBFL_HANDANY`); other billboards use `pos` [xc: scope narrowed]. The value seen by script is whichever scene ran last, and it is client-view dependent (a render→script path). | di:3644 | [V] code, [I] consequence |
| B7 | `HWFlat::Process` writes `dithertransflag = false` into the model sector's planes (the real sector, or its per-frame fake copy for heightsec/overlap sectors) from the scene worker (MT) [xc: not always the real sector], concurrently with nothing today but against the "renderer may only draw" rule. | flats:592-593 | [V] |
| B8 | Experimental wall workers: `vertex_t::RecalcVertexHeights` (walls:2504-2505) and lazily computed texture data (`GetAreas`, walls:1951) can run on two workers at once for walls sharing a vertex/texture. | bsp:280-311 | [I] |
| B9 | `RenderJobQueue`/`WallWorkQueue` have no bounds check (comment says 300,000 is enough). | bsp:140-147,174-181 | [V] |
| B10 | `AllocVertices` reads `GetBuffer()` before `fetch_add` (pointer from a stale index if two threads allocate). Single allocating thread today on Vulkan (walls/sprites allocate at draw time) except skybox flats and portal/sky wall parts (`PutPortal` -> `MakeVertices` at walk time, `walls:913`) on the scene worker [xc: portal walls added]. | common/rendering/hwrenderer/data/flatvertices.cpp:141-142 | [I] |
| B11 | Dither: `RenderedTargets` scan reads `thing->validcount` on the main thread while the scene worker's `RenderThings` writes it (MT). Dormant (needs OoB camera). In ST (the VR default) it is also broken: `RenderThings` runs first (`bsp:1078-1083`) and stamps `validcount` on every thing in `touching_renderthings` before any cull (`bsp:923-924`), so the scan's `thing->validcount == validcount` skip (`bsp:1091`) drops every thing and no dither target is ever collected [xc: added]. | bsp:1088-1097, bsp:923-924 | [V] |
| B12 | Distance-cull impostors draw with an indeterminate light level: the culled branch of `Process` calls `DoTexture` without ever assigning `lightlevel`/`rellight` (assigned only in the later tier branches, `walls:2607,2682,2756,2807,2824`); `HWWall` has no constructor (`hw_drawstructs.h:143-235`) and every caller builds a fresh stack `HWWall` (`bsp:648` ST, `bsp:335` scene worker, `bsp:304` wall workers); `PutWall` reads it for the glow test (`walls:854`) and `RenderTexturedWall` feeds it to `SetFog`/`SetColor` (`walls:336,469-470`). [xc: added] | walls:2563-2580 | [V] code, [I] visible effect |
| B13 | `IsDistanceCulled` compares only the two endpoints against 4000 (`bsp:457-459`); a long seg that passes close to the viewer with both ends far away is culled, replaced by an impostor, and `SafeAddClipRange` (`bsp:653`) then hides everything behind it, including near geometry. [xc: added] | bsp:451-462, 632-655 | [V] code, [I] how often maps have such segs |

### 6.2 What the clocks cover (for reading a measured run)

All `glcycle_t` tick only while `glcycle_t::active` (`common/engine/stats.h:260-273`), set by `stat rendertimes`,
`printstats` or `r_perflog > 0` (`clock:438-445`). Reset once per frame (`ep:669`), so they sum both eyes, all portals
and camera textures [V].

| Clock | Clocked at | Covers | Does not cover / caveat |
|---|---|---|---|
| `ProcessAll` | di:3684-3709 | whole `CreateScene` incl. walk, join, billboards, render hacks | draw side |
| `Bsp` | bsp:1425-1541 | walk; in MT stops before the worker join (`bsp:1482/1531`) | `MTWait` |
| `ClipWall` | bsp:846-869, 891-904 | `AddLines` = clipping **plus**, in ST mode (VR default), inline `HWWall::Process` | so "Clip" includes wall setup in VR |
| `SetupWall` | bsp:741-745 (ST), bsp:337-343 (worker) | `Process` (nested inside `ClipWall` in ST); on worker thread in MT | distance-culled ST path not clocked (bsp:648-652) |
| `SetupFlat` / `SetupSprite` | bsp:1134-1136, 353-357 / bsp:1008-1034, 1054-1083, 364-377 | `ProcessSector`; things/particles | nested inside `Bsp` |
| `MTWait`, `SceneWorkerElapsed`, `WTTotal`, `WTWallJobs/FlatJobs/ThingJobs` | bsp:1483-1534, 277-329 | MT join wait; worker lifetime; per-job type | MT only |
| `RenderWall` / `RenderFlat` | dl:795-807, 843-863 | draw-list traversal **including** Vulkan draw-time `SetupLights` + `MakeVertices` + setters + `Draw` | not the GPU |
| `RenderAll` | di:3726-3839, 3858-4031 | `RenderScene` + `RenderTranslucent` | portals' own scenes are clocked inside their nested calls |
| `drawcalls` | vk_renderstate.cpp:225-247 | `VkRenderState::Apply` only | command recording of the draw itself |
| `VRSceneBuild`, `VRScenePostBSP`, `VRSubsectors`, `VRSubsectorCull/Visible`, `VRLineBuild/Clip/Decide`, `VRThingBuild`, `VRFlatBuild`, `VRPlayerSprites`, `VRSceneDraw`, `VRSceneEyes` | di:3683-3710, bsp:845-870, 1195-1346, 526-627, ep:370-515 | VR scenes only (`IsVRScene`) | `GetRenderTimeSummary` adds `VRScenePostBSP` and `VRPlayerSprites` to `VRSceneBuild` although both are nested inside it (di:3700 within 3683-3710; bsp:1560 within di:3690) → the "Scene" bucket double-counts them (`clock:157`) |
| perf_track `scene` | M: hw_perftrack.cpp:526 from `GetRenderTimeSummary` | VR clocks only → **0 in flat-screen runs on M** | R falls back to `ProcessAll+RenderAll` (R:clock:163-168) and adds `cpu_bsp/clip/walls/flats/sprites` columns (R:hw_perftrack.cpp:539-543) |

### 6.3 Counter caveats

- `rendered_lines` counts `Process` calls: in ST the distance-culled path is not counted (bsp:648-652), in MT it is
  (bsp:342), wall workers add `mesh.wallCount` (bsp:1500); render-hack minisegs add more (rh:1185) [V].
- `render_vertexsplit` is never incremented anywhere (only reset/printed) → always 0 [V: grep finds no increment].
- `WallWorkersElapsed` is never clocked (`Reset` at clock:124, read at clock:174 as a fallback that is always 0); elapsed is taken from `WallWorkersElapsedCycles` [V] [xc: it is read, not only reset].
- The per-wall `rdtsc()` in the wall workers runs whether or not clocks are active (bsp:305,311,402-404) [V].

---

## 7. Why the walk is needed today (everything only it produces)

This is the list of outputs whose **only producer on M** is the per-scene walk (`RenderBSP` and what it calls). Any
replacement must produce each one or drop it knowingly. [V] unless marked.

| # | Output | Produced by | Consumers |
|---|---|---|---|
| 1 | Visibility: which subsectors, segs, sectors, sections are drawn (front-to-back angular occlusion via `Clipper`) | bsp:1361-1399, 517-749 | everything below |
| 2 | **All wall geometry** (positions, UVs, lightmap UVs, T-junction splits, 3D-floor blocks and light-list slices, mid-texture `GetAreas` slices, fog boundaries, mirror envmap overlays). There is no persistent wall VBO for raster drawing on M (the map-load `DoomLevelMesh` of every side, `maploader/maploader.cpp:3251`, `doom_levelmesh.cpp:22-28`, feeds only the ray tracer, `ep:318`) [xc: DoomLevelMesh note added]. | walls:2419-2886, wv:187-288 | opaque/masked/translucent lists |
| 3 | Per-part light level, colormap, glow, translucency and list choice | walls:2533-2549, 827-898, dla:33-58 | draw |
| 4 | The visible flat set (per section, per plane, 3D planes), with per-plane light/colormap from 3D light lists | flats:672-918 | draw |
| 5 | Moving-plane updates of the static flat VBO, only for sectors reached (`CheckUpdate`) | bsp:1040-1043, vb:462 | flat draws (stale until next seen) |
| 6 | Every portal and its lines: sky (incl. sky walls on upper/lower edges), skybox, horizon, mirror, line-to-line, stacked sector (+ coverage), plane mirror | walls:906-1041, sky:133-456, di:4729 | `RenderFirstSkyPortal`, `EndFrame` |
| 7 | Sprite/model/particle submission (things only for sectors reached; particles per subsector; portal-attached things via `ProcessActorsInPortal`) | bsp:914-1036, walls:997 | sprite lists [area of sprites] |
| 8 | Decals (only on processed walls) | walls:875-887 | `DrawDecals` |
| 9 | Render-hack inputs: missing upper/lower textures, hacked (deep water) subsectors, flood gaps, lower minisegs | walls:2732,2880, bsp:1143, rh:* | `DispatchRenderHacks` flats |
| 10 | Boom heightsec view state: `in_area` refined per seg (`hw_CheckViewArea`) and fake sectors | bsp:586,691, ff:200 | walls/flats/sprites of heightsec sectors |
| 11 | Automap and save data: `ML_MAPPED`, `SSECMF_DRAWN`, `SECMF_DRAWN`. The walk is the only **view-driven** producer on the Vulkan engine, not the only producer: `ML_MAPPED` is also set by `Line_SetAutomapFlags` (`playsim/p_lnspec.cpp:2771-2778`), the UDMF `mapped` key (`maploader/udmf.cpp:960`) and ZScript writes to `Line.flags`; `SSECMF_DRAWN` is also restored on savegame load (`p_saveg.cpp:457,539`); the software renderer sets all three (`swrenderer/line/r_line.cpp:99-169`, `swrenderer/scene/r_opaque_pass.cpp:506`) [xc: "only producer" was too strong] | bsp:713,608-624,1070 | `am_map.cpp:2093,2230,2684,3323`; `p_saveg.cpp:444-539`; ZScript constants in repo-root `wadsrc/static/zscript/mapdata.zs:215,469` |
| 12 | Distance-cull impostors and their occlusion | bsp:632-655 | draw |
| 13 | Dither targets and dither flags (dormant). Targets: walk only (and never collected in ST, B11). Flags: also set outside the walk, by `DrawScene` for the camera's tracer or the player before `CreateScene` (`di:4461-4464`) [xc: flags are not walk-only] | bsp:1085-1098,1547-1553 | `RenderWall`, `DrawFlat` |
| 14 | Ordering hooks: `PreparePlayerSprites` and `RenderAttached` run at the end of `RenderBSP` | bsp:1556-1563 | weapon/portal areas |

Note: R's own stage-3 plan (`docs/plans/STAGE3_VISIBILITY_PLAN.md`, read only as a lead; summarised in
`R:HANDOFF_RENDER_2026-11-11.md:32`) reaches the same conclusion — the static mesh does not remove the walk because of items
1, 5, 6, 7, 9, 11.

---

## 8. What the walk writes into level / sim-visible state

| Write | Where | Readers / risk |
|---|---|---|
| `line_t::flags |= ML_MAPPED` | bsp:713 | automap, savegames, ZScript `Line.flags`; per-client (view dependent) [V] |
| `subsector_t::flags |= SSECMF_DRAWN` | bsp:608,612,624 | automap, savegames (p_saveg.cpp:444-539) [V] |
| `sector_t::MoreFlags |= SECMF_DRAWN` | bsp:1070 | constant exposed to ZScript (repo root `wadsrc/static/zscript/mapdata.zs:469`); no C++ reader found [V grep] |
| global `validcount` (++ per RenderBSP and in render hacks); `sector/line/subsector/AActor::validcount` | bsp:1436, 681, 720, 924, 1069; rh:608-917,1195-1219 | shared with the playsim's sight/iterator code; equality-only use makes it harmless between tics [I] |
| `vertex_t::angletime/viewangle/radar*` | clip.h:187-195 | render-only [V grep: outside `hw_clipper.*` the names appear only in their declaration/initialisation, `gamedata/r_defs.h:99-102,158-161`] [xc: grep done, upgraded from I] |
| `vertex_t` height lists (`RecalcVertexHeights` when dirty, `gl_seamless`) | walls:2504-2505 | render-only (only caller) [V grep] |
| `subsector_t::BSP` (polyobject BSP rebuild when dirty) | bsp:819-822 | render-only (hw + sw renderers) [V grep] |
| `sector_t::vboheight[pipeline][plane]` | vb:446-451 | render-only [V] |
| `side_t::Flags` WALLF_DITHERTRANS_*, `dithertranscount`; `secplane_t::dithertransflag` | di:4154-4163,4173-4213; walls:225-239; flats:592-593 | dormant (needs `r_dithertransparency` + `VPSF_ALLOWOUTOFBOUNDS` camera, `rendering/r_utility.cpp:160,912-915`) [V] |
| `FBillboard::drawPos` | di:3644 | read by ZScript hit-tests (`scripting/vmthunks.cpp:3076`) — render→script leak (B6) [V] |
| `Level->HeatIntensity` decay (not the walk: `RenderScene` -> `SetupHeatmap`, per scene) | di:3247-3261 | written by ZScript natives (`scripting/vmthunks.cpp:5460-5538`) and read back by ZScript through the clearscope native `HeatmapAt` (`vmthunks.cpp:5618-5629`, `doombase.zs:2098`) — render→script leak (B5); not serialized [V grep] [xc: was "read only by the renderer"] |
| `light->...` via `gl_GetLightPosRelative` caches keyed on `gl_dynlight_viewid` | walls:672, flats:212 | render-only [I] |

---

## 9. Hooks, cvars and dead code in this area

| Item | Definition | Default | Read at | Status | Evidence |
|---|---|---|---|---|---|
| `gl_multithread` | bsp:45 | true | bsp:1439 | ACTIVE flat screen; inert in VR unless `vr_scene_multithread` | [V] |
| `vr_scene_multithread` | hw_cvars.cpp:278 | false | bsp:1438 | DORMANT (VR serial by default) | [V] |
| `gl_bsp_worker_threads` | bsp:46-52 | 1 | bsp:395,1440,1452 | DORMANT. Message says "takes effect after restart" but it is read every scene [V]; pool is fixed at 8 threads (bsp:87), so 8 workers + scene worker serialise one job [I] | [V]/[I] |
| `gl_bsp_worker_sky_mainthread` | bsp:53 | false | bsp:469 | DORMANT (wall workers only) | [V] |
| `gl_bsp_wall_batch_size` | bsp:54 | 96 | bsp:56-61 | DORMANT (wall workers only) | [V] |
| `gl_line_distance_cull` | bsp:449 | 4000 | bsp:453 (via `IsDistanceCulled`, called from bsp:632 and walls:2563); menu slider `menudef.txt:3215`; reset to default on configs older than engine version 223 (`gameconfigfile.cpp:699-700`) | ACTIVE (bugs B1-B4, B12, B13) | [V] [xc: B12/B13 added] |
| `gl_sprite_decor_distance_cull` | hw_sprites.cpp:1042 | 1200 | bsp:72-79 | ACTIVE (things) | [V] |
| `gl_render_walls/flats/things` | bsp:416-418 | true | bsp:723,1101,1046,1072 | debug, ACTIVE gate; walls gate misses impostors (B4) | [V] |
| `gl_render_segs` | — | — | `EXTERN_CVAR` at bsp:413 only | DEAD declaration (no definition, no read) | [V grep] |
| `gl_seamless` | hw_cvars.cpp:106 | true | walls:2502, wv:283 | ACTIVE (T-junction vertex splits) | [V] |
| `gl_sort_textures` | di:74 | false | di:3798 | DORMANT | [V] |
| `gl_max_vertices` | flats:53 | 0 | walls:218, flats:282 | DORMANT budget | [V] |
| `gl_max_portals` | portal:35 | -1 | di:4044 | DORMANT budget | [V] |
| `gl_light_wall/flat_max_lights` | playsim/a_dynlight.cpp:47-48 | 1000 | walls:653, flats:177, rh:125 | ACTIVE (per eye, draw order dependent) | [V] |
| `gl_light_wall/flat_candidate_budget` | a_dynlight.cpp:49-50 | 8 / 16 | walls:654, flats:178, rh:126 | ACTIVE | [V] |
| `r_radarclipper` | rendering/r_utility.cpp:159 | false (CHEAT) | bsp:559,1244,1430 | DORMANT (+ needs OoB camera) | [V] |
| `r_dithertransparency` | r_utility.cpp:160 | false (CHEAT) | bsp:1085, di:4461 | DORMANT (+ needs OoB camera) | [V] |
| `gl_mirror_envmap` / `gl_mirrors` | hw_cvars.cpp:105/103 | true/true | walls:981, 1652 | ACTIVE on mirror lines | [V] |
| `gl_noskyboxes` | sky:34 | false | sky:141,226 | DORMANT; not set directly but driven by the menu-exposed `gl_skydome` callback (`gl_noskyboxes = !gl_skydome`, sky:35-38; `menudef.txt:3199`), so `gl_skydome 0` turns skyboxes off [xc: added driver] | [V] |
| `gl_no_skyclear` | di:75 | false | di:4497 | DORMANT | [V] |
| `topskew/midskew/bottomskew` | walls:2410-2412 | 0 | walls:2613,2689,2761,2830 | debug defaults | [V] |
| `hw_npottest` + NPOT code | walls:329-331,392-407 | — | — | DEAD (no `NPOT_EMULATION` define found in src or CMake) | [V grep] |
| `gl_coronas` | di:82 | true | only inside a commented block (di:4356) | DEAD cvar | [V grep] |
| `DrawCorona` body / `DrawCoronas` loop | di:4060-4125, 4300-4327 | — | `#if 0`; `DrawCoronas` call commented out | DEAD | [V] |
| `bool inited` | bsp:88 | — | never read | DEAD | [V grep] |
| `WallWorkersElapsed` clock | clock.h:42 | — | reset (clock:124) and read (clock:174) but never clocked → always 0 | DEAD clock | [V grep] [xc: it is read, not only reset] |
| `render_vertexsplit` | clock:70 | — | never incremented | DEAD counter | [V grep] |
| `HWWall::SplitWallComplex` | walls:1069 | — | only when `RFL_NO_CLIP_PLANES` (walls:1213); Vulkan sets `RFL_SHADER_STORAGE_BUFFER|RFL_BUFFER_STORAGE` only (`common/rendering/vulkan/system/vk_renderdevice.cpp:510`) | DEAD on the Vulkan-only engine (reachable only on legacy GL) | [V] |
| Non-persistent branches (`PutWall` lights/verts walls:859-866, `PutFlat` lights flats:572-578) | — | — | Vulkan is persistent | DEAD on Vulkan | [V] |
| `DoHorizon` mesh-builder branch | walls:1350-1354 | — | horizon lines never go to wall workers (bsp:489-490) | DEAD on M; may be reachable from R's static build (area 09) | [V]/[I] |
| `EnableClipDistance` | vk_renderstate.cpp:159-161 | — | called by dl:942-946 | no-op on Vulkan | [V] |
| Surface damage hook | walls:378-380, flats:409-410 | — | `SurfaceDamage::WallKey/FlatKey` hash lookup when `mDrawable` | ACTIVE when damage exists | [V] |
| Effect-light mode hook | walls:386-389, flats:416-418 | — | when `EffectLightBuffer` has live lights | ACTIVE conditionally | [V] |
| Glow seed / planes / inset | walls:149-174,351-373, flats:439,490 | — | every wall and flat draw | ACTIVE | [V] |
| Flat-edge glow (≤64 lines per draw) | flats:454-496 | — | planes with `FlatGlowColor` | ACTIVE conditionally | [V] |
| Heat grid | di:3233-3314 | — | `Level->HeatScale > 0` | ACTIVE conditionally (B5) | [V] |
| `gl_portalinfo` CCMD | portal:69 | — | EndFrame | diagnostic | [V] |

---

## 10. Branch differences

| Branch | Differences in this area (against M) | Evidence |
|---|---|---|
| **netplay (N)** | Scene path identical except `DrawScene` lacks the `[WORLDPSPR]` `DrawWorldPlayerSprites` call (`git diff origin/main origin/netplay -- .../hw_drawinfo.cpp`); `hw_bsp/walls/flats/clipper/fakeflat/renderhacks/portal/sky/vertexbuilder/entrypoint`, `hw_clock`, `flatvertices`, `hw_cvars` unchanged | [V] |
| **renderer (R)** | (a) `RenderBSP` asks `LevelResidency_BeginScene` (R:bsp:1464); wall jobs/inline walls call `LevelResidency_SkipWall` and `WallCache_Process` before `Process` (R:bsp:346,349,661,760,763); flat jobs/inline call `LevelResidency_FlatPlanes` (R:bsp:368,1158); `RenderScene` draws `LevelResidency_Draw/DrawFlats` before the plain lists (R:di:3830,3835). Defaults off: `r_newrenderer` 0 (R:rendering/hwrenderer/hw_levelresidency.cpp:137), `r_wallcache` 0 (R:rendering/hwrenderer/scene/hw_wallcache.cpp:40). (b) wall workers disabled while the level mesh is on (R:bsp:650, distance-culled segs, and R:bsp:743) and also while `r_wallcache` > 0 (R:bsp:743 only, normal segs) [xc: R:bsp:650 tests only `mLevelMeshScene`]; with the level mesh on, a culled seg whose side the mesh draws skips the impostor but still adds its clip range (R:bsp:661-667) [xc: added]. (c) `Process` skips the distance-cull impostor in the static build (`HWMeshHelper::noDistanceCull`, R:walls:2616-2617). (d) `PutWall` clears `HWF_GLOW` at white light 255 without a draw info (R diff in walls:886-897). (e) new `HWWall::SetSectorLight/SetSideGlow/SetTierLight/SetTierLightLevel` (R:hw_drawstructs.h; R:walls:2431-2461) used only by the residency light patch. (f) `HWMeshHelper` records emission order (R:wd). (g) `P_FreeLevelData` resets wall cache and residency (R:p_setup.cpp:641-643). (h) Tracy zones `REMA_ZONE_N` in RenderViewpoint/eye/RenderBSP/CreateScene/RenderScene/RenderTranslucent/RenderPortal/DrawScene/ProcessScene/DrawWalls/DrawFlats/SortTranslucent (no-ops when off). (i) `opaque.walls/flats/masked/models/decals` GPU groups (R:di:3828-3866). (j) perf_track `scene` falls back to `ProcessAll+RenderAll` in flat runs and adds `cpu_bsp/clip/walls/flats/sprites` (R:clock:163-168, R:hw_perftrack.cpp:539-543). The walk itself is otherwise the same. Area 09 covers the static path. | [V] |
| **selaco (S)** | Adds a `ParticlePoolJob` and `RenderDefinedParticles` per subsector (S:bsp, diff `+ParticlePoolJob`); lacks effect-light mode, glow seed (`SetGlowPlanes` takes two planes), glow inset in walls/flats; `HWViewpointUniforms` smaller: it has the stamps (`S:common/rendering/hwrenderer/data/hw_viewpointuniforms.h:493-497`) but lacks `mBeamScroll[128]` and the fog-light arrays [xc: was "without stamps"]; `hw_drawinfo.cpp` ~1,000 lines fewer (fork uniforms), `hw_entrypoint.cpp` older `CollectLights`; same `gl_line_distance_cull` (S:bsp:460), same `vr_scene_multithread` false (S:hw_cvars.cpp:253) | [V] |

---

## 11. Open questions and the measurements that settle them

| # | Question | Measurement |
|---|---|---|
| Q1 | Is multiview on in the owner's VR config? If not, the CPU scene build runs twice per frame. | Read `vr_openxr_multiview` in the owner's ini via the bridge; `stat rendertimes` "VR: SceneBuild" with it 0 vs 1 (same spot, `cl_capfps 1`). |
| Q2 | In VR, how is the frame's CPU scene time split between walk/clip, `Process`, draw-time vertex+light work, and `Apply`? | `stat rendertimes` (or `r_perflog 1`) in VR: `BSP` (=Bsp−ClipWall, contains flats/sprites setup), `Clip` (contains `W: Setup` in ST), `W: Render` (contains draw-time `SetupLights`+`MakeVertices`+setters), `F: Render`, `Drawcalls` (Apply only). Repeat with `r_dynlights 0` to isolate light gathering. |
| Q3 | Gain from the existing worker in VR. | Same spot, `vr_scene_multithread 1`: compare `Bsp`, `MTWait`, `Scene worker elapsed`; check images identical (render-lane image gate). |
| Q4 | How many impostors/duplicates does the distance cull create, and does B1 hit skybox maps? | No counter exists: add a temporary counter of `Process(isculled)` calls vs unique linedefs, or compare `rendered_lines` with `gl_line_distance_cull 0`; screenshot a skybox map with `gl_line_distance_cull 4000` vs `0`. |
| Q5 | Draw count and material churn per scene. | `stat renderstats` (walls/flats/portals), RenderDoc capture of one eye: count draws, descriptor-set binds; A/B `gl_sort_textures 1`. |
| Q6 | Cost of uniform-block copies/uploads. | Count `HWViewpointBuffer::SetViewpoints` calls and bytes per frame (instrument or RenderDoc buffer updates); compare a scene with and without translucent flats. |
| Q7 | Share of portals. | `gl_portalinfo` once; `stat renderstats` portals; `PortalAll` in `stat rendertimes`; A/B `gl_portals 0` (image changes, cost difference is the portal share). |
| Q8 | Is B5 (heat decay with absolute time) real in play? | Map with `HeatmapAdd` + decay > 0: watch heat lifetime after long uptime vs fresh start; log `drop` per call. |
| Q9 | Does the multiview single walk (left-eye clip) drop right-eye-only geometry? | Multiview on, stand next to a doorway edge, compare right-eye image with multiview off. |
| Q10 | Any ZScript/mod reading `ML_MAPPED`/`SECMF_DRAWN`/billboard `drawPos`/`HeatmapAt` in the owner's fixed mod set (netplay determinism)? | grep the mod set's ZScript for `ML_MAPPED`, `SECMF_DRAWN`, `MoreFlags`, billboard hit-test natives, `HeatmapAt` [xc: HeatmapAt added]. |

---

## Cross-check (date 2026-11-13)

Checked against the code in M (`/home/claude/uzdx_main`), R, N (file-level `diff` against M) and S. Greps for dead/dormant
claims were re-run over all of `src/` and `wadsrc/` (MENUDEF, ZScript, engine .txt). No timings added.

| Category | Checked | Right | Line-fixed | Corrected | Unconfirmed | Added |
|---|---|---|---|---|---|---|
| a. Summary claims | 30 | 25 | 0 | 3 | 0 | 3 |
| b. "What slows it down" (10 items + preface) and bug table B1-B11 | 36 | 27 | 0 | 5 | 0 | 5 (B12, B13, B2/B5/B11 extensions) |
| c. DEAD / DORMANT / never-read / render-only claims | 24 | 21 | 0 | 3 | 0 | 0 |
| d. cvar defaults | 26 | 26 | 0 | 0 | 0 | 0 |
| e. branch differences (N 2, R 10, S 7 sub-claims) | 19 | 17 | 0 | 2 | 0 | 1 |
| f. table / call-graph rows (call graphs ~35 of ~60 lines; per-function 31 of 73; sec. 4 6/9; sec. 5 6/13; sec. 6.2 6/11; sec. 7 14/14; sec. 8 10/14) | 108 | 99 | 0 | 7 | 0 | 2 |

"14 things only the walk produces" (section 7): all 14 are shown in code at the cited lines. Two are not walk-only (11 automap
bits, 13 dither flags); item 2 got a note that M does hold a static side mesh, for the ray tracer only.

Corrections and additions (one line each):
1. Summary 2: the flat path patches vertex z in the static vertex buffer (`vb:406-421`), not the index buffer.
2. Summary 2 / sec. 7 item 2: `DoomLevelMesh` (all sides, map load, `maploader.cpp:3251`) exists on M but feeds only the ray tracer [added].
3. Summary 5 / B12 [added]: distance-cull impostors draw with uninitialised `lightlevel`/`rellight` (no `HWWall` ctor; culled branch never assigns them).
4. Summary 5 / B13 [added]: `IsDistanceCulled` tests only endpoints; a long seg passing near the viewer is culled and its clip range hides near geometry.
5. B2 [added]: on a linedef with near and far segs, far segs lay coplanar full-linedef impostors over the real wall.
6. Summary 7, B5, sec. 8: `Level->HeatIntensity` is read by ZScript (`HeatmapAt`, clearscope native), not "only by the renderer" — the per-scene, uptime-scaled decay is a render→script leak.
7. Summary 9 / sec. 7 item 11: the walk is the only view-driven producer of `ML_MAPPED`/`SSECMF_DRAWN`, not the only producer (`Line_SetAutomapFlags`, UDMF `mapped`, savegame load, ZScript).
8. Sec. 7 item 13: dither flags are also set by `DrawScene` (`di:4461-4464`) outside the walk.
9. B11 [added]: in ST (VR default) `RenderThings` stamps `validcount` first, so the dither scan never collects a target.
10. Sec. 3 `DoSubsector`: 6 `VRSubsector*` clock calls per visible subsector (4 per culled), not 10.
11. Sec. 5: `HWViewpointUniforms` has 5×128 beam arrays, not 6×128 (plus fog-light arrays).
12. Sec. 10 S: Selaco's `HWViewpointUniforms` has the stamps; it lacks `mBeamScroll` and fog lights.
13. Sec. 10 R(b): R:bsp:650 disables wall workers only for the level mesh, not for `r_wallcache`; culled segs still add a clip range under the mesh [added].
14. Sec. 3 `StartScene` / sec. 6 item 7: the fog-light walk also needs `Level->FogDynLights` > 0; it measures from `r_viewpoint.Pos`.
15. B1: camera textures render before the main view's `R_SetupFrame`, so they cull against the previous frame's main-view position.
16. B6: ZScript reads `drawPos` only for view-locked/hand-locked billboards.
17. B7: `HWFlat::Process` may write the fake-sector copy, not always the real sector.
18. B10: portal/sky wall parts also allocate vertices on the scene worker (`walls:913`).
19. Sec. 6.3 / sec. 9: `WallWorkersElapsed` is read (always 0), not only reset.
20. Sec. 8: vertex clip-angle cache render-only claim upgraded from [I] to [V grep].
21. Sec. 9 `gl_noskyboxes`: driven by the menu-exposed `gl_skydome` callback.
22. Sec. 3 `SetupHeatmap` [added]: 65,536-cell loop per scene plus two 256 KB heap arrays + memcpy whenever a cell moved.
23. Summary 10: cross-reference fixed (section 10, not 9).
