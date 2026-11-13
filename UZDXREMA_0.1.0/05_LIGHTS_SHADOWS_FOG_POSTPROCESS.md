# 05 — Lighting, shadows, fog and the post-process chain

Teardown of UZDXREMA, area 05. Primary tree: `main` at `f182cbcc` (`/home/claude/uzdx_main`). Paths below are
relative to the tree root; `file:line` refers to `main` unless another branch is named. Every claim is marked
**V** (VERIFIED: the code that does it was read) or **I** (INFERRED: deduced, with the basis given). No timings
here are measured by this teardown; the few quoted numbers are marked "previously measured" with their source.

---

## 1. Summary

1. Dynamic lights (`FDynamicLight`) are a render-only linked list ticked once per tic after an O(all actors) scan;
   any light whose X/Y or max radius changed **relinks**: unlink from every side/section hash map, then a 2D flood
   fill over sections that heap-allocates one `FLightNode` per touched section and side (V, `a_dynlight.cpp:377-727`).
2. Per-surface light lists are built **at draw time, per eye, on the main thread** (Vulkan reports persistent buffers),
   by iterating the side/section `TMap`, filtering, insertion-sorting to a budget (8 walls / 16 flats) and appending
   4 vec4 per light into one 80,000-light SSBO; the shader gets a per-draw index in a push constant (V).
3. Sprites and particles get one CPU-summed RGB per draw (with a CPU AABB ray test per light when shadow-mapped);
   models get a full light list via a BSP circle walk at BSP time (V, `hw_spritelight.cpp:101-321`).
4. Shadows: a 1D shadow map (1024 rows × `gl_shadowmap_quality`=128 R32F), one full-target fragment pass that ray-walks
   a 2D AABB tree of one-sided lines per texel, run once per frame when wanted; optional ray-query shadows decided at
   start-up (10 rays per light per pixel at the default filter) (V).
5. Defaults: `gl_light_shadowmap 0`, `gl_light_castshadows 0`, so the pass runs only for effect lights
   (`r_effectlights 1`, `r_effectlights_walls 1`) and lingers 10 s after the last one (V).
6. The fork added a **second, already binned light system** ("effect lights": world-space 3D grid of bins around the eye,
   CPU-built once per frame, ≤32 lights per pixel loop) — the natural template for a clustered replacement (V).
7. The fork's sector-light additions (wall/flat glow, sweep bands, darkness, beams, stamps, fog slab, fog dyn-lights,
   standing shapes) are all per-pixel code in a 5,128-line `main.fp`, uniform-gated; their per-draw state rides in a
   2,000-byte `StreamData` record that is copied for **every** draw (I/V).
8. Post-process per eye (no layered PP; VR = 2×): exposure (mip chain to 1×1) + bloom (22 passes at ¼…1/32 res) by
   default; SSAO, FXAA, lens, tonemap off by default; fork passes (smoke, emissive volumes, volumetric beams, heat,
   shockwave, exposure impulse, light mask) run only when their content exists (V).
9. Ranked costs with heavy light mods: per-tic relinks of moving lights; per-eye draw-time `SetupLights`; per-eye sprite
   and model light gathering; StreamData copies; fragment light loops ×2 eyes; bloom/exposure pass count (I, §5).
10. On the `renderer` branch every wall/flat a dynamic light touches is **evicted** from the static mesh back to the old
   path, so dynamic lights are the main blocker of the static path (V, `hw_levelresidency.cpp:3835-3873, 4357-4362`).

---

## 2. Entry points and call graph

### 2.1 Per tic (sim thread = main thread)

```
TryRunTics -> ... P_Ticker / client tick
  p_tick.cpp:199  level->ClientSideThinkers.RunClientSideThinkers(level, worldStep)
    dthinker.cpp:293  FThinkerCollection::RunClientSideThinkers
      dthinker.cpp:303-311   dolights = r_dynlights (the Level->lights test is commented out)
      dthinker.cpp:384       recreateLights(): iterate ALL actors + ALL client-side actors
                             -> RecreateDynamicLights (dthinker.cpp:279) -> AActor::SetDynamicLights (a_dynlight.cpp:762)
                                  -> AActor::AttachLight(count, def) (a_dynlight.cpp:735)
                                       -> GetLight (arena/free list, a_dynlight.cpp:62) for new slots
                                       -> FLightDefaults::ApplyProperties (a_dynlightdata.cpp:114)
                                       -> FDynamicLight::UpdateLocation (a_dynlight.cpp:377)
      dthinker.cpp:385-410   for every light in Level->lights: FDynamicLight::Tick (a_dynlight.cpp:253)
                               -> UpdateLocation -> ResolvePoseAnchor (a_dynlight.cpp:995) | target offset/bob/clamp
                               -> if X,Y or radius changed: LinkLight (a_dynlight.cpp:671)
                                    -> UnlinkLight (a_dynlight.cpp:695)   TMap::Remove per touched side/section
                                    -> PointInRenderSubsector
                                    -> CollectWithinRadius (a_dynlight.cpp:541)  flood fill, DistToSeg per seg/side,
                                         line portals, plane portals -> AddLightNode (a_dynlight.cpp:455)
                                         (new FLightNode + TMap::TryEmplace + touchlist SortedAddUnique)
  p_mobj.cpp:1099-1103 (SetState) / :6096-6099 (PostBeginPlay): set MF8_RECREATELIGHTS when the class has
  GLDEFS frame lights or the state has a light -> consumed by the scan above next tic.
  A_AttachLight / A_AttachLightDef / A_RemoveLight (a_dynlight.cpp:841-953) also set MF8_RECREATELIGHTS.
```
While paused, `P_Ticker` still runs the recreate scan (`p_tick.cpp:662-679`, V).

### 2.2 Per displayed frame, once (main view)

```
D_Render (d_main.cpp:690-699): Level->HasDynamicLights = r_dynlights && (lights || LightProbes)
RenderView (hw_entrypoint.cpp:640)
  :664  iter/draw counters reset; :669 ResetProfilingData; :675 screen->mLights->Clear()   (light SSBO cursor -> 0)
  RenderViewpoint (hw_entrypoint.cpp:264)
    :273  EmissiveVolumes::BeginFrame        (hands its lights to EffectLights)
    :285  BeamLights_HandOver                (beam slot lights -> EffectLights::AddFrameLight)
    :292  EffectLights::BeginFrame (hw_effectlights.cpp)  drain queue, evaluate pool, rank/cap, row order -> wantsRows
    :299  DynamicLightShadowRowsWanted (hw_dynlightdata.cpp:210)  O(lights) walk unless setting is Off
    :301-316  if wanted: SetAABBTree, SetCollectLights(CollectLights), screen->UpdateShadowMap()
              VulkanRenderDevice::UpdateShadowMap -> VkPostprocess::UpdateShadowMap (vk_postprocess.cpp:586)
                IShadowMap::PerformUpdate (hw_shadowmap.cpp:162): UploadAABBTree (:194), UploadLights (:182)
                  -> CollectLights (hw_entrypoint.cpp:78): 1-2 walks of Level->lights, EffectLights::AssignShadowRows
                PPShadowMap::Update (hw_postprocess.cpp:2840): one draw, viewport quality x 1024, shadowmap.fp
    :322  FLightDefaults::SetAttenuationForLevel
    :327  vrmode->SetUp(); :330 R_UpdatePoseAnchoredLights (hw_entrypoint.cpp:185): relink if moved > 4 units
    :337  PrepareFrameCompute -> EffectLights::PrepareFrame (bin + upload), SmokeVolume::PrepareFrame
          (-> SmokeVolume::GatherLights hw_smokevolume.cpp:1349, O(lights)), ... ; screen->RunFrameCompute
    :341  lightmask.SetPinnedLightLive; :345 ExposureImpulses::BeginFrame
    eye loop :359 (2 iterations in stereo VR) -> §2.3
```

