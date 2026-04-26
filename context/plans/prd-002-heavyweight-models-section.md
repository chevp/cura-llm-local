---
id: prd-002
title: Heavyweight-tier models section — implementation
status: accepted
proposed-by: ai
decided-by: —
approved-by: —
approved-at: —
created: 2026-04-26
related-plan: exp-002
related-adr: —
gate-note: G3 passed 2026-04-26 (see governance-log.md; user said "continue" — treated as approval)
---

## Goal

Implement the curated "Heavyweight tier" section in [docs/index.html](../../docs/index.html) per [exp-002](exp-002-heavyweight-models-section.md) Candidate A: a new `<section id="heavyweight">` between the existing "Recommended models" and "Platforms" sections, with a GPU-required caveat banner and a four-row model table. No other files touched.

## Pre-conditions

- `/approve exp-002` recorded in [governance-log.md](../governance-log.md) — ✅ done 2026-04-26.
- UX prototype reviewed at [context/specs/heavyweight-section-mockup.html](../specs/heavyweight-section-mockup.html) — accepted at G2.
- Working tree clean on `main` before edit (or a feature branch ready).

## Step-by-step implementation

A single edit to a single file. Insertion point and exact content below.

### Step 1 — Insert new section in `docs/index.html`

**File:** [docs/index.html](../../docs/index.html)
**Insertion point:** between line 300 (closing `</section>` of "Recommended models") and line 303 (opening `<section id="platforms">` of "Platforms").

Insert exactly the following block (preserving the surrounding blank line above and below for diff cleanliness):

```html
  <!-- Heavyweight tier -->
  <section id="heavyweight" class="max-w-4xl mx-auto px-6 py-16">
    <h2 class="text-2xl md:text-3xl font-bold text-center mb-3 gradient-text">Heavyweight tier</h2>
    <p class="text-slate-400 text-center mb-8">When you have a GPU and you want the good stuff.</p>

    <!-- GPU-required caveat banner -->
    <div class="glass rounded-xl p-5 mb-8" style="background: linear-gradient(135deg, rgba(245, 158, 11, 0.08), rgba(239, 68, 68, 0.06)); border-color: rgba(245, 158, 11, 0.3);">
      <div class="flex items-start gap-3">
        <i class="fa-solid fa-triangle-exclamation text-amber-400 text-xl mt-0.5"></i>
        <div class="text-sm">
          <p class="text-amber-200 font-semibold mb-1">GPU required.</p>
          <p class="text-slate-300 mb-2">
            These models target Linux / Windows hosts with an NVIDIA GPU and <strong>&ge;&nbsp;24&nbsp;GB VRAM</strong>.
            Use <code class="mono text-arctic-300">docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d</code>.
          </p>
          <p class="text-slate-300">
            <span class="pill" style="background: rgba(239, 68, 68, 0.12); border-color: rgba(239, 68, 68, 0.3); color: #fca5a5;"><i class="fa-brands fa-apple"></i> macOS Apple Silicon</span>
            &nbsp;Do <strong>not</strong> attempt these locally. The Apple GPU is not exposed to Docker &mdash; inference falls back to CPU and these models will swap your machine into the ground.
          </p>
        </div>
      </div>
    </div>

    <!-- Heavyweight model table -->
    <div class="glass rounded-xl overflow-hidden">
      <table class="w-full text-sm">
        <thead class="border-b border-slate-700/50">
          <tr class="text-slate-300">
            <th class="text-left px-5 py-3 font-semibold">Model</th>
            <th class="text-left px-5 py-3 font-semibold">Disk</th>
            <th class="text-left px-5 py-3 font-semibold">VRAM</th>
            <th class="text-left px-5 py-3 font-semibold">Use case</th>
          </tr>
        </thead>
        <tbody class="text-slate-400">
          <tr class="border-b border-slate-700/30">
            <td class="px-5 py-3 mono text-arctic-300">llama3.1:70b-instruct-q4_K_M</td>
            <td class="px-5 py-3">~40 GB</td>
            <td class="px-5 py-3">&ge; 48 GB</td>
            <td class="px-5 py-3">Flagship general &amp; reasoning</td>
          </tr>
          <tr class="border-b border-slate-700/30">
            <td class="px-5 py-3 mono text-arctic-300">qwen2.5:32b-instruct-q4_K_M</td>
            <td class="px-5 py-3">~20 GB</td>
            <td class="px-5 py-3">&ge; 24 GB</td>
            <td class="px-5 py-3">Strong coding &amp; multilingual</td>
          </tr>
          <tr class="border-b border-slate-700/30">
            <td class="px-5 py-3 mono text-arctic-300">mixtral:8x7b-instruct-v0.1-q4_K_M</td>
            <td class="px-5 py-3">~26 GB</td>
            <td class="px-5 py-3">&ge; 32 GB</td>
            <td class="px-5 py-3">MoE &mdash; fast inference for its size</td>
          </tr>
          <tr>
            <td class="px-5 py-3 mono text-arctic-300">deepseek-r1:32b</td>
            <td class="px-5 py-3">~20 GB</td>
            <td class="px-5 py-3">&ge; 24 GB</td>
            <td class="px-5 py-3">Reasoning specialist</td>
          </tr>
        </tbody>
      </table>
    </div>

    <p class="text-slate-500 text-xs text-center mt-6">
      <code class="mono text-arctic-300">q4_K_M</code> is the documented default &mdash; pull <code class="mono text-arctic-300">q5_K_M</code> or <code class="mono text-arctic-300">q8_0</code> if you have VRAM headroom and want quality over throughput.
    </p>
  </section>
```

