# UZDXREMA 0.1.0 teardown

Engine: UZDXREMA (UZDoom 5.0 base, DoomXR VR, Vulkan-only, OpenXR), repo `presidentkoopa/UZDXREMA`.
Read at: `main` f182cbcc (engine of record), `renderer` 175063a, `netplay` 1c1805f8, `selaco` 7e111a3d.
Rules: only the code is evidence; every claim is `file:line` and marked VERIFIED (read) or INFERRED (deduced).

Status: section drafts. Each section is still to be cross-checked against the code by a second reader;
the summary `00_UZDXR_ENGINE_TEARDOWN.md` and the measured numbers (`10_MEASUREMENTS.md`) come after.

| Section | Area |
|---|---|
| [01](01_FRAME_LOOP_TIMING_THREADS.md) | Frame loop, tic timing, threads, every place the main thread waits |
| [02](02_PLAYSIM_SCRIPTING.md) | Playsim and ZScript/ACS cost per tic; what blocks a parallel sim |
| [03](03_SCENE_BUILD_BSP_WALLS_FLATS.md) | CPU scene build: BSP walk, clipper, walls, flats, portals, sky; why the walk is needed |
| [04](04_SPRITES_MODELS_PARTICLES_WEAPONS.md) | Sprites, models, particles, decals, weapons, HUD; finding actors without the walk |
| [05](05_LIGHTS_SHADOWS_FOG_POSTPROCESS.md) | Dynamic lights, shadows, fog, post-process chain |
| [06](06_VULKAN_BACKEND_SUBMISSION.md) | Vulkan backend and draw submission |
| [07](07_VR_OPENXR_STEREO.md) | VR path: OpenXR, stereo, per-eye cost |
| 08 | Fork additions inventory, every cvar and hook, dead code |
| [09](09_BRANCHES_AND_STATIC_LEVEL_PATH.md) | The four branches; the render lane's static level path |
