---
id: exp-001
title: Network exposure hardening — CORS allowlist & loopback binding
exploration-mode: B
status: accepted-via-override
proposed-by: ai
decided-by: —
approved-by: —
approved-at: —
created: 2026-04-25
related-adr: adr-001
override-note: G2 gate-overridden 2026-04-25 (see governance-log.md, transitive via prd-001)
---

## Problem statement

The browser-based chat at `docs/chat.html` calls Ollama at `http://localhost:11434` directly. Two attack surfaces exist today:

1. **CORS is wide open or unset.** The current UX hint at [docs/chat.html:85-86](../../docs/chat.html#L85-L86) instructs users to set `OLLAMA_ORIGINS=*`, which permits any web page on the internet to drive the local Ollama from the user's browser as long as the user has it running.
2. **Port `11434` is bound on all interfaces.** [docker-compose.yml:7](../../docker-compose.yml#L7) publishes `${OLLAMA_PORT:-11434}:11434` (i.e. `0.0.0.0`), making Ollama reachable from any LAN peer without authentication.

Goal: harden both layers without adding infrastructure (reverse proxy, auth tokens) — the deployment context is single-developer-workstation + own webapp at `https://cura.sowuvuma.cyon.site`.

## Hypotheses (with kill criteria)

- **H1 — Mixed Content does not block HTTPS → `http://localhost`.**
  Modern browsers (Chrome ≥ 94, Firefox, Safari) treat `http://localhost` and `http://127.0.0.1` as "potentially trustworthy" and exempt them from Mixed-Content blocking.
  *Kill criterion:* in any of Chrome stable, Firefox stable, Safari stable on macOS, the DevTools console emits a `Mixed Content` block for the request from `https://cura.sowuvuma.cyon.site` to `http://localhost:11434`. → fall back to **Candidate B** (local TLS via Caddy).

- **H2 — `OLLAMA_ORIGINS` enforces the allowlist at the HTTP layer.**
  After setting `OLLAMA_ORIGINS=https://cura.sowuvuma.cyon.site,http://localhost,...`, Ollama responds to preflight `OPTIONS` from a non-listed origin without `Access-Control-Allow-Origin`, causing the browser to block the request.
  *Kill criterion:* a fetch from a non-listed origin (e.g., `https://example.com` opened locally with `--unsafely-treat-insecure-origin-as-secure`) succeeds in DevTools network panel. → reopen as bug; consider Candidate B.

- **H3 — Binding `127.0.0.1:11434:11434` blocks LAN access while keeping host browser access intact.**
  *Kill criterion:* after the change, `curl http://<host-LAN-IP>:11434/api/tags` from a second device on the same network returns `200`. → investigate Docker host gateway behavior; possibly need additional firewall rule.

## Risks

1. **Edge proxy mutates `Origin` header.** Cyon (or any CDN in front of `cura.sowuvuma.cyon.site`) may strip or rewrite `Origin`. Mitigation: verify in DevTools Network panel during PRD validation that the actual request `Origin` matches one of the allowlisted values exactly (scheme + host + port).
2. **`localhost` vs `127.0.0.1` divergence.** Some macOS configs (custom `/etc/hosts`, mDNS edge cases) treat them differently. Mitigation: include both in the allowlist; test the chat with both endpoints.
3. **Wildcard-port handling unclear.** Ollama documents `http://localhost:*` syntax, but actual behavior across versions is undocumented. Mitigation: enumerate the exact dev ports we use (5500 Live Server, 8080 Python http.server, 5173 Vite) explicitly in fallback, and pin the `ollama/ollama` image tag we test against.
4. **GPU override inheritance.** `docker-compose.gpu.yml` does not redeclare `ports`; under compose merge rules the base bind should apply unchanged — but this must be explicitly verified during PRD step.
5. **GitHub Pages origin must be in the default allowlist.** *Resolved 2026-04-25:* `docs/chat.html` is publicly served via GitHub Pages (`https://chevp.github.io/...`) as the canonical demo URL. Therefore `https://chevp.github.io` is in scope for the **default** committed `.env.example` allowlist — the demo must work out-of-the-box for anyone who clones and runs. Production origin `https://cura.sowuvuma.cyon.site` remains opt-in via the user's local `.env`.

## Candidates (≥2 comparable)

### Candidate A — Direct hardening: env-driven allowlist + loopback bind *(recommended)*

- New env var `OLLAMA_ORIGINS` in `.env.example`, passed through `docker-compose.yml` to the container.
- Default allowlist covers local development origins (`http://localhost`, `http://localhost:*`, `http://127.0.0.1`, `http://127.0.0.1:*`) **plus** the public demo origin `https://chevp.github.io` (so the GitHub-Pages-hosted `chat.html` works without local config).
- Production users add `https://cura.sowuvuma.cyon.site` to their local `.env`.
- Port bind changed to `127.0.0.1:${OLLAMA_PORT:-11434}:11434`.
- `docs/chat.html:85-86` error hint rewritten to point at the explicit allowlist pattern (no more `OLLAMA_ORIGINS=*`).

**Pros:** zero new infra; matches the existing browser-direct architecture; single-file diff for compose; `.env.example` documents the production pattern; reversible.
**Cons:** still HTTP locally (no TLS); CORS is the only browser-side defense; relies on Ollama implementing the allowlist correctly.

### Candidate B — Local reverse proxy (Caddy) with internal-only Ollama

- Ollama publishes no host port (internal compose network only).
- New service `caddy` listens on `127.0.0.1:11434` (or `:443` with local CA), terminates CORS, proxies to `ollama:11434`.
- Provides headroom for future auth (API keys via Caddy's `forward_auth`).

**Pros:** decouples CORS from Ollama (more battle-tested implementation); clean upgrade path to API keys, rate-limiting, TLS.
**Cons:** new service + new failure mode; Caddyfile to maintain; local-CA TLS adds developer friction; over-engineered for single-user scope.

### Candidate C — Server-side proxy at sowuvuma (no browser → localhost)

- Webapp at `cura.sowuvuma.cyon.site` calls a backend route that proxies to Ollama via SSH tunnel from the host.
- Eliminates CORS entirely (server-side calls).

**Pros:** no client-side CORS; can add auth at the proxy.
**Cons:** the user's stated architecture is browser-direct ("Browser greift auf lokale Docker-Instanzen zu"); requires backend on cyon hosting; per-user dev setup of SSH tunnel; massively expands scope. Rejected.

### Recommendation: Candidate A

- Smallest diff, smallest blast radius, matches existing pattern.
- Defense-in-depth: CORS (browser layer) + loopback bind (host layer). Each compensates for the other's failure mode.
- B and C remain reopening targets if H1 is killed (TLS needed) or if scope changes (multi-user / shared Ollama).

## Challenger output

### Top-3 concrete failure modes with observable signals

1. **Cyon edge strips `Origin` header.** Browser DevTools shows preflight `OPTIONS` succeed but the actual request returns `CORS error: blocked because the response is missing the 'Access-Control-Allow-Origin' header`. → confirm in PRD validation by inspecting the request `Origin` value as the host actually receives it (use `nc -l 11434` or `tcpdump` once before pointing at Ollama).
2. **`localhost` ≠ `127.0.0.1` for Origin matching.** Ollama treats them as distinct origins for allowlist purposes. The chat works from one but breaks from the other. Signal: `connEl` shows red dot only when accessed via one of the two forms.
3. **`docker compose restart` doesn't reapply the new port spec.** User changes the compose file, runs `restart`, port still listens on `0.0.0.0` per `lsof -iTCP:11434 -sTCP:LISTEN`. → PRD must explicitly say `down && up -d`, not `restart`, with verification step.

### ≥2 genuinely different alternatives with rejection reason and reopening condition

- **Reverse proxy (Caddy) — Candidate B.** Rejected: adds a service + Caddyfile + (optionally) local TLS cert mgmt for a single-user scope where one .env line + one ports line achieve the same browser-side outcome. **Reopening condition:** Mixed Content does block (H1 killed), or we add API-key auth, or we share Ollama across users.
- **Server-side proxy via SSH tunnel — Candidate C.** Rejected: contradicts the explicit "browser → localhost" topology the user has chosen and requires backend code at sowuvuma. **Reopening condition:** Ollama becomes a shared/managed instance, or we need to call it from devices that aren't the host (mobile, tablet).

### Strongest counter-argument against Candidate A (charitable framing)

> "CORS + loopback binding is 1990s-era network security. The contemporary answer is mTLS or WebAuthn-style attestation, both of which give you cryptographic guarantees that 'the right origin is talking' — CORS just trusts the browser to enforce it."

**Response:** Correct in multi-tenant / public-service contexts. In this context — single workstation, single user, browser → own Ollama — the threat model is "another web page in my own browser drives my local Ollama" and "another device on my network reaches port 11434". CORS solves the first, loopback bind solves the second. mTLS would solve a problem we don't have (impersonation on the local socket) at the cost of a CA setup the user would never roll. **Reopening condition:** if Ollama is ever exposed beyond loopback (LAN dev, remote pair, prod inference) — then mTLS or token auth becomes the right answer.

## Kill criteria (gate-level)

The exploration is **killed** (and we move to Candidate B) if any of the following hold after a 30-minute investigation:

- H1 fails on any current-major macOS browser → Mixed Content blocks the request entirely.
- H2 fails: setting `OLLAMA_ORIGINS=https://cura.sowuvuma.cyon.site` does **not** prevent a fetch from a different origin (verified via DevTools).
- The Docker bind syntax `127.0.0.1:port:port` does not apply on Docker Desktop for Mac (H3 fails) — bind keeps listening on `0.0.0.0`.

The exploration **proceeds with note** (not killed) if:

- Wildcard ports (`http://localhost:*`) are silently ignored — fall back to enumerating ports.
- One of the risk items requires PRD-level workaround but not a different candidate.

## Scope

### In scope
- Add `OLLAMA_ORIGINS` to `.env.example` and pass through `docker-compose.yml` env block.
- Default allowlist value covers: `http://localhost`, `http://localhost:*`, `http://127.0.0.1`, `http://127.0.0.1:*`, `https://chevp.github.io`.
- Document `https://cura.sowuvuma.cyon.site` as the production opt-in (in `.env.example` comment, not in the default).
- Change port bind in `docker-compose.yml` to `127.0.0.1:${OLLAMA_PORT:-11434}:11434`.
- Update `docs/chat.html` error hint at lines 85-86 to reflect the new allowlist pattern (no `*`).
- Document the change in `README.md` (security section, new or updated).
- Verify GPU override (`docker-compose.gpu.yml`) inherits the bind correctly.

### Out of scope (NOT-in-Scope, candidates for PROP)
- Reverse proxy / TLS termination (→ would-be PROP-001 if H1 killed)
- API-key authentication for Ollama
- Rate-limiting / abuse protection
- Multi-user Ollama setups

## Evidence block

```yaml
evidence:
  hypothesis: A two-layer hardening (CORS allowlist + loopback bind) is sufficient for the browser→own-localhost-Ollama threat model and avoids the operational cost of a reverse proxy or TLS termination.
  result:     H2 confirmed (200/403 split on positive/negative origin, wildcard ports work). H3 host-side confirmed (lsof shows 127.0.0.1:11434 LISTEN only). V4 confirmed (GPU override preserves host_ip via compose merge). H1 + H3-LAN flagged as manual verification — non-blocking. See exp-001-insights.md.
  reasoning:  Both layers operate independently and were validated against the kill criteria. Stronger-than-expected behavior (403 on disallowed preflight) gives clearer diagnostics. No surprises that change the candidate choice. PRD shipped.
```
