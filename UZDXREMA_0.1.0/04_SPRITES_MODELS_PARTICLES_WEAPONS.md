# 04 — Everything that is not level geometry: sprites, models, particles, decals, weapons, HUD

Tree of record: `main` = `/home/claude/uzdx_main` (f182cbcc). All paths below are relative to that tree unless a
branch is named. VERIFIED = I read the code that does it. INFERRED = deduced; the basis is stated.
Helion references are to `/home/claude/ext/Helion` (read for the comparison section only).

## 1. Summary

1. Actors become draws only through the BSP walk: `ProcessVisibleSubsector` calls `RenderThings` once per reached
   sector and `RenderParticles` once per reached subsector (`hw_bsp.cpp:1038-1099`). There is no per-sprite frustum
   test; visibility granularity is "the sector was reached" plus two distance culls (2000 all actors, 1200 decor).
2. In VR the scene build is single-threaded by default (`vr_scene_multithread` false, `hw_cvars.cpp:278`,
   `hw_bsp.cpp:1438-1439`) and, with `vr_openxr_multiview` false by default (`hw_vrmodes.cpp:643`), the whole scene is
   built and drawn once per eye (`hw_entrypoint.cpp:354,372,481-482`): every sprite `Process`, the sort, every
   `DrawSprite`, every bone evaluation, every HUD model and every laser trace run twice per frame.
3. Every sprite, particle, decal and model surface is its own draw call with a full `VkRenderState::Apply` (stream
   data + matrix block writes, `vk_renderstate.cpp:223-248`). No batching or instancing exists for actors. Only the
   fork's GPU particle ring, debris pool and mesh particles are instanced/one-draw.
4. All non-model sprites (solid ones included) except solid wall sprites (`RF_WALLSPRITE`), and all particles, go to
   `GLDL_TRANSLUCENT` (`hw_drawlistadd.cpp:136-151`) [xc: solid non-model wall sprites go to `GLDL_MODELS`, condition at
   `hw_drawlistadd.cpp:140`] and are BSP-style sorted on the render thread inside `RenderTranslucent` (`hw_drawlist.cpp:702-708, 931-948`).
5. On Vulkan sprite vertices are built at draw time, not in the scene build (`BuffersArePersistent()` true,
   `vk_renderdevice.cpp:510`; `hw_sprites.cpp:459-462, 854-857`), so billboard math is on the draw thread.
6. Dynamic light on sprites/particles is summed on the CPU per sprite per draw (`hw_sprites.cpp:315-323` →
   `hw_spritelight.cpp:101-210`), including an AABB ray test per shadow-casting light; lit models gather lights with a
   `BSPWalkCircle` per model per scene (`hw_spritelight.cpp:228-321`).
7. Skeletal (IQM) bones are evaluated on the CPU per model per draw (no frame cache), ~5 4x4 multiplies per bone,
   copied to an 80,000-bone buffer (`models.cpp:4122-4241`, `models_iqm.cpp:823-965`, `hw_bonebuffer.cpp:27,71-100`).
8. The draw path calls the ZScript VM: `AnimateBones` per decoupled-animation model draw (`hw_sprites.cpp:501-507`),
   `ModifyBobLayer`/`ModifyBobLayer3D`/`ModifyBobPivotLayer3D` per psprite (`hw_weapon.cpp:2412,2518,2531`).
9. A walk-free renderer has every index it needs already: `sector_t::touching_renderthings` is exactly the set
   `RenderThings` reads from its first list (`p_maputl.cpp:441-477`); the walk also draws `sectorportal_thinglist` and
   `lineportal_thinglist` actors, which are separate lists [xc: "exactly the set the walk draws" overstated: the two
   portal lists (`hw_bsp.cpp:965-1001`, `hw_sprites.cpp:3224-3306`) are not in `touching_renderthings`]; the blockmap
   misses `MF_NOBLOCKMAP` and client-side actors; Helion uses its own one-block-per-entity render blockmap, linked
   incrementally on every `LinkToWorld` (`PhysicsManager.cs:135`), whose draw list is rebuilt per tic, interpolated in
   the shader, with OIT instead of a sort [xc: the render blockmap itself is not rebuilt per tic; the entity draw list
   is (`LegacyWorldRenderer.cs:143,331-339`). Helion also has a BSP mode that finds entities by sector,
   `LegacyWorldRenderer.Bsp.cs:56-69`, chosen by `UseBspBasedOnHeuristic`, `LegacyWorldRenderer.cs:306`].
10. Dead: coronas (`hw_drawinfo.cpp:4287-4333` `#if 0` + call commented at `:4356-4359`, `gl_coronas` never read),
    `gl_weapon_purelightlevel` (defined, never read). Dither-target collection is logically dead in single-thread mode.

## 2. Entry points and call graph

### 2.1 Frame → eye → scene (VERIFIED)

```
D_Render (d_main.cpp:690-711)                         once per frame
  P_FindParticleSubsectors(Level)  (p_effect.cpp:204)  O(numsubsectors) clear + link VisualThinkers and particles
  action() → RenderView → RenderViewpoint (hw_entrypoint.cpp:264)
    mBones->Clear(), mLights->Clear()  (hw_entrypoint.cpp:675-676)   once per frame [xc: these are in RenderView before
                                       the canvas and camera-texture loops, not inside RenderViewpoint; camera textures
                                       rendered in the same frame use the same bone/light buffer capacity]
    PrepareFrameCompute (hw_entrypoint.cpp:217-234)    once per frame: DebrisPool::PrepareFrame, SurfaceDamage::PrepareFrame ...
    for eye_ix in eyes (hw_entrypoint.cpp:359-516)
      renderSceneThisEye = !useMultiviewScene || eye_ix == 0     (:372)
      HWDrawInfo::ProcessScene (hw_drawinfo.cpp:4597)               per eye unless multiview
        mBones->Map; GPU ring Sync, mesh-particle Sync, SyncDrawnLines, SyncViewLights ...
        DrawScene(DM_MAINVIEW) (hw_drawinfo.cpp:4450)
          CreateScene (hw_drawinfo.cpp:3665)
            RenderBSP (hw_bsp.cpp:1422)
              RenderBSPNode → DoSubsector → ProcessVisibleSubsector
                 RenderParticles(sub)   per visible subsector with particles/VisualThinkers
                 RenderThings(sub)      once per visible sector
              P_CheckSight per dither target (hw_bsp.cpp:1547-1553)
              mCurrentPortal->RenderAttached (hw_bsp.cpp:1556)       portal views only
              PreparePlayerSprites (hw_bsp.cpp:1558-1563)             main view only
            DispatchBillboards (hw_drawinfo.cpp:3694)                 the fork's UI billboards, no walk
          RenderScene (hw_drawinfo.cpp:3722)   GLDL_MODELS draw (:3831), decals (:3837)
          MeshParticleBuffer::Draw, DebrisPool::DrawMeshes (:4510-4533)
          DrawPlayerSprites (:4536-4541)         VR: psprites inside the scene, per eye
          DrawWorldPlayerSprites (:4545-4550)    other players' weapon models
          portalState.EndFrame (:4562)           portals recurse into DrawScene(DM_PORTAL) → sprites again
          RenderTranslucent (:4570)  → GLDL_TRANSLUCENT DrawSorted, GPU particles, debris billboards, drawn lines
          DrawMountedHud, DrawHitscanTracers, DrawLaserSightWorld, VRWheel_Draw (:4579-4586)
      EndDrawScene / DrawEndScene2D (hw_entrypoint.cpp:488,501-503)   psprite paths gated off in VR (hw_drawinfo.cpp:4364,4400)
  Flush3D → VR eye composite: Draw2D(true)/Draw2D(false) per eye (vk_renderdevice.cpp:680-711)
```

### 2.2 How actors are found by the walk (VERIFIED)

- `ProcessVisibleSubsector` (`hw_bsp.cpp:1038`):
  - particles: if `sub->sprites.Size() > 0 || ParticlesInSubsec[sub] != NO_PARTICLE` → `RenderParticles` (queued as
    `ParticleJob` when multithreaded) (`:1045-1057`). Per **subsector**.
  - things: inside `if (sector->validcount != validcount)` (first subsector of that sector this scene), if
    `touching_renderthings || sectorportal_thinglist` → `RenderThings` (queued as `SpriteJob` when multithreaded)
    (`:1066-1084`). Per **sector**.
