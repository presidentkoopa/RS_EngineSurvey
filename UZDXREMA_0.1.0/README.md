# UZDXREMA 0.1.0 teardown

Engine: UZDXREMA (UZDoom 5.0 base, DoomXR VR, Vulkan-only, OpenXR), repo `presidentkoopa/UZDXREMA`.
Read at: `main` f182cbcc (engine of record), `renderer` 175063a, `netplay` 1c1805f8, `selaco` 7e111a3d.
Rules: only the code is evidence; every claim is `file:line` and marked VERIFIED (read) or INFERRED (deduced).

**Start here: [00_UZDXR_ENGINE_TEARDOWN.md](00_UZDXR_ENGINE_TEARDOWN.md)** — how a frame runs, what slows it down
(measured first), what the engine does not do, bugs found, dead and dormant code, the branches, open measurements.

Status (2026-11-13): sections 01-09 written and cross-checked against the code by a second reader (corrections
marked `[xc: ...]`, each file ends with its cross-check record); 10 holds the numbers measured on the owner's PC;
00 is the summary built from 01-10. Not in this folder: the rebuild instruction set (built from this teardown and
the Helion breakdown), and the headset measurements listed in 00 §8.

| Section | Area |
|---|---|
| [00](00_UZDXR_ENGINE_TEARDOWN.md) | Summary reference: frame, costs ranked, gaps, bugs, dead code, branches, open measurements, index |
| [01](01_FRAME_LOOP_TIMING_THREADS.md) | Frame loop, tic timing, threads, every place the main thread waits |
| [02](02_PLAYSIM_SCRIPTING.md) | Playsim and ZScript/ACS cost per tic; what blocks a parallel sim |
| [03](03_SCENE_BUILD_BSP_WALLS_FLATS.md) | CPU scene build: BSP walk, clipper, walls, flats, portals, sky; why the walk is needed |
| [04](04_SPRITES_MODELS_PARTICLES_WEAPONS.md) | Sprites, models, particles, decals, weapons, HUD; finding actors without the walk |
| [05](05_LIGHTS_SHADOWS_FOG_POSTPROCESS.md) | Dynamic lights, shadows, fog, post-process chain |
| [06](06_VULKAN_BACKEND_SUBMISSION.md) | Vulkan backend and draw submission |
| [07](07_VR_OPENXR_STEREO.md) | VR path: OpenXR, stereo, per-eye cost |
| [08](08_FORK_ADDITIONS_HOOKS_DEAD_CODE.md) | Fork additions inventory, every cvar and hook, dead code |
| [09](09_BRANCHES_AND_STATIC_LEVEL_PATH.md) | The four branches; the render lane's static level path |
| [10](10_MEASUREMENTS.md) | Measurements on the owner's PC: frame and tic split with and without the mod set, per-class thinker cost, busy-wait A/B |
| [30](30_GAMEPLAN_90HZ.md) | The 90 Hz gameplan for the owner's mod list: headset budget, sim/render/GPU waterfalls, tracks, milestones M0-M14, owner decisions, risks, effort |
| [31](31_FORK_DELTA_CHECKLIST.md) | Implementer checklist (anti-drift): rules block, one row per item with lane, fork file:line, what to do, done-when run, don't-touch, netplay gate |
| [SIM LANE HANDOFF](SIM_LANE_HANDOFF_2026-11-13.md) | Start point of the sim lane (opened 2026-11-13, D1): scope, hard rules, traps, first week, decisions |
| [gameplan_sources/](gameplan_sources/) | The seven code-cited fork-delta reports (A-G) behind 30/31, the owner's decisions log, and the review logs |
