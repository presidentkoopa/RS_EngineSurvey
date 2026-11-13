# H6 — Helion map loading, BSP/geometry build, load-time precomputation, and feature coverage vs UZDoom 5.0

Helion does not use the node lumps stored in a WAD. On every new map it runs an in-process C# port of ZDBSP (`zdbspSharp`) in GL-only mode, writes the result to a temp WAD, loads that WAD again, and reads only GL nodes (v2/v5) or the GL variants of ZNODES (XGL*/ZGL*). Its own ~3k-line BSP builder (`Core/Maps/Bsp/BspBuilder.cs` plus `States/` and `Geometry/`) has no callers. From the result it builds a flat, struct-array BSP (`BspNodeCompact[]`, `SubsectorSegment[]`) and adds several precomputed lookup tables. Two have no UZDoom counterpart: a 16-unit grid that maps a point to its starting BSP node, and the monster-closet classification. The others have UZDoom equivalents: "islands" (connected subsector components, which UZDoom computes as `mapsection`), flood/"bad subsector" sets for render hacks (UZDoom: `hw_renderhacks`), and per-sector line arrays (UZDoom: `GroupLines`). The feature set is narrow next to UZDoom 5.0:
- No ZScript, portals, slopes, polyobjects, dynamic lights, models, voxels, custom shaders, netplay, or Heretic/Hexen/Strife game data.
- Of 359 DECORATE action-function names Helion maps, 238 point to empty stubs (re-counted: 121 have a body).
- ACS covers only the map's own BEHAVIOR lump; library modules are refused.

**Inference:** much of Helion's speed comes from not supporting the expensive features. Its genuinely transferable ideas are the BSP point grid, the monster-closet classification, and static-vs-dynamic geometry classification. Map-wide islands and their render cull already exist in UZDoom as map sections. Helion's scripting and load pipeline are not transferable.

## Load pipeline: what is read, in which order

### 1. Locating the map (lazy)
`ArchiveCollection.FindMap` walks archives newest-first. It handles `maps/<name>.wad` embedded in a PK3 by extracting it to a temp file, and otherwise iterates WAD map markers, `[H] Core/Resources/Archives/Collection/ArchiveCollection.cs:174-220`. `IMap.Read` dispatches on `MapEntryCollection.MapType`: UDMF if `TEXTMAP` is present, Hexen if `BEHAVIOR` is present, else Doom, `[H] Core/Maps/IMap.cs:39-56`, `[H] Core/Maps/MapEntryCollection.cs:57-61`. With `FindMapOptions.Default` the map object is created but `LoadData()` is not called, so nothing is parsed yet (`(options & FindMapOptions.LoadMapData) != 0`, `[H] Core/Resources/Archives/Collection/ArchiveCollection.cs:206`).

The lump iterator recognises THINGS…GL_PVS, `[H] Core/Resources/Archives/Iterator/ArchiveMapiterator.cs:16-48`. The readers ignore BLOCKMAP, NODES/SEGS/SSECTORS, DIALOGUE, GL_PVS and SCRIPTS. Proof: `DoomMap.LoadData` reads only vertices, sectors, sides, lines, things, GL components and REJECT, `[H] Core/Maps/Doom/DoomMap.cs:63-102`. A grep for `Blockmap`, `Dialogue`, `GLPVS`, `Scripts` and the `Nodes`/`Segments`/`Subsectors` properties of `MapEntryCollection` over `Core` and `Client` finds only their declarations and the iterator's lookup table.

### 2. Node building on every map: zdbspSharp → temp WAD → reload
`Client.LoadMap` runs on a worker thread (`Task.Run`, `[H] Client/Client.Commands.cs:914-920`). For any map that is not the previously loaded one it unconditionally calls `m_zdbsp.RunZdbsp(map.ArchivePath, …)`. It then drops the original map and uses the compiled one, copying only `CompatibilityDefinition` and `Behavior` across:

```csharp
// [H] Client/Client.Commands.cs:944-963
if (!sameMap)
{
    var mapCompat = map.CompatibilityDefinition;
    var behavior = map.Behavior;
    if (!m_zdbsp.RunZdbsp(map.ArchivePath, mapInfoDef.MapName, out var compiledMap))
    {
        Log.Error("Failed to run zdbsp.");
        return result;
    }

    // Large UDMF maps don't seem to free the memory without this.
    map.ClearAll();
    map.Reject = null;
    map = compiledMap;

    if (map != null)
    {
        map.CompatibilityDefinition = mapCompat;
        map.Behavior = behavior;
    }
}
```

`RunZdbsp` opens the source archive file from disk, runs `FProcessor` with `GLOnly = true, BuildGLNodes = true, ConformNodes = false`, and writes a temp WAD. It then builds a new `ArchiveCollection` over that temp file and calls `FindMap(..., FindMapOptions.LoadMapData)`, which is where TEXTMAP and the binary lumps are finally parsed. The temp file is then deleted, `[H] Core/Maps/Bsp/Zdbsp/Zdbsp.cs:20-85`. zdbspSharp is a NuGet binary (`[H] Core/Core.csproj:35`, version 1.0.9). Its source is not in the repo or the local package cache (`find / -iname "*zdbspSharp*"` found nothing).

**Inference:**
- Every new-map load pays a full node build, even when the WAD ships perfectly good ZNODES or GL nodes.
- A UDMF TEXTMAP is tokenised twice: once inside zdbspSharp and once by Helion's `UdmfMap.Parse`.
- There is no on-disk node cache. UZDoom loads the map's own nodes and builds only when `gennodes`/`ForceNodeBuild` is set or the node lumps are unusable, `[U] src/maploader/maploader.cpp:2932`, `[U] src/maploader/maploader.cpp:3031-3130`. When the shipped nodes are not GL-friendly, `CheckNodes` moves them to `gamenodes` and then loads cached/GWA GL nodes or builds new ones. Builds that take at least `gl_cachetime` (0.6 s) are cached when `gl_cachenodes` is on (default true), `[U] src/maploader/glnodes.cpp:921-984`, `[U] src/g_cvars.cpp:39-40`.
- The one cache is in memory and covers only the same map: `sameMap` reuses `m_lastLoadedMap` and skips zdbsp, `[H] Client/Client.Commands.cs:935-936`.

### 3. Node formats accepted
`GLComponents.Read` accepts either a complete GL_VERT/GL_SEGS/GL_SSECT/GL_NODES set, or a ZNODES lump, `[H] Core/Maps/Components/GL/GLComponents.cs:52-76`.
- **GL nodes:** only the `gNd2` and `gNd5` vertex headers; anything else throws, `[H] Core/Maps/Components/GL/GLComponents.cs:86-91`. Node splitters and child bboxes are read as `Int16`, so GL nodes cannot carry fractional splitters, `[H] Core/Maps/Components/GL/GLComponents.cs:166-172`, `[H] Core/Maps/Components/GL/GLComponents.cs:224-229`. V5 node length is found by scanning back over trailing zeros, `[H] Core/Maps/Components/GL/GLComponents.cs:182-222`.
- **ZNODES:** only the GL forms `XGLN/XGL2/XGL3` and their zlib forms `ZGLN/ZGL2/ZGL3`. Non-GL `XNOD/ZNOD` fail `ReadHeader`, `[H] Core/Maps/Components/ZNodes/ZNodesDefinition.cs:67-99`. XGL3 fractional splitters are supported, `[H] Core/Maps/Components/ZNodes/ZNodesDefinition.cs:101-112`. For the non-fractional forms (XGLN/XGL2/ZGLN/ZGL2), the `Int16` splitter values are passed through `MathHelper.FromFixed` (÷65536), `[H] Core/Maps/Components/ZNodes/ZNodesDefinition.cs:113-120`, `[H] Core/Util/MathHelper.cs:320-323`. Those formats store whole map units, so their splitters would be scaled wrongly. **Inference:** this path is only safe if zdbspSharp never emits those forms; not verified (see open questions).
- **Vanilla NODES/SEGS/SSECTORS:** never read; the `Nodes` entry has no consumer.
- **No GL data at all:** `GeometryBuilder.CreateBspBuilder` logs "Unable to find GL nodes" and the load fails, `[H] Core/World/Geometry/Builder/GeometryBuilder.cs:76-83`. That is why the zdbsp pass is mandatory.
- **Helion's own builder:** `Core/Maps/Bsp/BspBuilder.cs` (partitioner, miniseg, convex checker) is unused. `grep -rn "\bBspBuilder\b"` in Core/Client/Tests/BenchmarkSuite finds only its own definition. Only `Node/` (`BspNode`, `SubsectorEdge`) is shared with the live `GLBspBuilder` path.