### 2.3 Per eye (scene build and draw; VR without multiview = twice)

```
hw_entrypoint.cpp:361  ++gl_dynlight_viewid (keys the optional per-view light caches)
HWDrawInfo::StartDrawInfo -> StartScene (hw_drawinfo.cpp:2066) -> FillFogDynLights (:1783, only with fog slab, O(lights))
di->ProcessScene (hw_drawinfo.cpp:4597) -> SyncViewLights (:671, O(lights), only while lit GPU particles live)
  RenderBSP (hw_bsp.cpp:1422; multithread off in VR unless vr_scene_multithread, :1438-1439)
    HWWall::PutWall (hw_walls.cpp:827): walls with decals -> SetupLights NOW (:879-884); others deferred
    HWSprite::PutSprite (hw_sprites.cpp:840): per-pixel-lit model -> hw_GetDynModelLight + UploadLights (:843-849)
    render hacks: AddOtherFloorPlane/CeilingPlane -> SetupLightsForOtherPlane (hw_renderhacks.cpp:112, 233-251)
  DrawScene -> draw lists:
    HWDrawList::DrawWalls (hw_drawlist.cpp:840) -> HWWall::DrawWall (hw_walls.cpp:559)
        -> SetupLights (hw_walls.cpp:600) -> AddLightToList (hw_dynlightdata.cpp:112) -> FLightBuffer::UploadLights
        -> RenderTexturedWall (:333): SetFog/SetColor (hw_setcolor.cpp), glow, SetGlowPlanes, effect-light mode
    HWDrawList::DrawFlats -> HWFlat::DrawFlat (hw_flats.cpp:403): flat glow lines (:455-492)
        -> DrawSubsectors (:279) -> SetupLights (:157) -> UploadLights
    HWSprite::DrawSprite (hw_sprites.cpp ~:300-572): if no list index -> GetDynSpriteLight (hw_spritelight.cpp:101/212)
        (+ IShadowMap::ShadowTest -> LevelAABBTree::RayTest on CPU when the light is shadowmapped)
    DrawPlayerSprites (hw_weapon.cpp:2421-2549): GetDynSpriteLight / hw_GetDynModelLight (gl_light_weapons only)
  SSAO (only gl_ssao>0 and GBUFFER_PASS): screen->AmbientOccludeScene (hw_drawinfo.cpp:4552-4556)
PostProcess per eye: screen->PostProcessScene (hw_entrypoint.cpp:501 / :398) -> §2.5
```

### 2.4 Per pixel (scene fragment shader)

```
main.fp main() (:4813) -> SetupMaterial -> ApplySurfaceDamage -> alpha test
  fog factor (:4903-4921)
  getLightColor (:4005): sector light eq / fog diminish, DarknessAt, sweep passed-region grade, sweep recolour,
     wall glow top/bottom (:4159-4227), flat-edge glow loop <=64 segments (:4234-4337), sweep bands <=8 (:4347-4455),
     BeamLightAt <=n beams (:2297), ApplySurfaceStamps, outline, glow tex, brightmap, ProcessLight, lightmap,
     EffectLightsAt (one bin, <=32 lights, :882; gated by uEffectLightMode),
     ProcessMaterialLight (material_normal.fp:85): 3 loops over lights[uLightIndex] ranges,
        lightContribution (material_normal.fp:18) -> spotLightAttenuation, shadowAttenuation (main.fp:581 RT / :748 SM)
  colored fog applyFog; ApplyFadeColor; FogSlabAt (:3238, incl. <=8 fog dyn lights :3726);
  SweepAirLattice; ShapesAt (<=128); StandingShapesAt (<=128); BeamAirGlow (<=128); eye fade / dither
```

### 2.5 Per eye post-process (Vulkan)

```
VulkanRenderDevice::PostProcessScene (vk_renderdevice.cpp:805)
  VkPostprocess::BlitSceneToPostprocess (vk_postprocess.cpp:131): resolve/blit SceneColor -> PipelineImage (+ light mask)
  VkPostprocess::PostProcessScene (vk_postprocess.cpp:96)
    Postprocess::Pass1 (hw_postprocess.cpp:3557)
      exposure.Render (:2286)                 [gl_bloom]
      customShaders.Run("beforebloom")        [mod GLDEFS shaders]
      smokevolume.Render (:659)               [smoke published this eye]
      volbeam.Render (:88)                    [beam count > 0]
      emissivevolumes.Render (:1103)          [volumes]
      heatmap.Render (:160)                   [heat grid]
      heatrefraction.Render (:254)            [heat sources / shockwaves]
      lightmask.RenderDebug | bloom.RenderBloom (:1688) [gl_bloom] -> exposureimpulse.Render (:3485) [live wash]
    afterBloomDrawEndScene2D (DrawEndScene2D)
    Postprocess::Pass2 (:3583): tonemap (:2552), colormap (:2433), lens (:2141), fxaa (:2195),
                                 customShaders.Run("scene"), UpdateLastInputTexture
```
Each pass is one `VkPPRenderState::Draw`: `EndRenderPass` of whatever was open, image transitions, descriptor set
lookup, its own render pass and one full-viewport quad (V, `vk_pprenderstate.cpp:81-118, 146-228`). The output
targets one array layer, the current eye's (`vk_pprenderstate.cpp:111, 158, 171-173`, V); there is no layered PP pass.

---

## 3. Per-function table

Cost classes: L = number of lights in `Level->lights`; Lm = lights that moved/resized this tic; S = sections+sides
touched by a light's flood; Ls = lights in one side/section map; W/F = walls/flats drawn per eye; Sp = sprites and
particles drawn per eye; M = per-pixel-lit models per eye; A = actors (+ client-side actors).

