# Verify-and-publish brief

You are an independent verifier. You did NOT write the section you are checking. Assume it contains mistakes.

Repos (READ-ONLY, shared — never checkout/reset/clean):
- Helion: /home/claude/src/helion @ 5edd437f39ddcbcd647a305d7bdd306847d6686b
- UZDoom: /home/claude/src/uzdoom @ tag 5.0.0 (292cf4203ebd3ced951cb67f6819180f588c1d44)

## Step 1 — Verify
For EVERY citation `[H] path:a-b` / `[U] path:a-b` in the section:
- Open those exact lines. Does the code there actually support the sentence it's attached to? Line numbers off by a few → fix them. Wrong file/claim → correct the claim from the code or delete it.
- Check every code excerpt matches the file verbatim at the cited lines (whitespace aside).
- Spot-check every "Absent" claim with your own grep (search more broadly than the author did: alternate names, other dirs).
- Check numbers (struct sizes, counts, defaults of cvars/config) against code.
- Inference-labelled reasoning: leave it unless it contradicts the code; if it does, fix it and say so.
Edit the section file IN PLACE with your fixes. Append a log to /home/claude/work/verify/<SECTION_ID>.log listing each correction (old → new, why) and the totals: citations checked, corrected, removed.

## Step 2 — Publish into the Claude Doc
Doc container: {"kind":"project","id":"e1182b29-7e7b-4865-8fea-c5667041418c"}; prose node: {"object":"node","id":"a00666f7-9033"}, engine "prose".
Tools: mcp__Claude_Docs__update (load via ToolSearch "select:mcp__Claude_Docs__update" if needed).
Replace your assigned pending block (id given to you) with the verified section as markdown, in ONE update call:
payload = {"ops":[{"op":"replace","target":{"kind":"blocks","ids":["<PENDING_ID>"]},"with":{"as":"markdown","from":{"kind":"inline","content":"<markdown>"}}}]}
Conversion rules for the doc version:
- Top heading becomes `## <title>`; its subsections become `###`, deeper become `####`. No `#` level-1 headings.
- Turn every citation into a clickable link at the pinned commit, keeping the visible label:
  [H] path:a-b → [`path:a-b`](https://github.com/Helion-Engine/Helion/blob/5edd437f39ddcbcd647a305d7bdd306847d6686b/path#La-Lb)
  [U] path:a-b → [`path:a-b`](https://github.com/UZDoom/UZDoom/blob/292cf4203ebd3ced951cb67f6819180f588c1d44/path#La-Lb)
  (single line → #La). Prefix the label with H: or U: so readers know which engine, e.g. [`H: Core/Render/Renderer.cs:10-20`](...).
- Code excerpts in fenced blocks with language (```csharp / ```cpp / ```glsl).
- Keep ALL substantive content; you may tighten wording. Keep "Inference:" labels.
- If the update is refused, read the refusal, fix and resend. If the content is too large for one call (refused for size), split: replace the pending block with the first half, then `insert` the rest with "side":"after" the last block id returned in the ack.
Do not touch any other block in the doc.

Reply with: corrections count, the 3 most important corrections, and confirmation the doc block was replaced.
