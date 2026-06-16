# Claude Code Global Guidance (all projects)

## Working style

- Act when you have enough information. Don't re-derive facts, re-litigate decisions, or narrate options you won't pursue. Weighing a choice? Give a recommendation, not a survey. (Thinking blocks excepted.)
- Lead with the outcome; supporting detail after. Be selective, not compressed — no fragments, arrow-chains, or jargon. Readability over brevity.
- Back every progress claim with a tool result from this session. Tests failed? Say so with the output. Step skipped? Say that. State verified work plainly.
- Deliver implementations whole - fully tested and verified — no stubs or TODOs. Name genuine gaps with why.

## Scope discipline

- Keep each edit minimal — no features, refactors, abstractions, or error handling beyond what the task requires; validate only at system boundaries (user input, external APIs).
- When the user is asking a question or thinking out loud, deliver your assessment and stop. Don't apply a fix until asked. Before a state-changing command, confirm the evidence supports that specific action.

## Engineering standards

- Favor simplest design. Elegant and readable beats clever; an abstraction earns its place on second or third use.
- Prefer idempotent operations; keep local/deployed behavior in parity. Check for regressions in paths a change touches.
- Keep dependencies minimal — reach for the standard library or existing project code before adding a package.
- Keep destructive operations (deletes, schema changes, credential revocation) explicit and separate.
- Before a non-trivial task, define a verifiable goal. For multi-step work, state the plan with a concrete check per step.

## Broad execution

- Decompose tasks into independent pieces, then cover them in parallel. Keep working while subagents run.
- Use **Agent** for a single independent workstream. Use **Workflow** when structure is known up front and full coverage matters — fan out → verify each result → synthesize. Bias toward thoroughness on research/review/audit; direct pass on small scoped changes.

## Checkpoints

- Pause only on a genuine blocker: destructive/irreversible action, real scope change, or input only the user can provide. Otherwise finish the work.
- Before ending a turn: if your last paragraph is a plan, question, or promise of undone work, do it now. Don't stop to suggest a new session while context remains.

## Subagents and verification

- Classify the mission first; **commit to the tier before the first token** — in an agentic loop the model won't surface failure, it will produce a plausible result and continue. Use `opus` when: genuinely ambiguous, multi-domain, or dangerous-if-wrong. Otherwise `sonnet`. Pass `model` to every subagent — omitting it silently runs N copies of the main-loop model.
  - `haiku` — **Quick/Routine**: file lookup, grep, diff, summary, format/lint, JSON reshaping, docs edits. 200K context cap — not for large codebases or long sessions.
  - `sonnet` — **Standard/Deep**: implementation, bug fix, tests, code review, research, architecture, security review. Right default for ~90% of tasks.
  - `opus` — **Hardest Problems**: genuinely ambiguous, multi-domain synthesis, irreversible high-stakes decisions where wrong is dangerous.
  - `fable` — **Novel/Unsolved**: long-chain novel reasoning. Only when `opus` is demonstrably insufficient.
  - `opusplan` — Opus plans, Sonnet executes. When plan quality matters more than Opus plan-phase cost.
- If the current model fits the mission, stay quiet — don't suggest switching unless the mismatch is clear.
- Prefer fresh-context verifier subagents over self-review. Run deterministic checks (tests, lint) first; use verifiers for semantic review.
- Brief every verifier with established ground truth (settled decisions, verified facts) — without it, they re-litigate instead of catching real errors.
- Subagent briefs: goal, done-criteria, constraints, non-goals, verification method.

## Workflows

- **`pipeline()` by default, `parallel()` only for genuine barriers.** `pipeline` streams each item through all stages independently. Use `parallel` only when stage N needs ALL of stage N-1's results (dedup, early-exit, cross-item synthesis). A filter between pipelines is correct; a barrier just to flatten is not.
- **Always set `model` on per-item agents.** Set `model: 'sonnet'` (or `'haiku'`) on every per-item stage; omit only on judge/synthesis.
- **`schema` returns `null` on failure — always `.filter(Boolean)` before any `.map`/`.reduce`/property access.**
- **`phase()` and `agent({phase:})` titles must match `meta.phases[].title` exactly** — a typo silently orphans agents. `meta` must be a pure object literal (no variables, template literals, or function calls).
- **Loop-until-dry:** `while (dry < 2) { if (!fresh.length) { dry++; continue } dry = 0; ... }` Track seen items with a `Set`.
- **Budget-aware scaling:** guard loops with `while (budget.total && budget.remaining() > 50_000)`.

## Facts and research

- Don't assert facts about post-cutoff products, APIs, or prices from memory — verify against live sources or project code.
- Prefer the web-researcher MCP for open-web research; always provide sources and verify trust.

## Documentation

- Docs must reflect current code — validate claims against real code/tests before writing; update docs in the same change that alters behavior.
- Anchor to stable things (interfaces, paths, signatures); link rather than duplicate. Omit what churns: line counts, version numbers, large diagrams.
- Tool/command descriptions must match runtime behavior exactly: mark read/write and idempotency.

## Memory

- One lesson per file with a one-line summary. Capture corrections and confirmed approaches with why. Don't store what the repo already records; update rather than duplicate; delete stale notes on correction.
- Always-true rules belong in CLAUDE.md (loaded in full), not in the memory index (~200 lines loaded).
- Never put secrets or credentials in memory files or web-research queries.
- If the user hands you a secret, redact it when done (session transcript, history.jsonl, any file it landed in). If it may have reached a committed/synced surface, tell the user to rotate it.

## Fable 5 (shorthand: `fable`)

- Classifiers (cyber, bio/chem, reasoning extraction) are conservative — a trip returns `refusal` with no fallback. Route security/bio subagents to `opus`; suggest relaunching on Opus 4.8 if such a task arrives mid-session.
- For reasoning visibility, read structured `thinking` blocks (`display: "summarized"`).
