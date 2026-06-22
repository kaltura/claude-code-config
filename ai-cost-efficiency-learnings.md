# What I Learned Cutting My AI Bill by 86%

**Real data from Claude on Amazon Bedrock · June 8–15, 2026**

---

I reduced my daily AI spend from $1,119 to $160 — without changing the type or volume of work. This doc walks through what I found, why it happened, and the specific practices I now follow. I'm sharing it because the patterns apply to any team running Claude at scale.

---

## The numbers

| | Before (Jun 8–11) | After (Jun 12–15) | Change |
|---|---|---|---|
| Daily average cost | $1,119 | $160 | **−86%** |
| 4-day total | $4,476 | $640 | **−86%** |
| Input tokens | 7.7 MTok | 4.6 MTok | −40% |
| Output tokens | 13.7 MTok | 8.5 MTok | −38% |
| **Cache read tokens** | **5,579 MTok** | **725 MTok** | **−87%** |
| Cache write tokens | 229 MTok | 51 MTok | −78% |

Same projects. Same type of work. The change was in configuration, not workload.

**Quality held.** The work across both periods was the same in type and complexity — releases shipped, docs written and reviewed, code reviewed, tools executed and tested, research was performed. There was no noticeable degradation in output quality across all tasks. Opus was reached for in only a handful of cases where the task genuinely warranted it: a multi-domain architecture audit and refactor of a full repo that required long-horizon reasoning. Everything else ran on Sonnet or Haiku without issue. The 86% cost reduction came from eliminating Opus on work Sonnet could handle — not from accepting lower quality.

---

## What actually drives the bill

Before looking at the data, I assumed input and output tokens were where the money went. They aren't.

**Cache reads are 90–96% of total token volume in agentic sessions.** Every agent turn re-reads the full cached context — system instructions, project guidance, conversation history. A deep workflow with 50 agent turns re-reads that context 50 times. At frontier-model rates, it compounds fast.

| Token type | Before | After |
|---|---|---|
| Cache reads | 95.7% of volume | 91.9% of volume |
| Cache writes | 3.9% | 6.5% |
| Fresh input | 0.1% | 0.6% |
| Output | 0.2% | 1.1% |

The practical consequence: optimizing input/output tokens is fine-tuning. Optimizing cache read rate and volume is where the real leverage is.

---

## The rate card (derived from actual billing data)

| Model | Input | Cache read | Cache write (5m) | Output |
|---|---|---|---|---|
| Opus 4.8 · global | $5.00 | $0.50 | $6.25 | $25.00 |
| Sonnet 4.6 · global | $3.00 | $0.30 | $3.75 | $15.00 |
| Haiku 4.5 · global | $1.00 | $0.10 | $1.25 | $5.00 |
| Sonnet 4.5 · regional | $3.30 | $0.33 | $4.13 | $16.50 |

*All prices in USD per million tokens (MTok).*

**The most important number is cache read rate.** Opus charges $0.50/MTok. Sonnet charges $0.30/MTok. On 5,000+ MTok/day of cache reads, that 1.67× difference was a $3,800 gap over four days — from one rate difference on one token type.

**Global inference profiles are 10% cheaper than regional.** On Bedrock, bare model aliases (`sonnet`, `opus`) resolve to regional endpoints — 10% more expensive and sometimes pinned to older versions. Always use explicit global profile IDs: `global.anthropic.claude-sonnet-4-6`, `global.anthropic.claude-opus-4-8`, `global.anthropic.claude-haiku-4-5-20251001-v1:0`. The Sonnet 4.5 rows in my data confirmed this: they ran on regional endpoints and show exactly the 10% premium ($3.30 vs $3.00).

---

## Practice 1 — Route each task to the right model tier

Before the change, Opus 4.8 was running everything: main loop, all subagents, every turn. On some days its cost share was 99.8% of the total bill.

After the change, the same work distributed across the tier it deserved:

| Tier | Model | Context window | Use for |
|---|---|---|---|
| Quick / Routine | **Haiku 4.5** | 200K | File lookup, grep, diff, format/lint, JSON reshaping, docs edits, existence checks — **not suitable for large codebases or long sessions** |
| Standard Build | **Sonnet 4.6** | 1M | Bounded implementation, bug fix, test writing, code review, multi-file investigation, research |
| Deep Reasoning | **Sonnet 4.6** | 1M | Architecture decisions, security review, production risk, ambiguous debugging — Sonnet handles most of this well |
| Hardest Problems | **Opus 4.8** | 1M | Multi-domain synthesis, genuinely ambiguous problems, irreversible high-stakes decisions |

Haiku's 200K cap is a hard constraint, not a soft one — if a task requires deep context (large codebase, long conversation history, big documents), Sonnet is the floor regardless of cost. Sonnet 4.6 is the cheapest model with a 1M context window.

