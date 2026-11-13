# E — Scene build and the render lane's new renderer (playbook Phases 3, 4, 5, 7) against the fork

Area: the CPU scene build on `main` [F] and what the render lane has built on `renderer` [FR] behind `r_newrenderer`,
mapped to playbook Phase 3 (tic-rate scene build + GPU interpolation), Phase 4 (GPU-resident static world), Phase 5
(GPU visibility, portals) and Phase 7 (sprites, particles, models, overlays).

Trees: [F] `/home/claude/uzdxrema` @ e84623f (shallow, one commit). [FR] `/home/claude/uzdxrema_renderer` @ abb19af
(shallow; `origin/renderer` = abb19af). The newest render handoff is `[FR] HANDOFF_RENDER_2026-11-11.md` ("STANDBY
BEFORE STAGE 3 (2026-11-12, 23:20)": stage 2d built at b3fc177, latest exe `doomxr_rend1113a.exe`, "no stage 3 code until
he answers" Q1-Q4). abb19af adds only docs on top of b3fc177 (the survey's own cross-check, [S] 09 summary item 1). I
checked by diffing the two trees: the scene files differ only by the lane's hooks (`hw_bsp.cpp` 42 changed lines,
`hw_drawinfo.cpp` 34, `hw_walls.cpp` 56, `hw_drawstructs.h` 9, `hw_entrypoint.cpp` 3). `hw_flats.cpp`, `hw_sprites.cpp` and
`hw_weapon.cpp` are byte-identical. `models.cpp` differs by main's gun-fit work (166 lines). So [F] and [FR] line numbers
agree outside the hooks, give or take the hook offsets.

Numbers marked "prev. measured" come from the lane's handoff and plan files (bridge runs on the owner's PC). They are
leads, not [M]. [M] is `10_MEASUREMENTS.md` (flat 3056x1689, `doomxr_rend1113a.exe` at `r_newrenderer 0`).

---

## Delta table

Column key: **Status** = Done / Partial / Not started / N/A. The **Est. saving** column always derives from [M] (or
says it cannot). VR means `vr_mode 15`. "Non-MV" means `vr_openxr_multiview 0`, the default ([F]
`hw_vrmodes.cpp:643`), where the scene is built per eye. "MV" means one build for both eyes.

