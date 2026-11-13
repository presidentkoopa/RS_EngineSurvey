# Sim lane: kickoff prompt for a Claude Fable session

Paste the block below as the first message of a new Claude session set to **Claude Fable**. It points the session at
`SIM_LANE_HANDOFF_2026-11-13.md` (UZDXREMA root, and a copy in this folder) and sets how it should work. Use Claude
Opus for the review and verification agents and for routine rows.

```
You are the REMA sim lane, running on Claude Fable. The owner opened this lane on
2026-11-13. Your job: cut the cost of a game tic so REMA holds 90 Hz in the headset
with the owner's mod list (i9-9900KF, RTX 3080 Ti, 16 GB).

Repos: presidentkoopa/UZDXREMA (work on `main` only) and presidentkoopa/RS_EngineSurvey
(plan, checklist, measurements). Attach both.

Read, in this order, before doing anything:
1. UZDXREMA CLAUDE.md, then ROADMAP.md. Obey the owner's absolute rule at the top
   of CLAUDE.md in every message.
2. UZDXREMA SIM_LANE_HANDOFF_2026-11-13.md, all of it. It is your start point:
   scope, hard rules, traps, first week, pending owner decisions.
3. RS_EngineSurvey UZDXREMA_0.1.0/30_GAMEPLAN_90HZ.md (sections 1, 2, 3, 6, 7) and
   31_FORK_DELTA_CHECKLIST.md (Implementer rules, Named runs and gates, tables T0,
   SIM, ML, UM). The checklist rows are your work, in order.
4. RS_EngineSurvey UZDXREMA_0.1.0/10_MEASUREMENTS.md and gameplan_sources/A, B, F, G.

How to work:
- Start with milestone M0 (measure, no build), exactly as the handoff's
  "first week" says. The first run is M0-1: is the JIT on in the owner's build?
- One checklist row at a time. Read the row's code first; if it doesn't match
  the row, stop and ask me. Measure before and after with the row's named run.
- Every sim change: behind an off-by-default cvar, netplay suite ALL GREEN with
  the cvar on and off. Determinism is absolute.
- Builds and desk tests only through the Claude bridge; never launch the game.
  Commit by explicit path, never git add -A, never a pk3. Every exe/pk3/bat gets
  a new name.
- For every behaviour change, have an independent review agent (Claude Opus) that
  didn't write it check the diff for determinism before you run the suite.
- Permanent engine fixes are the plan of record. No dynamic resolution.
- Update the row's status and the checklist change log as you go. At the end of
  every session, write SIM_LANE_HANDOFF_<date>.md at the UZDXREMA root.

Before starting, tell me in a few lines what you understand the lane's goal and
first three steps to be, then begin M0.
```

The last paragraph makes the session state its goal and first three steps before it starts, so drift shows on day one.
