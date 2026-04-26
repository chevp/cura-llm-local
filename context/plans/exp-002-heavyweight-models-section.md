---
id: exp-002
title: Heavyweight-tier models section in docs/index.html
exploration-mode: B
status: accepted
proposed-by: ai
decided-by: —
approved-by: —
approved-at: —
created: 2026-04-26
related-adr: —
ux-prototype: ../specs/heavyweight-section-mockup.html
gate-note: G2 passed 2026-04-26 (see governance-log.md)
---

## Context summary (verbal CTX, captured retrospectively)

The user requested a new "Heavyweight tier models" section in `docs/index.html`. The clarifying questions in the prior turn were not answered explicitly; the user said "continue", which is interpreted as *"proceed with reasonable defaults — flag every assumption so I can correct it at the gate."*

### Assumptions made (each is a gate-review point)

1. **Audience** — Linux/Windows + NVIDIA GPU users. macOS Apple-Silicon users get an explicit "do not attempt locally" caveat. *Rationale:* CLAUDE.md says *"macOS is CPU-only. Stick to q4-quantised 7B–8B models in defaults and examples."* Heavyweight is, by definition, the opposite of the default tier.
2. **Model list** — four curated entries at q4_K_M:
   - `llama3.1:70b-instruct-q4_K_M` — flagship general/reasoning (~40 GB)
   - `qwen2.5:32b-instruct-q4_K_M` — strong coding/general (~20 GB)
   - `mixtral:8x7b-instruct-v0.1-q4_K_M` — MoE, fast inference for its size (~26 GB)
   - `deepseek-r1:32b` — reasoning specialist (~20 GB)
3. **Structure** — *new standalone section* below the existing "Recommended models" table. Existing table is **not** restructured into tiers (Candidate B), to avoid disturbing the "feels fast" framing for macOS users.
4. **Hardware guidance** — top-of-section banner with GPU/VRAM caveat + per-row VRAM column.
5. **Scope** — docs only. No `scripts/`, no compose, no `.env.example` changes. No new `setup-heavyweight.sh`.
6. **Default tier unchanged** — `mistral:7b-instruct-q4_K_M` remains the documented default.

If any assumption is wrong, that's a G2 block — please correct at gate review.

## Problem statement

