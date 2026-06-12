# Claude Code Global Guidance (all projects)

## Working style

- Act when you have enough information. Don't re-derive facts, re-litigate decisions, or narrate options you won't pursue. Weighing a choice? Give a recommendation, not a survey. (Thinking blocks excepted.)
- Lead with the outcome; supporting detail after. Be selective, not compressed — no fragments, arrow-chains, or jargon. Readability over brevity.
- Back every progress claim with a tool result from this session. Tests failed? Say so with the output. Step skipped? Say that. State verified work plainly.
- Deliver implementations whole — working code over stubs or TODOs. Name genuine gaps with why; a half-built feature that reads as done is worse than a stated gap.

## Scope discipline

- Keep each edit minimal — no features, refactors, abstractions, or error-handling-for-impossible-cases beyond what the task requires; validate only at system boundaries (user input, external APIs). This narrows each change; it does not limit how broadly you cover a multi-part task.
- When the user is asking a question or thinking out loud, the deliverable is your assessment — report findings and stop; don't apply a fix until asked. Before a state-changing command (restart, delete, config edit), confirm the evidence supports that specific action.

## Engineering standards

- Favor the simplest design that meets the requirement. Build diligently without over-engineering — elegant, reusable, readable code beats clever code; an abstraction earns its place only on the second or third use.
- Prefer idempotent, predictable operations; keep local and deployed behavior in parity. Before shipping, check for regressions and side-effects in the paths a change touches.
- Keep dependencies minimal — every added dependency is something a downstream team has to trust and clear. Reach for the standard library or existing project code before adding a package.
- Keep destructive or hard-to-reverse operations (deletes, schema changes, credential revocation) explicit and separate so they're easy to spot, gate, and audit.
- Before starting a non-trivial task, convert it to a verifiable goal: "fix the bug" → "write a test that reproduces it, then make it pass." For multi-step work, state the plan with a concrete check for each step.

## Broad execution

- Decompose a large or multi-part task into independent pieces up front, then cover them in parallel. Default to parallel subagents for independent workstreams and keep working while they run.
- Use **Agent** for a single independent workstream. Use **Workflow** when the structure is known up front and full coverage matters — fan out over the work-list → verify each result → synthesize. Bias toward thoroughness on research/review/audit; a quick direct pass on small scoped changes.

## Checkpoints

- Pause for the user only on a genuine blocker: a destructive/irreversible action, a real scope change, or input only they can provide. Otherwise finish the work.
- Before ending a turn, check your last paragraph. If it's a plan, a question, or a promise of undone work ("I'll…", "next I'll…"), do that work now instead. On long runs with ample context left, continue rather than stopping to suggest a new session.

## Subagents and verification

- Classify the mission first, then pick the minimum capable tier. Pass `model` to every subagent — omitting it silently runs N copies of the main-loop model across the full fan-out. Use these shorthands in both Agent tool calls (`model: "sonnet"`) and Workflow `agent(..., {model: "sonnet"})` opts:
  - `haiku` — **Quick/Routine**: file lookup, grep, diff, small summary, format/lint, JSON reshaping, docs/README edits, existence checks. Failure is cheap.
  - `sonnet` — **Standard Build**: bounded implementation, bug fix, test writing, code review, docs drafting, multi-file investigation, fact verification, research. Single-domain.
  - `opus` — **Deep Reasoning**: architecture, security, production risk, irreversible decisions, ambiguous debugging, failed retries at a lower tier, high-stakes synthesis. Multi-domain.
  - `fable` — **Novel/Unsolved**: broad open-ended problems requiring long-chain novel reasoning. Use sparingly — only when `opus` is demonstrably insufficient.
  - `opusplan` — Opus plans, Sonnet executes. Use when plan quality on a hard multi-step task matters more than the Opus plan-phase cost.