| Function | file:line | Called from | Per | Cost class | Notes |
|---|---|---|---|---|---|
| `RunClientSideThinkers` recreate scan | dthinker.cpp:318-333, 384 | p_tick.cpp:199 | tic | O(A) | Walks every actor of both thinker collections to test `MF8_RECREATELIGHTS`, even with no lights; also runs P_RunEffect (V) |
| `AActor::SetDynamicLights` | a_dynlight.cpp:762 | recreate scan, `RecreateAllAttachedLights` | state change of a light-bearing actor | O(#LightAssociations of the class) + per light `AttachLight` | Linear scan of class associations by sprite/frame (:774-780) (V) |
| `AActor::AttachLight` | a_dynlight.cpp:735 | SetDynamicLights | light slot | O(1) + `UpdateLocation` | New slot: `GetLight` arena/free-list (V) |
| `FLightDefaults::ApplyProperties` | a_dynlightdata.cpp:114 | AttachLight | light slot | O(1) | Pointers into the definition, pose anchor (V) |
| `FDynamicLight::Tick` | a_dynlight.cpp:253 | dthinker.cpp:389-408 | tic × light | O(1) + relink | Flicker uses `FCRandom randLight` (:42); clamps to `gl_light_max_intensity` (V) |
| `FDynamicLight::UpdateLocation` | a_dynlight.cpp:377 | Tick, AttachLight, RelinkDynamicLightsForLevel | tic × light | O(1) or relink | Relinks only on X/Y/radius change (:434); Z moves do not relink (V) |
| `FDynamicLight::LinkLight` | a_dynlight.cpp:671 | UpdateLocation, R_UpdatePoseAnchoredLights | relink | Unlink + flood | Increments `dynlights_links_total` (V) |
| `FDynamicLight::UnlinkLight` | a_dynlight.cpp:695 | LinkLight, DeleteAttachedLights, Tick | relink | O(S) hash removes (each frees a `unique_ptr<FLightNode>`) | (V; free INFERRED from `std::unique_ptr` value type, g_levellocals.h:58-59) |
| `FDynamicLight::CollectWithinRadius` | a_dynlight.cpp:541 | LinkLight | relink | O(sections reached × (segs+sides)) DistToSeg, capped at `gl_light_max_collected_subsectors`=1000 sections | 2D only; sets `shadowmapped` if it saw the back of a one-sided line (:662) (V) |
| `FDynamicLight::AddLightNode` | a_dynlight.cpp:455 | CollectWithinRadius | touched side/section | heap `new FLightNode` + `TMap::TryEmplace` + `SortedAddUnique` | One heap alloc per link (V) |
| `RelinkDynamicLightsForLevel` | g_level.cpp:2981 | level code | event | O(L) relinks | (V) |
| `R_UpdatePoseAnchoredLights` | hw_entrypoint.cpp:185 | RenderViewpoint :330 | frame | O(anchored) + relink if moved > 4 units | Main view only (V) |
| `EffectLights::BeginFrame` | hw_effectlights.cpp (BeginFrame) | RenderViewpoint :292 | frame | O(pool) eval + O(pool log pool) rank when over cap + sort of row candidates | `r_effectlights_max`=1024 cap (V) |
| `DynamicLightShadowRowsWanted` | hw_dynlightdata.cpp:210 | RenderViewpoint :299 | frame | O(L) when cast-shadow setting on; O(1) when Off (default) | 10 s linger (V) |
| `CollectLights` | hw_entrypoint.cpp:78 | IShadowMap::UploadLights | shadow pass frame | O(L) (+ another O(L) and `nth_element` when the asking setting is on) + fill 1024 rows | Non-asking lights get rows in list order (newest first, GetLight pushes at head :73) (V) |
| `IShadowMap::UploadAABBTree` | hw_shadowmap.cpp:194 | PerformUpdate | shadow pass frame | full tree upload on tree change, else `DoomLevelAABBTree::Update` diff of polyobject lines | Tree = one-sided lines only (doom_aabbtree.cpp:70-90) (V) |
| `IShadowMap::UploadLights` | hw_shadowmap.cpp:182 | PerformUpdate | shadow pass frame | 16 KB `SetData(Stream)` | (V) |
| `PPShadowMap::Update` | hw_postprocess.cpp:2840 | VkPostprocess::UpdateShadowMap | shadow pass frame | GPU: quality×1024 fragments, each walks the AABB tree (stack 32) | Empty rows early-out (`radius > 0` test, shadowmap.fp) (V) |
| `FillFogDynLights` | hw_drawinfo.cpp:1783 | StartScene :2433 | draw info (eye, portal) with fog slab | O(L) + nth_element | `fog_dynlights_max`=8 cap; level asks via `Level->FogDynLights` (V) |
| `SyncViewLights` | hw_drawinfo.cpp:671 | ProcessScene :4681 | eye (toscreen) | O(L) × 5-plane frustum test + nth_element | Only while lit GPU particles/mesh particles/debris are alive (:687-692) (V) |
| `SmokeVolume::GatherLights` | hw_smokevolume.cpp:1349 | SmokeVolume::PrepareFrame | frame with smoke | O(L) + nth_element to `SMOKE_LIGHTS_MAX`=32 | (V) |
| `HWWall::SetupLights` | hw_walls.cpp:600 | DrawWall :565 (every eye); PutWall :883 (walls with decals) | wall part × eye | O(Ls) filter + insertion sort to budget 8 + per-candidate 4-vertex touch test + AddLightToList + UploadLights | Called twice for walls with decals (BSP and draw) (V) |
| `HWFlat::SetupLights` | hw_flats.cpp:157 | DrawSubsectors :289 | flat (section) × eye | O(Ls) + insertion sort to 16 + UploadLights | Plane side test via `ZatPoint` (V) |
| `HWDrawInfo::SetupLightsForOtherPlane` | hw_renderhacks.cpp:112 | AddOtherFloor/CeilingPlane :238/:248 | hack subsector × eye | as flats | (V) |
| `GetLight` | hw_dynlightdata.cpp:52 | SetupLightsForOtherPlane (budget 0 path) | light | O(1) | Only reached with `gl_light_flat_candidate_budget 0` (V) |
| `AddLightToList` | hw_dynlightdata.cpp:112 | all list builders | light × surface | 16 floats written; spot trig unless `gl_light_spot_cache` | Shadow index 1025 when the shadow map is off; attenuate in sign bit (V) |
| `gl_IsDistanceCulled` | hw_dynlightdata.h:103 | list builders | light × surface | O(1); called up to twice per light | Uses global `r_viewpoint.Pos`, not the eye (V) |
| `gl_GetLightPosRelative` | hw_dynlightdata.h:58 | list builders | light × surface | O(1) | Cache off by default (V) |
| `gl_InsertBestLightCandidate` | hw_dynlightdata.h:76 | list builders | candidate | O(budget) insertion | (V) |
| `FLightBuffer::UploadLights` | hw_lightbuffer.cpp:81 | list builders | surface × eye | atomic fetch_add + memcpy 16 B + 64 B/light into persistent SSBO | Overflow at 80,000 lights → returns -1 and the surface is drawn unlit (V) |
| `HWDrawInfo::GetDynSpriteLight` | hw_spritelight.cpp:101 / :212 | DrawSprite hw_sprites.cpp:320, DrawPlayerSprites hw_weapon.cpp:2426 | sprite/particle × eye | FindLightProbe (3×3 cells) + O(Ls of the actor's section) + CPU `ShadowTest`→`RayTest` per shadow-mapped light | Spot trig recomputed every call (V) |
| `hw_GetDynModelLight` | hw_spritelight.cpp:228 | PutSprite hw_sprites.cpp:847, hw_weapon.cpp:2548 | model × eye | BSPWalkCircle over subsectors; per subsector its section's whole map; linear dedupe `std::find` | Section revisit guard never armed (see §6) (V) |
| `FindLightProbe` | hw_spritelight.cpp:40 | sprite/model paths | call | 9 cells × probes | Probes are static (maploader.cpp:3384-3433) (V) |
| `SetColor`, `SetFog`, `SetShaderLight`, `CalcLightLevel`, `GetFogDensity` | hw_setcolor.cpp; hw_lighting.cpp:69-208 | every draw | draw | O(1) table lookups | Cheap (V) |
| `CheckFog` | hw_lighting.cpp:221 | wall Process (fog boundaries) | wall | O(1) | Fog boundary drawn as an extra translucent wall with `EFF_FOGBOUNDARY` (hw_walls.cpp:248-263) (V) |
| Flat glow line copy | hw_flats.cpp:455-492 | DrawFlat | flat × eye | ≤64 line copies into StreamData | First 64 lines of the sector only (V) |
| `VkRenderState::Apply` → `ApplyStreamData` | vk_renderstate.cpp:226-246, 379-395 | every draw | draw | memcpy of `sizeof(StreamData)` (2,000 B, I) | Unconditional; 32 records per 64 KB block (vk_shader.h:49) (V/I) |
| `PostProcessScene` chain | vk_postprocess.cpp:96; hw_postprocess.cpp:3557-3593 | hw_entrypoint.cpp:398/501 | eye | ~34 GPU passes by default (I, §4.6) | Each pass its own render pass (V) |

---

## 4. Data structures that matter for speed

### 4.1 Light objects and links
- `FDynamicLight` (a_dynlight.h:250-390), arena-allocated in blocks of 200 (`DynLightArena`, a_dynlight.cpp:40) with a
  free list; linked into `Level->lights` at the head (:73). Holds pointers into actor/definition data (`pArgs`,
  `pLightFlags`, spot angles), so every read of colour/flags is an indirection (V). It also carries render caches
  written from render code: `mDistanceCullViewId/Result`, `mSpotCacheViewId`, `mPosRelativeCache*`,
  `mModelLightGatherId` (:336-347) — used only with the `gl_light_*_cache` cvars on (V).
- `FGlobalDLightLists` (g_levellocals.h:58-59): `TArray<TMap<FDynamicLight*, std::unique_ptr<FLightNode>>>`
  for sections (`flat_dlist`) and sides (`wall_dlist`), indexed by section/side index, grown lazily (V). Readers iterate
  the hash map (`TMap::Iterator`), whose cost follows the map's allocated size, not only its count (I, standard
  open-hash iteration).
- `FDynamicLightTouchLists` (a_dynlight.h:244-248): per light, sorted arrays of touched sections and sides, used to
  unlink (V).

### 4.2 Light buffer (the per-surface lists on the GPU)
- `FLightBuffer` (hw_lightbuffer.h): Vulkan creates exactly one (`new FLightBuffer()`, vk_renderdevice.cpp:610, pipeline
  depth 1), SSBO (`useSSBO`) of `gl_max_lights`=80,000 lights × 4 vec4 = 5.12 MB, persistent mapping (V,
  hw_lightbuffer.cpp:24-60). Cursor reset once per frame in `RenderView` (hw_entrypoint.cpp:675), so both eyes append
  into the same buffer (V).
- One upload = 1 header vec4 `{0, n_mod, n_mod+n_sub, n_mod+n_sub+n_add}` + 4 vec4 per light, in three classes:
  modulated, subtractive, additive (V, hw_lightbuffer.cpp:81-126; class chosen in hw_dynlightdata.cpp:119-149).
- Record: `[pos.xzy, radius] [r,g,b, ±(shadowRow+1)] [spotDir.xyz, type] [cosInner, cosOuter, 0, 0]` (V,
  hw_dynlightdata.cpp:172-188). Shader binding: set 1 binding 3 `vec4 lights[]` (vk_shader.cpp:852-856, V).
- The same light is copied once per surface it lights, per eye (V from the per-surface upload in every SetupLights).
- Per-draw index: push constant `uLightIndex` (vk_renderstate.cpp:440; `PushConstants`, vk_shader.h:56-81, V).

### 4.3 StreamData (per-draw uniform record)
- `StreamData` (hw_renderstate.h:213-385). A byte-for-byte replica of its fields compiled with g++ gives **2,000 bytes**,
  i.e. `MAX_STREAM_DATA = 65536/2000 = 32` records per 64 KB block (I: replica, not the engine's own `sizeof`).
  Of this, `uFlatGlowLines[64]` is 1,024 B, the sweep arrays `uSweepBands/Colors/BandOrigin[8]` 384 B, wall glow
  ~144 B (I, from the field list).
- `VkStreamBufferWriter::Write` memcpys the whole record for every `Apply` (vk_streambuffer.cpp:34-48), and `Apply`
  calls `ApplyStreamData` unconditionally (vk_renderstate.cpp:234, V). When the stream buffer is full,
  `WaitForStreamBuffers` (vk_renderstate.cpp:390-394, V).

### 4.4 Shadow map
- Texture `quality × 1024`, `VK_FORMAT_R32_SFLOAT` (vk_texture.cpp:294-311, V). Row = light, four 2D directions per row
  (hw_shadowmap.cpp:25-52 comment; the shader `shadowmap.fp` main confirms the layout, V).
- `mLights` 1024×4 floats, uploaded as a Stream buffer each pass (hw_shadowmap.cpp:182-191, V).
- AABB tree: `DoomLevelAABBTree` with a static subtree and a dynamic (polyobject) subtree; one-sided lines only
  (doom_aabbtree.cpp:30-105, V). Uploaded as SSBOs bindings 4/5 (shadowmap.fp).
- Ray-traced variant: BLAS/TLAS of `LevelMesh` built when the level mesh changes (vk_raytrace.cpp:51-60, 86-200, V);
  not rebuilt for moving sectors (I: only `SetLevelMesh` triggers `CreateVulkanObjects`).

### 4.5 Effect-light bins (the fork's existing binned light system)
- Records: 4 vec4 (`a` start+radius, `b` end+flags, `color`+shadow row, `extra`) (main.fp:789-795, V).
- Grid header + one 32-bit word per bin `(offset<<6)|count` + index list (main.fp:806-811, hw_effectlightbuffer.h:96-108,
  V). Grid covers 3072 × 768 × 3072 map units (x, up, y) around the eye, eye-centred and moving by whole bins: quality
  1/2/3 = 64/32/24-unit bins, 48×12×48 / 96×24×96 / 128×32×128 bins (`GridForQuality`, `GridAround`,
  hw_effectlightcore.h:440-468; used at hw_effectlights.cpp:467), V.
- Limits: pool 4,096, records 8,192, bins 128×32×128, indices 262,144, per-bin loop 32
  (hw_effectlightbuffer.h:99-108, V). Built on the CPU once per frame by `EffectLightCore::BinBuilder::Build`
  (hw_effectlightcore.h:530-630: per-light bin touch, counting sort, crowded bins merged into a residual glow), V.

### 4.6 Post-process targets (Vulkan)
- Scene: `SceneColor` RGBA16F (vk_renderbuffers.cpp:304-317), `SceneFog` RGBA8 (:344-357), `SceneNormal`
  A2R10G10B10 or RGBA8 (:359-370), depth D24S8/D32S8 (:319-342); optional `SceneLightMask` RG16F/RGBA16F and
  `ScenePostMask` R8 created on first need (:443-580) (V).
- PP: 4 `PipelineImage` RGBA16F (vk_renderbuffers.h:63; .cpp:237-262), layered when multiview (V).
- Bloom: 4 levels, level 0 = ceil(scene/4), each next half, V+H RGBA16F textures (hw_postprocess.cpp:56-80,
  `NumBloomLevels = 4` hw_postprocess.h:2222, V). Pinned-bloom duplicates (`pinLevels`) only when used (:1935-1950).
- Exposure: R32F mip chain from scene/2 to 1×1, plus a 1×1 camera texture blended over time (:2397-2429, :2344-2357, V).
- SSAO: half-res R32F linear depth + 2 RG16F (:2663-2677, V). Smoke: half-res R32F/RGBA16F/RGBA32F set (:451-484, V).
  Heat offset half-res RGBA16F (:238-252, V).

### Pass counts per eye (I, from the loops read)
- Exposure: 1 extract + (levels-1) averages + 1 combine; levels ≈ log2(max(scene)/2)+1 (≈11 for a ~2k eye).
- Bloom (legacy plan): 1 extract + 7 blurred levels × 2 + 3 downscales + 3 upscales + 1 final combine = **22** draws
  (RenderChain hw_postprocess.cpp:1851-1912: blur levels 0,1,2,3,2,1,0).
- Default chain ≈ 34 draws per eye, ≈ 68 per VR frame, each a separate render pass; plus the scene blit and present
  (other area).

---

## 5. What slows it down (ranked)

Ranked for heavy dynamic-light content (BD22-style muzzle flashes and projectile lights, gore/impact lights, environment
mods with many lamps), VR, no multiview (default `vr_openxr_multiview false`, hw_vrmodes.cpp:643).

### CPU

1. **Per-tic relink of every moving light** — confidence high.
   Every light attached to a moving actor (projectiles, flashes following a weapon, gore) changes X/Y every tic and
   relinks (a_dynlight.cpp:434-439, V). A relink = O(S) hash removals + a flood fill doing `DistToSeg` against every seg
   and side of every reached section + one heap allocation and hash insert per touched side/section (a_dynlight.cpp:
   455-495, 541-727, V). Cost ∝ Lm × S, and S grows with radius² (I). Light creation goes through the same path
   (AttachLight → UpdateLocation). The flood is 2D, so a light near a floor/ceiling in a tall multi-level area touches
   sections it cannot reach in Z (I).
   *Measure:* `profilecsthinkers` (dthinker.cpp:1512) times all light ticks under `InternalDynamicLight`
   (dthinker.cpp:396-409, V); `r_perflog` "relinks" from `dynlights_links_total` (hw_perflog.cpp:285, 591, V).
   `stat lightstats`' link counters are reset at the top of each render, after the tics ran (hw_entrypoint.cpp:669,
   hw_clock.cpp:139-143), so they show only render-time relinks (I from call order; the hw_clock.h:50-53 comment says the same).

2. **Per-eye draw-time light lists for walls and flats** — high.
   Vulkan sets `RFL_BUFFER_STORAGE` (vk_renderdevice.cpp:510, V), so `DrawWall`/`DrawSubsectors` build lists at draw
   time (hw_walls.cpp:561-566; hw_flats.cpp:287-289, V) for every lit surface, every eye, on the main thread. Per surface:
   hash-map iteration, up to two distance-cull calls per light, plane distance, (walls) a 4-vertex footprint test per
   candidate, insertion sort, 64 B copy per light, atomic upload. Walls with decals do it twice (PutWall :879-884 and
   DrawWall). Cost ∝ eyes × Σ_surfaces Ls (I).
   *Measure:* `RenderWall`/`RenderFlat` clocks wrap the whole draw loop incl. SetupLights and `Apply`
   (hw_drawlist.cpp:843-848, 858-864, V); `drawcalls` (vk_renderstate.cpp:228) is nested inside, so SetupLights+MakeVertices
   ≈ RenderWall − drawcalls share (I). `iter_dlight[f]`/`draw_dlight[f]` count light×surface pairs (V).

3. **Sprites, particles and models** — medium-high (scales with monster/particle count).
   `GetDynSpriteLight` per sprite/particle per eye walks its section's map and, for shadow-mapped lights while the shadow
   pass runs, does a CPU `RayTest` per light (hw_spritelight.cpp:174, hw_shadowmap.cpp:154-160, V).
   `hw_GetDynModelLight` walks every subsector a model's radius touches and re-scans the same section's map for each
   subsector of that section (guard at :259 never armed, §6), with `std::find` dedupe (:298) (V). No candidate budget
   for models (V); the shader caps each class at `gl_light_range_limit` (§GPU 1).

4. **Whole-list O(L) walks per frame** — medium with large L (lamps in big maps, BD22 lights never culled by distance at
   link time). `DynamicLightShadowRowsWanted`, `CollectLights` (1–2 walks), `SyncViewLights` (per eye),
   `FillFogDynLights` (per draw info), `SmokeVolume::GatherLights`, and on `renderer` the residency pass step 3
   (hw_levelresidency.cpp:3841-3873). Each is conditional (V, see table); none uses a spatial structure (V).

5. **StreamData copy per draw** — medium, independent of lights but filled mostly by lighting features: 2,000 B memcpy
   per `Apply` (§4.3). At D draws per eye that is 2·D·2,000 B per VR frame into host-visible memory (I).

6. **Recreate scan per tic** — low-medium: O(A) iteration of both actor collections every tic just to find
   `MF8_RECREATELIGHTS` (dthinker.cpp:318-333, V), and every state change of an actor whose class has GLDEFS frame lights
   requests a rebuild (p_mobj.cpp:1099-1103, V).

### GPU

1. **Fragment light loops, twice** — high for light-heavy scenes.
   `ProcessMaterialLight` loops the surface's modulated, subtractive and additive ranges; each iteration: distance,
   normalize, dot, spot smoothstep, and a shadow tap (material_normal.fp:18-146, V). A light in a surface's list is
   evaluated over the whole surface (early out per pixel only after the distance) (V). Per class the loop stops after
   `i > start + gl_light_range_limit` (64 vec4) = 17 lights (I, from `i += 4` and the `>` test); `material_pbr.fp` and
   `material_specular.fp` have no range limit (V by grep). CPU budgets keep walls ≤8 / flats ≤16 (V); models are
   unbudgeted on CPU (V).
2. **Bloom + exposure: ~34 full-pipeline passes per eye** — medium; every pass is its own render pass with barriers
   (§2.5, §4.6). Exposure runs per eye into one shared camera texture with an alpha-blend running average, so in VR the
   adaptation runs twice per frame (I from :2344-2361 and the per-eye call).
   Previously measured (HANDOFF_RENDER_2026-11-11.md:83, flat benchmark, build not named there): "bloom 0.5" ms of a
   14.5 ms frame — single view, not VR.
3. **Shadows when on** — medium-high. Shadow-map pass: quality×1024 fragments each walking the AABB tree for rows in use
   (V). Per surface pixel: one tap at filter 0, `2·filter+1` taps PCF (default filter 1 = 3 taps,
   main.fp:688-723, hw_cvars.cpp:250, V). Ray-traced mode: `min(filter,2)·9+1` ray queries per light per pixel =
   **10 at the default filter** (main.fp:593-613, V). Effect lights with rows add one tap/ray each (main.fp:821-844).
4. **Volumetric beam cones** — medium when present: one full-scene-viewport additive pass per beam, up to 32
   (hw_postprocess.cpp:126-155, hw_postprocess.h:765, V).
5. **Smoke / emissive volume marches** — medium-high when present: half-res march + temporal resolve + 2 keeps +
   2 blurs + curve + scatter + full-res composite per eye (hw_postprocess.cpp:659-1046, V); shared march across eyes
   optional (`r_effects_sharedmarch` 1, :739-784, V).
6. **`main.fp` size** — unknown magnitude. All fork features are compiled into one program per material/variant and
   gated by uniforms (V for the gates, e.g. ShapesAt :1574, BeamLightAt :2301, FogSlabAt :3245). Branches are
   dynamically uniform per draw, but register allocation is sized for the largest path (I). Settle with the driver's
   shader statistics (VGPR count / occupancy) for the default program.
7. **Flat-edge glow** — low-medium: up to 64 segment tests per floor/ceiling pixel when a sector has flat glow
   (main.fp:4234-4337, V; on by sector setting).

### Interaction with the renderer lane
On `renderer`, the static wall/flat mesh evicts every side/flat marked `FB_DYNLIGHTS` back to the old per-draw path
(hw_levelresidency.cpp:4359-4360 for walls; flats marked at :3853-3870), V. Previously measured
(HANDOFF_RENDER_2026-11-11.md:43, 218, 225): "Abandon 288,577 wall and 431,575 flat-plane evictions in the run",
"15% evicted for dynamic lights", "10% for dynamic lights" of the sides reached. So with light-heavy mods the static path
falls back to item 2 above for every lit surface.

---

## 6. Hooks, cvars and dead code in this area

Status: ACTIVE (on by default and on the hot path), DORMANT (off by default but reachable), DEAD (shown unreachable or
without effect), VR-only / flat-only where it matters.

| Item | Where | Default | Status | Evidence |
|---|---|---|---|---|
| `r_dynlights` | g_cvars.cpp:66 | true | ACTIVE | gates tick (dthinker.cpp:303-311) and `HasDynamicLights` (d_main.cpp:695-699); toggling deletes/recreates all lights (g_cvars.cpp:68-72) |
| `gl_lights` | g_cvars.cpp:63 `DEPR_CVAR` | false | DEAD as a switch | `DEPR_CVAR` = hidden value, `CVAR_HIDDEN` (c_cvars.h:777-779); only read by the bench header hw_clock.cpp:353 (grep) |
| `gl_light_max_intensity` | a_dynlight.cpp:44 | 1000 | ACTIVE | clamps radius :243, :364 |
| `gl_light_distance_cull` | a_dynlight.cpp:45 | 2000 | ACTIVE | hw_dynlightdata.h:107-128; uses global `r_viewpoint.Pos` |
| `gl_light_max_collected_subsectors` | a_dynlight.cpp:46 | 1000 | ACTIVE | :551 |
| `gl_light_flat_max_lights` / `gl_light_wall_max_lights` | a_dynlight.cpp:47-48 | 1000 | ACTIVE (per-eye total cap across all flats/walls) | hw_flats.cpp:177,221; hw_walls.cpp:653,689; counters reset per eye hw_entrypoint.cpp:362 |
| `gl_light_flat_candidate_budget` / `wall_…` | a_dynlight.cpp:49-50 | 16 / 8 | ACTIVE | hw_flats.cpp:178-227; hw_walls.cpp:654-733; hw_renderhacks.cpp:126-167 |
| `gl_light_range_limit` | a_dynlight.cpp:51 | 64 | ACTIVE (normal material only) | `ResetFadeColor` hw_renderstate.h:1273; material_normal.fp:98,105,137; absent in pbr/specular |
| `gl_light_distance_cull_cache`, `gl_light_model_dedupe_cache`, `gl_light_spot_cache`, `gl_light_pos_relative_cache` | hw_dynlightdata.cpp:41-44 | false | DORMANT | read in hw_dynlightdata.h:60,105; hw_dynlightdata.cpp:77; hw_spritelight.cpp:239,292,305 |
| `gl_light_sprites` / `gl_light_particles` | hw_dynlightdata.cpp:38-39 | true | ACTIVE | hw_sprites.cpp:305-320, 844 |
| `gl_light_weapons` | hw_dynlightdata.cpp:40 | false | DORMANT (VR default off) | hw_weapon.cpp:2546 |
| `gl_max_lights` | hw_lightbuffer.cpp:24 | 80000, flags 0 | ACTIVE, read once at buffer creation | hw_lightbuffer.cpp:33 |
| `gl_light_shadowmap` | hw_shadowmap.cpp:60 | false | DORMANT | hw_entrypoint.cpp:301; `LightShadowAllowed` :112-117 |
| `gl_light_castshadows` (0/1/2) | hw_shadowmap.cpp:87 | 0 | DORMANT; 2 read once at device creation | vk_renderdevice.cpp:365-367 |
| `gl_light_shadowmap_lights` | hw_shadowmap.cpp:92 | 0 | DORMANT | :107-110 |
| `gl_shadowmap_quality` | hw_shadowmap.cpp:127 | 128 | ACTIVE when the pass runs | texture recreate vk_texture.cpp:70-73 |
| `gl_shadowmap_filter` | hw_cvars.cpp:250 | 1 | ACTIVE in shaders | hw_viewpointbuffer.cpp:93; hw_drawinfo.cpp:2108 |
| `vk_raytrace` | (Vulkan device) | — | DORMANT, start-up only | vk_renderdevice.cpp:365 |
| `LEVEL3_NOSHADOWMAP` | level flag | — | ACTIVE gate | hw_entrypoint.cpp:294,299,301 |
| `r_effectlights`, `_max`, `_quality`, `_perbin`, `_distance`, `_walls`, `_test` | hw_effectlights.cpp:42-74 | 1, 1024, 2, 16, 1536, 1, 0 | ACTIVE (Vulkan only), inert until something spawns an effect light | BeginFrame `mActive` |
| `r_gpuparticles_lights` | hw_cvars.cpp:419 | 32 | ACTIVE while lit particles live | SyncViewLights :686 |
| `fog_dynlights_max` | hw_drawinfo.cpp:1781 | 8 | ACTIVE only when a level asks (`Level->FogDynLights`) and the fog slab is on | :1787-1789, :2429-2433 |
| `gl_fogmode` | hw_cvars.cpp:35 | 2 | ACTIVE | hw_setcolor.cpp SetFog; vk_renderstate.cpp:399-413 |
| `gl_distfog` | hw_lighting.cpp:36 | 70 | ACTIVE (table) | :36-61 |
| `gl_weaponlight` | hw_lighting.cpp:27 | 0 (fork; upstream 8) | ACTIVE | — |
| `gl_bloom` | hw_postprocess_cvars.cpp:37 | true | ACTIVE | :1691 bloom, :2288 exposure |
| `gl_bloom_pin_beams` (+ `gl_bloom_pin_*`) | :139-192 | false | DORMANT | PPLightMask::WantedByCvars hw_postprocess.cpp:3418-3421 |
| `r_lightmask_debug` | :252 | 0 | DORMANT (debug) | :3441-3456 |
| `gl_tonemap` | :416 | 0 (None) | DORMANT | :2554-2559 |
| `gl_lens` | :472 | false | DORMANT | :2143 |
| `gl_fxaa` | :478 | 0 | DORMANT | :2197 |
| `gl_ssao` | :486 | 0 | DORMANT | :2681; also switches the scene to `GBUFFER_PASS` (hw_entrypoint.cpp:384, 413) |
| `r_smoke` / `r_smoke_preset` | hw_cvars.cpp:561; :287 | true / 3 | ACTIVE only with smoke content | :666 |
| `r_effects_sharedmarch`, `r_effects_foveated`, `r_emissivevolumes_*` | :357-394 | on | ACTIVE with content | — |
| `vr_openxr_multiview` / `_postprocess` | hw_vrmodes.cpp:643, 655 | false / true | DORMANT (multiview off by default) | |
| `stat lightstats`, `stat shadowmap`, `stat gpu`, `bench` | hw_clock.cpp:282; hw_shadowmap.cpp:119; hw_postprocess.cpp:48; hw_clock.cpp bench | — | ACTIVE tools | see §8 |
| `#if 0` ColorFlickerLight / RandomColorFlickerLight | a_dynlight.cpp:319-344 | — | DEAD | compiled out; types still in `ELightType` (a_dynlight.h:37-48) |
| `SectorLight`, `DummyLight` types | a_dynlight.h:43-44 | — | SectorLight ACTIVE (:346-358); DummyLight: no case in Tick, so radius stays the activation value | V (switch has no DummyLight case) |
| Section revisit guard in `hw_GetDynModelLight` | hw_spritelight.cpp:253, 259 | — | DEAD check | `dl_validcount++` then `if (section->validcount == dl_validcount) return;` but nothing in the function sets `section->validcount`; `BSPWalkCircle` (models.h:162-168) does not either |
| Coronas | hw_drawinfo.cpp:4356-4359 | — | DEAD | commented out |
| `SHADER_LITE` paths in main.fp / material_normal.fp | material_normal.fp:26-50 | — | DEAD on Vulkan | defined only by the GL backend (gl_shader.cpp:1064); Vulkan never defines it (grep) |
| `FLightBuffer` UBO path (`mBufferType=false`, `GetBinding`) | hw_lightbuffer.cpp:42-49, 128-136 | — | DEAD on Vulkan | Vulkan device sets SSBO caps (vk_renderdevice.cpp:510) (I: `useSSBO()` true there) |
| `!BuffersArePersistent()` branches in PutWall/PutSprite/flat | hw_walls.cpp:859-866; hw_sprites.cpp:854; hw_flats.cpp:572-576 | — | DEAD on Vulkan | `RFL_BUFFER_STORAGE` always set (vk_renderdevice.cpp:510) |
| `IShadowMap::ShadowTest(lpos,pos)` 2-arg overload | hw_shadowmap.cpp:145 | — | DEAD (no caller) | full-tree grep of `ShadowTest(`: the only call is the 3-arg one at hw_spritelight.cpp:174 |
| "unused" record floats 14/15 | hw_dynlightdata.cpp:187-188 | — | DEAD payload | 8 B per light per surface |

---

## 7. Branch differences

Compared file by file (`diff` of the four trees) for every file read above.

- **renderer** (`175063a`): identical to `main` in a_dynlight.*, hw_dynlightdata.*, hw_lightbuffer.cpp, hw_shadowmap.*,
  hw_lighting.cpp, hw_spritelight.cpp, hw_flats.cpp, hw_renderhacks.cpp, hw_sprites.cpp, all post-process files,
  hw_effectlights.*, hw_smokevolume.cpp, material_normal.fp, shadowmap.fp (V). Differences in this area:
  - `hw_walls.cpp`: white-light glow rule also in the view-independent build (:889-897), new light setters
    `SetSectorLight/SetSideGlow/SetTierLight*` for the light patch (:2419-2462), no distance-cull impostor in a static
    build (:2616-2617) (V).
  - `main.fp`: `LEVEL_MESH` block (renderer :32-131) that reads per-surface "light records" (320 B: fog, light parms, glow,
    colours) from set 1 binding 17 instead of StreamData (V). Dynamic lights are not part of the record:
    `SetLightIndex(-1)` for mesh draws (hw_levelresidency.cpp:4667, 4903) and lit surfaces are evicted (§5) (V).
  - `hw_levelresidency.cpp:3835-3873`: once-per-pass O(L × touched sides/sections) walk of every light's touch lists to
    mark `FB_DYNLIGHTS` (V).
  - `hw_entrypoint.cpp`: Tracy zones only (V). `dthinker.cpp`: `sim_prefetch` (prefetch next thinker), not the light loop
    (V). `hw_clock.cpp`: flat-screen fallback for the summary (V).
- **netplay** (`1c1805f8`): every lighting/post file identical to `main` except `main.fp` (look-down fade in
  `MODEL_EYE_FADE` only, :5082-5088) and `hw_drawinfo.cpp` (world psprites), neither affecting lights (V).
- **selaco** (`7e111a3d`): no effect lights (`hw_effectlights.*` missing), no emissive volumes or exposure impulse in
  `Pass1` (selaco hw_postprocess.cpp:2564-2580), no cast-shadow ladder (selaco hw_shadowmap.cpp has only
  `gl_light_shadowmap` and `gl_shadowmap_quality`), no `DynamicLightShadowRowsWanted`, no `[LIGHTLIFETIME]`
  `ReleaseLightsBuiltFrom` in `RemoveLight` (the slow-mo muzzle-flash use-after-free fix is absent), no
  `dynlights_links_total`; adds `A_AttachLightEx` and `Actor.DeleteAttachedLights` natives (V, diff of
  a_dynlight.cpp, hw_dynlightdata.cpp). `main.fp` differs by ~1,170 lines and hw_postprocess.cpp by ~1,070 (V, diff
  line counts; contents not walked).

---

## 8. Clustered / tiled light design — what it replaces

The fork already contains a working binned design for effect lights (§4.5): CPU-built world-space 3D bins around the eye,
uploaded once per frame, read per pixel by `EffectLightsAt` (main.fp:882-902) with shadow rows from the same shadow map.
A clustered path for `FDynamicLight` can reuse that pipeline (BinBuilder, the bin SSBO layout, the per-pixel loop) or
move the binning to a compute pass. Either way:

**Replaced (exact functions):**
- Sim-side linking: `FDynamicLight::LinkLight`, `UnlinkLight`, `CollectWithinRadius`, `AddLightNode`, `DistToSeg`
  (a_dynlight.cpp:455-727); `FDynamicLightTouchLists` and the relink branch of `UpdateLocation` (:434-439);
  `FGlobalDLightLists` (g_levellocals.h:58-59) and its clears (g_level.cpp:589-590); `RelinkDynamicLightsForLevel`
  (g_level.cpp:2981); the relink in `R_UpdatePoseAnchoredLights` (hw_entrypoint.cpp:192-193).
- Per-surface lists: `HWWall::SetupLights` (hw_walls.cpp:600-806), `HWFlat::SetupLights` (hw_flats.cpp:157-271),
  `HWDrawInfo::SetupLightsForOtherPlane` (hw_renderhacks.cpp:112-202), `GetLight` (hw_dynlightdata.cpp:52),
  `gl_InsertBestLightCandidate`, `gl_IsDistanceCulled`, `gl_GetLightPosRelative` (hw_dynlightdata.h:58-130), the
  `thread_local` candidate arrays (hw_walls.cpp:102, hw_flats.cpp:38), per-draw `FLightBuffer::UploadLights` and
  `FDynLightData`; the wall-with-decals double setup (hw_walls.cpp:879-884); `dynlightindex` plumbing
  (hw_drawstructs.h:208,356,421,509; `SetLightIndex`); the budget cvars `gl_light_{flat,wall}_{max_lights,candidate_budget}`.
- Sprites/models: `HWDrawInfo::GetDynSpriteLight` (hw_spritelight.cpp:101-222; including the per-sprite CPU
  `ShadowTest` ray), `hw_GetDynModelLight` (:228-321), the model branch of `HWSprite::PutSprite` (hw_sprites.cpp:843-849),
  psprite lighting in `DrawPlayerSprites` (hw_weapon.cpp:2421-2426, 2546-2549). Sprites would read the cluster at the
  vertex or fragment stage like `effectLightMode 2`.
- Shader: the `lights[uLightIndex]` range header and three loops in `ProcessMaterialLight` (material_normal.fp:85-146,
  material_pbr.fp:~100-180, material_specular.fp:~55-120) become a loop over the pixel's cluster list (keeping three
  classes: modulated, subtractive, additive).
