# 10 — Measurements on the owner's PC

Every number here was measured on the owner's PC (RTX 3080 Ti, i9-9900KF, 16 GB) through the Claude bridge's
`boottest`, flat screen (no headset), window 3056×1689, on `doomxr_rend1113a.exe` (branch `renderer` at `b3fc177`;
`r_newrenderer 0` = the old render path, the same render code as `main` except where section 09 lists differences).
Runs are uncapped (`+vid_vsync 0 +vid_maxfps 0 +cl_capfps 0`) unless stated, `-rngseed 1113`, player at the map
start, not moved. Raw data: `E:\DOOMWork\_reports\render_bench\teardown_*` (perf_track CSV + summary per run) and
the bridge responses `rr341`-`rr352`.

**What the clocks cover** (see sections 01 and 06 for the blind spots): `frame_ms` is the whole frame;
`cpu_all_ms` the CPU render path including the GPU fence wait (`cpu_syncwait_ms`); `think_ms` the playsim tics run
in that frame; `r_script_ms` the frame's script time as the effects governor attributes it; `ticprofile` (renderer
branch only) splits `P_Ticker` per tic; `profilethinkers -t N` times every thinker's tick for one tic, by class.
`simprofile` (renderer branch) printed negative totals for `whole` and `ticker` in every run: its `cycle_t`
accumulation is broken for those two fields (bug; its other fields look sane). Do not use those two.

## 1. The frame, with and without the owner's mod set (Sunder MAP15 at the start)

| Run (bridge id) | Mods | FPS | frame ms | think ms | script ms | scene build ms | GPU wait ms | thinkers | sprites | dyn. lights |
|---|---|---|---|---|---|---|---|---|---|---|
| rr341 | none | 66.7 | 13.89 | 2.52 | 3.14 | 3.80 | 3.24 | 8,065 | 52 | 56 |
| rr342 | owner's 57-file load order | 3.4 | 315.29 | 249.88 | 258.24 | 4.35 | 25.87 | 53,116 | 192 | 122 |

Medians over the last three quarters of each run. Scene build (`cpu_scene_ms`) splits, no mods: BSP 0.48, clip
1.27, walls 2.30, flats 0.60, sprites 0.14, draw-call state 0.67. With mods: 0.43 / 1.24 / 2.56 / 0.75 / 0.62 /
0.93. GPU with mods: opaque 6.15 ms (walls 3.09, flats 1.21, models 0.84), exposure 1.38, bloom 3.23 plus the
BloomBoost mod's two passes 1.60 + 1.64.

**Reading:** with the owner's mod set the renderer is about 2% of the frame. The game simulation is about 80%.

## 2. Where the tic goes (`ticprofile`, ms per tic)

| Run | Map, mods | interp. reset | particles/beams | player think | WorldTick handlers | thinkers |
|---|---|---|---|---|---|---|
| rr344 | Sunder MAP15, none | 1.25 | 0.00 | 0.03 | 0.01 | 3.24 |
| rr349 | Sunder MAP15, none (300 tics) | 1.39 | 0.00 | 0.03 | 0.01 | 3.79 |
| rr347 | Sunder MAP15, owner | 9.02 | 0.72 | 0.07 | 4.07 | 130.80 |
| rr343 | Abandon MAP13, owner | 6.49 / 6.81 | 0.65 / 0.73 | 0.06 | 3.41 / 2.28 | 87.17 / 61.38 |
| rr348 | Abandon MAP13, owner | 6.50 | 0.70 | 0.06 | 2.87 | 75.24 |

The tic budget is 28.6 ms (35 Hz). With the owner's mods, Sunder MAP15 spends 4.6× the budget on thinkers alone;
Abandon MAP13 2.1-3.0×. The per-actor interpolation reset alone is 6.5-9 ms (section 02, §5.1).

## 3. Which thinkers (`profilethinkers -t 60`, one tic, ms)

**Sunder MAP15 with the owner's mods (rr347)** — top entries:

