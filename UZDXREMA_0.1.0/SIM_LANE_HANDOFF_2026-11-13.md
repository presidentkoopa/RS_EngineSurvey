# SIM LANE HANDOFF 2026-11-13: 90 Hz on the owner's mod list, the simulation side

The owner opened this lane on 2026-11-13 (gameplan decision **D1 = (a)**: open a sim lane now, on `main`). It is the
third lane, beside the avatar lane (ROADMAP steps 1-3, `main`) and the render lane (step 4, `renderer`). Its job is the
one thing standing between REMA and 90 Hz on the owner's mod list that no other lane owns: **the cost of a game tic.**

Written from the cloud session that produced the Helion teardown, the 90 Hz gameplan and its checklist. Every fact
below is cited in those documents; nothing here was compiled or run.

---

## Step 0: done

This file and the sim-lane bullet in `CLAUDE.md` were committed to `main` on 2026-11-13 by the session that
wrote them. Bring the owner's copy up with `scripts\pull_main.py` through the bridge (`tools/claude_bridge/README.md`)
if it is behind. The canonical copy of the plan and checklist stays in RS_EngineSurvey `UZDXREMA_0.1.0/`.

---

## Read in this order

1. UZDXREMA `CLAUDE.md`, then `ROADMAP.md`. The owner's absolute rule is at the top of `CLAUDE.md`: obey it in every
   message.
2. This file, all of it. Most of it is about what is **not** yours and what bites.
3. RS_EngineSurvey `UZDXREMA_0.1.0/30_GAMEPLAN_90HZ.md`: §1 (verdict), §2 (budget), §3 (the sim-tic waterfall, the
   double-counting rules, the unknowns, which monster classes tick), §6 (tracks, milestones M0-M14), §7 (owner
   decisions D1-D17).
4. `31_FORK_DELTA_CHECKLIST.md`: the **Implementer rules**, **Named runs and gates**, then tables **T0**, **SIM**,
   **ML**, **UM**. These rows are your work, in order. Each row has its lane tag, the code (`path:lines`), what to
   do, an exact "Done when" run with a number, a Don't-touch list and its netplay gate.
5. `10_MEASUREMENTS.md` (the baseline every number starts from) and `gameplan_sources/A_sim.md`, `B_modlist.md`,
   `F_monster_stack.md`, `G_use_the_machine.md` (the evidence behind your rows).