[docs/index.html:247-300](../../docs/index.html#L247-L300) currently lists five "Recommended models" capped at 8B q4 — appropriate for the macOS-CPU baseline. Cura developers running on Linux + NVIDIA hardware (per the Platform notes section, [docs/index.html:303-333](../../docs/index.html#L303-L333), *"any — GPU handles 13B+ easily"*) currently have **no curated guidance** for which larger models are worth pulling. The result is one of:

- Developer wades through ollama.com manually and picks something inconsistent across the team.
- Developer sticks with the 8B default and never exercises GPU capacity.
- Developer pulls a 70B fp16 by mistake (~140 GB) and fills the disk.

Goal: a curated, opinionated list of GPU-tier models with explicit hardware thresholds, surfaced on the same docs page, without weakening the existing macOS-CPU guidance.

## Hypotheses (with kill criteria)

- **H1 — At least one Cura developer has GPU hardware that benefits from a heavyweight tier.**
  *Kill criterion:* if no developer on the team currently has a Linux/Windows NVIDIA box (≥ 24 GB VRAM) and none is on the procurement roadmap, this section is documentation theatre. → kill the EXP; the right artifact is a `PROP-NNN` to revisit when hardware arrives.

- **H2 — A standalone "Heavyweight tier" section adjacent to "Recommended models" does not confuse macOS users into pulling 70B models.**
  *Kill criterion:* in mockup review (visual), the GPU caveat banner is missed on first read OR the table is reachable from the Hero "Try it in 2 minutes" button without crossing the macOS guidance first. → fall back to **Candidate C** (collapsible accordion, closed by default).

- **H3 — q4_K_M is the right quantisation default for the heavyweight tier.**
  *Kill criterion:* community consensus (Ollama model pages, llama.cpp benchmarks) shows q4_K_M produces materially worse output than q5_K_M / q8_0 for any of the four chosen models, at acceptable VRAM cost. → swap that specific row to a higher quantisation; do not change the section structure.

## Risks

1. **Disk-footprint accident.** A user on macOS misreads the page and pulls 40 GB. Mitigation: GPU caveat banner + per-row "Disk" column + explicit "macOS Apple Silicon: do not attempt" inline note.
2. **Curated list goes stale.** Models churn quarterly. Mitigation: stick to widely-deployed, stable releases (the four chosen are all > 6 months old as of 2026-04). Owner reviews quarterly via a recurring `PROP` (out of scope here, raise as proposal at G2).
3. **VRAM thresholds are approximations.** Real VRAM usage depends on context length and KV cache. Mitigation: state VRAM as "≥ N GB recommended"; link to Ollama's docs for exact sizing.
4. **Conflict with CLAUDE.md guidance.** *"Stick to q4-quantised 7B–8B models in defaults and examples"* — heavyweight tier is **explicitly not a default** and is gated by a GPU caveat. The default stays mistral:7b. Resolution: no conflict; the rule is about *defaults*, not about *what we document for GPU users*.
5. **GitHub Pages anchor-link change.** Adding `#heavyweight` to the header nav extends the nav row; on narrow viewports this may wrap. Mitigation: don't add a new nav link; rely on intra-section flow ("Heavyweight" appears immediately after "Recommended models").

## Candidates (≥2 comparable)

### Candidate A — New standalone section, below "Recommended models" *(recommended)*

- New `<section id="heavyweight">` directly after the closing `</section>` of "Recommended models", before the "Platforms" section.
- GPU caveat banner at the top of the section (amber pill, fa-solid fa-triangle-exclamation).
- Table with four rows, columns: Model · Disk · VRAM (recommended) · Use case.
- Footer one-liner: *"q4_K_M is the default; pull q5_K_M / q8_0 if you have headroom."*
- No nav link added (intentional — section is opt-in, not a top-level concept).

**Pros:** zero changes to existing "Recommended models" content; reversible (one section block); clear audience separation; smallest diff.
**Cons:** macOS user must scroll past it to reach Platform notes. Two model tables on one page.

### Candidate B — Restructure existing table into tiers (Lightweight / Default / Heavyweight)

- Single unified table with a "Tier" column.
- Lightweight: llama3.2:3b, phi3:mini.
- Default: mistral:7b (current default), llama3.1:8b.
- Heavyweight: the four GPU models.
- nomic-embed-text moves to its own mini-section ("Embeddings").

**Pros:** single source of truth; visual tier comparison; cleaner information architecture long-term.
**Cons:** disturbs the existing "CPU-only on Apple Silicon — these are the ones that *actually* feel fast" framing; mixes audiences in one table; larger blast radius (touches every existing row); harder to revert.

### Candidate C — Collapsible accordion ("Heavyweight tier (GPU only) ▸") closed by default

- Same content as Candidate A but inside a `<details>` block, closed on first render.
- Reuses the existing FAQ accordion styling ([docs/index.html:104-107](../../docs/index.html#L104-L107)).

**Pros:** maximum protection against accidental 70B pulls; macOS user never sees the heavyweight content unless they actively expand it.
**Cons:** SEO / discoverability worse (collapsed content sometimes deprioritised by indexers); requires an extra click for the GPU users this is *for*; the GPU caveat banner becomes redundant once the user has already opted in by expanding.

### Recommendation: **Candidate A**

- Smallest diff, smallest blast radius.
- Banner + per-row disk column already gates against accidental misuse.
- Existing "Recommended models" framing for macOS users is preserved verbatim.
- C remains a fallback if H2 is killed at UX review.
- B remains a future option if more tiers are ever added (would justify the restructure).

## Challenger output

### Top-3 concrete failure modes with observable signals

1. **macOS user pulls 70B by mistake.** Signal: GitHub issue or Slack message saying *"my disk is full / pull is taking forever"*; container OOM-kills on inference. → Audit log: was the GPU banner visible above the fold of the section? Was the per-row VRAM column rendered on mobile (might be the column that gets squashed)? Mitigation in PRD: explicit *"macOS Apple Silicon: do not attempt locally"* line inside the banner, not just the icon.
2. **Curated model is removed/renamed upstream.** Signal: `./scripts/pull-model.sh deepseek-r1:32b` fails with `manifest unknown`. → Mitigation in PRD: pin the tag we test against; add a note about checking ollama.com for current tag.
3. **Banner styling regression on narrow viewports.** The amber pill wraps awkwardly on < 380 px viewports. Signal: visible during UX-prototype review. → Mitigation in PRD: test the prototype at 320 px / 375 px / 768 px / 1024 px before commit.

### ≥2 genuinely different alternatives with rejection reason and reopening condition

- **Candidate B (unified tiers).** Rejected: large diff, disturbs proven framing, premature for a 4-row addition. **Reopening condition:** if/when we add a 4th tier (e.g., "Quantised-only" or "Vision models"), the restructure becomes worthwhile.
- **Candidate C (accordion, closed).** Rejected: hurts the actual target audience (GPU users) for hypothetical macOS protection that the banner already provides. **Reopening condition:** UX-prototype review (H2) shows the banner is genuinely missed by macOS readers; or we observe a real "user pulled 70B by mistake" incident.
- **Move heavyweight content to a separate page** (`docs/heavyweight.html`). Rejected: docs is currently a single-page site (`index.html`, `chat.html`, `status.html` are functionally distinct, not topical splits); a topical split would fragment the model conversation. **Reopening condition:** the heavyweight list grows past ~10 entries or includes per-model deep-dives.

### Strongest counter-argument against Candidate A (charitable framing)

> "You're shipping a curated list with no measurement that anyone uses these models. The right move is to *first* survey the team for actual GPU access, then write the section against confirmed demand — otherwise this is just a wishlist that ages into staleness."

**Response:** Fair, and this is exactly what H1's kill criterion encodes. If H1 is killed at G2 (no developer has the hardware), this EXP **does not advance to PRD**; we instead file a `PROP` to revisit when hardware lands. The cost of writing the curated list now, conditional on H1, is low (~one section in one file); the value if H1 holds is real (consistent team choices). The risk is staleness, which is a recurring-review problem the next quarterly sweep handles.

## Kill criteria (gate-level)

The exploration is **killed** if:

- H1 fails: no developer on the Cura team currently has Linux/Windows + NVIDIA ≥ 24 GB VRAM, and none is on the procurement roadmap.
- All four chosen models prove unstable in current Ollama (`pull` fails or inference crashes) — at which point the curation premise collapses.

The exploration **falls back to Candidate C** (accordion) if:

- UX-prototype review shows the GPU banner is missed on first read by a macOS-perspective reviewer.

The exploration **proceeds with note** (not killed) if:

- One or two of the four models need swapping for a current alternative (e.g., qwen2.5:32b → qwen3:32b) — surface in PRD, do not re-explore.
- VRAM thresholds need tightening per Ollama docs — surface in PRD.

## Scope

### In scope
- Add new `<section id="heavyweight">` between [docs/index.html:300](../../docs/index.html#L300) (close of "Recommended models") and [docs/index.html:303](../../docs/index.html#L303) (open of "Platforms").
- Banner: GPU/VRAM caveat with explicit *"macOS Apple Silicon: do not attempt locally"* line.
- Table: 4 rows × 4 columns (Model · Disk · VRAM · Use case), reusing existing `.glass` / `.mono` / `pill` styling.
- Footer one-liner about quantisation.
- UX prototype at `context/specs/heavyweight-section-mockup.html` (standalone, renderable in browser).

### Out of scope (NOT-in-Scope, candidates for PROP)
- Restructure of the existing "Recommended models" table → would-be **PROP-002** (revisit when ≥4 tiers exist).
- New `scripts/pull-heavyweight.sh` helper → would-be **PROP-003** (revisit if pulling is itself a friction point).
- Quarterly review automation for the curated list → would-be **PROP-004**.
- Adding `#heavyweight` to the header nav → would-be **PROP-005** (revisit if section becomes a primary entry point).
- Vision / multimodal model tier → would-be **PROP-006**.

## Evidence block

```yaml
evidence:
  hypothesis: A standalone, GPU-gated "Heavyweight tier" section adjacent to the existing "Recommended models" table gives Linux/Windows GPU developers curated guidance without weakening macOS-CPU defaults, and the GPU-caveat banner is visually strong enough to prevent accidental 70B pulls on Apple Silicon.
  result:     User approved at G2 on 2026-04-26 without explicit external verification of H1 (GPU-equipped developer present on team) or H2 (UX-prototype review feedback). H3 (q4_K_M default) accepted on the back of current ollama.com model-page conventions. UX prototype rendered as a standalone HTML mockup at context/specs/heavyweight-section-mockup.html.
  reasoning:  User holds direct visibility into team hardware and rendered the prototype in their own environment. Approval interpreted as: assumptions accepted, proceed to PRD. The change is reversible (single new <section> block in docs/index.html) — if H1 or H2 turn out to be wrong post-ship, retracting the section is a one-commit revert.
```
