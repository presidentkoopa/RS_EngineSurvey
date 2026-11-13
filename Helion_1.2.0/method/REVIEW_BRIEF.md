# Adversarial review brief (playbook)

You are an independent senior engine reviewer (Vulkan, OpenXR, GPU-driven rendering, ZDoom-family playsim). You did NOT write the file you review. Assume it has errors. Your bar: a demanding expert would hand this to a senior engine programmer as-is.

Repos (READ-ONLY, shared — never checkout/reset/clean): Helion /home/claude/src/helion @5edd437; UZDoom /home/claude/src/uzdoom @5.0.0 (292cf420).
Verified research the playbook builds on: /home/claude/work/sections/H*.md, U*.md (and correction logs /home/claude/work/verify/*.log).
Owner measurements: /home/claude/rs_enginesurvey/UZDXREMA_0.1.0/10_MEASUREMENTS.md (read-only; do not edit that repo).

Check, and FIX IN PLACE with the Edit tool only (never rewrite the whole file with Write; other reviewers work on sibling files):
1. Every [U]/[H] citation: open it; line numbers right; the code says what the text says. The edit instructions ("change X at file:line to Y") must be correct against the actual code — if an instruction would not compile or would break behaviour, fix it.
2. Technical correctness of the design: Vulkan/OpenXR/multiview API facts (e.g. multiview limits, XR_KHR_composition_layer_depth, frame loop semantics, descriptor indexing requirements), GPU culling/Hi-Z correctness, tic-rate scene build + head-motion visibility, determinism/demo/netplay implications of sim changes, ZScript compatibility claims. Wrong → fix; uncertain → add an explicit "Open risk:" line.
3. Internal consistency: phase dependencies and order; numbers consistent with the measurement file and the research sections; no contradictions between sections.
4. Gaps: anything a senior implementer would need that is missing (e.g. a migration step, a fallback, a test). Add it concisely where it belongs.
5. Bloat: cut repetition; keep every substantive instruction.
Keep labels (Inference / API fact / sketch / [M]). Log every change (old → new, why) to /home/claude/work/verify/<ID>.log with totals at the end.
Reply with: totals, the 5 most important fixes, and any open risks you could not resolve.
