# H1 — Helion world geometry pipeline and GPU submission

Helion's world renderer has two paths, and the default setting chooses the hybrid. In the **static cache** path (`render.mode = Static`, the default), every non-moving wall and flat in the map is triangulated once at map load. The triangles go into one GL vertex buffer per (geometry type × texture array or standalone texture), and **every frame the whole buffer is drawn**: one `glDrawArrays` per group, no visibility culling. Only the small **dynamic** set is rebuilt on the CPU and streamed with `glBufferData`: moving sectors, scrolling sides and flats, and everything when the camera is in a Boom transfer-heights view. That rebuild happens **once per game tic (35 Hz), not once per frame**. Frames between tics re-issue the same buffers, and the shader interpolates `prevPos → pos` with `timeFrac`. Lighting, sector colour and fog are not baked into vertices. Each vertex holds a packed sector index into small R8UI/RGBA32F texture buffers, which the CPU patches in place when an event fires. The result: per-frame CPU work grows with the number of texture groups, per-tic CPU work grows with the number of dynamic sectors inside the render distance, and GPU vertex work grows with **map size**. All of this runs on GL 3.3-level features (VAO + `glDrawArrays`, sampler2DArray, TBOs, glUniform). The only GL 4.x features are persistent-mapped TBOs, `glClipControl` and one `glMemoryBarrier` in the fuzz pass, all optional [H] Client/Client.cs:282-288.

---

## 1. Frame flow: from the top-level call to GL draws

### 1.1 Call chain

`Client` calls `m_layerManager.Render(m_window.Renderer)` [H] Client/Client.cs:335. `GameLayerManager.Render` calls `m_renderer.Default.Render(m_renderDefaultAction)` [H] Core/Layer/GameLayerManager.cs:755-759. `GLSurface.Render` lets the layers record a command list and then executes it synchronously with `m_renderer.Render(ctx.Commands)` [H] Core/Render/OpenGL/GLSurface.cs:24-29. The world layer records a `DrawWorld` command that carries `world.Gametick` and `m_context.InterpolationFrac` [H] Core/Render/OpenGL/GLWorldRenderContext.cs:43.

`Renderer.Render(RenderCommands)` [H] Core/Render/Renderer.cs:546-618 does the following:
1. Binds the world FBO, then binds six texture buffers to fixed units every frame: colormap, sector colormap, sector fog, sector light, map line data, line heights [H] Core/Render/Renderer.cs:550-556, 638-666. Each bind is `glActiveTexture` + `glBindTexture(TEXTURE_BUFFER)` + `glTexBuffer` [H] Core/Render/OpenGL/Textures/GLBufferTextureStorage.cs:57-62.
2. Walks the command list and dispatches `RenderCommandType.World` to `HandleRenderWorldCommand` [H] Core/Render/Renderer.cs:562-609.

`HandleRenderWorldCommand` [H] Core/Render/Renderer.cs:884-920 does the following:
- Computes `ShaderUniforms` once per frame: two MVP matrices (with and without pitch), colormap and colour-mix state, light mode, gamma and max distance [H] Core/Render/Renderer.cs:301-371, 895.
- Switches to reversed-Z (`glClipControl` + `GL_GREATER`) if it is enabled [H] Core/Render/Renderer.cs:902-908.
- **Flushes the dirty-sector lists into the light, colour and fog texture buffers (`UpdateBuffers()`)** [H] Core/Render/Renderer.cs:911.
- Calls `m_worldRenderer.Render(...)` → `LegacyWorldRenderer.PerformRender` [H] Core/Render/OpenGL/Renderers/WorldRenderer.cs:24-27.

### 1.2 `LegacyWorldRenderer.PerformRender`: tic gating and draw order