- `RenderThings` (`hw_bsp.cpp:914-1003`):
  - walks `sub->sector->touching_renderthings` (msecnode_t list). Dedup per scene: `thing->validcount == validcount`
    (`:923-924`). Skip secret sectors in OoB mode (`:927`). Per-class `distancecheck` cvar (`:928-937`).
    `ShouldCullDecorRenderThing` (`:71-80,942`): any actor that is not a player, not `MF_SPECIAL|MF_SHOOTABLE|MF_MISSILE`,
    not `MF3_ISMONSTER`, at distance ≥ `gl_sprite_decor_distance_cull` (1200, `hw_sprites.cpp:1042`) is skipped.
    `CurrentMapSections[thing->subsector->mapsection]` (`:947`). Optional sprite shadow (`:952-959`), then
    `HWSprite::Process` (`:961`).
  - walks `sec->sectorportal_thinglist` (actors on the other side of a sector portal plane) with the same culls but
    **no validcount dedup** (`:965-1001`), `Process(..., thruportal=1)`.
- `RenderParticles` (`hw_bsp.cpp:1005-1036`): `sub->sprites` (DVisualThinker array rebuilt each frame) then the
  `Level->Particles` chain from `ParticlesInSubsec[sub]`; `mClipPortal->ClipPoint` per item in portal views;
  `HWSprite::ProcessParticle`. No distance cull.
- Line portals: `HWWall` processing calls `ProcessActorsInPortal` (`hw_walls.cpp:997`, `hw_sprites.cpp:3224-3306`):
  walks `FLinePortal::lineportal_thinglist`, temporarily **moves the actor** (`SetXYZ`, `Prev`, `Angles.Yaw`) to the
  portal's far side, calls `Process(..., thruportal=2)`, moves it back. Runs on the scene worker when multithreaded.
- Portal views: `HWLineToLinePortal::RenderAttached` → `ProcessActorsInPortal` (`hw_portal.cpp:821-823`,
  `hw_bsp.cpp:1556`).
- Player weapon: `PreparePlayerSprites` (`hw_weapon.cpp:2654-2687`) after the walk, main view only.
- UI billboards: `DispatchBillboards` (`hw_drawinfo.cpp:3546-3663`) iterates `Level->Billboards` (flat list) and calls
  `Level->PointInSector` per billboard: the one actor-like path that does not use the walk.

### 2.3 Threading (VERIFIED)

- `multithread = gl_multithread && (!IsVRScene || vr_scene_multithread)` (`hw_bsp.cpp:1438-1439`). Defaults:
  `gl_multithread` true (`:46`), `vr_scene_multithread` false (`hw_cvars.cpp:278`). So **flat screen**: one scene
  worker (`renderPool.push(WorkerThread)`, `:1447-1449`) runs `RenderThings`/`RenderParticles`/walls/flats while the
  main thread walks the BSP; **VR**: everything on the main thread.
- `gl_bsp_worker_threads` (default 1) only adds wall workers (`hw_bsp.cpp:1440, 1450-1468`); sprites stay on the one
  scene worker.
- Draw time (`DrawSprite`, `RenderModel`, bones, sort) is always on the main/render thread.
- Data race: in multithread mode the worker sets `thing->validcount` (`hw_bsp.cpp:924`) while the main thread's dither
  loop reads it (`:1088-1096`); also `ProcessActorsInPortal` moves actors on the worker. INFERRED harmless today
  because the playsim is not running during the render, but it is a race on the same field.

## 3. Per-function table

Cost classes: S = per visible sprite/particle, M = per model draw, B = per bone, P = per psprite, L = per light
touching, E = per eye (non-multiview), F = per frame, T = per tic (playsim). "Thread": W = scene worker when
multithreaded (flat), else main; R = render/main.

### 3.1 Finding and processing (scene build)

| Function | file:line | Called from | Per | Thread | Cost class / notes |
|---|---|---|---|---|---|
| `P_FindParticleSubsectors` | p_effect.cpp:204-239 | D_Render d_main.cpp:701 | F | main | O(numsubsectors) `sprites.Clear()` + `fillshort`, O(VisualThinkers) push, O(particles) link. Not per eye. |
| `P_ThinkParticles` | p_effect.cpp:278-351 | p_tick.cpp:760 | T | main | O(active particles): fade, `GetPortalOffsetPosition`, `PointInRenderSubsector` per particle. |
| `HWDrawInfo::ProcessVisibleSubsector` | hw_bsp.cpp:1038-1181 | DoSubsector :1344 | visible subsector, E | main | Gates particle job per subsector, thing job per sector, dither loop. |
| `HWDrawInfo::RenderThings` | hw_bsp.cpp:914-1003 | :1081 / worker :366 | visible sector, E | W | O(things touching the sector); two `LengthSquared`, `distancecheck` cvar deref, decor cull, mapsection test. Clocks `VRThingBuild` in VR. |
| `HWDrawInfo::RenderParticles` | hw_bsp.cpp:1005-1036 | :1055 / worker :376 | visible subsector, E | W | O(VisualThinkers + particles in subsector). Clocks `SetupSprite` itself AND its callers clock it (`:1054-1056`, `:374-377`): nested → double-counted. |
| `HWSprite::Process` | hw_sprites.cpp:1057-1988 | RenderThings, ProcessActorsInPortal, RenderAttached | S, E | W | Large constant: `IsDistanceCulled` (`:1044-1055`, uses global `r_viewpoint.Pos`), `IsVisibleToPlayer`, `GetCVar` by name if `VisibleCVar`/`AlphaCVar` set (`:1094-1108`), `InterpolatedPosition/Scale/Alpha`, `FindModelFrame` hash (`:1197`), voxel distance latch (`:1236-1257`), background texture/model cache checks on sprite/frame change (`:1261-1361, 1532-1563`), `GetSpriteFrame` rotation, anamorphosis (4 `ZatPoint` + sqrt, `:1714-1762`, `gl_spriteclip` -1 default), light level + `CheckSpriteGlow`, render style resolution, `PutSprite`. Writes actor fields: `VoxelFarLatch`, `lastModelSprite/Frame`, `LastPatch`, `lastScaleFlags`, `isoscaleY`, `isotheta`, toggles `RF_XFLIP` twice, sets `MF7_FLYCHEAT`. |
| `HWSprite::PerformSpriteClipAdjustment` | hw_sprites.cpp:941-1033 | Process :1641 | S | W | Only when `gl_spriteclip > 0`; default -1 → DORMANT. |
| `HWSprite::SplitSprite` | hw_sprites.cpp:892-933 | Process :1973 | S | W | Only when `RFL_NO_CLIP_PLANES` (GL/GLES caps, `gl_interface.cpp:135-187`); never set on Vulkan (`vk_renderdevice.cpp:510`) → unreachable on Vulkan. |
| `HWSprite::PutSprite` | hw_sprites.cpp:840-859 | Process, ProcessParticle, Billboard | S, E | W | For lit per-pixel models: `hw_GetDynModelLight` + `mLights->UploadLights` (M×L). On Vulkan no vertices here (persistent buffers). `AddSprite`. |
| `HWDrawInfo::AddSprite` | hw_drawlistadd.cpp:136-151 | PutSprite | S | W | Translucent, particle, or non-model non-wallsprite → `GLDL_TRANSLUCENT`; else `GLDL_MODELS`. Copies the whole `HWSprite` into the arena (`NewSprite`, hw_drawlist.cpp:776-781). |
| `hw_GetDynModelLight` | hw_spritelight.cpp:228-321 | PutSprite :847, PreparePlayerSprites3D :2548, DrawWorldPlayerSprites :2633 | M, E | W/R | `BSPWalkCircle` over the actor's render radius; per section TMap iteration of lights; dedupe by `std::find` (O(L²)) unless `gl_light_model_dedupe_cache` (default false, hw_dynlightdata.cpp:42). Counters `dynlights_model_*`. The "section already done" early-out (`hw_spritelight.cpp:259`) never fires: `dl_validcount` is incremented (`:253`) but `section->validcount` is never set to it in this function, so a section's whole light TMap is re-walked for every subsector of that section the circle touches, duplicates caught only by the dedupe [xc: added]. |
| `HWSprite::ProcessParticle` | hw_sprites.cpp:1997-2186 | RenderParticles | particle, E | W | Light level, 3D-floor lightlist scan, colormap, `TexAnim.UpdateStandaloneAnimation` for animated particles, quad extents. VisualThinkers → `AdjustVisualThinker` (:3132-3216). |
| `HWDrawInfo::ProcessActorsInPortal` | hw_sprites.cpp:3224-3306 | hw_walls.cpp:997, hw_portal.cpp:823 | line portal span, E | W | `TMap<AActor*,bool>` built per call; moves actors temporarily. No decor/distance cull of its own beyond Process's. |
| `HWDrawInfo::DispatchBillboards` | hw_drawinfo.cpp:3546-3663 | CreateScene :3694 | billboard, E | main | O(N) + `std::sort` of distances when `rs_bb_maxpanels` > 0; `PointInSector` per billboard; `ProcessBillboard` (hw_sprites.cpp:2876). |
| `HWDrawInfo::PreparePlayerSprites` | hw_weapon.cpp:2654-2687 | RenderBSP :1561 | E (main view) | main | Calls 3D then 2D pass, then targeter. Clocked by `VRPlayerSprites` in VR. |
| `PreparePlayerSprites3D` | hw_weapon.cpp:2435-2566 | above | P, E | main | `HudAnchor_BeginFrame`; per psprite `FindModelFrame`; `GetWeaponPosition3D` (two `FindPSprite`); VM `ModifyBobLayer3D`, `ModifyBobPivotLayer3D` when overridden and `PSPF_ADDBOB`; `GetWeaponRenderStyle`; with `gl_light_weapons` (default false) `hw_GetDynModelLight` + `FindLightProbe`. |
| `PreparePlayerSprites2D` | hw_weapon.cpp:2298-2433 | above | P, E | main | `FindModelFrame` per psprite; nested owner scan when `r_hudflatoverlay < 1`; VM `ModifyBobLayer`; `GetDynSpriteLight` for the player (`gl_light_sprites`). |
| `PrepareTargeterSprites` | hw_weapon.cpp:2696-2737 | above | E | main | Trivial. |
| `HWDrawInfo::GetDynSpriteLight(AActor*,particle*)` | hw_spritelight.cpp:212-222 | DrawSprite :320, PreparePlayerSprites2D :2426 | S, E | R | See 3.2. |