### 4. Map format readers
- **Doom:** fixed records (line 14 B, sector 26 B, side 30 B, thing 10 B, vertex 4 B), `[H] Core/Maps/Doom/DoomMap.cs:23-29`. Line types go through `VanillaLineSpecTranslator.Translate` (1,853 lines) into ZDoom special numbering, `[H] Core/World/Geometry/Builder/DoomGeometryBuilder.cs:183-195`.
- **Hexen:** 16-byte lines and 20-byte things with args/TID, `[H] Core/Maps/Hexen/HexenMap.cs:23-24`. BEHAVIOR is read raw, `[H] Core/Maps/Hexen/HexenMap.cs:55`.
- **UDMF:** hand-written streaming parser keyed on field names, `[H] Core/Maps/Udmf/UdmfMap.cs:179-241`.
  - Accepted namespaces: `doom`, `dsda`, `eternity`, `zdoom`, `helion`. Anything else throws "Unsupported udmf namespace", `[H] Core/Maps/Udmf/UdmfMap.cs:243-265`. So `zdoomtranslated`, `hexen`, `heretic`, `strife` and `vavoom` maps fail to load. UZDoom accepts all of these and falls back to a game default for unknown namespaces, `[U] src/maploader/udmf.cpp:2494-2560`.
  - Vertices parse only `x`/`y` (no `zfloor`/`zceiling`), `[H] Core/Maps/Udmf/UdmfMap.cs:720-734`.
  - Sector `heightfloor`/`heightceiling` are parsed as ints and cast to `short`, `[H] Core/Maps/Udmf/UdmfMap.cs:475-478`.
  - No `floorplane_*`, `portal`, terrain or colormap keys; the full sector key set is `[H] Core/Maps/Udmf/UdmfMap.cs:473-554`.
  - UZDoom parses `floorplane_a` and vertex `zfloor`, `[U] src/maploader/udmf.cpp:1922-1924`, `[U] src/maploader/udmf.cpp:2358-2360`.

### 5. Geometry build (`GeometryBuilder` → `MapGeometry`)
`GeometryBuilder.Create` dispatches to the Doom/Hexen/UDMF builders, which follow the same pattern, `[H] Core/World/Geometry/Builder/GeometryBuilder.cs:37-74`:
1. `PopulateSectorData`: one `Sector` plus two `SectorPlane` per map sector. Flat textures are resolved to integer handles here, `[H] Core/World/Geometry/Builder/DoomGeometryBuilder.cs:36-57`.
2. `PopulateLineData`: one `Line` and one or two `Side` objects, each `Side` with three `Wall` objects. Wall textures are resolved to handles. Zero-length lines are pruned, `[H] Core/World/Geometry/Builder/DoomGeometryBuilder.cs:166-197`. Specials and Boom generalised types are translated, and 242/SetSectorColormap side colormaps are resolved at load, `[H] Core/World/Geometry/Builder/DoomGeometryBuilder.cs:92-111`.
3. The BSP is built (step 6), then `new MapGeometry(builder, bspTree)`. Tag lookup and line-ID lookup are hashed once into `Dictionary<int, IList<…>>`, including UDMF `moreids`, `[H] Core/World/Geometry/MapGeometry.cs:46-59`, `[H] Core/World/Geometry/MapGeometry.cs:195-234`.

### 6. Compact BSP (`CompactBspTree`)
`GLBspBuilder.Build` converts GL segs, subsectors and nodes into an object graph (`SubsectorEdge`, `BspNode`). It patches subsectors with fewer than 3 segs into degenerate triangles and flags them `Malformed`, `[H] Core/Maps/Bsp/Builder/GLBSP/GLBspBuilder.cs:39-147`. `CompactBspTree` then flattens it by post-order recursion, `[H] Core/World/Bsp/CompactBspTree.cs:210-347`:
- `Segments: SubsectorSegment[]` (readonly struct: two `Vec2D` endpoints plus side/line/front/back sector ids) is laid out contiguously per subsector, `[H] Core/World/Geometry/Subsectors/SubsectorSegment.cs:9-29`.
- `PartnerSegmentSubsectors: int[]` runs parallel to `Segments` and stores the subsector on the other side of each seg, `[H] Core/World/Bsp/CompactBspTree.cs:291-293`. This is the precomputed subsector adjacency graph.
- `Subsectors: Subsector[]` holds class instances, each with an AABB computed from its edges and `SegIndex/SegCount` into the seg array, `[H] Core/World/Bsp/CompactBspTree.cs:228-268`.
- `Nodes: BspNodeCompact[]` holds sequential structs: `SplitStart`, `SplitDelta`, the node's own `Box2D` (union of its children), and `fixed uint Children[2]` with bit 31 marking a subsector. Root is `Nodes[^1]`, `[H] Core/World/Bsp/BspNodeCompact.cs:11-56`.

**Inference:** at 8-byte doubles that is 16+16+32+8 = 72 bytes per node, the same footprint as UZDoom's `node_t` on 64-bit. UZDoom stores fixed_t splitters, two float child bboxes, `len`, `nodenum` and pointers, `[U] src/gamedata/r_defs.h:1709-1733`.

Point location is a branch-light walk; the "on line" special cases reproduce vanilla `R_PointOnSide`:

```csharp
// [H] Core/World/Bsp/CompactBspTree.cs:117-130
public unsafe Subsector ToSubsector(uint nodeIndex, double x, double y)
{
    while (true)
    {
        ref var node = ref Nodes[nodeIndex];

        bool onRight = OnRightNode(x, y, node);
        int next = *(byte*)&onRight;
        nodeIndex = node.Children[next];

        if ((nodeIndex & BspNodeCompact.IsSubsectorBit) != 0)
            return Subsectors[(int)(nodeIndex & BspNodeCompact.SubsectorMask)];
    }
}
```

Gameplay point-location uses these GL nodes, not the map's original nodes. UZDoom keeps a separate `gamenodes` array for the original nodes when they are not GL nodes, `[U] src/maploader/glnodes.cpp:935-937`, `[U] src/maploader/maploader.cpp:3160-3161`.

### 7. World construction: precomputation inside `WorldBase` and `SinglePlayerWorld`
`WorldLayer.CreateWorldGeometry` builds the geometry, or reuses `LastMapGeometry` for the same map, then runs `geometry.ClassifyIslands()` once per new map, `[H] Core/Layer/Worlds/WorldLayer.cs:259-305`. The `WorldBase` constructor then does the following, `[H] Core/World/WorldBase.cs:231-337`:

