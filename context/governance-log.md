# governance-log

Append-only. One line per gate event: `YYYY-MM-DD <gate> <plan-id> <verdict> <approved-by>`.

2026-04-25 G2+G3 prd-001 gate-override chevp (reason: "ok"; transitively overrides exp-001 + adr-001 since prd-001 depends on both)
2026-04-26 G2 exp-002 pass chevp (reason: "approve exp-002"; H1/H2 accepted on user authority without external verification — see exp-002 evidence block)
2026-04-26 G3 prd-002 pass chevp (reason: "continue"; implementation authorised — single insertion in docs/index.html)