### 3.2 Drawing

| Function | file:line | Called from | Per | Thread | Cost class / notes |
|---|---|---|---|---|---|
| `HWDrawList::DrawSorted` / `Sort` / `DoSort` | hw_drawlist.cpp:931-948, 702-708, 629-695 | RenderTranslucent :3871 | E | R | Builds `SortNode` list over all translucent items; recursive split by translucent flats (`SortSpriteIntoPlane`, :323-372, **always splits models**, `|| ss->modelframe` :333) and walls (`SortSpriteIntoWall`, :483-574; with default `gl_billboard_faces_camera` true sprites are not split, :515-535); remaining sprite runs `std::stable_sort` by `depth` (:598-622). Maps/unmaps `mVertexData` around the sort. Not clocked separately (inside `RenderAll`/`VRSceneDraw`; renderer branch adds a Tracy zone `SortTranslucent`). |
| `HWDrawList::DoDraw` | hw_drawlist.cpp:788-820 | Draw, DrawSorted | item | R | Clocks `RenderSprite` around `DrawSprite`. |
| `HWSprite::DrawSprite` | hw_sprites.cpp:140-572 | DoDraw | S (+ per 3D-floor light slice), E | R | `FogScaleForSector`, `SetGlowPlanes`, render style/alpha func, outline, post mask, effect-light mode, **`GetDynSpriteLight` when `dynlightindex == -1`** (every non-model sprite and particle when the level has dynamic lights), object/add color, `SetColor`, `SetFog`, `SetMaterial`, then per lightlist slice: `SetSplitPlanes` + `CreateVertices` (persistent path, :459-462) + `Draw(DT_TriangleStrip, 4)`; colored-fog subtractive sprites draw twice (:482-490). Models: `FindLightProbe`, VM `AnimateBones` for `MF9_DECOUPLEDANIMATIONS` (:501-507), `RenderModel`. Resets ~8 states after. |
| `HWSprite::CreateVertices` / `CalculateVertices` | hw_sprites.cpp:867-883 / 593-832 | DrawSprite (Vulkan) | S, E | R | Camera-facing billboard (default) → `Matrix3x4` build, `atan2`, rotate, 4 matrix-vector products; `AllocVertices(4)`. |
| `HWDrawInfo::GetDynSpriteLight(...)` | hw_spritelight.cpp:101-210 | DrawSprite | S×L, E | R | `FindLightProbe` 3x3 cell scan (:40-93); iterates the section's whole `flat_dlist` TMap; per light in radius: optional spot smoothstep, **`mShadowMap.ShadowTest` = CPU AABB-tree ray** when `light->shadowmapped` and `LightShadowAllowed(asksToCast)` (hw_shadowmap.cpp:154-160). |
| `RenderModel` | models.cpp:724-764 | DrawSprite :539 | M, E | R | `getFlags`, `ObjectToWorldMatrix` (:1624-1908 → :1910-2673: VR rig/worn-bone/follow frames, placement prefix → up to ~13 `FString::Format` + `GetCVar` lookups, :2445-2512), eye-fade and look-down fade cvar lookups by formatted name (:662-722), `BeginDrawModel`, `RenderFrameModels`, `EndDrawModel`. |
| `RenderFrameModels` | models.cpp:4595-4644 | RenderModel, RenderHUDModel | M | R | `CalcModelFrame` (:3635-3782, `FindModelFrame` of the next state), then per sub-model `CalcModelOverrides` + `RenderModelFrame`. |
| `RenderModelFrame` | models.cpp:4122-4593 | above | M×submodel | R | `BuildVertexBuffer` (first use only), `ProcessModelFrame` → bones, `ModelJointDrive_Apply`, `VRAvatar_Pose` (avatar only), `ApplyBoneLengths`, `UploadBones`, `HudAnchor_Store`, per-surface override table (16 slots) incl. live hand drive (`GetHandTransform`), `mdl->RenderFrame`. |
| `ProcessModelFrame` | models.cpp:4012-4120 | RenderModelFrame | M | R | `CalculateBones` / `CalculateBonesOnlyOffsets`. |
| `IQMModel::CalculateBonesIQMSpecialized` | models_iqm.cpp:823-965 | CalculateBones | B | R | Per bone: up to 3 `InterpolateBone` (quat nlerp + normalize), `loadIdentity/translate/multQuaternion/scale`, 4-5 `multMatrix` (16 MAC×4 each). Writes the model's **shared** `boneData` member (not per actor). |
| `IQMModel::RenderFrame` | models_iqm.cpp:528-584 | RenderModelFrame | M×mesh | R | `SetupFrame` (uploads the model's `boneData` member if no start index given, :531; on the plain non-attachment, unposed path this is the only upload, not a second one [xc: "again" was wrong for that path]), per mesh: `SetMaterial` on skin change, `DrawElements` (one `DrawIndexed` per mesh), blended surface = second draw. |
| `BoneBuffer::UploadBones` | hw_bonebuffer.cpp:71-100 | above | M | R | `memcpy` 64 B × bones into an 80,000-bone persistent buffer (`:27`); full → returns -1 → model drawn unskinned, `DPrintf` warning. Cleared once per frame (`hw_entrypoint.cpp:676`), so both eyes share the capacity. |
| `FHWModelRenderer::BeginDrawModel` etc. | hw_models.cpp:47-284 | RenderModel | M | R | State setup; `SetEyeFade` switches `EFF_EYEFADE`. |
| `HWDrawInfo::DrawPlayerSprites` | hw_weapon.cpp:1534-1600 | DrawScene :4539 (VR), EndDrawScene/DrawEndScene2D (flat) | P, E | R | Hand resolution, `VRWheel_ShouldSuppressWeaponHand`, `AdjustPlayerSprites` for flat layers, `DrawPSprite`. Not covered by any CPU clock except `VRSceneEyes`/`All`. |
| `HWDrawInfo::DrawPSprite` | hw_weapon.cpp:149-370 | above, DrawWorldPlayerSprites | P, E | R | Model: `RenderHUDModel`. Flat: `SetMaterial`+`Draw`; then **unconditionally** `FindState(NAME_Spawn)`, `GetSpriteFrame`, `FMaterial::ValidateTexture` (:256-267) even though the crossed-quad block they feed only runs when `r_PlayerSprites3DMode != BACK_ONLY` (default 1 = BACK_ONLY, r_utility.cpp:153, hw_weapon.cpp:130-136). |
| `RenderHUDModel` | models.cpp:3053-~3580 | DrawPSprite :238 | P, E | R | `GetViewToWorldMatrix` (4x4 inverse), placement cvars by formatted name (:3117-3147), `GetWeaponTransform`, mod-hand fit, `BeginDrawHUDModel`, `RenderFrameModels` (bones as above). |
| `HWDrawInfo::DrawWorldPlayerSprites` | hw_weapon.cpp:2586-2652 | DrawScene :4548 | other players × P, E, every non-sky-portal scene | R | For 4-player co-op: 3 players × model layers per eye in the main view; in mirror/camera/portal scenes the local player's own layers are drawn here too (skip only for `DM_MAINVIEW`, hw_weapon.cpp:2600-2606) [xc: added]; each `FindModelFrame`, `GetWeaponLighting`, `GetWeaponPosition3D`, optional model light gather, `DrawPSprite`. `vr_weapons_world` default true (models.cpp:2674). |
| `DrawLaserSightWorld` | hw_weapon.cpp:1397-1526 | DrawScene :4584 | hand, E | R | `GetLaserBeamEndpoints` → `Trace` (:776) per hand per eye, plus other players' hands in multiplayer. |
| `DrawHitscanTracers` | hw_weapon.cpp:1333-1395 | DrawScene :4583 | tracer, E | R | `erase(remove_if)` per eye, geometry per live tracer. |
| `HWDrawInfo::DrawDecals` / `HWDecal::DrawDecal` | hw_decal.cpp:149-177 / 44-142 | RenderScene :3837 | decal (+ slice), E | R | One `Draw` per decal (per lightlist slice); `FindLightProbe` per decal when probes exist; no sort, grouped only by consecutive side. |
| `HWWall::ProcessDecals` / `ProcessDecal` | hw_decal.cpp:485-499 / 208-477 | `HWWall::PutWall` hw_walls.cpp:875-886 | visible solid wall part × decals on its side, E | W/main | Walks `sidedef->AttachedDecals`; clip to the wall part; background texture check; `AddDecal` + `AllocVertices(4)`. On the wall-worker path `PutWall` runs at merge time on the main thread (hw_bsp.cpp:1501-1508). |
| `RenderTranslucent` GPU ring draw | hw_drawinfo.cpp:3890-3995 | DrawScene | E | R | `SetSceneDepthReadable(true/false)` = `EndRenderPass` + new pass each way (vk_renderstate.cpp:598-640) when `r_gpuparticles_soft` > 0 (default 8, hw_cvars.cpp:311) or a soft particle/piece is alive; the whole block runs only with no portal (`mCurrentPortal == nullptr`) and once the level has written any GPU particle or has debris billboards (`Level->GpuParticleWritten > 0 || HasBillboardsAt`, hw_drawinfo.cpp:3890-3891) [xc: added gate]; then 1-2+ draws of `ranges × 6` vertices; `DebrisPool::DrawBillboards` one draw of `highWater × 6` vertices. |
| `GpuParticleBuffer::Sync` | hw_gpuparticlebuffer.cpp:164-244 | ProcessScene :4651 | E (second eye early-outs on serial, :170) | R | memcpy of new 80-B records (or full ring on new level/burst ≥ ring, 65536 records default = 5.2 MB) into a single persistently mapped buffer; `SpawnRecordLights`. |
| `MeshParticleBuffer::Draw` | hw_meshparticles.cpp:492-534 | DrawScene :4514 | mesh definition | R | One `DrawIndexedInstanced` per definition. |
| `DebrisPool::PrepareFrame` / `DrawMeshes` | hw_debrispool.cpp:374-564 / 1284-1329 | PrepareFrameCompute / DrawScene :4528 | F / mesh def | main/R | CPU side is queue bookkeeping (`fx.debrispool` CPU sample); simulation is GPU compute; meshes instanced. |
| `SyncDrawnLines` | hw_drawinfo.cpp:372-~520 | ProcessScene :4674 | E | main | Rebuilds the drawn-line list every scene (beam slots, wakes). |
| `SurfaceDamage::WallKey/FlatKey` | hw_surfacedamage.cpp:146-160 | wall/flat draws | wall/flat draw | R | `unordered_map::find` per draw while the damage atlas is drawable. |
| `Draw2D` | hw_draw2d.cpp:51-290 | vk_renderdevice.cpp:699,707 | pass × eye | R | Per call: `state.Clear(CT_Stencil)` (:86) → `EndRenderPass` on Vulkan (vk_renderstate.cpp:163-181), full vertex/index upload of the 2D lists, one draw per command. Called up to twice per eye in VR, each call only when `twod->HasCommandsForPass(true/false)` and no screen layer/cinema mode (vk_renderdevice.cpp:671,697,705) [xc: not unconditionally twice]. Clocked by `twoD` and inside `VREyeComposite`. |
| `DrawHudToSurface` | d_main.cpp:1254-~1440 | D_Display | F | main | Mounted HUD: status bar `CallDraw` (VM) once per frame into a canvas; face HUD path (d_main.cpp:1739-1790) is the alternative, never both. |

## 4. Data structures that matter for speed

| Structure | Where | Notes (VERIFIED unless marked) |
|---|---|---|
| `sector_t::touching_renderthings` (msecnode_t) | r_defs.h:723; built by `P_CreateSecNodeList(this, RenderRadius(), ...)` in `AActor::LinkToWorld` p_maputl.cpp:471 | Every sector the actor's `max(radius, renderradius)` box crosses (`actor.h:2922-2925`). Excludes `MF_NOSECTOR` (`:441`) and `renderradius < 0` (`:471-476`). **Includes client-side actors** (the `!clientside` guard at `:443` does not cover `:471`). Maintenance is a playsim cost: every relink runs `FBlockLinesIterator` over the box twice (radius list and render list) (p_secnodes.cpp:216-283); once for client-side actors, which build only the render list [xc: added]. Membership = sectors of every line that crosses the box, plus the centre sector (p_secnodes.cpp:236-264) [xc: added]. |
| `sector_t::thinglist` (AActor snext/sprev) | p_maputl.cpp:449-454 | Own sector only; excludes `MF_NOSECTOR` and client-side actors. |
| `sector_t::touching_thinglist` | p_maputl.cpp:468 | Gameplay radius; excludes `MF_NOSECTOR` and client-side. |
| `sector_t::sectorportal_thinglist`, `FLinePortal::lineportal_thinglist` | p_secnodes.cpp:375-427 | Rebuilt by `UpdateRenderSectorList` on every `LinkToWorld` outside map spawn (p_maputl.cpp:533): its `Pos() != OldRenderPos` gate is always true because `OldRenderPos` is only ever set to FLT_MAX at spawn (p_mobj.cpp:5716) and never updated. Both lists are cleared and re-made each time; the line-portal part scans `Level->linePortals` only when the map has line portals and a neighbour block holds one; the sector-portal part walks non-blocking ceiling/floor portals within `SPRITE_SPACE` (64) [xc: "only when the actor moved and is near a portal" was wrong]. |
| Blockmap `blocklinks` / `FBlockNode` | p_maputl.cpp:480-531 | One node per block the **gameplay radius** touches, per connected portal group. Excludes `MF_NOBLOCKMAP` and client-side actors (`:481`). |
| `Level->Particles`, `ParticlesInSubsec` | p_effect.cpp:131-239 | Array of `r_maxparticles` (default 4000, g_cvars.cpp:115) with t-links; per-subsector s-links rebuilt every frame. |
| `subsector_t::sprites` (DVisualThinker*) | p_effect.cpp:207-218 | Rebuilt every frame for every subsector. |
| `HWSprite` | hw_drawstructs.h:390-445 | ~60 scalar fields + pointers + `DRotator` (3 doubles) + `bbVerts[4]` (48 B): INFERRED ~250 B (sizeof not measured). Copied by value into the draw list (`hw_drawlistadd.cpp:149-150`); copied again for splits. |
| `RenderDataAllocator` (FMemArena, 1 MB blocks) | hw_drawlist.cpp:30-35 | All HWWall/HWFlat/HWSprite/HWDecal allocations; reset per frame. |
| `SortNodes` (StaticSortNodeArray) | hw_drawlist.cpp:42-63 | Grows by `new SortNode` once; released per list. |
| `HWDrawItem` lists `drawlists[GLDL_*]` | hw_drawlist.cpp | One per list; `GLDL_TRANSLUCENT` holds all sprites except solid models and solid wall sprites (those go to `GLDL_MODELS`, hw_drawlistadd.cpp:140-147) [xc: "all sprites" overstated], particles, translucent walls/flats. |
| `FSpriteModelFrame` hash `SpriteModelHash` | models.cpp:5451-5487 | O(1) chained hash per `FindModelFrame`; voxel fallback by sprite frame. Called per Process, per psprite per pass, per `CalcModelFrame` next state. |
| `BoneBuffer` | hw_bonebuffer.cpp:25-100 | 80,000 × 64 B = 5.1 MB, one pipeline copy on Vulkan (`vk_renderdevice.cpp:611`, header default 1), atomic `fetch_add` index. |
| `IQMModel::boneData` | models_iqm.cpp:827 | One scratch palette **per model, not per actor**: the result must be uploaded before the next actor using the same model is evaluated. Prevents evaluating bones of two instances in parallel. |
| Joint-drive `FPoseCache` | r_data/model_jointdrive.cpp:166-176, 567-573 | The one per-frame cache in the bone path: the local joint edits of a registered joint drive, keyed on `screen->FrameCount`, replayed for the second eye. It caches edits, not the palette; the IQM palette is still recomputed per draw. Inert unless a drive is registered or `r_jointpose_test` is set (model_jointdrive.h:46,79). [xc: added] |
| `mVertexData` sprite quads | hw_sprites.cpp:873 | 4 vertices per sprite per draw, allocated at draw time on Vulkan. |
| GPU particle ring | hw_gpuparticlebuffer.cpp:57-113 | `r_gpuparticles_ringsize` (65536) × 80 B persistent SSBO + static 6-vertex quad buffer per slot. |
| `Level->Billboards` | hw_drawinfo.cpp:3548-3662 | Flat array; walk-free precedent. |
| `FLightNode` TMap per section (`lightlists.flat_dlist`) | hw_spritelight.cpp:118-122, 261-267 | Iterated per sprite draw and per model per scene. |
| Renderer branch only: `FWorldSnapshot` `Recs` + CSR `ThingIndex`/`PortalThingIndex` | uzdxrema src/rendering/r_snapshot.cpp:131-151, 312-314; r_snapshot.h:34-60 | Per-sector CSR image of `touching_renderthings` and `sectorportal_thinglist` with `FSnapActorRec` (pos, prev, angles, sprite, frame, style, flags, sector, subsector, light). Written after `P_PredictClient` (d_net.cpp:2493,2557) only when `r_snapshot` ≥ 1 (default 0). **No reader exists** (grep: only writers): DORMANT measurement code. |

## 5. What slows it down (ranked)

Rank by expected impact on the owner's VR case (Quest 2 PCVR, BD22 + LegenDoom + RSQ + environment mods).
Previously measured figures are quoted with their source; none are mine.

1. **Two full scene builds per frame in VR (no multiview by default).** VERIFIED: `vr_openxr_multiview` false
   (`hw_vrmodes.cpp:643`); with it off `renderSceneThisEye` is true for both eyes (`hw_entrypoint.cpp:354,372`) and
   `ProcessScene` runs per eye (`:481-482`), each with its own `RenderBSP` (`validcount++`, `hw_bsp.cpp:1436`), so
   every `HWSprite::Process`, `PutSprite` model-light gather, sort, `DrawSprite`, bone evaluation, `RenderHUDModel`,
   `DrawWorldPlayerSprites`, laser `Trace`, and `Draw2D` pair runs twice. Confidence high that sprite CPU cost ~doubles;
   the owner's ini may already have multiview on (CVAR_ARCHIVE). Settle: `bench` header line "XR: multiview=… active=…"
   (`hw_clock.cpp:325-334`), then `stat rendertimes` "S: Render/Setup" and "VR: Things/PlayerSprites" with
   `vr_openxr_multiview 0` vs `1`.
2. **Main-thread-only scene build in VR.** VERIFIED (`hw_bsp.cpp:1438-1439`, `vr_scene_multithread` false). Sprite
   processing is serialized with the BSP walk and clipping. Settle: `VRThingBuild` ("VR: Things") vs `VRSceneBuild`
   in `stat rendertimes`.
3. **One draw call + full Apply per sprite/particle/decal/model mesh; every sprite in the sorted translucent list**
   (except solid models and solid wall sprites [xc: qualifier added]).
   VERIFIED (`hw_sprites.cpp:476`, `hw_decal.cpp:106`, `models_iqm.cpp:567`, `vk_renderstate.cpp:59-65,223-248`,
   `hw_drawlistadd.cpp:140-143`). Per draw: `ApplyStreamData` + `ApplyMatrices` (writes a matrix block every Apply,
   `:448-455`), push constants, buffer/material binds. With BD22 gore and LegenDoom effects the visible sprite count
   is the multiplier. Previously measured: flat view in Abandon MAP13, "sprites 0.8-3.5 ms" scene-build share
   (HANDOFF_RENDER_2026-11-11.md:273) and "sprites 1.2-4.3" (docs/plans/STAGE3_VISIBILITY_PLAN.md:79), flat screen,
   renderer-branch builds named there. Settle: `stat renderstats` "Sprites:" count (`rendered_sprites`, incremented in
   Process/ProcessParticle `hw_sprites.cpp:1987,2185`), `RenderSprite` vs `SetupSprite`, `drawcalls`; renderer
   branch: perf_track `cpu_sprites` (= SetupSprite + RenderSprite, hw_perftrack.cpp:543 on renderer) and GPU group
   `opaque.models`.
4. **CPU dynamic lighting of sprites at draw time.** VERIFIED (`hw_sprites.cpp:315-323`, `hw_spritelight.cpp:101-210`):
   for every non-model sprite and particle (`gl_light_sprites`/`gl_light_particles` default true,
   hw_dynlightdata.cpp:38-39), and also every voxel and `MDL_NOPERPIXELLIGHTING` model (they leave `dynlightindex` -1
   in `PutSprite`, hw_sprites.cpp:843-851) [xc: added], in a level with dynamic lights: light-probe 3x3 search + walk of the whole section light
   map + per-light distance + optional CPU ray test (`ShadowTest`) for shadow-casting lights. Cost S × L per eye on the
   render thread. BD22/RSQ muzzle flashes and effect lights raise L. Settle: none of the clocks isolates it (it is
   inside `RenderSprite`); A/B with `gl_light_sprites 0` / `gl_light_particles 0` and compare `RenderSprite`.
5. **Skeletal models: CPU bones per draw, no cache, shared scratch palette, avatar IK per draw.** VERIFIED for bones
   (`models.cpp:4146`, `models_iqm.cpp:823-965`); INFERRED "no frame cache" for `VRAvatar_Pose`
   (`vr_avatar_pose.cpp:1422+`; grep for `FrameCount|lastFrame|cache` in that file finds nothing). In-hand BD22 weapon
   meshes (psprite models) evaluate bones per psprite per eye; other players' weapons again via
   `DrawWorldPlayerSprites`; IQM monsters per draw; translucent models are additionally duplicated by the translucent
   sort when a translucent flat is present (`hw_drawlist.cpp:333`) [xc: only translucent models are in that list; solid
   models go to `GLDL_MODELS` and are never split]. Within one draw, attachment/decoupled models evaluate the palette
   once for all sub-models (`evaluatedSingle`, models.cpp:4144-4191); other multi-model stacks evaluate per sub-model.
   The only per-frame cache in the path is the joint-drive `FPoseCache` (edits, not palettes; inert by default,
   model_jointdrive.cpp:166-176) [xc: added]. Bone buffer overflow at 80,000 bones/frame silently draws
   unskinned. Settle: `stat bufferstats` "bonebuffer=" (`bonebuffer_curindex`) at the end of a frame;
   `RenderSprite` with models on/off; GPU groups `scene.psprites`, `scene.worldpsprites`, renderer `opaque.models`.
6. **Translucent sort on the render thread.** VERIFIED (`hw_drawlist.cpp:931-948`): all sprites + translucent
   walls/flats, recursive plane/wall splits, `stable_sort` of sprite runs. O(n log n) for sprite-only lists, O(n·k)
   with k translucent planes/walls. Not separately clocked on main. Settle: renderer branch Tracy zone
   `SortTranslucent` (`REMA_TRACY` build) or wrap `Sort` in a new glcycle_t.
7. **Render-pass breaks from soft particles and 2D.** VERIFIED: soft GPU particles (default on) end and restart the
   scene render pass twice per scene (`hw_drawinfo.cpp:3928-3933, 3989-3994`; `vk_renderstate.cpp:624-640`), main
   (non-portal) scene only and only once the level has written a GPU particle or has debris billboards
   (`hw_drawinfo.cpp:3890-3891`), and only when the target accepts the read-only depth pass (`SetSceneDepthReadable`
   returns early with no pass break if the state does not change, `vk_renderstate.cpp:624-625`) [xc: gate added]; each
   `Draw2D` call clears stencil → `EndRenderPass` (`hw_draw2d.cpp:86`, `vk_renderstate.cpp:180`), up to 2 calls per eye
   in VR, each only when that 2D pass has commands and no screen layer is in use (`vk_renderdevice.cpp:671,697-708`)
   [xc: not unconditionally 2]. GPU cost on a tiler-like path is load/store of the attachments; on a 3080 Ti
   INFERRED small but per eye. Settle: GPU group `fx.depthread` (r_perflog) and `twoD`/`VREyeComposite`.
8. **VM calls and name lookups in the draw.** VERIFIED: `AnimateBones` per decoupled model draw
   (`hw_sprites.cpp:501-507`); `ModifyBobLayer*` per psprite per eye (`hw_weapon.cpp:2412,2518,2531`); placement and
   fade cvars looked up by formatted name per model draw (`models.cpp:662-722, 2445-2512, 3117-3147`); `GetCVar` by
   name per sprite with `VisibleCVar`/`AlphaCVar` (`hw_sprites.cpp:1094-1108`). Cost per call small; count = models
   with prefixes × eyes. Settle: Tracy zones around `RenderModel` (not present today).
9. **Per-frame full-level touches.** VERIFIED: `P_FindParticleSubsectors` clears every subsector's `sprites` array and
   fills `ParticlesInSubsec` for every subsector each frame (`p_effect.cpp:207-225`), independent of visibility.
   O(numsubsectors) ~ cheap per element but a cache sweep of the whole subsector array.
10. **Playsim cost of keeping the render index.** VERIFIED: each `LinkToWorld` builds the render sector list with a
    blockmap line iteration (`p_maputl.cpp:471`, `p_secnodes.cpp:216-283`) and, near portals, the portal lists. Paid per
    moving actor per tic, by the sim. `UpdateRenderSectorList` also runs on every relink regardless of movement because
    `OldRenderPos` is never updated after spawn (`p_secnodes.cpp:378`, `p_mobj.cpp:5716`); it is cheap unless the map
    has portals [xc: added]. Gore-heavy mods (BD22 `A_SpawnItemEx` blood/gibs) multiply actor count; the
    previously measured sim tic is ~46 ms with 37,000 thinkers under the owner's list (HANDOFF_RENDER_2026-11-11.md
    "Where things are" paragraph); how much of that is relinking is not separated. Settle: renderer `ticprofile`.
