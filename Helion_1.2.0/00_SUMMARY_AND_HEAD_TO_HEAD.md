# 00: Summary, architecture and head-to-head (Helion 1.2.0 vs UZDoom 5.0)

The repo copy of the sections written directly in the shared Claude Doc "Helion Teardown & UZDoom VR Performance
Rebuild" (https://claude.ai/artifact/Uo9TTtxY2M9yPhMSHihiy9). Pins: Helion `5edd437`, UZDoom tag `5.0.0` (`292cf42`).
`[H] path:lines` = Helion, `[U] path:lines` = UZDoom 5.0, `[M] §n` = `../UZDXREMA_0.1.0/10_MEASUREMENTS.md`. Cost
reasoning is labelled Inference. Nothing was compiled or run.

## Executive summary

Helion is fast because it does almost no rendering work per frame: it rebuilds the scene at most once per 35 Hz game
tic, keeps the static level permanently on the GPU, and lets shaders interpolate between tics. UZDoom 5.0 does the
opposite, rebuilding the entire scene every frame, once per eye, and once more per portal. Every one of Helion's
techniques can be carried into UZDoom, and a Vulkan/OpenXR rebuild can go past Helion, because Helion itself is a
single-threaded OpenGL 3.3-level engine with real gaps for VR.

### Why Helion screams, ranked by impact

