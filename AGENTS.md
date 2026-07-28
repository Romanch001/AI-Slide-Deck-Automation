# AI Slide Deck Automation — Agent Runbook

Reusable pipeline: topic → research → editable Canva presentation.
Works with any LLM/tool that has connector access to **NotebookLM**, **Canva**, and **GitHub**. Not tied to any single project — the operator supplies a topic each run.

## Required access

Before starting any run, verify all three:

| Tool | Check | Fix if failing |
|---|---|---|
| NotebookLM | `notebooklm auth check --test --json` → `"status":"ok"` and `"checks.token_fetch":true` | `notebooklm login` |
| GitHub | Write access to `Romanch001/AI-Slide-Deck-Automation` | Ask operator to grant/reconnect |
| Canva | Connector/MCP exposes: design generation, per-element editing (add text, format text, insert shape, insert/replace image fill), and page-merge/combine | Ask operator to reconnect Canva connector |

If any check fails, stop and report which one before proceeding — do not skip a phase and hope it resolves itself later.

## Naming convention

Every run gets a slug: lowercase topic, spaces→hyphens, date suffix.
Example: topic "Wire Rope Groove Design" on 2026-07-28 → `wire-rope-groove-design-20260728`.

Use this slug for: the NotebookLM notebook title, the GitHub staging folder, and the final Canva design title.

## Phase 0 — Topic intake and research scoping (interactive, do not skip)

1. Operator gives a topic.
2. Agent proposes a research scope: what sources to pull (web research via NotebookLM, specific URLs/PDFs the operator supplies, or both), and roughly how many sources / how deep.
3. **Discuss with the operator and get explicit agreement on the research scope before adding a single source.** This is a real conversation, not a formality — the operator may already have specific documents, angles, or exclusions in mind.

## Phase 1 — Research (NotebookLM)

```bash
notebooklm create "<slug>" --json                       # capture notebook id
notebooklm source add "<url-or-file>"                    # once per agreed source
notebooklm source add-research "<query>" --mode fast     # if web research agreed
notebooklm source list --json                            # poll until all sources status=ready
```

Wait for every source to reach `ready` before moving on — chat/generation on a `processing` source gives incomplete answers.

## Phase 2 — Content synthesis (outline + theme)

Query the notebook, don't guess from memory:

```bash
notebooklm ask "Summarize the key points for a presentation on <topic>" --json
notebooklm ask "What are the main sections/arguments?" --json
```

Produce two artifacts and **present both to the operator for approval before generating anything**:

1. **Slide outline** — ordered list of slide titles + one-line content brief each.
2. **Theme spec** — color palette (hex), font pairing (heading/body), layout style (e.g. "three-panel", "text-left image-right"). Keep it simple enough to describe in one paragraph — NotebookLM's own slide-deck generator does not accept a structured theme file, only prompt text (see Phase 3).

Do not proceed to Phase 3 until the operator has signed off on both.

## Phase 3 — Generate the deck (NotebookLM)

NotebookLM has no dedicated "theme file" input — fold the approved outline + theme into the generation prompt itself:

```bash
notebooklm generate slide-deck "Create a slide deck covering: <outline>. Visual style: <theme spec>." --json
notebooklm artifact wait <task_id> -n <notebook_id> --timeout 1200
notebooklm download slide-deck ./out/<slug>.pptx --format pptx -a <task_id> -n <notebook_id>
```

Known limitation: slide-deck generation is rate-limit-prone. On failure, wait 5–10 min and retry once before escalating to the operator.

Known output shape: each NotebookLM slide is a single flattened image (not editable text) — this is expected, not a bug. Phases 4–6 exist specifically to recover editability.

## Phase 4 — Extract slide images

Pull each slide out of the downloaded PPTX/PDF as its own image file (e.g. via `python-pptx` to read embedded images, or a PDF→PNG render if downloaded as PDF). Output: `slide-01.png`, `slide-02.png`, … in run order.

## Phase 5 — Stage images publicly (GitHub)

Canva's ingestion tools (image upload, design import) require a public HTTPS URL — they do not accept local file bytes. This repo exists solely to bridge that gap.

```
Push to: Romanch001/AI-Slide-Deck-Automation, path runs/<slug>/slide-NN.png
URL to feed Canva: https://raw.githubusercontent.com/Romanch001/AI-Slide-Deck-Automation/main/runs/<slug>/slide-NN.png
```

Push all slides for the run before moving to Phase 6.

## Phase 6 — Recreate each slide as an editable Canva design

There is no single "convert image to editable design" tool. The real mechanism (verified working, see run from 2026-07-28):

1. Read each staged slide image (vision) — identify text blocks, panel/background shapes, and any non-text graphic (diagrams, charts).
2. Rebuild it as native Canva elements on one page:
   - Text blocks → add-text + format-text (set font, size, color, alignment to match the theme)
   - Colored panels/backgrounds → insert-shape
   - Non-text graphics (diagrams, photos) → insert/replace an image fill, sourced from the same staged URL (crop to just that region if needed)
3. **Do not recreate the NotebookLM logo/watermark.** It's simply never added — cleaner than deleting it after the fact.
4. Match the approved theme spec from Phase 2 (colors, fonts) rather than copying the NotebookLM rendering pixel-for-pixel — small drift is fine, broken brand consistency is not.

One Canva design per slide at this stage.

## Phase 7 — Combine into one deck (Canva)

Use the page-combine/merge capability to assemble all per-slide designs into a single design, in original slide order. Title it `<slug>` (human-readable form, e.g. "Wire Rope Groove Design — 2026-07-28").

## Phase 8 — Cleanup (GitHub)

Once Canva has successfully ingested every slide image for the run, delete `runs/<slug>/` from the staging repo. This repo is a transient bridge, not an archive — nothing from a completed run should linger publicly.

## Phase 9 — Deliver

Report to the operator:
- Final combined Canva design link (editable)
- Confirmation that staging images were deleted (Phase 8 done)
- Optionally: exported PPTX/PDF download if the operator wants a static copy too

## Failure handling (all phases)

On any tool error: retry once if it's rate-limiting or a transient network error; otherwise stop and report the exact error to the operator with which phase failed. Never silently skip a phase or fabricate a result to keep moving.
