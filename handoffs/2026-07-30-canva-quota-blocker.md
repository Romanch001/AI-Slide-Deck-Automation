# Handoff: Wire Rope 40-Slide Canva Deck Rebuild

## Goal
Convert 40 NotebookLM-generated slide images (Chapter 23, Wire Ropes/Sheaves/Drums, V.B. Bhandari "Design of Machine Elements") into a genuinely **editable Canva design**, visually as close to the NotebookLM originals as possible. Final deliverable: one merged 40-page Canva design.

## Hard requirements from user (do not deviate without re-confirming)
- **Method must be Canva's own image→editable-design AI conversion** (the same thing the user's ChatGPT+Canva connector does when told "make this image editable") — **NOT** an HTML file authored by Claude and imported via `import-design-from-url`. User explicitly rejected the HTML-import approach this session after discovering a prior session built Part 1 that way ("current presentation in canva look very bad. I want exact as in from notebooklm").
- Dense data-table slides (Tables 23.3–23.9, fatigue diagram, quick-reference sheet) must be **static image placement only** — no AI recreation. Numbers must not be regenerated/reinterpreted by AI (100% accuracy mandate). Confirmed via AskUserQuestion this session.
- No NotebookLM watermark may be cropped/painted out of source images before upload — user removes watermarks themselves after conversion (standing rule, see project memory `feedback_notebooklm_watermark.md`).
- 100% data accuracy required throughout — nothing fabricated, rounded, or mislabeled.

## The correct tool pipeline (confirmed via tool schema inspection this session)
For AI-recreated editable slides:
1. `mcp__claude_ai_Canva__upload-asset-from-url` — upload the source slide PNG (must already be a public URL) → returns `asset.id`.
2. `mcp__claude_ai_Canva__generate-design` with `asset_ids: [that asset id]`, a `query` describing the slide, `design_type` (NOTE: `generate-design` explicitly does **not** support `design_type: "presentation"` — used `"poster"` as the closest single-canvas analog this session, untested for actual output quality since it never got past the quota error).
3. Poll `mcp__claude_ai_Canva__get-design-candidates` with the returned `job_id`.
4. `mcp__claude_ai_Canva__create-design-from-candidate` with `job_id` + chosen `candidate_id` → returns a real editable Canva `design_id`.

For static-image (table) slides: `upload-asset-from-url`, then need a blank canvas to place it on. **No plain "create blank design" tool exists** in this Canva MCP surface — every option (`generate-design`, `create-design-from-brand-template`, `import-design-from-url`) either goes through the blocked AI quota or reintroduces the rejected HTML path. This is unresolved — see Open Problem below.

Final assembly: `mcp__claude_ai_Canva__merge-designs` (`type: "create_new_design"`, `insert_pages` operations pulling in order from each per-slide design) to combine all 40 single-page designs into one deck.

## Blocker hit this session (unresolved)
`generate-design` returns **"User has reached their quota limit"** immediately, on every call, regardless of query content (tested both a detailed descriptive prompt and the literal short prompt "Make this image editable" per the user's own suggestion — same instant error both times). This is almost certainly the Canva↔Claude **app-level** API quota (separate from the user's own Canva account, and separate from whatever quota bucket their ChatGPT+Canva connector uses — which is why the same "make it editable" trick works for them in ChatGPT but not here). Not something fixable from this session. A prior session's Part 1 build agent hit this same wall.

**User's decision (this session):** wait and retry later, rather than switch strategy now. Next session should:
1. Try `generate-design` again (simplest smoke test: one call with `asset_ids` on the title slide) to check if the quota has reset.
2. If still blocked, ask the user whether to (a) keep waiting, (b) have them run the "make editable" conversion themselves in ChatGPT/Canva UI for all 40 slides and hand back resulting design IDs for Claude to assemble/verify/merge, or (c) fall back to static-image-only for everything (rejected once already this session as not fully meeting the ask, but may become the pragmatic choice if quota never clears).

## Open problem: no blank-canvas tool for static table-image placement
Even the "safe" static-image path (no AI, just place the table PNG on a page) has no clean tool. Every design-creation tool in this MCP surface is either AI-generation-based (`generate-design`, blocked) or template-based (`create-design-from-brand-template`, needs `search-brand-templates` first, untested this session) or HTML-import-based (`import-design-from-url`, rejected by user for whole-deck authoring — unclear if user would tolerate it for a single mechanical single-image wrapper page; **do not assume yes, ask first** if this route is considered). Next session should investigate `create-design-from-brand-template` + `search-brand-templates` as an AI-quota-free way to get a blank/simple canvas, then use `edit-design` to swap in the uploaded table-image asset full-bleed.