**Classify upfront — don't plan to escalate.** In a long-running agentic loop, the model will not tell you it failed. It will do its best, produce a plausible-looking result, and continue. By the time you notice something went wrong, you've paid for the full Sonnet run, paid again for the Opus retry, and potentially accepted bad intermediate output. The right approach is to characterize the task before the first token and commit to the appropriate model. The signals that point to Opus: the problem is genuinely ambiguous (no clear success criteria), it spans multiple domains, or a plausible-but-wrong answer is dangerous. If those signals aren't present, Sonnet is the right call — not as a trial run, but as the final answer.

**In any agentic framework or workflow, subagents inherit the parent model by default.** On an Opus session, every grep and file read runs at Opus prices unless you explicitly override. Set the model on every spawned agent — `model: 'haiku'` for mechanical tasks, `model: 'sonnet'` for everything else, `model: 'opus'` only when the upfront task assessment says it's warranted.

**Model diversity reduces cost, not just risk.** Counterintuitively, using more models (Haiku + Sonnet + Opus) costs less than using one frontier model for everything. The rate difference dominates: Haiku cache reads at $0.10/MTok vs Opus at $0.50/MTok is a 5× difference. At scale, that wins over cache fragmentation every time.

**Resulting model cost share post-config:**
Sonnet 4.6: 50–90% of daily spend · Opus 4.8: 6–50% · Haiku 4.5: ~6%

---

## Practice 2 — Keep context lean

Cache is the dominant cost driver on long running sessions and agentic loops (common instructions, behavior guidelines, project scopes, session history, etc.), and cache cost scales with context size. Every token in the context that doesn't contribute to the current task is a token paid for on every subsequent turn, for every agent in the session.

**Lean system prompts and project guidance.** A 10K-token system prompt on a 50-agent workflow costs 500K tokens in cache reads just to deliver instructions. I review guidance files regularly and cut anything that isn't load-bearing.

**`/compact` before agent or workflow fan-outs.** Each spawned agent cold-starts its own cache namespace from whatever context it inherits. In a long session, that means every agent in a fleet re-reads the full accumulated context — multiplied across the whole fleet. On June 16, a development session that had grown large spawned 16 agents and ran 3 workflows without compacting first; Sonnet cache reads spiked **21× in a single day** ($0.73 → $15.57), entirely from context multiplication rather than more work. Running `/compact` before launching a multi-agent or Workflow fan-out shrinks the context each agent inherits and cuts that multiplier at the source.

**`/compact` mid-session.** Compacts and summarizes conversation history into a shorter representation. It writes a new, smaller cache entry and reduces every subsequent turn's cache read cost. Use it when the early history is no longer load-bearing — long debugging sessions, multi-step implementations where the exploration phase is resolved.

**`/clear` between tasks.** Resets context entirely. Carrying forward the history of an unrelated session is pure cost with no benefit. Use it whenever a new task is genuinely independent of what came before.

**The config that enforces this automatically.** My `managed-settings.json` includes two keys that make compaction proactive rather than reactive:

```json
"CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "60",
"CLAUDE_CODE_AUTO_COMPACT_WINDOW": "300000"
```

`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=60` triggers auto-compaction at 60% context fill rather than waiting for the hard limit (~99%). This means the model compacts while there's still room to do it cleanly, instead of being forced into emergency compaction at the edge.

`CLAUDE_CODE_AUTO_COMPACT_WINDOW=300000` caps the compaction window at 300K tokens. This limits how large a compacted context can be, preventing runaway context growth across long agentic sessions.

> **Going further: hooks.** For teams running very long agentic sessions where losing mid-task context is costly, Claude Code's `PreCompact` and `PostCompact` hook events let you save state before compaction and inject a recovery brief after. The config keys above cover the common case; hooks are an optional layer for more demanding workflows.

**`CLAUDE_CODE_ATTRIBUTION_HEADER=0` — eliminate attribution-induced cache misses.** Claude Code normally appends an attribution string ("🤖 Generated with Claude Code") to API calls. A bug causes this text to be injected at a byte offset that varies per-call depending on request context. Since KV/prompt caching is a strict byte-level prefix match, any byte difference at position N invalidates all cache breakpoints from N onward. In a 10-agent parallel workflow, this means 10 cache misses instead of 1 write + 9 reads — even when every agent's logical system prompt is identical. Setting `CLAUDE_CODE_ATTRIBUTION_HEADER=0` disables the injection entirely, stabilizing system prompt bytes and restoring expected cache hit rates. Add it to `managed-settings.json` alongside the compaction keys.

---

## Practice 3 — Set defaults at the org level

The June 10 spike ($1,566 in a single day, vs $160/day after the fix) came from sessions defaulting to Opus and frontier models across hundreds of agent turns. One misconfigured workflow can cost more than a week of optimized usage.

Individual sessions shouldn't be able to silently default to frontier models. Enforce defaults in `managed-settings.json` (the org-level config that takes precedence over user settings):

```json
{
  "model": "sonnet",
  "availableModels": ["sonnet", "opus", "haiku", "opusplan"],
  "env": {
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "global.anthropic.claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_OPUS_MODEL":   "global.anthropic.claude-opus-4-8",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL":  "global.anthropic.claude-haiku-4-5-20251001-v1:0",
    "CLAUDE_CODE_ATTRIBUTION_HEADER": "0",
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "60",
    "CLAUDE_CODE_AUTO_COMPACT_WINDOW": "300000"
  }
}
```

