# Helion: visibility, culling, portals, sky and 3D-floor rendering

By default Helion barely culls on the CPU. In the default `Static` render mode, all non-moving world geometry is uploaded once into per-texture GPU buffers and drawn whole every frame. CPU visibility work only gathers *dynamic* things (moving or scrolling sectors and sides, entities). It does this with a coarse 512-unit "render blockmap" walk, limited to a box of `MaxDistance` (6000 units by default) around the viewer and a half-plane test behind the camera. A GZDoom-style front-to-back BSP walk with a 2D angular clipper exists as a second mode (`Bsp`), and an `Adaptive` mode picks between the two from the timing of a background-thread BSP walk. The most important point for VR: in every mode the CPU visibility pass and the refill of the dynamic vertex buffers run **at most once per game tic (35 Hz)**, not once per frame. Frames between tics redraw the same buffers without re-uploading them, and the vertex shader interpolates `prevPos`→`pos` with `timeFrac`. Helion has no GPU occlusion queries, no PVS, no frustum-side culling, and no real portals (line, sector, skybox or mirror). The "Portals" directory holds only the flood-fill emulation of missing textures.

---

## 1. Per-frame control flow: who decides what to draw

`LegacyWorldRenderer.PerformRender` is the single entry point. It decides each frame:

1. **Mode selection.** `m_renderStatic` is true unless the config mode is `Bsp`, the view is above or below the fake planes of a Boom transfer-heights (242) sector (`TransferHeightView` is `Top` or `Bottom`), or the adaptive heuristic chose BSP [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:299-308. When the view is `Middle`, the result comes from `UseBspBasedOnHeuristic`, which returns true for `Bsp`, false for `Static`, and the heuristic's choice for `Adaptive` [H] LegacyWorldRenderer.BspHeuristics.cs:19-34. When `TransferHeightView != Middle`, the static cache cannot be used: "If the transfer height view is not the middle then the cached static geometry cannot be used. Render all sectors dynamically instead." In that case the code forces `m_renderStatic = false`, so it takes the BSP path [H] LegacyWorldRenderer.cs:297-300.
2. **Tic gate.** `renderTickChange = NeedsRenderTickChange(...)` is true only when `world.GameTicker` changed, the static/BSP mode flipped, or the transfer-heights view changed, and `Developer.Render.Lock` is off [H] LegacyWorldRenderer.cs:143, 310. `GameTicker` is incremented once per `WorldBase.Tick()`, including while paused [H] Core/World/WorldBase.cs:1042-1102.
3. **Clear only on a new tic.** The entity and dynamic world data buffers are cleared only when `renderTickChange` is true [H] LegacyWorldRenderer.cs:789-799.
4. **Traversal only on a new tic.**
   ```csharp
   if (renderTickChange)
   {
       m_renderProfiler.WorldTraversal.Start();
       SetupRenderData(world, renderInfo);
       if (m_renderStatic)
           IterateBlockmap(world);
       else
           TraverseBsp(world, renderInfo);
       m_renderProfiler.WorldTraversal.Stop();
   }
   ```
   [H] LegacyWorldRenderer.cs:331-341
5. **Every frame:** sky (dynamic), then flood fill, then static skies, then dynamic walls and flats, then static walls and flats (if static mode), then two-sided middles and 3D middles, then opaque sprites, then the OIT transparent pass [H] LegacyWorldRenderer.cs:345-379.

Between tics, motion stays smooth because every dynamic vertex carries both `prevPos` and `pos` (`DynamicVertex` has `pos` and `prevPos` attributes [H] Core/Render/OpenGL/Renderers/Legacy/World/DynamicVertex.cs:7-27). Static-cache vertices (`StaticVertex`) have only `pos`, because anything that moves is pulled out of the static set and drawn through the dynamic path [H] Core/Render/OpenGL/Renderers/Legacy/World/StaticVertex.cs:7-26. The vertex shader does `mix(prevPos, pos, timeFrac)` [H] Core/Render/OpenGL/Renderers/Legacy/World/InterpolationShader.cs:63-68. Sprites are interpolated the same way (`interpolatedPos = mix(prevPos, pos, timeFrac)`) [H] Core/Render/OpenGL/Renderers/Legacy/World/Entities/EntityProgram.cs:144.

**Dynamic buffers are uploaded once per tic, not once per frame.** The dynamic world VBOs (`RenderWorldData`, a `DynamicVertexBuffer<DynamicVertex>` with `DynamicDraw`) are cleared and refilled only when `renderTickChange` is true [H] LegacyWorldRenderer.cs:789-799; [H] Core/Render/OpenGL/Renderers/Legacy/World/Data/RenderWorldData.cs:16-27, 40-43. Every frame, `Draw()` calls `Pipeline.Vbo.Upload()` [H] Data/RenderWorldData.cs:45-68. But `Upload()` returns at once when the `Uploaded` flag is set, and only `Add*`/`Clear` reset that flag [H] Core/Render/OpenGL/Buffer/BufferObject.cs:58-101, 141-145. So the first frame after a tic change does one full `glBufferData` of each touched buffer [H] Core/Render/OpenGL/Buffer/ArrayBufferObject.cs:24-29. The frames that follow re-issue only the draw calls. Sprite buffers follow the same pattern (`RenderData.Draw` → `Vbo.Upload()`, cleared only on tic change) [H] Data/RenderData.cs:43-67; [H] LegacyWorldRenderer.cs:794-798. So do the dynamic sky geometry (`StreamVertexBuffer`, `UploadIfNeeded`) [H] Sky/Sphere/SkySphereComponent.cs:30, 40-60 and the BSP-mode flood-fill buffers [H] Geometry/Portals/FloodFill/FloodFillRenderer.cs:345, 385-396.

**Inference:** CPU visibility cost is amortised over `fps/35` frames: about 3.4 frames per tic at 120 Hz, or about 2.6 at 90 Hz. This is the largest single CPU saving in the renderer. GPU cost stays per frame.

The view position is handled conservatively across the tic interval. `SetupRenderData` records both `Camera.Position` (the end-of-tic position) and `Camera.PositionInterpolated` (the position at the frame that triggered the rebuild) [H] LegacyWorldRenderer.cs:180-196. `OldCamera` stores both [H] Core/Render/Common/Shared/OldCamera.cs:33-35, 63-78, and `Player.GetCamera` builds the interpolated position from `prevPos` and `currentPos` [H] Core/World/Entities/Players/Player.cs:682-711. All culling tests below run against **both** positions.

---

## 2. Default mode: Static geometry cache + render-blockmap walk

### 2.1 The static cache (no culling at all)

On map load, `StaticCacheGeometryRenderer.UpdateTo` walks *every* sector and *every* line. It adds each non-moving floor and ceiling (`(sector.Floor.Dynamic & IgnoreFlags) == 0`, where `IgnoreFlags = SectorDynamic.Movement`), every 3D-floor sector, and every line into VBOs, then uploads them all [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Static/StaticCacheGeometryRenderer.cs:36, 103-172.

Geometry is bucketed by `(GeometryType, array-texture handle, repeatY)` [H] Geometry/Static/GeometryTextureLookup.cs:18-28, Geometry/Static/TextureGeometryLookup.cs:6-26. One bucket is one `VertexPipeline`, and each bucket gets one `DrawArrays` per frame:

```csharp
for (int i = 0; i < geometry.Count; i++)
{
    var data = geometry[i];
    if (data.Pipeline.Vbo.Count == 0)
        continue;
    ...
    data.Pipeline.Vbo.UploadIfNeeded();
    data.Pipeline.Bind();
    data.Pipeline.DrawArrays();
```
[H] Geometry/Static/StaticCacheGeometryRenderer.cs:859-896

`RenderWalls`, `RenderFlats`, `RenderTwoSidedMiddleWalls` and `RenderMiddle3D` simply call `RenderGeometry` on the whole list of their type [H] StaticCacheGeometryRenderer.cs:814-824. Nothing tests the view position, a frustum, distance or occlusion before these draws.

**What happens when a sector starts moving.** `World_SectorMoveStart` → `HandleSectorMoveStart` marks the plane `SectorDynamic.Movement`, links the sector into the render blockmap via `StaticDataApplier.SetSectorDynamic`, and then *zeroes* the plane's static vertices in place, along with its walls, cover geometry and sky geometry. The zeroing is a `glBufferSubData` of zeros (degenerate triangles) [H] StaticCacheGeometryRenderer.cs:965-1027, 1587-1593; [H] Core/World/Static/StaticDataApplier.cs:81-110. When movement completes, the geometry is re-added: `World_SectorMoveComplete` at StaticCacheGeometryRenderer.cs:1120-1212, and `ClearSectorDynamicMovement` unlinks from the blockmap, unless a flat still scrolls [H] StaticDataApplier.cs:130-143. On re-add, the zeroed slot is normally still owned by the plane or wall. `UpdateVertices` then rewrites it in place with `UploadSubData` [H] StaticCacheGeometryRenderer.cs:1360-1402. A new slot is needed only when there is no slot or the vertex count grew; texture changes also free slots [H] StaticCacheGeometryRenderer.cs:1272, 1320. In that case `AddNewGeometry` first tries a free-list slot with the same texture, type, repeatY and vertex count [H] StaticCacheGeometryRenderer.cs:1536-1553. Only when none matches is new geometry appended to the bucket, and then the *whole* VBO is flagged for re-upload. A TODO admits this: "this causes the entire vbo to be uploaded when we could use sub-buffer" [H] StaticCacheGeometryRenderer.cs:1555-1561.

**Inference:**
- Per-frame CPU cost of static geometry is O(number of texture buckets): a few hundred draw calls at most, independent of map size.
- GPU cost is O(entire map), every frame, for every eye. Buckets are ordered by texture, not by depth, so early-Z rejection depends on submission order luck.
- On large maps (tens of thousands of lines) the GPU vertex cost is trivial on modern hardware. Fragment overdraw from back-to-front texture order is the real cost, and the cost grows with resolution × 2 eyes.
- The static set is only modified on sector-move start and stop, texture changes, and similar events, so steady-state CPU cost is close to zero.

### 2.2 The render blockmap

The render blockmap is a separate `BlockMap` built over the physics blockmap's bounds with a **512-unit** cell [H] Core/World/WorldBase.cs:429-439. It holds three per-cell lists: `HeadRenderEntities` (intrusive doubly-linked list of entities), `DynamicSectors` (`LinkableList<DynamicIsland>`) and `DynamicSides` [H] Core/World/Blockmap/BlockMap.cs:57-59, 95-115.

- **Entities** are linked by their *centre point only*, into exactly one cell [H] BlockMap.cs:240-257. Linking happens on every `LinkToWorld` [H] Core/World/Physics/PhysicsManager.cs:126-135.
- **Dynamic sectors** are sectors flagged `Movement` or `Scroll`, so scrolling flats are included [H] StaticDataApplier.cs:107-110. They are linked into every cell overlapped by the bounding box of each *sector island* (a connected piece of the sector). Voodoo and monster-closet islands are skipped [H] BlockMap.cs:282-322.
- **Dynamic sides** are the sides with scrolling textures. They are linked along their line with a seg iterator [H] BlockMap.cs:324-341, and only scrollers are linked at load time [H] StaticDataApplier.cs:23-42.

### 2.3 `IterateBlockmap` (the per-tic walk)

```csharp
Box2D box = new(m_renderData.ViewPosInterpolated.X, m_renderData.ViewPosInterpolated.Y, m_renderData.MaxDistance);
...
for (int by = it.BlockStartY; by <= it.BlockEndY; by++)
{
    for (int bx = it.BlockStartX; bx <= it.BlockEndX; bx++)
    {
        var index = by * it.Width + bx;
        if (occlude && !BlockInView(bx, by, dimension, origin, occluder, m_renderData.ViewDirection))
            continue;

        RenderSectors(world, index);

        if (m_renderStatic)
            RenderSides(world, index);

        var renderIndex = 0;
        for (var entity = world.RenderBlockmap.HeadRenderEntities[index]; entity != null; entity = entity.RenderBlockNext)
        {
            renderIndex = renderIndex % EntityRenderIndexMax;
            RenderEntity(world, entity, renderIndex++);
        }
    }
}
```
[H] LegacyWorldRenderer.cs:145-178

- **Distance:** a square box of half-size `MaxDistance` around the interpolated view position [H] LegacyWorldRenderer.cs:147. `MaxDistance` config 0 maps to `Constants.DefaultMaxDistance = 6000` [H] Core/Render/Renderer.cs:347-349; [H] Core/Util/Constants.cs:432.
- **"Occlusion" test (really a back half-plane test).** `SetOccludePosition` places an occluder point 32 units *behind* the camera, and disables the test entirely when |pitch| > 45°. The comment calls it "a hack until frustum culling exists" [H] LegacyWorldRenderer.cs:773-787. `BlockInView` keeps a cell if any of its 4 corners has a non-negative dot product with the 2D view direction relative to that point [H] LegacyWorldRenderer.cs:198-215.
- **Dynamic sectors** (`RenderSectors`):
  - Island-level de-duplication through `sectorIsland.BlockmapCount == CheckCount`, skipped for 3D sectors.
  - A connectivity cull: `if (sectorIsland.ParentIsland != null && sectorIsland.ParentIsland != m_renderData.ViewIsland) continue;`.
  - Sector-level de-duplication through `sector.CheckCount`.
  - A Euclidean box-distance test against `MaxDistanceSquared`.
  - Then `m_geometryRenderer.RenderSector(...)` [H] LegacyWorldRenderer.cs:217-247.
  - `ViewIsland` is the island containing the viewer's subsector [H] LegacyWorldRenderer.cs:187. `ParentIsland` is the full map island, the subsector-connected component [H] Core/World/Geometry/MapGeometry.cs:68-86; [H] Core/World/Geometry/Islands/Island.cs:16-28. This is a cheap "different disconnected region, can't be seen" filter, the only PVS-like mechanism Helion has.
- `RenderSector` emits *all* lines of the sector with a per-line back-face test against both positions. It also emits the sector's flats, unless the flats are static [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/GeometryRenderer.cs:475-494, 712-768.
- **Scrolling sides** (`RenderSides`) are de-duplicated with `side.BlockmapCount`. They are skipped if their sector is moving, because the sector path already draws them [H] LegacyWorldRenderer.cs:249-269.

**Inference (cost):**
- At the default 6000 distance with 512-unit cells, the 12,000-unit box spans 24-25 cells per axis, about 576-625 cells. The half-plane test drops roughly half of them, and on small maps the box is clipped to the map bounds.
- Per-cell cost is a null check plus list walks. Dynamic content is normally a handful of doors and lifts, so the walk costs microseconds.
- The cost is proportional to (cells in range) + (entities in range) + (lines of moving or scrolling sectors in range). It does not depend on what is actually visible.

**Inference (correctness side effect):**
- A sector that starts moving has its static vertices zeroed, and is then drawn only by `RenderSectors`, which is distance-limited. So a door or lift farther than `MaxDistance` from the viewer is invisible while it moves.
- `MaxDistance` is also overridable per map by compatibility definitions and line triggers [H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:176-184, 478-485.

### 2.4 Line of sight is not used for rendering in static mode

In static mode, nothing occludes an entity except the 2D distance test and the half-plane test. Every monster within 6000 units and in front of the camera generates a sprite quad each tic, even through walls. The depth buffer resolves visibility [H] LegacyWorldRenderer.cs:271-293.

---

## 3. `Bsp` mode: front-to-back BSP with an angular clipper

### 3.1 Traversal

```csharp
while ((nodeIndex & BspNodeCompact.IsSubsectorBit) == 0)
{
    ref var node = ref world.BspTree.Nodes[nodeIndex];
    if (!ShouldRenderBox(node.BoundingBox, pos2D, prevPos2D))
        return;
    var onRight = (node.SplitDelta.X * (position.Y - node.SplitStart.Y)) - (node.SplitDelta.Y * (position.X - node.SplitStart.X)) < 0;
    int front = *(byte*)&onRight;
    RecursivelyRenderBsp(node.Children[front], position, pos2D, prevPos2D, world);
    nodeIndex = node.Children[front ^ 1];
}
```
[H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.Bsp.cs:33-46

- Nodes are a flat array of `BspNodeCompact` structs: split start and delta, the node's own `Box2D`, and `fixed uint Children[2]` with a high bit for leaves [H] Core/World/Bsp/BspNodeCompact.cs:12-37. Recursion goes into the front child, and a loop handles the back child.
- The front/back side is computed from the *end-of-tic* position (`m_renderData.ViewPos3D`) [H] LegacyWorldRenderer.Bsp.cs:29.
- The bounding-box test happens at every node entered (front and back) and again at each subsector [H] LegacyWorldRenderer.Bsp.cs:38, 49.

### 3.2 `ShouldRenderBox`: near-plane + two 360° clippers

```csharp
if (box.Contains(pos2D) || box.Contains(prevPos2D)) return true;
if (m_occlude && !m_frustumPlanes.BoxInFront(box)) return false;
box.GetSpanningEdge(pos2D, out var x1, out var y1, out var x2, out var y2);
if (!m_viewClipper.InsideAnyRange(x1, y1, x2, y2)) return true;
box.GetSpanningEdge(prevPos2D, out x1, out y1, out x2, out y2);
return !m_viewClipperPrev.InsideAnyRange(x1, y1, x2, y2);
```
[H] LegacyWorldRenderer.Bsp.cs:73-88

- **Frustum:** the planes come from `MvpNoPitch` [H] LegacyWorldRenderer.Bsp.cs:19; [H] Core/Render/Common/Shared/Frustum.cs:7-44. However, `BoxInFront` tests **only the Near plane** [H] Core/Render/Common/Shared/FrustumPlane.cs:17-20. Left/right/top/bottom planes are computed but never used for world culling. The test is disabled for |pitch| > 45° through the same `m_occlude` flag [H] LegacyWorldRenderer.cs:328-329, 778-782.
- **Clippers:** `m_viewClipper` is centred on the end-of-tic position and `m_viewClipperPrev` on the interpolated position [H] LegacyWorldRenderer.Bsp.cs:24-27. A box is culled only if *both* clippers fully occlude it.
- No FOV ranges are ever added to the clipper. Unlike GZDoom it covers the full 360°, and the near plane acts as the only angular bound.

**Inference:** the 360° clipper plus the near-plane half-space keeps visibility valid while the view yaw changes within a tic. The view is rebuilt only at 35 Hz while yaw updates every frame, so this margin is required, not optional. The cost is that geometry in the wedges beside the true FOV is also emitted. The near-plane half-space is about 180° wide, so at 90° FOV that is roughly 45° on each side (90° in total). *(Verifier: corrected from "~90°-wide wedges at each side".)*

### 3.3 `ViewClipper`

- Angles are "diamond angles": a monotonic, division-only substitute for `atan2`, mapped to `[0, 2^32)` [H] Core/Render/Common/Shared/World/ViewClipping/ViewClipper.cs:58-95.
- Spans are stored as a sorted `DynamicArray<ClipSpan>` (struct array, initial capacity 256) [H] ViewClipper.cs:32.
- Insert uses a binary search for the first span whose end ≥ start [H] ViewClipper.cs:323-336, then merges forward with `RemoveRange` [H] ViewClipper.cs:278-312, 347-380.
- **Queries are a linear scan with early exit** [H] ViewClipper.cs:232-247, even though a binary search for the insert point exists. The file opens with "TODO: Implement this with some kind of discrete fused interval tree for O(lg n)" [H] ViewClipper.cs:12.
- `ClipSpan.Contains` is inclusive at both ends [H] Core/Render/Common/Shared/World/ViewClipping/ClipSpan.cs:42.
- Ranges that wrap past angle 0 are split into two [H] ViewClipper.cs:124-139, 201-224.

### 3.4 Subsector processing

`RenderSubsectorWalls` iterates the subsector's segs [H] GeometryRenderer.cs:417-448:
- Minisegs (`LineId == -1`) are skipped.
- One-sided segs that are back-facing from *both* positions are skipped.
- The seg is added to **both** clippers if it is one-sided or `RenderBlock.IsBlocked(...)`.
- Each line is emitted once per traversal, using a `BitArray m_hitLines` that is cleared each traversal [H] GeometryRenderer.cs:379-382.
- Segs are **not** tested against the clipper. The code says: "Bsp segs aren't checked against the clipper because it's more expensive than just rendering the lines. Most subsectors are rejecting by the bounding box check before this is reached." [H] GeometryRenderer.cs:433-434. The code matches the comment: no `InsideAnyRange` call exists in that loop.

`RenderBlock.IsBlocked` decides whether a two-sided line occludes [H] Core/Render/RenderBlock.cs:30-64:
- It is closed if the back ceiling ≤ the front floor, *and the upper texture is not null*.
- The lower and "back sector closed" cases are handled similarly.
- Transfer-heights sectors never block.
- The texture check exists so missing-texture "see-through door" tricks (TNT MAP02) render [H] RenderBlock.cs:10-12.

**Flats are emitted per sector, not per visible subsector:** "Flats are rendered by sector, walls are rendered by subsector" [H] LegacyWorldRenderer.Bsp.cs:52-54. On the first subsector of a sector reached in a traversal (`sector.CheckCount`), `RenderSectorFlats` emits the triangulation of **all** of that sector's subsectors [H] GeometryRenderer.cs:532-612. The per-sector subsector lists are built at load [H] GeometryRenderer.cs:213-223. A flat is skipped if the view is on the wrong side of the plane at both positions (`ViewPositionForFlatInvalid`) [H] GeometryRenderer.cs:1785-1798, 1910-1915.

**Entities in BSP mode** are pulled from `subsector.Sector.Entities` the first time a sector is reached. They are de-duplicated with `entity.BlockmapCount`, because entities link into every sector they intersect [H] LegacyWorldRenderer.Bsp.cs:56-70; [H] Core/World/Physics/PhysicsManager.cs:1391-1411. They then go through the same `RenderEntity` distance, half-plane and flag filter as in static mode.

### 3.5 Per-tic geometry rebuild with caching

BSP mode uses `GeometryRenderMode.All` [H] LegacyWorldRenderer.cs:316. Every visible wall and flat is memcpy'd into per-array-texture dynamic VBOs each tic (`renderData.Pipeline.Vbo.AddMemoryCopy(data)`) [H] GeometryRenderer.cs:918-925, 1893-1897. Vertex *generation* is cached:
- Wall vertices are cached per side (`m_vertexLookup`). They are regenerated only on `side.OffsetChanged || m_sectorChangedLine || data == null` [H] GeometryRenderer.cs:904-915.
- Flat vertices are cached per sector and regenerated only when `FlatChanged` or the invalidation counter fires [H] GeometryRenderer.cs:1805, 1849, 1933-1940.
- Change detection uses `LastRenderChangeGametick` and `PrevZ != Z` [H] Core/World/Geometry/Sectors/SectorPlane.cs:90-102; [H] Core/World/Geometry/Sectors/Sector.cs:245-257.

**Inference (cost model):**
- CPU per tic ≈ O(nodes visited × (diamond-angle math + linear clip scan)) + O(segs in visited subsectors) + O(bytes of vertices memcpy'd).
- The memcpy covers whole sectors' flats even when only a sliver of the sector is visible.
- GPU per frame ≈ O(emitted geometry), which is usually much less than the static path on big open maps and more CPU on detailed maps.

### 3.6 Adaptive mode

The `AutomapMarker` background thread runs the same BSP walk, with a full 360° clipper, the near plane and the left/right frustum (for marking automap lines). It does this once per queued tic position and records its time in microseconds and the seg and line counts [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:114-182, 201-280. The stopwatch is restarted once per batch of dequeued positions, not once per position [H] AutomapMarker.cs:124, 178. A position is offered every world tick [H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:230-236. In `Adaptive` mode it is enqueued only when the queue is empty [H] AutomapMarker.cs:108-112. This thread runs in **all** modes, because it also drives automap line marking. In non-adaptive modes it sleeps the rest of the tic; in adaptive mode it `Thread.Yield()`s [H] AutomapMarker.cs:154-161.

`UseBspBasedOnHeuristic` [H] LegacyWorldRenderer.BspHeuristics.cs:19-88:
- Takes the median of a sliding window (`TimeWindow`, default 10).
- Compares it with `TimeThreshold` (2200 µs).
- Reads `SwitchCount` (3), but **as written it has no effect**. A sample below the threshold sets `belowCount++` and `aboveCount = 0`, and makes `shouldUseBsp` true already. A sample above does the reverse. So neither override branch (`!shouldUseBsp && below >= N`, `shouldUseBsp && above >= N`) can ever fire [H] LegacyWorldRenderer.BspHeuristics.cs:48-65. The switch happens on the first median sample that crosses the threshold outside the dead-band. *(Verifier: corrected from "requires SwitchCount consecutive samples".)*
- Applies a ±`HysteresisPercent` (5%) dead-band. The code comment says "10%", but the config default is 0.05 [H] LegacyWorldRenderer.BspHeuristics.cs:67-75.
- Processes a new sample only if it is more than 500 µs old and has a new id [H] LegacyWorldRenderer.BspHeuristics.cs:40-46.
- **Never** picks BSP if the visible seg count exceeds `SegThreshold` (8000).

Unlike the background walk, the renderer's own BSP walk does not test segs against the clipper, so the background timing is only a proxy for the real cost.

---

## 4. Sprite/entity gathering (both modes)

`RenderEntity` [H] LegacyWorldRenderer.cs:271-293:
- Skips invisible frames and entities flagged `Invisible`, `NoSector`, `RenderStyle.None`, or "self when not chasecam".
- Applies the back half-plane test using the occluder point 32 units behind the camera.
- Computes the 2D distance squared and rejects beyond `MaxDistanceSquared`.
- Sets `LastRenderGametick`.
- Calls `EntityRenderer.RenderEntity`.

`renderIndex` cycles 0..99 (`EntityRenderIndexMax = 100`) and is used purely as a per-sprite depth bias against z-fighting [H] LegacyWorldRenderer.cs:34, 168-173; [H] Entities/EntityProgram.cs:168-169.

- Sprite rotation is chosen on the CPU each tic from the diamond angle between the view position and the entity [H] Entities/EntityRenderer.cs:200-206.
- Billboard quads are expanded in the vertex shader with `DrawArraysInstanced` (4 corners from `gl_VertexID`), along a `viewRightNormal` uniform [H] Entities/EntityProgram.cs:100, 142-163. That uniform is set per frame from `Camera.Direction.XY.RotateRight90()` [H] Entities/EntityRenderer.cs:369.
- There is a GPU-side **distance fade**: `FadeDistance = maxDistanceSquared/2`. Alpha fades once `renderDistSquared > maxDistanceSquared - fadeDistance`, that is, beyond MaxDistance/√2 ≈ 4243 units at the default [H] Entities/EntityRenderer.cs:387-389; [H] Entities/EntityProgram.cs:418-419.

There is no per-actor-class distance cull like UZDoom's `distancecheck`. The only limit is the single global `MaxDistance`.

**Inference:** entity gathering is O(entities in range) per tic in static mode and O(entities in visible sectors) in BSP mode. Neither mode checks per-sprite occlusion.

---

## 5. Sky rendering

Sky geometry is the real wall and flat polygons that carry sky textures:
- Ceiling skies are extruded to `MaxSky = 16384` [H] GeometryRenderer.cs:46, 1825.
- In static mode they live in a persistent per-sky-component VBO owned by the static cache's own sky renderer. It is patched in place with `UploadSubData` [H] Geometry/Static/SkyGeometryManager.cs:39-72, 127-157; StaticCacheGeometryRenderer.cs:565-611.
- In dynamic and BSP mode they are appended each tic [H] GeometryRenderer.cs:1839-1844.

`LegacySkyRenderer.Render` handles each sky component (key = texture, options, offset; capped at 255) [H] Core/Render/OpenGL/Renderers/Legacy/World/Sky/LegacySkyRenderer.cs:54-117:
1. `glClear(STENCIL)` (full-screen).
2. Draws the sky geometry with color writes off, stencil = index (depth writes on).
3. Draws a 64×64-quad sphere (24,576 vertices) with depth test **off**, stencil EQUAL.
4. Draws a second sphere if the sky has a foreground layer.

Sphere sizes are at [H] Sky/Sphere/SkySphereRenderer.cs:19-20, 48-79, and the component path at [H] Sky/Sphere/SkySphereComponent.cs:50-69. Sky runs *first* in the frame, before world geometry [H] LegacyWorldRenderer.cs:346-350. Static mode renders two independent sky passes: `m_geometryRenderer.RenderSkies` for dynamic and `RenderStaticSkies` for static. The static cache owns its own `LegacySkyRenderer` [H] StaticCacheGeometryRenderer.cs:49, 76, 898-901.

The sphere MVP is built from `Camera.Direction` and a mono `PerspectiveFov` [H] SkySphereRenderer.cs:81-103.

**Inference (cost):**
- Each sky component costs one full-screen stencil clear, plus a geometry pass, plus a sphere pass.
- Fragment work for the sphere is limited to stencil-marked pixels.
- Sky is drawn before opaque world geometry, so every sky pixel later covered by a wall has been shaded for nothing. GZDoom-style "sky last" ordering would avoid that.
- With one or two sky textures this cost is small.

---

## 6. "Portals" = flood fill only

`PortalRenderer` contains only `FloodFillRenderer` instances, one static and one dynamic [H] Geometry/Portals/PortalRenderer.cs:29-72. It emulates Doom's behaviour for missing upper or lower textures where the floor or ceiling "bleeds" across the gap:

- **Which sides flood** is decided at load and on change: a lower flood needs a lower texture ≤ the null-compat index *and* a floor height step. An upper flood is similar [H] Core/World/Static/StaticDataApplier.cs:44-79.
- **Geometry.** Each flooding side produces a wall quad (6 vertices). Planes add 2 extra projected quads offset by `MaxTextureHeight`. There is also an "alt" quad for when the camera passes the plane [H] PortalRenderer.cs:95-243; [H] FloodFill/FloodFillRenderer.cs:155-272.
- **Shading.** The fragment shader ray-casts from the `camera` uniform through the fragment to the horizontal plane at `planeZ`, and samples the flat texture at the hit point [H] FloodFill/FloodFillProgram.cs:121-149. The vertex shader collapses the quad to the origin when the camera is outside `[minViewZ, maxViewZ]`, to match Doom [H] FloodFillProgram.cs:82-86.
- **Draw order.** It draws immediately after the sky, with polygon offset pushing it behind middle textures [H] LegacyWorldRenderer.cs:429-448.
- **Static mode.** The static flood renderer keeps per-key VBO slots: updates are `UploadSubData`, and cleared slots are zeroed and recycled through a free list [H] FloodFillRenderer.cs:95-153, 274-315. It draws **all** flood quads in the map, one `DrawArrays` per array texture [H] FloodFillRenderer.cs:321-354.
- **BSP mode.** The dynamic flood renderer is cleared each tic and refilled as visible sides are emitted [H] PortalRenderer.cs:59-72; [H] GeometryRenderer.cs:845-850.

**Inference (cost):**
- One extra quad per flooded side.
- One ray-plane intersection per fragment, which is cheap ALU.
- Like the static cache, it is drawn unculled in default mode.

Line, sector, skybox and mirror portals are **not implemented**. The relevant ZDoom specials are only name-mapped to enum values [H] Core/Maps/Specials/ZDoom/ZDoomActions.cs:127-128, 227-228, 293-294, 333-334, and no code consumes them (see Absent).

---

## 7. 3D floors (Sector3D)

The model:
- Each `Sector3D` owns a `FakeSector` with fake top and bottom planes and cloned `Lines` (`CreateSector3DLines`), plus an optional flipped fake sector for swimmable or render-inside floors [H] Core/World/Geometry/Sectors/Sector3D.cs:137-200.
- Render style is `Normal`, `Translucent` (alpha < 1) or `Add` [H] Sector3D.cs:172-178.
- The 3D-floor code paths are globally gated by `WorldStatic.Sector3D`, which is true only if the map defines any 3D floors [H] Core/World/Special/SpecialManager.cs:742.

**Static mode:** at load every 3D floor's planes and walls go into the static cache (`AddSectors3D` → `AddSector3D` → `AddSectorPlanes3D` and `RenderSectorLine3D`) [H] StaticCacheGeometryRenderer.cs:138-146; [H] Geometry/Static/StaticCacheGeometryRenderer.Sector3D.cs:18-53, 57-94. When a control sector moves, `World_SectorMove` recomputes the parent's 3D height ordering (`Sector3D.SetHeights3D(sector, SetHeightsMode.Update)`) [H] StaticCacheGeometryRenderer.cs:947-963. Moving 3D floors are linked into the render blockmap under the *parent* sector's islands (`LinkDynamic(world, Sector3D)`) [H] BlockMap.cs:292-300. In `RenderSectors`, the per-island early-out is disabled for 3D sectors because several fake sectors share an island [H] LegacyWorldRenderer.cs:227-230.

**BSP mode:** when a subsector's parent sector is first reached, each of its `Sectors3D[i].FakeSector` is rendered through `RenderSector` (all lines, all flats) [H] GeometryRenderer.cs:464-468. Culling is therefore parent-sector granular.

**Walls:**
- `RenderSectorLine3D` / `RenderSide3D` pick the parent-side 3D plane list (merged with the opposite side's list for translucent floors).
- `RenderWallSlices3D` cuts the wall into slices between sorted 3D planes, like GZDoom's 3D-floor wall splitting.
- The relevant code is at [H] Geometry/GeometryRenderer.Sector3D.cs:41-105, 107-165, 166-344.
- Back-face selection for 3D walls tests the current position and adds the back side when the two positions straddle the line. In vanilla-render mode, opaque fake walls are always rendered from both sides [H] GeometryRenderer.cs:712-732.

**Occlusion:** 3D floors **never** add to the view clipper. `RenderBlock.IsBlocked` only looks at real sector planes and textures [H] RenderBlock.cs:30-64, and no 3D-floor reference exists in RenderBlock.cs or ViewClipping/ (grep found nothing).

**Transparency:** translucent 3D floors use the shared weighted-OIT transparent pass, not sorting [H] LegacyWorldRenderer.cs:586-695.

**Inference:** in default mode 3D floors cost nothing on the CPU per tic unless they move. Every 3D slice of every 3D floor in the map is drawn every frame on the GPU.

---

## 8. GPU-side culling / queries

`Core/Render/OpenGL/Queries` contains only `GLQuery` (generate and delete) and `GLElapsedTimeQuery` (`QueryTarget.TimeElapsed`) [H] Core/Render/OpenGL/Queries/GLQuery.cs:7-41; Queries/GLElapsedTimeQuery.cs:32-37. `grep -rn "GLElapsedTimeQuery"` outside its own file found no users. There is no occlusion query, conditional render, indirect draw or compute culling (see Absent).

The only GPU-side "culling" mechanisms are:
- Hardware back-face culling (`GL.CullFace` toggled in clip passes [H] LegacyWorldRenderer.cs:512-526, 550-552).
- The vertex-shader collapse of flood quads [H] FloodFillProgram.cs:82-86.
- The sprite distance fade [H] EntityProgram.cs:418-419.
- In vanilla-sprite-clip mode, the plane-clip and wall-clip framebuffers that sprites sample to emulate software clipping. This is an extra full geometry re-render per frame into two FBOs [H] LegacyWorldRenderer.cs:410-411, 450-584.

---

## Absent / not done

| Technique | Search | Result |
|---|---|---|
| Occlusion queries / conditional render | `grep -rn "SamplesPassed\|AnySamples\|BeginConditionalRender\|ConditionalRender" Core` | nothing |
| Indirect draw / GPU compute culling / Hi-Z | `grep -rn "MultiDrawIndirect\|DrawElementsIndirect\|DrawArraysIndirect\|ComputeShader\|DispatchCompute\|HiZ" Core` | nothing |
| Side and top/bottom frustum planes for world culling | `BoxInFront` uses Near only [H] FrustumPlane.cs:17-20. `PointInFrustum` (L/R) is used only by AutomapMarker [H] AutomapMarker.cs:254-256. `SphereInFrustum` has no callers (grep) | Not used for rendering |
| Clipper FOV ranges | `ViewClipper` users: `AddLine` is called only in GeometryRenderer.cs:438-439 and AutomapMarker.cs:244 | 360° only |
| Seg-vs-clipper test in render BSP | GeometryRenderer.cs:417-448 has no `InsideAnyRange` | Not done (intentional, per comment) |
| PVS / REJECT for rendering | `grep -rni reject Core`: only map loading and `m_lineOfSightReject` in WorldBase.cs:189, 263-268, used by the LOS check at WorldBase.cs:2761 | Gameplay LOS only |
| Line/sector/skybox/mirror portals | `grep -rniE "lineportal\|skybox\|SkyViewpoint\|StackedSector\|SetPortal\|Line_Mirror"`: only name→enum mapping in ZDoomActions.cs and ActionSpecialHelper.cs:258-266, 410. GLDEFS `skybox` blocks are parsed into `GldefsDefinition.Skyboxes` (GldefsDefinition.cs:30, 84-85, 140-177), but nothing reads that dictionary. `PortalStencilProgram` has no references outside its own file | Not implemented |
| Per-class sprite distance cull (`distancecheck`) | No equivalent; only global `MaxDistance` | Absent |
| Multithreaded render traversal | The render traversal is synchronous in `PerformRender`. The only render-related thread is AutomapMarker (AutomapMarker.cs:70). Its visibility result is not consumed by the renderer; only its timing and seg count feed the adaptive heuristic | Absent |
| Per-frame visibility | Traversal gated by `NeedsRenderTickChange` | Per tic only |
| Per-frame dynamic VBO upload | `BufferObject.Upload()` is a no-op while `Uploaded` is set, and that flag is reset only by `Add*`/`Clear` on a tic change (BufferObject.cs:58-101, 141-145) | Per tic only |

---

## Perf-relevant settings

| Setting | Default | Effect | Citation |
|---|---|---|---|
| `Render.Mode` | `Static` | Static cache + blockmap / BSP / Adaptive | [H] Core/Util/Configs/Components/ConfigRender.cs:52-60, 91-94 |
| `Render.MaxDistance` | 0 (→ 6000; clamp 0-46000) | Blockmap box, dynamic sector cull, entity cull and fade | [H] ConfigRender.cs:134-136; Renderer.cs:347-349; Constants.cs:432 |
| `Render.Adaptive.TimeThreshold` | 2200 µs | BSP vs static switch point | [H] ConfigRender.cs:242-243 |
| `Render.Adaptive.HysteresisPercent` | 0.05 | Dead-band | [H] ConfigRender.cs:245-246 |
| `Render.Adaptive.SegThreshold` | 8000 | Never BSP above this seg count | [H] ConfigRender.cs:248-249 |
| `Render.Adaptive.TimeWindow` | 10 | Median window | [H] ConfigRender.cs:251-252 |
| `Render.Adaptive.SwitchCount` | 3 | Intended as consecutive samples to switch; no effect as coded (see 3.6) | [H] ConfigRender.cs:254-255 |
| `Render.SkyMode` | `Dynamic` | Sky texture mapping mode | [H] ConfigRender.cs:202-204 |
| `Render.VanillaRender` | false | Enables plane-clip and wall-clip FBO re-renders (extra geometry passes) | [H] ConfigRender.cs:178-180; LegacyWorldRenderer.cs:102-110, 410-411 |
| `Render.DownScaleVanillaRenderSampleBuffer` | 1 | Clip-buffer resolution divisor | [H] ConfigRender.cs:182-184 |
| `Render.TextureTransparency` | true | Alpha-side emission | [H] ConfigRender.cs:232-233; GeometryRenderer.cs:794-795 |
| `Developer.Render.Lock` | false | Freezes traversal (debug) | [H] Core/Util/Configs/Components/ConfigDeveloper.cs:14-15; LegacyWorldRenderer.cs:310-314 |
| Render blockmap cell | 512 (hard-coded) | Granularity of the dynamic walk | [H] WorldBase.cs:437 |
| Occluder push-back / pitch cutoff | 32 units / 45° (hard-coded) | Half-plane cull | [H] LegacyWorldRenderer.cs:773-787 |

---

## Transfer notes for UZDoom VR

UZDoom 5.0 baseline, as read:
- `RenderViewpoint` loops over `vrmode->mEyeCount`. For **each eye** it does `StartDrawInfo` → `SetupView` → `ProcessScene` [U] src/rendering/hwrenderer/hw_entrypoint.cpp:127-187. `ProcessScene` → `DrawScene` → `CreateScene` [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:1017-1026, 1055-1064. So the full BSP walk and draw-list build run per eye, per frame.
- `CreateScene` seeds the clipper with the horizontal frustum (`SafeAddClipRangeRealAngles(yaw + a1, yaw - a1)`) [U] hw_drawinfo.cpp:470-475.
- `RenderBSPNode` checks only the back child's bbox (`mClipper->CheckBox`) [U] hw_bsp.cpp:947-985.
- `AddLine` back-face culls and `SafeCheckRange`-tests **every seg** [U] hw_bsp.cpp:257-330.
- The clipper is a linked list with a linear `IsRangeVisible` [U] hw_clipper.cpp:121-138.
- Things come per sector through `touching_renderthings` with `validcount`, plus a per-class `distancecheck` cvar and a mapsection test [U] hw_bsp.cpp:566-637, 820-837.
- Flats are processed once per *section* [U] hw_bsp.cpp:872-895.
- Wall, flat and sprite processing can be offloaded to a worker thread (`gl_multithread`, default true) [U] hw_bsp.cpp:43, 109-140, 1022-1043.

| Helion mechanism | UZDoom counterpart | Grade | Notes for stereo / Vulkan |
|---|---|---|---|
| **Tic-rate visibility + GPU interpolation** (traverse at 35 Hz, `prevPos`/`pos` + `timeFrac` in VS) | Per-eye, per-frame `CreateScene` (hw_entrypoint.cpp:127-187) with CPU-interpolated actor and sector positions | **Rebuild** | Biggest CPU win available. It needs every dynamic vertex to carry a previous position, and visibility made conservative over the tic (union of start and end viewpoints, as Helion does with two clippers). For VR it must also cover the head-pose envelope: compute once per tic from a "cyclops" point and inflate by IPD/2 plus expected head translation. A cheaper first step is a **Port**: run `CreateScene` once per frame for a combined stereo frustum (centre eye, FOV widened to cover both eyes) and reuse the draw lists for both eyes with multiview. That removes the 2× per-eye traversal without touching interpolation. |
| **Static whole-map GPU cache, no CPU culling** | UZDoom already keeps level geometry in a persistent vertex buffer (`CheckUpdate(screen->mVertexData, sector)` [U] hw_bsp.cpp:794-797) but builds per-frame draw lists from the BSP walk | **Rework** | Suits Vulkan: put static geometry in device-local buffers and draw with `vkCmdDrawIndirect`, one per texture array or bindless. For VR, pair it with GPU culling (compute frustum + Hi-Z per eye, written to indirect args) instead of Helion's "draw everything". Doom maps are vertex-cheap, but 2 eyes × 4K² × overdraw from texture-ordered submission is the real cost. Keep a depth prepass or a front-to-back sort key. |
| **Render blockmap (512 cells) for dynamic content + MaxDistance** | Not present. UZDoom has per-class `distancecheck` only | **Port** (as an option) | Useful for "huge map" VR modes. Avoid Helion's side effect where moving sectors beyond MaxDistance vanish. |
| **Back half-plane "occlude" test, 32 u behind camera, off above 45° pitch** | Clipper frustum ranges plus `OoBFrustumAngle` | Don't port | In VR the head pitches freely and both eyes diverge. Use a proper per-eye (or union) frustum. |
| **360° dual clipper (current + prev pos), near plane only** | Single clipper seeded with FOV | **Port** (idea) | Seeding UZDoom's clipper with the *union of both eye frusta plus margin* gives the same robustness. A 360° clipper is the fallback if traversal is decoupled from frame rate. |
| **No seg-vs-clipper test** | UZDoom tests every seg | Keep UZDoom | UZDoom's per-seg test produces tighter lists. Under multiview the GPU pays for every emitted seg twice. |
| **Linear clip-span scan** (array) | Linear linked list | **Port** (both) | An array is more cache-friendly than UZDoom's `ClipNode` list. A binary search on query is a trivial further win. |
| **Flats per whole sector** | Per section | Keep UZDoom | Helion's choice inflates GPU work on big sectors. |
| **Island/connectivity cull** | `CurrentMapSections` mapsection test (hw_bsp.cpp:696-697) | Equivalent | Nothing to transfer. |
| **Sky: stencil per component, sphere depth-off, drawn first** | UZDoom draws sky as a portal type (`HWSkyPortal`; not inspected here) | **Rework** | For stereo, the sphere must be rendered per eye with that eye's rotation-only view and projection. Helion's mono `Camera.Direction` + `PerspectiveFov` [H] SkySphereRenderer.cs:81-103 shows the coupling. Draw sky *after* opaque geometry with depth test = far, to avoid shading hidden sky. With multiview, use `gl_ViewIndex` to select the matrices. |
| **Flood-fill missing textures via per-fragment ray-plane** | UZDoom uses its own missing-texture/flat-fill handling in the hw renderer (not inspected) | **Port** | The shader uses a single `camera` uniform [H] FloodFillRenderer.cs:365 and FloodFillProgram.cs:121-138. For stereo it must be the per-eye camera position (`gl_ViewIndex`-indexed), or the bleed will show stereo disparity errors. Otherwise cheap. |
| **No real portals** | Full portal system in UZDoom | n/a | Helion offers nothing to reuse here. UZDoom's recursive portal `DrawScene` re-runs the BSP walk per portal per eye, so it is the main CPU multiplier to watch under stereo. |
| **3D floors: wall slicing, culled at parent-sector granularity, never occlude** | UZDoom 3D floors (`x3d` paths in hw_walls/hw_flats; not inspected) | Equivalent | Nothing to transfer. 3D-floor occlusion in the clipper would be a new feature in either engine. |
| **Sprites: CPU gather per tic, instanced quad expansion in VS, GPU distance fade** | `HWSprite::Process` per sprite per eye on the CPU | **Port** | VS expansion along a shared `viewRightNormal` (from camera yaw, [H] EntityRenderer.cs:369; would be head yaw, not per-eye) keeps billboards consistent between eyes. That is good for VR comfort, and it moves per-sprite CPU work off the per-eye path. |
| **Adaptive mode via background BSP timing** | None | Not worth porting | On a VR-class PC the GPU is the bottleneck. A simpler heuristic would do. |

**Inference:** at 90-120 Hz stereo, UZDoom spends roughly `2 eyes × fps` BSP walks plus draw-list builds per second (e.g. 240/s at 120 Hz), plus more for each portal. Helion's architecture spends 35/s. Most of that gap can be closed without Helion's full interpolation rewrite. Do one walk per frame with a combined stereo frustum, submit once with multiview, and then optionally cache draw lists across frames when the head moves less than a threshold.

---

## Open questions

- ~~Whether the `DynamicVertex` buffers are re-uploaded every frame or only on tic change.~~ **Resolved by the verifier: only on tic change.** See "Dynamic buffers are uploaded once per tic" in §1. The upload is a full `glBufferData` (no orphaning or persistent mapping) on the first frame after the tic [H] ArrayBufferObject.cs:24-29.
- Exact behaviour of `StaticCacheGeometryRenderer.World_SectorMoveComplete` (lines 1120-1212), i.e. re-insertion cost and free-list reuse. Partly answered by the verifier: re-add normally rewrites the plane's or wall's own zeroed slot with a sub-upload. The free list or a whole-VBO re-upload is used only when the slot is missing or too small (see §2.1). Frequency in practice was not measured.
- Whether the half-space margin in BSP mode can be exceeded by fast yaw within one tic at very high FOV. This is reasoning only: a near-plane half-space gives (180° − HFOV)/2 margin per side, about 30° at 120° HFOV, and the case was not tested.
- How UZDoom renders skies (`HWSkyPortal`) and missing-texture fills. Not opened; cited only by name.
- `WorldTriangulator.HandleSubsector` costs for flats regenerated each tic in BSP mode when sectors move. Not measured.
