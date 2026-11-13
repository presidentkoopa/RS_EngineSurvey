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
- ~~Whether the ZScript JIT is on in the owner's build~~: answered, it is on (§6.1).
- The upload-queue family on the 3080 Ti (startup log line `Upload Queue Family`).
- Mid-frame stalls (stream buffer, matrices, descriptor pools): no counters exist (section 06).

## 6. Sim lane M0 runs (2026-11-13, bridge requests `sm001`-`sm011`)

All on `doomxr_rend1113a.exe` (`renderer`, `r_newrenderer 0`), desk, flat, `-Map MAP15 -Tics 700`,
`+vid_vsync 0 +vid_maxfps 0 +cl_capfps 0 +perf_track 1 -rngseed 1113`, `r_ticstability` at its default (1; no
tic-less frames occur at this load), `perf_bench_hash` not in play (no demo). Load order: `SIM_SUNDER_1113_a.zdl`,
written by `make_owner_zdl.py` from the rotation folder as it was that day: **47 files**, the same list as rr342/rr347's
`RENDER_SUNDER_1113_ownera.zdl` except `1.2-BD22_VR-Reloaded_2.0.4t.pk3` in place of `2.0.4s` (the old zdl no longer
loads: `sm002` failed on the missing file). `RSQ_Monsters.pk3` and `RSQ_GunCards.pk3` are in the rotation folder but the
script leaves them out (they do not compile on REMA), so **no boot-test number in this file includes RSQ_Monsters**.
Full boot logs are copied to the bridge's `work\boot_<stamp>.txt`; the four full class tables are in
`measurements_raw/sm006_profilethinkers_sunder15.tsv`.

### 6.1 M0-1: the JIT is on (`sm004`, log `boot_20261113-041757.txt`)

| What | Value |
|---|---|
| `vm_jit` | true (default true) |
| `vm_jit_aot` | true (default true) |
| `vm_debug` | false (and `vm_debug=false` in the owner's ini, `sm007`) |
| "is using too many registers" lines | **1**: `wr_Rig.buildSheetRows` (239 of max 200) |

The same single register line is in rr347's own log (`boot_20261113-003832.txt`, line 208). That line is only printed
from `CanJit`, which runs only when `vm_jit` is true (vmframe.cpp:318) and, at load time, only when `vm_jit_aot` is true
too (vmbuilder.cpp:955), so **[M] §2-3 were measured with the JIT on**. Closer L4 does not apply. Not covered: functions
with `blockJit` fall back silently (vmframe.cpp:302), and the owner's headset launcher was not inspected for `-debug`.

### 6.2 M0-2: where the thinker time goes (`sm006`, log `boot_20261113-042224.txt`; `sm010`, log `boot_20261113-042852.txt`)

`ticprofile` resets when it prints, so a print on every tic gives one-tic samples, and a print just before and just
after a profiled tic gives the thinker clock **of the same tic** as the class table.

**Windows (ms a tic):**

| Window | reset | WorldTick | thinkers |
|---|---|---|---|
| tics 1-199 (start-up included) | 9.15 | 4.54 | 139.48 |
| tics 200-399 | 8.98 | 2.85 | 117.15 |
| tics 404-499 / 500-599 / 600-689 | 8.72 / 9.14 / 8.97 | 2.80 / 2.87 / 2.75 | 120.26 / 120.48 / 113.87 |

[M] §2's 130.80 (rr347) is one print over tics 1-269, so it carries the start-up window. The steady value is
**≈113-120 ms** thinkers and **≈2.8 ms** `WorldTick`.

**64 consecutive one-tic samples, no profiler (`sm010`, tics 440-503):**

| | mean | median | p95 | min | max |
|---|---|---|---|---|---|
| thinkers | 125.4 | **78.6** | **224.0** | 67.8 | 975.7 |
| whole `P_Ticker` (sum of the buckets) | 138.5 | **90.3** | **236.8** | 79.2 | 990.4 |

The tic is not flat. Three tics in four cost ≈70-80 ms of thinkers; **every fourth tic costs ≈185-225 ms**; and one
tic of the 64 cost 976 ms (cause not yet known). Light tics average 75.4 ms (n = 44).

**Four profiled tics in a row (`sm006`):**

| Tic | classes | calls | sum of class time | thinkers clock, same tic | outside any class |
|---|---|---|---|---|---|
| A (light) | 270 | 53,015 | 65.65 | 80.34 | 14.7 |
| B (heavy) | 272 | 56,957 | 155.87 | 197.37 | 41.5 |
| C (light) | 269 | 53,037 | 60.62 | 74.79 | 14.2 |
| D (light) | 269 | 53,037 | 59.68 | 74.51 | 14.8 |

**The heavy tic is one Legion of Bones actor.** B minus C, by class: `BalrogFireSpawner` +82.40 ms (191 calls, 0.43 ms
each; 0.06 ms in total on a light tic), `BalrogFireGiver` +2.75 (1,307 new actors), `PowerBalrogFireBuffGiver` +2.74
(1,307 new), `PowerBalrogFireBuff` +2.46, `BalrogBuffEffectSpawner` +1.34; everything else together +3.6. The source
(`2.1-Enemies_LegionOfBones_v1.05.pk3:decorate/baron.txt:112-164`, read through the bridge, `sm009`): each of the 191
Balrogs owns a `BalrogFireSpawner` whose loop is 4 tics long and makes **32 `A_RadiusGive("BalrogFireGiver", 500, …)`
calls in a row, one per monster class**, unless `LOB_Hellfire > 1`. Each call is a blockmap scan
(p_actionfunctions.cpp:4012-4022): 191 × 32 = 6,112 scans in one tic, ≈13.5 µs each. Every monster reached gets a
`BalrogFireGiver`, a `PowerBalrogFireBuffGiver` and a `PowerBalrogFireBuff` spawned that tic: 3,920 more thinkers than
the tic after, each through every
`WorldThingSpawned` handler, which is the extra ≈27 ms "outside any class" on the heavy tic (≈7 µs a spawn).
**Inference:** the Balrog aura costs ≈120 ms on every fourth tic, ≈30 ms a tic on average.

**The light tic has no long tail.** Tic C: 269 classes, 60.62 ms; the top 14 are 47.0 ms, the top 25 are 54.6, the top
60 are 59.3, and classes 61-269 together are 1.3 ms. By family: mk-champions 27.7 ms (51 classes, 10,594 thinkers),
`RSQTHP_*` 14.5, crits 1.6, `Shifter` 2.3, torches 1.1.

**All 1,646 champion controllers are in the table** (17 colours = 1,646 `champion_Icon`), 14.44 ms together:

| Colour | n | µs each | | Colour | n | µs each |
|---|---|---|---|---|---|---|
| Silver | 21 | 68.8 | | Violet | 101 | 4.3 |
| Cyan | 122 | 56.5 | | Indigo | 118 | 4.1 |
| DarkRed | 102 | 25.5 | | the other nine colours | 960 | ≈1.0 |
| Gold | 13 | 8.6 | | | | |
| Grey / Orange | 97 / 112 | 7.9 / 6.6 | | | | |

So the "≈1,400 unlisted controllers at 7-21 µs" hypothesis (gameplan §3.2 rule 8a, 10 / 20 / 30 ms) is wrong: the
controllers [M] did not list (all but Cyan, DarkRed and Silver) cost ≈3.5 ms in total.

**What the old "≈78 ms unattributed" was** (130.80 − 52.5 listed): ≈30 ms the Balrog aura on every fourth tic;
≈14.5 ms a tic that sits in `RunThinkers` but in no class row (loop, the profiler's own clocks, spawn events: needs the
F-MON-0 spawn bucket to split); ≈11-17 ms the start-up window inside the 130.80; ≈9 ms the classes below [M]'s cut (rr347's own top 60 sum to 61.45 ms);
the rest rare spikes like the 976 ms tic.

