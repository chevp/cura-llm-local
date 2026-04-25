# CLAUDE.md — cura-llm-local

Operating system for AI work in this repo: **chevp-ai-framework**.
Authoritative source: <https://chevp.github.io/chevp-ai-framework/chevp-ai-framework.md> · <https://github.com/chevp/chevp-ai-framework>

> Core principle: *"The human writes naturally. The AI owns the process."*
> Code cannot exist without prior specification. Process drives progress.

---

## Project at a glance

- **What:** Local Ollama runtime in Docker (port `11434`, persistent named volume).
- **Who it serves:** Cura developers needing dev/prod parity with the AWS Linux/Docker target.
- **Components:** `docker-compose.yml` (CPU baseline), `docker-compose.gpu.yml` (NVIDIA override), `scripts/*.sh` (helper scripts), `docs/` (GitHub Pages site), `.env.example`.
- **Non-goals:** Native Ollama install path, model fine-tuning, application code.
- **Language:** User communicates in German; documentation and code stay in English unless explicitly requested otherwise.

---

## Lifecycle: Context (G1) → Exploration (G2) → Production (G3)

Sequential. No step skippable. Gates are **blockers** requiring evidence-based human approval.

| Step | Reduces uncertainty about | Key deliverables |
|------|---------------------------|------------------|
| **Context (G1)** | *What* the problem is and *who* has it | Context-Plan, Problem Statement, ≥2 Hypotheses w/ kill criteria, ≥3 Risks, System Spec, Architecture, ADRs, Context Inventory, Scope Confirmation |
| **Exploration-A (G2)** | *Whether the problem framing is right* | Updated hypotheses (confirmed/killed), `insights.md` |
| **Exploration-B (G2)** | *Which solution is best* | Feature Plan/Spec with `exploration-mode: B`, ≥2 comparable candidates, ADR, Challenger output, `insights.md` |
| **Production (G3)** | *Whether the chosen solution actually ships* | Production-Plan (PRD), step-by-step implementation, validation, regression checks |

### Mode inference

The AI infers the mode from intent — the human does **not** declare it.

| Mode | Signals | Allowed | Forbidden |
|------|---------|---------|-----------|
| Context | "what does", "explain", "analyse", "why does" | Read/verify, ask questions, write CTX artifacts | Code changes, feature plans |
| Exploration | "plan", "design", "spec", "how should we" | Feature Plan/Spec, ADRs, prototypes, risks | Production code, scope expansion |
| Production | "implement", "build", "execute the plan", "fix" (with approved PRD) | Execute approved plan, build, test, commit | New plans, scope expansion, unplanned changes |

Mixed intent → decompose by lifecycle order, execute earliest portion, stop at the gate.

### Evidence block (every plan frontmatter)

```yaml
evidence:
  hypothesis: <falsifiable belief before this gate>
  result:     <observable outcome>
  reasoning:  <why result justifies proceed / fall back / kill>
```

Empty or boilerplate `evidence:` is automatic gate failure.

### Kill Criteria (every plan)

Each plan answers: *what evidence would tell us not to advance?*
Bad: "if it does not work." Good: "if container cold-start exceeds 30s on M1 Air after step 3."

### `insights.md`

Every Exploration produces `insights.md` before G2 passes (confirmed/killed hypotheses, surprises, what remains unknown, consequence). Empty file = G2 blocker.

### Challenger output (required before G2)

1. Top-3 concrete failure modes with observable signals
2. ≥2 genuinely different alternatives with rejection reason and reopening condition
3. Strongest counter-argument against the chosen approach (charitable framing)

Generic objections ("scope creep", "use a different tool") trigger automatic block.

---

## Context hierarchy

```
CLAUDE.md                      ← always read first
context/
├── architecture/              ← system + software architecture docs
├── adr/                       ← Architecture Decision Records
├── guidelines/                ← extension points; AI must enforce when present
├── plans/
│   ├── proposals/             ← PROP-NNN out-of-scope items (and rejected/)
│   └── finished/              ← completed PRD + EXP plans land here
└── specs/                     ← feature specifications
```

ADR / plan IDs use kebab-case prefixes: `ctx-NNN` · `exp-NNN` · `prd-NNN` · `adr-NNN` · `PROP-NNN`. Filenames mirror the ID (`exp-001-foo.md`). Never use parens (`exp(001)` etc.) — neither in filenames nor in IDs/prose/frontmatter.

---

## Provenance (every governed artifact)