| ID | Item | Fork status | Where in fork | What is left | Done when (measurable) | Don't touch | Netplay / determinism | Est. saving on modlist (Inference) |
|---|---|---|---|---|---|---|---|---|
| P3.A | `TicScene` (scene built once a tic, redrawn per frame) | Not started | No such object. The walk runs per scene per frame: [F] `hw_bsp.cpp:1422` `RenderBSP`, called per eye ([F] `hw_entrypoint.cpp:372`) | Not in the lane's plan (`RENDERER_REBUILD.md` keeps "ticFrac and r_viewpoint computed as today"). Superseded if P4+P5+P7 land. Recommend not building it (see Findings §6) | `Scene.Create` ≤ 35/s at 90 Hz | `models.cpp`, `hw_sprites.cpp`, `hw_weapon.cpp`, `vr_*` | A 360° build must not set `ML_MAPPED`/`*_DRAWN` (automap and saves only, not sync-hashed) | 90 frame builds become 35 tic builds of 2-4x the items ([P] Phase 3 C: "about the same CPU"). ≈0 on its own |
| P3.B | Rebuild triggers (tic, heightsec zone, envelope) | Not started | — | Only if P3.A is built | — | — | none | — |
| P3.C | Conservative visibility over head motion (capsule, multi-sample clipper) | Not started | Under MV the walk runs once from eye 0 with no IPD margin ([F] `hw_entrypoint.cpp:453`; [S] 03 §6 item 1) | Needed for **MV even without P3**: cull from the union of the eye frusta or the head centre plus an IPD margin | Head-shake gate ([P] 22 §5.3) passes under MV | `vk_openxrdevice.cpp` grip/hand half | none | Enables MV; see F-RL-9 |
| P3.D | prev/cur plane and scroll tables in SSBOs; `D_Render` stops `DoInterpolations` | Not started (precursors exist) | `DoInterpolations`/`RestoreInterpolations` write live sectors every frame ([F] `d_main.cpp:700,708`). `DSectorPlaneInterpolation::Interpolate` calls `P_RecalculateAttached3DFloors` and `CheckPortalPlane` ([F] `r_interpolate.cpp:470-499`). The flat VBO is patched per frame (`CheckUpdate`, [F] `hw_bsp.cpp:1042`). Precursors on [FR]: the `SectorPlanes` SSBO (set 1 binding 12, read by `levelmesh.vp:108-113`) and `FSnapActorRec` with prev fields | Plane prev/cur into binding 12; static flats/walls take z from it; movers stop being evicted | `DoInterpolations` call count = 0 with the new path on; pixel diff with moving floors | — | Removes a per-frame write/restore of sim structs (good for the thread). The tic itself is unchanged | Keeps movers on the mesh. Sunder is lift-heavy: today every mover side goes to the old path (F-RL-3) |
| P3.E | Per-frame residue (pose, cull, ticfrac, sky, material timer) | N/A | — | — | — | — | — | — |
| P3.RH | Render-hack "eye above/below" variants | Not started | Hacks run per scene in `DispatchRenderHacks` ([F] `hw_drawinfo.cpp:3704`) | Only with P4.7 | — | — | none | small (map-dependent) |
| P4.1 | Cluster mesh built at load (walls + flats in persistent buffers, surface id per vertex) | **Partial** (walls and flats of static sides/planes) | [FR] `hw_levelresidency.cpp` `Build` (≈:2088-2596): runs `HWWall::Process` into `HWMeshHelper` with `noDistanceCull`, copies `CreateVertices`, writes the surface id into `lu`. `hw_levelbuffers.cpp:77` allocates. Flats reuse the `CreateVBO` buffer with index runs (`BuildFlatPart`). Vertices hold absolute z (not plane-indexed). Classifier refusals: [FR] `hw_levelstatic.cpp:69-88` | Plane-indexed z (P3.D), then 3D floors, heightsecs, portals, animated textures, translucent mids, polyobjects (all refused today) | Static share of reached sides ≥ 95% on Abandon/Sunder with the owner's list (`levelmeshstats`) | `hw_walls.cpp` `Process`/`PutWall` image-exact (the lane's own 1112p lesson) | Reads sim only; the build writes `vertex_t` through `RecalcVertexHeights`, as `Process` does | See F-RL-1: measured net **negative** today |
| P4.2 | T-junction verts baked at load (`gl_seamless`) | Partial | Worst-case slot per side: `SideVertexBound` ([FR] `hw_levelresidency.cpp:1703`). A vertex height-list change evicts the side (`VertexChanged`, `:4371`) | Plane-indexed splits so movers stay resident | 0 EV_SIGNATURE evictions from vertex lists on a door/lift map | — | none | Part of F-RL-3 |
| P4.3 | Material slots (animation and switches write 4 bytes) | Not started | One `FMaterialGroup` per (texture, clamp), one `SetMaterial` + `DrawIndexed` per group ([FR] `:4675-4685`). Animated base pics are refused (`LSR_ANIMATED`, `hw_levelstatic.cpp:84`). A texture change evicts, then fold-back regroups (`WallGroupOf`) | Needs bindless (stage 5) or a slot table | Animated/switch sides resident; ≤ 50 world draws per pass | — | none | Prev. measured 54-107 groups = draws per scene. Draw-call CPU 0.76→0.37 ms no-mod |
| P4.4 | Light/colour/fog tables patched per tic | **Done** (walls and flats; exact against `Process`) | 320 B light records per key, `RunLightPatch` ([FR] `:2994`), `RunFlatLightPatch` (`:3109`). The /fp:fast `RescaleLightLevel` form is learned at build (`ObserveRescale :1372`, `PatchRescale :1395`) | The duplicated light logic is guarded only by `r_levelmesh_verify`. Flats on a level with undecided rounding stay old | `r_levelmesh_verify 1`: 0 differ (prev. measured Abandon 1,395,071 parts, Sunder 70,914) | `hw_walls.cpp` setters | none | Light flicker no longer evicts. Prev. measured: light patch 0.03 ms a frame on Sunder |
| P4.5 | 3D floors without per-slice draws | Not started | Refused: `LSR_3DFLOOR` (`hw_levelstatic.cpp:74-75`). The sector shadow evicts on any `XFloor` change | Light-list offsets per cluster ([P] P4.5) | 3D-floor sides resident on a test map | — | none | 0 on Sunder/Abandon (no 3D floors there per `levelstats`, prev. measured) |
| P4.6 | Slopes | Partial (implicitly) | The classifier does not refuse slopes. Flats come from `CreateVBO` (slope-aware). A moving sloped plane evicts | With P3.D | — | — | none | — |
| P4.7 | Render hacks resident | Not started | Hacks stay on the walk: hacked subsectors ([FR] `hw_bsp.cpp:1166` per STAGE3 plan), missing textures to `HandleMissingTextures` | Load-time classification plus variants | — | — | none | small, map-dependent |
| P4.8 | Decals resident | Not started; **decals evict their wall** | `side->AttachedDecals != nullptr` → `EV_DECALS` ([FR] `hw_levelresidency.cpp:4359`). Lasting surface damage (fork) → `EV_DAMAGE` (`:4362`, flats `:4452`) | Resident decal quads; damage key as a surface-record field | Evictions for decals/damage = 0 in a BD22 fight | `hw_surfacedamage*` semantics | none | In BD22 fights blood decals accumulate, so the static share falls over a fight. Not measured on the modlist |
| P4.9 | Patches once per tic, never vertices | Partial | One compare pass per tic over all sectors (`RunTicPass :3681`, triggered when `gametic`/`LocalTimer`/lights change, `:3946`). Moving sectors are re-checked per frame (`RefreshMovingSectors :3892`). Plane moves **evict**; fold-back rebuilds slots after one still tic (`FoldBack :3637`, ≤500 sides a pass, `:837`) | Plane-index patching instead of eviction (P3.D). The O(all sectors) pass should become dirty-driven | Pass ≤ 0.2 ms/tic on Sunder | — | Reads `interpolations[].ForceGet()` (a read barrier that may write the pointer, [S] 09 B.8.6) | Prev. measured 1.43 ms a pass on Sunder (13,028 sectors). At 90 Hz: 1.43 × 35/90 = 0.56 ms/frame average, **1.43 ms spikes** |
| P4.10 | Indirect buckets (`vkCmdDrawIndexedIndirectCount`) | Not started | ZVulkan wraps `drawIndirect`/`drawIndexedIndirect` ([FR] `libraries/ZVulkan/include/zvulkan/vulkanobjects.h:391-392`). **No caller in `src`** (grep). `drawIndirectCount` is not requested. `OptionalDescriptorIndexing` exists (`vulkanbuilders.cpp:1834`) but has no caller | STAGE3 3c: wrapper, feature request, `FRenderState::DrawIndexedIndirect` | Draw-call CPU for the mesh ≈ 0.05 ms/scene | Descriptor sets 0/1 layout (shared) | none | ≤ 0.37 ms/scene left (prev. measured mesh draw-call CPU) |
| P5.1 | Cull compute pass (combined frustum) | Not started | Planned as STAGE3 3d (`r_gpucull 1`). Frustum planes already built for the light cull (`BuildViewLightFrustum`, [F] `hw_drawinfo.cpp` ≈612-660) | Q1 must be answered first (see P5.6) | — | — | none | 0 while the walk stays (Q3) |
| P5.2 | Two-phase Hi-Z | Not started | STAGE3 3d. The depth image is already in set 0 binding 3 | Same | ≤ 1.5x perfect-visibility triangles | — | none | GPU only. The walk list is already angle-occluded; prev. measured GPU equal to the old path (1112r) |
| P5.3 | Portals on the resident world | Not started | `BeginScene` refuses portal, skybox and mirror scenes ([FR] `:4229-4233`). Sky sides replay their kept portal walls through `PutPortal` every scene (`:4388-4401`) | Per-portal cull plus conditional rendering | ≤ 0.05 ms CPU per visible portal | `hw_portal.cpp` mirror stencil (an MV gap) | none | 0 on Sunder/Abandon (no portals, prev. measured `levelstats`). Sky sides cost a copy + `PutPortal` per scene |
| P5.4 | Skybox → cubemap | Not started | — | Optional | — | — | none | map-dependent |
| P5.5 | Automap/`*_DRAWN` bits from GPU visibility | Not started; **decision open** (STAGE3 Q2) | The walk writes `ML_MAPPED` ([F] `hw_bsp.cpp:713`), `SSECMF_DRAWN` (`:604-624`), `SECMF_DRAWN` (`:1070`). Stage-6 contract rule 4: queue on the render thread, apply before `G_Ticker` ([FR] `RENDER_LANE_HANDOFF_2026-11-09.md`) | Owner picks the timing and what "identical" means | Saves/automap equal to the owner's definition; `suite.sh` ALL GREEN | — | Local, not in the consistency hash, not in `usercmd_t` (netplay lane's answer, §4). Saves may differ from the old path | 0 (correctness item) |
| P5.6 | Retire the per-frame BSP walk | Not started; **blocked on Q1** | The mesh draws only what the walk reached (`ReachedSides.Push`, [FR] `:4406`; `ReachedFlatRuns` `:4491`). Sprites, particles, portals, hacks, `CheckUpdate`, evicted/non-static surfaces and automap bits all come from the walk ([S] 09 B.7, STAGE3 Q1) | Owner: Q1 (a) keep the walk, or (b) a walk-free main view plus a non-walk finder for actors and dynamic surfaces | No per-frame `Scene.BSP` for the main view | — | Automap bits per P5.5 | bsp 0.43 + clip 1.24 = **1.67 ms/scene** [M]§1, so 3.3 ms/frame non-MV |
| P7.1 | Per-actor instance record | Partial precursor | `FSnapActorRec` (312 B, prev. measured), written at `R_SnapshotWrite` ([FR] `d_net.cpp:2493,2557`) when `r_snapshot ≥ 1`. **No reader**: `R_SnapshotAcquire/Release` are never called (grep). `r_renderthread` is a stub that resets itself ([FR] `r_snapshot.cpp:31-40`) | A reader: a new world-actor submit path in new files | Records consumed by an instanced draw; write < 1 ms (split across cores) | `hw_sprites.cpp` `Process` (avatar lane) | Pure read after the tic; deterministic ordering is not needed | Prerequisite for P7.2 and stage 6. Prev. measured write 5.5 ms for 17,481 records |
| P7.2 | Vertex-shader billboard, rotation, interpolation | Not started | `HWSprite::Process` interpolates and rotates per sprite per scene on the CPU ([F] `hw_sprites.cpp:1057`, `InterpolatedPosition :1141`). On Vulkan the vertices are built at draw time | New instanced path; the old path is kept for models, avatar, worn items and weapons, and for any actor using fork features (`modelData`, follow-hand, outline, voxel latches) | ≤ 10 sprite draws per pass; identical rotation frames per eye | `hw_sprites.cpp`, `models.cpp`, `vr_*` | Must not write `RF_XFLIP`/`isoscale*` into actors | [M] sprites **0.62 ms/scene** (192 sprites), so 1.24 ms/frame non-MV. Prev. measured 0.8-4.3 ms per flat view in Abandon fights |
| P7.3 | Opaque sprite routing (out of the sorted translucent list) | Not started | All non-model sprites except solid wall sprites go to `GLDL_TRANSLUCENT` ([F] `hw_drawlistadd.cpp:136-146`) | Route solid sprites of the new path into an early-Z bucket | Translucent list size ÷10 on a monster map | `hw_drawlistadd.cpp` is not on the protected list, but it routes the avatar's models: hand any change there to the avatar lane | none | Sort cost unclocked ([S] 04 §5.6). Cannot derive from [M] |
| P7.4 | Commuting blends unsorted, GPU sort or OIT | Not started | `RenderTranslucent` sort ([F] `hw_drawinfo.cpp:3855`) | With P7.2 | — | — | none | GPU: [M] shows 25.9 ms GPU wait with ~12 ms not attributed to named groups (Findings §5) |
| P7.5 | Cyclops-relative fuzz | Not started | — | Per-eye correctness under MV | — | — | none | 0 |
| P7.6 | GPU translations | Not started | — | Long-term | — | — | none | — |
| P7.7 | Particles as instances | Partial (fork-only GPU paths exist) | GPU particle ring `r_gpuparticles` (true, 65,536 slots, [F] `hw_cvars.cpp:291,312`), `MeshParticleBuffer::Draw` ([F] `hw_meshparticles.cpp:492`), `DebrisPool::DrawMeshes` ([F] `hw_debrispool.cpp:1284`). CPU particles (`r_maxparticles` 4000, [F] `g_cvars.cpp:115`) stay one `HWSprite` each through `RenderParticles` ([F] `hw_bsp.cpp:1005`). `P_FindParticleSubsectors` runs O(subsectors) per frame ([F] `d_main.cpp:701`) | Route CPU particles and VisualThinkers through the instance path | — | — | Particles use client RNG; keep `FCRandom` (handoff §8) | [M] particles/beams 0.72 ms per **tic** (sim side). Render share not separated |
| P7.8 | Models: bones once a frame, instanced | Not started; **out of lane** | Bones per draw per eye, `AnimateBones` VM per draw ([F] `hw_sprites.cpp:503`); avatar pose solved in the draw (`VRRig_*`) | Avatar lane, through the owner | ≤ 5 µs per model per frame | **All of it** (lane rule 1) | Gameplay used to read the drawn pose; holsters now hang on the sim torso (netplay note 2, 2026-11-11) | [M] GPU models 0.84 ms. CPU not separated |
| P7.9 | Native overlays (HP bars) | Not started (Phase 10) | `RSQTHP_Bar` is a mod actor (3,622 bars, [M]§3) | Phase 10 (sim agents) | — | — | — | Mostly sim (16.5 ms/tic [M]§3) |
| P7.10 | VR weapon as a world-space model | **Done differently** | `DrawPlayerSprites` in scene ([F] `hw_drawinfo.cpp:4539`), `DrawWorldPlayerSprites` (`:4548`) | Avatar lane | — | `hw_weapon.cpp`, `models.cpp` | — | — |
| F-RL-1 | `r_newrenderer` 1/2/3 static path (stages 2a-2d) | Built, gated, **slower** | Entry [FR] `hw_bsp.cpp:1464` → `LevelResidency_BeginScene` (`hw_levelresidency.cpp:4221`); walk hooks `hw_bsp.cpp:661,760,1158` (workers `:346,368`); draws `hw_drawinfo.cpp:3830,3835`; shader `levelmesh.vp`, `main.fp` `LEVEL_MESH` | Keep it off on the owner's builds until stage 3 (b) and clustered lights exist | New ≥ old FPS on Sunder MAP15, no mods and modlist, `r_dynlights 1` | — | Suite never run with it on (F-RL-11) | Prev. measured: 1112y 14.27 → 16.45 ms (+2.18); 1112x 13.12 → 14.88 (+1.76). Both no mods, `r_dynlights 0` |
| F-RL-2 | Per-frame "first scene" work: `ClearEvictions`, change pass, light-key hash, per-reached-side own-field hash | Built | `:4251-4275`; `SkipWallDecision :4371-4376` (`WallCache_SideFastHash` once a frame per reached side) | Dirty-driven pass; drop the per-side hash where the pass already covers it | — | — | none | Included in the 1.43 ms/pass and the 1.05 ms walls figure |
| F-RL-3 | Evictions return whole sides to the old path (dynamic lights, decals, damage, dither, movers, texture changes) | Built (by design) | `SkipWallDecision :4358-4381`, `FlatDrawableNow :4439-4457`. Mesh draws set `SetLightIndex(-1)` (`:4667`, flats `:4903`): no dynamic light in the mesh | Clustered lights (stage 5 / Phase 6) and P4.8 | Evicted share < 5% with `r_dynlights 1` on the modlist | — | none | Modlist has 122 dynamic lights at the start [M]§1. Prev. measured (2c-1, Abandon): 15% of reached static sides evicted for lights, 5% for signature. With lights on, Abandon had 288,577 wall and 431,575 flat evictions in one run. **Saving goes toward 0, overhead stays** |
| F-RL-4 | Camera-texture scenes are not excluded; 2-scene index cap | Hazard | `BeginScene` has no `mainview` test (`:4221-4234`); `FRAME_INDEX_SCENES = 2` (`hw_levelbuffers.h:78`); fallback at `:4302-4313` | Gate on the main view; size regions by eye count | `levelmeshstats` `ScenesNoRoom` = 0 in non-MV VR with a camera texture | — | none | Correctness. A VR eye can draw the whole static buffer (≈+4 ms GPU, prev. measured 1112q) |
| F-RL-5 | Distance cull: the mesh draws real walls where the old path draws impostors | Divergence | [FR] `hw_bsp.cpp:660-666` ("a static side ignores the distance cull"). Owner default `gl_line_distance_cull 4000` ([F] `hw_bsp.cpp:449`). The lane's gates run with `+gl_line_distance_cull 0` | Decide the image reference (impostors are buggy, [S] 03 B1-B3, B12-B13) | Image gate also run at 4000 | — | none | 0 |
| F-RL-6 | Memory | Built | Worst-case slots, staging copies kept (`vk_hwbuffer.cpp:73-97`), 2x per-frame index buffer, CPU index copies | Shrink when the cull exists | — | — | none | Prev. measured 157.7/180.0 MB GPU (Abandon/Sunder), staging not counted. On 16 GB host / 12 GB card |
| F-RL-7 | Stage 6: snapshot writer and render thread | Writer only | [FR] `r_snapshot.cpp:28-40`, `d_net.cpp:2493,2557`; contract in `RENDER_LANE_HANDOFF_2026-11-09.md` | A reader; the actor path (P7.1-2); a walk-free level or level images | `r_renderthread 1`: `suite.sh` ALL GREEN both ways + one headset round | `TryRunTics` (netplay lane) | The contract is agreed (snapshot after `P_PredictClient`; `vr_input_in_command`) | Prerequisite for 90 Hz with any tic > ~3 ms (Findings §7) |
| F-RL-8 | Wall cache `r_wallcache` (stage 2b) | Dormant, still called | `WallCache_Process` per processed wall ([FR] `hw_bsp.cpp:349,763`) | Remove or leave | — | — | none | One call per wall at defaults (tiny) |
| F-RL-9 | Multiview `vr_openxr_multiview` | Partial (never run, per the lane audit) | Default false ([F] `hw_vrmodes.cpp:643`); one build when on ([F] `hw_entrypoint.cpp:354,372`); `levelmesh.vp:116-119` writes `gl_ViewIndex` | 4 gaps (mirror stencil, SSAO left only, eye-0 culling, layered PP), `RENDERER_REBUILD.md` "experimental multiview" | Both eyes correct (RenderDoc); head-shake gate | `vk_openxrdevice.cpp` input half | none | **−1 scene build/frame: −4.35 ms** at map start [M]§1. Prev. measured fights 5-11 ms per flat scene |
| F-RL-10 | `vr_scene_multithread` off in VR; wall workers disabled with the mesh | Existing cvar | [F] `hw_cvars.cpp:278`, `hw_bsp.cpp:1438-1440`; [FR] `hw_bsp.cpp:650,743` | Measure it on and off in VR | — | — | Scene worker writes `dithertransflag` (B7) and shares `validcount` | Between 0 and (SetupWall+SetupFlat+SetupSprite) per scene, which [M] does not split (Findings §4) |
| F-RL-11 | Netplay merge gate with `r_newrenderer` on | **Not run** | No run recorded in the handoffs; cloud sessions cannot reach the rig packages | Netplay lane runs `suite.sh` on `renderer`, on and off | ALL GREEN, both | — | The gate itself | — |
| F-SC-1 | Fork distance-cull impostors (one per seg, ceiling texture, uninitialised light) | Live on main | [F] `hw_bsp.cpp:449-462,632-655`; `hw_walls.cpp` cull branch ([S] 03 B1-B3, B12, B13) | Fix or retire before the new path's reference image is chosen | — | — | none | small CPU; image bugs |
| F-SC-2 | Render→script writes from the scene: billboard `drawPos`, heat decay | Live on main | [F] `hw_drawinfo.cpp:3644` (`bb.drawPos`), `:3233-3261` (`SetupHeatmap` decays `HeatIntensity`, read by ZScript `HeatmapAt`) | Avatar lane (handoff note 2); move to the sim or make it per-frame-independent | — | `hw_drawinfo.cpp` heat code (avatar lane note) | **Machine/eye/frame-rate dependent values reach ZScript.** A netplay hazard if any mod reads them in play scope | 0 |
| F-SC-3 | Per-draw state cost (2 KB `StreamData` per `Apply`) | Live | [F] `hw_renderstate.h:213` (`uFlatGlowLines[64]` at `:311`) | Phase 1 (another area) | — | — | none | [M] draw-call state 0.93 ms/scene; the mesh cut it about in half (prev. measured) |

