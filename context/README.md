# context/

Governed artifacts for the chevp-ai-framework lifecycle. See [`../CLAUDE.md`](../CLAUDE.md).

```
architecture/   System + software architecture docs
adr/            Architecture Decision Records (ADR-NNN)
guidelines/     Extension points; AI must enforce when present
plans/
  proposals/    PROP-NNN out-of-scope items (rejected/ for archived)
  finished/     Completed PRD + EXP plans
specs/          Feature specifications
```

Plan ID prefixes (kebab-case, no parens): `ctx-NNN` · `exp-NNN` · `prd-NNN` · `adr-NNN` · `PROP-NNN`. Filenames mirror the ID — e.g. `exp-001-foo.md`.
Every governed artifact carries provenance frontmatter (`proposed-by` / `decided-by` / `approved-by` / `approved-at`) — the AI fills only `proposed-by`.
