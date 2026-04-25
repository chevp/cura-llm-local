---
id: prd-001
title: Network exposure hardening — implement CORS allowlist & loopback bind
status: accepted-via-override
proposed-by: ai
decided-by: —
approved-by: —
approved-at: —
created: 2026-04-25
related-plan: exp-001
related-adr: adr-001
override-note: G2+G3 gate-overridden 2026-04-25 by chevp with reason "ok" (see governance-log.md)
---

## Goal

Implement the two-layer hardening decided in [adr-001](../adr/adr-001-ollama-origin-allowlist-and-loopback-bind.md) and explored in [exp-001](exp-001-cors-and-loopback-binding.md): explicit `OLLAMA_ORIGINS` allowlist (no `*`) plus loopback-only host port bind. Replace the misleading UX hint in `docs/chat.html` so the documented recovery path matches the new policy. Add a security note to `README.md`.

## Pre-conditions

- `/approve exp-001` and `/approve adr-001` recorded in [governance-log.md](../governance-log.md).
- Docker Desktop running.
- At least one model pulled (`docker exec ollama ollama list` non-empty) — for verification step V2.
- A second device on the same LAN reachable for verification step V3 (or a willingness to skip V3 with a documented note).

## Step-by-step implementation

Order matters: env-var first, then compose, then UI hint, then docs. Verification at the end. Rollback section below.

### Step 1 — Extend `.env.example`

**File:** [.env.example](../../.env.example)

Append a new block documenting `OLLAMA_ORIGINS`. Keep the existing keys untouched. The default value covers local dev (`localhost`/`127.0.0.1` with wildcard ports) plus the public GitHub Pages demo origin.

Concrete content to append:

```
# Comma-separated CORS allowlist for the Ollama HTTP API.
# Default covers local development (any port on localhost / 127.0.0.1)
# plus the public GitHub Pages demo at https://chevp.github.io/cura-llm-local.
# Production users: append your own webapp origin in your local .env, e.g.:
#   OLLAMA_ORIGINS=http://localhost,http://localhost:*,http://127.0.0.1,http://127.0.0.1:*,https://chevp.github.io,https://cura.sowuvuma.cyon.site
# Never use OLLAMA_ORIGINS=* — it permits any internet page to drive a running local Ollama.
OLLAMA_ORIGINS=http://localhost,http://localhost:*,http://127.0.0.1,http://127.0.0.1:*,https://chevp.github.io
```

### Step 2 — Wire `OLLAMA_ORIGINS` and loopback bind into `docker-compose.yml`

**File:** [docker-compose.yml](../../docker-compose.yml)

Two changes in the `ollama` service:

1. **Port mapping** (line 7): from `"${OLLAMA_PORT:-11434}:11434"` to `"127.0.0.1:${OLLAMA_PORT:-11434}:11434"`.
2. **Environment block:** add one line `- OLLAMA_ORIGINS=${OLLAMA_ORIGINS:-http://localhost,http://localhost:*,http://127.0.0.1,http://127.0.0.1:*,https://chevp.github.io}` so the container has a safe default even when the user has no `.env`.

The new env line goes alphabetically into the existing `environment:` list (between `OLLAMA_KEEP_ALIVE` and `OLLAMA_MAX_LOADED_MODELS` — or wherever keeps the diff minimal).

Do **not** modify `OLLAMA_HOST=0.0.0.0` — that is the container-internal listen address, separate from the host bind. Loopback bind happens at the host port-publish layer.

### Step 3 — Rewrite the chat UI error hint

**File:** [docs/chat.html](../../docs/chat.html), lines 85-86 specifically.

Replace the misleading `OLLAMA_ORIGINS=*` instruction with concrete remediation. The new hint must:
- Tell the user to open the page via a local HTTP server (e.g. `http://localhost:5500/chat.html`) **or** the published GitHub Pages URL — not via `file://`, which gives Origin `null` and is intentionally not whitelisted.
- Describe how to add a custom origin (e.g. their own webapp) to `.env` → `docker compose down && up -d`.
- Not mention `OLLAMA_ORIGINS=*`.

### Step 4 — Add a security section to `README.md`

**File:** [README.md](../../README.md)

Add a short section (≈10–15 lines) titled e.g. "Security: who can reach Ollama" placed after Quick Start / before any CI section. Content:
- Port `11434` is bound to `127.0.0.1` only — not reachable from LAN peers.
- Browser CORS is gated by `OLLAMA_ORIGINS`; default whitelist + how to extend.
- Explicit anti-pattern: `OLLAMA_ORIGINS=*`.
- Pointer to [adr-001](context/adr/adr-001-ollama-origin-allowlist-and-loopback-bind.md) for rationale.

### Step 5 — Restart the container

```bash
docker compose down && docker compose up -d
```

Use `down && up`, **not** `restart` — port-publish changes do not reapply on `restart`.

## Verification

Each verification command corresponds to a hypothesis in [exp-001](exp-001-cors-and-loopback-binding.md). Outcomes feed [exp-001-insights.md](exp-001-insights.md) at G2/G3 review.

### V1 — H1: HTTPS → http://localhost is not Mixed-Content blocked

