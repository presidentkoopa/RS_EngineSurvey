# UZDXREMA 0.1.0 teardown

Engine: UZDXREMA (UZDoom 5.0 base, DoomXR VR, Vulkan-only, OpenXR), repo `presidentkoopa/UZDXREMA`.
Read at: `main` f182cbcc (engine of record), `renderer` 175063a, `netplay` 1c1805f8, `selaco` 7e111a3d.
Rules: only the code is evidence; every claim is `file:line` and marked VERIFIED (read) or INFERRED (deduced).

Status (paused by the owner, 2026-11-13): all nine section drafts are here. Not yet done, in order:
1. a cross-check of each section against the code by a second reader who did not write it (fix or flag every claim that does not hold);
2. `10_MEASUREMENTS.md`: the runs already made on the owner's PC (old path, flat screen, `doomxr_rend1113a.exe`):
   Sunder MAP15 no mods 66.7 fps / 13.9 ms (sim 2.5, script 3.1, scene build 3.8, GPU wait 3.2);
   Sunder MAP15 owner's mod set 3.4 fps / ~300 ms (sim ~250 ms, 53,116 thinkers);
   Abandon MAP13 owner's mod set: thinkers 61-87 ms a tic. Raw CSVs: `E:\DOOMWork\_reports\render_bench\teardown_*`.
   Then the A/B runs the sections ask for (first: `r_ticstability 0/1`, polled vs blocking fence waits, `vr_openxr_multiview`, `vr_scene_multithread`);
3. `00_UZDXR_ENGINE_TEARDOWN.md`: the summary reference built from 01-10;
4. the rebuild instruction set, from this teardown and the Helion breakdown.

| Section | Area |
|---|---|
| [01](01_FRAME_LOOP_TIMING_THREADS.md) | Frame loop, tic timing, threads, every place the main thread waits |
| [02](02_PLAYSIM_SCRIPTING.md) | Playsim and ZScript/ACS cost per tic; what blocks a parallel sim |
| [03](03_SCENE_BUILD_BSP_WALLS_FLATS.md) | CPU scene build: BSP walk, clipper, walls, flats, portals, sky; why the walk is needed |
| [04](04_SPRITES_MODELS_PARTICLES_WEAPONS.md) | Sprites, models, particles, decals, weapons, HUD; finding actors without the walk |
| [05](05_LIGHTS_SHADOWS_FOG_POSTPROCESS.md) | Dynamic lights, shadows, fog, post-process chain |
| [06](06_VULKAN_BACKEND_SUBMISSION.md) | Vulkan backend and draw submission |
| [07](07_VR_OPENXR_STEREO.md) | VR path: OpenXR, stereo, per-eye cost |
| [08](08_FORK_ADDITIONS_HOOKS_DEAD_CODE.md) | Fork additions inventory, every cvar and hook, dead code |
| [09](09_BRANCHES_AND_STATIC_LEVEL_PATH.md) | The four branches; the render lane's static level path |
