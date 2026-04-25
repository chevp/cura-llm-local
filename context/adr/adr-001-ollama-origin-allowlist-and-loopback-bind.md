---
id: adr-001
title: Two-layer network exposure hardening for local Ollama — explicit `OLLAMA_ORIGINS` allowlist + loopback-only port bind
status: accepted-via-override
proposed-by: ai
decided-by: —
approved-by: —
approved-at: —
created: 2026-04-25
override-note: gate-overridden 2026-04-25 (see governance-log.md, transitive via prd-001)
related-plan: exp-001
supersedes: —
---

## Context

The repo runs Ollama in Docker, exposed on the host at `:11434`, intended to be called directly from a browser. Two paths exist:

- **Local development:** `docs/chat.html` opened from disk or a static dev server.
- **Production:** the developer's own webapp at `https://cura.sowuvuma.cyon.site` calling `http://localhost:11434` from the user's browser.

Today both layers of network exposure are wide open:

1. The chat's UX hint (at [docs/chat.html:85-86](../../docs/chat.html#L85-L86)) actively instructs the user to set `OLLAMA_ORIGINS=*`, which permits any web page on the internet to drive the local Ollama as long as it's running.
2. The compose port mapping at [docker-compose.yml:7](../../docker-compose.yml#L7) (`${OLLAMA_PORT:-11434}:11434`) defaults to `0.0.0.0` — i.e. any LAN peer can reach the unauthenticated Ollama.

Threat model: *a malicious page in the user's own browser*, and *any peer on the user's LAN*. Multi-user, public-service, or remote-access scenarios are explicitly out of scope.

## Decision

Apply two independent layers of defense:

1. **Configurable origin allowlist via `OLLAMA_ORIGINS`.**
   - Add `OLLAMA_ORIGINS` to `.env.example` with a default that covers both local development and the public demo:
     `http://localhost,http://localhost:*,http://127.0.0.1,http://127.0.0.1:*,https://chevp.github.io`
   - The `chevp.github.io` entry is required because `docs/chat.html` is publicly hosted on GitHub Pages and must work for anyone cloning + running the repo without further config.
   - Document the production-extension pattern as a `.env.example` comment (append `https://cura.sowuvuma.cyon.site`); production users add it to their **local** `.env`, never the committed example.
   - Pass `OLLAMA_ORIGINS` through `docker-compose.yml` as a container env var with the same default for users who do not maintain a `.env`.
   - Do **not** ship `*` as a default at any layer.
2. **Loopback-only host port bind.**
   - Change the port mapping to `127.0.0.1:${OLLAMA_PORT:-11434}:11434`. The container still listens on `0.0.0.0` internally (Docker network), but the host publishes only on the loopback interface.
3. **Update `docs/chat.html` error hint** so the documented recovery path is the explicit allowlist pattern, not `*`.

The two layers are independent: CORS is enforced by the browser at HTTP level; loopback bind is enforced by the kernel at socket level. Either alone closes the dominant attack vector; together they form defense-in-depth.

`OLLAMA_ORIGINS` is treated as **non-secret configuration**. It is a list of public hostnames; no GitHub-Secrets or credential store is needed. `.env` stays git-ignored (per-machine reality), `.env.example` is committed and documents the schema.

## Consequences

### Positive
- Browser pages from non-listed origins cannot drive the local Ollama (CORS layer).
- LAN peers cannot reach Ollama on port 11434 (loopback layer).
- Production origin is opt-in via the user's own `.env`, so the default checked-in config is safe even if a user clones-and-runs without reading anything.
- Reversible — the change is three small edits across two files plus a docs-hint update.
- No new infrastructure (no reverse proxy, no TLS, no auth service).

### Negative
- Two configs to keep in sync (`.env` and `.env.example`).
- Each new dev tool that runs from a fresh port needs a one-time addition to `OLLAMA_ORIGINS` (Vite default `5173`, Live Server `5500`, etc.) — wildcard-port behavior `:*` mitigates this *if* the running Ollama version honors it; otherwise enumerate.
- HTTP remains in use locally — no TLS guarantees against on-host process snooping. Acceptable in single-user scope; revisit if scope expands.

### Neutral
- GPU override (`docker-compose.gpu.yml`) does not redeclare `ports`, so the loopback bind from the base file applies under compose merge rules. Verified during PRD validation, not at decision time.

## Alternatives considered

### Alt-1: Reverse proxy (Caddy) terminating CORS in front of an internal-only Ollama
**Rejected.** Adds a service, a Caddyfile, optional local-CA TLS cert management, and a new failure mode — to solve a problem that one `.env` line plus one `ports:` line already solve at the browser layer.
**Reopening condition:** Mixed Content blocks `http://localhost` from `https://cura.sowuvuma.cyon.site` on a current-major browser; or API-key auth becomes a requirement; or Ollama becomes shared across multiple users.

### Alt-2: Server-side proxy at sowuvuma via SSH tunnel
**Rejected.** Contradicts the explicit "browser → localhost" topology and adds backend code at the cyon-hosted webapp. Massive scope expansion for no current benefit.
**Reopening condition:** Ollama becomes a shared/managed/remote instance, or we need to drive it from devices other than the host workstation (mobile/tablet).

### Alt-3: `OLLAMA_ORIGINS=*`
**Rejected.** This is the current de-facto state via the misleading UX hint. It permits any internet page to drive a running local Ollama. Concretely undermined by the new direction.
**Reopening condition:** none. This is the anti-pattern we are removing.

### Alt-4: mTLS or WebAuthn-style attestation
**Rejected.** Solves a problem we don't have (cryptographic origin attestation in a single-user, same-host context). The cost is a CA setup the user would never roll for personal-use Ollama.
**Reopening condition:** Ollama is exposed beyond loopback (LAN dev share, remote pair, prod inference) and we need cryptographic origin guarantees.

## References
- Plan: [exp-001-cors-and-loopback-binding.md](../plans/exp-001-cors-and-loopback-binding.md)
- Code under change: [docker-compose.yml](../../docker-compose.yml), [.env.example](../../.env.example), [docs/chat.html:85-86](../../docs/chat.html#L85-L86)