---

## Findings

### 1. What the render lane has built, exactly

**Switch and scope.** `r_newrenderer` defaults to 0 and is not archived ([FR] `hw_levelresidency.cpp:137`
`CVAR(Int, r_newrenderer, 0, 0)`). 1 = static walls of non-sky sectors; 2 = plus sky-ceiling sides (their sky portal
walls are kept and replayed); 3 = plus static flats. Dev switches: `r_levelmesh_verify`, `_sidescan`, `_clock`,
`_drawall` (all 0); `r_levelmesh_foldback` is **1** (`:155`). Files: `hw_levelresidency.{h,cpp}` (5,152 lines),
`hw_levelstatic.{h,cpp}`, `common/.../hw_levelbuffers.{h,cpp}`, `levelmesh.vp`, a `LEVEL_MESH` block in `main.fp`,
`EFF_LEVELMESH`, set 1 bindings 16/17, and `r_snapshot.{h,cpp}` for stage 6.

**Where it enters.** One call per scene, before the walk, on the main thread:

```
[FR] hw_bsp.cpp:1464   mLevelMeshScene = LevelResidency_BeginScene(this);
```

`BeginScene` ([FR] `hw_levelresidency.cpp:4221-4321`) refuses portal, fullbright, OoB, ortho, skybox and mirror scenes
(`:4229-4233`). It builds on a new level or when a global input moves (rate-limited to once per 35 frames, `:4242-4248`).
At the first scene of each frame it clears last frame's evictions, runs `RunChangePass` and rewrites the light records
if the light key moved (`:4251-4275`). It then reserves this scene's region of a per-frame index buffer
(`:4286-4313`).