6. UZDXREMA `tools/claude_bridge/README.md` ("Current state": how builds and boot tests reach the owner's PC),
   `tools/netplay/README.md` and `tools/linux_rig/README.md` (your gate), and `HANDOFF_NETPLAY_2026-11-11.md` (the
   netplay lane's open items; item 1 overlaps your M1).

---

## The goal, in numbers

The owner's machine: i9-9900KF (8 cores / 16 threads), RTX 3080 Ti, 16 GB. Headset at 90 Hz: 11.1 ms per frame.

| | Today | Target | Plan's projection (pess / med / opt) |
|---|---|---|---|
| Sim tic, 4 players, mod list, Sunder MAP15 | ≈160 ms (≈145 single player) | **median ≤ 20 ms, p95 ≤ 28.6 ms** on its own core | 65.1 / 33.5 / 12.7 ms |
| Thinkers (count) | 53,116 | ≤ 25k after M4 | ≈22k |

The tic budget is 28.6 ms (35 Hz). Today thinkers alone take 130.8 ms of it, the interpolation reset 9.0 ms, and
`WorldTick` handlers 4.1 ms (`10_MEASUREMENTS.md` §2). With the mod set the renderer is about 2% of the frame; the
simulation is about 80% (§1). **The sim is the bottleneck, and it is yours.**

The median does not close on the base plan. What closes it (gameplan §3.5): at least 30 ms of the ≈78 ms of thinker
time no profiled class explains turns out to be removable mod work, or 20 ms of it plus parallel thinkers at p ≥ 0.7
and AOT speed-up k ≥ 2.2, or the owner opts into Tick LOD (D9). **Your first job is to find out which.**

The sim thread itself (running `TryRunTics` off the headset frame) is milestone M11 and belongs to the render lane's
stage 6 with the netplay lane co-signing. You make it possible by getting the tic under budget, and by never adding a
render→sim read or write.

---

## What is yours, and what is not

**Yours (checklist tables T0, SIM, ML engine half, UM; milestones M0, M3, M4, M5, M10, M13):**
- measurement and read-only sim tooling (`sim_profile`, `vm_profile`, spawn-event bucket, allocation and action
  counters, JIT startup report);
- the exact engine rows: dense actor array and interpolation reset, word-wise netgame hash, `CallTick`/`CallAction`
  call path, dormancy with a verify mode, BSP point grid, sight sort and cache, portal render-list key, GC pacing;
- the mod-list engine mechanisms: native per-actor overlays and attachments, parked inventory, native sound-pitch and
  state-tic scale (deletes `Shifter`), spawn-event filters, client-side promotion of effect classes (with the RNG and
  event guards), gore pools with caps, translation fix, O(1) `EventHandler.Find`;
- the use-the-whole-machine track: job system, parallel engine passes, VM call path, then AOT and deterministic
  parallel thinkers under `sim_rule 1` (only after D3).

**Not yours. Hand these to their owner through the owner, never edit them yourself:**
- **Avatar lane:** `src/r_data/models.cpp`, `src/r_data/vr_*.cpp/h`, `hw_sprites.cpp`, `hw_weapon.cpp/h`, the
  `VRRig_*` calls in `hw_drawinfo.cpp`, and the controller/grip half of `vk_openxrdevice.cpp`.
- **Netplay lane:** `TryRunTics`, `MakeConsistencies`, prediction, `d_protocol.h`, `hw_vrmodes.h/.cpp`
  (`SuppressVRInput`), `hw_vrwheel.cpp`, anything writing `handinput` or the `xrLast*`/`xrNet*` edge states. You may
  propose changes there (the word-wise hash F-SIM-4 sits in `MakeConsistencies`), but the netplay lane co-signs.
- **Render lane (`renderer`):** everything on that branch, the XR frame loop, GPU, multiview, frames in flight.
- **Owner:** his ini, his mod pk3s, the RSQ uplift (D14) and the RSQ_Monsters rewrite (D17). Mod fixes you design
  ship as **compat copies** (`*_REMA.pk3`, `tools/netplay/compat/README.md`) only with his yes, and are labelled
  STOPGAP when an engine row will replace them.

A symbol read by `g_game.cpp`, `d_net.cpp`, `p_user.cpp` or ZScript is sim, whatever folder it sits in.

---

## Hard rules

1. **Branch `main` only.** No new branches unless the owner asks. The avatar lane also works on `main`: pull with
   rebase before every push, keep commits small, and never touch its files.
2. **Until ROADMAP step 3 freezes the netplay suite:** you land read-only tooling freely; every behaviour change sits
   behind a cvar that is **off by default**. Flipping defaults comes after the freeze, one row at a time.
3. **Netplay gate for every sim change:** `bash tools/netplay/suite.sh` ALL GREEN on the Linux rig, once plain and
   once with `EXTRA="+<cvar> 1"`; `NETSET=1` when the net mods are involved. Exact rows also need GATE-BENCH
   bit-identical. The suite is the owner's pride; it does not go red on your watch.
4. **Determinism is absolute.** No `consoleplayer`, no real time, no thread timing, no hash-map iteration order, no
   uninitialised memory reaching sim state. Parallel work must give the same bits with 1 thread or 12.
5. **One checklist row at a time.** Read its code first. If the code doesn't match the row, **stop and ask**; don't
   adapt the row. Measure before and after with the row's named run. Update its Status and add one dated line to the
   checklist's change log.
6. **Builds and tests go through the owner's PC via the Claude bridge.** Desk tests only through `boottest` or
   `benchreplay`. Never launch the game. Never rebuild to test where the Linux rig works. Every exe, pk3, bat or cfg
   the owner may test gets a new name (`python scripts\build_named.py <name>`, `scripts\pack_as.py`).
7. **Commit by explicit path. Never `git add -A`. Never commit a pk3.** Keep `E:\DOOMWork\UZDXREMA` and GitHub equal.
8. **Permanent engine fixes are the plan of record** (owner decision 1). A workaround is a STOPGAP row with a removal
   condition, never a reason to skip the permanent row.
9. **Dynamic resolution is not a solution** (owner decision 6). It is not in your toolbox.
10. **Memory:** anything that adds memory states its footprint against 16 GB shared with the OS, the OpenXR runtime
    and 57 mods (owner decision 8).

---

## Facts that will bite you (all verified in code; details and lines in the checklist and gameplan)

- **`main` is slower than the build the owner measured.** His numbers come from a `renderer` exe that has the
  per-source sound index, the evicted-sound restart budget, `sim_maxcatchup` and `ticprofile`/`simprofile`. On `main`,
  `ChangeSoundPitch` walks every sound channel; with Universal Entropy's 7,686 `Shifter`s calling `A_SoundPitch` twice
  a tic that is an estimated ≈280 ms/tic instead of 2.2. Merging those commits is M1, and the netplay lane's item 1.
  Agree with it, through the owner, who does it. Until then, never time anything on a `main` exe without saying so.
- **`sim_maxcatchup` only applies when `!netgame`.** It does not protect 4-player games.
- **`profilethinkers` clocks only `CallTick`.** `CallPostBeginPlay` (where `WorldThingSpawned` fires) runs before the
  per-class clock, so 43 spawn handlers at ≈10-20 µs per spawned actor are invisible. That is part of the 78 ms. M3's
  spawn-event bucket fixes the blind spot.
- **The command is `profilecsthinkers`**, not `csprofilethinkers`.
- **`r_ticstability` busy-waits** on every frame without a tic, for as long as the last tic took (up to 1 s), and VR
  doesn't switch it off. Its removal belongs to the XR loop (M6, render lane), but every desk number you take is
  polluted by it unless `r_ticstability 0` is set: say which you used.
- **4-player games hash every actor byte by byte each tic** (`WorldStateHash`): ≈8-12 ms per tic at 53k actors,
  missing from every measurement because all were single player. F-SIM-4 fixes it.
- **The owner's own mods write `Actor.Prev`** (HP bar, damage numbers, QuestSaber), so a lazy interpolation reset
  needs the codegen hook in P9.9b, not just a dirty flag.
- **`RSQTHP_Bar` counts spawn health as full health.** Universal Entropy rescales health at spawn and Champions and
  LegenDoom raise it, so almost every monster gets a bar before any fight: 3,622 bars, 16.5 ms/tic. It also reads
  `consoleplayer` in play scope (a latent desync with `rsqthp_aimed 1`). D8 covers the fix.
- **Which monsters tick is not settled.** BD22 and XMVE always load and stay live for gore, decorations, items and
  weapons. For each vanilla monster type, Legion of Bones' `RandomSpawner` is the last replacement among them and
  spawns its class with `NO_REPLACE` (the measured 1,724 `SkeletonImp` are the map's imps). But RSQ_Monsters loads
  after LoB, and its roster handler at default would swap in Colorful Hell monsters, and XMVE's `CyberdemonSpawner`
  still runs. Run M0-2 / U10 before you trust any per-class number. RSQ_Monsters is due for a rewrite (D17).
- **Universal Entropy leaks translations:** one per monster, never freed, and the engine stops at 65,535, so a session
  aborts after ≈9-10 loads of Sunder MAP15. F-MON-6 is the permanent fix; D7 has the stopgap.
- **LegenDoom's map-start roll** creates ≈45,000 items in one tic (a 0.5-1 s hitch), which `sim_maxcatchup` can't
  cover in a netgame. D13.
- **The fork lowered gore caps:** `sv_corpsequeuesize` 10 (stock 64), `cl_maxdecals` 20 (stock 1024). D6 recommends
  96 and 128, identical on all four machines.
- **Controller data still leaks into the sim outside `usercmd_t`** (grip/trigger/stick/`VRTurnYaw` on the pawn,
  `VRMode::SetUp`'s `multiplayer` gate, `Level.GetRawStickMove/Turn` from play scope). That is M9 and belongs to the
  netplay and avatar lanes (owner decision 4), but **don't build anything that reads those values**.
- **The JIT's state in the owner's build is unknown.** `vm_jit` defaults on, but nobody has checked the startup log.
  If it is off, turning it on alone may save 30-60 ms/tic. M0-1 is your first run.
- **Netplay suite traps:** 11 plain scenarios, 13 with `NETSET=1`; the `netset`/`netset_room` scenarios overwrite
  `EXTRA` (suite.sh:160, 166), so a cvar never reaches them that way: ask the netplay lane before claiming a NETSET run
  with your cvar on. RUN-SUNDER needs `-Iwad`, or the boot test loads Ultimate Doom's `doom.wad`.

---

## Your first week, in order

**Day 1-2: M0, measure (no build).** Checklist rows M0-1, M0-2, M0-5, M0-7, M0-9, M0-11 are yours; M0-3, M0-6 and M0-8
need the owner in the headset or at his PC (ask him); M0-4 and M0-10 are GPU rows (hand the numbers to the render
lane).
- M0-1: is the JIT on? (`vm_jit`, `vm_debug`, `stat vm`, startup log line.)
- M0-2: the full per-class list (`profilethinkers 8 0` on a `renderer` exe via RUN-SUNDER), `ticprofile`, and which
  monster set ticks (U10).
- M0-7: `main` vs `renderer` on Abandon MAP13 (proves the sound-index gap).
- M0-9: a fight-load capture, not just the map start.
- M0-11: the translation count over 10 loads.
File every value in RS_EngineSurvey `10_MEASUREMENTS.md` (new section) and fill the unknowns in gameplan §3.6.

**Day 2-3: decide what closes the gap.** With M0-2 in hand, recompute the waterfall (gameplan §3.3) and tell the owner
plainly which closer applies. Re-rank the rows in M4 and M5 by measured ms.

**Day 3-5: M3, sim tooling on `main` (read-only, lands freely).** P0.5-1 `sim_profile` (continuous per class), P0.5-2
`vm_profile` (per function, per `WorldTick` handler), the spawn-event bucket, P0.5-3/4/5 counters, P0-F13 JIT startup
report. Gate: per-class sum plus spawn bucket within 5% of `thinkers`, and `WorldTick`'s 4.07 ms split by handler.
From here on, every row's "before" and "after" comes from these tools.

**Then, in checklist order:**
- **M4, the mod-list engine mechanisms** (largest measured win): F-MOD-8 parked inventory, F-MOD-9 native pitch and
  tic scale (deletes `Shifter`), P10.1 overlays with the RSQTHP rewrite path, P10.2 attachments, P10.5 per-class
  lists, F-MON-1 spawn-event filters, F-MON-2 client-side effects with the RNG and event guards, F-MON-3 gore pools,
  F-MON-6 translations. Gate: thinkers ≤ 25k, helper families ≤ 5 ms/tic, `RSQTHP_Bar` ≈ 0 at map start.
- **M5, the exact engine rows**, each GATE-BENCH bit-identical: P9.9a dense array + reset Step A, P9.9b codegen hook,
  F-SIM-4 word-wise hash (netplay co-signs), P0-F4 `CallTick`, P0-F1 `CallAction` copies, P9.1b thunks, P0-F2 portal
  key, P0-F5 stable sight sort, P9.2 dormancy with `sv_dormancy_verify`, P9.4 BSP grid, P9.5 inline blockmap, P9.10
  GC pacing.
- **M10 and M13, use the whole machine:** the job system, parallel engine passes, the VM call path; then AOT for the
  fixed mod set and parallel thinkers under `sim_rule 1`, **only after the owner's final yes on D3**.

---

## Owner decisions: taken and pending

**Taken (binding, `gameplan_sources/OWNER_DECISIONS_2026-11-13.md`):** permanent engine fixes; netplay is the gate;
the RSQ series is uplifted into REMA, and Radiance stays, maxed out; controller data fixed at engine level; multiview
fixed; no dynamic resolution; use the whole machine; 16 GB budget; RSQ_Monsters to be rewritten; **D1 = (a)**.

**Pending; don't act ahead of them:** D2 (merge the renderer sim fixes now: who does it), D3 (`sim_rule 1`), D6 (gore
caps), D7 (Universal Entropy), D8 (RSQTHP redesign), D9 (Tick LOD), D10 (mk-champions compat copy), D13 (LegenDoom
roll), D14 (RSQ uplift rules), D15 (content density), D17 (RSQ_Monsters standard). Bring each to the owner with the
gameplan's recommendation when your row needs it, not before.

---

## How to work so you don't drift

- The checklist is the plan. If you think a row is wrong, say so and propose the change; don't quietly do something
  else.
- Every claim you make cites code (`path:lines`) or a run id. Your reasoning is labelled **Inference**.
- For every behaviour change, have an **independent agent** that didn't write it re-read the diff for determinism
  (RNG order, iteration order, uninitialised state, `consoleplayer`) before you run the suite.
- At the end of every session, write `SIM_LANE_HANDOFF_<date>.md` at the UZDXREMA root: rows done (with before →
  after and run ids), rows in progress, what you learned that the next session must know, and the decisions you
  asked the owner for.
- Suggested models: **Claude Fable 5.2** for the hard implementation (the dense array, dormancy, the job system,
  AOT, `sim_rule 1`); **Claude Opus 5.5** for review and verification agents and for routine rows.

## When this lane is done

M14, sim half: on Sunder MAP15 and Abandon MAP13 with the full net set, 4 players, a 10-minute fight, every peer
shows a tic median ≤ 20 ms and p95 ≤ 28.6 ms, and the four-machine suite is ALL GREEN. Then REMA's sim can live on
its own core, and the headset never waits for it.