Notes for the implementer:

- The block above is wrapped in `<!-- Heavyweight tier -->` so the diff is grep-able.
- No nav-link added at [docs/index.html:135-139](../../docs/index.html#L135-L139). Intentional — section is opt-in, not a top-level concept (per exp-002 NOT-in-Scope item PROP-005).
- No CSS rule added to the `<style>` block. The amber/red caveat colours are inlined via `style="..."` to keep the diff to one section. If a second use case for the caveat banner appears later, promote it to a `.caveat` class then.
- All other styling reuses existing utility classes (`.glass`, `.mono`, `.pill`, `.gradient-text`, Tailwind atoms).

### Step 2 — Render & visual check (no commit yet)

```bash
# From repo root
python3 -m http.server -d docs 5500
# Open http://localhost:5500/index.html#heavyweight
```

Verify against the Verification section below before staging.

## Verification

### V1 — Visual smoke test (desktop)

- Section renders between "Recommended models" and "Platforms".
- Caveat banner is the first thing the eye lands on inside the section (amber + warning icon).
- Four model rows render with correct `q4_K_M` tags, disk sizes, and VRAM thresholds.
- "macOS Apple Silicon — Do not attempt" pill is red (not blue) and visually distinct from the standard `.pill`.

### V2 — Responsive viewport check (Challenger failure mode 3)

Resize the browser (or use DevTools device toolbar) to:

- 320 px (smallest supported)
- 375 px (iPhone SE)
- 768 px (tablet)
- 1024 px+ (desktop)

At each width:
- The caveat banner does not visually break — text wraps gracefully, icon stays aligned.
- The model table either fits with horizontal scroll *or* the columns reflow without overlapping.
- The red macOS pill stays on its own line at narrow widths (no awkward wrapping mid-pill).

If the table overflows uncontrollably at 320 px: wrap the `<div class="glass rounded-xl overflow-hidden">` with `<div class="overflow-x-auto">`.

### V3 — Existing sections unaffected

- "Recommended models" table at [docs/index.html:247-300](../../docs/index.html#L247-L300) renders identically to before. Diff `git diff -- docs/index.html` shows only insertions, no edits to surrounding lines.
- Anchor links from the header nav (`#what`, `#quickstart`, `#models`, `#platforms`, `#troubleshooting`) all still scroll correctly.
- The new `#heavyweight` anchor is reachable via direct URL but is not in the nav (intentional).

### V4 — Model-tag sanity (Challenger failure mode 2)

Spot-check that each of the four documented tags currently exists on ollama.com:

```bash
# Optional, host-side; verifies pull would succeed
for m in llama3.1:70b-instruct-q4_K_M qwen2.5:32b-instruct-q4_K_M mixtral:8x7b-instruct-v0.1-q4_K_M deepseek-r1:32b; do
  echo "=== $m ==="
  curl -sSI "https://ollama.com/library/${m%%:*}" | head -1
done
```

Expect: all four return `200`. If any returns `404`, replace that row before committing — see exp-002 H3 / kill criteria. Do **not** invent a substitute; flag back to the human.

## Rollback plan

Single-file insertion, single-commit revert:

```bash
git diff -- docs/index.html        # review the insertion
git checkout -- docs/index.html    # revert
```

No container restart, no env change, no deploy step beyond GitHub Pages re-publish (automatic on `main` push).

## Kill criteria (G3 → done blocked)

This PRD is **not** considered shipped, and the change must be reverted, if any of:

- V1 visual smoke fails (banner not eye-catching, wrong row content, wrong colours).
- V2 reveals broken layout below 768 px width that the inline overflow-x fix doesn't resolve.
- V3 shows the existing "Recommended models" or "Platforms" sections drifted by even one line — i.e. the diff is not pure insertion.
- V4 finds any of the four tags has been renamed/removed upstream and a replacement is not yet agreed with the human.

## Files touched (explicit, no scope expansion)

- [docs/index.html](../../docs/index.html) — insertion of one new `<section id="heavyweight">` between line 300 and line 303.

Explicitly **not** touched (would be scope expansion):

- [docs/_config.yml](../../docs/_config.yml) — no SEO/title change needed.
- [docs/chat.html](../../docs/chat.html), [docs/status.html](../../docs/status.html) — heavyweight tier is not relevant on those pages.
- [README.md](../../README.md) — defaults are unchanged; README continues to point at `mistral:7b-instruct-q4_K_M`.
- [scripts/](../../scripts/) — no new helper script (NOT-in-Scope, PROP-003 territory).
- [docker-compose.yml](../../docker-compose.yml), [docker-compose.gpu.yml](../../docker-compose.gpu.yml), [.env.example](../../.env.example) — defaults unchanged.

## Out-of-scope reminders (carry-forward to PROP if discovered during execution)

- Restructure of "Recommended models" into tiered table → PROP-002 (defer until ≥4 tiers exist).
- `scripts/pull-heavyweight.sh` helper → PROP-003 (defer until pulling is itself a friction point).
- Quarterly review automation for the curated list → PROP-004.
- Adding `#heavyweight` to the header nav → PROP-005 (revisit if section becomes a primary entry point).
- Vision / multimodal model tier → PROP-006.

## Evidence block

```yaml
evidence:
  hypothesis: A single-file, additive section insertion (HTML only, reusing existing styles, no nav change) ships the heavyweight-tier guidance without touching defaults or other docs and passes V1–V4 in one render cycle.
  result:     V3 ✅ (`git diff --stat`: 68 insertions, 0 deletions — pure additive). V4 ✅ (all four model families return 200 on ollama.com/library). V1 ✅ (user confirmed "ok" after rendering on http://localhost:5500). V2 ⏳ (per-viewport responsive check not separately reported by user; carrying assumption that visual review covered narrow widths). See exp-002-insights.md.
  reasoning:  All automated checks passed. Visual smoke approved by the user. Implementation matched prd-002 step-by-step exactly; insertion point at docs/index.html:300-303 was stable between PRD authoring and execution. Section is shipped to working tree; commit is held pending explicit user request, per repo's "never commit unless asked" rule.
```