**What it draws.** Static wall parts (TOP / one-sided MID / BOTTOM) and static flat planes, opaque only. Static means the
classifier ([FR] `hw_levelstatic.cpp:69-88`) found none of: polyobject, line portal/horizon/mirror, translucent mid,
heightsec, 3D floor/light list, sector portal, sky (accepted at ≥2 for sky ceilings), animated base texture,
warped/canvas/translucent texture. On top of that the build refuses lightmaps, light lists, styles, non-default
shaders and parts over the slot bound. The vertex shader reads per-surface data by surface id from SSBOs, collapses
evicted surfaces to a point (`levelmesh.vp:236`), and culls a flat seen from its closed side against the *view's own*
camera (`:222-231`). One `SetMaterial` + `DrawIndexed` per non-empty material group, walls before `GLDL_PLAINWALLS`
and flats before `GLDL_PLAINFLATS` ([FR] `hw_drawinfo.cpp:3830,3835`; `hw_levelresidency.cpp:4675-4685`). Dynamic lights
are **off** in mesh draws (`state.SetLightIndex(-1)`, `:4667`, `:4903`): any side or flat a light touches is evicted.

**What it does not draw** (all stay on the old path, through the walk): moving or changed sectors' walls and flats,
lit surfaces, decal-carrying and damaged surfaces, dithered surfaces, every refused class above, masked two-sided
mids, translucent anything, sky planes, portals and their scenes, render hacks, sprites, particles, decals, models,
weapons, HUD, and every camera/portal/skybox scene.