[H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:295-427

```csharp
m_renderStatic = m_config.Render.Mode.Value != AdaptiveRenderMode.Bsp && renderInfo.TransferHeightView == TransferHeightView.Middle && !m_lastUseBsp;
...
var renderTickChange = !m_config.Developer.Render.Lock.Value && NeedsRenderTickChange(world, renderInfo.TransferHeightView);
...
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
[H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:300, 310, 331-341

`NeedsRenderTickChange` returns true only when `m_lastTicker != world.GameTicker`, when the static/BSP mode flips, or when the transfer-heights view changes [H] LegacyWorldRenderer.cs:143. `Clear()` empties the dynamic per-texture vertex lists only when `renderTickChange` is true [H] LegacyWorldRenderer.cs:789-799. **Visibility traversal and dynamic geometry generation therefore run at the tic rate. Every other frame re-draws the previous tic's buffers.** `RenderWorldData.Draw()` calls `Vbo.Upload()`, which returns at once if the data has not changed since the last upload [H] Core/Render/OpenGL/Buffer/BufferObject.cs:94-101, Core/Render/OpenGL/Renderers/Legacy/World/Data/RenderWorldData.cs:45-68.

Opaque draw order in the default true-colour path (`!m_vanillaRender`) [H] LegacyWorldRenderer.cs:345-378:
1. `RenderSkies`: dynamic sky geometry, one stencil pass per sky texture.
2. `RenderFloodFill`: portal/flood-fill planes, drawn with polygon offset.
3. `RenderStaticSkies` (static mode only).
4. Interpolation program: dynamic walls, then dynamic flats.
5. Static program (static mode only): static walls, then static flats.
6. `RenderTwoSidedMiddleWalls`: dynamic two-sided middle textures and 3D middles, then their static counterparts [H] LegacyWorldRenderer.cs:745-766.
7. Opaque entities, primitives, then `RenderTransparent`, which runs weighted-blended OIT (accumulate pass + composite pass) for dynamic alpha, static alpha and entities [H] LegacyWorldRenderer.cs:586-695.

The vanilla-render path (`render.vanillarender`, default false [H] Core/Util/Configs/Components/ConfigRender.cs:180) adds a plane-clip pass and a wall-clip pass. These re-draw dynamic and static flats/walls into off-screen sprite-clip buffers [H] LegacyWorldRenderer.cs:450-584.

### 1.3 When each piece of work runs

| Work | When | Evidence |
|---|---|---|
| Texture preload and texture-array build | Map load, skipped if `SameAsPreviousMap` | [H] GeometryRenderer.cs:201-204, 238-242; GLTextureArrayBuilder.cs:21-48 |
| Static VBO build (all walls and flats not flagged `Movement`) | Map load | [H] StaticCacheGeometryRenderer.cs:103-172 |
| Per-side/per-sector CPU vertex caches (`m_vertexLookup` etc.) sized | Map load | [H] GeometryRenderer.cs:184-195 |
| Sector light / colormap / fog TBO creation and fill | Map load (re-used and re-filled if same map) | [H] Renderer.World.cs:74-104, 156-179 |
| Colormap TBO (palette mode or invuln emulation only) | Startup | [H] Renderer.cs:238-255; Client/Client.cs:420 |
| Static VBO patch (glBufferSubData of one surface) | On world events: sector move start/complete, side/plane texture change, plane transform, fog toggle | [H] StaticCacheGeometryRenderer.cs:113-118, 1360-1406, 1587-1593 |
| TBO patch (light/colormap/fog/line heights) | Per frame, only for sectors flagged by events since the last frame | [H] Renderer.World.cs:386-421 |
| Blockmap or BSP traversal, dynamic vertex gen, dynamic VBO re-upload | Per tic (35 Hz) | [H] LegacyWorldRenderer.cs:143, 331-341, 789-799 |
| Uniform upload, draw calls | Per frame | [H] LegacyWorldRenderer.cs:345-427, 833-911 |

---

## 2. The static cache path

### 2.1 What counts as "static"

The static renderer subscribes to `SectorMove` (3D floors only), `SectorMoveStart`, `SectorMoveComplete`, `SideTextureChanged`, `PlaneTextureChanged`, `SectorPlaneTransformed` and `SectorFogColorChanged` [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Static/StaticCacheGeometryRenderer.cs:110-118.

At load it adds every floor/ceiling whose `Dynamic` flags do not include `Movement` (`IgnoreFlags = SectorDynamic.Movement`) and every line [H] StaticCacheGeometryRenderer.cs:36, 127-160. The `SectorDynamic` flags are `None, Movement=1, TransferHeights=2, Scroll=4, ScrollY=8` [H] Core/World/Static/SectorDynamic.cs:6-13. At load, `StaticDataApplier` marks only scrolling sides (`Scroll`/`ScrollY`) and links them into the render blockmap's `DynamicSides` [H] Core/World/Static/StaticDataApplier.cs:14-42. During load, `Movement` is suppressed (`IsLoading`) [H] StaticDataApplier.cs:83-84. Scrolling flats are flagged `Scroll` when their special is added [H] Core/World/Special/SpecialManager.cs:464-469.

So **scrolling sides and flats are still put into the static cache**. Two-sided upper and lower walls are skipped only when an adjacent plane is flagged `Movement` and the side itself is dynamic (`Side.IsDynamic` = any flag set) [H] StaticCacheGeometryRenderer.cs:405-409. A scrolling middle texture is excluded only when it scrolls in Y or has transparent pixels [H] StaticCacheGeometryRenderer.cs:549-563. The code comment there says the static copy exists "for when the camera is outside the dynamic distance". Inside that distance, the dynamic copy is drawn first (dynamic walls precede static walls, [H] LegacyWorldRenderer.cs:358-370).

**Inference:** within range the static duplicate is hidden only by a strict depth test against bit-identical depth from a *different* shader program, and no `invariant` qualifier is used anywhere (`grep -rn invariant Core` finds only comments). Z-fighting on scrolling walls is therefore possible on some drivers.

### 2.2 Vertex generation is shared with the dynamic path

The static renderer does not have its own triangulator. It calls `GeometryRenderer` with `SetBuffer(false)` (generate but do not enqueue) and `SetRenderMode(Dynamic, Middle)` [H] StaticCacheGeometryRenderer.cs:120, 1157-1158; GeometryRenderer.cs:523-529. It then converts the returned 52-byte `DynamicVertex[]` into 32-byte `StaticVertex`, dropping `prevPos`/`prevUV`:

```csharp
staticVertices.Data[staticStartIndex + i] = new StaticVertex(v.X, v.Y, v.Z, v.U, v.V,
    v.SurfaceOptions, v.LightLevelAdd, v.RenderOptions);
```
[H] StaticCacheGeometryRenderer.cs:613-626

### 2.3 Vertex layouts (measured)

| Struct | Fields | Size | Citation |
|---|---|---|---|
| `DynamicVertex` | pos(3f) uv(2f) surfaceOptions(f) lightLevelAdd(f) prevPos(3f) prevUV(2f) renderOptions(f) | 13 floats = **52 B** | [H] Core/Render/OpenGL/Renderers/Legacy/World/DynamicVertex.cs:6-35 |
| `StaticVertex` | pos(3f) uv(2f) lightLevelAdd(f) surfaceOptions(f) renderOptions(f) | 8 floats = **32 B** | [H] Core/Render/OpenGL/Renderers/Legacy/World/StaticVertex.cs:6-26 |
| `SkyGeometryVertex` | pos(3f) prevZ(f) | **16 B** | [H] Core/Render/OpenGL/Renderers/Legacy/World/Sky/Sphere/SkyGeometryVertex.cs:6-16 |
| `FloodFillVertex` | pos(3f) planeZ minViewZ maxViewZ prevZ prevPlaneZ surfaceOptions renderOptions mapId | 11 floats = **44 B** | [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Portals/FloodFill/FloodFillVertex.cs:7-36 |
| `TriangulatedWorldVertex` (CPU scratch) | x y z u v prevZ prevU prevV | 32 B | [H] Core/Render/Common/Shared/World/TriangulatedWorldVertex.cs:5-14 |

All are `Pack = 1` sequential layouts. Three of the "float" attributes are actually bit-packed integers reinterpreted with `*(float*)&packed` [H] Core/Render/OpenGL/Renderers/Legacy/World/VertexOptions.cs:9-35:

- `SurfaceOptions`: alpha (8 b), topLeft (1), addAlpha (1), upper (1), lower (1), overrideLightIndex (1), **texture-array layer (masked to 20 b but shifted to bit 13, so only the low 19 bits survive in the 32-bit word)**.
- `RenderOptions`: light-buffer index (22 b), uvFlags (2), per-wall light level (8).
- `LightLevelAdd`: sign (1), fake-contrast add (8), **line id / map id (23 b)**.

The shaders unpack them with `floatBitsToInt` [H] Core/Render/OpenGL/Renderers/Legacy/World/Shader/VertexFunction.cs:40-65. The attributes are declared as `float` (`VertexAttribPointer`, not `IPointer`; [H] Core/Render/OpenGL/Vertex/Attributes.cs:186-188).

**Inference:** a 20-bit layer shifted left by 13 reaches the exponent bits, so some packings form NaN bit patterns. These survive only because GL passes float attributes through unconverted. A Vulkan port should declare these as `R32_UINT`.

**Topology.** Walls are 6 non-indexed vertices per quad (two triangles; [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/GeometryRenderer.cs:2200-2321). Flats are triangle fans of each BSP subsector, expanded to a triangle list. A subsector with *n* segs yields `(n−2)·3` vertices [H] GeometryRenderer.cs:1871-1887, 1951-1961. No index buffer exists anywhere: `grep -rn "DrawElements\|ElementArrayBuffer" Core Client` returns 0 hits.

**Measured cost:** each static wall quad is 6 × 32 = **192 B**, and each dynamic wall quad is 6 × 52 = **312 B** per tic upload. Per vertex, Helion's static format is the same size as UZDoom's `FFlatVertex` (x, z, y, u, v, lu, lv, lindex = 32 B, [U] src/common/rendering/hwrenderer/data/flatvertices.h:35-41). UZDoom's flats are indexed, though.

### 2.4 Storage: one VBO+VAO per (GeometryType, texture-array-or-texture)

`GeometryType` has 9 buckets: `Wall, TwoSidedMiddleWall, Middle3D, Flat, Fuzzy, Translucent, TranslucentAdd, TranslucentColorAdd, FogBarrier` [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/GeometryType.cs:6-19. `AllocateGeometryData` creates one `StaticVertexBuffer<StaticVertex>` (`GL_STATIC_DRAW`) plus one VAO through `VertexPipeline` for each new key [H] StaticCacheGeometryRenderer.cs:680-708; Core/Render/OpenGL/Buffer/VertexBufferObject.cs:51-58; Core/Render/OpenGL/VertexPipeline.cs:24-31.

The key is `(type, worldArrayTextureHandle(textureHandle, repeatY), repeatY)` [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Static/GeometryTextureLookup.cs:18-29. Textures that share a texture array therefore share one VBO and one draw.

World texture arrays are built at load by **exact image dimension**: textures are sorted by height then width, and one `GL_TEXTURE_2D_ARRAY` is created per distinct dimension (split further if a bucket exceeds `GLInfo.MaxArrayLayers`), separately for repeat and ClampY [H] Core/Render/OpenGL/Texture/Legacy/GLTextureArrayBuilder.cs:21-48, 165-192, 218-240. **Animated textures and textures with brightmaps are excluded** and remain standalone single-layer 2D arrays [H] GLTextureArrayBuilder.cs:29-33, 201-205; Core/Render/OpenGL/Texture/Legacy/LegacyGLTextureManager.cs:159-161. The layer index is baked into each vertex (`texture.ArrayIndex` → `PackSurface`) [H] GeometryRenderer.cs:910, 2235.

The CPU-side `DynamicArray<StaticVertex>` mirror is kept for the life of the map (`BufferObject.Data`, [H] Core/Render/OpenGL/Buffer/BufferObject.cs:21). Sub-range updates are copied from it.

### 2.5 Upload and in-place updates

At the end of `UpdateTo`, every geometry list calls `UploadIfNeeded()`, which does `glBufferData(size = Count × 32)` [H] StaticCacheGeometryRenderer.cs:162-169; Core/Render/OpenGL/Buffer/ArrayBufferObject.cs:24-29.

Each surface remembers where it lives through `StaticGeometryData { GeometryData, Index, Length }` [H] Core/World/Static/StaticGeometryData.cs:5-10. This is stored on `Wall.Static` and `SectorPlane.Static`, so each wall and plane object knows its own VBO slice.

**Sector movement (doors/lifts/crushers).**

- `SectorMoveStart` → `HandleSectorMoveStart` does the following [H] StaticCacheGeometryRenderer.cs:983-1027, 1029-1095, 1587-1593:
  - Flags the plane and all of its lines `Movement` and links the sector into the render blockmap's `DynamicSectors` [H] StaticDataApplier.cs:81-105.
  - **Zeroes its static vertices in the CPU mirror and pushes that range with `glBufferSubData`**. This happens for the plane and for every dynamic upper, lower or middle wall and fog barrier on the sector's lines. The geometry collapses to degenerate triangles at the origin; the slot is not freed.
- While moving, the sector is drawn by the dynamic path each tic and interpolated on the GPU.
- `SectorMoveComplete` → `HandleSectorMoveComplete` does the following [H] StaticCacheGeometryRenderer.cs:1120-1200, 1222-1252:
  - Clears the movement flags.
  - Regenerates the plane (`AddSectorPlane(..., update: true)`) and **every line of the sector** (`AddLine(line, true)`).
  - `UpdateVertices` copies into the same slot and issues `glBufferSubData(startIndex, length)` [H] StaticCacheGeometryRenderer.cs:1390-1405.

```csharp
var startIndex = staticGeometry.Index;
var vbo = staticGeometry.GeometryData.Pipeline.Vbo;
CopyVertices(vbo.Data.Data, vertices, startIndex);
if (m_worldReload) vbo.SetNotUploaded();
else { vbo.Bind(); vbo.UploadSubData(startIndex, vertices.Length); }
```
[H] StaticCacheGeometryRenderer.cs:1390-1402

If a surface now needs more vertices (3D-floor slicing), its old slot is returned to `FreeGeometryManager` [H] StaticCacheGeometryRenderer.cs:1374-1382. A new slot is then taken by best-fit search of the free list for that (texture, type, repeat) [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Static/FreeGeometryManager.cs:87-133. Failing that, the vertices are **appended, which forces a full re-upload of that texture's VBO** (`SetNotUploaded`; the in-code TODO notes that sub-data could be used instead) [H] StaticCacheGeometryRenderer.cs:1536-1569. `BufferSubData` also refuses and falls back to a full upload if the CPU array was reallocated [H] ArrayBufferObject.cs:38-69.

**Texture changes (switches, `SetTexture` specials).** `World_SideTextureChanged` zeroes the old slot and puts it on the free list. It then regenerates the line, which lands in the new texture's VBO (free-list slot or append) [H] StaticCacheGeometryRenderer.cs:1265-1307. Planes follow the same path [H] StaticCacheGeometryRenderer.cs:1315-1349. The texture must move VBOs because the array layer is in the vertex (code comment at 1267, behaviour at 1269-1283).

**Animated textures (ANIMDEFS).** These need **no geometry work**. `TextureManager.Tick` rewrites `m_translations[anim.TranslationIndex]` [H] Core/Resources/TextureManager.cs:479-494, and `GetTexture(index)` resolves through that table [H] TextureManager.cs:446-449. Animated textures are standalone textures, not array layers [H] GLTextureArrayBuilder.cs:29-33. `RenderGeometry` re-resolves `m_textureManager.GetTexture(data.TextureHandle)` before binding at draw time [H] StaticCacheGeometryRenderer.cs:870-878, so the next frame binds the new frame's texture.

**Light changes.** These need **no geometry work**. The vertex stores a sector-based index into the light texture buffer, not a light value (§6).

**Scrollers.** Static geometry for scrolling surfaces is frozen at load. The live copy comes from the dynamic path every tic within render distance (§3).

**Persistent mapping or fences for VBOs: none.** Static and dynamic VBOs use only `glBufferData` and `glBufferSubData` [H] ArrayBufferObject.cs:24-36, 66. `GL.BufferData`/`GL.BufferSubData` appear only in `ArrayBufferObject.cs` and `GLBufferTexture.cs` (grep found 4 call sites). Persistent mapping is used only for the TBOs (§6).

### 2.6 Drawing the static cache

```csharp
for (int i = 0; i < geometry.Count; i++)
{
    var data = geometry[i];
    if (data.Pipeline.Vbo.Count == 0)
        continue;
    ...
    GL.ActiveTexture(BindTextures.BoundTexture);
    texture.Bind();
    ...
    GL.ActiveTexture(BindTextures.BrightmapTexture);
    if (brightmapTexture != null)
        brightmapTexture.Bind();
    else
        GL.BindTexture(texture.Target, 0);

    data.Pipeline.Vbo.UploadIfNeeded();

    data.Pipeline.Bind();
    data.Pipeline.DrawArrays();
```
[H] StaticCacheGeometryRenderer.cs:864-892 (`Pipeline.Bind()` = `glBindVertexArray`; `DrawArrays()` = `glDrawArrays(Triangles, 0, Count)`, [H] Core/Render/OpenGL/VertexPipeline.cs:33-38; Core/Render/OpenGL/Buffer/VertexBufferObject.cs:14-21)

Each non-empty group gets exactly one `glDrawArrays` **over its entire VBO, including zeroed (moving or freed) slots** [H] Core/Render/OpenGL/Buffer/VertexBufferObject.cs:14-21. No frustum, distance or portal culling is applied to static geometry: `RenderWalls`/`RenderFlats` just pass the whole list [H] StaticCacheGeometryRenderer.cs:814-827.

**Inference:** the GPU transforms every static vertex of the map, twice per frame in stereo. Near-plane clipping and early-Z discard most of the work. For a 30k-sidedef map at roughly 1.3 visible wall pieces per side, about 230k wall vertices plus a similar number of flat vertices comes to about 0.5M vertices × 32 B ≈ 15 MB of VBO. At 2 eyes × 120 Hz that is about 120M vertex invocations per second. This is affordable on a modern dGPU, but it is wasted work that scales with map size, not view.

---

## 3. The dynamic (non-static) path

### 3.1 When it is used

- **Static mode (default)**, per tic: `IterateBlockmap` walks the 512-unit render blockmap ([H] Core/World/WorldBase.cs:437) over a box of `MaxDistance` around the camera. It renders the sectors in `DynamicSectors` (moving or scrolling sectors, linked by `StaticDataApplier.SetSectorDynamic`) and the sides in `DynamicSides` (scrolling sides), plus entities [H] LegacyWorldRenderer.cs:145-178, 217-269. A coarse half-plane test (`BlockInView`) discards blocks behind the camera [H] LegacyWorldRenderer.cs:198-215. It is disabled when |pitch| > 45° [H] LegacyWorldRenderer.cs:773-787. `MaxDistance` defaults to 0, which means `DefaultMaxDistance = 6000` [H] ConfigRender.cs:136; Core/Util/Constants.cs:432; Core/Render/Renderer.cs:347-349.
- **BSP mode** (`render.mode = Bsp`, or `Adaptive` choosing BSP), **and any frame where the camera is in a transfer-heights top/bottom view**: `m_renderStatic = false`, and `GeometryRenderMode.All` generates *all* visible geometry dynamically through a front-to-back BSP walk [H] LegacyWorldRenderer.cs:299-308, 316. That walk uses a 2D angular view clipper fed by one-sided and blocked segs, and a near-plane-only frustum test (`BoxInFront`) [H] LegacyWorldRenderer.Bsp.cs:17-88; Core/Render/Common/Shared/FrustumPlane.cs:17-20. The static cache is not drawn in that case [H] LegacyWorldRenderer.cs:362-371.
- **Adaptive mode** reads a background thread's BSP timing (`AutomapMarker` runs its own thread, [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:25, 70, 177-180). It switches between modes with hysteresis and a seg-count cap [H] LegacyWorldRenderer.BspHeuristics.cs:19-88.

### 3.2 How vertices are produced

The dynamic path keeps a **per-surface CPU vertex cache** (`m_vertexLookup`, `m_vertexLowerLookup`, `m_vertexUpperLookup`, per side; `m_vertexFloorLookup`/`m_vertexCeilingLookup`, per sector) [H] GeometryRenderer.cs:81-89. It re-triangulates a surface only when change detection fires:

```csharp
if (side.OffsetChanged || m_sectorChangedLine || data == null) {
    int lightIndex = Renderer.GetLightBufferIndex(side, side.Middle, lightLevelSector, out var overrideLightIndex);
    ...
    WorldTriangulator.HandleOneSided(side, offsetSide ?? side, floor, ceiling, texture.UVInverse, ref wall, isFront: isFront);
    if (data == null)
        data = GetWallVertices(wall, ...);
    else
        SetWallVertices(data, wall, ...);
    SetCachedSide(m_vertexLookup, side, data);
}
if (m_buffer) {
    ...
    var renderData = m_worldDataManager.GetRenderData(arrayTexture, geometryType, brightmapTexture);
    renderData.Pipeline.Vbo.AddMemoryCopy(data);
```
[H] GeometryRenderer.cs:904-925

Change detection compares gametick stamps and Z values. `SectorPlane.CheckRenderingChanged` is true if `LastRenderChangeGametick >= LastRenderGametick-1`, if `PrevZ != Z`, or if the plane has render offsets this tic (`RenderOffsets.Gametick != 0`, which covers scrolling flats) [H] Core/World/Geometry/Sectors/SectorPlane.cs:90-102. `Sector.CheckRenderingChanged(gametick)` applies the gametick and Z tests (but not the `RenderOffsets` test) to both planes and the transfer-heights control sector [H] Core/World/Geometry/Sectors/Sector.cs:245-257.

A moving sector's walls and flats are therefore re-triangulated **every tic while moving**. A scrolling flat re-fans **all of its subsectors** every tic [H] GeometryRenderer.cs:1848-1888. Wall triangulation computes both current and previous Z/UV in one call (`HandleOneSided`, `HandleTwoSidedLower/Upper/Middle`) [H] Core/Render/Common/Shared/World/WorldTriangulator.cs:23-252. Flats are fanned from BSP segs by `HandleSubsector` [H] WorldTriangulator.cs:254-368.

**Bug noted:** in the rotated-flat branch, `prevUV` is computed from `uvVertex`, not `prevUVVertex`, so rotated scrolling flats do not interpolate their UV [H] WorldTriangulator.cs:327-341.

Even unchanged surfaces are **re-copied every tic** (`AddMemoryCopy` of 6 × 52 B per wall) into per-texture arrays. In static mode only dynamic sectors and sides reach this code. In BSP mode every visible surface does.

### 3.3 Buffer strategy

`RenderWorldDataManager` holds a `GeometryTypeLookup<RenderWorldDataList>`: one list per `GeometryType` [H] Core/Render/OpenGL/Renderers/Legacy/World/Data/RenderWorldDataManager.cs:11-25. Each list maps `texture.TextureId` → `RenderWorldData` [H] RenderWorldDataList.cs:32-56. Each `RenderWorldData` owns its own VAO plus a `DynamicVertexBuffer<DynamicVertex>` (`GL_DYNAMIC_DRAW`) [H] RenderWorldData.cs:16-20. These objects come from a pool pre-filled with **1024 VAO/VBO pairs** [H] RenderWorldDataManager.cs:22; RenderWorldDataPool.cs:19-25, 38-43.

Upload is `glBufferData(Count × 52 B, ...)`, a full re-specification (orphan), once per touched texture per tic [H] ArrayBufferObject.cs:24-29; BufferObject.cs:94-101. Draw is bind texture + brightmap + VAO + `glBufferData` if dirty + `glDrawArrays` [H] RenderWorldData.cs:45-68.

**Inference:** this is the classic GL 3.3 streaming pattern. It needs no fences because `glBufferData` orphans, but the cost is one driver allocation per texture per tic. There is also no ring buffer, no `MAP_UNSYNCHRONIZED`, and no persistent mapping for vertex data.

---

## 4. Draw call structure and GL feature use

### 4.1 Draws per frame (default static mode, true colour, no vanilla clipping)

Counted from the code above:
- **Skies:** per sky component with geometry: `glClear(STENCIL)`, a depth/stencil-only draw of sky geometry, then the sky sphere draw with `stencil EQUAL` and depth test off [H] Core/Render/OpenGL/Renderers/Legacy/World/Sky/LegacySkyRenderer.cs:81-114. This happens once for the dynamic sky renderer and once for the static one (`RenderStaticSkies`, [H] StaticCacheGeometryRenderer.cs:898-901).
- **Flood fill:** one draw per non-empty `FloodFillInfo` (per texture) [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Portals/FloodFill/FloodFillRenderer.cs:321-354.
- **Dynamic:** one draw per (GeometryType, texture) touched this tic [H] RenderWorldDataList.cs:60-67.
- **Static:** one draw per non-empty (GeometryType, array-or-standalone texture) [H] StaticCacheGeometryRenderer.cs:859-896. The profiler counts these as `GeometryStatic` and `GeometryDynamic` [H] Core/Util/Profiling/Timers/RenderProfiler.cs:5-17.
- **Transparent:** OIT accumulate pass, then composite pass, each issuing the same per-texture draws again for alpha geometry [H] LegacyWorldRenderer.cs:624-688.

**Inference:** the number of static draws equals the number of distinct texture dimensions × {repeat, clamp} × the geometry types they appear in, plus one per animated or brightmapped texture. For typical Doom content that is tens to low hundreds of draws. It does *not* grow with sector or line count. CPU cost per draw is about 8 GL calls.

### 4.2 State changes

- Program switches: at least Interpolation → Static → Interpolation → Static for the opaque world, then up to four more for OIT [H] LegacyWorldRenderer.cs:354-376, 745-766, 624-688.
- On each switch, `SetStaticUniforms` or `SetInterpolationUniforms` re-sends about 26 uniforms through individual `glUniform*` calls (`ProgramUniforms.Set`) [H] LegacyWorldRenderer.cs:833-911; Core/Render/OpenGL/Renderers/Legacy/World/RenderProgramBase.cs:76-106. No UBO is used (`grep UniformBuffer` → 0).
- Per draw: 2× `glActiveTexture`, 2× `glBindTexture`, one `glBindVertexArray`, and in the dynamic path `glBindBuffer` plus unbinds [H] RenderWorldData.cs:45-68.
- Fixed global state: depth test, alpha blending, back-face culling (CCW), first-vertex provoking (needed by the UV-clamp `flat` varyings) [H] Core/Render/Renderer.cs:697-713.

### 4.3 GL version and features

- The context is GL 3.3 core minimum. The client probes from 4.6 down to 3.3 [H] Core/Render/OpenGL/Util/GLVersion.cs:13; Client/Client.cs:262-279.
- Optional: `ARB_clip_control`/4.5 for reversed-Z, `ARB_buffer_storage`/4.4 for persistent TBO mapping, and `ARB_shader_image_load_store`/4.2 for one `glMemoryBarrier(FRAMEBUFFER)` in fuzz [H] Client/Client.cs:282-288.
- All world shaders are `#version 330` [H] Core/Render/OpenGL/Renderers/Legacy/World/InterpolationShader.cs:23; StaticShader.cs:23.
- **Used:** VAOs (legacy `glGenVertexArrays`, [H] VertexPipeline.cs:27), `glDrawArrays`, `sampler2DArray` for every world texture, `samplerBuffer`/`usamplerBuffer` TBOs, `glBufferStorage(MAP_WRITE|MAP_PERSISTENT)` for TBOs only [H] Core/Render/OpenGL/Textures/GLBufferTexture.cs:34-37, and MRT for OIT and vanilla plane-clip.
- **Not used:** see §8.

---

## 5. Shaders: per-vertex vs per-pixel work

The static and interpolation shaders are generated from shared C# string snippets [H] InterpolationShader.cs:22-95; StaticShader.cs:22-91. There are 15 world-geometry program variants (7 interpolation + 8 static) plus 5 entity variants [H] Core/Render/OpenGL/Renderers/Legacy/World/WorldRenderPrograms.cs:8-28. The vertex shaders differ only in `mix(prevPos, pos, timeFrac)` / `mix(prevUV, uv, timeFrac)`, which the interpolation shader does and the static shader does not [H] InterpolationShader.cs:63-68; StaticShader.cs:59-65.

**Vertex shader per vertex** [H] VertexFunction.cs:14-65; Core/Render/OpenGL/Renderers/Legacy/World/Shader/LightLevel.cs:24-28; SectorColorMap.cs:15-29:
- Unpacks the three packed words.
- `texelFetch(sectorLightTexture, lightIndex)` (R8UI), plus `vertexLightLevel + lightLevelAdd × (1 − fog.a)`, clamped.
- If sector colour is enabled: `texelFetch(sectorColormapTexture, index)`. If sector fog is enabled: `texelFetch(sectorFogTexture, index)`. Both are uniform-gated.
- `dist2D = (mvpNoPitch·pos).z|w` and `dist3D = (mvp·pos).z|w`. `dist3D` repeats the same `mvp·pos` product as `gl_Position` (a compiler can share it), so the genuinely extra mat4×vec4 is the `mvpNoPitch` one for `dist2D` [H] LightLevel.cs:30; VertexFunction.cs:7; InterpolationShader.cs:76.
- Pixel-gap correction: `textureSize(boundTexture)` and UV clamp bounds sent as `flat` varyings from the provoking vertex.
- **15 `flat` varyings plus 4 smooth varyings** (`uvFrag`, `depthFrag`, `dist2D`, `dist3D`) in the main world programs [H] InterpolationShader.cs:33-46; VertexFunction.cs:5, 9; LightLevel.cs:20; SectorColorMap.cs:5-7.

**Fragment shader per pixel** [H] Core/Render/OpenGL/Renderers/Legacy/World/Shader/FragFunction.cs:181-238, 381; LightLevel.cs:44-62:
- One `texture(sampler2DArray, vec3(uv, layer))`, using a clamped UV when gap correction is on.
- Doom light diminishing: `lightIndex = min(2560/|dist2D|, 47)`, `startMap = (30 − lightNum)·2`, giving 32 bands or smooth (`lightMode`). This **emulates the software COLORMAP light table in-shader**.
- True colour: `rgb *= lightLevel`, or a brightmap fetch (`useBrightmaps`, second array sample).
- Palette mode (`window.colormode = Palette`, default TrueColor [H] Core/Util/Configs/Components/ConfigWindow.cs:105): textures store the palette index in R and a sentinel alpha of 254/255. The shader computes `texIndex = lightColorIndex·256 + colormap·(256·34·14) + palette·(256·34) + index` and does a `texelFetch` from the RGBA32F colormap TBO, with an optional brightmap fetch to lower the light index [H] FragFunction.cs:108-171.
- Sector RGB multiply, exponential sector fog `1 − exp(−density·dist3D)`, invulnerability grey (or a colormap emulation with 2 TBO fetches), `pow(rgb, 1/gamma)` per pixel, alpha discard [H] FragFunction.cs:205-223, 381-429.

**On the CPU per surface:** UVs including scroll offsets (current and previous), wall endpoints with the vertex-gap push (`render.pixelgapcorrection`, 0.01 units, [H] Core/Util/Constants.cs:45; StaticCacheGeometryRenderer.cs:151-157), fake contrast (`GetLightLevelAdd`, [H] GeometryRenderer.cs:949-971), and per-wall light (`GetWallLightLevel`, [H] GeometryRenderer.cs:2375-2381). All of these are **baked into vertices**. Changing a side's own light level or offsets at runtime needs a geometry rebuild (they are inputs to `SetWallVertices`). Changing the sector light does not.

**Inference:** the fragment cost is low by modern standards: 1 texture sample, about 15 ALU ops, one `exp`, one `pow`, plus one TBO fetch in palette mode. Vertex cost is moderate: 2–3 TBO fetches and 2 mat4 multiplies. For VR the dominant costs are the extra `mvpNoPitch` mat4 multiply and transforming the entire static map per eye.

---

## 6. Sector lighting, colormaps and fog: how changes reach the GPU

### 6.1 Buffer layout

Six `GLBufferTextureStorage` TBOs (sector light, sector colormap, sector fog, colormap, map line data, line heights; the last three may be null) are bound every frame [H] Core/Render/Renderer.World.cs:37-42; Renderer.cs:638-666; Core/Render/BindTextures.cs:8-21. Index layout [H] Core/Util/Constants.cs:364-380:
- `0` = dark, `1` = fullbright, `2..33` = 32 fixed colormap light values.
- `SectorIndexStart = 34`.
- Each sector then occupies `BufferSize = 3` consecutive slots: Floor, Ceiling, Wall.

`GetLightBufferIndex` returns `sector.Id·3 + 34 + offset`. It uses `TransferFloorLightSector`/`TransferCeilingLightSector` for flats, so Boom light transfers are resolved by **index aliasing at triangulation time** [H] Renderer.World.cs:63-72.

| TBO | Format | Size (S = sectors, L = lines) | Citation |
|---|---|---|---|
| Sector light | R8UI | 34 + 3S bytes | [H] Renderer.World.cs:100, 181-185 |
| Sector colormap | RGBA32F | (34 + 3S) × 16 B | [H] Renderer.World.cs:101 |
| Sector fog | RGBA32F | (34 + 3S) × 16 B | [H] Renderer.World.cs:102 |
| Colormap (palette mode / invuln emulation only) | RGBA32F | 256 × 34 × 14 × 16 B = **1,949,696 B per colormap** | [H] Core/Render/ColorMapBuffer.cs:9-10; Core/Graphics/Palettes/Colormap.cs:14-15; Palette.cs:20 |
| Map line data, line heights (vanilla render only) | RGBA32F | 16 L B each | [H] Renderer.World.cs:118-154, 308-336 |

Example: with S = 5,000, the light TBO is 15 KB and the colour and fog TBOs are 240 KB each.

### 6.2 Update mechanism (dirty lists, persistent mapping)

World events `SectorLightChanged`, `SectorColorMapChanged`, `SectorFogColorChanged`, `SectorMove` and `SectorMoveComplete` add the sector to a `SectorUpdates` set [H] Renderer.World.cs:92-96, 386-409. `SectorUpdates` deduplicates with a per-sector stamp and a frame counter [H] Core/Render/SectorUpdates.cs:27-43. Once per frame `UpdateBuffers()` walks the lists and writes through a mapped pointer [H] Renderer.World.cs:411-527:

```csharp
var lightBuffer = m_sectorLightsBuffer.GetMappedBufferAndBind();
...
var index = sector.Id * LightBuffer.BufferSize + LightBuffer.SectorIndexStart;
lightData[index + LightBuffer.FloorOffset] = lightLevel;
lightData[index + LightBuffer.CeilingOffset] = lightLevel;
lightData[index + LightBuffer.WallOffset] = lightLevel;
```
[H] Renderer.World.cs:428-436, 456-463

A light change also rewrites the fog entry, because fog density derives from the light level [H] Renderer.World.cs:442-453. When `ARB_buffer_storage` is present, the TBO is created with `glBufferStorage(MAP_WRITE|MAP_PERSISTENT)` and mapped **once** with `WRITE|UNSYNCHRONIZED|PERSISTENT`. The pointer is then cached [H] GLBufferTexture.cs:34-37, 49-54; GLBufferTextureStorage.cs:36-55. Without it, each update does `glMapBufferRange` (still with `MAP_UNSYNCHRONIZED_BIT`) → write → `glUnmapBuffer` [H] GLBufferTexture.cs:53; GLBufferTextureStorage.cs:53-70.

**Cost per light flicker:** 3 bytes plus 48 bytes of fog. No vertex is touched. CPU cost is O(changed sectors).

**Correctness observations (code facts, consequences are Inference):**
1. `GLMappedBuffer` maps `length = data.Length`. That is an *element* count, but `glMapBufferRange` takes *bytes* [H] Core/Render/OpenGL/Util/GLMappedBuffer.cs:17. For the float TBOs (colormap, sector colour, sector fog, map line data, line heights) only the first quarter of the buffer is formally mapped, and writes go beyond that range. This works only because drivers map the whole allocation.
2. The persistent mapping has no `MAP_COHERENT_BIT`, no explicit flush, and no `glMemoryBarrier(CLIENT_MAPPED_BUFFER_BARRIER_BIT)`. The only `MemoryBarrier` call is `FramebufferBarrierBit` [H] LegacyWorldRenderer.cs:609-610 (grep). Writes are unsynchronized against in-flight frames. **Inference:** a light change can tear across frames in flight. That is harmless visually, but a Vulkan port must double-buffer or use per-frame offsets.

### 6.3 Colormaps

- Per-sector Boom colormaps in palette mode go into `sectorColormapTexture` as `(index, −1, −1)` (`SectorRgb` colormaps stay an RGB multiplier even in palette mode). In true colour they are stored as an RGB multiplier [H] Renderer.World.cs:264-306.
- The view-dependent colormap (transfer heights, water, 3D-floor light) is a **uniform** (`colormapIndex`, `colorMix`) computed per frame from the viewer's sector [H] Renderer.cs:401-458.
- `m_sectorColor` and `m_sectorFog` are global flags that turn the TBO fetches on in the shader only if any sector uses colour or fog [H] Renderer.World.cs:254-255, 269-270; SectorColorMap.cs:19-27.

---

## 7. CPU cost scaling: map size vs visible set

| Work | Scales with | Frequency | Evidence |
|---|---|---|---|
| Static geometry submission (CPU) | # texture groups × geometry types | per frame | [H] StaticCacheGeometryRenderer.cs:859-896 |
| Static geometry vertex shading (GPU) | **total map geometry** (no culling, zeroed slots included) | per frame | same |
| Uniform setup | # program switches (~26 glUniform each) | per frame | [H] LegacyWorldRenderer.cs:833-911 |
| TBO dirty updates | # sectors with events since last frame | per frame | [H] Renderer.World.cs:411-527 |
| `IterateBlockmap` (static mode) | blocks inside a 2·MaxDistance square (6000 → about 24×24 blocks of 512 units), dynamic sector/side lists in them, and **all entities in them** (half-plane test only) | per tic | [H] LegacyWorldRenderer.cs:145-178 |
| Dynamic re-triangulation | moving or scrolling surfaces inside range (all subsectors of a scrolling flat) | per tic | [H] GeometryRenderer.cs:1785-1908 |
| BSP traversal (BSP mode) | visible set (view clipper) plus near-plane test; every visible surface copied into dynamic VBOs | per tic | [H] LegacyWorldRenderer.Bsp.cs:17-71 |
| Static patch on door/lift start and stop | lines of the moving sector | per event | [H] StaticCacheGeometryRenderer.cs:1029-1095, 1222-1252 |
| Static VBO full re-upload | size of one texture's VBO | when a surface grows or a new texture appears | [H] StaticCacheGeometryRenderer.cs:1555-1568 |
| Map load: triangulate everything, build arrays, fill TBOs | map size | once | [H] StaticCacheGeometryRenderer.cs:103-172 |

**Inference:** for VR the static design is very attractive on the CPU side, because CPU work per eye is close to constant. Its weak point is GPU vertex work that ignores the view and is paid twice per frame in stereo. On "slaughter/megawad" maps with 100k+ sides, this becomes the main vertex load. Helion counters it only by switching to BSP mode (Adaptive), which moves the cost back to the CPU.

**Inference (VR hazard):** visibility in both modes is decided at tic rate from the tic-time view direction: near-plane culling of BSP boxes [H] LegacyWorldRenderer.Bsp.cs:78-79, and the block half-plane test [H] LegacyWorldRenderer.cs:160-161, 198-215. Both are switched off when |pitch| > 45°. Only the block (and entity) half-plane test uses the 32-unit pull-back [H] LegacyWorldRenderer.cs:276-283, 784-786; the BSP test uses the near plane of the `MvpNoPitch` frustum at the camera [H] LegacyWorldRenderer.Bsp.cs:19. A head turn of more than (90° − hFOV/2) inside one 28.6 ms tic would expose culled BSP geometry or dynamic sectors. That is about 35° at a 110° per-eye FOV, which a fast VR head turn can exceed. For VR the cull must use the union of both eye frusta, widened by the head-motion bound, or be redone per frame.

---

## 8. Absent / not done (proven by search)

The searches below were run over `Core` and `Client` (`grep -rn "<pattern>" --include=*.cs`):
- **Multi-draw / indirect:** `MultiDraw`, `DrawArraysIndirect`, `DrawIndirect`, `BaseInstance` → 0 hits.
- **Index buffers:** `DrawElements`, `ElementArrayBuffer` → 0 hits. World geometry is entirely non-indexed triangle lists.
- **UBOs / SSBOs:** `UniformBuffer`, `ShaderStorage` → 0. Every uniform is a separate `glUniform*`.
- **DSA buffers:** `NamedBuffer`, `CreateBuffers` → 0. `CreateVertexArrays` exists only for `VertexArrayType.Modern`, and `VertexPipeline` always passes `Legacy` [H] Core/Render/OpenGL/Vertex/VertexArrayObject.cs:19-24; VertexPipeline.cs:27.
- **Bindless textures:** `BindlessTexture`, `GetTextureHandle` → 0.
- **Sync objects:** `FenceSync`, `ClientWaitSync` → 0. There is no CPU/GPU synchronization for persistent maps.
- **Compute shaders:** `ComputeShader` → 0.
- **Persistent-mapped vertex buffers:** `BufferStorage` appears only in `GLBufferTexture` (the TBOs) [H] GLBufferTexture.cs:34-35.
- **Stereo / multiview / layered rendering:** `MultiviewOVR`, `OVR_multiview`, `ViewportArray`, `gl_ViewportIndex`, `gl_Layer` → 0.
- **GPU occlusion queries / conditional render:** `SamplesPassed`, `ConditionalRender` → 0. `GLQuery` exists only for elapsed-time profiling.
- **Instancing for world geometry:** `DrawArraysInstanced` is called only by `RenderData` (entities, quad strip) [H] Core/Render/OpenGL/Renderers/Legacy/World/Data/RenderData.cs:63.
- **Frustum/portal culling of the static cache:** `RenderGeometry` draws `[0, Count)` of every VBO [H] StaticCacheGeometryRenderer.cs:859-896; VertexBufferObject.cs:14-21.
- **`invariant gl_Position`:** absent (only comments match `invariant`).
- **Multithreaded scene building:** world traversal and vertex generation run on the render thread. The only extra thread in this area is the BSP-timing heuristic [H] AutomapMarker.cs:70.

---

## 9. Perf-relevant settings

| Setting | Default | Effect | Citation |
|---|---|---|---|
| `render.mode` (Static / Bsp / Adaptive) | `Static` | Static cache + per-tic dynamic set vs full per-tic BSP | [H] ConfigRender.cs:93; LegacyWorldRenderer.cs:300 |
| `render.adaptive.timethreshold` | 2200 µs | Adaptive switch threshold | [H] ConfigRender.cs:243 |
| `render.adaptive.hysteresispercent` | 0.05 | Switch hysteresis | [H] ConfigRender.cs:246 |
| `render.adaptive.segthreshold` | 8000 | Above this seg count, never use BSP | [H] ConfigRender.cs:249; LegacyWorldRenderer.BspHeuristics.cs:78-79 |
| `render.adaptive.timewindow` / `switchcount` | 10 / 3 | Smoothing | [H] ConfigRender.cs:252, 255 |
| `render.maxdistance` | 0 (→ 6000) | Range of dynamic blockmap iteration and entities | [H] ConfigRender.cs:136; Constants.cs:432 |
| `render.vanillarender` | false | Adds plane-clip and wall-clip passes that redraw world geometry (2 extra passes, MRT) | [H] ConfigRender.cs:180; LegacyWorldRenderer.cs:410-411, 450-584 |
| `render.downscalevanillarendersamplebuffer` | 1 | Resolution of those passes | [H] ConfigRender.cs:184 |
| `render.pixelgapcorrection` | true | Vertex push at load plus UV clamp per pixel | [H] ConfigRender.cs:208; VertexFunction.cs:14-38 |
| `render.texturetransparency` | true | Translucent (line alpha < 1) walls via OIT | [H] ConfigRender.cs:233; GeometryRenderer.cs:794-795 |
| `render.postprocessingeffects` | true | Required for the FBO and therefore for reversed-Z | [H] ConfigRender.cs:237; Renderer.cs:266-278 |
| `window.colormode` | TrueColor | Palette mode adds the colormap TBO fetch per pixel | [H] ConfigWindow.cs:105; FragFunction.cs:108-164 |
| `render.lightmode` | Smooth | Banded or smooth diminishing | [H] ConfigRender.cs:143 |
| `render.contrastmode` | Vanilla | Fake contrast baked into vertices | [H] ConfigRender.cs:200; GeometryRenderer.cs:949-971 |
| `developer.render.lock` | false | Freezes traversal (debug) | [H] LegacyWorldRenderer.cs:310-314 |
| `developer.usereversedz` / `reversedz` | false / false | Overrides the automatic reversed-Z choice | [H] Core/Util/Configs/Components/ConfigDeveloper.cs:32-35; Renderer.cs:266-278 |

---

## 10. Transfer notes for UZDoom VR

**UZDoom counterpart, briefly:**
- UZDoom walks the BSP **every frame and per eye**: `for (eye_ix...) { StartDrawInfo; SetupView; ProcessScene }` [U] src/rendering/hwrenderer/hw_entrypoint.cpp:127-188. The walk is optionally multithreaded (`gl_multithread`, default true) [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:43.
- Each visible wall allocates fresh vertices from a per-frame streaming buffer (`AllocVertices`, atomic bump allocator, 2M-vertex limit) [U] src/rendering/hwrenderer/scene/hw_walls_vertex.cpp:280-288; src/common/rendering/hwrenderer/data/flatvertices.cpp:138-148.
- Each wall then gets its own `SetColor`/`SetFog` and its own `state.Draw(DT_TriangleFan, ...)` [U] src/rendering/hwrenderer/scene/hw_walls.cpp:95, 290.
- Flats are indexed and persistent. Plane Z is patched in place when heights change (`CheckPlanes` → `UpdatePlaneVertices` → `Upload(range)`) [U] src/rendering/hwrenderer/hw_vertexbuilder.cpp:406-421, 441-453.

In short, UZDoom is "per-frame, per-eye, per-surface draws with a persistent flat VBO". Helion is "per-tic visibility + whole-map static VBOs + per-texture batches".

| # | Helion mechanism | UZDoom counterpart | Grade | Stereo/Vulkan notes |
|---|---|---|---|---|
| 1 | **Tic-rate scene build, frame-rate redraw** with GPU interpolation (`prevPos/pos`, `timeFrac`) | Full scene build per frame per eye (hw_entrypoint.cpp:129-163) | **Rework** | The biggest CPU win for stereo 90–120 Hz: build the draw lists once per tic (or once per frame for both eyes) and replay them per eye. It requires interpolating plane heights and scroll UVs in the shader, as Helion does, and moving UZDoom's per-frame `CheckUpdate` of plane Z to the GPU. Cull with the union of both eye frusta plus a head-motion margin (§7 hazard). |
| 2 | **Whole-map static VBOs per (type, texture-array)**, one `glDrawArrays` each | Per-wall streamed vertices + per-wall draws | **Rebuild** (wall submission) | In Vulkan, put all static geometry in one device-local buffer sorted by material and draw with `vkCmdDrawIndexedIndirect` / `DrawIndirectCount`. Use a **GPU culling compute pass** over per-surface or per-sector meshlets (Helion has no culling: about 2× wasted vertex work in stereo). Multiview (`VK_KHR_multiview`) makes one submission serve both eyes. |
| 3 | Event-driven static patching: zero slot on move start, rebuild on move complete, free-list allocator, `glBufferSubData` | Flat VBO plane-Z patch (`UpdatePlaneVertices`) | **Port** (concept) | Keep the slot/free-list design. In Vulkan, do patches through a staging ring + `vkCmdCopyBuffer`, or through host-visible memory with per-frame-in-flight copies. Better: store `sectorPlaneIndex` per vertex and fetch Z from a per-sector plane SSBO (current + previous Z), so doors and lifts never touch vertex data and never fall back to the dynamic path. |
| 4 | Dynamic path: per-texture CPU arrays, `glBufferData` orphan per tic, 1024 pooled VAO/VBO | `FFlatVertexBuffer` streaming + per-wall draws | **Rebuild** | Replace with one persistently mapped ring buffer per frame in flight and per-texture offset ranges. If #3 (plane SSBO) and #6 (scroll SSBO) are adopted, almost nothing remains dynamic. |
| 5 | **Sector-indexed lighting TBOs** (light R8UI, colour/fog RGBA32F, 3 slots per sector, dirty-list updates) | Per-wall `SetColor`/`SetFog` → per-draw uniform/stream data (hw_walls.cpp:290) | **Rework** | Move UZDoom light, colormap and fog into a per-sector SSBO indexed from the vertex. Draws then stop depending on light, which enables batching (#2). Fix Helion's two flaws when porting: byte-correct map length, and per-frame-in-flight copies (or a coherent ring), not unsynchronized writes into a buffer the GPU is reading. |
| 6 | Scroll and texture offsets baked into vertices; scrolling surfaces re-triangulated per tic | UZDoom computes UVs on CPU per frame per wall | **Rework** | Add per-side and per-plane scroll offsets (current and previous) to an SSBO and add them in the vertex shader. That removes scrolling surfaces from the dynamic set entirely. |
| 7 | Texture arrays grouped by exact dimension; animated and brightmapped textures standalone; animation by rebinding | UZDoom material system, per-draw texture binds | **Rework** | Vulkan: use descriptor indexing (a bindless array of images) so the material is a per-vertex or per-draw index. Then all opaque world geometry goes in **one** indirect draw per pipeline. Animation becomes an indirection table in an SSBO (as Helion's `m_translations`). |
| 8 | Packed float attributes (`floatBitsToInt`) | n/a | **Port** | Use `R32_UINT` vertex formats in Vulkan. Do not rely on NaN passthrough. |
| 9 | Shader light diminishing emulation (2560/dist, 32 bands, palette TBO) | UZDoom software-light-mode shaders | **Port** | Both are per-pixel and cheap. In stereo, compute `dist2D` from the **centre eye**, not per eye, or the light bands will differ between eyes (**Inference**). Take `dist3D` straight from `gl_Position` (it is the same `mvp·pos`); the remaining extra mat4 multiply is `mvpNoPitch` for `dist2D`. |
| 10 | Stencil-masked sky per sky texture with a full stencil clear each | UZDoom sky portals | **Rework** | Per-texture `glClear(STENCIL)` is a full-screen clear per sky. In Vulkan, write a sky-id into stencil once and draw each sky with compare-equal, or draw sky geometry directly with a cubemap/sphere-lookup shader (no stencil). With multiview this is natural. |
| 11 | OIT (weighted blended) for translucent walls | UZDoom sorted translucent lists | **Port** | Order-independent, so it suits stereo (no per-eye sort). Costs 2 extra targets per eye. |
| 12 | Adaptive BSP vs static switching (background timing thread) | `gl_multithread` BSP | **Rebuild** | With GPU culling (#2), a CPU BSP walk is needed only for gameplay-visible sets (automap, sound). Do not port. |

---

## 11. Open questions

1. **Scrolling-surface double draw:** I could not confirm visually whether the static and dynamic copies z-fight. The code draws both, at equal depth from different programs, with no `invariant` (§2.1).
2. **Real draw counts and VBO sizes** for representative maps (e.g. MAP01 vs a 30k-line map) were not measured. Building and running Helion was out of scope; `DrawCounts` in `RenderProfiler` would give them directly.
3. **Whether `RenderOffsets.Gametick` is reset each tic** (so scrolling flats stop re-triangulating once scrolling stops): not traced into the scroll specials.
4. **`GeometryRenderer.Sector3D.cs` (794 lines) and `StaticCacheGeometryRenderer.Sector3D.cs` (99 lines)** were only skimmed. 3D-floor wall slicing and its interaction with the free list (vertex count growth) deserve a dedicated pass.
5. **Portal and flood-fill internals** (`PortalRenderer`, `FloodFillRenderer`, 711 lines) were covered only at the draw-call level.
6. Whether drivers in practice honour the under-sized `glMapBufferRange` on float TBOs (§6.2 #1). This is AMD-sensitive; the in-code comment in Client.cs mentions AMD Vega persistent-map quirks.
7. Measured line counts for this area: `Core/Render/*.cs` 1,812 lines; `Renderers/Legacy/World/**` 13,883 lines (12,044 excluding Entities/Automap); `GeometryRenderer.cs` 2,409; `StaticCacheGeometryRenderer.cs` 1,602; `LegacyWorldRenderer*.cs` 1,108; `WorldTriangulator.cs` 583; GL buffer/vertex plumbing 980.