## Source assets (all confirmed present, public, already correct — no need to re-derive)
GitHub repo: `Romanch001/AI-Slide-Deck-Automation` (public). All 40 raw NotebookLM slide PNGs already staged at:
`runs/wire-ropes-drum-sheaves-20260728/p1-slide-01.png` through `p1-slide-21.png` (Part 1, 21 slides)
`runs/wire-ropes-drum-sheaves-20260728/p2-slide-01.png` through `p2-slide-19.png` (Part 2, 19 slides)
Raw download pattern: `https://raw.githubusercontent.com/Romanch001/AI-Slide-Deck-Automation/<ref>/runs/wire-ropes-drum-sheaves-20260728/<filename>` — use `get_file_contents` on the dir to get a fresh `download_url`/ref if the pinned commit SHA used this session (`d6498e540c62106809b98d5563cb7c8398bc5db4`) has since moved.
Confirmed image dimensions: 1376×768 (≈16:9).
Two test assets already uploaded into the user's Canva account this session (reusable, no need to re-upload): asset `MAHQ1J7rkBE` = p1-slide-01 (title slide), asset `MAHQ1BZ6B5o` = p1-slide-09 (Table 23.3).

Dense-table slide indices (image-embed only, per earlier verified outline):
- Part 1: slides 9 (Table 23.3), 10 (Table 23.4), 11 (Table 23.5), 12 (Table 23.6), 18 (Table 23.7), 21 (Fig 23.6 fatigue diagram)
- Part 2: slides 1 (Table 23.8), 12 (Table 23.9), 18 (quick-reference sheet)

Also present in the repo (older cropped/watermark-cleaned variants from an earlier, now-superseded approach — probably not needed under the new method, but available): `runs/final-part1-20260728/`, `runs/final-part2-20260728/`, `runs/part1-20260729-194338/`, `runs/part2-20260729-194537/`. Ground-truth real-textbook page renders for Table 23.3/23.4 cross-checking (`page823_real.png`, `page824_real.png`) existed in a now-expired local scratchpad from a prior session — not confirmed to still exist anywhere; may need re-extraction from the source PDF if needed again for verification.

## Existing Canva designs to abandon (do not merge into final deck)
- `DAHQxq7l9Ec` ("Wire Ropes Part 1", https://www.canva.com/d/MRcK0OGUjnDqjIw) — built via the rejected HTML-import method. User confirmed abandon, do not delete, just don't use.
- An unnamed/unknown Part 2 design — a prior session's build agent died mid-task (hit account monthly spend limit) before reporting its design ID. Likely built the same wrong way. User confirmed: don't bother investigating/salvaging it, treat as abandoned.
- Three stray debug design IDs from Part 1's build process, self-reported as discardable by that agent: `DAHQxvX16k0`, `DAHQxj7Ir7I`, `DAHQxgyQ3S4`.

## Task list state (carry forward, then rewrite once unblocked)
- #1 [completed] Crop watermark-free diagram images for HTML deck (superseded — HTML approach abandoned, this task is now moot)
- #3 [in_progress] Stage 40 raw images, import each to Canva, merge — **redefine**: staging is done (see Source assets above), the "import" method needs to change to the generate-design/candidate pipeline once unblocked
- #4 [pending] Verify Canva import matches intent
- #5 [pending] Clean up staged files from GitHub
- #6 [in_progress] Build Part 1 Canva design (21 slides) — must restart from scratch with correct method
- #7 [pending] Build Part 2 Canva design (19 slides) — must restart from scratch with correct method
- #8 [pending] Merge Part 1 + Part 2 into final 40-slide deck

## Suggested skills for next session
- None of the installed skills directly cover "Canva MCP image-to-editable-design pipeline debugging" — this is bespoke tool-integration work. If the user wants to stress-test the resumed plan before executing, invoke `grill-me` again (was used productively this session to pin down test-first / static-table-only / abandon-old-designs / batch-checkpoint decisions — those decisions are already final, don't re-litigate them, just execute).
- If Canva MCP tools appear to have changed/disconnected at session start, do a `ToolSearch` sanity check before assuming anything is broken (happened twice already this project).

## Everything else (context, not required reading)
Full prior history — content-fidelity verification against the real textbook, the corkscrew-wording fix, duplicate-text fix on slide 5, the missing rope-texture decorative graphic on the Part 1 title slide (still unresolved, moot now since Part 1 is being rebuilt from scratch) — is in the previous session transcripts and the project's supermemory/claude-mem observation log (searchable via the `mem-search`/`smart_search` tools, session ref search terms: "wire rope", "Chapter 23", "Bhandari"). Not reproduced here since it's superseded by the method change; re-derive only if the new build surfaces a fidelity question those fixes already answered.