**How it updates movers.** By eviction plus fold-back, Helion's model. A once-per-tic compare (`RunTicPass :3681`,
triggered when `gametic`, `Level->LocalTimer` or the dynamic-light state changed, `:3946`) checks every sector against a
264-byte shadow. Sectors with an active interpolation are re-checked every frame in between (`RefreshMovingSectors
:3892`). A side whose sector or vertex height list changed is evicted to the old path the moment the walk reaches it
(`SectorChanged | VertexChanged`, `:4371`). Once the sector has been still for one tic, `FoldBack :3637` re-runs
`HWWall::Process` for up to 500 sides a pass (`:837`) and rewrites their slots with `SetSubData`. Light changes do not
evict: they patch the 320 B light record (`RunLightPatch :2994`). Mesh positions are absolute and carry no prev/cur,
so there is **no GPU interpolation**. Interpolated movers are always old-path.

**Per tic or per frame?** Change *detection* is per tic, with movers re-checked per frame. Everything the frame sees
is per scene, per frame: the BSP walk, the clipper, the per-side decision (`SkipWallDecision :4327-4407`, including a
`WallCache_SideFastHash` of each reached side once a frame, `:4371-4376`), the sky portal replay (`:4388-4401`), the
reached-list sort, and the index copy (`MakeSceneRanges :4522`).

**Multiview-ready?** The shader is: it writes `hwViewIndex = gl_ViewIndex` (`levelmesh.vp:116-119`) and uses the
per-view prelude macros (`ViewMatrix`, `ProjectionMatrix`, `uCameraPos`). Under MV there is one scene, so one index
region. The CPU side is not: the draw list is the walk's reached list, which under MV comes from eye 0's walk with no
IPD margin, the same gap as the old path. In non-MV VR, each eye is a scene and takes one of
`FRAME_INDEX_SCENES = 2` regions ([FR] `hw_levelbuffers.h:78`). `BeginScene` does not test `mainview`, so a camera
texture rendered first ([F] `hw_entrypoint.cpp:719-730` before the main view) takes a region and eye 2 falls back to
drawing the whole static buffer (`:4302-4313`). No headset or MV run of the mesh is recorded.

**Status of the numbers (prev. measured, flat, no mods, `r_dynlights 0`).** The new path is slower than the old one in
every recorded pair: 1112x 76.2 → 67.2 fps; 1112y 70.1 → 60.8 fps ([FR] `HANDOFF_RENDER_2026-11-11.md`). The CPU
moves (1112r): walls 2.56 → 1.05 ms and draw calls 0.76 → 0.37 ms, but bsp 0.56 → 2.49 ms, 1.42 of it the sector
compare, later 1.43 ms a pass after the shadow rewrite (1112s). GPU equals the old path only when drawn through the
walk's list; drawing everything costs +4.2 ms of GPU wait (1112q). Image: view 1 identical; view 2 differs in 52-58 px
in every build since 1112j, still unexplained.

### 2. The lane's own plan says stage 3 as written will not pay

`[FR] docs/plans/STAGE3_VISIBILITY_PLAN.md` Q1-Q3 states it plainly. The walk cannot be skipped for static sides,
because the clipper's occlusion comes from those same walls (`SafeAddClipRange`, [FR] `hw_bsp.cpp:682,716`), and
because the walk is the only producer of sprites, particles, evicted/non-static surfaces, portal coverage, hacks,
`CheckUpdate` and the automap bits ([S] 03 §7, items 1, 5-11). Under Q1 (a) "walk stays", 3a (CPU block cull,
frustum-only) draws more than today's walk-culled list and is "expected to be slower than 2d on Sunder MAP15". The CPU
it can win is "at most the walk's per-static-side decisions and the reached-list index build (1.05 + 0.07-0.16 ms)". 3c
(indirect) pays only with bindless. 3d (Hi-Z) replaces an occlusion test the walk already does. The owner put the lane
on standby until he answers Q1-Q4.

