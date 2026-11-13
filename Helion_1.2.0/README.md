# Helion 1.2.0 teardown, and the Helion pathway for a UZDoom 5.0 VR engine

Source: `https://github.com/Helion-Engine/Helion`, `Directory.Build.props` AssemblyVersion 1.2.0.0, commit `5edd437f39ddcbcd647a305d7bdd306847d6686b` (2026-11-10).
Compared against: `https://github.com/UZDoom/UZDoom` tag `5.0.0`, commit `292cf4203ebd3ced951cb67f6819180f588c1d44` (2026-08-28).
Rules: only the code is evidence. `[H] path:lines` = Helion, `[U] path:lines` = UZDoom 5.0, `[M] §n` = the owner's measurements in
[`../UZDXREMA_0.1.0/10_MEASUREMENTS.md`](../UZDXREMA_0.1.0/10_MEASUREMENTS.md). Cost reasoning is labelled **Inference**; facts about
Vulkan/OpenXR/Tracy are labelled **API fact**; code sketches are marked *(sketch)*. Nothing was compiled or run.

Every section was written by one reader and cross-checked line by line by a second reader who did not write it; the playbook files
were then reviewed adversarially by a third. Correction logs (old → new, why) are in [`verification/`](verification/).
The same material, with clickable citations, is in the shared Claude Doc "Helion Teardown & UZDoom VR Performance Rebuild".

## Why Helion is fast, in one paragraph

Helion does almost no rendering work per frame. It rebuilds visibility, dynamic vertices and sprite instances only when the 35 Hz tic
counter changes ([H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:143, 310-341), keeps the whole static level on the
GPU in one vertex buffer per geometry type and texture array and draws it uncull'd every frame, looks lighting up through per-sector slots
instead of baking it into geometry, expands sprite quads from `gl_VertexID` with instancing, and blends translucency with weighted-blended
OIT so nothing is sorted. In the playsim it has no VM, parks monsters in teleport-only closets on a cheap 16-tic look, jumps point
lookups into the BSP through a 16-unit grid, and pools its entities. UZDoom 5.0 rebuilds the whole scene every frame, once per eye and
once more per portal ([U] src/rendering/hwrenderer/hw_entrypoint.cpp:126-187), and runs every actor's tick through the VM call path.

## What the owner's measurements change

With the owner's 57-file mod set on Sunder MAP15 the renderer is about 2% of a 315 ms frame and the simulation about 80% [M] §1;
thinkers take 130.8 ms of a 28.6 ms tic, dominated by per-monster helper actors [M] §2-3. So the playbook leads with the simulation
track (Phases 0, 9, 10) and runs the render track (Phases 1-8) in parallel, because the no-mod flat frame (13.9 ms) is already over the
90 Hz budget before a second eye.

## Files

| File | What it answers |
|---|---|
| [01_WORLD_GEOMETRY_PIPELINE.md](01_WORLD_GEOMETRY_PIPELINE.md) | Static cache, GPU buffers, vertex formats, sector light tables, incremental mover updates, draw structure, shaders |
| [02_VISIBILITY_CULLING_PORTALS_SKY.md](02_VISIBILITY_CULLING_PORTALS_SKY.md) | Tic-rate culling, render blockmap, BSP mode and angle clippers, adaptive mode, flood fill, sky, 3D floors |
| [03_SPRITES_TEXTURES_TRANSLUCENCY_HUD.md](03_SPRITES_TEXTURES_TRANSLUCENCY_HUD.md) | Instanced sprites, texture arrays, palette mode, WBOIT, fuzz, HUD batching, models (none) |
| [04_PLAYSIM.md](04_PLAYSIM.md) | Tick loop, entity layout and pools, blockmap, movement, action dispatch, monster closets, slow tick, sound |
| [05_RUNTIME_MEMORY_TIMING_THREADS.md](05_RUNTIME_MEMORY_TIMING_THREADS.md) | Main loop, ticker, input split, frame limiter, GC discipline, data-oriented patterns, threads, build flags, BenchmarkSuite |
| [06_MAPLOAD_PRECOMPUTE_FEATURES.md](06_MAPLOAD_PRECOMPUTE_FEATURES.md) | Map load and node building, islands, BSP grid, and the full feature-coverage table vs UZDoom 5.0 |
| [UZDoom_5.0_counterparts/U1_HW_SCENE_SETUP.md](UZDoom_5.0_counterparts/U1_HW_SCENE_SETUP.md) | UZDoom 5.0 scene build: BSP walk, clipper, walls, flats, sprites, portals, draw lists, threading, stereo loop |
| [UZDoom_5.0_counterparts/U2_VULKAN_BACKEND_STEREO.md](UZDoom_5.0_counterparts/U2_VULKAN_BACKEND_STEREO.md) | UZDoom 5.0 Vulkan backend: per-draw state, buffers, frames in flight, pipelines, post-process, stereo modes |
| [UZDoom_5.0_counterparts/U3_PLAYSIM_THINKERS_VM.md](UZDoom_5.0_counterparts/U3_PLAYSIM_THINKERS_VM.md) | UZDoom 5.0 playsim: thinker lists, AActor::Tick, VM call path, JIT, blockmap, sight, GC, particles |
| [20_REBUILD_PLAYBOOK.md](20_REBUILD_PLAYBOOK.md) | The instruction set: Phases 0-10 with files, functions, steps, sketches, risks, metrics, effort and order |
| [21_VR_VULKAN_SPECIFICS.md](21_VR_VULKAN_SPECIFICS.md) | OpenXR integration, multiview, per-view items, budgets, frame pacing, foveation, post-process at VR resolution |
| [22_MEASUREMENT_PLAN.md](22_MEASUREMENT_PLAN.md) | Benchmark scenarios, metrics, Tracy zones, harness changes, determinism gates, acceptance targets per phase |
| [verification/](verification/) | Every correction made by the cross-check and review passes |

## Bugs found while reading (not reproduced in a running build)

Helion: entity-list tick skip when an entity destroys its successor (04 §1); true-colour translations disabled by commit `f543542d9`
(03); unlocked writes from the `AutomapMarker` thread (05); slow-tick flags stale after Dehacked pointer swaps (04); persistent-mapped
texture buffers mapped by element count with no coherent bit or fence (01).
UZDoom 5.0: `OldRenderPos` never updated after spawn, so the render-sector relink skip never fires (U3); `FState::CallAction` heap-copies
argument types and default args on every call (U3); Vulkan stereo shows only the last eye (U2); behind-view culling off above 46°
pitch (U1); opaque sprites routed through the sorted translucent list (U1).