**Which monsters tick (U10).** No stock, BD22 or XMVE monster class is in the table, apart from 13 `TheCyberdemon`
(with `CybersHitBox`, `TargetIsACyberdemon`), as gameplan §3.8 predicted. RSQ_Monsters is not loaded, and
`rsq_monster_set` / `rs_roster_enable` are not in the ini.

**Counts the checklist asked for (tic C):** `RSQTHP_Bar` 3,622, `RSQTHP_Part` 1,706 (0.06 ms), `RSQTHP_ChampKind` 1,622;
torch helpers 3,111 in 22 classes (1.10 ms); `TechLamp1Flare` 0, `Column1Flare` 0; `Brutal_*` 10 (`Brutal_BloodSpot`);
`csh_critController` + `csh_Brain` 5,666 each; `LDLegendaryMonsterToken` 165.

**Not settled:** the script share s (U3: `stat vm` is on screen only, so it waits for the `vmstats` tooling); what
the 976 ms tic is; the split of the ≈14.5 ms outside the class clocks.

### 6.3 M0-5: values in the owner's ini (`sm007`, read only)

`vid_vsync=false`, `vid_maxfps=500`, `vid_refreshrate=72`, `vr_mode=15`, `vr_openxr_sync_mode=1`, `vr_desktop_view=2`,
`vr_openxr_multiview=false`, `vr_scene_multithread=false`, `vr_openxr_render_scale=1`, `gl_multisample=1`, `gl_ssao=0`,
`vk_gpu_checkpoints=true`, `gl_custompost=true`, `gl_bloom=true`, `gl_bloom_amount=1.6`, `gl_bloom_step=1`,
`gl_bloom_threshold=0.8`, `gl_bloom_knee=0.55`, `gl_bloom_pin_beams=false`, `gl_bloom_pin_step=1`,
`r_ticstability=true`, `perf_bench_hash=true`, `vm_debug=false`, `sv_corpsequeuesize=10`, `cl_maxdecals=20`,
`r_maxparticles=4000`, `zdoombrutaljanitor=0`, `bd_disabledecorations=0`, `x5_multiplier=100`, `champion_Icons=1`,
`champion_Particles=1`, `csh_critical_enabled=true`, `rsqthp_aimed=false`, `ue_recolour=5`, `ue_resize=5`,
`ue_decor_recolour=5`, `ue_decor_resize=5`, `ue_friend=false`, `mp_host_skill=1`, `bd_skill=0`.
Not in the ini (so at their defaults): `rsq_monster_set`, `rs_roster_enable`, `sim_maxcatchup`, `vm_jit`, a skill line.