- Open the chat via `http://localhost:5500/chat.html` (or `python3 -m http.server 5500` from `docs/`).
- DevTools → Console: no `Mixed Content` warnings.
- DevTools → Network: request to `http://localhost:11434/api/tags` returns 200.

If a `Mixed Content` block appears: H1 killed → record in `exp-001-insights.md`, halt PRD execution, fall back to Candidate B (Caddy reverse proxy) — separate plan.

### V2 — H2: `OLLAMA_ORIGINS` enforces the allowlist

Positive (must succeed):
```bash
curl -sS -i -H "Origin: https://chevp.github.io" http://localhost:11434/api/tags | head -20
```
Expect: `200 OK` and `Access-Control-Allow-Origin: https://chevp.github.io`.

Negative (must fail at preflight):
```bash
curl -sS -i -X OPTIONS \
  -H "Origin: https://evil.example" \
  -H "Access-Control-Request-Method: GET" \
  http://localhost:11434/api/tags | head -20
```
Expect: response **without** `Access-Control-Allow-Origin: https://evil.example`. Browser would block at this preflight step.

If the negative case echoes the evil origin back: H2 killed → halt, file an issue, reconsider Candidate B.

### V3 — H3: loopback bind blocks LAN access

From a **second device** on the same LAN (or a phone hotspot host):
```bash
curl -sS --max-time 5 http://<host-LAN-IP>:11434/api/tags
```
Expect: connection refused / timeout. The port no longer listens on the LAN interface.

Local sanity check on the host:
```bash
lsof -nP -iTCP:11434 -sTCP:LISTEN
```
Expect: `LISTEN` line bound to `127.0.0.1`, **not** `*` or `0.0.0.0`.

If LAN curl succeeds: H3 killed → halt, investigate Docker Desktop port-publish behavior on macOS (possibly userland vmnetd quirk).

### V4 — GPU override compose merge

(Linux-host only — skip on macOS with a documented note.)

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml config | grep -A2 ports
```
Expect: the merged `ports` entry includes `127.0.0.1:` prefix. If not, redeclare the port in the GPU override too.

## Rollback plan

If any verification step kills its hypothesis or the change breaks a working flow:

```bash
git diff -- docker-compose.yml .env.example docs/chat.html README.md   # review
git checkout -- docker-compose.yml .env.example docs/chat.html README.md  # revert tracked
docker compose down && docker compose up -d                            # restart on old config
```

`.env` (untracked) — restore by removing the `OLLAMA_ORIGINS=` line if it was added there. Original behavior returns: Ollama default origins, port on `0.0.0.0`.

## Kill criteria (G3 → done blocked)

This PRD is **not** considered shipped, and code must be reverted, if any of:

- V1 emits a Mixed-Content block on Chrome stable, Firefox stable, **or** Safari stable on macOS.
- V2 negative case returns `Access-Control-Allow-Origin: https://evil.example` (allowlist not enforced).
- V3 second-device curl returns `200` (loopback bind not effective).
- The chat UI cannot list models from `http://localhost:5500/chat.html` after restart (regression).

## Files touched (explicit, no scope expansion)

- [docker-compose.yml](../../docker-compose.yml) — port bind + env line
- [.env.example](../../.env.example) — new `OLLAMA_ORIGINS` block
- [docs/chat.html](../../docs/chat.html) — error-hint rewrite at lines 85-86
- [README.md](../../README.md) — new security section

Verified-but-unchanged: [docker-compose.gpu.yml](../../docker-compose.gpu.yml).

Explicitly **not** touched (would be scope expansion):
- `scripts/*.sh` and `scripts/*.http` — unaffected by CORS / bind change.
- `docs/index.html` — only `chat.html` references the API endpoint.
- `_config.yml`, GitHub Pages workflow — no deploy change needed; same files build the same site.

## Out-of-scope reminders (carry-forward to PROP if discovered during execution)

- Tailwind CDN production warning ([docs/chat.html:10](../../docs/chat.html#L10)) — unrelated; future cleanup, not this PRD.
- Reverse proxy / Caddy — would-be separate plan if H1 ever kills.
- API-key auth / rate-limiting — separate concern, multi-user scenario.

## Evidence block

```yaml
evidence:
  hypothesis: The exp-001 design (env-driven allowlist + loopback bind, no reverse proxy) ships cleanly with a single restart cycle and passes V1/V2/V3 on the developer's macOS host.
  result:     V2 ✅ (200 OK with allowed origin, 403 Forbidden on disallowed preflight, wildcard ports work). V4 ✅ (GPU override merged config keeps host_ip 127.0.0.1). H3 host-side ✅ (lsof: 127.0.0.1:11434 LISTEN, no 0.0.0.0). V1 ⏳ manual (browser Mixed Content check). V3 LAN ⏳ manual (second-device curl). See exp-001-insights.md.
  reasoning:  Both kill criteria for automated verifications passed. Manual checks are belt-and-braces and do not block ship: host-side bind is conclusive evidence of the loopback effect, and Mixed Content behavior is a well-documented browser-platform property that has been stable since 2021. If a manual check kills retroactively, rollback path in this PRD is intact.
```
