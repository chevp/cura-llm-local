---
id: exp-002-insights
related-plan: exp-002
status: G3-evidence
created: 2026-04-26
updated: 2026-04-26
---

# Insights — exp-002 heavyweight-tier models section

Filled after PRD execution on 2026-04-26. Section shipped as Candidate A (standalone, GPU-gated, no nav-link, no tier restructure of the existing "Recommended models" table).

## Confirmed hypotheses

- **H3 — q4_K_M is the right quantisation default for the heavyweight tier.** ✅ accepted on convention.
  - All four chosen models have q4_K_M as the leading variant on their ollama.com library pages. No community signal at time of writing suggests a different default would be materially better at the listed VRAM thresholds.

- **V4 — ollama.com tag sanity.** ✅ confirmed.
  - All four model families return HTTP 200 from `https://ollama.com/library/<family>`: `llama3.1`, `qwen2.5`, `mixtral`, `deepseek-r1`.

- **V3 — diff is pure insertion.** ✅ confirmed.
  - `git diff --stat docs/index.html` reports `68 insertions(+)`, zero deletions. No surrounding-line drift; "Recommended models" and "Platforms" sections are byte-identical.

- **H2 — banner reads strongly enough on first render that macOS users will not pull a 70B by mistake.** ✅ accepted via user visual review.
  - User confirmed "ok" after rendering the page on `http://localhost:5500/index.html#heavyweight`.
  - Banner places "GPU required." in amber as the first vertical element of the section, with a distinct red `macOS Apple Silicon` pill that breaks visually from the existing blue pills used elsewhere on the page. The colour contrast was the carrying signal.

## Awaiting manual verification

- **H1 — at least one Cura developer has GPU hardware that benefits from a heavyweight tier.** ⏳ Not externally verified.
  - Accepted on user authority at G2; user holds direct visibility into team hardware and procurement roadmap.
  - **Re-verify trigger:** if 90 days from now no one on the team has pulled any of the four documented models, the section is decoration → file a `PROP` to revisit (revert vs. update vs. broaden the tier).

- **V2 responsive viewport check (320 / 375 / 768 / 1024 px).** ⏳ User-side visual.
  - User answered "ok" without an explicit per-viewport report. Carrying assumption: the visual review covered narrow widths.
  - **Fallback if a layout regression shows up later:** wrap the table in `<div class="overflow-x-auto">` per prd-002 V2 fallback note.

## Killed hypotheses

None.

## Surprises

- **The caveat banner did not need a dedicated CSS class.** Inlining the gradient + border colour via `style="..."` kept the diff to a single section block. If a second use case for this banner pattern appears later, that's the right time to promote it to a `.caveat` class — not before.
- **PRD insertion-point coordinates were stable.** Unlike exp-001, where `docs/chat.html` line numbers shifted between PRD authoring and execution, here the insertion point at [docs/index.html:300-303](../../docs/index.html#L300-L303) matched exactly. No other write activity touched the file in the same conversation.

## What remains unknown

- **Real-world model pulls.** No telemetry exists for which models the team actually uses. The kill criterion in H1 ("90-day check") is the only feedback loop.
- **Quantisation deltas.** No empirical comparison of q4_K_M vs q5_K_M vs q8_0 quality on the four chosen models was performed in this EXP — accepted on community defaults.
- **Cross-browser visual regression.** The visual review was Chrome-on-macOS-shaped (assumption based on user environment). Firefox and Safari rendering of the inline-styled banner is not verified.

## Consequence

prd-002 ships. The heavyweight-tier section is in `docs/index.html` working tree, ready to commit when the human asks. The change is a single insertion in a single file with a one-commit revert path. Out-of-scope items (PROP-002 through PROP-006) were captured in exp-002 and remain open as future-work pointers; none are filed as standalone PROP files in `context/plans/proposals/` (consistent with the prevailing repo convention for exp-001's NOT-in-Scope items).
