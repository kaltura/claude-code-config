---
name: authoring-workflows
description: Patterns and pitfalls for writing Workflow scripts (pipeline vs parallel, per-item model tiers, schema null-handling, phase-title matching, loop-until-dry, budget-aware scaling). Use when authoring or reviewing a Workflow script.
---

# Authoring Workflow scripts

- **`pipeline()` by default, `parallel()` only for genuine barriers.** `pipeline` streams each item through all stages independently. Use `parallel` only when stage N needs ALL of stage N-1's results (dedup, early-exit, cross-item synthesis). A filter between pipelines is correct; a barrier just to flatten is not.
- **Batch per-item relay calls into one call per unit of work.** If a stage's only job is to relay an already-computed result through a `schema` (not compute anything new), don't fan out one relay call per sub-item — one per grading dimension, one per file, one per audit step. Combine the sub-items into one payload and relay them in a single call. Relay calls are cheap per call but almost always cold (no shared context to cache), so N calls per item multiplies cache-write overhead N-fold and can erase a cheaper model's entire rate-card advantage — measured case: a model with a 3x rate-card discount landed at near-parity blended cost because its cache was reused 1.8x per write vs. 13x for the pricier model, traced to 10 per-dimension relay calls per fixture instead of 1. Exception: skip batching when sub-items are genuinely sequential with no shared upfront data (each depends on the prior step's live state) — check the dependency shape before assuming it applies.
- **Always set `model` on per-item agents.** Set `model: 'sonnet'` (or `'haiku'`) on every per-item stage; omit only on judge/synthesis.
- **`schema` returns `null` on failure — always `.filter(Boolean)` before any `.map`/`.reduce`/property access.**
- **`phase()` and `agent({phase:})` titles must match `meta.phases[].title` exactly** — a typo silently orphans agents. `meta` must be a pure object literal (no variables, template literals, or function calls).
- **Loop-until-dry:** `while (dry < 2) { if (!fresh.length) { dry++; continue } dry = 0; ... }` Track seen items with a `Set`.
- **Budget-aware scaling:** guard loops with `while (budget.total && budget.remaining() > 50_000)`.