| Step | What is built | Code |
|---|---|---|
| REJECT | Raw bytes kept as-is, size-checked; used in `CheckLineOfSight` | `[H] Core/World/WorldBase.cs:263-269`, `[H] Core/World/WorldBase.cs:2761`, `[H] Core/World/WorldBase.cs:2899-2907` |
| Physics blockmap | `new BlockMap(Lines, 128)`: own 128-unit grid built from lines, BLOCKMAP lump ignored; `BlockLine` structs (seg, block flags, front/back sector) sorted by block index, then line id (vanilla order) | `[H] Core/World/WorldBase.cs:412-427`, `[H] Core/World/Blockmap/BlockMap.cs:61-85`, `[H] Core/World/Blockmap/BlockLine.cs:8-28` |
| BSP grid | 16-unit grid of `uint` "start node" per cell (below), built right after the physics blockmap | `[H] Core/World/WorldBase.cs:424`, `[H] Core/World/WorldBase.BspBlockMap.cs:13-34` |
| Render blockmap | 512-unit grid of per-cell render-entity heads, lists of dynamic sector islands, and dynamic (scrolling) sides | `[H] Core/World/WorldBase.cs:429-439`, `[H] Core/World/Blockmap/BlockMap.cs:95-115` |
| `StructLines` | `DynamicArray<StructLine>` value copies of hot line data (segment, sector and plane refs, flag bits) | `[H] Core/World/WorldBase.cs:459-500`, `[H] Core/World/Geometry/Lines/StructLine.cs:23-88` |
| Sector→line adjacency | Exact-sized `sector.Lines`, `sector.LineIds`, `sector.MidTex3DLines`, counted then filled (two passes) | `[H] Core/World/WorldBase.cs:463-540` |
| 3D-midtex entities | Pre-allocated per midtex3d line | `[H] Core/World/WorldBase.cs:497-499` |
| 3D floors | `SpecialManager.InitSectors3D`: per-parent sorted `Sector3D[]`; global `WorldStatic.Sector3D` flag gates 3D-floor code | `[H] Core/World/WorldBase.cs:335-336`, `[H] Core/World/Special/SpecialManager.cs:732-773` |
| Monster/voodoo closets | `ClosetClassifier.Classify` when `Game.MonsterCloset` is on (same-map restarts use `ClassifySameMap`) | `[H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:134-138` |
| Static vs dynamic | `StaticDataApplier.DetermineStaticData` at `Start()`: flood-fill flags, scroll sides linked into the render blockmap | `[H] Core/World/WorldBase.cs:823`, `[H] Core/World/Static/StaticDataApplier.cs:14-42` |

The renderer then loads all level textures into GL texture arrays and builds the static vertex cache (`PreloadAllTextures`, `GLTextureArrayBuilder.BuildLevel`, `m_staticCacheGeometryRenderer.UpdateTo`), `[H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/GeometryRenderer.cs:201-247`. Static-geometry rendering belongs to the render sections; here it matters only as a load-time cost.

**BSP grid (16×16-unit cells → node).** `BspBlockmapBuilder.Resolve` recursively quarters the map rectangle. Each sub-rectangle descends the BSP while `ClassifyBoxSide` says the box is wholly on one side. If it reaches a subsector whose segs do not intersect the box, every cell gets that subsector index directly. Otherwise each single cell stores the last node whose splitter did not cut it (the parent of the first node whose splitter does), `[H] Core/World/BspBlockmapBuilder.cs:43-115`. At run time:

```csharp
// [H] Core/World/WorldBase.BspBlockMap.cs:36-49
public Subsector ToSubsector(double xPos, double yPos)
{
    int x = (int)(xPos - m_bspBlockmapOrigin.X) >> BspBlockShift;
    int y = (int)(yPos - m_bspBlockmapOrigin.Y) >> BspBlockShift;
    int blockIndex = y * m_bspBlockmapDimensions.Width + x;
    if (blockIndex < 0 || blockIndex >= m_bspBlockmapNodeIndices.Length)
        return BspTree.ToSubsector((uint)BspTree.Nodes.Length - 1, xPos, yPos);

    var startIndex = m_bspBlockmapNodeIndices[blockIndex];
    if ((startIndex & BspNodeCompact.IsSubsectorBit) != 0)
        return BspTree.Subsectors[startIndex & BspNodeCompact.SubsectorMask];

    return BspTree.ToSubsector(startIndex, xPos, yPos);
}
```

Consumers include every entity link/move (`PhysicsManager` try-move and center subsector), spawning, and the renderer's view-sector lookup, `[H] Core/World/Physics/PhysicsManager.cs:1399`, `[H] Core/World/Physics/PhysicsManager.cs:1626`, `[H] Core/World/Entities/EntityManager.cs:91`, `[H] Core/Render/Renderer.cs:875`. UZDoom's `PointInSubsector` walks from the head game node every time, `[U] src/playsim/p_maputl.cpp:1946-1962`.

**Inference:**
- For most cells the lookup is one load plus zero to three node tests instead of the full log₂(N) walk, typically 12–20 node tests on large maps. Each node test is a dependent load, so this is a real win in hitscan- and monster-heavy maps.
- Memory is 4 bytes per 256 map units². A 32k×32k map gives 2048×2048×4 = 16 MiB, which is acceptable on desktop VR hardware.
- Only the flat index is bounds-checked, not x and y separately. A point left or right of the grid wraps into a neighbouring row's cell. Such points lie outside the line bounding box, so this should only affect out-of-map positions.

**Same-map reuse.** On a restart of the same map, Helion reuses `LastMapGeometry`, `LastIslandGeometry`, `LastBlockMap`, BSP-grid arrays, `LastRenderBlockMap` and `LastStructLines`, calling `Reset()` on lines/sides/sectors, `[H] Core/Layer/Worlds/WorldLayer.cs:266-322`, `[H] Core/World/WorldBase.cs:412-457`. **Inference:** this makes death-restarts near-instant. It is a load-time optimisation only, with no per-frame effect.

## Geometry data layout and how render and playsim share it

- **Containers:** `MapGeometry.Lines`, `Sides` and `Sectors` are `List<T>` of classes, `[H] Core/World/Geometry/MapGeometry.cs:32-35`. UZDoom uses `TArray<sector_t/line_t/side_t>` of value structs, `[U] src/g_levellocals.h:491-501`. **Inference:** Helion's sector/line/side records are scattered heap objects, while UZDoom's are contiguous. Helion compensates only on its hottest paths (`StructLine[]`, `BlockLine[]`, `SubsectorSegment[]`, `BspNodeCompact[]`).
- **`Sector`** (sealed class), `[H] Core/World/Geometry/Sectors/Sector.cs:30-125`:
  - Identity, tags and plane references: `Id`, `Tag`, `MoreTags[]`, `Floor/Ceiling: SectorPlane`.
  - Adjacency: `Lines[]`, `LineIds[]`, `MidTex3DLines[]`, `Island`.
  - 3D floors: `Sectors3D[]`, `TaggedSectors3D[]`, `SectorPlanes3D[]`.
  - Thing and render-grid links: `Entities: LinkableList<Entity>`, `BlockmapNodes` (render-blockmap links).
  - Movement, lighting and environment: active move specials, `TransferHeights`, colormap, light/fog colours, sky strings and handles, friction, gravity.
  - Mixed render/playsim scratch: `RenderGametick`, `CheckCount`, `SoundValidationCount`, `Flood`, `MoveState`.
  - The sector bounding box is computed lazily from `Lines` and memoised, `[H] Core/World/Geometry/Sectors/Sector.cs:1089-1123`.
- **`SectorPlane`** (class) is always flat. The constructor sets `Plane = new PlaneD(0, 0, 1.0, -z)` and `SetZ` only moves D, `[H] Core/World/Geometry/Sectors/SectorPlane.cs:42-56`, `[H] Core/World/Geometry/Sectors/SectorPlane.cs:82-86`. The only other writes copy another flat plane (TransferHeights). It also carries render state: `Static: StaticGeometryData` (a reference into a GPU vertex buffer range), `SkyGeometry`, `RenderOffsets`, `LastRenderGametick`, `Dynamic` flags, `[H] Core/World/Geometry/Sectors/SectorPlane.cs:8-36`, `[H] Core/World/Static/StaticGeometryData.cs:5-10`.
- **`Line`** (sealed class): `Seg2D Segment`, `RenderSegStart/End`, `Front/Back: Side`, `Args`, `Flags`, `Special`, `Alpha`, `[H] Core/World/Geometry/Lines/Line.cs:20-65`. It also has two scratch counters, `BlockmapCount` and `PhysicsCount`, that are only reset (`[H] Core/World/Geometry/Lines/Line.cs:43-44`, `[H] Core/World/Geometry/Lines/Line.cs:120-121`); a grep finds no reader.
- **`Side`** (sealed class): `Sector`, `Upper/Middle/Lower: Wall`, optional `Fog` wall, `Offset`, `PartnerSide`, scroll data, colormaps, flood keys, invalidation counters and `LastRender*` ticks, `[H] Core/World/Geometry/Sides/Side.cs:31-80`.
- **`Wall`** (class): `TextureHandle` is a `virtual` property, overridden only by `SectorWall3D`. It also holds offset/scale floats and `Static` (GPU buffer range), `[H] Core/World/Geometry/Walls/Wall.cs:7-18`, `[H] Core/World/Geometry/Walls/SectorWall3D.cs:3-7`.
- **Sharing vs duplication.** The renderer reads the same `Sector/Side/Wall/SectorPlane` objects the playsim mutates, and stores its GPU-buffer bookkeeping inside them (`Static`, `SkyGeometry`, flood keys). There is no separate render scene, and Helion keeps no copies of the simulation state for rendering. The value-type copies are:
  - `StructLine[]`, used by the automap, the renderer's line-heights buffer, sound propagation and physics, `[H] Core/Render/Renderer.World.cs:135-149`, `[H] Core/World/WorldBase.cs:1001`, `[H] Core/World/Physics/PhysicsManager.cs:1852`.
  - `BlockLine[]` for physics.
  - `SubsectorSegment[]` for the BSP.
  - After load, `StructLine` changes only through `Update`, which can only set the automap-seen flag, and through `MarkSpecials`, which toggles `MarkAutomap`, `[H] Core/World/Geometry/Lines/StructLine.cs:90-94`, `[H] Core/World/Impl/SinglePlayer/MarkSpecials.cs:438-442`. **Inference:** this is safe because segment and sector references never change after load; polyobjects, which would move lines, do not exist. The `HasSpecial`/teleport/exit flag bits are not refreshed when a line special is changed or cleared at run time.