```yaml
proposed-by: ai          # ai | human | pair
decided-by: —            # human only
approved-by: —           # human identifier
approved-at: —           # YYYY-MM-DD
```

The AI **must not** write `decided-by`, `approved-by`, or `approved-at`. Only the human, via `/approve <artifact-id>`, sets these. Every gate crossing appends a line to `governance-log.md`.

---

## Gatekeeper subagents

| Agent | Validates | Verdict | Spawns proposals from |
|-------|-----------|---------|------------------------|
| `gatekeeper-g1` | Context → Exploration | `pass` / `block` / `conditional-pass` | NOT-in-Scope items in CTX plan |
| `gatekeeper-g2` | Exploration → Production | same | NOT-in-Scope items + Challenger failure modes |
| `gatekeeper-g3` | Production → Done | same | Follow-ups discovered during Production |

Out-of-scope items become `PROP-NNN` stubs (max 5 per gate-check; excess collapsed into a Sammel-Notiz). Triage commands: `/promote`, `/defer`, `/reject <reason>`, `/gate-override <plan-id> <reason>`. Auto-defer after 90 days. Promotion is human-only.

---

## Repo-specific conventions

- **Surface area is small.** Most changes touch `docker-compose*.yml`, `scripts/*.sh`, `.env.example`, or `docs/`. Use the framework's **Abbreviations** liberally — but never skip a step entirely.
- **Container-first.** Never recommend `brew install ollama` as the supported path; the README explicitly excludes it. Parity with AWS Linux/Docker is the design constraint.
- **macOS is CPU-only.** Apple GPU is not exposed to Docker. Stick to q4-quantised 7B–8B models in defaults and examples.
- **Linux GPU is opt-in.** GPU support lives in the override file and requires `nvidia-container-toolkit`.
- **Default port is `11434`.** Make it configurable via `OLLAMA_PORT` in `.env` — do not hard-code overrides.
- **Scripts are bash, executable, and idempotent.** Pulling an existing model must succeed.
- **Docs site is GitHub Pages from `docs/`.** `_config.yml` + `index.html`. Don't introduce a build step without an ADR.

### Abbreviations applicable here

| Scenario | Allowed |
|----------|---------|
| README typo / wording (<10 lines) | CTX + EXP verbal; PRD one-liner. Context deliverables read/verified. |
| Script tweak / refactor | Skip UX-Tooling in EXP/PRD |
| Adding/removing a model from defaults | Verbal CTX, written EXP with kill criteria, short PRD |
| Compose / env-var change | Written EXP + ADR; PRD step-by-step |
| Docs site change (HTML/CSS) | Visual feature — UX prototype mandatory before G2 |
| GPU override change | Written EXP + ADR; PRD step-by-step; manual verification path documented |

No step is ever entirely skipped. Human approval is always required.

---

## Commit & delivery rules

- Stage **specific** files only — never `git add .` / `git add -A`.
- Prefix commits: `ctx-NNN:` · `exp-NNN:` · `prd-NNN:` · or conventional (`chore:`, `fix:`, `docs:`) for non-governed changes.
- Messages explain **what + why**, not the diff.
- Never commit non-compiling code, never `--no-verify`, never `--force-push` without explicit instruction.
- Never commit/push unless asked.
- Move completed plans to `context/plans/finished/` (PRD + EXP) on G3 close.

---

## Hard rules — MUST NOT

- Silently switch modes; require the human to declare modes or use structured prompts.
- Skip steps, gates, or modes.
- Expand scope ("I also improved X").
- Write production code in Context or Exploration.
- Start implementing without an approved PRD.
- Skip a UX prototype for visual output (docs/index.html counts).
- Approve a gate with empty/boilerplate `evidence:`.
- Approve a plan without a `Kill Criteria` section.
- Submit generic Challenger output.
- Promote/defer/reject `PROP-NNN` on behalf of the human.
- Write `decided-by`, `approved-by`, or `approved-at`.
- Reference plans/ADRs/artifacts inside inline code comments — comments must be self-contained.
- Re-analyse or re-justify previously-approved artifacts.

---

## First-touch checklist for the AI

1. Read this file.
2. Read `README.md`, `docker-compose.yml`, `docker-compose.gpu.yml`, `.env.example`, `scripts/*.sh`.
3. Walk `context/` — note what exists vs. what's missing.
4. Identify the user's intent → infer mode → output the appropriate header.
5. If `context/` is empty for the relevant artifact, the first deliverable is the missing CTX/EXP artifact, **not** code.