| # | Helion technique | Evidence | UZDoom 5.0 today | Cost to bring over |
|---|---|---|---|---|
| 1 | **Scene built per tic, drawn per frame.** Visibility, dynamic vertices and sprite instances are rebuilt only when the tic counter changes; in-between frames redraw the same buffers and the vertex shader blends previous and current positions with a tick fraction. | [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:143, 310-341 | Full BSP walk, wall build, light lists and draw lists every frame, inside a per-eye loop. | Rework |
| 2 | **GPU-resident static world.** All non-moving walls and flats are built at map load into one vertex buffer per geometry type and texture array, then drawn whole every frame with no CPU culling. | 01 | Walls rebuilt every frame; only flats are cached and patched. | Rebuild |
| 3 | **Lighting by indirection.** Each vertex carries a sector slot that indexes small light, colour and fog tables; a flickering light writes a few bytes and touches no geometry. | 01 | Light baked into per-draw state; per-surface dynamic light lists rebuilt per frame. | Rework |
| 4 | **Instanced sprites with no CPU sort.** Sprite instances are built per tic; quads are expanded from the vertex index in the shader; translucency uses weighted-blended OIT. | 03 | Each sprite processed per frame; every non-model, non-wall sprite goes through the sorted translucent list. | Rework |
| 5 | **Few draws, little state.** A handful of programs and one draw per texture group. | 01 | One draw per wall tier, flat section, sprite and decal; each Vulkan draw copies a full per-draw record and rebinds material descriptors. | Rebuild (backend) |
| 6 | **Cheap playsim.** No VM; monsters in teleport-only closets idle on a cheap 16-tic look; a 16-unit grid jumps point lookups into the BSP; pooled entities and inline blockmap line data. | 04 | Every actor ticks fully every tic through the VM call path; no dormancy; BSP point lookups from the root. | Port to Rework |
| 7 | **Feature budget.** No portals, slopes, dynamic lights, models, voxels or ZScript, so none of their per-frame cost exists. | 06 | All present. Portals multiply the whole scene build. | Rebuild, keeping the features |

### What the numbers mean for VR

At 90 Hz stereo, UZDoom 5.0 builds the scene 2 × 90 = 180 times per second, before portals. Helion's approach builds it
35 times per second regardless of refresh rate or eye count: about 5.1× fewer builds at 90 Hz and 6.9× fewer at 120 Hz
(simple arithmetic, not a measurement). The GPU side goes the other way: Helion's static world draws the whole map
every frame, so GPU vertex cost scales with map size and doubles with two eyes unless you use multiview and GPU culling.

### What the owner's own measurements change

With the owner's mod set loaded, Helion's rendering ideas are not what saves the frame: the renderer is about 2% of a
315 ms frame and the simulation about 80% ([M] §1, Sunder MAP15, RTX 3080 Ti, flat screen).

| Sunder MAP15 | Frame | Sim (think) | Scene build | GPU wait | Thinkers |
|---|---|---|---|---|---|
| No mods | 13.9 ms | 2.5 ms | 3.8 ms | 3.2 ms | 8,065 |
| Owner's 57-file mod set | 315 ms | 250 ms | 4.4 ms | 25.9 ms | 53,116 |

- **Simulation first for the real workload.** Thinkers take 131 ms of a 28.6 ms tic, and helper actors dominate: one
  ZScript HP-bar actor per monster (3,622 of them, 4.5 µs each, more than the monster itself), champion controllers and
  icons, and crit helpers. The Helion lessons that apply are its cheapest ones: no VM in the tick path, idle monsters
  that barely think, and pooled entities. Going past Helion means engine-side ways for mods to stop spawning helper
  actors at all.
- **Rendering still matters without mods.** The no-mod frame is 13.9 ms on a flat screen, already over a 90 Hz budget
  (11.1 ms) before a second eye. The per-eye rebuild, the end-of-frame fence wait and the per-draw state cost are what
  stereo would double.

### Where Helion is weak, and where you go past it

- **Head turns outrun tic-rate culling.** Helion's visibility is computed at tic time with only a near-plane test, so a
  fast VR head turn can reveal unbuilt geometry. The fix is conservative visibility per tic plus GPU culling per frame.
- **One thread, OpenGL 3.3-level features.** No multi-draw, indirect, bindless, fences or multiview.
- **Single-camera tricks.** Sky, missing-texture flood fill and sprite facing assume one camera and need per-view
  versions under multiview.
- **No frames in flight in UZDoom's Vulkan backend.** It waits on every fence at the end of each frame, so the CPU and
  GPU never overlap. Fixing this comes first on the render side.

### The plan, in the order the measurements call for

1. **Measure and fix the cheap things (Phase 0).** Tracy zones, per-class and per-virtual thinker profiling, a JIT check
   at startup, replacing the `r_ticstability` busy-wait, and the one-line fixes the audit found (`OldRenderPos`,
   `CallAction` copies).
2. **Cut engine overhead per thinker (Phase 9, first steps).** A native `Tick` fast path that skips the VM when ZScript
   doesn't override it, dormancy for idle monsters (Helion's closet idea, generalised), and a cheaper interpolation
   reset (9 ms per tic in the owner's bench).
3. **Stop mods needing helper actors (Phase 10).** Native per-actor overlays, attachments, tickless actors, throttled
   tick lists, per-class actor lists for batch event handlers.
4. **In parallel, fix the Vulkan backend (Phase 1).** Frames in flight, per-draw data in a storage buffer, bindless
   textures, pipeline precompile, async uploads.
5. **Add OpenXR and multiview (Phase 2).** One scene build feeds both eyes.
6. **Finish the playsim work (rest of Phase 9).** BSP lookup grid, inline blockmap lines, sight caching, portal
   acceleration, actor arenas, GC pacing.
7. **Build the scene per tic and interpolate on the GPU (Phase 3), move sprites and overlays to instanced paths
   (Phase 7), then put the sim on its own thread (Phase 8).**
8. **Make the world GPU-resident (Phase 4), then cull and draw portals on the GPU (Phase 5), then move dynamic lights
   to clustered GPU culling (Phase 6).**

Every new path keeps a cvar fallback to the stock behaviour. The fork-specific version of this plan, with the owner's
decisions applied, is `../UZDXREMA_0.1.0/30_GAMEPLAN_90HZ.md` and `31_FORK_DELTA_CHECKLIST.md`.

## Architecture map: one frame in each engine

The two engines spend a frame in opposite ways: Helion's frame is mostly drawing, while UZDoom's frame is mostly
rebuilding what to draw, repeated per eye and per portal.

```
Helion, one frame                                UZDoom 5.0, one frame
─────────────────                                ────────────────────
Poll input, run at most one tic                  TryRunTics: run every due tic (catches up inside the frame)
  (35 Hz; a hitch drops tics, no catch-up)        ┌─ repeated per eye (2 in stereo) ───────────────────────┐
Rebuild scene ONLY if the tic changed            │ BSP walk + clipper on the main thread;                  │
  (visibility walk, dynamic buffers, sprites)    │   walls/flats/sprites on 1 worker                       │
Draw the static world                            │ Render hacks, per-surface dynamic light copies, sort     │
  (one draw per texture group, no culling)       │ Each portal re-runs the scene (BSP walk from the root)   │
Draw dynamic geometry and sprites                │ One draw per surface, full per-draw state apply          │
  (shader blends previous→current by tick)       │ Post-process (full chain)                                │
Translucents via weighted-blended OIT (no sort)  └──────────────────────────────────────────────────────────┘
Swap, frame limiter, sound (all on one thread)   Present, then wait on all fences (CPU and GPU never overlap)
```

**Helion.** `Window_MainLoop` polls the controller, runs logic, renders, then updates sound, all on one thread
([H] Client/Client.cs:387-404). `LegacyWorldRenderer.PerformRender` decides each frame whether the tic changed; only
then does it walk the render blockmap (static mode) or the BSP (BSP mode) and refill dynamic buffers
([H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:310-341). Every frame it then draws skies, the
interpolated dynamic walls and flats, the static walls and flats, two-sided middles, opaque sprites and translucents
([H] same file :343-378).

**UZDoom 5.0.** The stereo loop in `RenderViewpoint` runs once per eye. Each pass starts a fresh `HWDrawInfo`, shifts
the view, calls `ProcessScene` (the full BSP walk and draw-list build) and runs post-processing
([U] src/rendering/hwrenderer/hw_entrypoint.cpp:126-187). Portals call back into the same scene build. The Vulkan
backend then waits on every submit fence before the next frame (U2).

**Codebase scale.** Helion is about 196k lines of C# across 1,526 files; UZDoom 5.0's `src/` holds about 569k lines of
C and C++ (`wc -l` at the pinned commits).

## Head-to-head: the same job in both engines

On every rendering job, Helion moves work from per-frame to per-tic or per-map-load; on most playsim jobs, it removes a
layer UZDoom has (the VM call path, the root BSP walk, per-call allocations). Full evidence is in sections 01-06 and
U1-U3.

### Rendering

| Job | Helion | UZDoom 5.0 | Grade | VR consequence |
|---|---|---|---|---|
| Frame vs tic | At most one tic per frame; if 2+ are due, the extra tics and the leftover fraction are dropped ([H] Core/Util/Timing/Ticker.cs:81-104). | `TryRunTics` runs every due tic inside the frame ([U] src/d_net.cpp:2190). | Rework | Neither is right for VR: a catch-up burst stalls a frame, a dropped tic slows the game. Put the sim on its own thread. |
| Scene rebuild rate | Only when the tic counter changes ([H] LegacyWorldRenderer.cs:143). | Every frame, once per eye ([U] src/rendering/hwrenderer/hw_entrypoint.cpp:126-187). | Rework | 35 builds/s instead of 180-240. Needs conservative visibility so head turns between tics don't reveal holes. |
| Visibility | Static mode: 512-unit render blockmap inside `MaxDistance` (default 0 → 6,000) plus a behind-camera test; static world not culled at all. BSP mode: two 360° angle clippers and the near plane. | BSP walk with a horizontal angle clipper; behind-view culling switches off above 46° of pitch ([U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:388). | Rebuild | Headsets pitch past 46° constantly. Replace with GPU frustum and Hi-Z culling of a resident world. |
| Static walls and flats | Built once at load into one vertex buffer per geometry type and texture array, drawn whole each frame. | Walls rebuilt per frame; flats cached in a vertex buffer and patched when a plane moves. | Rebuild | Removes the largest per-frame CPU cost; GPU cost then needs culling and multiview. |
| Moving sectors | On move start the sector leaves the static buffer and is drawn dynamically; on move end it is written back in place. | Flat vertices patched on movement; walls are always per frame anyway. | Port | Direct fit for a Vulkan resident world with dirty ranges. |
| Sector light, colour, fog | Per-vertex sector slot indexes small light/colour/fog tables; a flicker writes a few bytes. | Light level and colour go into per-draw state; dynamic lights become per-surface light lists rebuilt each frame. | Rework | Light changes stop touching geometry; dynamic lights move to clustered GPU culling. |
| Sprites | 40-byte instances built per tic; quads expanded from `gl_VertexID`; `glDrawArraysInstanced` ([H] Core/Render/OpenGL/Renderers/Legacy/World/Entities/EntityProgram.cs:150-153). | `HWSprite::Process` per sprite per frame per eye; every non-model, non-wall sprite goes to the translucent list ([U] src/rendering/hwrenderer/scene/hw_drawlistadd.cpp:142-145). | Rework | Billboard per view in the shader so each eye faces the sprite correctly. |
| Translucency | Weighted-blended OIT, no CPU sort. | Sorted translucent list. | Port (optional) | OIT is order-independent across eyes; keep sorted mode as a compatibility option. |
| Draw submission | One draw per texture group; a few shader programs; GL 3.3-level features only. | One draw per surface; each Vulkan draw writes a 384-byte `StreamData` record and `SetMaterial` always rebinds. | Rebuild | Per-draw data into a storage buffer indexed by draw id; bindless textures; multi-draw-indirect. |
| CPU/GPU overlap | GL driver queues frames. | Vulkan waits on every submit fence at end of frame ([U] src/common/rendering/vulkan/system/vk_commandbuffer.cpp:163-188). | Rework | Two to three frames in flight is the first Vulkan fix. |
| Stereo | None. | Full scene per eye; on Vulkan `NextEye` is empty, so only the last eye survives. | Rebuild | OpenXR swapchain plus multiview. |
| Portals | None (missing-texture fill only). | Each scene portal re-runs the whole scene build ([U] src/rendering/hwrenderer/scene/hw_portal.h:178-186). | Rebuild, keeping portals | Draw portal views from the same resident world with a second view and stencil. |
| Threading | One thread. | Main thread walks the BSP; one worker builds walls, flats and sprites. | Beyond Helion | Vulkan lets command recording and culling go wide. |

### Playsim

| Job | Helion | UZDoom 5.0 | Grade | VR consequence |
|---|---|---|---|---|
| Idle monsters | Monsters in teleport-only closets switch to a 16-tic `A_ClosetLook` with no sight trace ([H] Core/World/Entities/Definition/States/EntityActionFunctions.cs:1599, 3406-3426). | Every actor ticks fully every tic; no dormancy of any kind. | Port | Big win on slaughter maps; guard actors whose ZScript overrides `Tick`. |
| Actor tick dispatch | Delegate call per action; no VM. | Every tick goes through the VM call path, even to native `AActor::Tick`. | Rework | Native fast path when `Tick` is not overridden. |
| Action-function calls | Direct delegate. | `CallAction` copies the argument-type array by value on every call, and the default args too ([U] src/gamedata/info.cpp:135, 179). | Port | `const auto&` fixes; 1-2 mallocs saved per call. |
| Point to subsector | 16-unit grid jumps straight to a BSP start node ([H] Core/World/WorldBase.BspBlockMap.cs:36-49). | Walks from the root every time. | Port | Cheaper relinks and spawns. |
| Render-sector relink | Not applicable. | Meant to skip unmoved actors, but `OldRenderPos` is never updated after spawn, so the skip never fires ([U] src/playsim/p_secnodes.cpp:378). | Port | Update it after a rebuild and reset it in `UnlinkFromWorld` (plus the portal-plane flags in the key; see 20, F2). |
| Allocation | Entities, sector links, movers and audio sources pooled; no per-tic allocations found. | One `malloc` per spawn; incremental DObject GC paced by allocation rate. | Rework | Actor arenas and pooled thinkers. |
| Map load | Rebuilds every map's nodes with zdbspSharp into a temp WAD. | Uses supplied nodes or builds them. | Do not port | Helion is slower here. |

## Bugs found while reading (not reproduced in a running build)

**Helion**
- **Entity tick skip.** `TickEntities` caches `entity.Next` before ticking; if an entity destroys its successor, the
  loop lands on a node whose `Next` was nulled and stops, so every later entity misses that tic (04).
- **True-colour translations off at HEAD.** Commit `f543542d9` made `StoreImageIndices` always true, so in true-colour
  mode player colours, blood colours and Dehacked colormaps are no longer applied (03).
- **Data race.** The `AutomapMarker` background thread ORs into `Line.DataChanges` and `StructLine.Flags` without a
  lock while the main thread writes both (05).
- **Slow-tick flags go stale after Dehacked** pointer swaps (04).
- **Persistent-mapped texture buffers** are mapped by element count rather than bytes and have no coherent bit,
  barrier or fence (01).

**UZDoom 5.0**
- `OldRenderPos` is never updated after spawn, so the render-sector relink skip never fires (U3).
- `FState::CallAction` heap-copies the argument-type array on every call, plus the default args when present (U3).
- Vulkan stereo shows only the last eye: `FirstEye`/`NextEye` have no Vulkan implementation (U2).
- Opaque sprites go through the sorted translucent list (U1, performance).
- Behind-view culling switches off above 46° of pitch (U1, performance).

## Verification totals

About 2,300 file:line references were re-opened by a second agent that did not write the section; about 170 were
corrected and 3 removed. The playbook files (20-22) were then reviewed adversarially (about 160 more changes). Every
correction is logged in `verification/`.