- Duplicate whole-list walks that could read one per-frame compact light array: `SyncViewLights`, `FillFogDynLights`,
  `SmokeVolume::GatherLights`, `DynamicLightShadowRowsWanted`, `CollectLights` (list order), and on `renderer` the
  `FB_DYNLIGHTS` residency marking and `EV_DYNLIGHTS` eviction (hw_levelresidency.cpp:3841-3873, 4359-4360).

**Kept:** `FDynamicLight::Tick` evaluation (flicker/pulse/sector, pose anchoring), the 4-vec4 light record from
`AddLightToList` (as a global list writer), `shadowIndex` convention (±(row+1), 1025 = none), shadow-row assignment
(`CollectLights`, `AssignShadowRows`), `shadowAttenuation`/`lightContribution` math, the shadow map pass.

**Things the current design does that a cluster must reproduce** (V unless noted): the side test that rejects lights
behind a wall (hw_walls.cpp:678) and below/above a flat (hw_flats.cpp:205-209) — a cluster needs N·L plus a side test or
the light leaks through thin walls (I); the wall 4-vertex footprint test; `DontLightMap`/`DontLightSelf`/
`DontLightOthers`/`DontLightActors` per receiver (a_dynlight.h:259-265); `RF2_LIGHTMULTALPHA` scaling;
`lightadditivesurfaces`; portal displacement (`PosRelative` per portal group, a_dynlight.h:254-257) — clusters must be
built per portal group or lights pre-offset per view (I); flood-fill occlusion: today a light only reaches sections
connected within its radius through open lines, a pure distance cluster lights through closed doors and solid walls
unless shadows are on (I).