11. **Smaller items.** `IsDistanceCulled` uses the global `r_viewpoint.Pos`, not the portal view's viewpoint
    (`hw_sprites.cpp:1050`) — wrong distance inside portals/mirrors (VERIFIED code; visual effect INFERRED).
    `DrawPSprite` flat-layer `FindState`/`ValidateTexture` waste (`hw_weapon.cpp:256-267`). `SetupSprite` double-counts
    the particle path (nested Clock on a non-reentrant counter, `stats.h:227-237`, `hw_bsp.cpp:374-377,1008,1054`).

### Mod-driven costs visible in engine paths (INFERRED from engine code; mod source not in this repo)

- **BD22 gore** (`A_SpawnItemEx` blood, gibs, decals): each spawned actor is a thinker + a `touching_renderthings`
  entry; non-shootable decor is culled beyond 1200 units (`hw_bsp.cpp:64-80`), everything beyond 2000
  (`hw_sprites.cpp:1041,1067`), inside that every one in a reached sector costs one `Process` + one sorted draw +
  CPU light sum. Flat sprites (`RF_FLATSPRITE`) take the matrix path and depth bias. Decals: one draw each per visible
  wall part; `ProcessDecal` re-runs per eye.
- **BD22 weapons in VR**: psprite models → `RenderHUDModel` per layer per eye with CPU bones, placement cvar lookups,
  `HudAnchor_Store`; flash overlays as flat psprites (`r_hudflatoverlay`).
