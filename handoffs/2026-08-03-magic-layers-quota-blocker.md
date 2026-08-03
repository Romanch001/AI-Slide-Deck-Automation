# Handoff: Wire Rope Deck Rebuild — Magic Layers Quota Blocker

## Status: Blocked on Canva's monthly Magic Layers AI quota. Resumable Sep 1, 2026 (or sooner if plan upgraded).

## What's confirmed working (this session, 2026-08-03)
Full pipeline validated end-to-end:
1. Extracted 40 slide images from two NotebookLM PPTX exports (`Hoisting_System_Design.pptx` = Part 1, 21 slides; `Wire_Rope_Design_Systems.pptx` = Part 2, 19 slides) via `python-pptx`, reading each slide's single embedded picture. Staged at `runs/wire-ropes-drum-sheaves-20260803/part1/` + `part2/` in this repo, and at `.../5. Mega (Sem-7)/Chapter 23 - Wire Ropes, Sheaves and Drums/part1/` + `part2/` locally.
2. Per-slide Canva conversion procedure (see AGENTS.md Phase 6, rewritten this session):
   - Navigate a Playwright browser to `https://www.canva.com/design/editor/shell?create&width=1376&height=768&units=px` — creates a blank design at the exact source aspect ratio, free, no quota/paywall (discovered this URL pattern; the UI "Resize" feature and the `resize-design` API tool are BOTH paywalled/trial-limited — avoid both).
   - Click Uploads tab → Upload files → `browser_file_upload` with the local slide PNG path (must be under the browser tool's allowed root, which is the current project directory — the Mega folder copy works, a scratchpad path does not).
   - Click the uploaded thumbnail to place it on the page (auto-fits since canvas matches image dimensions).
   - Select the image → click "Edit" → click "Magic Layers" → wait ~5-10s for "Image split into editable layers".
   - Via Canva API (`read-design` with `open_transaction: true`): find the watermark elements (a small icon RECT + a "Notebook<LM"/"NotebookL.M"/"@NotebookL.M" TEXT, both near the bottom-right, sometimes merged into one text element) and `delete_element` both, per operator's explicit instruction (2026-08-03) — supersedes the old "leave watermark, user removes it" rule now that Magic Layers makes it a clean separate layer (see memory `feedback_notebooklm_watermark.md`).
   - **Before committing:** visually check the thumbnail against the source slide, especially any dense table/data region — Magic Layers has a confirmed failure mode where it silently outputs a *blank* image for a packed region while the rest of the slide converts fine (happened on Part 1 slide 4's construction-notation table). Fix: crop the correct region from the source PNG (matching the broken element's aspect ratio), stage publicly (push to this repo, use the raw.githubusercontent.com URL), `upload-asset-from-url`, then `update_fill` on the broken element — swaps image content without disturbing position. Don't skip this check; it's the one place data accuracy actually broke.
   - Commit the transaction.

## Completed slides (Part 1, keep these — do not redo)
1. `DAHROUGQTP8` — title slide
2. `DAHROX2E4Kc` — Introduction: Why Wire Ropes?
3. `DAHRObD3aks` — Anatomy of a Wire Rope
4. `DAHROSFjiNs` — Rope Specification Notation (table was fixed, see above)
5. `DAHROfCdqxI` — Rope Lay: Direction of Twist

## Blocked / discard
- `DAHRObHhEz4` — Part 1 slide 6 attempt. Magic Layers never ran (quota hit right before/during this one); design is just the raw uploaded image, no layers. Discard, don't reuse — redo slide 6 from scratch next session.

## Remaining work
- Part 1: slides 6–21 (16 slides)
- Part 2: slides 1–19 (19 slides) — not started at all
- Total remaining: 35 slides
- After all 40 are converted: merge into one final deck via `merge-designs` (`create_new_design`, `insert_pages` from each per-slide design in order), Part 1 pages first then Part 2.

## The blocker itself
Canva showed: "You've hit your plan's monthly AI limit. It'll reset on Sep 1." followed by an upgrade paywall ("Try Canva Business for free... 20x more AI") when slide 6's Magic Layers was attempted. This is a plan-level quota shared by the whole account — not something this session can work around. Operator chose to wait for the Sep 1 reset rather than upgrade or investigate a separate personal-access path (2026-08-03 decision, see memory).

## Next session should
1. Confirm quota has actually reset (try Magic Layers on a throwaway test image first, same as the smoke-test approach used earlier in this project for the generate-design quota).
2. Resume at Part 1 slide 6, following the exact procedure above.
3. Re-check AGENTS.md Phase 6 for any updates if the Magic Layers UI or quota behavior has changed.