**Inference.** On the code, the playbook's Phase 5 (walk retired) is Q1 (b), and Q1 (a) is a dead end for speed. The
lane's own audit agrees. Q1 (b) needs two things the lane has not designed:
- a non-walk finder for actors and for the surfaces the mesh does not hold: the per-sector `touching_renderthings` of
  visible cells plus `sectorportal_thinglist` / `lineportal_thinglist` ([S] 04 §8), or a render blockmap like
  Helion's `IterateBlockmap`;
- a home for portals, hacks and sky.

### 3. How it maps to the playbook phases

- **Phase 3.** Not started and not planned. The lane chose static residency with a per-frame walk (Helion's static
  path plus GPU culling later), not a tic-rate `TicScene`. The playbook's own accounting (Phase 3 C) says a 360° tic
  build costs about what 90 frustum builds do; its value is getting the build off the frame. That value comes either
  from Phase 3 + Phase 8, or from the lane's route: P4 + P5 + P7 make the per-frame CPU small, then stage 6 moves what
  is left. The parts of P3.D that matter regardless are plane prev/cur on the GPU (keeping movers resident) and
  `D_Render` no longer writing live sectors ([F] `d_main.cpp:700,708`). The second is also a stage-6 requirement
  (`RENDER_THREAD_SHADOW_WORLD.md` rule 5).
- **Phase 4.** About a third done: P4.1, P4.2, P4.4 and P4.9 are partial or done for the plain classes. Not done:
  movers stay resident (P3.D), material slots (P4.3), 3D floors (P4.5), render hacks (P4.7), decals (P4.8), indirect
  (P4.10). The lane's "light by index, static with eviction" is Helion's model. The playbook's "patch tables, never
  vertices; movers via plane index" is the one that keeps Sunder's lifts on the mesh.
- **Phase 5.** Not started. Blocked on the owner's Q1/Q2.
- **Phase 7.** Not started in the lane, by contract: `hw_sprites.cpp`, `models.cpp`, `hw_weapon.cpp` and `vr_*` are
  frozen for this lane ([F] `RENDER_LANE_HANDOFF_2026-11-09.md` §1). The lane's own route is a new world-actor path
  fed by `FSnapActorRec`, with the old path kept for the rig ([FR] `HANDOFF_RENDER_2026-11-11.md` "Next, in order" item
  3). The fork already has instanced GPU particles, mesh particles and a debris pool (P7.7 row). CPU particles and
  every actor sprite are still one `HWSprite` + one sorted draw + one CPU light sum each ([F] `hw_sprites.cpp:320`,
  `hw_drawlistadd.cpp:136-146`).

### 4. Reading the [M] scene-build split honestly

[M]§1 with the modlist: `cpu_scene_ms` 4.35. Splits: BSP 0.43, clip 1.24, walls 2.56, flats 0.75, sprites 0.62,
draw-call 0.93. The splits sum to 6.53 ms, more than the 4.35. **Inference from code:** in these flat runs
`gl_multithread` is on ([F] `hw_bsp.cpp:46`), so the scene worker runs `SetupWall`/`SetupFlat`/`SetupSprite` in
parallel with the main thread's walk. The columns add worker and main time: `walls = SetupWall + RenderWall`
([FR] `hw_perftrack.cpp:541-543`), `scene = ProcessAll + RenderAll` (`hw_clock.cpp:167`). `drawcalls` is nested inside
`RenderAll`. In VR the walk is single-threaded by default ([F] `hw_bsp.cpp:1438-1439`, `vr_scene_multithread` false at
`hw_cvars.cpp:278`), so the worker's share runs serially.

**Inference:** VR per-scene CPU is between 4.35 ms (no serial penalty) and ≈6.5 ms (every Setup column fully serial),
at the map start with 192 sprites.

- Non-MV VR frame: 2 × (4.35 to 6.5) = **8.7 to 13 ms** of scene build at the map start.
- In fights: prev. measured flat-view scene build 5-11 ms per scene (Abandon MAP13 five vantages, 1112f) gives
  **10-22 ms per non-MV VR frame**.

None of this has been measured in a headset ([M]§5).

### 5. The GPU side, and what the lane cannot touch

[M]§1, modlist, flat 5.2 MP: GPU wait 25.87 ms.
- Named groups: opaque 6.15 (walls 3.09, flats 1.21, models 0.84), exposure 1.38, bloom 3.23, BloomBoost 1.60 + 1.64.
  Together ≈14.0 ms.
- **≈11.9 ms is not attributed** to any named group: translucent sprites, particles, smoke, portals, 2D and the like
  (Inference: 25.87 − 14.0).

Phases 4 and 5 touch only the walls+flats 4.30 ms, and the mesh shades the same pixels with an SSBO read per fragment
(`main.fp` `LEVEL_MESH` turns ~25 uniforms into record reads, [S] 09 B.4). Prev. measured: equal to the old path at
best.

**Inference (linear in pixels):** Quest 2 at 1.0x is 1832×1920×2 ≈ 7.0 MP, 1.35x the flat window, so the modlist's
GPU frame would be about 35 ms against a ≈9.5 ms app budget ([P] 21 §7). Post-process alone (7.85 × 1.35 ≈ 10.6 ms)
breaks the budget. That is lighting/post-process work (other areas), plus Phase 7's translucency (P7.3/P7.4), which
could hit the unattributed 11.9 ms. VDXR's actual render resolution is unknown (Open questions).

### 6. Fork-specific hot spots in the scene build that the lane must keep in mind

- **Distance-cull impostors** ([F] `hw_bsp.cpp:449`, default 4000; branch `:632-655`). The mesh path draws the real
  wall instead ([FR] `hw_bsp.cpp:660-666`), so at the owner's default the two paths differ by design. The lane's image
  gate always passes `+gl_line_distance_cull 0` ([FR] `STAGE2_STATIC_LEVEL_PLAN.md` "How to run it"). The old
  impostor has its own bugs: one per seg, ceiling texture, uninitialised light ([S] 03 B1-B3, B12, B13).
- **Render→script writes per scene**: billboard `drawPos` ([F] `hw_drawinfo.cpp:3644`) and heat decay
  ([F] `:3255-3257`). Both run per eye/portal and both reach ZScript. They are not the lane's to fix (handoff notes),
  but they must be off the render thread before stage 6.