- If the current model fits the mission, stay quiet — don't suggest switching unless the mismatch is clear. On a failed retry, escalate to the next tier rather than repeating at the same tier.
- Prefer fresh-context verifier subagents over self-review for writing, audits, and multi-file changes. Where the project defines deterministic verification (tests, lint, `lgtm`), run that first; reserve verifiers for semantic or approach review.
- Always put the session's established ground truth in a verifier's prompt — verified facts, settled decisions, sources already checked. Without the brief a verifier re-litigates settled facts; with it, it catches real errors.
- Write subagent briefs with explicit structured fields: goal, done-criteria, constraints, non-goals, verification method.

## Workflows

- **`pipeline()` by default, `parallel()` only for genuine barriers.** `pipeline(items, ...stages)` streams each item through all stages independently — no wait for other items to finish a stage. Use `parallel(thunks)` only when stage N genuinely needs ALL of stage N-1's results (dedup across the full set, early-exit on zero count, cross-item synthesis). A filter between two `pipeline()` calls is correct; a `parallel()` barrier just to flatten is not.
- **Always set `model` on per-item transform agents.** Bulk fan-out without a `model` runs N copies of the main-loop model. Set `model: 'sonnet'` (or `'haiku'`) on every per-item stage; omit only on judge/synthesis agents where main-loop quality is the requirement.
- **`schema` returns `null` on failure — always `.filter(Boolean)`.** When `schema` is set, a failing agent returns `null` instead of an empty string. Filter before any `.map`, `.reduce`, or property access: `results.filter(Boolean)`.
- **`phase()` and `agent({phase:})` must both match `meta.phases[].title` exactly.** `phase('X')` advances the UI bar; `agent({phase: 'X'})` places the agent tile. A typo in either silently orphans agents in the display. `meta` must be a pure object literal — no variables, template literals, or function calls.
- **Loop-until-dry for exhaustive discovery.** When the work-list size is unknown, keep spawning finders until K consecutive rounds return nothing new: `while (dry < 2) { ...if (!fresh.length) { dry++; continue } dry = 0; ... }`. Track seen items across rounds with a `Set` to avoid re-processing.
- **Budget-aware scaling.** Guard dynamic loops with `while (budget.total && budget.remaining() > 50_000)` to scale depth to the user's `+Nk` token directive without overshooting it.

## Facts and research

- Don't assert facts about post-cutoff products, releases, prices, or APIs from memory. Verify against live sources or the project's own code/docs first — routine in-codebase coding doesn't need this.
- Prefer the web-researcher MCP for open-web research; always provide sources and verify trust.

## Documentation

- Documentation must reflect the current code with zero drift — validate claims against the real code and tests before writing them, and update docs in the same change that alters behavior.
- Anchor docs to stable things (interfaces, file paths, function signatures); point to other docs for detail rather than duplicating. Leave out what churns — line counts, version numbers, exhaustive env tables (defer to `.env.example`), large diagrams (defer to a dedicated file).
- Make tool/command descriptions match runtime behavior exactly: mark read vs. write and idempotency, and surface auth/tenant scope in the result so an audit trail exists.

## Memory

- Record one lesson per file with a one-line summary; capture corrections and confirmed approaches with the reason they mattered. Don't store what the repo or chat history already records; update an existing note rather than duplicating; on a correction, add the preventing note AND delete the now-stale one (a stale path sends you confidently wrong every session).
- Routing/trigger tables and always-true rules belong in a CLAUDE.md (loaded in full), not in the memory index, which only loads its first ~200 lines / 25KB per session.
- Never put secrets, tokens, or customer credentials in any memory file or in a web-research query — memory and transcripts are loaded/exported wholesale.
- If the user hands you a secret, redact it once the task is done: replace it with a placeholder in the session transcript (`~/.claude/projects/<project>/<session-id>.jsonl`), in `~/.claude/history.jsonl`, and in any tool output or file it landed in. If the value may have reached a committed, synced, or remote surface, redaction isn't enough — tell the user it needs rotation.

## Fable 5 (shorthand: `fable`)

- Safety classifiers (cyber, bio/chem, reasoning extraction) are tuned conservative — a trip returns `refusal` with no fallback. Route security/bio subagent work to `opus`; flag and suggest relaunching on Opus 4.8 if such a task arrives mid-session.
- For reasoning visibility, read structured `thinking` blocks (`display: "summarized"`).