- **LegenDoom / TopHPBars**: previously measured 1,850 `RSQTHP_Bar` instances (HANDOFF_RENDER_2026-11-11.md, rr122).
  Whether they draw as actor sprites, VisualThinkers or billboards is a mod question; as actor sprites they would each
  be a sorted draw with a CPU light sum.
- **Voxel decoration packs**: `r_voxels_mode` auto resolves to 1 when a pack is loaded (`models.cpp:5389-5394`): only
  held (`VoxelOverride`) actors draw as voxels; with mode 0 every tagged decoration becomes a model draw.
  `r_voxeldistance` (default 0, off) is the LOD switch.
- **Decoupled-animation IQM monsters**: VM `AnimateBones` + CPU bones per draw per eye.

## 6. Hooks, cvars and dead code in this area

| Item | Status | Evidence |
|---|---|---|
| Coronas: `DrawCoronas`, `DrawCorona`, `gl_coronas`, `Coronas` | DEAD | `DrawCoronas` loop is `#if 0` (hw_drawinfo.cpp:4300-4327); its only call is inside `/* */` (:4356-4359); Process corona push is `#if 0` (hw_sprites.cpp:1079-1085); `Coronas` member commented out (hw_drawinfo.h:186); `gl_coronas` (hw_drawinfo.cpp:82) has no other reader (grep). |
| `gl_weapon_purelightlevel` | DEAD (cvar never read) | Defined hw_weapon.cpp:141; `GetWeaponLighting` callers pass 5 args, default `false` (:2314, :2459, :2611); grep finds no other reference. |
| `HWSprite::SplitSprite`, `SortSpriteIntoPlane`/`SortWallIntoPlane` CPU fallbacks | unreachable on Vulkan | Gated on `RFL_NO_CLIP_PLANES` (hw_sprites.cpp:1968, hw_drawlist.cpp:275,341); Vulkan hwcaps lack it (vk_renderdevice.cpp:510). Reachable only on GL/GLES. |
| Dither targets (`r_dithertransparency`, `RenderedTargets`, `P_CheckSight` loop) | DORMANT (cvar off, CHEAT) and logically dead in single-thread mode | Default false (r_utility.cpp:160). In single-thread, `RenderThings` stamps every thing of the sector's list with `validcount` (hw_bsp.cpp:923-924) right before the dither loop skips stamped things (:1091) on the same list → never collects. In multithread it races with the worker. |
| `gl_spriteclip` > 0 (`PerformSpriteClipAdjustment`) | DORMANT | Default -1 (hw_sprites.cpp:111) selects anamorphosis instead. |
| `gl_sprite_blend` | DORMANT | Default false (hw_sprites.cpp:110). |
| `gl_billboard_mode` 1 (XY billboard) | DORMANT | Default 0 (hw_sprites.cpp:117); per-actor `RF_FORCEXYBILLBOARD` still reachable. |
| Sprite shadows (`r_actorspriteshadow`) | DORMANT | Default 0 (r_utility.cpp:167); when on, a second `Process` per actor (hw_bsp.cpp:952-959). |
| `r_PlayerSprites3DMode` crossed/fat-item psprites | DORMANT (default BACK_ONLY) | r_utility.cpp:153, hw_weapon.cpp:130-136, 270-361; pre-work at :256-267 still runs. |
| `gl_light_weapons` | DORMANT | Default false (hw_dynlightdata.cpp:40); gates weapon model light gather (hw_weapon.cpp:2546). |
| `gl_light_model_dedupe_cache` | DORMANT | Default false (hw_dynlightdata.cpp:42); `std::find` path used. |
| `r_voxeldistance` LOD | DORMANT | Default 0 (r_utility.cpp:117). |
| `vr_scene_multithread`, `vr_openxr_multiview` | DORMANT (off) | hw_cvars.cpp:278, hw_vrmodes.cpp:643. |
| `vr_voxel_debug` | LIVE (default **true**, archived) | models.cpp:600; `Printf` for `VoxelOverride` (held) actors only, in `ObjectToWorldMatrix` (:1778-1803) and in `FindModelFrame` (:5609-5625). Throttle is one static last-actor/last-tic pair per site: 1/s for a single held voxel, but with two or more held voxel actors drawn alternately (two hands) `actor != lastActor` is true on every call and it prints on every draw, per eye [xc: "1/s per actor" was wrong]. |
| `vr_pose_debug`, `vr_surf_debug`, `vr_place_debug`, `r_visualstate_log`, `r_gpuparticles_debug` | DORMANT debug prints | models.cpp:275,285,608; `r_visualstate_log` defined scripting/vmthunks.cpp:4794 (default false) [xc: definition is not in hw_weapon.cpp]; hw_cvars.cpp:318. |
| `EndDrawScene`/`DrawEndScene2D` psprite draws | flat-only | Gated by `!RenderPlayerSpritesInScene()` (hw_drawinfo.cpp:4364,4400); `VKOpenXRDeviceMode` returns true (vk_openxrdevice.h:209); the base `VRMode` returns `vr_render_weap_in_scene` (hw_vrmodes.cpp:1476-1479). |
| `DrawWorldPlayerSprites` | LIVE (VR and flat) | `vr_weapons_world` default true (models.cpp:2674). |
| `AnimateBones` VM hook | LIVE for `MF9_DECOUPLEDANIMATIONS` actors | hw_sprites.cpp:501-507. |
| `ModifyBobLayer`, `ModifyBobLayer3D`, `ModifyBobPivotLayer3D` VM hooks | LIVE when a weapon class overrides them | hw_weapon.cpp:2395-2416, 2491-2534. |
| `ModelJointDrive_Apply`, `FModelDrawPoseScope` | DORMANT unless a script registers a drive or `r_jointpose_test` | models.cpp:746, 4168, 4221; gate is inline in model_jointdrive.h:46 (scope opens only if `ModelDrawPose_RegisteredCount > 0 || ModelDrawPose_TestOn`) and :79 (Apply returns the palette untouched under the same test) [xc: gate now read in code, was comment-only]. |
| `VRAvatar_Pose` | LIVE for `VRRigRole == 1` actors | models.cpp:4173-4174, 4223-4224. |
| Renderer-branch `r_snapshot`, `FSnapActorRec`, CSR thing index | DORMANT (renderer only, no reader) | uzdxrema r_snapshot.cpp:28; only callers are writers in d_net.cpp:2493,2557. |
| Selaco `DefinedParticlePool` | selaco only, LIVE there | selaco hw_bsp.cpp `RenderDefinedParticles` (diff vs main). |

