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

### 6.4 M0-11: ten more loads of Sunder MAP15 in one session (`sm017`, log `boot_20261113-044430.txt`)

`map MAP15` every 100 tics, ten times, after the first start: **11 loads, 272 s, no "Too many translations" abort**, run
GREEN. The predicted abort after ≈9-10 loads (7,686 Universal Entropy translations a load against the 65,535 limit) did
**not** reproduce with `ue_recolour=5`. The count itself was not read (no console command prints it), so either fewer
translations are stored per load than predicted or equal ones are already shared. F-MON-6 keeps its row; its
"removes an abort" claim is unproven until a counter exists.

### 6.5 M0-7: Abandon MAP13, renderer build only so far (`sm015`, log `boot_20261113-044301.txt`)

`doomxr_rend1113a.exe`, 47-file list with BD22 VR Reloaded 2.0.4t: thinkers 61.66 ms a tic over 296 steady tics
(75.86 over tics 1-299). Four profiled tics in a row: thinkers clock 47.7 / 88.0 / 46.1 / 46.4 ms. `Shifter` 7,040
calls, 1.79-2.02 ms (0.25-0.29 µs a call). The fourth-tic peak is here too: `BalrogFireSpawner` 40 calls, 20.75 ms on
the heavy tic. `RSQTHP_Bar` 1,891 calls, 6.8-8.2 ms.