- **`D_Render` writes live sim structs every frame**: `DoInterpolations`/`RestoreInterpolations` ([F]
  `d_main.cpp:700,708`) rewrite plane D/texZ and 3D-floor attachments ([F] `r_interpolate.cpp:470-499`).
  `P_FindParticleSubsectors` and `PO_LinkToSubsectors` relink every subsector every frame (`:701-702`). All three
  block a render thread on the old path.
- **Dynamic lights evict.** With 122 lights at the modlist's map start [M]§1 and BD22 muzzle/effect lights in fights,
  the static path's coverage on the owner's real settings is unmeasured. The only data is Abandon 2c-1 (15% of
  reached static sides evicted for lights) and the 1112x eviction counts. The lane benchmarks with `r_dynlights 0`.
- **Decals and the fork's surface damage evict** (`:4359,4362`, flats `:4452`). BD22 gore decals accumulate over a
  fight, so eviction grows with play time.
- **Multithread gates**: with the mesh on, the experimental wall workers are disabled ([FR] `hw_bsp.cpp:650,743`).
  In VR `vr_scene_multithread 0` keeps everything on the main thread anyway.
- **Memory**: 158-180 MB GPU (prev. measured) plus equal staging copies kept in host memory
  (`vk_hwbuffer.cpp:73-97`) on a 16 GB host that already runs 53k thinkers.

### 7. Why the render lane cannot deliver 90 Hz on the modlist alone

[M]§2, Sunder MAP15 modlist, per tic: thinkers 130.80 + interp reset 9.02 + WorldTick 4.07 + particles 0.72 = **144.6
ms**. That is 5.1x the 28.6 ms tic budget.