## 7. Branch differences

`git -C /home/claude/uzdxrema diff origin/main origin/<branch> -- <file>` (VERIFIED by reading the diffs).

| Area | renderer (175063a) | netplay (1c1805f8) | selaco (7e111a3d) |
|---|---|---|---|
| `hw_sprites.cpp` (Process, DrawSprite, particles) | identical | identical | different: adds `ProcessDefinedParticle`, `selfLighting` boosts (`gl_selflighting`); lacks effect-light mode, `PostMask`, `ModelTint`, `Affect` tilt/drop |
| `hw_bsp.cpp` RenderThings/RenderParticles | identical in this area; adds static level-mesh skips for walls/flats and Tracy zones; also the stage-2b wall-cache replay (`WallCache_Process`, `r_wallcache`, off by default) on the wall paths [xc: added] | identical | adds `ParticlePoolJob` + `RenderDefinedParticles` per subsector (a third CPU particle kind, one HWSprite each) |
| `hw_drawlist.cpp` sort | identical + Tracy zones (`SortTranslucent`) | identical | 1-line diff |
| `hw_drawinfo.cpp` scene | adds GPU groups `opaque.walls/flats/masked/models/decals` inside `scene.opaque`, Tracy zones, `mLevelMeshScene` | no `DrawWorldPlayerSprites` call | ~1000 fewer lines (fork VR/effects features absent) |
| `hw_weapon.cpp` | identical | no `DrawWorldPlayerSprites`; **the draw writes sim state**: `DrawLaserSightWorld` stores `LaserTraceTarget*/LaserTraceHitPos*` into the player actor from the render trace (netplay diff) — main moved that to the sim (`hw_weapon.cpp:1458-1460` comment) | lacks world psprites and several VR paths |
| `models.cpp` | lacks main's newest gun-fit (`VRGunFit_*`, `frameLocal`) changes (-156 lines); draw structure identical | -515 lines: no `ModelLookDownFade`, fewer placement lookups | large diff: no `VRAvatar_Pose`, `VRRig_ObjectToWorld`, `ModelJointDrive_Apply` |
| `perf_track` sprite column | `CPU_Sprites = SetupSprite + RenderSprite` (renderer hw_perftrack.cpp:543) | CPU sprite-time column absent; perf_track itself exists (same file as main) and records the `rendered_sprites` count (main hw_perftrack.cpp:562) [xc: "absent" applied to the column only] | absent (no hw_perftrack.cpp) |
| `r_snapshot` (actor records, CSR index) | present, dormant | absent | absent |
| Key defaults (`vr_openxr_multiview` 0, `vr_scene_multithread` 0, `r_gpuparticles_soft` 8, `gl_sprite_distance_cull` 2000, `r_maxparticles` 4000) | same | same | same |