**Inference for VR:** render state embedded in simulation objects makes it impossible to render frame N while ticking N+1 on another thread without a snapshot. UZDoom has the same coupling (`sector_t`/`side_t` carry render data), so neither engine gives a clean render/sim split for free.

## "Islands": what they are and what they are used for

**Definition.** `IslandClassifier.Classify` does a DFS over subsectors using `PartnerSegmentSubsectors` (adjacency across any seg pair, both real two-sided lines and minisegs). Each connected component is an `Island` holding its subsectors, its line ids, and an AABB built from subsector boxes, `[H] Core/World/Geometry/Islands/IslandClassifier.cs:61-156`, `[H] Core/World/Geometry/Islands/Island.cs:16-36`. It runs twice:
- `Islands`: map-wide components. These are effectively "regions not connected by any two-sided line". Each `Subsector.IslandId` and `Sector.Island` is set, `[H] Core/World/Geometry/Islands/IslandClassifier.cs:123-132`.
- `SectorIslands[sectorId]`: for each sector, the connected pieces of that sector. The traversal only follows partners in the same sector, `[H] Core/World/Geometry/Islands/IslandClassifier.cs:42-59`, `[H] Core/World/Geometry/Islands/IslandClassifier.cs:151-152`. Each sector-island gets `ParentIsland` set, `[H] Core/World/Geometry/MapGeometry.cs:78-85`.

UZDoom has both levels.
- **Map-wide:** `MapSectionGenerator::DoSetMapSection` floods `subsector_t::mapsection` across `PartnerSeg` links, the same partner-seg flood Helion uses, `[U] src/maploader/renderinfo.cpp:77-102`, `[U] src/maploader/renderinfo.cpp:187-199`.
- **Per sector:** `CreateSections` groups subsectors by (render sector, mapsection), then splits each group into connected sections for flat triangulation, `[U] src/r_data/r_sections.cpp:139-200`, `[U] src/r_data/r_sections.h:131-151`.

The one Helion-only use is the closet logic below.

**Uses:**
1. **Flood (missing-texture) render hacks.** Malformed subsectors (fewer than 3 GL segs) are added to `BadSubsectors`, and their sector plus the smallest enclosing sector-island's sector go into `FloodSectors`, `[H] Core/World/Geometry/MapGeometry.cs:86-170`. If the subsector's own sector-island contains it, that whole island's subsectors also go into `BadSubsectors`. The renderer copies these into `sector.Flood` / `subsector.Flood`, `[H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/GeometryRenderer.cs:363-369`. Sector-plane sound sources also do not treat a listener standing in a `BadSubsector` as being inside the sector, `[H] Core/World/Geometry/Sectors/SectorSoundSource.cs:33-35`.
2. **Render culling of dynamic sectors.** Moving or scrolling sectors are linked into the 512-unit render blockmap *per sector-island box*, skipping closet islands, `[H] Core/World/Blockmap/BlockMap.cs:302-322`. Per frame the renderer:
   - dedupes islands with `BlockmapCount`;
   - **skips any sector-island whose parent map-island is not the viewer's island**;
   - distance-culls by island AABB, `[H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:187`, `[H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:223-246`.

   UZDoom does the same viewer-section cull. It sets `CurrentMapSections` from the view point's subsector and skips any subsector whose `mapsection` is not set there, `[U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:262-268`, `[U] src/rendering/hwrenderer/scene/hw_bsp.cpp:696-697`. Portals add their destination's section, `[U] src/rendering/hwrenderer/scene/hw_portal.cpp:668`, `[U] src/rendering/hwrenderer/scene/hw_portal.cpp:808`, `[U] src/rendering/hwrenderer/scene/hw_portal.cpp:915`.

   **Inference:** sectors in regions with no two-sided-line path to the camera cannot be visible without portals. Helion has no portals, so its cull is exact and needs no portal handling.
3. **Monster-closet AI throttling.** `ClosetClassifier.SetCloset` marks a map-island as a monster closet when all three hold, `[H] Core/World/Geometry/Islands/ClosetClassifier.cs:82-122`:
   - every line special in it is a teleport or plane scroller;
   - every entity in it counts as a kill;
   - it contains at least one monster.

   Non-friendly monsters in closets get `ClosetFlags.MonsterCloset`, `[H] Core/World/Geometry/Islands/ClosetClassifier.cs:42-50`. Definitions whose spawn loop is only `A_Look` (checked over at most 20 frames) are eligible, `[H] Core/World/Entities/Definition/Composer/DefinitionStateApplier.cs:123-140`. `A_Look` then diverts them to a synthetic 16-tic invisible frame running `A_ClosetLook`. That frame only checks the sector's sound target; there is no LOS or blockmap search. On alert, `A_ClosetChase` (also a 16-tic frame) runs path-direction only, `[H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:1595-1604`, `[H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:3406-3427`, `[H] Core/World/Entities/Definition/EntityFrameTable.cs:73-84`. Start tics are staggered across monsters (`AddFrameTicks`), `[H] Core/World/Entities/Enemy.cs:105-132`. Config `Game.MonsterCloset` defaults to true, `[H] Core/Util/Configs/Components/ConfigGame.cs:163-164`.

   When a player teleports into a closet island, `EntityTeleported` clears the island's closet flags (and its sector-islands'), `[H] Core/World/WorldBase.cs:4174-4198`. The loop meant to clear the monsters' flags then skips every entity that *has* `MonsterCloset` set (`!= 0 → continue`), so in practice it clears none of them, `[H] Core/World/WorldBase.cs:4202-4219`. Monsters still wake through `A_ClosetLook` once the sector's sound target is set. **Inference:** this looks like an inverted condition.

   **Inference:** on slaughter maps where thousands of monsters sit in teleport closets, this turns per-tic `A_Look` (sight checks, mostly REJECT plus BSP/blockmap traversal) into a cheap sound-target poll every 16 tics. That is a large CPU saving with no UZDoom equivalent: `grep -rni closet src` finds only unrelated `CloseTarFiles`.
4. **Voodoo-closet and automap helpers.** Voodoo-closet islands let the "mark specials" feature follow scripted voodoo lines, `[H] Core/World/Impl/SinglePlayer/MarkSpecials.cs:104-138`. Island centres are used for sector highlighting, `[H] Core/World/WorldBase.cs:3254-3263`. `ClassifyBridges` exists but has no caller, `[H] Core/World/Geometry/Islands/ClosetClassifier.cs:126`.

## Feature coverage vs UZDoom 5.0

Status key: **Supported** = parsed and acted on; **Partial** = subset, or parsed but not acted on; **Absent** = no code path. Each Absent entry was verified by search; the search is quoted.

