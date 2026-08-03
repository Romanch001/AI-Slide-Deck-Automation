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
| Browser automation | A Playwright-drivable browser (headed or headless) with a way to establish one Canva login session that persists across calls | Set one up before Phase 6 — see that phase for why it's required |

If any check fails, stop and report which one before proceeding — do not skip a phase and hope it resolves itself later.

**Known bug (as of 2026-08-03):** `notebooklm-py` v0.7.3's login/session detection is hardcoded to the host `notebooklm.google.com`, but Google has since served the product at `notebook.google.com` for at least some accounts. When this happens, `notebooklm login` hangs indefinitely waiting for a host it will never see, even though the browser window itself shows a fully authenticated session — no retry, `--fresh`, or forced-reload trick fixes it, because the CLI's allowed-host list rejects the new domain outright. If `notebooklm login` won't complete after one retry, stop trying to force it and fall back to the operator manually downloading the PPTX from the NotebookLM web UI (Studio panel → slide deck artifact → download) directly into wherever Phase 4 expects its input. Re-check whether the CLI has been patched for the new host before spending more time on it in a future run.

## Naming convention

Every run gets a slug: lowercase topic, spaces→hyphens, date suffix.
Example: topic "Wire Rope Groove Design" on 2026-07-28 → `wire-rope-groove-design-20260728`.

Use this slug for: the NotebookLM notebook title, the GitHub staging folder, and the final Canva design title.

**Multi-part decks:** if a topic is too large for one NotebookLM slide-deck generation, split it into separate notebooks/generations (Part 1, Part 2, …), each producing its own PPTX. Carry them through Phases 1–6 independently, staging each part's images into its own `part1/`, `part2/`, … subfolder under the same run slug. Only combine them at Phase 7 (merge into one final deck, in part order).

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

**Superseded 2026-08-03.** The vision-rebuild approach previously documented here (manually re-authoring each slide as add-text/insert-shape/insert-fill elements) is abandoned — it does not reproduce the source slide closely enough, and the operator rejected its output on an earlier run. Do not use it.

The confirmed correct mechanism is Canva's **Magic Layers** feature (canva.com/help/editable-magic-layers) — it converts a flat image into genuinely editable layers while preserving the original content, rather than having an LLM guess at a re-creation. **There is no API or MCP tool for this** (confirmed via the Canva connector's own help tool: "described as a Canva app feature used in the UI, with no API flow listed") — it only exists as a manual action in the Canva web UI. Getting it into an automated pipeline means driving the actual UI with a browser:

1. **One-time setup:** open a Playwright-controlled browser and navigate to Canva. It won't have a session yet — the operator completes the Canva login once, manually, in that browser window. As long as the same browser profile/context is reused for every subsequent call, the session persists and no further logins are needed for the rest of the run (or future runs, if the profile is kept).
2. **One slide at a time, no bulk/batch conversion** — this is a hard requirement from the operator, not just a suggestion. For each staged slide image (in order):
   - Load the image into Canva (upload it, or open it on a page).
   - Trigger Magic Layers on it through the UI.
   - Confirm the result actually preserved the slide's content (read back the design, compare to the source image) before moving to the next slide.
3. Magic Layers requires a paid Canva plan (Pro/Teams/Edu/Nonprofit/Enterprise/Business) — it silently isn't available on Free. If it's not appearing in the UI, check the operator's plan before assuming the automation is broken.
4. Do not attempt to substitute `generate-design` + `create-design-from-candidate` as an automated stand-in for Magic Layers unless the operator explicitly says that tradeoff is acceptable for this run — it's a different mechanism (AI re-generation from a text query against the image, not layer-preserving conversion) and produced a rejected result on an earlier run (design `DAHREOYUVf8`, archived, do not reuse).

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
