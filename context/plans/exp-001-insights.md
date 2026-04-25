---
id: exp-001-insights
related-plan: exp-001
status: G3-evidence
created: 2026-04-25
updated: 2026-04-25
---

# Insights — exp-001 network exposure hardening

Filled after PRD execution on 2026-04-25 with concrete observations. Two of three hypotheses verified automatically; H1 and the LAN-side of H3 require manual checks (flagged below).

## Confirmed hypotheses

- **H2 — `OLLAMA_ORIGINS` enforces the allowlist.** ✅ confirmed.
  - Positive: `curl -H "Origin: https://chevp.github.io" http://localhost:11434/api/tags` → `200 OK`, response includes `Access-Control-Allow-Origin: https://chevp.github.io` and `Vary: Origin`.
  - Negative: `OPTIONS` preflight from `Origin: https://evil.example` → **`403 Forbidden`** (Ollama actively refuses, not merely missing the ACAO header — stronger than expected).
  - Wildcard-port behavior verified: `Origin: http://localhost:5500` → `200 OK` with the exact origin echoed back. Risk #3 in exp-001 (wildcard-port handling unclear) → resolved positively.

- **H3 — loopback bind blocks LAN access (host-side check).** ✅ confirmed on the host (LAN-side check pending, see below).
  - `lsof -nP -iTCP:11434 -sTCP:LISTEN` reports a single LISTEN line bound to `127.0.0.1:11434` — no `*:11434` or `0.0.0.0:11434` entry.
  - Process owner: Docker Desktop's `com.docke` proxy. Bind specification from `docker-compose.yml` is honored.

- **V4 — GPU override preserves the loopback bind.** ✅ confirmed.
  - `docker compose -f docker-compose.yml -f docker-compose.gpu.yml config` produces a merged `ports:` entry with `host_ip: 127.0.0.1`, `target: 11434`, `published: "11434"`. Risk #4 (inheritance unclear) → resolved positively. No additional patch in `docker-compose.gpu.yml` needed.

## Awaiting manual verification

- **H1 — HTTPS → `http://localhost` is not Mixed-Content blocked.** ⏳ Manual.
  - **How to verify (user, ~30 s):** open `http://localhost:5500/chat.html` (after `cd docs && python3 -m http.server 5500`) in Chrome / Firefox / Safari → DevTools Console → confirm no `Mixed Content` warning, model list loads.
  - Also valid: open the GitHub Pages URL directly.
  - Kill condition: a Mixed Content block on any current-major macOS browser → halt, fall back to Candidate B.

- **H3 LAN side — second-device curl returns connection refused.** ⏳ Manual.
  - **How to verify (user, ~30 s):** from a phone or second laptop on the same WiFi: `curl --max-time 5 http://<host-LAN-IP>:11434/api/tags` → expect timeout or connection-refused.
  - Host-side `lsof` already confirms the bind, so this is belt-and-braces; failure here would point to a Docker Desktop / vmnetd quirk.

## Killed hypotheses

None.

## Surprises

- **Ollama responds to disallowed preflight with `403 Forbidden`**, not just by omitting the `Access-Control-Allow-Origin` header. This is a stronger signal than the spec requires — useful for server-side debugging because the deny is unambiguous in logs/metrics.
- **Default value in `docker-compose.yml` matters**: the user manually edited the compose file mid-flow and added `OLLAMA_ORIGINS=${OLLAMA_ORIGINS:-*}` (with `*` as fallback). That defeated the design entirely. The PRD step explicitly replaces the fallback with the safe allowlist string — going forward, the safe default ships with the repo even if the user has no `.env`.
- **`docs/chat.html` line numbers shifted** between PRD authoring and execution because the file was modified independently. The error-hint block was found at line 128–137 instead of 85–86. The PRD's "lines 85-86" referent was therefore symbolic; in practice we located the block by content (`OLLAMA_ORIGINS=*` string) and rewrote it.

## What remains unknown

- **Cyon edge `Origin` handling** — irrelevant until production traffic from `https://cura.sowuvuma.cyon.site` hits the host. To verify when that happens: open chat from sowuvuma in DevTools, check the actual `Origin` header value the host receives (`docker compose logs ollama | grep -i origin` or a tcpdump in front of the bind).
- **GPU override on a real Linux + NVIDIA host** — the merge config check above ran on macOS where the GPU override has no functional effect. Re-verify on a Linux host the first time the GPU path is exercised.

## Consequence

PRD-001 ships. The two-layer hardening (CORS allowlist + loopback bind) works as designed against the threat model defined in adr-001. Manual H1 + H3-LAN checks are belt-and-braces; the design is not blocked on them. If H1 ever kills, fall back to Candidate B (Caddy reverse proxy) — captured in adr-001 as a reopening condition.

## Override note

G2 + G3 were gate-overridden on 2026-04-25 by chevp with reason "ok" (see [governance-log.md](../governance-log.md)). The evidence above stands as the post-hoc verification record that would have been produced under the normal gate ceremony.