| ms | calls | class | from |
|---|---|---|---|
| 16.46 | 3,622 | `RSQTHP_Bar` | the owner's RSQ top-HP-bar mod: one bar actor per monster |
| 7.16 | 122 | `champion_CyanController` | mk-champions (2.2-Enemies-mk-champions) |
| 4.79 | 1,646 | `champion_Icon` | mk-champions |
| 4.24 | 1,724 | `SkeletonImp` | Legion of Bones (replaces DoomImp) |
| 2.23 | 7,686 | `Shifter` | (owner's load order; one per ~7 actors) |
| 2.18 | 976 | `champion_Leaves` | mk-champions |
| 2.11 | 102 | `champion_DarkRedController` | mk-champions |
| 1.88 | 1,545 | `PhantomRevenant` | Legion of Bones (replaces Revenant) |
| 1.86 | 972 | `SlayerKnight` | Legion of Bones (replaces HellKnight) |
| 1.53 | 21 | `champion_SilverController` | mk-champions |
| 1.31 | 489 | `CacoLich` | Legion of Bones (replaces Cacodemon) |
| ~4.3 | ~2,900 | `champion_Aggro*` (4 classes) | mk-champions |
| 0.89 | 2,544 | `IA_Sound` | ambience mods |
| 1.57 | 11,332 | `csh_critController` + `csh_Brain` | 4.01-Gameplay_CriticalHits (two thinkers per actor) |

Totals by mod family for that tic (sum of the listed entries): **mk-champions ≈ 25 ms**, **RSQTHP_Bar ≈ 16.5
ms**, **Legion of Bones monsters ≈ 11 ms**, `Shifter` 2.2 ms, crits 1.6 ms, ambience 0.9 ms.

**Abandon MAP13 with the owner's mods (rr348)**: the same order — `RSQTHP_Bar` 7.97 ms (1,891 bars),
`champion_CyanController` 3.64, `champion_Icon` 3.16, `Shifter` 1.96 (7,040), Legion of Bones monsters ≈ 6 ms,
mk-champions controllers and aggro ≈ 12 ms.

**Sunder MAP15 without mods (rr349)**: the whole thinker list is ≈ 2.4 ms; the biggest class is `DoomImp`
(1,724 calls, 0.54 ms); a stock monster or item ticks in 0.27-0.47 µs.

**Reading:** a stock monster ticks in about 0.3 µs; a Legion of Bones replacement in 2-3 µs (≈8×); every monster
also carries a ZScript HP bar (4.5 µs each, more than the monster itself), champion helpers and crit helpers.
The engine's per-thinker overhead (section 02: the VM call layer, the interpolation reset, the three full actor
walks) multiplies by the actor count, which these mods raise from 8,065 to 53,116.

## 4. The busy-wait (`r_ticstability`, `TicStabilityWait`, section 01)

Sunder MAP15, no mods, same session, 700 tics:

| Run | `r_ticstability` | FPS | frame ms |
|---|---|---|---|
| rr345 | 1 (default) | 81.2 | 12.31 |
| rr346 | 0 | 99.9 | 10.01 |

Turning the busy-wait off gains 2.3 ms a frame (+23% FPS) in this flat-screen benchmark. In VR it spends time
`xrWaitFrame` would otherwise block on (section 01 cross-check), so the headset effect needs its own measurement.

## 5. Not measured yet (each section's open questions list the exact runs)

- Anything in the headset: per-eye cost, `vr_openxr_multiview 0/1`, `vr_scene_multithread 0/1`, the three polled
  fence waits, the XR blit, pose staleness. The bridge cannot run a headset.
- Polled vs blocking fence waits (needs a build).
- Whether the ZScript JIT is on in the owner's build (startup log line; `vm_jit` default true).
- The upload-queue family on the 3080 Ti (startup log line `Upload Queue Family`).
- Mid-frame stalls (stream buffer, matrices, descriptor pools): no counters exist (section 06).