**The `main` half did not run.** `doomxr_sim1113a.exe` was built from `main` e429f22 (`sm014`, 764 s, 0 errors), but by
the time its boot test ran (`sm024`) the `doomxr.pk3` beside it in the shared `build-dxr\RelWithDebInfo` folder had been
replaced by another lane's build: `doombase.zs line 3504: The function 'VRTorso.ViewLean' has not been exported from
the executable` (no `ViewLean` exists in `main` at e429f22). Every named exe in that folder shares one `doomxr.pk3`, so
an exe is only bootable until the next build by any lane. The rotation also changed twice during the session
(2.0.4s → t → v), so a before/after pair must be queued back to back on one zdl.

**M0-7, second try (`sm028`-`sm031`, one zdl `SIM_ABANDON_1113_c.zdl`, 47 files, BD22 VR Reloaded 2.0.4v).** Renderer
exe again: thinkers 61.83 ms a tic over 296 steady tics (`sm030`). **`main` does not run this list:**
`doomxr_sim1113b.exe` (main fedac7e, built in the sim lane's own folder `E:\DOOMWork\UZDXREMA_sim`, its own
`doomxr.pk3`) loaded Abandon MAP13 and stopped at level time 71 with `Attempted to write past end of stream`
(`I_Error` in src/common/engine/i_protocol.cpp:287-332), then sat in the error state until the 900 s timeout (`sm031`,
log `boot_20261113-060020.txt`). The renderer exe on the same files runs. So there is no `Shifter` number for `main`
yet, and the first fact about `main` on the owner's list is that it aborts.

### 6.6 First headset runs (owner, 2026-11-13; `E:\DOOMWork\_reports\sim_lane\headset_1113_A` and `_B`)

`doomxr_rend1113a.exe`, `vr_mode 15`, Sunder MAP15 start, a copy of the owner's ini (`vr_openxr_multiview=false`,
`r_ticstability=true`, `vid_refreshrate=72`), `perf_track` summary files. **Indicative only:** desk boot tests of this
lane were running on the same PC at about the same minutes, the runs were started several times and one file of each
survived, and the launcher's memory reading failed (a mistake in the launcher).

| Run | frames | frame ms avg / p50 / p95 | scene build (`cpu.scene`) avg | `cpu.syncwait` avg | `think_ms` per frame with a tic, avg / p50 / p95 |
|---|---|---|---|---|---|
| A, no mods | 2,094 in 60.5 s | 28.9 / 26.8 / 44.6 | 11.7 | 3.7 | 6.8 / 4.0 / 19.3 |
| B, 47-file list | 223 in 64.3 s | 288.5 / 294.4 / 514.8 | 16.2 | 22.3 | 319.4 / 290.0 / 419.7 (max 2,850) |

**Reading:** in the headset even the no-mod map runs at ≈35 frames a second (two scene builds of ≈5.8 ms each and a
frame of ≈29 ms), and with the list at ≈3.5. The runtime reported no dropped or stale frames in either.

### 6.7 God mode (owner: "iddqd crashes the engine")

Desk, flat, Sunder MAP15, 47-file list, `doomxr_rend1113a.exe`: `god` at tics 100, 200, 300 (`sm033`) and `iddqd` at
the same tics (`sm034`): both GREEN, "Degreelessness Mode ON / OFF / ON", no crash in 500 tics. **Not reproduced at the
desk.** Not tried: the headset, a `main` exe, and staying in the level longer.

## 7. Sim lane M3 tools, first runs on `main` (2026-11-13, bridge requests `sm053`-`sm063`)

`doomxr_sim1113c.exe`, built in `E:\DOOMWork\UZDXREMA_sim` from `main` 9c82694 (`sm053`, 744 s), desk, flat,
`-Tics 700`, `+vid_vsync 0 +vid_maxfps 0 +cl_capfps 0 +perf_track 1 -rngseed 1113 +sim_profile 1
+sim_profile_interval 0`, cfg `tools/sim_lane/sim_m3_dump1113c.cfg` (`sim_profile_reset` at tic 390, `sim_profile_dump 60`
at 690: 300 steady tics). `r_ticstability` and `perf_bench_hash` as in the owner's ini. The numbers include the
profiler's own cost (two clock reads and a map lookup per thinker, four clock reads per spawn). Load orders made fresh
(`sm054`, `sm055`): 47 files, the same lists as §6 except `1.2-BD22_VR-Reloaded_2.0.4u.pk3`. Every `simprof` line of the
four logs is in `measurements_raw/sm056..sm059_simprof_*.txt`. **This is the first run of the owner's list on a `main`
exe that did not abort** (netplay lane's fixes) **and the first with the sound fixes on `main`.**

| Run | map, list | loop mean | median | p95 | max | classes (Tick) | spawn bucket (events / native) | rest | thinkers | spawns a tic |
|---|---|---|---|---|---|---|---|---|---|---|
| `sm056` | Sunder MAP15, no mods | 3.44 | 2.96 | 5.78 | 8.85 | 3.26 | 0.00 | 0.18 | 8,065 | 0 |
| `sm057` | Sunder MAP15, 47 files | 115.09 | 73.47 | 203.46 | 881.96 | 97.22 | 17.01 (16.61 / 0.40) | 0.87 | 54,001 | 2,742 |
| `sm058` | the same, second run | 112.81 | 73.76 | 194.82 | 885.09 | 94.89 | 17.05 (16.68 / 0.37) | 0.87 | 54,001 | 2,742 |
| `sm059` | Abandon MAP13, 47 files | 80.54 | 54.80 | 194.64 | 662.71 | 65.20 | 13.76 (13.46 / 0.30) | 1.58 | 41,277 | 1,778 |

All ms per run of `RunThinkers` (one a tic). Classes + spawn bucket = 99.2 % of the loop on Sunder (P0.5-1's and
F-MON-0's gate: within 5 %).

**What the thinker loop is made of (Sunder, `sm057`, mean ms a tic over 300 tics):** `BalrogFireSpawner` 22.18;
`RSQTHP_Bar` 14.15; `champion_PinkController` 10.93; `champion_CyanController` 7.17; `champion_Icon` 5.17;
`SkeletonImp` 3.52; `champion_DarkRedController` 2.57; `Shifter` 2.56 (7,686 calls, 0.333 µs each); `champion_Leaves`
2.25; everything else ≈27. Spawn bucket by class: `champion_AggroPower` 3.53, `champion_AggroEffect` 3.20,
`champion_AggroGiver` 3.17 (562 spawns of each **every tic**), `PowerBalrogFireBuffGiver` 2.29, `PowerBalrogFireBuff`
2.16, `BalrogFireGiver` 2.10 (327 of each a tic on average). A spawn costs 5.5-7 µs of `WorldThingSpawned` handlers.

**The ≈14.5 ms "outside every class clock" (§6.2) is spawn events.** The loop itself (sleep cycle, list walk, profiler)
is 0.87 ms.

**The every-fourth-tic burst, split (`sm058`, six heavy tics, maptime 392-412):** loop 182-192 ms = classes 144-153 +
spawn events 36-38 (≈5,880 spawns) + rest ≈1. In it `BalrogFireSpawner` is 78.3 ms (191 calls, 410 µs each), and
its three gift classes cost 26.2 ms of spawn events (1,307 spawns each) plus ≈7.7 ms of their own first Tick.

**The one-second tic is `champion_PinkController`** (mk-champions). Sunder: every 70th tic (maptime 69, 139, 209, ...
699), loop 860-885 ms, of which 130 Pink controllers are ≈793 ms (6.1 ms a call). Abandon: the same tics at 300-510 ms
(63 controllers), one at 663. As a mean it is 10.9 ms a tic on Sunder; as a freeze it is 0.86 s every 2 seconds of
game time.

**Map start (Sunder):** tic 0 = 2,873 ms (45,428 spawns; spawn events 1,294 ms), tic 1 = 1,894 ms (84,892 spawns),
tic 4 = 459 ms. Abandon: 2,338 / 1,168 / 564 ms.

**Client side (`RunClientSideThinkers`, Sunder):** 4.36 ms a call, of which the dynamic lights' own ticks are 0.22; the
rest is the recreate-lights walk over every actor (P9.9c's target).

**Not measured here:** the script share s (F-MT-0), anything outside `RunThinkers` (no `ticprofile` on `main`), a fight.

## 8. Sim lane M3 tools, second batch (2026-11-15, bridge requests `sm066`-`sm078`)

`doomxr_sim1115a.exe`, built in `E:\DOOMWork\UZDXREMA_sim` from `main` fce83ec (`sm069`, 667 s). Same flags as §7, cfg
`tools/sim_lane/sim_m3_dump1115a.cfg` (totals restart and `vm_stats 1` at tic 390, dumps at 690: 300 steady tics). Load
orders made fresh (`sm070`, `sm071`): 47 files, as §7 except the owner's BD22 pack is now
`1.2_BD22_VR-Reloaded_2.0.4x.pk3`, so **these runs are not like-for-like with §7** (Abandon's loop is 63 ms here, 81
there). Raw tool lines: `measurements_raw/sm073..sm075_m3b_*.txt`. All four desk runs GREEN.

| Run | map | loop mean | median | p95 | max | classes | spawn bucket | rest | script inside the loop (`vm_stats`) | script a game tic |
|---|---|---|---|---|---|---|---|---|---|---|
| `sm073` | Sunder MAP15 | 113.14 | 74.48 | 195.77 | 826.09 | 93.10 | 18.71 | 1.33 | **95.77 (84.7 %)** | 98.42 |
| `sm074` | the same, `vm_profile` on (slows script) | 119.56 | 79.57 | 208.16 | 818.67 | 96.49 | — | — | — | — |
| `sm075` | Abandon MAP13 | 62.97 | 48.29 | 96.77 | 291.54 | 49.62 | 12.04 | 1.31 | **49.96 (79.3 %)** | 52.11 |

**s (U3).** Script entered from native code, with the natives it calls, is 84.7 % of the thinker loop on Sunder and
79.3 % on Abandon; the gameplan's working range was 0.6-0.8. 169,558 entries from native code a tic on Sunder.

**JIT.** `VM: JIT built=yes vm_jit=1 vm_jit_aot=1 compiled=129200 interpreted=2 not yet compiled=0`. The two interpreted
functions: `BaseStatusBar.SortScoreboardPlayers`, `wr_Rig.buildSheetRows`. No mods: compiled=2,718, interpreted=1.

**Per script function (Sunder `sm074`, self ms a tic, top of 1,421 functions called, 518,746 script calls a tic):**
`BalrogFireSpawner.StateFunction.4` 16.71 (47.6 calls a tic, 351 µs each); `champion_PinkController.champion_TickEffect`
9.74 (1.7 calls a tic, 5.6 ms each); `Actor.A_RadiusThrust` 7.18 (143 a tic, 50 µs); `RSQTHP_Bar.Update` 7.05
(3,622 a tic); `champion_Icon.Tick` 4.89; `Inventory.Tick` 3.48 (28,394 a tic); `champion_BaseController.Tick` 2.35
self, 25.61 inclusive; `champion_Leaves.Tick` 2.19; `Actor.DoGiveInventory` 2.11; `PowerupGiver.Use` 2.07;
`Shifter.Tick` 1.68. `WorldThingSpawned` by handler (2,739 calls a tic each, inclusive): `idleambiance_decoration_EventHandler`
2.36, `wr_StatEvents` 1.04, `RSLD_Handler` 0.53; `idleambiance_decoration_EventHandler.addItemSounds` is called
104,078 times a tic (1.17 ms).

**What each class's Tick does a tic (Sunder `sm073`, counters):**

| class | objects created | sight checks (ms) | P_TryMove (ms) | state actions | ThinkerIterator steps | A_RadiusGive candidates |
|---|---|---|---|---|---|---|
| `BalrogFireSpawner` | 1,028 | 815 (0.16) | 47 (0.09) | 239 | 0 | **443,780** |
| `champion_PinkController` | 1.7 | 0 | 0 | 0 | **87,349** (6,496,810 in its one tic of 70) | 0 |
| `champion_CyanController` | 0 | **7,567 (2.99)** | 0 | 0 | 0 | 0 |
| `champion_Icon` | 0 | 0 | 1,646 (2.27) | 0 | 0 | 0 |
| `champion_DarkRedController` | 712 | 237 (0.10) | 0 | 0 | 0 | 439 |
| whole loop | 2,746 | 9,394 (3.86) | 5,967 (6.25) | 1,560 | 87,349 | 445,237 |

All sight checks together are 3.9 ms a tic and all `P_TryMove` 6.2 ms: neither is where the time goes. No script
`BlockThingsIterator` steps at all on this map.

**The Balrog burst is `A_RadiusGive` testing candidates:** 443,780 a tic on average, ≈1.77 M on a burst tic (≈9,300 a
call for 191 spawners), each through `DoRadiusGive`. **The one-second tic is 130 whole-level `ThinkerIterator` walks**
(mod code in the checklist, F-SIM-13).

**Netplay suite note for this batch:** see the checklist change log, 2026-11-15 GATE-SUITE (`clamp` intermittent).

## 9. First M4 row: `sim_idleinventory` off and on (2026-11-15, bridge requests `sm080`-`sm105`)

`doomxr_sim1115b.exe` (`main` fdb9125, `sm081`, 409 s). Flags as §7 plus `+vr_synclog <file>`; cfg
`tools/sim_lane/sim_m4_sync1115c.cfg`; load orders `SIM_SUNDER_1115_c.zdl`, `SIM_ABANDON_1115_d.zdl` (47 files, as §8).
`sim_profile` and `vm_stats` are on in every run, so the times carry their cost equally. Raw:
`measurements_raw/sm092..sm096_m4_*.txt`, `sm087_m4_sunder15_vmprofile_all.txt`.

| Run | map | switch | loop mean | median | p95 | max | script a tic | entries from native a tic |
|---|---|---|---|---|---|---|---|---|
| `sm092` | Sunder MAP15 | off | 122.70 | 80.65 | 209.73 | 940.80 | 106.57 | 169,558 |
| `sm093` | Sunder MAP15 | off (repeat) | 121.13 | 80.50 | 208.28 | 949.87 | 105.24 | 169,558 |
| `sm094` | Sunder MAP15 | **on** | 121.57 | 79.92 | 209.87 | 954.57 | 104.65 | **151,981** |
| `sm095` | Abandon MAP13 | off | 68.64 | 54.70 | 107.43 | 422.37 | 56.35 | 108,931 |
| `sm096` | Abandon MAP13 | **on** | 64.30 | 49.73 | 95.22 | 330.23 | 52.53 | **99,103** |

**The game is identical with the switch on** (state log equal every tic through maptime 680, `sm097`-`sm099`).

**What it saves.** 17,577 VM calls a tic on Sunder and 9,828 on Abandon. By class on Sunder: `csh_Brain` 1.00 → 0.30,
`csh_critController` 1.03 → 0.33, `Shifter` 2.75 → 2.54 ms a tic (what is left is mostly the profiler's own cost per
thinker). **In the loop's mean it is about 1 ms on Sunder, which is inside the run-to-run spread** (the two off runs
differ by 1.6 ms). Abandon shows −4.3 ms, from one run each, so not all of that is this switch. The idle items were
cheap calls (≈0.1 µs each); the 10,875 `Inventory.Tick` calls a tic that remain (3.5 ms) are items lying in the world or
in an animated state, which this switch rightly does not touch.

**`WorldTick` by handler (Sunder, `sm087`, `vm_profile`, switch on):** 121 handlers, 3.02 ms a tic inclusive, 0.37 self.
`RSQTHP_Handler` 0.63, `RSQA_QoL` 0.47, `RSQGD_Handler` 0.26, `RSQ_Voxels` 0.17, `RSQFF_Shown` 0.14, `RSQLS_Sight` 0.13,
`SSQ_Handler` 0.08, `RSQWP_Whip` 0.07, `rsqdg_door` 0.06, `RSQ_Counters` 0.06.

## 10. Universal Entropy with and without its `Shifter` helpers (2026-11-15, bridge requests `sm121`-`sm133`)

Checklist PORT-UE, the mod half of F-MOD-9. `doomxr_sim1115c.exe` (`main` 1eaf50d). Flags as §9 with
`+sim_idleinventory 1` in all four runs; 700 tics, profile over the last 300; load orders `SIM_SUNDER_1115_c.zdl` /
`SIM_ABANDON_1115_d.zdl` (original mod) against `SIM_SUNDER_1116_ue_a.zdl` / `SIM_ABANDON_1116_ue_a.zdl` (same 47 files
with `2.4-Enemies_UniversalEntropy_REMA1116a.pk3`, sha256 5697e767..., in place of the original af6bb724...). Source of
the copy: UZDXREMA `tools/sim_lane/universal_entropy_rema1116a/`. Raw: `measurements_raw/sm123..sm126_m4_*.txt`.
One run each, all GREEN, no script errors.

| run | loop mean ms | median | p95 | max | thinkers | script in loop ms | `Shifter` ms / calls |
|---|---|---|---|---|---|---|---|
| sm123 Sunder MAP15, original | 116.47 | 76.00 | 203.6 | 915.6 | 54,001 | 100.29 | 2.451 / 7,686 |
| sm124 Sunder MAP15, copy | 115.84 | 78.97 | 215.0 | 790.5 | 46,299 | 99.39 | none |
| sm125 Abandon MAP13, original | 62.31 | 48.62 | 95.10 | 332.1 | 41,277 | 50.75 | 2.166 / 7,040 |
| sm126 Abandon MAP13, copy | 58.24 | 45.57 | 90.84 | 227.3 | 34,233 | 47.55 | none |

- The two games are **not** the same game tic for tic: the engine field scales a monster's first state too, the item
  did not, so monsters fall out of step with the original run early. Whole-loop numbers therefore carry play-out
  noise (Sunder's median and p95 went up while its mean and max went down). The clean reading is by class:
  `Shifter` -2.45 / -2.17 ms, `champion_PinkController` -1.42 / -1.29 ms (its whole-level `ThinkerIterator` walks step
  over about 7,700 fewer objects), every other class within +-0.5 ms.
- sm121/sm122: no other file in the 47 references `Shifter` (`tools/sim_lane/sim_zdl_grep.py`).
- sm133: the owner's play list (bridge BD22 pack) with the copy, MAP01, 700 tics, GREEN (boot check only, no profile).
- Not measured: anything in the headset; any netplay run with the copy.

## 11. `sim_handlerfind` off and on (F-MOD-6; 2026-11-16, bridge requests `sm135`-`sm147`)

`doomxr_sim1116a.exe` (`main` b43e831, `sm136`, 1087 s: the first link stopped on LNK1103 and the build script's
retry built it). Flags as §10 with `+sim_idleinventory 1` and `+sim_handlerfind 0` / `1`; load orders
`SIM_SUNDER_1116_ue_a.zdl`, `SIM_ABANDON_1116_ue_a.zdl` (47 files with the Universal Entropy copy). Raw:
`measurements_raw/sm137..sm140_m4_*.txt`. The two new right-hand columns of `sim_profile`'s counter lines are
`hdl finds` (script `Find` calls a tic) and `find ms`.

| run | `Find` calls / tic | `Find` ms / tic | loop mean ms | median | p95 | `RSQTHP_Bar` ms |
|---|---|---|---|---|---|---|
| sm137 Sunder MAP15, off | 7,605.6 | 3.413 | 115.63 | 75.28 | 204.4 | 14.81 |
| sm138 Sunder MAP15, on | 7,605.6 | 0.054 | 107.54 | 68.72 | 192.9 | 10.62 |
| sm139 Abandon MAP13, off | 3,977.1 | 1.935 | 71.54 | 55.86 | 112.3 | 8.74 |
| sm140 Abandon MAP13, on | 3,977.1 | 0.038 | 62.16 | 48.31 | 98.12 | 5.87 |

- Same game: `sm141`, `sm142` (`sim_synccmp2.py`, with the world hash, same build): IDENTICAL through maptime 680 on
  both levels, off against on.
- The clean saving is the `find ms` column: -3.36 ms (Sunder) and -1.90 ms (Abandon). The whole-loop means moved by
  more (-8.1, -9.4), but the Abandon "off" run is 13 ms slower than §10's run of the same game on `doomxr_sim1115c.exe`
  (58.24), so the PC was not equally quiet across these runs; do not read the loop means as the row's saving.
- Who calls it: `RSQTHP_Bar` (the owner's RSQ pack, the health bar on each monster) makes 7,597.5 of the 7,605.6 calls
  on Sunder: 3,623 bars, about two `Find` a tic each. After the change the class still costs 10.6 / 5.9 ms a tic.
  That is a row of its own for whoever owns RSQ (one handler lookup per bar per tic can be one per bar for life).
- Done-when of F-MOD-6 (`Find` <= 0.1 ms a tic): met, 0.054 and 0.038.

## 12. What whole-level walks from script cost inside the thinker loop (P10.5 / F-SIM-13; 2026-11-16, `sm150`-`sm156`)

`doomxr_sim1116b.exe` (`main` 64b5098, `sm152`). Flags as §11 with `+sim_idleinventory 1 +sim_handlerfind 1`. New
columns on `sim_profile`'s counter line: `ti ms` (time inside script `ThinkerIterator.Next`, the engine's walk only)
and `ti skipped` (thinkers stepped over because they were not of the asked class). Raw:
`measurements_raw/sm153_m4_sunder15_ti_walks.txt`, `sm154_m4_abandon13_ti_walks.txt`.

| run | loop mean ms | `ti steps` / tic (returned) | `ti ms` / tic | `ti skipped` / tic | who |
|---|---|---|---|---|---|
| sm153 Sunder MAP15 | 108.58 | 73,993 | 1.934 | 3,161 | `champion_PinkController` 1.934 ms, 3,030 skipped; `SlayerKnight` 131 skipped |
| sm154 Abandon MAP13 | 58.10 | (see raw) | 0.656 | 4,088 | `champion_PinkController` 0.656 ms, 3,798 skipped; `SkeletonDemon` 291 skipped |

- sm150: the 47 files hold 55 `ThinkerIterator.Create` sites; about 30 name one narrow class with no statnum, about
  25 ask for `Actor` / `Inventory` and filter in script.
- **Result: inside the thinker loop the narrow-class walks cost nothing measurable** (about 3-4 thousand skipped
  thinkers a tic in total, well under 0.1 ms). A per-class index used invisibly by the existing iterator (P10.5's
  engine half) would save under 0.1 ms a tic here. Not worth building on this evidence. Walks made from event
  handlers' `WorldTick` are outside these columns and are bounded by P9-WT's 3.02 ms for all 121 handlers.
- **Pink (F-SIM-13), what one call is made of:** class total 10.17 ms a tic on Sunder; the engine's walk is 1.93 ms
  of it (26 ns a returned thinker); the other 8.2 ms is the mod's own script body run once per actor in the level
  (about 110 ns a step). So an engine list cannot remove it: the script really does look at every actor. What is
  left for the engine is making each script step cheaper, or an exact spatial answer the script did not ask for.
- Top classes this run, ms a tic, Sunder: `BalrogFireSpawner` 19.63 (F-SIM-12, `A_RadiusGive`), `RSQTHP_Bar` 10.60,
  `champion_PinkController` 10.17, `champion_CyanController` 7.30 (7,506 sight checks a tic, 3.1 ms of it),
  `champion_Icon` 5.22; spawn events 18.5. Abandon: `RSQTHP_Bar` 5.45, `BalrogFireSpawner` 5.44, Cyan 4.16.

## 13. What `A_RadiusGive` is made of (F-SIM-12 "measure first"; 2026-11-16, `sm157`-`sm162`)

`doomxr_sim1116c.exe` (`main` 8a50a17, engine source e7487ca, `sm158`). Flags as §12. New `sim_profile` columns: `rg calls`,
`rg ms` (whole call), `rg gifts`, `gift ms` (`Spawn` + `CallTryPickup`). Raw:
`measurements_raw/sm159_m5_sunder15_radiusgive_split.txt`, `sm160_m5_abandon13_radiusgive_split.txt`. The PC was
busier in this batch (Sunder loop mean 131.3 against 108.6 in §12 for the same game), so read the shares, not the
absolute ms.

| run, per tic | calls | candidates | whole ms | gifts | gift ms | gather + test ms | ns / candidate |
|---|---|---|---|---|---|---|---|
| Sunder, `BalrogFireSpawner` | 1,432.5 | 444,538 | 22.98 | 326.8 | 3.82 | 19.16 | 43 |
| Sunder, whole loop | 1,446.9 | 446,003 | 30.56 | 893.7 | 10.72 | 19.83 | |
| Abandon, `BalrogFireSpawner` | 300.0 | 104,194 | 6.49 | 172.8 | 1.73 | 4.75 | 46 |
| Abandon, whole loop | 313.6 | 105,713 | 11.17 | 602.3 | 5.81 | 5.36 | |

- The Balrog's cost is **83 % gathering and testing candidates** (about 310 candidates a call, 43-46 ns each, almost
  all rejected), 17 % gifts. Sight checks are negligible (§11: 808 a tic, 0.17 ms). So the row's mechanism (b), one
  gathered candidate list reused by the same actor's consecutive calls, attacks the right 19 ms.
- A second, small caller (about 14 calls a tic) gives 567 gifts a tic for 6.9 ms: there the cost IS the gifts (12 us
  each: a spawned actor, its spawn events, a script pickup). That is F-MON-1's ground (spawn events), not (b)'s.

## 14. `sim_radiusgivecache` off and on (F-SIM-12 mechanism (b); 2026-11-16, `sm164`-`sm178`)

`doomxr_sim1116d.exe` (`main` 661932e, engine source 45254d0f; `sm174`). **Compiler changed under this build:** Visual
Studio updated itself on the owner's PC during the session (MSVC 14.51.36231 -> 14.52.37025); the sim folder was
re-configured (`sm172`, `sm173`) and fully rebuilt, so do not compare these absolute times with earlier sections.
Flags as §13 plus `+sim_radiusgivecache 0` / `1`. Raw: `measurements_raw/sm166..sm169_m5_*.txt`.

| run, per tic | loop mean | median | p95 | max | Balrog `rg ms` | of it gifts | Balrog class ms | whole-loop `rg ms` |
|---|---|---|---|---|---|---|---|---|
| sm166 Sunder MAP15, off | 119.25 | 77.34 | 212.6 | 896.5 | 20.21 | 3.43 | 21.31 | 27.00 |
| sm167 Sunder MAP15, on | 107.53 | 77.27 | 164.7 | 957.2 | 8.65 | 3.21 | 9.64 | 15.45 |
| sm168 Abandon MAP13, off | 60.96 | 46.14 | 96.64 | 291.4 | 6.35 | 1.25 | 6.64 | 10.12 |
| sm169 Abandon MAP13, on | 57.49 | 45.79 | 84.57 | 285.5 | 3.27 | 1.39 | 3.53 | 7.29 |

- Same game: `sm170`, `sm171` (`sim_synccmp2.py`, with the world hash): IDENTICAL through maptime 680 on both levels,
  off against on, with the same counts in both runs (1,432.5 calls, 444,538.2 candidates tested, 326.8 gifts a tic
  on Sunder): every candidate is still tested, only the gathering is saved.
- Saving: -11.6 ms a tic on Sunder, -3.1 on Abandon, and it lands on the heavy tics (Sunder p95 212.6 -> 164.7). What
  is left of the Balrog's gather + test is 5.4 ms on Sunder = about 12 ns a candidate for the tests themselves
  (444,538 a tic) plus a fresh scan after every call that gave a gift that stayed changed.
- Not done from the row: (a) reordering the cheap rejects (now the 5.4 ms above), (c) the gift spawn-event path; the
  row's Done-when (the every-fourth-tic peak within 15 ms of the light tic) was not evaluated.

## 15. Script-call switches and the spawn-event path, off and on (F-MT-0, F-VM-1..4, F-MON-1; 2026-11-16, `sm180`-`sm255`)

Three builds from `main`, all with MSVC 14.52: `doomxr_sim1116e.exe` (f8ebf3e; `sm181`), `doomxr_sim1116f.exe`
(678aad2; `sm203`), `doomxr_sim1116g.exe` (5ed9d30; `sm227`). Flags as §14 with `+sim_idleinventory 1 +sim_handlerfind 1
+sim_radiusgivecache 1` in every run (the three earlier switches are the baseline here), cfg
`tools/sim_lane/sim_m10_vm1116f.cfg`, load orders `SIM_SUNDER_1116_ue_a.zdl` / `SIM_ABANDON_1116_ue_a.zdl` (47 files, the
Universal Entropy copy). Raw: `measurements_raw/sm182..sm236_m10_*.txt`. All runs GREEN.

**What script calls on Sunder MAP15 a tic (`sm182`, `sm228`; `vmstats natives`, `vmstats frames`):**

- Natives called directly by compiled script: 376,623 a tic in 117 natives (`Actor.GetSpecies` 104,138,
  `Actor.Distance2D` 73,507, `ThinkerIterator.Next` 73,503, `Actor.InStateSequence` 22,726).
- Natives through the packed-argument path (`NativeScriptCall`): **121,268 a tic in 57 natives**. Nine are 88 %:
  `Actor.FindState` 28,890, `Actor.FindInventory` 15,538, `CVar.FindCVar` 15,015, `CVar.GetInt` 12,359,
  `Actor.GetRenderStyle` 8,980, `CVar.GetFloat` 8,413, `EventHandler.Find` 7,486, `CVar.GetCVar` 5,740,
  `Actor.SetState` 4,937. Only one virtual native in the list (`Actor.BeginPlay`, 2,670). Abandon MAP13: 72,263.
- Compiled functions that set up a full VM frame: **68,854 a tic in 286 functions** (Abandon 42,482), every one of the
  top forty only because of string registers (`S--`): `RSQSM_Clock.cv` 5,506, two `GetInt` services 4,544 each, and
  nine `WorldThingSpawned` handlers at about 2,751 each (once per spawn).
- Virtual call sites in the 47 files: 1,987; devirtualised with `vm_devirtualize`: 661 (17 to natives, 1 with a direct
  entry).

**Off and on, back to back (ms a game tic inside the thinker loop; each "on" run follows its "off" run):**

| build, switches on | level | off, mean (median, p95) | on, mean (median, p95) | spawn events off -> on | slow-path natives | full frames |
|---|---|---|---|---|---|---|
| e: tickentry, lightcalls, devirtualize, jit_lightframe (`sm183`-`sm186`) | Sunder | 112.74, 110.36 | 111.39, 108.60 | — | 121,268 both | 68,854 both |
| f: those four + directnatives + spawnevents (`sm205`-`sm208`) | Sunder | 104.39 (74.3, 160.4), 101.36 (72.4, 156.1) | 99.28 (71.1, 148.7), 94.69 (67.4, 143.3) | 20.3 -> 14.8 | 121,268 -> 13,521 | 68,854 both |
| f (`sm209`-`sm212`) | Abandon | 61.25 (48.5, 92.2), 59.88 (47.7, 89.6) | 56.82 (45.2, 84.0), 55.65 (44.5, 83.5) | 13.4 -> 9.5 | 72,263 -> 7,481 | 42,482 both |
| g: those six + jit_stringframe (`sm229`-`sm234`) | Sunder | 94.49 (67.6, 146.7), 104.44 (74.1, 161.0) | 84.94 (61.6, 126.5), 92.98 (66.8, 139.9); six only: 90.47, 95.13 | 20.3 -> 12.8 | 121,268 -> 13,521 | 68,854 -> 148 |
| g (`sm235`-`sm238`) | Abandon | 60.38 (48.0, 88.1), 60.69 (48.2, 89.8) | 53.30 (42.6, 77.3), 54.05 (43.3, 77.8) | 13.3 -> 8.4 | 72,263 -> 7,481 | 42,482 -> 147 |

- **Same game:** every off/on pair compared with `scripts\sim_synccmp.py` (world hash included): IDENTICAL through
  maptime 680 on both levels (`sm195`-`sm198`, `sm222`-`sm225`, `sm251`-`sm255`), including off against off.
- **Saving with all seven on (build g):** Sunder about -10.5 ms a tic (99.5 -> 89.0 over the two pairs, -10.5 %;
  p95 154 -> 133), Abandon -6.9 ms (60.5 -> 53.7, -11 %; p95 89.0 -> 77.5). By part, from the three builds: the four
  call-path switches of build e together about -1.5 on Sunder (inside the run-to-run spread); `vm_directnatives` and
  `sim_spawnevents` together about -6 on Sunder and -4.3 on Abandon, of which the spawn-event dispatch itself is -5.5
  and -3.9; `vm_jit_stringframe` about -4 on Sunder (it also takes another 1.6 ms off spawn events, whose handlers
  are string functions) and -2.6 on Abandon.
- **Trap, again:** Sunder's whole-loop mean moves by up to 10 ms between identical runs minutes apart (94.5 and
  104.4 for the same "off"). Only neighbouring runs compare; the counters (slow-path calls, full frames, spawn-event
  ms) are steady to three digits and are the numbers to trust.
- **What is left of spawn events** (12.8 ms a tic on Sunder) is the 45 handlers' own script: 2,742 spawns a tic, each
  offered to every handler, most of which test the actor and return. The engine cannot skip a handler without knowing
  what its script would do (F-MON-1's declared filters need the mod to declare them).

## 16. The owner's CURRENT mod list, re-measured (2026-11-23, `sm277`-`sm288`); the scratch change; the job system

**The mod list changed between 11-16 and 11-22** (risk R18 of the gameplan: re-measure before trusting a number).
Against the list of §14/§15: `1.2_BD22_VR-Reloaded_2.0.4x` became `1.2-BD22_VR-Reloaded_2.0.5d`; in: the Legion of
Bones randomizer and the twelve `RSQ_BDXMTiers_*` files; out: `4.01-Gameplay_CriticalHitsAmdHeadshots_mk-crits` and
Universal Entropy. The old load orders no longer load (the 2.0.4x file is gone). New ones: `SIM_SUNDER_1123_a.zdl`,
`SIM_ABANDON_1123_a.zdl` (58 files, copied from the render lane's `RENDER_ABANDON_1122_ownera.zdl`, which
`make_owner_zdl.py` read from his rotation folder). **Every mod-list number before this section is on the old list.**

Flags as §15; cfg `sim_m10_vm1116f.cfg`; builds `doomxr_sim1116g.exe` (before the job system and the scratch change)
and `doomxr_sim1122b.exe` (main dc50637). Raw: `measurements_raw/sm27*_m12_*`, `sm28*_m12_*`.

| run | level | switches | loop mean (median, p95) | script | spawn events | slow-path natives | full frames |
|---|---|---|---|---|---|---|---|
| g, `sm277`, `sm279` | Sunder MAP15 | ten on | 66.64 (44.6, 112.9), 69.53 (44.7, 120.4) | 52.2, 54.6 | 9.1, 8.9 | 11,680 | 151 |
| b, `sm278`, `sm280` | Sunder MAP15 | ten on | 66.34 (44.1, 113.2), 65.61 (44.4, 110.5) | 51.9, 51.3 | 8.7, 8.7 | 11,680 | 151 |
| b, `sm283` | Sunder MAP15 | seven of 11-16 off | 70.79 (46.8, 126.0) | 55.3 | 13.5 | 50,814 | 38,552 |
| g, `sm281` | Abandon MAP13 | ten on | 38.78 (29.3, 62.0) | 29.2 | 5.8 | 6,574 | 150 |
| b, `sm282` | Abandon MAP13 | ten on | 39.00 (29.9, 61.5) | 29.2 | 5.9 | 6,574 | 150 |
| b, `sm284` | Abandon MAP13 | seven of 11-16 off | 47.83 (34.8, 78.6) | 36.1 | 10.1 | 32,842 | 25,479 |

- **F-MT-1 scratch change (g against b): no cost, same game.** Sunder 68.1 against 66.0 as the mean of two runs
  each (inside the spread), Abandon 38.8 against 39.0; `sim_synccmp2.py ... nohash` IDENTICAL through 680 on both
  levels (`sm285`, `sm286`), and on against off within build b (`sm287`, `sm288`).
- **The seven switches of 11-16 on the current list:** Sunder 70.8 -> 66.0 (-4.8), Abandon 47.8 -> 39.0 (-8.8, 18 %).
- **Where the current list's 66 ms on Sunder goes** (`sm278`, 39,168 thinkers a tic, 274 classes):
  mk-champions about 28 (`champion_CyanController` 7.6, `champion_PinkController` 7.1 as a mean, `champion_Icon`
  4.8, `champion_Leaves` 2.1, the `champion_Aggro*` four 2.1, the other controllers about 4); the Balrog aura 8.9
  plus its gifts about 2.8; Legion of Bones monsters about 10; spawn events 8.7 (1,995 spawns a tic); everything
  else about 6. Script is 74 % of the loop. The loop's maximum is 552 ms (the Pink controllers' scan, every 70th
  tic). `RSQTHP_*` and `Shifter` are not in this list at all.
- **Job system on the owner's PC** (`sm260`-`sm266`): `job_selftest 1000 20000` identical hash at 8, 12, 15 and 16
  threads (and equal to the Linux rig's); the map alone x5.5 at 20,000 elements and x8.3 at 400,000 on 16 threads.

## 17. The per-tic reset: dense actor array, job system, used model slots (P9.9a, F-MT-3, F-SIM-5; 2026-11-23, `sm299`-`sm340`)

Owner's PC, current mod list, the ten earlier switches on, `sim_resetclock 1`, 300 tics a line (`sim_resetstats`).
Exes `doomxr_sim1123b.exe` to `e`. Sunder MAP15: 35,522 actors a tic, 2,154 with model data. Abandon MAP13: 24,911 / 2,258.

| reset, ms a tic | Sunder | Abandon |
|---|---|---|
| stock walk | 6.39, 6.79 | 5.01 |
| dense array, one thread (`sim_densereset 1`) | 6.30, 6.43 (copy 5.60, models 0.83) | — |
| dense + 15 threads (`+sim_parallelreset 1`) | 4.21, 4.18, 4.21 (copy 3.20, models 1.00) | 3.24 |
| dense + 15 threads + `sim_usedslots 1` | 3.58 (copy 3.35, models 0.23) | 2.39 (copy 2.23, models 0.16) |
| the same after the job system wakes all workers at once | 3.72 (copy 3.48) | 2.42 (copy 2.26) |
| verify mode (`sim_densereset 2`): mismatches | 0 | 0 |

- Same game: `sim_synccmp` IDENTICAL through 680 tics, off against on, both levels (`sm320`-`sm322`, `sm333`, `sm334`, `sm339`, `sm340`).
- The walk is not the cost: the dense array on one thread gains 0.1 to 0.4 ms.
- The calling thread copies 1,600 to 2,300 actors a tic, the 15 workers the rest, and the copy still takes 3.3 ms
  against 5.6 on one thread. Waking all workers at once changes nothing. Both levels move about 100 MB (Sunder) and
  70 MB (Abandon) of lines in 3.4 and 2.3 ms, the same rate: [I] the pass is bound by memory speed, which is what
  F-DOD-1 (hot prefix) addresses. Not yet proven; a run at `job_threads` 3 and 7 would settle it.
- **Settled 2026-11-23 (`sm343`, `sm344`, exe `doomxr_sim1123e.exe`, Sunder, dense + used slots):** reset 6.3 ms on one thread,
  4.09 on 4 threads (copy 3.88), 3.59 on 8 (copy 3.39), 3.6 to 3.7 on 16 (copy 3.3 to 3.5). Four threads give x1.5 and
  nothing is gained past eight: the pass is bound by memory speed.

## 18. The lights walk and the real-time walk on the dense array (P9.9c, F-SIM-6; 2026-11-23, `sm346`-`sm359`)

Owner's PC, current mod list (`SIM_SUNDER_1123_a.zdl`, `SIM_ABANDON_1123_a.zdl`), the earlier switches on, exe
`doomxr_sim1123f.exe` (UZDXREMA 5c54790), `sim_denseclock 1`, about 300 runs a line (`sim_densestats`), cfg
`sim_m14_dense1123a.cfg`. Timing runs; `perf_bench_hash` not set.

| lights pass (`RunClientSideThinkers`' walk), ms a client tic | Sunder MAP15 (35,587 actors, 110 with work) | Abandon MAP13 (24,945, 145 with work) |
|---|---|---|
| stock walk (`sim_denselights 0`) | 3.70 | 2.30 |
| dense array, one thread (`1`) | 0.91 (finding 0.82) | — |
| dense array, search on 15 threads (`+sim_parallelscan 1`) | 0.69 (finding 0.60) | 0.45 (finding 0.38) |
| verify mode (`2`): mismatches | 0 | — |
| whole client-side loop (`simprof client`, mean) | 3.98 -> 0.96 | 2.60 -> 0.74 |

| real-time pass (`P_TickRealTimeActors`), ms a skipped tic | Sunder + `tools/sim_lane/test_fsim6` (38,740 actors, 78 found) |
|---|---|
| stock walk (`sim_denserealtime 0`) | 1.96 (finding 1.87) |
| dense array, search on 15 threads (`1`, `sim_parallelscan 1`) | 0.70 (finding 0.51) |
| verify mode (`2`): mismatches | 0 |

- Same game: `sim_synccmp` IDENTICAL through 680 tics, off against verify (Sunder, `sm352`) and off against on (Abandon,
  `sm355`); the slow-motion pair the same for every line both logs have (`sm359`; the world reaches fewer than 680 tics).
- Here, unlike the reset, the walk was the cost: the stock loop follows one link per actor to do nothing for all but
  about a hundred of them. What is left is one cache miss an actor (about 20 ns), which threads barely help.
- The checklist's "about 9 ms" for the real-time walk was an estimate; the stock walk measures about 2 ms at 38.7k actors.