## 8. Finding actors without the BSP walk (for a Helion-style renderer)

What the walk contributes to sprites today (VERIFIED): only the **set of reached sectors** (sector-level, no per-sprite
frustum or occlusion test, hw_sprites.cpp:1057-1988), reached subsectors for particles, the per-scene `validcount`
dedup, `CurrentMapSections`, `mClipPortal` clipping in portal views, and `sectorportal_thinglist` /
`lineportal_thinglist` handling. Everything else in `Process` is per-actor work independent of the walk.

Candidate enumerators and what each misses:

| Enumerator | Contents | Misses vs today's draw set | Cost to enumerate |
|---|---|---|---|
| `touching_renderthings` of "visible" sectors | exactly what `RenderThings` reads from its first list | nothing for ordinary actors (it IS that set); actors seen through sector/line portals come from `sectorportal_thinglist` / `lineportal_thinglist`, which must be enumerated as well [xc: portal lists are separate] — but needs a sector visibility source (stage-3 block culling, portal flood, or PVS) and a new dedup stamp (multi-sector actors appear in several lists) | O(visible sectors + entries) |
| `sector_t::thinglist` of visible sectors | actors by own sector | client-side actors (`OF_ClientSide`, p_maputl.cpp:443); wide sprites whose center sector is not visible but whose render box reaches a visible one | O(visible sectors + actors) |
| Blockmap `blocklinks` over a view box | gameplay-radius links | `MF_NOBLOCKMAP` actors (typical for decorations, puffs, blood/gore effects), client-side actors (p_maputl.cpp:481), sprites wider than their gameplay radius (links use `radius`, not `RenderRadius`) | O(blocks in box + nodes); multi-block duplicates need dedup |
| Thinker lists (`TThinkerIterator<AActor>`) | everything | nothing; but no spatial order: O(all actors) per eye. Previously measured actor counts 17,163-17,481 sector-linked actors and 37,000 thinkers (HANDOFF_RENDER_2026-11-11.md, rr114/rr120 and summary) make this the expensive option |
| `Level->Particles` + `ParticlesInSubsec`, `subsector_t::sprites` | CPU particles, VisualThinkers | nothing, but they are keyed by subsector: need subsector visibility or a block grid | O(subsectors) rebuild per frame today |
| `Level->Billboards` | fork UI billboards | n/a (already walk-free) | O(N) |
| renderer-branch `FWorldSnapshot.ThingIndex` (CSR) | per-sector image of `touching_renderthings` + `sectorportal_thinglist`, `FSnapActorRec` per actor | same as `touching_renderthings`; records lack `modelData` (`HasModelData` flags them back to the old path, r_snapshot.h:59) | O(actors) write per tic (previously measured 5.5 ms for 17,481 records, HANDOFF_RENDER_2026-11-11.md rr120) |

Helion, for comparison (VERIFIED in `/home/claude/ext/Helion`):
- A separate **render blockmap** with 512-unit blocks (`WorldBase.cs:436`); every entity links into exactly one block
  by its position (`BlockMap.cs:240-257`), regardless of `NoBlockmap` (`PhysicsManager.cs:130-135`). The link is
  incremental, done in `LinkToWorld` on every move, not a per-tic rebuild [xc: added].
- The renderer iterates blocks in a box of `MaxDistance` around the viewer, rejects blocks behind the camera
  (`LegacyWorldRenderer.cs:145-178, 198-215`), and per entity rejects invisible/`NoSector`/behind/far
  (`:271-293`). The behind-camera rejects apply only when |pitch| ≤ 45°, from a point pushed 32 units back
  (`SetOccludePosition`, `:773-787`) [xc: added].
