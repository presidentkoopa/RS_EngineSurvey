# H4 — Helion playsim: what runs every tic

Helion's simulation is a single-threaded, allocation-averse rewrite of the idtech1 tic loop in C#. Positions, velocities and plane heights are all `double`, and it imitates fixed-point behaviour only in a few places. Its speed comes from data layout, not from skipping work. Every live entity is walked once per tic through an intrusive linked list. The blockmap stores a flat `int[]` of entity *pool indices* per block, plus a pre-sorted array of line records copied into each block (denormalised). Duplicates are filtered with a global check counter, the pattern idtech1 calls validcount. A 16-unit grid caches BSP point lookups. Spawned and destroyed entities come from and return to an index-addressed object pool. Two real "do less work" mechanisms exist, and both are AI-level rather than tic-loop culling. **Monster-closet detection** is on by default: it swaps `A_Look` for a cheap sound-only poll at map load. **Slow tick** is off by default: it multiplies the frame duration of chase-, look- and tracer-type frames (`A_Chase`/`A_VileChase`/`A_HealChase`, `A_Look`, `A_Tracer`/`A_SeekTracer`) for actors that are far away or were not drawn. Action functions are plain C# delegates looked up by name once at load. There is no VM and no expression evaluation, so the cost per action call is one indirect call. The other side of that: 237 of 359 registered action names map to empty `// TODO` stubs. Two code-traced defects matter for a port: the entity walk can end early when an entity destroys its list successor (§1), and the "player teleported into a closet" wake-up loop is inverted and does nothing (§6).

---

## 1. The tic loop: order of operations and per-entity work

### Top-level order