| Feature | Helion status | Evidence | UZDoom 5.0 has it |
|---|---|---|---|
| ZScript | Absent | `grep -rli zscript` over the whole Helion tree (all file types, excluding `.git`): no hits | Yes, `[U] src/common/scripting/frontend/zcc_parser.cpp:410` |
| DECORATE | Partial: actors, flags, properties, states, `#include`. Both `const` and `enum` throw ("not supported"), and a parse error discards every actor in that lump. Action args are stored as raw tokens with no expression AST. 359 action names are mapped and **238 are empty stubs** (incl. `A_Jump`, `A_SpawnItemEx`, `A_CustomMissile`, `A_FireBullets`, `A_Warp`, `A_SetTranslation`). The 121 implemented are vanilla Doom, MBF and MBF21 codepointers plus a few others: `A_GiveInventory`, `A_SelectWeapon` and `HealThing` read DECORATE args; `A_Stop`; Helion's `A_FireRailGun`. `A_PlaySound` and `A_Explode` use DEH/vanilla semantics and ignore DECORATE args | `[H] Core/Resources/Definitions/Decorate/Parser/DecorateParser.cs:32-57`; `[H] Core/Util/Parser/ParserBase.cs:68-73`; `[H] Core/Resources/Definitions/Decorate/DecorateDefinitions.cs:34-41`; `[H] Core/Resources/Definitions/Decorate/Parser/DecorateStateParser.cs:177-232`; table `[H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:36-397`; stubs e.g. `[H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:1429-1432` (A_Jump), `[H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:2472-2475` (A_SpawnItemEx), `[H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:969`, `[H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:1193`, `[H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:2361`, `[H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:2779`; `A_PlaySound` `[H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:2829-2835`, `A_Explode` `[H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:1044-1050` | Yes, `[U] src/scripting/decorate/thingdef_parse.cpp:1254`, `[U] src/playsim/p_actionfunctions.cpp:798` |
| ACS | Partial: the VM is in the closed NuGet `Helion.HelionACS` (an ACSVM port). The host registers 69 opcodes and 26 CallFunc ids; 83 + 83 are commented out (e.g. SetActorProperty-class, CheckSight, PlaySound, Warp, SpawnDecal, user vars, CVar access). Only the map's `BEHAVIOR:` module loads; any library module is refused (no LOADACS). `CheckTag(PolyObject)` returns false | `[H] Core/Core.csproj:36`; `[H] Core/ACS/WorldExecutor.cs:42-235`, `[H] Core/ACS/WorldExecutor.cs:237-368`, `[H] Core/ACS/WorldExecutor.cs:421-438`, `[H] Core/ACS/WorldExecutor.cs:440-447`; `grep -rn LOADACS Core` (case-sensitive): none | Yes, 385 p-codes `[U] src/playsim/p_acs.cpp:77-465`, ACSF enum `[U] src/playsim/p_acs.cpp:4723` |
| Dehacked / BEX | Supported (Thing/Frame/Ammo/Weapon/Cheat/Text/Pointer/Misc/Sound, BEX STRINGS/CODEPTR/PARS/SOUNDS/SPRITES) | `[H] Core/Dehacked/DehackedDefinition.cs:100-145` | Yes, `[U] src/gamedata/d_dehacked.cpp:1196` |
| MBF21 | Supported (thing/weapon/frame flags, frame args) | `[H] Core/Dehacked/DehackedConstants.cs:533-572`, `[H] Core/Dehacked/DehackedConstants.cs:619-656`; `[H] Core/Dehacked/DehackedDefinition.cs:476` | Yes, `[U] src/gamedata/d_dehacked.cpp:1418` |
| ID24 (DEH bits, pickups, ammo categories, GAMECONF, SKYDEFS, TRAKINFO, SBARDEF, translations) | Supported | `[H] Core/Dehacked/DehackedConstants.cs:574`, `[H] Core/Dehacked/DehackedConstants.cs:657`; `[H] Core/Dehacked/DehackedApplier.cs:663-666`, `[H] Core/Dehacked/DehackedApplier.cs:774-866`; `[H] Core/Resources/Definitions/DefinitionEntries.cs:128-131`, `[H] Core/Resources/Definitions/DefinitionEntries.cs:607` | Partial: UMAPINFO id24 level number and `enteranim`/`exitanim` interlevel animations, plus optional `id24.wad` support-WAD loading. No `ID24 Bits`, SKYDEFS, GAMECONF, SBARDEF or TRAKINFO readers (grep of `src` and `wadsrc`): `[U] src/gamedata/umapinfo.cpp:63-69`, `[U] src/wi_stuff.cpp:377-410`, `[U] src/d_iwad.cpp:53` |
| UDMF namespaces | Partial: doom, dsda, eternity, zdoom, helion; others throw; no plane equations, vertex heights or portal keys | `[H] Core/Maps/Udmf/UdmfMap.cs:243-265`, `[H] Core/Maps/Udmf/UdmfMap.cs:473-554`, `[H] Core/Maps/Udmf/UdmfMap.cs:720-734` | Yes (+zdoomtranslated/hexen/heretic/strife/vavoom), `[U] src/maploader/udmf.cpp:2494-2560` |
| Node formats | Partial: GL v2/v5 and XGL*/ZGL* only (non-fractional XGLN/XGL2 splitters mis-scaled), always rebuilt by zdbspSharp; vanilla/XNOD not read | `[H] Core/Maps/Components/GL/GLComponents.cs:52-91`; `[H] Core/Maps/Components/ZNodes/ZNodesDefinition.cs:67-120`; `[H] Client/Client.Commands.cs:948` | Yes (vanilla, V4, XNOD/ZNOD, GL, internal builder, cache), `[U] src/maploader/maploader.cpp:3031-3130`, `[U] src/maploader/glnodes.cpp:921-984` |
| 3D floors | Partial: Sector_Set3dFloor types 1 solid / 2 swimmable / 3 non-solid plus flag bits, TransferLight-as-3D; type 0 (Vavoom) gets no type flags; `Sector_Attach3dMidtex` unhandled; 3D midtex flag supported | `[H] Core/World/Special/SpecialManager.cs:947-1009`; `[H] Core/Maps/Specials/ZDoom/ZDoom3DFloorType.cs:5-11`; unhandled-special diff (below) | Yes, `[U] src/playsim/p_3dfloors.cpp:94` |
| Slopes | Absent: planes always `(0,0,1,-z)`; Plane_Align (181, enum `SetSlope`) has no handler; no slope things, no UDMF plane keys | `[H] Core/World/Geometry/Sectors/SectorPlane.cs:50`; `[H] Core/Maps/Specials/ZDoom/ZDoomLineSpecialType.cs:186`; `grep "SetSlope\|PlaneAlign" Core/World`: none | Yes, `[U] src/maploader/slopes.cpp:145`, `[U] src/maploader/slopes.cpp:367` |
| Line / sector / skybox portals | Absent: values 57 (Sector_SetPortal) and 156 (Line_SetPortal) are `Unused15`/`Unused52` in the enum (the names exist only in the ACS/UDMF name table); Line_Horizon (9) unhandled; GLDEFS `skybox` parsed into `Skyboxes` but never read; "Portals" folder is flood-fill only | `[H] Core/Maps/Specials/ZDoom/ZDoomLineSpecialType.cs:14`, `[H] Core/Maps/Specials/ZDoom/ZDoomLineSpecialType.cs:62`, `[H] Core/Maps/Specials/ZDoom/ZDoomLineSpecialType.cs:161`; `[H] Core/Maps/Specials/ZDoom/ZDoomActions.cs:127-128`; `[H] Core/Resources/Definitions/ZDoom/GldefsDefinition.cs:29-30`; `grep "\.Skyboxes"`: none; `[H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Portals/PortalRenderer.cs:29-67` | Yes, `[U] src/playsim/portal.cpp:312`, `[U] src/r_data/portalgroups.cpp:468`, `[U] wadsrc/static/zscript/actors/shared/skies.zs:25` |
| Polyobjects | Absent: all Polyobj_* specials unhandled; no polyobj things or classes; ACS PolyWait waits on a tag that `CheckTag` reports busy | special diff; `grep -rli "polyobj" Core` hits only name tables, `WorldExecutor.cs` and `CheckTagType.cs`; `[H] Core/ACS/WorldExecutor.cs:445`, `[H] Core/ACS/WorldExecutor.cs:558-562` | Yes, `[U] src/maploader/polyobjects.cpp:342`, `[U] src/playsim/po_man.cpp:803` |
| Dynamic lights (GLDEFS lights) | Absent: GLDEFS parser handles only `#include`, `brightmap`, `skybox`; other keywords are skipped line by line | `[H] Core/Resources/Definitions/ZDoom/GldefsDefinition.cs:77-88`; whole-repo `grep -rli "pointlight\|dynamiclight\|dynlight\|lightdef"`: none | Yes, `[U] src/r_data/gldefs.cpp:224`, `[U] src/r_data/a_dynlightdata.cpp:171` |
| Brightmaps | Supported (GLDEFS brightmap, `brightmaps/auto`, BRGHTMPS) | `[H] Core/Resources/Definitions/ZDoom/GldefsDefinition.cs:57-63`, `[H] Core/Resources/Definitions/ZDoom/GldefsDefinition.cs:108-138`; `[H] Core/Resources/Definitions/DefinitionEntries.cs:133-135` | Yes |
| Models (MODELDEF/MD2/MD3/OBJ/IQM) | Absent: `Core/Models/*` are save-game DTOs | whole-repo `grep -rli "modeldef\|\.md2\|\.md3\|IQM"`: none | Yes, `[U] src/r_data/models.cpp:699-764`, `[U] src/common/models/models_md3.cpp` |
| Voxels (VOXELDEF/KVX) | Absent | whole-repo `grep -rli "voxel\|KVX\|VOXELDEF"`: none | Yes, `[U] src/r_data/voxeldef.cpp:256-260` |
| Custom hardware shaders / materials | Absent: no `hardwareshader`/`material` parsing | whole-repo `grep -rli "hardwareshader\|material "`: only licence text | Yes, `[U] src/r_data/gldefs.cpp:237-240` |
| Hexen map format | Supported (lines/things with args, BEHAVIOR) | `[H] Core/Maps/Hexen/HexenMap.cs:23-55`, `[H] Core/Maps/Hexen/HexenMap.cs:124-128` | Yes |
| Hexen game features (polyobjs, hubs, classes, Hexen IWAD) | Absent/Partial: no Hexen IWAD entry; clusters parse `hub` but `IsHub` has no reader (only copied in `ClusterDef`); ACS hub id always 0 | `[H] Core/Resources/IWad/IWadInfo.cs:36-47`; `[H] Core/Resources/Definitions/MapInfo/MapInfoDefinition.cs:474`; `[H] Core/World/WorldBase.cs:311`, `[H] Core/World/WorldBase.cs:332` | Yes, `[U] wadsrc/static/zscript/actors/hexen/baseweapons.zs:24` |
| Heretic | Absent (no IWAD entry, no actor definitions; only bounce-type keywords) | `[H] Core/Resources/IWad/IWadInfo.cs:36-47`; `Assets/Assets/Decorate` and `Assets/Assets/Actors` contain only Doom/Chex | Yes, `[U] wadsrc/static/zscript/actors/heretic/beast.zs:23` |
| Strife | Absent (DIALOGUE lump detected but never read; no actors) | `[H] Core/Resources/Archives/Iterator/ArchiveMapiterator.cs:28`; no consumer of `.Dialogue` | Yes, `[U] src/maploader/strifedialogue.cpp:89`, `[U] wadsrc/static/zscript/actors/strife/acolyte.zs:23` |
| Netplay / multiplayer | Absent: only `World/Impl/SinglePlayer` (co-op/DM item-stay rules and the thing MP flag exist, but there is no network layer) | `ls Core/World/Impl` → SinglePlayer; whole-repo `grep -rli "socket\|UdpClient\|TcpClient\|netgame"`: none | Yes, `[U] src/d_net.cpp:1309` |
| Demos | Partial: own tick-command format in a zip-like archive (`demodata.lmp`); no vanilla/ZDoom demo playback found | `[H] Core/Demo/DemoRecorder.cs:9-61`; `[H] Core/Demo/DemoArchive.cs:22` | Yes, `[U] src/g_game.cpp:2674`, `[U] src/g_game.cpp:2801` |
| Savegames | Supported (JSON `WorldModel`, source-generated serializer) | `[H] Core/World/Save/SaveGame.cs:172-173`, `[H] Core/World/Save/SaveGame.cs:237` | Yes, `[U] src/g_game.cpp:2043` |
| MAPINFO / ZMAPINFO / UMAPINFO | Supported (hub semantics unused) | `[H] Core/Resources/Definitions/DefinitionEntries.cs:120-122` | Yes, `[U] src/gamedata/g_mapinfo.cpp:2764` |
| SNDINFO | Supported ($alias, $ambient, $random, $limit, $pitchshift, $playersound…) | `[H] Core/Resources/Definitions/DefinitionEntries.cs:117`; `[H] Core/Resources/Definitions/SoundInfo/SoundInfoDefinition.cs:108-131` | Yes, `[U] src/sound/s_advsound.cpp:619` |
| SNDSEQ | Absent | `grep -rni "sndseq\|SoundSequence" Core`: only commented-out ACS entries, e.g. op 96, `[H] Core/ACS/WorldExecutor.cs:83` | Yes |
| TEXTURES | Partial: patches and offsets are composed; per-patch options (`Translation`, `FlipX/FlipY`, `Rotate`) are parsed but never applied, because composition uses only `component.Name` and `component.Offset` | `[H] Core/Resources/Definitions/DefinitionEntries.cs:124`; `[H] Core/Resources/Definitions/Texture/TexturesDefinition.cs:140-208`; `[H] Core/Resources/Images/ArchiveImageRetriever.cs:133-148` | Yes, `[U] src/common/textures/texturemanager.cpp:1028` |
| ANIMDEFS | Partial: flat/texture/switch supported; `ANIMATEDDOOR` throws "not supported" (the exception is logged and the rest of that ANIMDEFS lump is skipped); `WARP`/`WARP2`/`CAMERATEXTURE` parsed but never consumed | `[H] Core/Resources/Definitions/Animdefs/AnimatedDefinitions.cs:17-29`, `[H] Core/Resources/Definitions/Animdefs/AnimatedDefinitions.cs:50-66`, `[H] Core/Resources/Definitions/Animdefs/AnimatedDefinitions.cs:289-308`; `[H] Core/Resources/Definitions/DefinitionEntries.cs:265-281`; `grep "WarpTextures\|CameraTextures"` outside the parser: none | Yes, `[U] src/gamedata/textures/animations.cpp:300-321` |
| Translations | Partial: ID24 translations (`TranslationEntry`) are applied. DECORATE `Translation` (ranges/ICE) is parsed into `Properties.Translation` but has no reader. TEXTURES component translations are not applied (above). No TRNSLATE lump. `A_SetTranslation` is a stub | `[H] Core/Resources/Definitions/DefinitionEntries.cs:515-530`; `[H] Core/Resources/Definitions/Decorate/Parser/DecoratePropertyParser.cs:721-728`, `[H] Core/Resources/Definitions/Decorate/Parser/DecoratePropertyParser.cs:1298`; `grep TRNSLATE Core`: none | Yes, `[U] src/r_data/r_translate.cpp:733` |
| Reverb / EAX (REVERBS, sound zones) | Absent: only a FluidSynth MIDI reverb toggle | `[H] Core/Util/Configs/Components/ConfigAudio.cs:80-82`; `grep -rni "eax\|efx\|REVERBS"`: none | Yes, `[U] src/common/audio/sound/s_reverbedit.cpp:527-531`, zones `[U] src/maploader/maploader.cpp:367` |
| Transfer heights (242) – visual | Supported (render sector per view zone, 242 colormaps) | `[H] Core/World/Geometry/Sectors/TransferHeights.cs:6-53`; `[H] Core/World/Geometry/Builder/DoomGeometryBuilder.cs:94-102` | Yes, `[U] src/rendering/hwrenderer/scene/hw_fakeflat.cpp:208` |
| Swimmable water | Partial: only via swimmable 3D floors; Transfer_Heights flag bits (underwater=8, fakefloor-only, clip…) not handled | `[H] Core/World/Geometry/Sectors/Sector.cs:712-716`; `[H] Core/World/Entities/Entity.cs:1162-1173` | Yes, `[U] src/maploader/specials.cpp:659-674` |
| Colormaps | Supported: C_START/CC_START namespaces, 242 and SetSectorColormap side colormaps, UDMF lightcolor/fadecolor/fogdensity | `[H] Core/Resources/Archives/Wad/Wad.cs:47-49`; `[H] Core/World/Geometry/Builder/DoomGeometryBuilder.cs:92-111`; `[H] Core/Maps/Udmf/UdmfMap.cs:535-540` | Yes, `[U] src/r_data/colormaps.cpp:77` |
| Boom/MBF specials (generalised, friction, pushers, scrollers, 260 translucency, 271/272 sky transfer) | Supported | `[H] Core/Maps/Specials/Boom/BoomLineSpecTranslator.cs`; `[H] Core/Maps/Specials/Vanilla/VanillaLineSpecialType.cs:229`, `[H] Core/Maps/Specials/Vanilla/VanillaLineSpecialType.cs:266`, `[H] Core/Maps/Specials/Vanilla/VanillaLineSpecialType.cs:277-278` | Yes |
| ZDoom line specials | Partial: 157 distinct specials have a `case` in `SpecialManager.cs` (plus Sector_Set3dFloor/TransferLight via `==` in `InitSectors3D`). Of the 218 non-`Unused` enum names, 56 have no reference outside `LineSpecial.cs`'s classification switches, i.e. no handler. They include all Poly*, SetSlope, LineHorizon, LineMirror, LineAlignFloor/Ceiling, Floor/CeilingWaggle, ChangeCamera, Lightning, SetPlayerProperty, ForceField*, SectorAttach3DMidtex, SectorChangeFlags, ThingSetTranslation, ThingDamage, ThingThrust, DoorAnimated, AutoSave, ChangeSkill, Hexen-style Stairs_Build*/Pillar_*, Ceiling_CrushRaiseAndStay*/Ceiling_CrushStay, ScrollWall, ScrollTextureBoth, LightStop | `comm` of enum names vs `ZDoomLineSpecialType.*` references in `Core/World` and `Core/ACS` excluding `LineSpecial.cs`; `[H] Core/Maps/Specials/ZDoom/ZDoomLineSpecialType.cs:3-279`; `[H] Core/World/Special/SpecialManager.cs:738` | Yes, `[U] src/playsim/p_lnspec.cpp` |
| Monster-closet AI throttle, BSP point grid (Helion-only) | Supported | sections above | No (`grep -rni closet src` → only `CloseTarFiles`). The island viewer cull is **not** Helion-only: UZDoom has the `mapsection` cull (see Islands) |

