# U1 — UZDoom 5.0 hardware renderer: game-side scene setup

UZDoom 5.0 rebuilds the whole scene on the CPU every frame, once per viewpoint. Each viewpoint gets a full front-to-back BSP walk with a 1-D angular clipper. Every visible linedef tier becomes an `HWWall` object, and the visible walls, sprites and decals get new vertices written into a 2M-vertex streaming buffer. Floors and ceilings are the exception: they come from a level-load-time static index buffer that is patched only when a plane moves. Draw lists are not batched. Every wall piece, flat section, sprite and decal is one draw call, with a per-draw material and stream-data update. Opaque lists are left in BSP order unless `gl_sort_textures` is on, and all billboard sprites, opaque or not, go into the translucent list and its recursive BSP-style translucency sort. The only parallelism is one worker thread (`gl_multithread`, default on): the main thread walks the BSP and clips, and the worker builds walls, flats and sprites. On the Vulkan and GL≥4.5 backends, though, wall vertex generation, dynamic-light list building and sprite quad generation are moved to draw time on the main thread. Stereo runs `ProcessScene` once per eye. Each eye repeats the BSP walk, clipping, draw-list build, sorting, portal recursion and post-processing, so for VR the CPU cost of a frame is roughly ×2 per eye pair and ×(1+number of visible scene portals).

---

## 1. Frame flow: entry point → per-eye loop → DrawScene

**Top entry.** `RenderView(player)` resets the streaming vertex buffer once per frame: `screen->mVertexData->Reset()` [U] src/rendering/hwrenderer/hw_entrypoint.cpp:314-315. It also clears the light, bone and viewpoint buffers [U] hw_entrypoint.cpp:346-348. It then renders every camera texture "used in the last frame" for every level. Each one is a full `RenderViewpoint` call [U] hw_entrypoint.cpp:375-386. Only then does it render the main view [U] hw_entrypoint.cpp:403.

**Per-viewpoint, per-eye loop.** `RenderViewpoint` calls `R_SetupFrame` once (interpolation, view angles) [U] hw_entrypoint.cpp:101, plus the shadow-map update when one is needed [U] hw_entrypoint.cpp:103-110. Then it loops over the eyes:

```cpp
auto vrmode = VRMode::GetVRMode(mainview && toscreen);
const int eyeCount = vrmode->mEyeCount;
screen->FirstEye();
for (int eye_ix = 0; eye_ix < eyeCount; ++eye_ix) {
    ...
    auto di = HWDrawInfo::StartDrawInfo(mainvp.ViewLevel, nullptr, mainvp, nullptr);
    ...
    vp.Pos += eye.GetViewShift(vp.HWAngles.Yaw.Degrees());
    di->SetupView(RenderState, vp.Pos.X, vp.Pos.Y, vp.Pos.Z, false, false);
    di->ProcessScene(toscreen);
    ... screen->PostProcessScene(...); ...
    di->EndDrawInfo();
```
[U] src/rendering/hwrenderer/hw_entrypoint.cpp:126-188

So each eye gets a fresh `HWDrawInfo`, a full `ProcessScene`, and its own post-process pass [U] hw_entrypoint.cpp:168-181. Nothing from the scene build is shared between the eyes.