`SinglePlayerWorld.Tick` runs two jobs before handing off to `WorldBase.Tick`. It pushes an automap marker, and it fires a full crosshair trace every tic: `GetAutoAimEntity` when autoaim is on, otherwise `FireHitscan` with the sentinel damage `HitscanTestDamage` [H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:230-247, 268-277. `AutoAim` defaults to `true` [H] Core/Util/Configs/Components/ConfigGame.cs:19-21, so the crosshair costs one blockmap ray trace of `EntityShootDistance = 2048` units per tic [H] Core/Util/Constants.cs:389.

`WorldBase.Tick` (Normal state) [H] Core/World/WorldBase.cs:1072-1095:

```csharp
TickPlayers();
TickEntities();
AcsExecutor.Exec();
SpecialManager.Tick();
TickScrollers();
...
ArchiveCollection.TextureManager.Tick();
SoundManager.Tick();
LevelTime++;
```
[H] Core/World/WorldBase.cs:1075-1094

When paused, only status bars tick [H] Core/World/WorldBase.cs:1044-1049. Profiler scopes wrap the world, players and entities (`Profiler.World.Total/TickPlayer/TickEntity`) [H] Core/World/WorldBase.cs:1051,1101,1169,1212,1238,1268.

`TickPlayers` handles the command, runs `PlayerTick` (inventory, weapon animation, status bar), checks secrets and checks music-changer sectors through a dictionary lookup [H] Core/World/WorldBase.cs:1236-1269, [H] Core/World/Entities/Players/Player.cs:835-865. Player *movement* does not happen here: players are ordinary members of the entity list and get moved in `TickEntities`.

### TickEntities: every entity, every tic

```csharp
while (entity != null) {
    nextEntity = entity.Next;
    entity.PrevPosition = entity.Position;
    if (!entity.IsDisposed) {
        PhysicsManager.Move(entity);
        if (entity.Respawn) HandleRespawn(entity);
        m_sectorDamageSpecials[entity.SectorId]?.Tick(entity, DamageTickOptions.CheckOnFloor);
        if (WorldStatic.Sector3D && entity.Sector.HasDamageSector3D) CheckDamageSector3D(entity);
        if (!WorldStatic.InfinitelyTallThings && (entity.HadOnEntity || entity.OnEntity() != null) && ...)
            m_fallCheckEntities.Add(entity);
    }
    entity.Tick();
    if (WorldState == WorldState.Exit) break;
    entity = nextEntity;
}
```
[H] Core/World/WorldBase.cs:1173-1206 (condensed)

After the loop, `EntityFallCheck` re-clamps entities that were standing on another entity that moved or vanished [H] Core/World/WorldBase.cs:1210-1211, [H] Core/World/Physics/PhysicsManager.cs:158-176.

**What idle entities skip.** Nothing is removed from the walk. Decorations, items, corpses and closeted monsters are all visited. Their cost is kept small by early exits:
- `Move` → `MoveXY` returns at once when `Velocity.X == 0 && Velocity.Y == 0`; a skullflyer first gets `HandleEntityHit` [H] Core/World/Physics/PhysicsManager.cs:2107-2118.
- `MoveZ` returns when there is no Z velocity, no gravity is due (on the ground), no float flag, and the entity is not standing on a moving 3D floor [H] Core/World/Physics/PhysicsManager.cs:2209-2212.
- `Entity.Tick` clears the teleport flag. It then returns at once if the entity is `Dormant`; otherwise it decrements `FrameState.CurrentTick`, and the frame's action runs only when the counter reaches zero [H] Core/World/Entities/Entity.cs:549-562, [H] Core/World/Entities/Definition/States/FrameState.cs:272-284. A `-1` tic count (static frame) returns before decrementing.
- `Dormant` is the only "sleep" flag. It skips `Entity.Tick` but **not** `Move` or the sector-damage check, because those run before `entity.Tick()` in the loop [H] Core/World/WorldBase.cs:1181-1200, [H] Core/World/Entities/Entity.cs:553-554.

**Inference:** a resting decoration costs a pointer chase to the `Entity` object, about 10 branches, one array load for the damage-special slot, and a frame-counter decrement. At 10k–50k entities this is a few hundred microseconds per tic in .NET, dominated by cache misses on ~650-byte objects scattered in the managed heap. They are not contiguous even though they are pooled (§2).

**Spawn order quirk.** New entities are *prepended* at `Head` [H] Core/World/Entities/EntityManager.cs:416-427. The walk goes head→tail, so an entity spawned during the walk is not ticked until the next tic. Save files serialise entities backwards to keep that order [H] Core/World/Entities/EntityManager.cs:333-334. UZDoom instead puts new thinkers on a "fresh" list that `TickThinkers` moves across and ticks in the same pass, using `OF_JustSpawned` [U] src/playsim/dthinker.cpp:812-850.

**Successor-destroy bug (real per code trace; not run).** The steps:
1. `TickEntities` caches `nextEntity = entity.Next` before it moves or ticks the current entity E [H] Core/World/WorldBase.cs:1175,1205.
2. `EntityManager.Destroy` calls `Entity.Dispose` immediately, with no deferral [H] Core/World/Entities/EntityManager.cs:133-151. `Dispose` sets `IsDisposed` and calls `Unlink`, which patches E's `Next` past the removed node and then sets the removed node's own `Next = null` and `Previous = null` [H] Core/World/Entities/Entity.cs:1238-1241,1304-1323.
3. So if E's `Move` or `Tick` destroys exactly E's successor N, the loop's next step is `entity = N`. It reads `nextEntity = N.Next == null`, skips `Move` (N is disposed), calls the harmless `N.Tick()`, and **exits**. Every entity after N gets no `Move` and no `Tick` that tic.

Nothing guards against this: there is no check that `nextEntity` is still linked, and no deferred-free list.

Concrete trigger: an item pickup during the player's `Move`. `IsPositionValid` → `PerformItemPickup` → `EntityManager.Destroy(item)` [H] Core/World/Physics/PhysicsManager.cs:1688-1693, [H] Core/World/WorldBase.cs:2507-2529. At map load all things are created first and the player afterwards, prepended at `Head` [H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:70-78, [H] Core/World/Entities/EntityManager.cs:416-427. The list is therefore: runtime spawns (newest first) → voodoo dolls → player → every map-loaded thing. `player.Next` is the last thing created at load. If that is an item and the player picks it up, all map-loaded entities skip one tic.

A second variant follows from the LIFO free list. `FreeEntity` pushes the index and `GetEntity` pops it first [H] Core/Util/DataCache.cs:100-105,125-132. If anything spawns after the destroy but before the loop advances, N's object is re-`Set` and prepended at `Head`. The loop then follows `N.Next` back to the old head and **ticks earlier entities a second time**. For pickups this needs a spawn later in the same player `Move`/`Tick`, for example a crossed line running `Thing_Spawn`, which is rare.

UZDoom avoids both: `DThinker::Remove` advances the global `NextToThink` when the removed thinker is the one about to be visited [U] src/playsim/dthinker.cpp:956-961. A port should use that pattern or deferred destruction.

### Rest of the tic
- `TickScrollers` drains a `Dictionary<int, ScrollAccumulator>` keyed by entity index into velocities [H] Core/World/WorldBase.cs:1104-1124.
- `TextureManager.Tick` steps animated-texture translations and sky scroll [H] Core/Resources/TextureManager.cs:479-504.
- `SpecialManager.Tick` and `SoundManager.Tick` are covered in §4 and §7.

---

## 2. Entity data layout, storage, linking, pooling

### Field layout

`Entity` is a `partial class` spread over Entity.cs, Enemy.cs and EntityBox.cs [H] Core/World/Entities/Entity.cs:40, [H] Core/World/Entities/Enemy.cs:10, [H] Core/World/Entities/EntityBox.cs:10. Grepping instance-field declarations finds about 88 fields [H] Core/World/Entities/Entity.cs:50-191, [H] Core/World/Entities/Enemy.cs:35-37. Notable ones:
- **28 object references**: `World`, `Next/Previous`, `RenderBlockNext/Previous`, `Definition`, `Properties`, `Sector`, `HighestFloorSector`, `LowestCeilingSector`, `HighestFloorObject/LowestCeilingObject` (typed as an interface), `Blocking*`, `ThingIdNode`, `AudioSource`, and the five weak-target references `m_target`, `m_tracer`, `m_owner`, `m_onEntity`, `m_overEntity` [H] Core/World/Entities/Entity.cs:50-191.
- **4 `Vec3D`s** (`Position`, `Velocity`, `PrevPosition`, `SpawnPoint`), each 3 doubles [H] Core/World/Entities/Entity.cs:67-68,94-95, [H] Core/Geometry/Vectors/Vec3D.cs:13-20.
- **About 9 more doubles**: angle, floor/ceiling Z, height, radius, gravity, render distance, closet chase speed, monster speed [H] Core/World/Entities/Entity.cs:66,111-112,142,145,158-160, [H] Core/World/Entities/Enemy.cs:37.
- `EntityFlags` is a struct of three ints, read through inlined bit tests [H] Core/World/Entities/Definition/Flags/EntityFlags.cs:8-31.
- `FrameState` is an embedded struct: an `EntityFrame` reference, options, current tic and frame index [H] Core/World/Entities/Definition/States/FrameState.cs:23-42.
- `SpecialArgs` (5 ints and a string ref) is inline [H] Core/Maps/Specials/SpecialArgs.cs:10-17.
- Two owned heap side-objects: `IntersectSectors` is a `DynamicArray<IntersectSectorNode>` of 16 slots rented from `ArrayPool`, and `IntersectMidTexLines` is a `DynamicArray<int>` [H] Core/World/Entities/Entity.cs:113,151.

**Inference:** about 600–700 bytes per object, plus about 400 bytes of side arrays, so roughly 1 KB per entity. `Player` is a subclass and much larger. Hot fields (`Flags`, `Position`, `Velocity`, `FrameState`) sit inside the first ~200 bytes of declarations, but the CLR does not guarantee field order for auto-layout classes.

**Weak references by ID.** Target, tracer, owner and similar links store both the object reference and its `Id`. A link is valid only if `m_target.Id == m_targetId` [H] Core/World/Entities/Entity.cs:76-89, 178-191. `Dispose` sets `Id = int.MinValue` [H] Core/World/Entities/Entity.cs:1237, and pooled reuse assigns a fresh ID [H] Core/Util/DataCache.cs:99-105. So every stale pointer is invalidated in O(1) with no back-reference scan. `WeakEntity` is the same idea as a struct, used for `Sector.SoundTarget` [H] Core/World/Entities/WeakEntity.cs:7-23.

`Properties` is a reference to the definition's shared `EntityProperties` class [H] Core/World/Entities/Entity.cs:220, [H] Core/World/Entities/Definition/Properties/EntityProperties.cs:10. `EntityManager.Create` writes `entity.Properties.MonsterMovementSpeed = FastSpeed` [H] Core/World/Entities/EntityManager.cs:99-100, so it mutates the shared definition. It does not affect the entity being created. That entity already copied `MonsterMovementSpeed` from `Properties` inside `Set`, which runs before this write [H] Core/World/Entities/Entity.cs:244, [H] Core/World/Entities/EntityManager.cs:97. Movement reads the per-entity copy [H] Core/World/Entities/Enemy.cs:341-342. So with fast monsters on, the first instance of each class with a `FastSpeed` keeps normal speed, and every later spawn of that class gets `FastSpeed`.

### Containers

| Relation | Structure | Evidence |
|---|---|---|
| All entities (tick order) | Intrusive doubly linked list (`Entity.Next/Previous`, `EntityManager.Head`) | [H] Core/World/Entities/Entity.cs:52-53, [H] Core/World/Entities/EntityManager.cs:37,416-427 |
| Entity ↔ blockmap block (physics) | Per block a struct `BlockEntities {int[] EntityIndices; int EntityIndicesLength}` holding **pool indices** | [H] Core/World/Blockmap/BlockMap.cs:27-31,203-238 |
| Entity ↔ render block (512-unit grid) | Intrusive list (`RenderBlockNext/Previous`, `HeadRenderEntities[]`) | [H] Core/World/Blockmap/BlockMap.cs:240-280, [H] Core/World/WorldBase.cs:437 |
| Entity ↔ sector (touching list) | `Sector.Entities` is a `LinkableList<Entity>`; the entity keeps `(Sector, LinkableNode)` pairs in `IntersectSectors` | [H] Core/World/Geometry/Sectors/Sector.cs:46,659-663, [H] Core/World/Entities/Entity.cs:31-35 |
| TID lookup | `Dictionary<int, LinkableList<Entity>>` + `ThingIdNode` | [H] Core/World/Entities/EntityManager.cs:49,648-660 |
| ID lookup | `LookupArray<Entity?>` | [H] Core/World/Entities/EntityManager.cs:48,75-80 |

`LinkableList` uses a dummy head so `Unlink` needs no list reference. Nodes come from a per-type static pool (`LinkableNode<T>.Pool`, initial capacity 1024) and return to it on unlink [H] Core/Util/Container/LinkableList.cs:42-54,135,198-204. Linking an entity into a sector therefore allocates nothing in steady state.

The blockmap block array starts at 8 slots and doubles on overflow [H] Core/World/Blockmap/BlockMap.cs:81-82,232-235. Unlinking scans the block's array from the end and closes the gap with `Array.Copy`, which keeps insertion order [H] Core/World/Entities/Entity.cs:521-547. **Inference:** this keeps iteration order deterministic (it matters for collision tie-breaks), but it is O(n) per block in crowded blocks. That is softened by the "same block range → skip relink" short-circuit [H] Core/World/Blockmap/BlockMap.cs:216-219: a monster moving 8 units in a 128-unit block keeps its block range most tics.

### Pooling on spawn and death

`DataCache` holds `Entity[] Entities` (index 0 reserved), pre-built with 1023 `Entity` objects when it is constructed [H] Core/Util/DataCache.cs:35-37,65-72. At map load it is grown to 1.2× the thing count [H] Core/Util/DataCache.cs:134-148. `GetEntity` pops a free index if one exists, otherwise takes the next unused slot, doubling the array when needed [H] Core/Util/DataCache.cs:96-123,150-165. `FreeEntity` pushes the index back for anything that is not a player [H] Core/Util/DataCache.cs:125-132. `Entity.Dispose` resets about 40 fields, unlinks from the world and the entity list, and frees to the cache. If a sound is still playing on the entity, the free is deferred until the sound clears, because the audio source holds a back-reference [H] Core/World/Entities/Entity.cs:1222-1302,1368-1380.

`Entity.Set` re-initialises from the definition. `Flags` (a struct) is copied by value. `Properties` is assigned by reference, so it is shared with the definition. A handful of mutable values (speed, alpha, ranges) are copied into per-entity fields, and `FrameState` is reset [H] Core/World/Entities/Entity.cs:208-252. **Inference:** spawning a projectile costs no GC allocation in steady state. The pool index doubles as the blockmap key, so blockmap arrays stay `int[]` instead of reference arrays. That halves their size and keeps them GC-scan free, at the cost of one extra indirection through `DataCache.Entities[]` per candidate.

---

## 3. Movement and collision

### Call graph per moving entity

`Move` → `MoveXY` → `TryMoveXY` → (per sub-step) `IsPositionValid` + `CheckDropOff` → `MoveTo` → `LinkToWorld` → `ClampBetweenFloorAndCeiling`. Then `MoveZ` → `ClampBetweenFloorAndCeiling` [H] Core/World/Physics/PhysicsManager.cs:147-156,2107-2162,1471-1590,1831-1857,127-141,2204-2264.

**Sub-stepping.** A move is split into `numMoves = maxAxis / (radius - 0.5) + 1` steps so fast objects cannot tunnel. Wall sliding is bounded by `MaxSlides = 3` [H] Core/World/Physics/PhysicsManager.cs:1494-1507,41. XY velocity is clamped to ±30 units/tic [H] Core/World/Physics/PhysicsManager.cs:50,2120-2126. Below `MinMovement = 0.0625`, velocity is zeroed for non-players, and for players with no move input; otherwise sector friction is applied [H] Core/World/Physics/PhysicsManager.cs:43,2130-2161.

### IsPositionValid (the PIT_CheckThing/PIT_CheckLine equivalent)

This is one fused loop over the blocks under the entity's box [H] Core/World/Physics/PhysicsManager.cs:1601-1797:
1. `ToSubsector(x, y)` for the destination sector [H] Core/World/Physics/PhysicsManager.cs:1626.
2. One counter per call: `checkCounter = ++WorldStatic.CheckCounter` [H] Core/World/Physics/PhysicsManager.cs:1638.
3. For each block, entities first (only when the mover is solid or a missile). Each entity is filtered by counter, then by a flag mask, then by a box test:

```csharp
nextEntity = m_dataCache.Entities[entityIndices[i]];
if (nextEntity.BlockmapCount == checkCounter) continue;
nextEntity.BlockmapCount = checkCounter;
if (entity == nextEntity || ((nextEntity.Flags.Flags1 & PositionValidFlags1) == 0 && (nextEntity.Flags.Flags2 & PositionValidFlags2) == 0))
    continue;
var blockDist = nextEntity.Radius + entity.Radius;
if (Math.Abs(nextEntity.Position.X - x) >= blockDist | Math.Abs(nextEntity.Position.Y - y) >= blockDist)
    continue;
```
[H] Core/World/Physics/PhysicsManager.cs:1668-1679

   Item pickup happens inline [H] Core/World/Physics/PhysicsManager.cs:1688-1693.
4. Then that block's lines, from a contiguous slice of `BlockLines[]`, are deduplicated by `WorldStatic.CheckedLines[lineId] == checkCounter`, then box-tested and run through `LineBlocksEntity` [H] Core/World/Physics/PhysicsManager.cs:1705-1751. While scanning, it fills `tryMove.IntersectSectors`, `IntersectSpecialLines` and `IntersectMidTexLines`. `LinkToSectors` and the cross-line special check reuse these afterwards without rescanning [H] Core/World/Physics/PhysicsManager.cs:1403-1414,1846-1856.

All scratch state lives in one reused `TryMoveData` instance owned by `PhysicsManager` [H] Core/World/Physics/PhysicsManager.cs:54, [H] Core/World/Physics/TryMoveData.cs:8. **Inference:** that makes the movement code non-reentrant and single-threaded by construction.

### Duplicate-avoidance counters

`WorldStatic.CheckCounter` is a single global `int` [H] Core/World/WorldStatic.cs:20. Its marks are:
- `Entity.BlockmapCount` for entities [H] Core/World/Entities/Entity.cs:62
- `int[] CheckedLines`, sized to the line count, for lines [H] Core/World/WorldStatic.cs:55, [H] Core/World/WorldBase.cs:708
- `Sector.CheckCount` for sectors [H] Core/World/Geometry/Sectors/Sector.cs:88, [H] Core/World/Physics/PhysicsManager.cs:1402-1411

Every traverser starts with `++WorldStatic.CheckCounter` [H] Core/World/Physics/Blockmap/BlockmapTraverser.cs:34,59,112,180,205,244,272,313,353,384,432. This is exactly idtech1/UZDoom `validcount` (UZDoom: `validcount++` in `P_CheckPosition` [U] src/playsim/p_map.cpp:1845,1930 and `P_FindFloorCeiling` [U] src/playsim/p_map.cpp:381, and in sight [U] src/playsim/p_sight.cpp:629,917). The difference is that Helion keeps the line marks in a separate dense `int[]` rather than inside each line object.

### Blockmap construction (built in-engine, not read from the lump)

`new BlockMap(Lines, 128)` builds a 128-unit grid from line geometry [H] Core/World/WorldBase.cs:423. The grid bounds are padded by one block on each side, so no bounds branch is needed in the DDA [H] Core/World/Blockmap/BlockMap.cs:126-154. Each (block, line) pair becomes a `BlockLine` struct holding the full `Seg2D` (start, end, delta and AABB as doubles), the blocking flags, a one-sided bool, a has-special bool and the front/back `Sector` references [H] Core/World/Blockmap/BlockLine.cs:8-17, [H] Core/Geometry/Segments/Seg2D.cs:16-22. The array is sorted by (block, lineId) "because Doom added lines to the block in line id order" [H] Core/World/Blockmap/BlockLine.cs:19-27, [H] Core/World/Blockmap/BlockMap.cs:372-373. Each block stores only `{start, count}` into that array [H] Core/World/Blockmap/BlockMap.cs:33-37,376-393. `MapEntryCollection` records a `Blockmap` entry [H] Core/Maps/MapEntryCollection.cs:26, but `grep -rni blockmap Core/Maps/*.cs Core/Maps/Doom/*.cs` finds no reader of it, so the BLOCKMAP lump is not used for physics.

**Inference:** this layout is the main "designed for speed" structure in the playsim. A block's line candidates are a contiguous run of roughly 120-byte structs (an 80-byte `Seg2D`, a 14-bool `LineBlockFlags`, two ints, two bools and two sector references): no `line_t*` dereference, no `vertex_t` loads, and the AABB reject comes first. The cost is memory duplication (a line spanning *k* blocks is stored *k* times) and a change in vanilla blockmap quirks: lines missing from a lump's blocks, or a 128-unit grid origin taken from the lump header. Vanilla-exact demo behaviour depending on blockmap-lump bugs cannot be reproduced.

Ray traversal uses `BlockmapSegIterator`, an error-term DDA over block indices in doubles [H] Core/World/Blockmap/BlockMap.cs:434-520.

### Line intersection maths

All of it is in `double`. The box-versus-segment test does an AABB reject, then two perpendicular-dot sign tests with an `Epsilon` snap to zero [H] Core/Geometry/Segments/Seg2D.cs:352-385. The side test is `PerpDot` [H] Core/Geometry/Segments/Seg2D.cs:112-115. Crossing a special line is detected by comparing `PerpDot` sign before and after the move [H] Core/World/Physics/PhysicsManager.cs:1846-1856. The BSP point test is a `double` cross product with special cases for axis-aligned splitters "to match dooms behavior" [H] Core/World/Bsp/CompactBspTree.cs:132-149. `grep "Fixed\." Core/World` (excluding `FromFixed` and `Fixed.Lowest`) finds no fixed-point arithmetic in the physics.

### Point → subsector cache

`ToSubsector` first looks up a 16-unit grid (`m_bspBlockmapNodeIndices`). Each cell holds either a subsector, when the cell is fully inside one, or the deepest BSP node containing the cell. The tree walk then starts from that node instead of the root [H] Core/World/WorldBase.BspBlockMap.cs:12-49. The tree walk itself uses a flat node array with a branch-free child select (`*(byte*)&onRight`) [H] Core/World/Bsp/CompactBspTree.cs:117-130. **Inference:** most `ToSubsector` calls (each `IsPositionValid`, `LinkToSectors`, spawn) become one array load. Memory is (map extent / 16)² × 4 bytes. For a 16k×16k map that is 1M cells, about 4 MB.

### Height clipping

`ClampBetweenFloorAndCeiling` → `SetEntityBoundsZ` → `GetEntityClampValues` finds the highest floor and lowest ceiling among the sectors the entity touches. `TryMoveData` supplies them when available [H] Core/World/Physics/PhysicsManager.cs:995-1063,1095-1207,1331-1389. The engine then considers things the entity can stand on or bump into (`CanPass` / "height-sensitive" clipping):
- During XY moves it reuses `tryMove.IntersectEntities2D` gathered by `IsPositionValid`.
- During pure Z moves (`tryMove == null`) it runs a **fresh blockmap entity traversal** [H] Core/World/Physics/PhysicsManager.cs:1139-1170.

**Inference:** every airborne or falling `CanPass` entity pays one blockmap box query per tic just for Z.

Clamping reads planes as scalars (`sector.Floor.Z`, `Ceiling.Z`) [H] Core/World/Physics/PhysicsManager.cs:1361-1388. A `short.MinValue` check emulates fixed-point overflow for "mikoportals" [H] Core/World/Physics/PhysicsManager.cs:1013-1025. Sloped floors are not implemented: `// TODO implement when slopes exist` [H] Core/World/Geometry/Sectors/Sector.cs:670, and `SectorPlane` builds a horizontal `PlaneD(0,0,1,-z)` [H] Core/World/Geometry/Sectors/SectorPlane.cs:50.

### Linking after a successful step

`MoveTo` calls `UnlinkFromWorld(unlinkBlockmapBlocks: false)`, which unlinks sector nodes and the render block, then `LinkToWorld(..., checkLastBlock: true)` [H] Core/World/Physics/PhysicsManager.cs:1831-1841. Each sub-step therefore costs:
- a pool pop and push for every touched sector,
- a render-block relink,
- a blockmap relink only when the block range changed [H] Core/World/Blockmap/BlockMap.cs:216-219,
- a full Z clamp.

---

## 4. Sector specials and movers

### Storage and ticking

All specials share one `System.Collections.Generic.LinkedList<ISpecial>`: movers, lights, scrollers, pushers, switches and stairs [H] Core/World/Special/SpecialManager.cs:46. Nodes are recycled through `DataCache.GetSpecialNode` [H] Core/World/Special/SpecialManager.cs:471, [H] Core/Util/DataCache.cs:225-241. `SpecialManager.Tick` walks the list and calls `ISpecial.Tick()`, an interface dispatch. A special returning `Destroy` is unlinked and recorded in `m_destroyedMoveSpecials`. Next tic it is finalised: the `SectorMoveComplete` event fires and the object goes back to the pool [H] Core/World/Special/SpecialManager.cs:362-446. Movers, light changes and switches are pooled (`GetSectorMoveSpecial`/`FreeSectorMoveSpecial`, etc.) [H] Core/Util/DataCache.cs:261-380.

Lookups such as `FindSpecialBySector`, `FindLineScrollSpecial` and `RemoveSpecial` are linear list scans [H] Core/World/Special/SpecialManager.cs:474-519. **Inference:** permanent lighting specials (flicker, strobe, glow) are ticked every tic forever. On maps with thousands of light-effect sectors the list walk becomes a fixed per-tic cost: one interface call and one node per sector.

### SectorMoveSpecial.Tick → MoveSectorZ

`SectorMoveSpecial.Tick` handles pause and delay, then computes the destination and calls `m_world.MoveSectorZ` through `PerformAndHandleMoveZ` [H] Core/World/Special/Specials/SectorMoveSpecial.cs:216-294,499-501. The world wrapper stamps `Sector.MoveEventGameTick`. It fires `SectorMoveStart` on the first move, runs the physics, then fires `SectorMove` [H] Core/World/WorldBase.cs:2994-3003. `PhysicsManager.MoveSectorZ` [H] Core/World/Physics/PhysicsManager.cs:178-437:
- moves linked sectors first,
- collects the sector's touching entities from `Sector.Entities` and **sorts** them with a cached comparison (lowest first, then by ID for renderer z-fighting stability) [H] Core/World/Physics/PhysicsManager.cs:203-207,595-612,
- sets the plane Z tentatively, re-clamps each entity twice, crushes or pushes, and resets the plane and every entity if blocked [H] Core/World/Physics/PhysicsManager.cs:214-400.

**Inference:** cost per moving plane per tic is O(k log k) in the k entities touching that sector, plus two full Z clamps each. Each clamp may do a blockmap `EntityTraverse` (§3). A lift carrying 50 corpses therefore performs about 100 blockmap queries per tic.

Code-quality note: `GetSectorMoveOrderedEntities` hits `continue` without advancing `node` when `EntityHasMovementSector` is false [H] Core/World/Physics/PhysicsManager.cs:597-603. That would loop forever. It is safe only because the list membership and `IntersectSectors` are kept in sync.

### How changes reach the renderer

The renderer is notified in three ways:

1. **Plane Z: polled.** `SectorPlane` carries `Z`, `PrevZ` and `LastRenderChangeGametick`. The geometry renderer calls `CheckRenderingChanged()` (true if changed in the last tic, if `PrevZ != Z`, or if render offsets changed) for each sector and side it draws [H] Core/World/Geometry/Sectors/SectorPlane.cs:15-21,82-100, [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/GeometryRenderer.cs:509-515.
2. **Mover start and end: events.** `StaticCacheGeometryRenderer` subscribes to `SectorMoveStart`/`SectorMoveComplete`, and to `SectorMove` only when 3D floors are present. It uses them to move sector geometry out of and back into the static vertex cache [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Static/StaticCacheGeometryRenderer.cs:110-118,965-989,1120-1143. These events are synchronous C# multicast delegates raised inside the tic.
3. **Light: event plus a dirty queue.** `Sector.SetLightLevel` sets a `DataChanges` flag and `RenderLightChangeGametick` [H] Core/World/Geometry/Sectors/Sector.cs:271-278. The world raises `SectorLightChanged` [H] Core/World/WorldBase.cs:4619-4631. The renderer adds the sector to a `SectorUpdates` set, deduplicated per frame with a counter array [H] Core/Render/Renderer.World.cs:386-388, [H] Core/Render/SectorUpdates.cs:27-43. It then writes only those sectors into a mapped buffer texture of per-sector light levels [H] Core/Render/Renderer.World.cs:423-440.

---

## 5. State machine and action dispatch

- **Frame table.** All states of all actors live in one global `List<EntityFrame>` (`WorldStatic.Frames`) [H] Core/World/WorldStatic.cs:54. `EntityFrame` is a class holding the sprite, tic count, `ActionFunction?` delegate, `NextFrameIndex`, branch type, Dehacked misc/args (ints) and DECORATE `FrameArgs`. Three booleans are precomputed: `IsSlowTickChase/Look/Tracer` [H] Core/World/Entities/Definition/States/EntityFrame.cs:10-41,63-66.
- **Binding.** `ActionFunction` is `delegate void ActionFunction(Entity entity)` [H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:32. A static `Dictionary<string, ActionFunction>` with case-insensitive keys maps names to static methods [H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:36-397. DECORATE frames call `Find(frame.ActionFunction?.FunctionName)` once while the frame table is built [H] Core/World/Entities/Definition/Composer/DefinitionStateApplier.cs:191-193. Dehacked does the same [H] Core/Dehacked/DehackedApplier.cs:460-482.
- **Execution.** `SetFrameIndexInternal` loops: set frame, apply fast/slow-monster tic scaling, apply slow tick (§6), destroy on a null frame, `Frame.ActionFunction?.Invoke(entity)`, then follow zero-tic frames until a non-zero tic count, with a 10,000-iteration guard [H] Core/World/Entities/Definition/States/FrameState.cs:175-224.
- **Arguments.** Dehacked and MBF21 args are plain `int` fields on the frame (`DehackedArgs1..8`) read directly, e.g. `A_JumpIfHealthBelow` [H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:3310-3318. DECORATE args are stored as `IList<object>` with a parallel type array. `GetInt` unboxes them with `(int)Values[index]` for both int- and double-typed args [H] Core/World/Entities/Definition/States/FrameArgs.cs:18-53. Unboxing a boxed `double` as `int` throws `InvalidCastException` in C#, so a double-typed argument read through `GetInt` would throw at run time. Only 7 sites in EntityActionFunctions.cs read `Frame.Args` (`grep -c "Frame.Args"` = 7). The parser accepts names starting `A_` plus action specials and hands the argument tokens to `ConsumeActionFunctionArguments` [H] Core/Resources/Definitions/Decorate/Parser/DecorateStateParser.cs:246-262. There is no expression compiler or VM.
- **Coverage.** The dictionary has 359 entries (`grep -c '^        \["'`). 237 of those entries point at methods whose whole body is `// TODO` (the file has 239 `// TODO` comments in all), for example `A_CheckSight`, `A_LookEx`, `A_MonsterRail` [H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:813-880,1606-1665.

**Inference (cost vs a VM):** a Helion action call is one null check plus one delegate invoke on a static method. That is an indirect call costing a few nanoseconds, with no argument marshalling.

UZDoom's `FState::CallAction` does more work per call:
- clocks a cycle counter,
- builds a `VMValue` array (defaults and implicit `self`/`stateowner`/`info`),
- calls `VMCallAction` inside a `try`, which runs native thunks or JIT code [U] src/gamedata/info.cpp:148-200.

`DThinker::CallTick` also checks every tic for a ZScript `Tick` override [U] src/playsim/dthinker.cpp:1265-1271. With `vm_jit` on by default [U] src/common/scripting/vm/vmframe.cpp:39-49, the per-call overhead is perhaps 20–60 ns for native actions. Helion's model is cheaper per call but cannot express ZScript or DECORATE expressions at all. It is a lower bound on cost, not a design UZDoom can adopt without dropping ZScript.

---

## 6. Line of sight and monster AI

### CheckLineOfSight

[H] Core/World/WorldBase.cs:2759-2824
1. **Reject table.** The table is loaded only if present and non-empty [H] Core/World/WorldBase.cs:263-269. If loaded, a set bit means "no sight" [H] Core/World/WorldBase.cs:2899-2911, the same rule as UZDoom's `CheckReject` [U] src/g_levellocals.h:445-448.
2. **Early outs**: same XY position, then transfer-heights (deep water) blocks.
3. **Blockmap ray march** (`BlockmapTraverser.SightTraverse`), not a BSP walk. Blocks are visited by DDA. For each block's line slice the code computes `IntersectionInclusive` *before* checking the counter, so a line shared by several blocks is intersected more than once. Any one-sided or sight-blocking line aborts immediately. Otherwise the block's hits are **sorted by fraction** (`Array.Sort` over a struct `IComparable`) and handed to `TestSightBlock` [H] Core/World/Physics/Blockmap/BlockmapTraverser.cs:57-107.
4. `GetBlockmapTraversalPitch` narrows top and bottom slopes through each two-sided opening. It skips lines whose front and back floor and ceiling are equal, and handles 3D floors through visible-span lists [H] Core/World/WorldBase.cs:3528-3616.

UZDoom's `P_CheckSight` adds two checks Helion does not have: invisibility/`MVISBLOCKED` handling and portal groups. It then runs a dedicated `SightCheck` object with its own blockmap path traversal [U] src/playsim/p_sight.cpp:840-930, [U] src/playsim/p_sight.cpp:619. In both engines the cost is proportional to the blocks crossed. **Inference:** Helion's per-block sort and early abort on the first one-sided line make long clear sightlines cost about O(blocks × lines/block). An occluded sightline usually ends within the first few blocks, because one-sided walls end the march.

`BlockmapIntersect` is `{int Index; double SegTime}`. Its comment says "Packs to 8 bytes", but there is no `[StructLayout(Pack=…)]` [H] Core/World/Physics/Blockmap/BlockmapIntersect.cs:1-22, so it is 16 bytes.

### Target acquisition and A_Look / A_Chase

- `A_Look` → `SetNewTarget(false)`: if the sector `SoundTarget` (a `WeakEntity`) is valid, take it. Ambush monsters still need a sight check for this. Otherwise run `GetLineOfSightPlayer`, which loops over players with an FOV-or-melee test, then `CheckLineOfSight` [H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:1595-1604, [H] Core/World/Entities/Enemy.cs:45-103, [H] Core/World/WorldBase.cs:927-943. Single player is therefore at most one sight trace per A_Look call. Friendly monsters use `GetLineOfSightEnemy`: a ±1280-unit blockmap `EntityTraverse` with a sight trace per candidate until one is seen [H] Core/World/WorldBase.cs:957-983.
- `A_Chase` mostly follows vanilla structure: reaction and threshold countdown, turn, retarget, melee check (`InMeleeRange` traces sight only when within range), missile check (`CheckMissileRange` traces sight first), active sound, `MoveEnemy` → `TryMoveXY`, and on failure `SetNewChaseDirection`. That function can try up to 12 `TryWalk`s: the diagonal, two axis directions, the old direction, a 7–8 direction sweep and the reverse [H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:736-811, [H] Core/World/Entities/Enemy.cs:165-330,460-523. One divergence from vanilla: after entering the missile state, `A_Chase` does not return. It still decrements `MoveCount` and calls `MoveEnemy` in the same call [H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:787-810.
- Noise propagation: `NoiseAlert` → `RecursiveSound` floods sectors through two-sided openings. Each sector is tagged with `m_soundCount` and a block count, so no per-sector clear is needed [H] Core/World/WorldBase.cs:985-1028.

### Monster closets (default ON)

At load, `ClosetClassifier.Classify` checks each geometry island. An island is a connected set of subsectors, flood-filled through partner segs (two-sided lines and minisegs) [H] Core/World/Geometry/Islands/IslandClassifier.cs:106-155. So only areas with **no** two-sided connection to the rest of the map qualify, which in practice means teleport-fed closets. The classification runs after the player is spawned, so the player's own island can never qualify [H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:70-78,134-138. An island counts as a monster closet if:
- its lines have no specials other than teleports and plane scrollers,
- every entity in it is `CountKill`,
- at least one monster is present.

Every non-friendly monster in such an island gets `ClosetFlags.MonsterCloset` [H] Core/World/Geometry/Islands/ClosetClassifier.cs:30-122. The pass is gated by `config.Game.MonsterCloset` [H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:134-138, default `true` [H] Core/Util/Configs/Components/ConfigGame.cs:163-164.

When such a monster runs `A_Look`, and its spawn loop contains only `A_Look` (checked at load) [H] Core/World/Entities/Definition/Composer/DefinitionStateApplier.cs:123-139, it jumps to a synthetic invisible 16-tic frame running `A_ClosetLook`. That frame only polls `Sector.SoundTarget`: **no sight check, no FOV test** [H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:1595-1601,3406-3414, [H] Core/World/Entities/Definition/EntityFrameTable.cs:73-84. Once woken (a valid `SoundTarget` exists), the monster switches to a second invisible 16-tic frame running `A_ClosetChase`. That action has no melee or missile checks. If the target is alive it just calls `SetNewChaseDirection`. If the target is dead or missing it calls `SetNewTarget(true)`, which does do a player sight trace, and falls back to `A_ClosetLook` [H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:3416-3427. Each successful step moves `ClosetChaseSpeed` units, default 40 rather than the monster's own speed. After a fully failed direction search the step falls back to the monster's normal speed [H] Core/World/Entities/Enemy.cs:327-328,341-342,541, [H] Core/World/Entities/Entity.cs:47. Wake-up times are staggered across tics by global counters through `AddFrameTicks` [H] Core/World/Entities/Enemy.cs:105-132.

During play, closet mode is cleared only by teleporting: `EntityTeleported` calls `ClearMonsterCloset()` on the teleported entity, which restores its spawn or see state [H] Core/World/WorldBase.cs:4174-4176, [H] Core/World/Entities/Enemy.cs:134-146. When the *player* teleports into a closet island, the island's flags are cleared and the code then loops over all entities to wake that island's monsters. **That loop is inverted, so it is a no-op.** It `continue`s on every entity that *has* `ClosetFlags.MonsterCloset` and calls `ClearMonsterCloset()` only on entities without it [H] Core/World/WorldBase.cs:4203-4219. Every entity in closet mode carries the `MonsterCloset` bit, because `ClosetLook`/`ClosetChase` are only added when `ClosetFlags != None` [H] Core/World/Entities/Definition/States/FrameState.cs:161-173, [H] Core/World/Entities/ClosetFlags.cs:6-12. So every remaining entity in the loop has `ClosetFlags == None`, and `ClearMonsterCloset()` returns immediately. Code-traced, not run: a player who teleports into a closet finds its monsters still invisible, sound-only and attack-less until each one teleports out.

**Inference:** on slaughter maps with thousands of closeted monsters, this removes nearly all idle sight traces and drops their look frequency from every ~10 tics to every 16. It also hides them from rendering via the invisible sprite. It is the biggest AI-cost cut in the engine. The trade-off is a behaviour change. Closet monsters ignore sight, only respond to sound, never attack while in closet mode, and move in 40-unit steps every 16 tics.

### Slow tick (default OFF)

`CheckSlowTickDistance` runs on each state entry. If the new frame's action is a chase, look or tracer action, the actor is not in a closet and not a player, and either `RenderDistanceSquared > SlowTickDistance²` **or the actor was not drawn this gametick**, the frame's tic count is multiplied by `(multiplier + offset)`. The multiplier is the chase multiplier (default 4), look multiplier (2) or tracer multiplier (4). The offset rotates through −1, 0, +1, +2 per category, so the effective factor is 3–6 for chase and tracer and 1–4 for look (mean 4.5 and 2.5). The check also requires `CurrentTick > 0`, slow tick enabled and `Distance > 0` [H] Core/World/Entities/Definition/States/FrameState.cs:226-261, [H] Core/Util/Configs/Components/ConfigSlowTick.cs:12-34. "Chase" means `A_Chase`, `A_VileChase` or `A_HealChase`; "look" means `A_Look`; "tracer" means `A_Tracer` or `A_SeekTracer`. These flags are computed once in the `EntityFrame` constructor [H] Core/World/Entities/Definition/States/EntityFrame.cs:63-66. Dehacked later overwrites `entityFrame.ActionFunction` without recomputing them [H] Core/Dehacked/DehackedApplier.cs:460-482. A Dehacked pointer swap therefore leaves stale flags: a frame changed *to* `A_Chase` is never slow-ticked, and a frame changed *away* from it still is. The renderer writes `RenderDistanceSquared` and `LastRenderGametick` while drawing sprites [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:283-292.

Slow-ticked chasers compensate so that average behaviour stays roughly the same:
- step size is multiplied by `SlowTickMultiplier`, clamped to ±128 [H] Core/World/Entities/Enemy.cs:341-342,
- reaction and threshold counters are decremented by the multiplier [H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:738-749,
- the distance term in the missile-chance roll is divided by the multiplier, which makes firing more likely per call [H] Core/World/Entities/Enemy.cs:518-522,
- the exhaustive 8-direction search is replaced by up to 3 tries from a random start on 3 of 4 calls, and none on the 4th, using a global counter [H] Core/World/Entities/Enemy.cs:249-312,
- after a fully failed direction search, the next 3–4 `SetNewChaseDirection` calls return at once (`ChaseFailureSkipCount`, default 4, plus an alternating 0/1). This applies whenever slow tick is enabled, even to actors currently ticking at the normal rate [H] Core/World/Entities/Enemy.cs:165-170,324-325.

**Inference:** the simulation now reads renderer state (visibility, view distance), so a given demo's outcome depends on resolution, FOV and window. The cvars are tagged `demo: true` and stored in the demo, but visibility is not. Helion has to treat this as "non-vanilla, approximate", which is presumably why it is off by default.

---

## 7. Sound: per-tic costs

`WorldSoundManager.Tick` runs once per world tic [H] Core/World/WorldBase.cs:1056,1091. `WorldSoundManager.Update()` also forwards to `Tick()` [H] Core/World/Sound/WorldSoundManager.cs:234-237. However, the per-frame `m_soundManager.Update()` in the client [H] Client/Client.cs:402 is a separate base `SoundManager` constructed by the client for menu and UI sounds [H] Client/Client.cs:62,107, not the world manager. So world sound work is per tic only. On each run it:
- sets the listener,
- re-sorts waiting looped sounds by distance (cached `Comparison`) [H] Core/Audio/Sounds/SoundManager.cs:201-240,
- starts queued sounds as a group,
- walks every playing source. Finished sources are freed to the `DataCache` pool. A source that fails the distance cutoff is bumped back to waiting. Otherwise it gets `Update(distance)`, `SetPitch` and `SetPosition` (OpenAL calls) [H] Core/World/Sound/WorldSoundManager.cs:239-284.

`OpenALAudioSource` objects are pooled [H] Core/Util/DataCache.cs:181-209. `SoundParams`, `AudioData` and `WaitingSound` are structs [H] Core/Audio/SoundParams.cs:7, [H] Core/Audio/AudioData.cs:6, [H] Core/Audio/Sounds/WaitingSound.cs:7. **Inference:** the per-tic cost is O(playing sounds) driver calls, bounded by the `MaxSounds` cvar (default 32) [H] Core/Util/Configs/Components/ConfigAudio.cs:35. It is negligible next to physics unless there are hundreds of looping ambient sources.

---

## 8. Per-tic allocations and structures built for speed

Allocation evidence:
- **No LINQ or collection allocation in the tick paths.** `grep` for `new List<|new Dictionary<|.ToList()|.Where(|.Select(|.FirstOrDefault(` in WorldBase.cs, PhysicsManager.cs, Entity.cs, EntityActionFunctions.cs, SpecialManager.cs and Player.cs finds only load-time or cheat paths (music info, cheat armour, boss-brain targets, 3D-floor setup) [H] Core/World/WorldBase.cs:568-572,1491,4410-4447, [H] Core/World/Special/SpecialManager.cs:734-735.
- **Traverser callbacks are cached delegates** held in fields (`m_canPassTraverseFunc`, `m_lineOfSightEnemyAction`, `m_testSightBlockAction`, `m_radiusExplosionEntityAction`, `m_pushEntityAction` …) [H] Core/World/Physics/PhysicsManager.cs:82-84,525,561,1143, [H] Core/World/WorldBase.cs:963,2791,2946, [H] Core/World/Special/Specials/PushSpecial.cs:69.
- **Pools**: entities, sector-link nodes, special nodes, sector movers, light changes, switches, stairs, audio sources, entity scratch lists, waiting-sound nodes and wall vertex arrays [H] Core/Util/DataCache.cs:39-51,96-441, [H] Core/Util/Container/LinkableList.cs:135.
- **Scratch arrays are reused**: `WorldStatic.Intersections` (1024) [H] Core/World/WorldStatic.cs:16, the `TryMoveData` lists, and the `PhysicsManager` `DynamicArray`s [H] Core/World/Physics/PhysicsManager.cs:68-76.
- **GC mode**: during gameplay the client switches to `GCLatencyMode.SustainedLowLatency` [H] Core/Util/GCUtil.cs:8-10, [H] Client/Client.cs:529, [H] Core/Layer/GameLayerManager.cs:317. Native AOT publishing is supported with `OptimizationPreference=Speed` [H] Client/Client.csproj:22-24.

Speed structures, listed from code only:
1. Per-block contiguous `BlockLine[]` slices with inline segment and AABB.
2. Per-block `int[]` entity indices into the pool array.
3. A global check counter plus a dense `CheckedLines[]`.
4. The 16-unit BSP-node cache for point lookup.
5. A compact BSP node array with branch-free child select.
6. ID-validated weak references (no back-pointer cleanup).
7. Pre-classified slow-tick action flags on frames (pointer compares at load, bool tests at run).
8. Monster-closet classification.
9. A separate 512-unit render blockmap with intrusive lists, so sprite gathering does not touch the physics blockmap [H] Core/World/WorldBase.cs:429-439.
10. A BenchmarkDotNet suite covering exactly these paths: sight 2D/3D, hitscan, move 2D/3D, entity-on-entity collision, creation [H] BenchmarkSuite/EntityLineOfSight2D.cs:10-33.

**Threading.** `grep -rn "Parallel\.|Task\.Run|new Thread"` in Core/World finds only the automap-marker thread and save-game `Task.Run` [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:70, [H] Core/World/Save/SaveGameManager.cs:129. The playsim is single-threaded and leans on process-wide statics (`WorldStatic.*`) [H] Core/World/WorldStatic.cs:14-63.

---

## 9. Numeric precision and demo compatibility

- **Precision**: `double` throughout. This covers `Vec3D` positions and velocities, `AngleRadians`, `SectorPlane.Z`, `Seg2D` and BSP splits [H] Core/World/Entities/Entity.cs:66-68, [H] Core/World/Geometry/Sectors/SectorPlane.cs:15-16. `Alpha` is `float` [H] Core/World/Entities/Entity.cs:135.
- **Fixed-point emulation is selective**: vanilla constants are written as `n/65536.0` (e.g. `Speed = 47000/65536.0`) [H] Core/World/Entities/Entity.cs:44, [H] Core/World/Physics/PhysicsManager.cs:45-48. `LowestPossibleZ = Fixed.Lowest().ToDouble()` [H] Core/World/Physics/PhysicsManager.cs:51, plus `short.MinValue/MaxValue` overflow guards [H] Core/World/Physics/PhysicsManager.cs:1013-1025, and AngleRadians snapping to 45° for monster turning [H] Core/World/Entities/Enemy.cs:413-431. The RNG keeps the vanilla `DoomRandom` table and a `BoomRandom` [H] Core/Util/RandomGenerators/DoomRandom.cs:6, [H] Core/Util/RandomGenerators/BoomRandom.cs:5.
- **Demos**: Helion records its own format, a zip containing `info.json` and `demodata.lmp` [H] Core/Demo/DemoArchive.cs:21-22. Each tic's `DemoCommand` stores `double` angle, pitch and move speeds [H] Core/Demo/DemoCommand.cs:6-14, together with demo-relevant config values [H] Core/Demo/DemoArchive.cs:24-36. `grep` for vanilla demo markers in Core/Demo finds none. **Inference:** Helion cannot play vanilla, Boom or MBF .lmp demos bit-exactly, and does not try to. Its own demos depend on IEEE double determinism under one runtime: x64 RyuJIT or AOT, with no FMA contraction by the JIT, which is not verified here. They break if slow tick reads the renderer (§6). UZDoom also stores positions as `DVector3` [U] src/playsim/actor.h:1142, so the two engines are in the same class of non-vanilla demo compatibility.

---

## Absent / not done (proven by search)

- **No thinker culling or sleeping in the tic loop.** Every list member is visited every tic. Only `Dormant` skips `Entity.Tick`, and it still runs `Move` [H] Core/World/WorldBase.cs:1173-1206. `grep -rn "Sleep\|Inactive\|IsAwake" Core/World/Entities` finds only `PlayerCheats.SetCheatInactive`, so there is no tic-skip flag besides Dormant and slow tick.
- **No multithreading of the playsim** (see §8 grep).
- **No VM, no ZScript, no DECORATE expressions.** The only action-dispatch site is `Frame.ActionFunction?.Invoke(entity)` [H] Core/World/Entities/Definition/States/FrameState.cs:212. 237 of 359 action names map to empty TODO stubs.
- **No slopes** in physics [H] Core/World/Geometry/Sectors/Sector.cs:670.
- **BLOCKMAP lump not used** for collision (blockmap is rebuilt from lines at 128 units) [H] Core/World/WorldBase.cs:423, with no reader found under Core/Maps.
- **No BSP-based sight check.** Only blockmap ray marching (`SightTraverse` is the only sight traverser) [H] Core/World/Physics/Blockmap/BlockmapTraverser.cs:57.
- **No vanilla .lmp demo playback** (§9).
- **UZDoom side**: `grep -rni "closet" src/playsim src/maploader` and `grep -rni "slowtick|skiptick|tickdist" src/playsim src/p_tick.cpp` both return nothing, so UZDoom 5.0 has neither mechanism.

## Perf-relevant settings

| Setting | Default | Effect | Citation |
|---|---|---|---|
| `Game.MonsterCloset` | `true` (map restart, demo-tagged) | Closet classification → `A_ClosetLook`/`A_ClosetChase` | [H] Core/Util/Configs/Components/ConfigGame.cs:163-164 |
| `SlowTick.Enabled` | `false` | Master switch for distance and visibility tic multiplier | [H] Core/Util/Configs/Components/ConfigSlowTick.cs:12-14 |
| `SlowTick.Distance` | 2000 | Threshold (map units) | [H] Core/Util/Configs/Components/ConfigSlowTick.cs:16-18 |
| `SlowTick.ChaseFailureSkipCount` | 4 | Skip direction search after failures | [H] Core/Util/Configs/Components/ConfigSlowTick.cs:20-22 |
| `SlowTick.ChaseMultiplier` / `LookMultiplier` / `TracerMultiplier` | 4 / 2 / 4 | Tic multipliers | [H] Core/Util/Configs/Components/ConfigSlowTick.cs:24-34 |
| `Game.AutoAim` | `true` | Crosshair uses the autoaim trace every tic | [H] Core/Util/Configs/Components/ConfigGame.cs:19-21, [H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:271-274 |
| `Compatibility.InfinitelyTallThings` | False | Skips entity-on-entity Z work and fall checks | [H] Core/Util/Configs/Components/ConfigCompat.cs:38-40, [H] Core/World/WorldBase.cs:1191, [H] Core/World/Physics/PhysicsManager.cs:1105-1123 |
| `Compatibility.MissileClip` | False | Missile Z overlap rule | [H] Core/Util/Configs/Components/ConfigCompat.cs:42-44 |
| `PhysicsManager.EnableMaxMoveXY` | true (field) | ±30 velocity clamp | [H] Core/World/Physics/PhysicsManager.cs:55 |
| GC latency | `SustainedLowLatency` in game | Fewer blocking gen2 GCs | [H] Core/Util/GCUtil.cs:10 |

## Transfer notes for UZDoom VR

General point: the playsim runs at 35 Hz independent of the 90–120 Hz render. In VR it matters as **spike risk**: a heavy tic landing inside one 8–11 ms frame budget causes a drop. It does not matter as average cost. Stereo itself does not change playsim work, except for anything that reads renderer visibility.

1. **Monster-closet classification**
   - UZDoom counterpart: none (search above). Monsters run `A_Look` → `P_LookForPlayers` [U] src/playsim/p_enemy.cpp:1757,1912.
   - Grade: **Port.** The island classifier and the A_Look swap are load-time and self-contained.
   - ZScript caveat: only apply it when the actor's spawn loop is pure `A_Look` *and* the class has no ZScript `Tick` override (`IFVIRTUAL` check, [U] src/playsim/dthinker.cpp:1267).
   - Stereo/Vulkan: none.
   - Port the wake-up-on-player-teleport with the condition the right way round (Helion's loop is inverted, §6).
   - Highest value per line of code for slaughter maps.

2. **Slow tick (distance/visibility tic multiplier)**
   - UZDoom counterpart: none.
   - Grade: **Port** as opt-in. In VR, define visibility as the union of both eyes, or better, as distance-only, so it is independent of headset FOV and resolution.
   - Demos and netplay then need the setting recorded, or it must be forced off there.
   - Recompute the per-state slow-tick class after Dehacked pointer patching. Helion computes it before Dehacked runs (§6).

3. **Per-block `int[]` entity arrays (replacing FBlockNode lists)**
   - UZDoom counterpart: `FBlockNode` linked lists walked by `FBlockThingsIterator`, which also carries a hash for dedupe [U] src/playsim/p_maputl.h:313-345, [U] src/playsim/actor.h:1197.
   - Grade: **Rework.** Link/unlink and all iterators change, but the iterator interfaces can survive.
   - Requires a stable small-integer actor index. That needs an actor slot table, because UZDoom actors are GC'd DObjects.

4. **Denormalised per-block `BlockLine` arrays**
   - UZDoom counterpart: blockmap offsets → line index lists → `line_t` (`FBlockLinesIterator` [U] src/playsim/p_maputl.h:241).
   - Grade: **Rework** (`FBlockLinesIterator` internals plus build at load).
   - Portals, polyobjects and moving lines (polyobjects move) need the cached segment updated. Helion has no polyobjects, so it never had to solve that.

5. **16-unit BSP point-lookup cache**
   - UZDoom counterpart: `PointInSubsector` tree walk.
   - Grade: **Port.** Rebuild on polyobject or map change only; memory is ~4 MB on large maps.

6. **Global check counter + dense `CheckedLines[]`**
   - UZDoom already has `validcount` [U] src/playsim/p_map.cpp:1845,1930.
   - Grade: **Port** (optional). It moves marks out of `line_t` into a dense array for cache locality. Small win.

7. **Entity pooling + ID-validated weak references**
   - UZDoom counterpart: GC-managed `AActor` with `TObjPtr` nulling.
   - Grade: **Rebuild** if pursued, since it touches DObject lifetime and ZScript references. Low value in C++ with a tuned allocator. Prefer arena-allocating actors over pooling semantics.

8. **Delegate action dispatch**
   - UZDoom counterpart: `FState::CallAction` → `VMCallAction` [U] src/gamedata/info.cpp:148-200.
   - Grade: **Rework** of a fast path only: call native `DEFINE_ACTION_FUNCTION`s directly when the state has no default args and no ZScript override. Dropping the VM entirely is a Rebuild and loses ZScript.

9. **Sight checks**
   - UZDoom `P_CheckSight` already combines reject and blockmap traversal [U] src/playsim/p_sight.cpp:840-930. Helion offers no algorithmic advantage beyond aborting at the first one-sided line.
   - Grade: keep UZDoom's.

10. **Sector movers → renderer**
    - Helion pairs polled `PrevZ/LastRenderChangeGametick` with synchronous move start/complete events and a per-frame dirty-sector set for lights. UZDoom counterpart was not examined here (see the renderer sections).
    - Grade: **Port** the dirty-set pattern (`SectorUpdates`) for a Vulkan per-sector SSBO, written once per frame and shared by both eyes.

11. **Single-threaded statics**
    - Helion's `WorldStatic` globals and single `TryMoveData` bake in single-threading. UZDoom is also single-threaded in `P_Ticker` [U] src/p_tick.cpp:168-275.
    - Grade: running the tic on a worker thread overlapped with render is a **Rebuild** in either engine: every renderer read of actor and sector state needs snapshotting. It is the only way to remove tic spikes from VR frame time completely.

## Open questions

- No runtime measurements were taken. `dotnet` is not installed here, so the BenchmarkSuite was not run, and per-entity byte sizes are estimates from field counts.
- The `TickEntities` successor-destroy bug (§1) and the inverted closet wake-up loop (§6) are confirmed by code trace only. Neither was reproduced in a running build. Whether real maps commonly put an item last in THINGS order, which is what exposes `player.Next`, was not surveyed.
- The OpenAL per-call cost was not measured.
- Whether .NET JIT or AOT can emit fused multiply-add for the `double` physics (which would make demos platform-dependent) was not verified.
- The UZDoom `AActor` size and the cost of `AActor::Tick` for idle actors were not measured; that belongs in the UZDoom playsim section.