- **Single-thread form** (today's loop, [F] `D_DoomLoop`/`D_Display`): at 90 Hz, 35/90 ≈ 39% of frames carry a tic, so
  a frame with a tic must fit tic + render in 11.1 ms. The tic budget is then ≈3 ms ([P] 21 §7), 48x below today.
- **With a render thread** (stage 6): the tic only has to fit 28.6 ms (≤20 ms target), so it must still fall ≈5-7x.
  The render thread costs, per the lane's measurements:
  - ≥13 ms a tic on the old draw path (whole-world images, [FR] `RENDER_THREAD_SHADOW_WORLD.md`);
  - 5.5 ms a write for compact records (Abandon, 17,481 actors), or under 1 ms if split across cores (the lane's
    inference).
- **Consequence.** The render thread is only viable once the level comes from the new path's own buffers *without the
  live walk* (Q1 (b)) and world actors come from records (P7.1-2). Both are render-lane work. **So the render lane's
  real contribution to 90 Hz is making the render thread possible, plus multiview.** Trimming the 4.35 ms scene build
  is secondary.

---

## 90 Hz contribution

**Budget for this area (Inference from [P] 21 §7: 11.1 ms frame, render-thread CPU ≤ 7 ms with a target of 4, GPU ≤
9.5 ms).**
- Scene build (walk/visibility + walls + flats + sprites + draw submission, both eyes): ≤ 3 ms per frame on the render
  thread, leaving ~4 ms for post, 2D/HUD, psprites, world weapons, avatar models and submission.
- Opaque world GPU: ≤ 3 ms for both eyes ([P] 21 §7 "Opaque world 3.0 ms"); masked + sprites ≤ 1 ms.
- Sim snapshot write (this lane's half of stage 6): ≤ 1 ms per tic ([P] 22 Phase 8 `snapshot_ms`).

**What the fork has today.**

| Item | Today (derivation) | Budget | Gap |
|---|---|---|---|
| Scene build, VR non-MV, map start, modlist | 2 × 4.35-6.5 = 8.7-13 ms ([M]§1, §4 above) | ≤ 3 ms | 5.7-10 ms |
| Scene build, VR MV, map start | 4.35-6.5 ms | ≤ 3 ms | 1.4-3.5 ms |
| Scene build, VR non-MV, fight | 2 × 5-11 = 10-22 ms (prev. measured flat, 1112f) | ≤ 3 ms | 7-19 ms |
| Opaque walls+flats GPU, both eyes | 4.30 × 1.35 ≈ 5.8 ms (Inference: [M]§1 scaled to 7.0 MP) | ≤ 3 ms (all opaque incl. models: 6.15 × 1.35 ≈ 8.3) | ≈3-5 ms |
| Snapshot write | 5.5 ms compact (prev. measured), no reader | ≤ 1 ms | 4.5 ms + a reader |
| Render thread | none (stub) | must exist | all of it |
| Sim tic (not this area, gating everything) | 144.6 ms ([M]§2) | ≤ 28.6 hard / ≤ 20 target | ≥ 116 ms |
| `r_newrenderer` 3 vs 0 | +1.8 to +2.2 ms/frame flat, no mods, no lights (prev. measured) | ≤ 0 | the path is a regression today |

**What each render-lane step buys on the modlist (Inference, per frame in VR, from [M]§1).**

1. **Multiview on** (stage 4 / F-RL-9): −4.35 to −6.5 ms CPU per frame (one fewer scene build). It also halves
   per-eye CPU work outside the walk: sprite lighting, bones, `StartScene` uniforms. This is the largest single
   render-CPU lever, and it needs no new geometry path. GPU: opaque pixel work unchanged, per-draw vertex work once.
2. **`vr_scene_multithread 1`** (existing cvar, F-RL-10): between 0 and ≈2 ms per scene off the main thread
   (Inference: the 6.53 − 4.35 = 2.18 ms the flat split shows running in parallel). Measure before trusting it; it
   carries B7/B8 hazards ([S] 03 §6.1).
3. **Walk-free main view + resident world** (Q1 (b): P4 completion + P5): the playbook's target is ≤ 1 ms per frame
   for level CPU. From [M]: bsp 0.43 + clip 1.24 + flats 0.75 + the walls/draw-call main-thread share ≈ 4.35 − 0.62
   (sprites) − ~1 (residual cull/submit) ≈ 2.7 ms per scene saved. That is 2.7 ms/frame under MV, 5.4 non-MV.
4. **World-actor instancing** (P7.1-P7.4, in new files): [M] sprites 0.62 ms/scene → ~0.05/frame (playbook target),
   so ~0.6 ms/frame under MV at the map start. In fights 0.8-4.3 ms per scene (prev. measured), so **up to ~4 ms/frame
   under MV**. More important: P7.1-2 is a prerequisite for the render thread.
5. **Clustered lights** (stage 5 / Phase 6, another area): keeps lit surfaces on the mesh. Without it, step 3's saving
   shrinks by the evicted share (≥15% prev. measured, likely more with BD22 lights).
6. **Render thread** (stage 6): takes the whole render CPU off the tic. This is what makes 90 Hz reachable once the tic
   is ≤ 20 ms.

**Order of render-lane work.**

*For the 90 Hz-on-the-modlist goal* (each step needs the owner's word, under `[FR] CLAUDE.md` "STICK TO THE PLAN";
these are reorderings of the lane's own stages, not inventions):
1. Leave `r_newrenderer 0` in every owner-facing build. Record that stage 2's acceptance target ("Helion's
   conditions": no mods, `r_dynlights 0`, flat) is not the 90 Hz target.
2. **Stage 4 (multiview) before stage 3**, on the old path. Close the four known gaps plus the eye-0 culling margin
   (P3.C), and gate F-RL-4 on `mainview`. This is the only render-lane step that pays on the modlist without a new
   pipeline.
3. Owner answers **Q1 = (b)** (a walk-free main view) and **Q2 = queue at tic start** (contract rule 4). With (a),
   stage 3 has no speed (§2).
4. **World-actor path** (P7.1-P7.3; records → instanced sprites, voxels and CPU particles; the old path for anything
   with `modelData` or fork actor features). It pays in fights and unblocks stage 6.
5. **Stage 3 as (b)**: CPU block cull (lavapipe) → GPU frustum/Hi-Z, indirect draws, with portals, hacks, sky and
   non-static surfaces found through a render grid.
6. **Stage 6**: render thread, with the snapshot reader and the write split across cores to ≤ 1 ms.
7. **Stage 5**: bindless + clustered lights (removes F-RL-3's light evictions); then P4.8 decals.

*For the long-term goals* (beat Helion's no-mod benchmark, slaughtermaps at any res): the lane's existing order
(stage 3 → 4 → 5 → 6 → 7), but with stage 3 as Q1 (b). Then P3.D (plane prev/cur so movers stay resident), P4.3
material slots, P4.5 3D floors, P4.7 render hacks, P5.3 portals, P5.4 skybox cubemap, P4.10 indirect, foveation.
Phase 3's `TicScene` is not needed on this route.

**Lane rules that bound all of this.**
- Models, the avatar, worn items and weapons (`models.cpp`, `vr_*.cpp`, `hw_sprites.cpp`, `hw_weapon.cpp/h`, the
  `VRRig_*` calls in `hw_drawinfo.cpp`) keep drawing through the old path "once per view per frame with the same
  `r_viewpoint` and `ticFrac`" (handoff §1). The P7 path must route those actors back to the unedited `HWSprite::Process`.
- Everything goes behind `r_newrenderer` (0).
- Nothing new reads the sim beyond the event-contract fields (`RENDERER_REBUILD.md` "event contract").
- Draw-side randomness stays on `FCRandom`.
- Lavapipe fallbacks for indirect-count, bindless and Hi-Z (handoff §6).
- Merge gate: `bash tools/netplay/suite.sh` ALL GREEN with the path on and off. Never run so far (F-RL-11).

---

## Open questions

Each needs a run on the owner's PC through the bridge. Exe `doomxr_rend1113a.exe` or later; flat window as in [M];
`-rngseed 1113 +vid_vsync 0 +vid_maxfps 0 +cl_capfps 0 +perf_track 1`.

1. **Real two-eye scene cost without a headset.** `boottest -Exe doomxr_rend1113a.exe -Zdl <owner list> -Map MAP15
   -Tics 700 -ExtraArgs "+vr_mode 3 +gl_multithread 0 ..."`. `vr_mode 3` is `VR_SIDEBYSIDEFULL` ([F]
   `hw_vrmodes.h:47`): two scene builds per frame, and `gl_multithread 0` reproduces VR's serial walk. Compare
   `cpu_scene_ms` with [M]'s 4.35. Then repeat with `+gl_multithread 1` to size F-RL-10. It settles the 8.7-13 ms range
   in §4.
2. **Fight-time scene build on the modlist.** `render_vantage13_300.cfg` on Abandon MAP13 with the owner's current ZDL
   (rebuild with `make_owner_zdl.py`), `r_dynlights 1`. Read `cpu_bsp/clip/walls/flats/sprites_ms` and `stat
   renderstats` "Sprites:". It gives [M]-grade numbers for the 5-11 ms lead.
3. **Does the static path win anything on the owner's real settings?** Same Sunder MAP15 and Abandon MAP13 runs with
   `r_dynlights 1`, `gl_line_distance_cull 4000` (owner default), `r_newrenderer 0` vs `3`. Then `levelmeshstats` at
   tic 600 for Skipped/Evicted by reason, `PassMS`/`PassMaxMS`, `FoldMaxMS`. No modlist pair with lights on exists.
4. **Unattributed GPU time (≈11.9 ms of 25.87).** Modlist Sunder with `r_perflog 10`. Read the GPU groups
   `scene.translucent`, `fx.gpuparticles`, `fx.depthread`, `fx.meshparticles`, `scene.psprites`,
   `scene.worldpsprites`, `scene.vrextras` ([S] 04 §9.6). This decides whether P7.3/P7.4 (translucency) matter more
   than P4/P5 on the GPU.
5. **Multiview in practice.** The bridge cannot drive a headset. Either the owner runs a new `RENDER TEST <date>.bat`
   with `+vr_openxr_multiview 1` and `perf_track 1` (start-up setting; it changes pass keys, [FR] `RENDERER_REBUILD.md`),
   or the Linux rig runs Monado + lavapipe with the cvar on, for correctness only. Read the bench header "XR:
   multiview=… active=…" ([S] 04 §9.1); also check whether the owner's ini already has it on (`CVAR_ARCHIVE`).
6. **VDXR render resolution and refresh** (the GPU scaling in §5 assumes 1.0x Quest 2). Read the OpenXR swapchain size
   from the start-up log of any owner headset session (`vk_openxrdevice.cpp` swapchain creation print).
7. **Camera-texture hazard F-RL-4.** Non-MV VR (or `vr_mode 3`) on a map with a camera texture, `r_newrenderer 3`.
   `levelmeshstats` "scenes found no room" / `ScenesNoRoom` > 0 confirms it.
8. **Netplay gate for the branch.** Netplay lane: `bash tools/netplay/suite.sh` on a `renderer` build, with
   `+r_newrenderer 3` and without. Never run (F-RL-11).
9. **Snapshot write split.** `r_snapshot 1` + `snapshotstats` on Abandon MAP13 modlist after the writer is split
   across cores (the lane's planned next change). It must come in ≤ 1 ms for stage 6 to fit [P] 22's Phase 8 target.