**Perf cost in UZDoom of each feature Helion lacks or only partly supports (all Inference):**
- **ZScript:** every actor runs `Tick()` and `PostBeginPlay`/state actions through the VM (JIT on x64). On busy maps this is the largest CPU sink. Mods with per-tic `Tick` overrides or `ThinkerIterator` scans scale O(actors×iterators).
- **Full DECORATE:** anonymous functions and expressions compile to the same VM. `A_Jump*`/`A_SpawnItemEx`-heavy effects spawn many short-lived actors (allocation plus blockmap/sector linking per spawn).
- **Full ACS:** usually cheap. Cost comes from per-tic `while(1) delay(1)` HUD scripts and string ops, and `SetActorProperty`/`CheckSight` calls inside loops.
- **ID24 (UZDoom side):** no cost.
- **UDMF plane/vertex keys and other namespaces:** no runtime cost; slopes are the cost (below).
- **Node formats/cache:** UZDoom's choice to load the map's own nodes avoids Helion's always-rebuild. Its cost is the separate `gamenodes` array and memory.
- **Vavoom 3D floors / 3D midtex attach:** each 3D floor adds per-sector `extsector` light lists and clip planes. Hardware rendering splits walls per 3D-floor slice, multiplying wall draw records.
- **Slopes:** every `ZatPoint` becomes a dot product instead of a constant, in physics and rendering. Sloped flats cannot use axis-aligned shortcuts, and reflective planes are disabled on slopes, `[U] src/maploader/maploader.cpp:3215-3220`.
- **Portals:** each visible portal re-runs the BSP walk and scene setup with stencil. In stereo VR that cost is paid per eye, and recursion depth multiplies it. Line portals also add portal-group offset handling in collision and sight checks.
- **Polyobjects:** moving polyobjs rebuild per-subsector mini-BSPs (`BuildPolyBSP`) and update wall vertex data each move.
- **Dynamic lights:** per-frame light-to-section and light-to-wall linking (`touching_sides`, light lists), larger shader permutations, and extra light-buffer uploads. Shadowmaps add a 1D ray pass. This is one of the biggest GPU/CPU multipliers in VR.
- **Models/voxels:** extra draw calls and vertex processing per actor per eye, and model interpolation on the CPU.
- **Custom shaders/materials:** extra pipeline permutations that break batching and cost compile stalls on first use.
- **Hexen/Heretic/Strife game content:** only the cost of their actors (ZScript); polyobjs and hubs as above.
- **Netplay:** lockstep; CPU cost is negligible but adds latency coupling. Usually irrelevant for single-player VR.
- **Vanilla/ZDoom demos:** none.
- **SNDSEQ:** negligible.
- **ANIMDEFS warp:** a warp shader per texture, i.e. extra permutations. Camera textures are a full extra scene render per camera per frame, per eye in VR.
- **TRNSLATE / TEXTURES patch options:** none at run time (palette remap tables and patch compositing at load).
- **Reverb/EAX:** OpenAL EFX processing; the zone flood fill happens at load.
- **242 underwater flags:** extra heightsec-area classification per frame (`hw_fakeflat`).
- **Missing ZDoom specials:** waggle/lightning add thinkers; ChangeCamera implies camera textures (see ANIMDEFS warp/camera); otherwise negligible.