---

## 9. Open questions and the measurements that would settle them

| Question | Measurement / instrument |
|---|---|
| Real per-tic relink cost under BD22 fire | `profilecsthinkers` (dthinker.cpp:1512; lights timed as `InternalDynamicLight`, :396-409) during a firefight; `r_perflog` "relinks" (dynlights_links_total growth per window). `stat lightstats` link counters are not usable (reset after the tics, §5). |
| CPU of draw-time SetupLights per eye | Tracy zone on `HWWall::SetupLights`/`HWFlat::SetupLights` (renderer branch has `REMA_ZONE_N`, hw_entrypoint.cpp:267,363 there), or `RenderWall`/`RenderFlat` minus `drawcalls` with lights on vs `r_dynlights 0` |
| How many light×surface pairs per eye | `stat lightstats` `iter_dlight[f]` (processed) / `draw_dlight[f]` (rendered); perf_track `dlights` = `draw_dlight + draw_dlightf` (hw_perftrack.cpp:567) — note it counts pairs, not lights |
| Sprite/model light gathering cost | `SetupSprite` clock (hw_bsp.cpp:1054-1056, only when not multithreaded — VR default) and `RenderSprite`; `dynlights_model_*` counters (hw_spritelight.cpp:257-314) |
| GPU cost of light loops | `stat gpu` groups `scene.opaque*` with lights on/off; note timestamps are written at `COLOR_ATTACHMENT_OUTPUT` stage (vk_commandbuffer.cpp:543, 567), not bottom-of-pipe |
| Shadow pass cost | `stat shadowmap` (`UpdateCycles` = CPU of tree upload + CollectLights + record, hw_shadowmap.cpp:162-180, FinishUpdate) and GPU group `shadowmap` (hw_postprocess.cpp:2847); perf_track `shadow_rows`, `shadow_cpu_ms` |
| Post-process cost per eye | GPU groups `exposure`, `bloom`, `bloom.pinned`, `tonemap`, `colormap`, `lens`, `fxaa`, `ssao`, `pp.smoke*`, `pp.emissive`, `volumetricbeam`, `heatmap`, `pp.heat*`, `pp.exposureimpulse`; CPU clocks `PostProcess` and `VRPostProcessScene` (hw_entrypoint.cpp:391,487; vk_postprocess.cpp:98) |
| Is exposure adapting twice per VR frame? | Capture the 1×1 camera texture per eye in RenderDoc; or log `FirstExposureFrame`/blend per eye |
| `main.fp` register pressure | Driver shader statistics (VGPRs/occupancy) for `Default` + `EFFECT_LIGHTS` + `SURFACE_DAMAGE` program; compare a build with fork blocks compiled out |
| Light SSBO reuse across frames with one buffer | Vulkan uses a single persistently mapped `FLightBuffer` (vk_renderdevice.cpp:610). Whether the CPU can overwrite lights the GPU still reads depends on the frame fence placement (other area) — check with sync validation |
| StreamData real size | `static_assert(sizeof(StreamData)==2000)` or print `MAX_STREAM_DATA` at start-up (the 2,000 here is from a replica) |
| Cost of the effect-light CPU binning at the pool cap (1,024–4,096 lights, long tracers) | `r_perflog` CPU sample `fx.effectlights` (BeginFrame and PrepareFrame both add to it, hw_effectlights.cpp) with `r_effectlights_test N` |
| Effect of `gl_light_range_limit` on models with > 17 lights | RenderDoc capture of a model in a lamp-heavy room; compare with limit 0 |