This makes Sonnet the default, pins exact global Bedrock profile IDs, and enforces early compaction. Escalating to Opus is deliberate (one `/model` keystroke) — forgetting to escalate leaves you on the cheap path, which is the right failure mode.

---

## Cache mechanics: the hidden amplifier

Prompt caching is one of the highest-ROI optimizations available. My data:

| Model | Cache writes | Cache reads | Read/write ratio | Write cost | Net saving |
|---|---|---|---|---|---|
| Opus 4.8 | 204 MTok | 5,402 MTok | **26.4×** | $1,277 | **$77,050** |
| Sonnet 4.6 | 34 MTok | 494 MTok | 14.7× | $126 | $1,207 |
| Haiku 4.5 | 22 MTok | 314 MTok | 14.5× | $27 | $256 |

A cache write pays for itself after 2 reads. At 14–26× ratios, every $1 spent writing Opus cache saved ~$60 in avoided re-sends.

**Cache is per-model — diversity fragments it.** Sonnet cannot read an Opus cache entry and vice versa. Before the config change, one Opus cache served every agent and turn: maximum reuse, single namespace. After the change, context is written into Sonnet, Opus, and Haiku caches independently — each starts cold. This is why cache read volume dropped 7.7× even though actual work (input/output) dropped only ~40%. The rate savings more than compensate, but fragmentation is real and worth planning for.

**Consistency within a session maximizes reuse.** A workflow that uses Sonnet for 90% of its agents keeps Sonnet's cache hot. Mixing in Opus for 10% is correct on quality grounds, but those agents cold-start their own namespace. Plan for per-model warm-up cost in deep workflows.

**There are two cache write durations.** Standard cache writes last 5 minutes ($6.25/MTok on Opus, $3.75 on Sonnet). There is also a 1-hour cache write tier ($10.00/MTok on Opus, $6.00 on Sonnet) — more expensive to write, but keeps the cache alive between sessions. For teams running multiple sessions daily on the same codebase, 1-hour cache writes can reduce cold-start costs between sessions. Worth benchmarking if you run many short sessions that share a large common context.

---

## The efficiency metric worth tracking

The most useful single number for measuring AI session efficiency is **cost per MTok of output** — it captures all upstream costs (cache reads, writes, input) that produce each unit of meaningful generation.

| | Before | After |
|---|---|---|
| Cost per MTok output | $326 | **$75** |
| Output / input ratio | 1.78× | 1.86× |

The output/input ratio held stable (~1.8×), confirming the workload type didn't change. When this ratio drops sharply, it usually signals sessions spending more turns re-reading context than generating useful output — a sign of over-large system prompts or sessions that should have been compacted earlier.

---

## What about batch pricing?

Anthropic's Batch API cuts prices in half (Sonnet 4.6: $1.50/MTok input, $7.50/MTok output). The obvious question is whether Claude Code's overnight autonomous sessions can use it.

**The short answer is no — not with Claude Code today.** The Batch API is architecturally incompatible with agentic sessions. It requires all requests to be independent and fully composed at submission time; it does not support streaming; and it does not support multi-turn tool-call loops where each step depends on the prior response. Claude Code's entire execution model is a sequential agentic loop where turn N cannot be composed until turn N−1 returns.

For genuinely overnight, non-interactive work that can be decomposed into independent, self-contained prompts — a batch of code reviews, independent file analyses, parallel summarizations — it is possible to build a custom pipeline on top of the Batch API directly. But that is a custom engineering effort, not a Claude Code setting.

The better lever for overnight autonomous tasks is model tiering: a well-configured Claude Code session running on Sonnet with Haiku sub-agents already captures most of the savings the Batch API would offer, without the 24-hour latency or the architectural constraints.

---

## Summary of what changed

| Practice | Before | After |
|---|---|---|
| Default model | Opus 4.8 (everything) | Sonnet 4.6 (default), tiered by task |
| Subagent model | Inherited from parent (Opus) | Explicit: Haiku/Sonnet/Opus by task type |
| Bedrock profile | Bare aliases → regional | Explicit global IDs |
| Auto-compaction | Near hard limit (~99%) | At 60% context fill |
| Compact window | Uncapped | Capped at 300K tokens |
| Daily cost | $1,119 | $160 |

> The 86% reduction came from two root causes: routing tasks to the right model tier (including Haiku for mechanical sub-tasks), and carrying less context into each turn through proactive compaction and lean system prompts. The mechanism is cache reads — they are 90–96% of token volume, and the rate difference between models ($0.50/MTok on Opus vs $0.10/MTok on Haiku) compounds at billions of tokens per day.

---

*Data: AWS Cost Explorer, Amazon Bedrock, Kaltura-DevOps-testing. Period: June 8–15, 2026. Global inference profiles, us-east-1.*