## Absent / not done (proven by search)

- **No use of WAD-supplied nodes or BLOCKMAP.** Nodes always come from zdbspSharp (`[H] Client/Client.Commands.cs:948`). The blockmap is always generated at 128 units from lines (`[H] Core/World/WorldBase.cs:423`). Searching for consumers of `.Blockmap`/`.Nodes`/`.Segments` on `MapEntryCollection` finds none.
- **No persistent node cache.** The zdbsp temp file is deleted after load, `[H] Core/Maps/Bsp/Zdbsp/Zdbsp.cs:49-50`.
- **Internal BSP builder unused:** `grep -rn "\bBspBuilder\b"` finds only its own definition.
- **No ZScript** (whole-repo `grep -rli zscript`: no hits), **no portals, slopes, polyobjects, dynamic lights, models, voxels or custom shaders** (searches in the table).
- **No ACS libraries / LOADACS.** `LoadModule` returns `[]` for any module name that is not `BEHAVIOR:<current map>`, `[H] Core/ACS/WorldExecutor.cs:421-438`.
- **No SNDSEQ, TRNSLATE, LOCKDEFS, DECALDEF, TERRAIN, REVERBS, VOXELDEF, MODELDEF, KEYCONF, MENUDEF or SBARINFO lump readers.** None of these names is registered in the lump dispatch table, `[H] Core/Resources/Definitions/DefinitionEntries.cs:111-135`. A case-insensitive grep for each under `Core`/`Client` finds nothing, except:
  - `LockDefinitions`, which is hard-coded defaults, `[H] Core/Resources/Definitions/Locks/LockDefinitions.cs:13-139`;
  - commented-out ACS terrain functions;
  - the stub `A_CHECKTERRAIN`.
- **No precomputed sector bounding boxes at load** (lazy, `[H] Core/World/Geometry/Sectors/Sector.cs:1089-1123`) and **no load-time sound-zone flood** (UZDoom has `FloodZones`, `[U] src/maploader/maploader.cpp:367-380`).
- **No lightmaps, level mesh or AABB tree.** UZDoom 5.0 builds `DoomLevelAABBTree` and `DoomLevelMesh` and loads lightmaps at load, `[U] src/maploader/maploader.cpp:3196`, `[U] src/maploader/maploader.cpp:3250-3251`; Helion has no equivalent (`grep -rni "lightmap\|aabbtree"` in `Core`: none).