**StartDrawInfo / StartScene.** `HWDrawInfo` objects are pooled in `di_list` and reused [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:84-100. `StartScene` clears the three static clippers (main, vertical, radar). There is only one set, because "all scenes are processed sequentially" [U] hw_drawinfo.cpp:123-136. It then copies the viewpoint and calls `ClearBuffers()` [U] hw_drawinfo.cpp:128-191. `ClearBuffers` memsets three flag arrays sized to the level (sections, subsectors, nodes) [U] hw_drawinfo.cpp:235-247. It also clears four `TMap`s and the missing-texture arrays [U] hw_drawinfo.cpp:219-232. `Release()` calls `ClearBuffers()` again [U] hw_drawinfo.cpp:95-100.

**DrawScene ordering** [U] hw_drawinfo.cpp:991-1046:
1. `CreateScene()`: BSP walk, then render hacks, which fill the draw lists [U] hw_drawinfo.cpp:1017-1026.
2. `RenderFirstSkyPortal()`: one sky portal drawn without a stencil, before the scene [U] hw_drawinfo.cpp:1030.
3. `RenderScene()`: opaque passes [U] hw_drawinfo.cpp:1032.
4. SSAO for this scene if it is enabled [U] hw_drawinfo.cpp:1034-1038.
5. `portalState.EndFrame()`: every remaining portal, recursively [U] hw_drawinfo.cpp:1042-1044.
6. `RenderTranslucent()` [U] hw_drawinfo.cpp:1045.

CPU scene building for a portal therefore starts only after the parent's opaque geometry has been submitted. Scene building and command submission are interleaved on the main thread.

**CreateScene** [U] hw_drawinfo.cpp:470-511 does the following:
- Adds the "behind the viewer" range to the clipper using `FrustumAngle()` [U] hw_drawinfo.cpp:473-475.
- Maps the vertex, light and bone buffers [U] hw_drawinfo.cpp:491-493.
- Calls `RenderBSP(Level->HeadNode(), …)` [U] hw_drawinfo.cpp:495.
- Runs the single-threaded hack passes `HandleMissingTextures`, `HandleHackedSubsectors`, `PrepareUnhandledMissingTextures` and `DispatchRenderHacks` [U] hw_drawinfo.cpp:501-504. The code comments that these "cannot be multithreaded … because all these depend on the global 'validcount' variable" [U] hw_drawinfo.cpp:497-499.
- Unmaps the buffers [U] hw_drawinfo.cpp:505-507.

### 1a. RenderBSP and the worker split

```cpp
multithread = gl_multithread;
if (multithread) {
    jobQueue.ReleaseAll();
    auto future = renderPool.push([&](int id) { WorkerThread(); });
    ... RenderBSPNode(node);
    jobQueue.AddJob(RenderJob::TerminateJob, nullptr, nullptr);
    Bsp.Unclock();
    MTWait.Clock(); future.wait(); MTWait.Unclock();
}
```
[U] src/rendering/hwrenderer/scene/hw_bsp.cpp:1024-1039

- `RenderBSP` increments the global `validcount` once per scene [U] hw_bsp.cpp:1022.
- The thread pool has exactly one thread: `ctpl::thread_pool renderPool(1)` [U] hw_bsp.cpp:50.
- The job queue is a static 300,000-entry array with atomic read and write indices and no overflow check [U] hw_bsp.cpp:82-107. Its comment says the largest count seen on one viewpoint is about 40,000 jobs [U] hw_bsp.cpp:84.
- When the queue is empty, the worker spins. On x86 it issues ten `_mm_pause` per poll. It never sleeps [U] hw_bsp.cpp:118-137.

**Inference:** on non-x86 builds the `#ifdef ARCH_IA32` block disappears, which leaves a tight spin. Every `renderPool.push` heap-allocates a `std::packaged_task` through `make_shared` and a `new std::function`, then takes a mutex and calls `notify_one` [U] src/common/thirdparty/ctpl.h:189-198. That cost is paid once per scene: per eye, per portal and per camera texture.

**RenderBSPNode** [U] hw_bsp.cpp:947-985 is the classic recursive walk:
- It descends the near child first.
- It culls the far child with `mClipper->CheckBox(bsp->bbox[side])`, unless the node is marked `SSRF_SEEN` by a sector-stack portal.
- It adds an ortho pitch check when the view is orthographic.
- The node coordinates are fixed-point, so the view position is kept in fixed point as well [U] hw_bsp.cpp:1014-1015.

**DoSubsector** (main thread) [U] hw_bsp.cpp:680-933 does the following in order:
- Rejects subsectors outside `CurrentMapSections` [U] hw_bsp.cpp:697.
- Unclips the range when the subsector is `SSRF_SEEN` [U] hw_bsp.cpp:700-706.
- Calls `hw_FakeFlat` (the Boom heightsec copy). This must run on the main thread because the worker cannot create fake sectors [U] hw_bsp.cpp:709, [U] src/rendering/hwrenderer/scene/hw_fakeflat.cpp:234-238.
- Calls `CheckUpdate()` the first time a sector is reached in the scene, which patches the static flat vertices if the plane moved (§1b) [U] hw_bsp.cpp:794-797.
- Queues a `ParticleJob` [U] hw_bsp.cpp:800-812.
- Runs `AddLines` (clip each seg; §1c) [U] hw_bsp.cpp:814.
- Queues one `SpriteJob` per sector, the first time the sector is reached [U] hw_bsp.cpp:820-838.
- Queues one `FlatJob` per **section**, deduplicated by `section_renderflags` [U] hw_bsp.cpp:874-891.
- Queues `PortalJob`s for stacked-sector portals. These are deferred to the worker so that the portal list is touched by one thread only [U] hw_bsp.cpp:900-929.

**Worker dispatch** [U] hw_bsp.cpp:111-216:
- `WallJob`: resolves front and back fake sectors, then calls `HWWall::Process`.
- `FlatJob`: `HWFlat::ProcessSector`.
- `SpriteJob`: `RenderThings`.
- `ParticleJob`: `RenderParticles`.
- `PortalJob`: `AddSubsectorToPortal`.

The worker is the only writer to the draw lists, `Portals`, `Decals` and the missing-texture arrays while BSP runs. That is how the code avoids locks (Inference from the comment at hw_bsp.cpp:900-903 and the absence of mutexes: `grep -n "std::mutex\|lock_guard" -r src/rendering/hwrenderer` returns nothing).

### 1b. What is regenerated per frame vs cached

| Data | Lifetime | Evidence |
|---|---|---|
| Flat (floor/ceiling/3D-plane) vertices + indices | Built at map load (`CreateVBO` from maploader) into `vbo_shadowdata`/`ibo_data`, fan-triangulated into indexed triangles per subsector (earcut for hole subsectors), grouped per section | [U] src/maploader/maploader.cpp:3232; [U] src/rendering/hwrenderer/hw_vertexbuilder.cpp:30-72, 94-110, 299-340 |
| Flat vertex Z on plane movement | `CheckPlanes` compares `GetPlaneTexZ` with the cached `vboheight[pipelinepos]`. On a change, `UpdatePlaneVertices` recomputes Z for all of that plane's vertices and uploads the range. It runs once per sector per scene, on the main thread, and also covers heightsec and 3D-floor models | [U] hw_vertexbuilder.cpp:406-421, 441-469 |
| Wall vertices | Allocated anew from the streaming buffer every time a wall is put or drawn (`MakeVertices` → `AllocVertices`): 4 vertices, more when `gl_seamless` adds T-junction splits | [U] src/rendering/hwrenderer/scene/hw_walls_vertex.cpp:279-288, 187-203 |
| Sprite quads, decals, portal caps, render-hack planes, skybox flats | Per-frame `AllocVertices` | [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:624-640; [U] hw_decal.cpp:415-421; [U] hw_portal.cpp:194-215; [U] hw_renderhacks.cpp:143-158; [U] hw_flats.cpp:435-440 |
| Dynamic-light data per surface | Rebuilt for each surface draw (`UploadLights`) | §2c |
| Draw items (`HWWall`/`HWFlat`/`HWSprite` copies) | Allocated from `RenderDataAllocator` (an FMemArena with 1 MB blocks) and freed in bulk when the outermost DrawInfo ends, i.e. after each eye | [U] src/rendering/hwrenderer/scene/hw_drawlist.cpp:30-35, 752-781; [U] hw_drawinfo.cpp:199-208 |

The streaming buffer holds `BUFFER_SIZE = 2000000` vertices [U] src/common/rendering/hwrenderer/data/flatvertices.h:100-101. Each `FFlatVertex` is 8 floats (32 B) [U] flatvertices.h:35-41. Allocation is a single `fetch_add` [U] src/common/rendering/hwrenderer/data/flatvertices.cpp:138-148. Running out is a fatal error [U] flatvertices.cpp:142-146. The buffer is reset only once per frame [U] hw_entrypoint.cpp:315, so camera textures, both eyes and every portal all share it.

**Inference:** the pointer is read with `GetBuffer()` *before* the `fetch_add` [U] flatvertices.cpp:140-141. That is correct with one allocating thread, but it would race if a second worker were added.

### 1c. Clipper

`Clipper` is a doubly linked list of occluded **angular ranges** (pseudo-angles). Nodes come from a free list backed by an FMemArena [U] src/rendering/hwrenderer/scene/hw_clipper.h:31-97. Vertex angles are memoised per vertex with a `starttime` stamp [U] hw_clipper.h:186-190.

`AddLine` handles each seg as follows [U] hw_bsp.cpp:257-403:
- It backface-culls by angle order [U] hw_bsp.cpp:303-306.
- It tests visibility with `SafeCheckRange` [U] hw_bsp.cpp:324-327.
- It adds the seg as an occluder when it is one-sided or `WALLF_BLOCKRENDERING` [U] hw_bsp.cpp:338-342, or when `hw_CheckClip` says the back sector closes the opening [U] hw_bsp.cpp:362-370.
- It deduplicates per **linedef** with `linedef->validcount` before queuing a `WallJob` [U] hw_bsp.cpp:381-401. A wall is therefore one linedef tier spanning the whole linedef, not one seg. `HWWall::Process` sets `fracleft = 0`, `fracright = 1` and uses the linedef vertices [U] src/rendering/hwrenderer/scene/hw_walls.cpp:2188-2208.

The clipper is purely horizontal: there is no vertical occlusion. The back-of-view culling angle is a heuristic of FOV, aspect and pitch. Above 46° of absolute pitch (or when the computed half-angle reaches 180°) it returns `0xffffffff`, which means no back culling at all [U] hw_drawinfo.cpp:377-397. `CreateScene` still passes that value to `SafeAddClipRangeRealAngles(yaw + a1, yaw - a1)`, which wraps to a range of about 2 BAM directly ahead, i.e. effectively nothing [U] hw_drawinfo.cpp:473-474; portal setups skip the call when `a1 >= ANGLE_180` [U] hw_portal.cpp:451-452, 594-595. Out-of-bounds viewpoints use a separate `OoBFrustumAngle` with its own pitch/aspect thresholds [U] hw_drawinfo.cpp:357-375, 379-382. A vertical clipper exists only for out-of-bounds or orthographic viewpoints (`VPSF_ALLOWOUTOFBOUNDS`) [U] hw_drawinfo.cpp:476-483, [U] hw_bsp.cpp:714-780.

**Inference:** in VR the user's head pitch routinely exceeds 46°, and then the whole level behind the viewer is traversed up to the first closed geometry. The clipper only rejects geometry hidden by solid one-sided walls or closed doors. Large open areas and outdoor maps pass almost everything.

---

## 2. Per-wall CPU work

### 2a. HWWall::Process

`HWWall::Process` covers [U] hw_walls.cpp:2141-2566:
- `gl_seamless` vertex-height recalculation when a vertex is dirty [U] hw_walls.cpp:2203-2207.
- Glow lookup [U] hw_walls.cpp:2248.
- Four `ZatPoint` plane evaluations for the front sector [U] hw_walls.cpp:2250-2253, and four more for the back sector [U] hw_walls.cpp:2305-2308.
- Horizon lines [U] hw_walls.cpp:2255-2260, and one-sided lines with portal, sky-transfer or mid texture [U] hw_walls.cpp:2266-2302.
- For two-sided lines: `SkyTop` and `SkyBottom` [U] hw_walls.cpp:2342-2343, the upper tier [U] hw_walls.cpp:2346-2417, the mid tier and fog boundary [U] hw_walls.cpp:2421-2491, 3D-floor blocks [U] hw_walls.cpp:2485-2490 and the lower tier [U] hw_walls.cpp:2493-2565.
- Each tier calls `GetLightLevel`, `CalcRelLight` and `TexMan.GetGameTexture` (e.g. [U] hw_walls.cpp:2287-2289, 2504-2506), then `DoTexture` [U] hw_walls.cpp:1338-1403.

`DoTexture` computes texture coordinates (`SetWallCoordinates`). Mirrors become portals [U] hw_walls.cpp:1386-1389. Otherwise it goes to `PutWall`, or to `SplitWall` when the sector has a 3D-floor light list [U] hw_walls.cpp:1396-1398.

**Splitting** happens in four ways:
1. *3D-floor light splitting.* `SplitWall` walks the sector's `lightlist`. When a light plane cuts the wall diagonally and the hardware has clip planes, it attaches the whole light list to the wall and defers the split to draw time [U] hw_walls.cpp:942-954. At draw time `RenderTexturedWall` issues **one draw per light slice**, each with its own `SetColor`, `SetFog` and `SetSplitPlanes` [U] hw_walls.cpp:293-320. Without clip planes it recursively bisects the wall on the CPU [U] hw_walls.cpp:803-895.
2. *Masked mid-texture slicing.* For textures with empty vertical areas, `GetAreas` splits the wall into several draw items [U] hw_walls.cpp:1665-1712.
3. *T-junction (`gl_seamless`) vertex insertion* in `CreateVertices`, using the neighbouring segs' `sidefrac` and the vertex `heightlist` [U] hw_walls_vertex.cpp:31-203.
4. *Translucency-sort splits* at draw time (§3).

**Fog boundaries.** `CheckFog` compares the front and back sectors' fade colours [U] src/rendering/hwrenderer/scene/hw_lighting.cpp:221-251. When there is a boundary, the wall is put a second time as `RENDERWALL_FOGBOUNDARY` into the translucent list [U] hw_walls.cpp:1622-1636. At draw time that wall is drawn with `EFF_FOGBOUNDARY`, depth bias, and draw-buffer attachments disabled [U] hw_walls.cpp:120-135.

**Missing-texture render hacks.** When an upper or lower texture is absent, the worker records `AddUpper/LowerMissingTexture` [U] hw_walls.cpp:2408-2414, 2555-2562. After the BSP walk, the main thread runs flood-fill searches over neighbouring subsectors:
- Each search bumps `validcount` [U] src/rendering/hwrenderer/scene/hw_renderhacks.cpp:535-600.
- There is an O(n²) inner scan over `MissingUpperTextures` [U] hw_renderhacks.cpp:562-571.
- It generates "other plane" vertices plus a per-subsector light upload [U] hw_renderhacks.cpp:143-184, and flood-gap quads [U] hw_renderhacks.cpp:723-835.
- It inserts results into `TMap`s keyed by sector [U] hw_renderhacks.cpp:166-184.
- `DispatchRenderHacks` then creates hack flats [U] hw_renderhacks.cpp:42-75.
- Flood planes are drawn with a three-pass stencil sequence per gap (increment, draw, decrement) [U] src/rendering/hwrenderer/scene/hw_flats.cpp:241-295.

The decision depends on the view: `Planez > Viewpoint.Pos.Z` [U] hw_renderhacks.cpp:543.

**Decals.** For solid walls with `AttachedDecals`, `PutWall` calls `ProcessDecals` immediately. Each decal allocates an `HWDecal` from the arena plus four streaming vertices [U] hw_walls.cpp:625-637, [U] src/rendering/hwrenderer/scene/hw_decal.cpp:384-421, 430-445. When buffers are persistent, the wall's lights are computed here as well, because the decal copies `dynlightindex` [U] hw_walls.cpp:628-635, [U] hw_decal.cpp:396-399. Decals are drawn in `RenderScene` part 4 with one `SetMaterial` plus one draw each, and one draw per light slice when they sit on a 3D-floor light list [U] hw_decal.cpp:38-124, 131-159.

### 2b. Persistent-buffer backends defer wall work to draw time

`BuffersArePersistent()` is `hwcaps & RFL_BUFFER_STORAGE` [U] src/common/rendering/v_video.h:245. The flag is set unconditionally on Vulkan [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:164 and on GL ≥ 4.5 [U] src/common/rendering/gl_load/gl_interface.cpp:159-162. With it set:

```cpp
void HWWall::DrawWall(HWWallDispatcher*di, FRenderState &state, bool translucent)
{
	if (screen->BuffersArePersistent())
	{
		if (di->di && di->Level->HasDynamicLights && !di->isFullbrightScene() && texture != nullptr)
		{
			SetupLights(di->di, lightdata);
		}
		MakeVertices(!!(flags & HWWall::HWF_TRANSLUCENT));
	}
```
[U] hw_walls.cpp:364-373

`PutWall` does the same work at setup time only when buffers are *not* persistent [U] hw_walls.cpp:609-616. Flats follow the same pattern: `SetupLights` runs in `DrawSubsectors` when persistent [U] hw_flats.cpp:192-197, and in `PutFlat` otherwise [U] hw_flats.cpp:390-396. So do sprites, with `CreateVertices` in `DrawSprite` [U] hw_sprites.cpp:278-281 versus `PutSprite` [U] hw_sprites.cpp:610-614. On persistent backends that `CreateVertices` call sits inside the light-slice loop, so a sprite on a 3D-floor light list allocates 4 new vertices per slice [U] hw_sprites.cpp:243-287.

**Consequence (Inference):** on the Vulkan backend, the worker thread only computes coordinates and classifies items. Vertex writing and the per-surface light-list walk run on the main thread, inside the command-recording loop. For VR on Vulkan, the "multithreaded" setup therefore hides much less of the cost than its design suggests.

### 2c. Per-surface dynamic light lists

Lights are linked into per-section and per-sidedef `TMap<FDynamicLight*, unique_ptr<FLightNode>>` lists (`flat_dlist`, `wall_dlist`) [U] src/g_levellocals.h:56-57. This happens on the playsim side in `FDynamicLight::LinkLight` → `CollectWithinRadius` [U] src/playsim/a_dynlight.cpp:622-634.

`HWWall::SetupLights` iterates the wall's sidedef map. Polyobjects use the subsector section's map instead [U] hw_walls.cpp:427-490. For each active light it does the following [U] hw_walls.cpp:493-553:
- computes the distance from the light to the wall plane;
- projects the four wall corners into the light's tangent space and rejects the light when all four corners are outside on one side;
- calls `GetLight`, which checks the side and radius and then `AddLightToList` writes **16 floats** [U] src/rendering/hwrenderer/hw_dynlightdata.cpp:44-60, 66-151.

The result is copied with `UploadLights` [U] hw_walls.cpp:555 into the light buffer: a 1-vec4 header plus 4 vec4 per light, appended with an atomic `fetch_add` [U] src/common/rendering/hwrenderer/data/hw_lightbuffer.cpp:75-123. Capacity is 80,000 lights × 4 vec4 [U] hw_lightbuffer.cpp:22-31. On overflow it silently returns −1 and the surface is drawn unlit [U] hw_lightbuffer.cpp:109-122. The buffer is cleared once per frame [U] hw_entrypoint.cpp:346, so both eyes, all portals and all camera textures share its capacity.

Light data is never deduplicated: two surfaces lit by the same light each get their own copy (Inference from `UploadLights` copying `data.arrays` per call). The scratch `lightdata` is `thread_local` [U] hw_dynlightdata.cpp:26.

Flats do the same for the section's `flat_dlist`, with a side-of-plane test [U] hw_flats.cpp:142-184.

### 2d. Per-wall cost and allocation count (Inference)

**Per wall tier, no lights:**
- About 8 plane evaluations, 3 to 6 texture-manager lookups, `GetLightLevel` per tier, texture-coordinate math (~100–200 flops).
- A ~300 B struct copy into the arena. The size is estimated from the field list [U] src/rendering/hwrenderer/scene/hw_drawstructs.h:173-228: about 10 pointers, `glseg`, 2×4 texcoords, 2×4 glow-colour floats and so on.
- Two `TArray::Push`es (`walls` and `drawitems`) [U] hw_drawlist.cpp:752-757.
- 4–10 vertices × 32 B.

Estimated at about 0.3–1 µs per tier on a modern core. Add about 50–100 ns per light that touches the sidedef (plane distance, 4-corner projection, 64 B write). A 2,000-tier view with an average of 2 lights would cost roughly 1–2.5 ms per eye on the setup thread plus the draw-time thread, before any draw-call overhead.

**Allocations per scene (DrawInfo):**

*Arena (bump-pointer, no malloc in steady state):* one 16-byte-aligned block per wall, flat, sprite and decal item [U] src/common/utility/memarena.cpp:111-114. Blocks are recycled through `FreeAll` [U] memarena.cpp:139-149.

*Heap mallocs, roughly 15–20 per scene in steady state plus per-portal growth:*
- Four `TMap::Clear()` calls in `ClearBuffers`, each a free plus a `M_Malloc` of the node vector [U] src/common/utility/tarray.h:1326-1330, 1530-1557. `ClearBuffers` runs twice per DrawInfo [U] hw_drawinfo.cpp:95-100, 177, which makes **8 mallocs/frees**, plus regrowth whenever render hacks fill the maps.
- One `new` per portal object per scene [U] hw_walls.cpp:671-760, deleted in `EndFrame` [U] src/rendering/hwrenderer/scene/hw_portal.cpp:83-94. The portal's `TArray<HWWall> lines` and `mPrimIndices` grow from empty each time [U] src/rendering/hwrenderer/scene/hw_portal.h:87-92, 120-125.
- Two allocations from the ctpl task push [U] ctpl.h:190-191.
- `SortNode` heap nodes only on growth [U] hw_drawlist.cpp:53-60.

`TArray::Clear` keeps its capacity [U] tarray.h:779-786, so the draw-list arrays do not reallocate after warm-up.

---

## 3. Draw lists, sorting, draw calls, state changes

**List set** [U] hw_drawinfo.h:100-113: `PLAINWALLS`, `PLAINFLATS`, `MASKEDWALLS`, `MASKEDFLATS`, `MASKEDWALLSOFS`, `MODELS`, `TRANSLUCENT`, `TRANSLUCENTBORDER`.

Classification rules:
- **Walls:** translucent walls go to `TRANSLUCENT`. Otherwise the wall goes to masked or plain depending on the pass flag and the texture's `isMasked` [U] src/rendering/hwrenderer/scene/hw_drawlistadd.cpp:35-58.
- **Flats:** translucent or fogged flats go to `TRANSLUCENT` when they are 3D planes and to `TRANSLUCENTBORDER` otherwise. Translucent-texture flats are split into further cases. Masked flats go to `MASKEDFLATS` only for 3D planes and stacks [U] hw_drawlistadd.cpp:99-130.
- **Sprites:** every sprite that is not a model and not a wall sprite goes to `GLDL_TRANSLUCENT`, **even when it is fully opaque** (`PutSprite(di, hw_styleflags != STYLEHW_Solid)` passes `translucent = false` for solid sprites, but that flag only matters for models and wall sprites) [U] hw_drawlistadd.cpp:138-153, [U] hw_sprites.cpp:1459. Only non-translucent models and wall sprites that have an actor go to `GLDL_MODELS`; anything without an actor (particles, visual thinkers) always goes to `GLDL_TRANSLUCENT` [U] hw_drawlistadd.cpp:142-149.

**Opaque ordering.** With `gl_sort_textures` (default **false**) [U] hw_drawinfo.cpp:45, the opaque lists are drawn in insertion order, i.e. the order the worker received jobs from the front-to-back BSP walk [U] hw_drawinfo.cpp:531-538. With it on, they are `std::sort`ed by texture pointer, then by clamp flags for walls [U] hw_drawlist.cpp:716-743.

**Translucent ordering** (`DrawSorted`) [U] hw_drawlist.cpp:629-708, 871-948:
- Builds a linked list of `SortNode`s [U] hw_drawlist.cpp:150-175.
- Recursively picks a flat as the splitter, or else the wall nearest the median `ViewDistance` [U] hw_drawlist.cpp:183-234.
- Partitions the remaining items left, right or equal by Z or by `PointOnSide`.
- Splits walls and sprites that straddle the splitter: a new arena item plus new vertices when there are no clip planes [U] hw_drawlist.cpp:260-316, 391-461, 483-574.
- Sorts the leaf runs of sprites with `std::stable_sort` on depth [U] hw_drawlist.cpp:583-622. A translucent list that holds only sprites (no flat or wall to split on) goes straight to this stable sort [U] hw_drawlist.cpp:687-690.
- At draw time, flat splitters become user clip planes (`SetClipSplit`) for the sub-trees [U] hw_drawlist.cpp:871-924.

**Inference:** sprite-heavy scenes such as particle effects or hordes run every particle through this tree sort, which is O(n log n) with splits. The tree is built twice per frame in stereo.

**Pass sequence** [U] hw_drawinfo.cpp:521-601:
1. Plain walls and plain flats.
2. Masked walls and flats at `gl_mask_threshold`.
3. Masked walls with depth offset, only when that list is non-empty.
4. Models.
5. Decals (depth-func LEqual).
6. *(portals)*
7. Translucent-border list, unsorted.
8. Translucent list, BSP-sorted, with depth writes off.

**Draw calls per item:**
- **Wall:** one `DT_TriangleFan` draw [U] hw_walls.cpp:93-96, ×N light slices when there is a 3D-floor light list [U] hw_walls.cpp:297-317.
- **Flat:** one `DrawIndexed` per section per plane, covering all subsectors of that section [U] hw_flats.cpp:192-204. Render-hack planes draw one fan per subsector [U] hw_flats.cpp:224-232, and flood gaps take 3 draws each [U] hw_flats.cpp:257-293.
- **Sprite:** one strip draw, ×light slices, +1 for colored-fog subtractive layers [U] hw_sprites.cpp:243-298. Models make one draw per mesh surface through `FHWModelRenderer::DrawArrays/DrawElements` [U] src/rendering/hwrenderer/hw_models.cpp:114-122.
- **Decal:** one draw, ×light slices [U] hw_decal.cpp:87-117.

**State changes per wall draw** [U] hw_walls.cpp:205-330: optional glow setup, `SetMaterial` (always), optional texture clamp, optional extended-colour block (object colours, add colour, texture FX, gradient planes), `SetFog`, `SetColor`, `SetLightIndex`, the draw itself, and then about 9 resets (`SetObjectColor`, `SetAddColor`, `SetTextureMode`, `SetTextureClamp`, `EnableGlow`, `EnableGradient`, `ApplyTextureManipulation`…).

`FRenderState::SetMaterial` always marks the material changed and calls `FMaterial::ValidateTexture`, even when the texture is the same as before [U] src/common/rendering/hwrenderer/data/hw_renderstate.h:629-652. On Vulkan, `Draw`/`DrawIndexed` default to `apply = true`, so every draw runs `Apply()` [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:58-72, [U] vk_renderstate.h:47-48. `Apply()` always writes a fresh `StreamData` record into the stream buffer and a matrix record, pushes constants, and calls the render-pass, scissor, viewport, stencil-ref, depth-bias, vertex-buffer, buffer-set and material appliers; most of those early-out on change flags, but the material is always rebound because `SetMaterial` always sets `mChanged` [U] vk_renderstate.cpp:181-207, 337-352, 401-452.

**Inference:** draw calls per eye ≈ visible wall tiers + visible section-planes + visible sprites/particles + decals + portal stencil draws, typically 1–5 k in vanilla-style maps and 10 k+ in detailed mods. That is ×2 for stereo, with no batching or instancing (see Absent).

---

## 4. Portals and skies

**Collection.** Wall processing turns sky, skybox, horizon, mirror, line-to-line, plane-mirror and sector-stack surfaces into portal objects. They are looked up with a linear `FindPortal` scan and created with `new` when not found. Each portal gets the wall pushed into its `lines` [U] hw_walls.cpp:656-775, [U] hw_drawinfo.cpp:441-447.

**Rendering.** `EndFrame` pops every portal that has lines, renders it, then deletes it [U] hw_portal.cpp:73-103. `HWDrawInfo::RenderPortal` [U] hw_drawinfo.cpp:610-623 runs four steps:
1. `SetupStencil`
2. A *new* `HWDrawInfo` (`StartDrawInfo` with the parent's uniforms)
3. `DrawContents`
4. `RemoveStencil`

**Scene portals re-run the entire pipeline.** `HWScenePortalBase::DrawContents` calls `Setup()` and then `di->DrawScene(DM_PORTAL)` [U] hw_portal.h:178-186. That means a full `CreateScene`: BSP walk from the root, clipper narrowed to the portal's angular window by `ClearClipper` [U] hw_portal.cpp:427-456 (line-to-line, skybox, sector-stack and plane-mirror portals; mirrors instead clear the clipper and add only the frustum range and the mirror line itself [U] hw_portal.cpp:592-598), worker spin-up, render hacks, all passes, and the portal's own nested portals.

This applies to mirrors [U] hw_portal.cpp:515-600, line-to-line portals [U] hw_portal.cpp:624-677, skybox viewpoints [U] hw_portal.cpp:703-744, sector stacks [U] hw_portal.cpp:820-860 and plane mirrors [U] hw_portal.cpp:921-945. The class hierarchy is at [U] hw_portal.h:168-416.

Sector-stack portals additionally:
- mark coverage with `SSRF_SEEN`;
- run `SetCoverage`, a recursion over **every BSP node** [U] hw_portal.cpp:779-813;
- start with the clipper fully blocked when the viewer is outside the portal [U] hw_portal.cpp:850-858.

Recursion limits:

| Portal type | Limit | Evidence |
|---|---|---|
| Mirror, line-to-line, plane mirror | `r_portal_recursions` (default 4) | [U] hw_portal.cpp:518, 628, 924; [U] src/common/rendering/hwrenderer/data/hw_cvars.cpp:45 |
| Skybox | 3 | [U] hw_portal.cpp:708 |
| Sector stack | 100 | [U] hw_portal.cpp:823 |

**Stencil protocol** [U] hw_portal.cpp:249-418:
- `SetupStencil` draws the portal's wall polygons as triangle fans plus top/bottom caps, with `SOP_Increment` and colour off.
- When the portal needs its own depth buffer, it redraws the polygons to set depth to 1 inside the stencil.
- `RemoveStencil` resets depth to the far plane, redraws the portal polygons with `DF_LEqual` to restore depth, and decrements the stencil.

That is 3–4 passes over each portal's polygons [U] hw_portal.cpp:180-240. The sky-type portal with the most lines (or the one whose bounding box contains the camera) is instead drawn **without** a stencil before the scene; out-of-bounds views still use the stencil. Inside a portal recursion it is skipped when it needs a depth buffer [U] hw_portal.cpp:113-155. Note that `IsSky()` is also true for skybox-viewpoint and sector-stack portals, so this pre-pass can be a full scene portal [U] hw_portal.h:284, 308, 402.

**Skies.** `HWSkyPortal` is not a scene portal: its `DrawContents` draws a dome or cube box with the sky vertex buffer, plus an optional fog layer [U] src/rendering/hwrenderer/scene/hw_skyportal.cpp:34-104. It needs no depth buffer and no BSP re-run [U] hw_portal.h:393-415. Horizon portals are also geometry-only [U] hw_portal.h:352-365. Sky scrolling is computed once per frame in `R_UpdateSky` [U] src/rendering/r_sky.cpp:130-146.

**Inference:** a map whose visible portals include a skybox viewpoint and two line portals costs about 4× the main view's BSP and setup per eye. In VR that becomes about 8×, plus the stencil passes, which interleave GPU state changes with CPU scene builds.

---

## 5. Sprites, particles, models

**Selection.** `RenderThings` iterates `sec->touching_renderthings` for each newly reached sector. It deduplicates with `thing->validcount`, applies the per-class `distancecheck` cvar, rejects things outside the current map sections, optionally processes a sprite shadow, then calls `HWSprite::Process` [U] hw_bsp.cpp:566-637. There is **no per-sprite frustum or clipper test** before `Process`: `grep -niE "frustum" hw_sprites.cpp` returns nothing. Every thing touching a visited sector is fully processed, including things behind the viewer within that sector (Inference).

**HWSprite::Process** [U] hw_sprites.cpp:798-1461 does the following:
- Visibility filters and the stealth check [U] hw_sprites.cpp:810-837.
- Interpolated position and scale [U] hw_sprites.cpp:840-848.
- Mirror and viewer checks [U] hw_sprites.cpp:857-890.
- `FindModelFrame` [U] hw_sprites.cpp:892-893.
- Portal clip [U] hw_sprites.cpp:918-922.
- Fake-sector resolution into a **local copy**, because the worker may not write the cache [U] hw_sprites.cpp:929-938.
- Rotation frame chosen from the view angle `(thingpos - vp.Pos).Angle()` [U] hw_sprites.cpp:969-1037.
- Billboard corner computation by sprite type [U] hw_sprites.cpp:1120-1175.
- Depth for sorting [U] hw_sprites.cpp:1177-1188.
- Optional "anamorphic" sprite-clip adjustment toward the camera (`gl_spriteclip == -1`, the default) [U] hw_sprites.cpp:1190-1238, [U] hw_sprites.cpp:76.
- Light level, colormap and render-style resolution [U] hw_sprites.cpp:1240-1411.
- 3D-floor light list or CPU split [U] hw_sprites.cpp:1438-1457.
- `PutSprite` [U] hw_sprites.cpp:1459.

The worker also writes actor fields: `RF_XFLIP` toggles and `isoscaleY`/`isotheta` [U] hw_sprites.cpp:1055-1056, 1077-1095.

**Sprite lighting.** For non-model sprites, dynamic light is **one RGB value per sprite**, computed at **draw time** by `GetDynSpriteLight` [U] hw_sprites.cpp:165-175. That function:
- looks up a light probe in a 3×3 cell grid [U] src/rendering/hwrenderer/scene/hw_spritelight.cpp:39-92;
- iterates the section's `flat_dlist` with distance and spot falloff;
- if `light->shadowmapped`, calls `screen->mShadowMap.ShadowTest`, a **CPU ray test through the level AABB tree** per light per sprite [U] hw_spritelight.cpp:100-207, [U] src/common/rendering/hwrenderer/data/hw_shadowmap.cpp:84-90.

**Models.** A model gets a full per-pixel light list via `hw_GetDynModelLight`. It walks the subsectors within the model's radius using `BSPWalkCircle`, deduplicates lights, and uploads the list once per model per scene [U] hw_spritelight.cpp:223-278, [U] hw_sprites.cpp:597-608.

Model *drawing* does substantial work at draw time:
- light-probe lookup;
- a **ZScript VM call** `AnimateBones` for `MF9_DECOUPLEDANIMATIONS` actors [U] hw_sprites.cpp:301-317;
- skeletal bone evaluation (`CalculateBones`) and `UploadBones` inside `RenderModelFrame` [U] src/r_data/models.cpp:570-652.

All of this runs per draw, i.e. per eye and per portal.

**Particles.** Particles come from `sub->sprites` (visual thinkers) and the level particle list. They are processed per visited subsector, with a portal clip [U] hw_bsp.cpp:639-668. They always land in the sorted translucent list (actor == nullptr) [U] hw_drawlistadd.cpp:142-145.

**HUD weapon.** `PreparePlayerSprites` runs once per scene in the main view [U] hw_bsp.cpp:1060-1061, [U] src/rendering/hwrenderer/scene/hw_weapon.cpp:868-898. 2D weapon sprites are drawn after post-processing in screen space with `GetHUDSpriteProjection`, an orthographic projection [U] hw_drawinfo.cpp:936-957, [U] src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:120-129. HUD models are drawn after a depth clear in `EndDrawScene` [U] hw_drawinfo.cpp:917-924.

---

## 6. Multithreading in 5.0

- **One worker thread** for scene setup, controlled by `gl_multithread` (Bool, default true, archived) [U] hw_bsp.cpp:43, 50. The main thread does the BSP walk, clipping, fake-flat creation, flat VBO patching and job enqueueing. The worker does wall, flat, sprite and particle processing plus portal-subsector registration [U] hw_bsp.cpp:111-216.
- **Serial on the main thread after the join:** render hacks [U] hw_drawinfo.cpp:501-504, dither tracing [U] hw_bsp.cpp:1049-1055, attached-portal actors [U] hw_bsp.cpp:1058, player sprites, translucency sort, all draw submission and, on persistent backends, wall vertex and light generation (§2b).
- `r_multithreaded` (default 1) [U] src/common/rendering/r_thread.cpp:33 belongs to the software-renderer drawer threads. `grep -rln "DrawerThreads\|r_thread.h" src | grep -v swrenderer` finds only `r_thread.h` and `r_thread.cpp` themselves, so the hardware path does not use it.
- No parallelism across eyes, portals or camera textures. Each is a sequential `ProcessScene` or `DrawScene` call (§1, §4). The arena, the clippers, the job queue and `validcount` are global singletons [U] hw_drawinfo.cpp:123-126, hw_bsp.cpp:109, [U] src/rendering/r_utility.cpp:176.

**Inference:** the best case is about a 2× overlap of BSP/clip with item processing, and the main thread then waits on `future.wait()` (`MTWait` clock) [U] hw_bsp.cpp:1036-1038. `stat rendertimes` reports "Main thread waiting" and "Worker thread waiting", which measure this directly [U] src/common/rendering/hwrenderer/data/hw_clock.cpp:78-98.

---

## 7. Stereo

- **Eye loop.** `VRMode` has `mEyeCount` 1 or 2 [U] hw_vrmodes.cpp:49-56, selected by `vr_mode` [U] hw_vrmodes.cpp:33, 68-100. Each eye runs `StartDrawInfo`, `SetupView` with the shifted position, then `ProcessScene`, i.e. the **full BSP traversal, clipping, worker job stream, render hacks, translucency sort, portal recursion and post-processing per eye** [U] hw_entrypoint.cpp:126-188. Nothing is shared between eyes except `R_SetupFrame`, the shadow-map update [U] hw_entrypoint.cpp:101-110 and the per-frame buffers, which only accumulate: vertices and lights are not reset between eyes [U] hw_entrypoint.cpp:315, 346.
- **Eye model.** The eye offset is a lateral shift of `vr_ipd × vr_hunits_per_meter / 2` along the yaw-perpendicular axis [U] hw_vrmodes.cpp:131-135, 189-202. The projection is an asymmetric frustum shifted by `zNear × shift / vr_screendist` [U] hw_vrmodes.cpp:137-183. These are display-stereo modes (anaglyph, side-by-side, top-bottom, interleaved, quad-buffer) [U] hw_vrmodes.cpp:68-100. There is no HMD runtime: `grep -rli "openxr\|openvr\|XrSession" src` returns nothing.
- **Vulkan stereo plumbing.** `FirstEye` and `NextEye` are empty virtuals in `DFrameBuffer` [U] src/common/rendering/v_video.h:258-259. The Vulkan device does not override them: `grep -rn "FirstEye\|NextEye" src/common/rendering/vulkan` finds only `VkPostprocess::NextEye`, which has an empty body [U] src/common/rendering/vulkan/renderer/vk_postprocess.cpp:358-360. GL routes them to `FGLRenderBuffers` [U] src/common/rendering/gl/gl_framebuffer.cpp:408-416.
- **View-dependent outputs that differ per eye** (Inference from code paths):
  - clipper angles, since the clipper is positional;
  - sprite rotation frame selection [U] hw_sprites.cpp:969, 1027;
  - translucent `ViewDistance` and sort order [U] hw_walls.cpp:592-595;
  - anamorphic sprite pull toward the eye [U] hw_sprites.cpp:1201-1237.

  Missing-texture hacks depend only on Z [U] hw_renderhacks.cpp:543, which is identical for both eyes as the code shifts only XY [U] hw_vrmodes.cpp:189-202. Plane facing compares `plane.ZatPoint(vp.Pos)` with `vp.Pos.Z` [U] hw_flats.cpp:519, 577, so for sloped planes it also depends on the (eye-shifted) XY position; for flat planes it is identical per eye.

---

## 8. What per-frame cost is proportional to (Inference, grounded in the loops cited)

Per eye and per scene portal:
- **BSP nodes visited** (CheckBox per far child) + **segs in visited subsectors** (`AddLine`: 2 memoised pseudo-angles, range check, possibly an occluder insert) [U] hw_bsp.cpp:257-403, 947-985.
- **Visible linedef tiers × (1 + 3D-floor/light-list slices + mid-texture gap slices)** → `HWWall` items, each 4–10 streamed vertices and one or more draws.
- **Visible sections × planes** → flat items, one indexed draw each. Vertex cost is O(0) unless the plane moved.
- **Things touching visited sectors** (not just visible ones) + **particles in visited subsectors** → sprite items, all in the sorted translucent list.
- **Σ(lights touching each lit surface)** → 64 B per light per surface written per scene, plus CPU AABB ray tests for shadow-mapped lights on sprites.
- **Translucent items**: n log n with splitting.
- **O(subsectors + sections + nodes) memsets ×2** per DrawInfo [U] hw_drawinfo.cpp:235-247, plus O(nodes) `SetCoverage` per sector-stack portal.
- **Number of draw calls**, each a full `Apply()` and a 1-record stream write on Vulkan.

Total ≈ (eyes) × (1 + scene portals, recursively) × (the above) + camera textures × (the above).

---

## Absent / not done (proven by search)

- **GPU visibility or occlusion culling:** `grep -rniE "occlusionquery|glBeginQuery|vkCmdBeginQuery|GL_SAMPLES_PASSED|ANY_SAMPLES" src/rendering src/common/rendering` finds only a `GL_TIME_ELAPSED` timer query in `gl_debug.cpp:126` and loader stubs. Culling is CPU-only through the angular clipper.
- **Instancing / multi-draw / indirect:** `grep -rnE "glDrawArraysInstanced|glDrawElementsInstanced|glMultiDraw|DrawIndirect" src --include=*.cpp | grep -v gl_load` returns nothing. Vulkan draws use instanceCount = 1 [U] vk_renderstate.cpp:63, 71.
- **Single-pass stereo / multiview:** `grep -rniE "multiview|OVR_multiview|VK_KHR_multiview|gl_ViewIndex|ViewIndex" src/rendering src/common/rendering wadsrc/static/shaders` returns nothing.
- **HMD runtime (OpenXR/OpenVR):** none (see §7).
- **Cross-frame caching of wall geometry:** wall vertices are always `AllocVertices` from the per-frame stream [U] hw_walls_vertex.cpp:279-288. The persistent `DoomLevelMesh` built at map load [U] src/maploader/maploader.cpp:3251 is consumed by `SetLevelMesh` [U] hw_entrypoint.cpp:118, which is an empty virtual in `DFrameBuffer` [U] src/common/rendering/v_video.h:174 and is overridden only by the Vulkan device, which forwards it to `VkRaytrace` [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:524-527. No raster draw list reads it.
- **Per-sprite frustum test:** none in hw_sprites.cpp, hw_bsp.cpp, hw_flats.cpp or hw_walls.cpp (`grep -niE frustum` matches only a comment at hw_bsp.cpp:987).
- **Sharing of scene setup between eyes:** none. The loop builds a new DrawInfo per eye [U] hw_entrypoint.cpp:142.
- **Default state sorting:** `gl_sort_textures` defaults to false [U] hw_drawinfo.cpp:45.

## Perf-relevant settings

| CVar | Default | Effect in this area | Cite |
|---|---|---|---|
| `gl_multithread` | true | 1 worker for wall/flat/sprite setup | [U] hw_bsp.cpp:43 |
| `gl_sort_textures` | false | sort opaque lists by texture | [U] hw_drawinfo.cpp:45 |
| `gl_seamless` | true | T-junction vertex splitting per wall | [U] src/common/rendering/hwrenderer/data/hw_cvars.cpp:43; hw_walls_vertex.cpp:283 |
| `gl_portals` | true | portal rendering on/off | [U] hw_cvars.cpp:40; hw_portal.cpp:166 |
| `gl_mirrors` | true (debug) | line mirrors as portals | [U] hw_cvars.cpp:41; hw_walls.cpp:1386 |
| `gl_mirror_envmap` | true | extra reflective mirror layer + decals | [U] hw_cvars.cpp:42; hw_walls.cpp:727-731 |
| `r_portal_recursions` | 4 | depth limit for mirrors/line/plane portals | [U] hw_cvars.cpp:45 |
| `gl_no_skyclear` | false | skip pre-scene first-sky-portal draw | [U] hw_drawinfo.cpp:46, 1030 |
| `gl_noskyboxes` | false | (declared in hw_sky.cpp) | [U] src/rendering/hwrenderer/scene/hw_sky.cpp:33 |
| `gl_light_sprites` / `gl_light_particles` | true / true | per-sprite CPU dynlight + model light lists | [U] hw_dynlightdata.cpp:35-36 |
| `gl_light_models` | true | (declared) | [U] hw_models.cpp:36 |
| `gl_light_shadowmap` | false | enables shadowmap + CPU AABB `ShadowTest` per sprite-light | [U] hw_shadowmap.cpp:57, 84-90 |
| `gl_shadowmap_quality` | 1024 | shadowmap resolution | [U] hw_shadowmap.cpp:66 |
| `gl_shadowmap_filter` | 1 | (uniform) | [U] hw_cvars.cpp:141 |
| `r_dynlights` | true | dynamic lights master | [U] src/g_cvars.cpp:64 |
| `gl_ssao` | 0 | SSAO per scene | [U] src/common/rendering/hwrenderer/postprocessing/hw_postprocess_cvars.cpp:62 |
| `gl_ssao_portals` | 1 | SSAO for N portal scenes | [U] hw_postprocess_cvars.cpp:68; hw_drawinfo.cpp:1000-1015 |
| `gl_fogmode` | 2 | fog/fog-boundary path | [U] hw_cvars.cpp:32; hw_walls.cpp:122 |
| `gl_lightmode` | 1 | light mode (shader) | [U] src/g_level.cpp:146 |
| `gl_spriteclip` | -1 | anamorphic sprite clip math per sprite | [U] hw_sprites.cpp:76, 1190 |
| `gl_billboard_mode` / `gl_billboard_faces_camera` | 0 / false | billboard math; affects sort splitting | [U] hw_sprites.cpp:82-83; hw_drawlist.cpp:512-535 |
| `gl_sprite_blend` | false | opaque-sprite blend mode | [U] hw_sprites.cpp:75 |
| `gl_fuzztype` | 8 | fuzz shader override | [U] hw_sprites.cpp:86 |
| `gl_mask_threshold` / `gl_mask_sprite_threshold` | 0.5 / 0.5 | alpha-test thresholds | [U] hw_drawinfo.cpp:50-51 |
| `r_actorspriteshadow` | 1 | sprite shadows (2nd `Process` per actor) | [U] src/rendering/r_utility.cpp:103; hw_bsp.cpp:594-602 |
| `r_actorspriteshadowdist` | 1500 | shadow distance | [U] r_utility.cpp:110 |
| `r_maxparticles` | 4000 | particle count cap | [U] src/g_cvars.cpp:113 |
| `r_particles` | true | particles on | [U] src/playsim/p_effect.cpp:55 |
| `r_radarclipper` | false | extra radar clipper (OOB cams) | [U] r_utility.cpp:95 |
| `r_dithertransparency` | false | Trace-based dither flags (main thread) | [U] r_utility.cpp:96; hw_bsp.cpp:1049-1055 |
| `r_drawplayersprites` | true | HUD weapon prep | [U] r_utility.cpp:94 |
| `gl_pipeline_depth` | 0 | GL buffer pipelining (`mPipelineNbr`); Vulkan stays 1 | [U] src/common/rendering/v_video.cpp:57; gl_framebuffer.cpp:125 |
| `vr_mode` | 0 | eye count (1/2) | [U] hw_vrmodes.cpp:33 |
| `vr_ipd` / `vr_screendist` / `vr_hunits_per_meter` | 0.062 / 0.80 / 41.0 | eye shift & frustum | [U] hw_vrmodes.cpp:39-45 |
| `cl_capfps` | false | TicFrac=1 (no interpolation) | [U] src/common/rendering/v_framebuffer.cpp:51 |
| `vid_maxfps` | 500 | frame cap | [U] v_video.cpp:71 |
| `gl_render_walls/flats/things` | true | debug toggles | [U] hw_bsp.cpp:224-226 |
| Per-class `distancecheck` cvar | per-actor | sprite distance cull | [U] hw_bsp.cpp:578-587 |

Instrumentation: `stat rendertimes` (BSP, Clip, wall/flat/sprite Setup vs Render, MT waits, Drawcalls), `stat renderstats` (walls, splits, vertices, flats, sprites, portals) and `stat lightstats` [U] hw_clock.cpp:78-135.

## Transfer notes for UZDoom VR

| Mechanism | UZDoom location | Grade | VR / Vulkan change and cost |
|---|---|---|---|
| Per-eye full scene rebuild | hw_entrypoint.cpp:126-188 | **Rework** | Build the draw lists once per frame from a *combined* cyclops viewpoint. That means a clipper `FrustumAngle` covering the union of both eye frusta, plus an IPD-padded position or a clip from the eye midpoint with angular padding. Then submit both eyes from the same lists with multiview (one pass) or two command-buffer replays. This needs `vp`-independent outputs: move the per-eye `ViewDistance`, sprite rotation and anamorphic pull into a cheap per-eye post-pass, or into the shader. Expected saving: about 45–50 % of scene-setup CPU. The interfaces (`HWDrawInfo`, draw lists) survive. |
| Angular clipper with 46° pitch cutoff | hw_drawinfo.cpp:377-397; hw_clipper.h | **Rework** | In an HMD, pitch beyond 46° is routine, and then back-culling disappears. Add a real frustum test (planes from the combined VR frustum) on BSP node bboxes alongside the angular occlusion clipper. Optionally add GPU occlusion (Hi-Z) for open maps. |
| Single worker, job queue | hw_bsp.cpp:43-216, 1024-1039 | **Rework → Rebuild** | Generalise to N workers with per-worker arenas, per-worker draw lists merged after the join, and an atomically padded vertex/light allocator (fix the `GetBuffer`-before-`fetch_add` ordering). The `validcount`-based render hacks need per-scene epoch arrays. Replace the busy-spin with a bounded spin then sleep, so a spinning worker does not steal a core from the playsim or the compositor. |
| Draw-time light/vertex generation on persistent backends | hw_walls.cpp:364-373; hw_flats.cpp:192-197; hw_sprites.cpp:278-281 | **Port** | Move these back to setup (they are in the worker in the non-persistent path already [U] hw_walls.cpp:609-616) so recording is pure submission. Small change, large win for main-thread time on Vulkan. |
| Wall vertex streaming every frame | hw_walls_vertex.cpp:279-288 | **Rework** | Walls are linedef-tier quads with static XY. Cache them per sidedef tier in a persistent buffer keyed by plane heights and texture offsets (same scheme as `CheckPlanes` for flats [U] hw_vertexbuilder.cpp:441-469). The draw list then carries indices only. `DoomLevelMesh` (maploader.cpp:3251) is a candidate base. |
| Per-surface light-list copies | hw_lightbuffer.cpp:75-123 | **Rebuild** | Replace with a per-frame global light array plus clustered/tiled light assignment on the GPU (or per-surface index lists into one light array). That removes the O(surfaces × lights) CPU writes and the silent overflow at 80 k entries, which stereo pressure doubles. |
| Unbatched draws, `SetMaterial` dirtying every draw, per-draw StreamData | hw_walls.cpp:205-330; hw_renderstate.h:629-652; vk_renderstate.cpp:181-207 | **Rebuild** | Use bindless textures plus per-draw data in an SSBO and `vkCmdDrawIndexedIndirect`/multi-draw: one indirect call per pass. Sort opaque by pipeline then material. Required to hold 2× draw counts inside an 8–11 ms frame budget. |
| Translucent BSP-tree sort incl. all sprites | hw_drawlist.cpp:629-948; hw_drawlistadd.cpp:138-153 | **Rework** | Route opaque (`STYLEHW_Solid`) sprites to an alpha-tested opaque list. Sort the rest once per frame from the cyclops view and reuse the order for both eyes. Ordering errors are acceptable at IPD scale (Inference). |
| Scene portals = recursive full DrawScene | hw_portal.h:178-186; hw_portal.cpp:73-103 | **Rework** | Keep stencil portals, but build portal scenes in parallel jobs. Their viewpoints are known after the main BSP, so the CPU builds can run concurrently and submission happens in order. Cap the recursion for VR (`r_portal_recursions` 4 → 1–2). Skybox viewpoints could be rendered once per frame into a cubemap rather than per eye (Inference). |
| Sky portals | hw_skyportal.cpp:34-104 | **Port** | Already cheap (no BSP). For VR, draw the sky dome after opaques with depth-equal-far to avoid the stencil path. |
| Sprite CPU shadow-ray test per light | hw_spritelight.cpp:171; hw_shadowmap.cpp:84-90 | **Port** | Cache per actor per frame, so the result is shared across eyes and portals, or move it to the shader using the existing shadowmap. |
| Model bone evaluation + VM `AnimateBones` at draw time | hw_sprites.cpp:301-317; models.cpp:570-652 | **Rework** | Evaluate once per actor per frame in setup and cache the bone offset, so draws in each eye and portal reuse it. |
| Vulkan stereo plumbing (`FirstEye/NextEye` not overridden) | v_video.h:258-259; vk_postprocess.cpp:358-360 | **Rebuild** | The eye-image path for Vulkan must be written anyway for OpenXR swapchains. Use 2-layer images + `VK_KHR_multiview`. |
| Per-DrawInfo memsets and TMap reallocs | hw_drawinfo.cpp:217-254; tarray.h:1326-1330 | **Port** | Use epoch-stamped flag arrays instead of memsets, and `TMap` reuse (clear without freeing). This matters with many portals and eyes. |

## Open questions

1. (Resolved by verifier.) `DoomLevelMesh` reaches the renderer only through `SetLevelMesh` [U] hw_entrypoint.cpp:118, a no-op on GL/GLES [U] v_video.h:174 and a pass-through to `VkRaytrace` on Vulkan [U] vk_renderdevice.cpp:524-527 (`vk_raytrace` cvar [U] vk_renderdevice.cpp:63). `grep -rn "levelMesh" src` finds no other consumer, so no raster pass reads it.
2. On Vulkan, `mPipelineNbr` stays 1 (only the GL and GLES framebuffers set it from `gl_pipeline_depth` [U] gl_framebuffer.cpp:125, gles_framebuffer.cpp:112). I did not verify how the Vulkan backend synchronises CPU writes into the persistent streaming and light buffers against in-flight frames, i.e. whether there is a full wait per frame. This belongs to the backend section.
3. I did not trace the GL backend's `NextEye` / `FGLRenderBuffers::NextEye` [U] gl_renderbuffers.cpp:1016 or how the final eye composite happens.
4. I did not read the full bodies of `HWSprite::ProcessParticle` [U] hw_sprites.cpp:1470-1657 and `DoFFloorBlocks/BuildFFBlock` [U] hw_walls.cpp:1750-2114. Their per-item costs are assumed similar to sprites and walls.
5. The per-wall cost numbers in §2d are estimates from operation counts. They have not been measured, and `stat rendertimes` on a reference map should be captured to calibrate them.
