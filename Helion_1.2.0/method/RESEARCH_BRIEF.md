# Shared brief for all research agents

Project: a code-level teardown of the Helion Doom engine (C#/.NET, OpenGL) and a comparison with UZDoom 5.0 (C++),
feeding a plan to rebuild UZDoom 5.0 into a high-performance PC VR engine (Vulkan, stereo 90-120 Hz, modern GPUs).
The user owns his fork outright: nothing is off the table, so never write "can't be done"; write what it costs.

Repos (READ-ONLY, shared with other agents — never run git checkout/reset/clean in them; use `git show <rev>:path` if you need another revision):
- Helion: /home/claude/src/helion  @ 5edd437f39ddcbcd647a305d7bdd306847d6686b
- UZDoom: /home/claude/src/uzdoom  @ tag 5.0.0 (292cf4203ebd3ced951cb67f6819180f588c1d44)

Rules:
1. ONLY TRUST CODE. READMEs, .md, .adoc, comments describing intent are hints, never evidence. If a comment claims something, check the code does it.
2. Cite every factual claim as `[H] path:start-end` or `[U] path:start-end` (repo-relative paths, real line numbers you looked at with `grep -n`/`sed -n`/Read). No citation = don't claim it.
3. Read whole functions before describing them. Don't infer behaviour from names.
4. Separate facts from reasoning: label your own performance reasoning as **Inference:** (e.g. cost estimates, why something is cache-friendly). Facts carry citations.
5. Short code excerpts are welcome (<= 15 lines each, <= 8 per section), always with their citation.
6. Note what is ABSENT in your area (features/techniques the engine does not do) only after a code search proves it; say what you searched (`grep -rn "xyz"` found nothing).
7. Record cvars/config options that change performance in your area, with their defaults (cited).

Output: write ONE markdown file at the path you are given. Structure:
- `# <Section title>` then a 3-5 sentence lead stating the main finding.
- `## <Mechanism>` subsections: what it does, key types/functions, excerpts, data flow per frame/tic, why it's cheap or expensive (Inference labelled).
- `## Absent / not done` (proven by search).
- `## Perf-relevant settings` (cvar/config, default, citation).
- `## Transfer notes for UZDoom VR` — for each mechanism: the UZDoom counterpart (cite if you looked), grade **Port** (local change), **Rework** (restructure subsystem, interfaces survive) or **Rebuild** (replace subsystem), and what stereo/Vulkan changes.
- `## Open questions` — anything you couldn't verify.
Aim for 3,000-7,000 words of dense, specific content. No filler.

When finished, reply with: the file path, and a <= 250-word summary of your top findings (ranked by performance impact).