- This block iteration is used only in Helion's static render mode; when `UseBspBasedOnHeuristic` picks BSP mode
  (`:306-307`), `TraverseBsp` finds entities by sector (`LegacyWorldRenderer.Bsp.cs:56-69`) [xc: added].
- The entity list is rebuilt only when the game tick changes (`NeedsRenderTickChange`, `:143, 331-339`); each entity is
  one vertex holding `Pos` and `PrevPos` (`EntityRenderer.cs:289-314`) and the shader interpolates with `TimeFrac`
  (`:367`). Entities are bucketed by render style and array texture (`:279`); translucency uses an OIT framebuffer
  pass, not a sort (`:426-438`, `m_oitFrameBuffer` in LegacyWorldRenderer.cs:322).

Consequences for REMA (INFERRED from the above): (a) the cheapest drop-in is to keep `touching_renderthings` as the
actor index and replace "sector reached by the walk" with the new path's sector/block visibility; a single-block
render grid like Helion's would miss today's wide-sprite reach (`renderradius`) unless an actor links into every block
of its render box. (b) The per-frame work that makes sprites expensive here (CPU light sums, matrices, sort, per-sprite
Apply) is what Helion removes by tic-rate lists + shader interpolation + OIT; REMA's VR needs the interpolation at
display rate, which Helion already does in the vertex shader from Pos/PrevPos. Helion's tic-rate list is built with
the view direction of the frame it was built in (behind-camera block and entity rejects, `LegacyWorldRenderer.cs:160,
276-283`) and is not rebuilt when only the view turns (`NeedsRenderTickChange`, `:143`); with head rotation at display
rate inside a tic that list would lack entities that turned into view, so a REMA port must drop or widen that reject
[xc: added, INFERRED]. (c) Models, the avatar, psprite weapons
and anything with `modelData`, VM hooks (`AnimateBones`, `ModifyBobLayer*`) or VR rig placement cannot be expressed as
such records today and would stay on an old path (the renderer handoff's plan says the same; that is a lead, the code
reasons are §5 items 5 and 8).

## 9. Open questions and the measurements that settle them

1. Is multiview on in the owner's config? Read the `bench`/perflog header "XR: multiview=… active=…"
   (`hw_clock.cpp:325`). If off, A/B `vr_openxr_multiview 1` with `stat rendertimes` ("S: Render/Setup",
   "VR: Things", "VR: PlayerSprites") in a gore-heavy BD22 fight.
2. How many sprites/particles/decals per eye in real play? `stat renderstats` "Sprites: N, Decals=N" (counters reset
   per frame in `ResetProfilingData`, hw_clock.cpp:138). Note `rendered_sprites` counts Process calls that reached
   `PutSprite`, per eye.
3. Draw-time split: `RenderSprite` (DrawSprite incl. models, CPU light sums, vertex build) vs `drawcalls` (all Apply,
   not just sprites) vs `SetupSprite` (scene build; particle path double-counted). No clock covers
   `DrawPlayerSprites`, `DrawWorldPlayerSprites`, `DrawLaserSightWorld`, `VRWheel_Draw`, or the translucent `Sort`:
   add glcycle_t around each (or use the renderer branch's Tracy build) before ranking them.
4. CPU light cost: A/B `gl_light_sprites 0`, `gl_light_particles 0`, and the light-shadow cast setting, compare
   `RenderSprite`; `stat lightstats` "ModelGather" for model gathers.
5. Bone pressure: `stat bufferstats` "bonebuffer=" at peak; any "We have run out of BUFFERS!" developer message means
   unskinned draws.
6. GPU side: r_perflog groups `scene.translucent`, `fx.gpuparticles`, `fx.depthread`, `fx.meshparticles`,
   `scene.psprites`, `scene.worldpsprites`, `scene.vrextras` (hw_drawinfo.cpp:4499-4587); renderer branch adds
   `opaque.models`, `opaque.decals`. Previously measured GPU opaque 6.8 ms flat (HANDOFF_RENDER_2026-11-11.md summary)
   says nothing about the translucent pass in VR.
7. Does `VRAvatar_Pose` really recompute the whole IK per draw (per eye, per mirror)? Read vr_avatar_pose.cpp fully
   (3,470 lines; only its entry was read here) or time it with a zone.
8. The GPU particle ring and the bone buffer are each a single persistently mapped buffer written by the CPU while
   earlier frames may still read them (hw_gpuparticlebuffer.cpp:63-64,186; hw_bonebuffer.cpp:47-50 with one pipeline
   copy on Vulkan). Whether the Vulkan backend waits for the previous frame's fence before these writes was not traced
   here; a flicker under heavy bursts would point at it.
9. Which mod actors dominate: the renderer branch's `render_profile_300.cfg` per-class sim profile exists; a
   per-class **draw** count (Process calls by class) does not — add a counter keyed by `GetClass()` in `PutSprite` to
   learn whether gore, HP bars or champion effects are the sprite load.

## Cross-check (date 2026-11-13)

Checked against main f182cbcc, renderer tree abb19af1 (175063a + docs), netplay 1c1805f8, selaco 7e111a3d, Helion
5edd437, by opening each cited line. Dead/never-read greps re-run over all of `src/` and `wadsrc/` (menudef, ZScript,
shaders included).

| Category | Checked | Right | Line-fixed | Corrected | Unconfirmed | Added |
|---|---|---|---|---|---|---|
| a. Summary items | 10 | 8 | 0 | 2 | 0 | 0 |
| b. "What slows it down" (11 ranked + 5 mod-driven) | 16 | 13 | 0 | 3 | 0 | 3 |
| c. DEAD / DORMANT / never-read | 19 | 16 | 0 | 3 | 0 | 0 |
| d. cvar defaults | 30 | 30 | 0 | 0 | 0 | 0 |
| e. branch differences | 14 | 13 | 0 | 1 | 0 | 1 |
| f. call graph, table rows, Helion bullets | 75 | 64 | 2 | 6 | 0 | 9 |

Ranking in section 5 was not changed; it is a judgement, and none of the checks contradicted it.

Corrections:
- Summary 4 / §4 drawlist row / §5.3: solid non-model wall sprites (and solid models) go to `GLDL_MODELS`, not translucent (hw_drawlistadd.cpp:140).
- Summary 9 / §8 table: `touching_renderthings` is the set of `RenderThings`' first list only; `sectorportal_thinglist` and `lineportal_thinglist` actors are separate lists.
- Summary 9: Helion's render blockmap is linked incrementally in `LinkToWorld` (PhysicsManager.cs:135), not rebuilt per tic; the entity draw list is the per-tic part.
- §4 portal-list row: `UpdateRenderSectorList` rebuilds both portal lists on every relink; the `OldRenderPos` gate never fires (never updated after spawn, p_mobj.cpp:5716).
- §5.5: only translucent models are duplicated by the translucent sort.
- §5.7 / §3.2 Draw2D row: Draw2D runs up to twice per eye, each gated on `HasCommandsForPass` and no screen layer; soft-particle pass breaks are gated on non-portal scene, GPU particles written / debris billboards, and an actual state change.
- §3.2 IQM `RenderFrame`: its `UploadBones(boneData)` is the only upload on the plain unposed path, not a second one.
- §2.1: `mBones/mLights->Clear` sit in `RenderView` before canvases and camera textures, which share the capacity.
- §6 `vr_voxel_debug`: throttle is a single static last-actor per site; two held voxels alternating print every draw.
- §6 debug cvars: `r_visualstate_log` is defined in scripting/vmthunks.cpp:4794, not hw_weapon.cpp.
- §6 joint drive: the gate is now read in code (model_jointdrive.h:46,79) instead of taken from comments.
- §7 perf_track: netplay has perf_track and a sprite count; only the CPU sprite-time column is absent.
- Line fixes: §2.2 particle gate `:1045-1057`; Helion `WorldBase.cs:436`.

Additions: `hw_GetDynModelLight` re-walks a section's light map per subsector (section stamp never set); voxel and
NOPERPIXELLIGHTING models also take the CPU sprite light sum; joint-drive `FPoseCache` is the only per-frame cache in the
bone path (edits, not palettes); `evaluatedSingle` shares one palette across attachment/decoupled sub-models;
world-psprite path draws the local player's own layers in mirror/camera/portal scenes; GPU-ring draw gate; client-side
actors build only the render sector list; Helion behind-camera rejects only at |pitch| ≤ 45° and its BSP mode; the
tic-rate list built with one frame's view direction would miss entities when the head turns inside a tic (INFERRED);
renderer branch also carries the `r_wallcache` replay path.
