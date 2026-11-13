# Fork-delta brief (shared by all agents in this swarm)

Goal (owner's words): "I want 90 Hz on my modlist." Produce the material for a detailed gameplan that gets the owner's
engine UZDXREMA (UZDoom 5.0 base + DoomXR VR, Vulkan-only, OpenXR) to a steady 90 Hz in the headset with his
57-file mod set. Nothing is impossible on his engine, only costed. Be concrete: files, functions, lines, steps.

## Sources (all READ-ONLY; never commit, checkout, reset, clean, or launch anything)
- [F]  fork main:      /home/claude/uzdxrema            @ e84623f  (engine of record)
- [FR] fork renderer:  /home/claude/uzdxrema_renderer   @ abb19af  (render lane: Helion-style static level path behind a cvar)
- [U]  stock UZDoom:   /home/claude/src/uzdoom          @ 5.0.0 (292cf420)
- [H]  Helion:         /home/claude/src/helion          @ 5edd437
- [S]  owner's teardown of his fork: /home/claude/rs_enginesurvey/UZDXREMA_0.1.0/00..10_*.md (cross-checked by his team,
       but written at main f182cbcc; main has moved to e84623f — re-verify any line you rely on).
- [M]  owner's measurements: /home/claude/rs_enginesurvey/UZDXREMA_0.1.0/10_MEASUREMENTS.md (RTX 3080 Ti, i9-9900KF,
       flat screen 3056x1689, old render path). Key: modlist Sunder MAP15 frame 315 ms, sim 250 ms, thinkers 130.8 ms/tic,
       interp reset 9.0 ms/tic, WorldTick handlers 4.1, 53,116 thinkers; GPU wait 25.9 ms (opaque 6.15, exposure 1.38,
       bloom 3.23, BloomBoost mod 1.60+1.64); no-mod frame 13.9 ms; r_ticstability busy-wait 2.3 ms/frame.
- [P]  the reviewed Helion-pathway playbook for STOCK 5.0: /home/claude/rs_enginesurvey/Helion_1.2.0/20_REBUILD_PLAYBOOK.md,
       21_VR_VULKAN_SPECIFICS.md, 22_MEASUREMENT_PLAN.md, plus teardown 01-06 and UZDoom_5.0_counterparts/U1-U3.
- Fork rules: /home/claude/uzdxrema/CLAUDE.md and ROADMAP.md, RENDER_LANE_HANDOFF_2026-11-09.md. Respect them in your
  recommendations: roadmap lanes (avatar lane on main; render lane on `renderer` behind an off-by-default cvar; the model/avatar
  draw path untouched by the render lane), four-player co-op netplay with bit-for-bit replays as the regression suite
  (ANY sim change must be deterministic across machines and keep tools/netplay replays green), all controller data into the
  sim through usercmd_t, "the renderer may only draw".

## Rules
1. Only trust code. Cite every fact: `[F] path:lines`, `[FR] path:lines`, `[U]`, `[H]`. Survey/playbook text is a lead, not evidence.
2. Label your reasoning **Inference**. Every saving estimate must show its derivation from [M] numbers (e.g. "9.0 ms/tic x ...").
3. Where the playbook (written for stock 5.0) says to change X, find whether the fork already did it, did it differently,
   or moved the code. Fork-only code that matters for perf (DoomXR, avatar/IK, holsters, BD22 world weapons, Radiance, etc.)
   must be included even if the playbook doesn't mention it.

## Output: one markdown file at the path you are given, containing
A. `## Delta table` — one row per playbook item in your area plus fork-only items:
   | ID | Item | Fork status (Done/Partial/Not started/Not applicable) | Where in fork ([F]/[FR] path:lines) | What is left (1-3 sentences) | Done when (measurable) | Don't touch | Netplay/determinism impact | Est. saving on modlist (Inference, with derivation) |
   Use playbook IDs (e.g. P0-F1, P0.5, P1.3, P9.2, P10.1) or F-<area>-n for fork-only items.
B. `## Findings` — the evidence behind the rows, including fork-specific hot spots, with citations and short excerpts.
C. `## 90 Hz contribution` — what your area must deliver for 90 Hz in the headset on the modlist (per-tic ms or per-frame ms
   budget for your area), what the fork has today, and the gap.
D. `## Open questions` — what needs a measurement on the owner's PC (name the exact boottest/cvar/command run).
Aim for 3,000-7,000 words, dense. Reply with the path and a <=250-word summary of the biggest findings.