## Perf-relevant settings

| Setting | Default | Effect | Citation |
|---|---|---|---|
| `Game.MonsterCloset` | `true` | Enables closet classification and `A_ClosetLook` throttling; map restart required | `[H] Core/Util/Configs/Components/ConfigGame.cs:163-164` |
| `Render.Mode` | `Static` | Static (pre-baked vertex cache) vs BSP vs Adaptive rendering; decides how much of the load-time static classification is used | `[H] Core/Util/Configs/Components/ConfigRender.cs:91-93` |
| `Render.MaxDistance` | `0` (off → `Constants.DefaultMaxDistance`) | Distance cull applied to island AABBs in dynamic-sector rendering | `[H] Core/Util/Configs/Components/ConfigRender.cs:134-136`; `[H] Core/Render/Renderer.cs:347-349`; use `[H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:239-241` |
| `Compatibility.PreferDehacked` | `True` | DEH over DECORATE when both exist (affects which definition path loads) | `[H] Core/Util/Configs/Components/ConfigCompat.cs:18-20` |
| Hard-coded grid sizes | physics 128, render 512, BSP grid 16 | Memory vs lookup cost trade-offs; not configurable | `[H] Core/World/WorldBase.cs:423`, `[H] Core/World/WorldBase.cs:437`; `[H] Core/World/WorldBase.BspBlockMap.cs:13-14` |
| UZDoom `gennodes` / `genblockmap` / `gl_cachenodes` / `gl_cachetime` | false / false / true / 0.6 | Force node or blockmap build; cache built nodes that took at least `gl_cachetime` seconds | `[U] src/maploader/maploader.cpp:62-63`; `[U] src/g_cvars.cpp:39-40` |

## Transfer notes for UZDoom VR

| Helion mechanism | UZDoom counterpart | Grade | Notes for stereo/Vulkan |
|---|---|---|---|
| BSP point grid (16-unit cell → start node/subsector) | `FLevelLocals::PointInSubsector` walks from `HeadGamenode()` each call, `[U] src/playsim/p_maputl.cpp:1946-1962` | **Port** | Build after `LoadLevel`'s node stage (`[U] src/maploader/maploader.cpp:3161`). It must index **gamenodes**, and must be rebuilt or disabled when polyobjects or nodes change (UZDoom has polyobjs; the grid stays valid only if `gamenodes` never change). Use `R_PointOnSide` fixed-point semantics so demos stay compatible. Bounds-check x and y separately (Helion checks only the flat index). Stereo-neutral; pure CPU win in `P_TryMove`/sight/spawn. |
| Island classification (map-wide components) and viewer-island cull | `subsector_t::mapsection` via `DoSetMapSection` `[U] src/maploader/renderinfo.cpp:77-102`; `CurrentMapSections` cull in `hw_bsp.cpp` `[U] src/rendering/hwrenderer/scene/hw_bsp.cpp:696-697`, portal-aware `[U] src/rendering/hwrenderer/scene/hw_portal.cpp:668`; per-sector `CreateSections` `[U] src/r_data/r_sections.cpp:167-200` | (already present) | UZDoom already has both the classification and a portal-aware version of the cull. Use `mapsection` directly as the "island" id for the closet port below. |
| Monster-closet AI throttle | None (`grep closet` none) | **Port** (Doom/DEH actors) / **Rework** (ZScript actors) | Hook `A_Look` in `p_enemy.cpp` (`[U] src/playsim/p_enemy.cpp:1912`). Classify per `mapsection`. ZScript monsters with custom `Tick` or non-`A_Look` spawn states must be excluded (Helion's `ValidForMonsterCloset` rule). Use a 16-tic polling state, and clear it on teleport-in or `NoiseAlert`; do not copy Helion's inverted clear loop. Stereo-neutral; helps keep the playsim inside the 11 ms (90 Hz) budget on slaughter maps. |
| Flood/bad-subsector sets | `hw_renderhacks.cpp` + `subsector_t::hacked` `[U] src/gamedata/r_defs.h:1693-1697` | (no transfer) | UZDoom already has a mature equivalent; no reason to port. |
| Always-rebuild GL nodes via zdbsp | `ForceNodeBuild`-only builds, `gl_cachenodes` | (reject) | Do not port. UZDoom's load-existing-nodes-plus-cache path is strictly better for load time. |
| Struct copies of hot line data (`StructLine`, `BlockLine`) | `line_t` already in a contiguous `TArray<line_t>` `[U] src/g_levellocals.h:496` | **Port** (selectively) | UZDoom's `line_t` is large. A hot SoA/AoS mirror (v1/v2, bbox, flags, front/back sector index) for `P_BlockLinesIterator`/sight would cut cache misses. It is valid only while polyobjects do not move lines, so update it in `FPolyObj::UpdateLinks` (`[U] src/playsim/po_man.cpp:803`) and whenever a line's special or flags change. |
| Exact-sized sector→line arrays | `sector_t::Lines` (`TArrayView`) built in `GroupLines` `[U] src/maploader/maploader.cpp:3166` | (already present) | — |
| Static/dynamic classification at load | UZDoom VBO (`CreateVBO` `[U] src/maploader/maploader.cpp:3232`) plus per-frame dirty checks | **Rework** | Covered in the render sections. Key for Vulkan: static geometry in device-local buffers, dynamic sectors in per-frame ring buffers; Helion's island-box render blockmap is a usable spatial index for "which dynamic sectors to re-upload". |
| Same-map restart reuse | `G_InitLevel` reloads fully | **Port** (optional) | Caching the built level and nodes across restarts reduces VR death-restart hitching. It is lower priority than node caching. |
| Feature subset (no ZScript, portals, etc.) | Full feature set | **Rebuild** only if you choose to drop features | You keep UZDoom's features. Helion shows where the costs are. In VR, gate the expensive features per map: portals per eye, camera textures, dynamic lights. They need a per-eye cost model (multiview or single-pass stereo for portals; clustered lights instead of per-surface light lists). |

## Open questions

- **zdbspSharp internals.** The package source is not available, so the following are unverified:
  - (a) whether it emits GL v5 or XGL3/ZGL3 for UDMF. If it ever emits XGLN/XGL2/ZGLN/ZGL2, Helion's ÷65536 splitter scaling (above) would corrupt the BSP.
  - (b) whether it copies, rebuilds or zero-fills REJECT. Helion nulls the source reject and uses the temp WAD's, `[H] Client/Client.Commands.cs:955-957`.
  - (c) whether it removes zero-length linedefs. This matters because `DoomGeometryBuilder` prunes zero-length lines and assigns `Line.Id = builder.Lines.Count` (`[H] Core/World/Geometry/Builder/DoomGeometryBuilder.cs:172-195`), while `CompactBspTree.GetSideFromEdge` indexes `builder.Lines[edge.Line.Id]` by *map* line id (`[H] Core/World/Bsp/CompactBspTree.cs:197-208`). If a zero-length line survives zdbsp, line ids would be off by one after it.
- **HelionACS VM** (`Helion.HelionACS 0.1.1-pre.52`) source is not available. Its p-code coverage (0–56, 69–79 and similar "internal" ranges) and its CheckTag semantics are unverified. `CF_PolyWait` → `CheckTag(PolyObject) == false` probably blocks forever (Inference from `CheckSectorMovement` returning `true` when idle, `[H] Core/ACS/WorldExecutor.cs:450-458`).
- **Actual load-time cost** of the zdbsp pass vs UZDoom's node load or build was not measured (no runtime in this audit).
- **Island cull with 3D floors.** `RenderSectors` intentionally does not short-circuit when 3D sectors share an island (`[H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:227-230`). Whether every case is correct was not verified.
- **`MarkSpecials` comparison.** `world.ToSubsector(...).Id != sector.Id` compares a subsector id with a sector id (`[H] Core/World/Impl/SinglePlayer/MarkSpecials.cs:266`). This looks like a bug, but it is outside this section's perf scope.
