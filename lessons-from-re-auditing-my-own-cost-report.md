# Four Lessons From Auditing Agentic AI Costs

**Real data from a production Claude/Bedrock deployment · June 15 – July 9, 2026**

---

Cutting an AI bill with model tiering and cache discipline is well understood by now. What's less understood is how to check, afterward, whether the savings actually held — and where they quietly leak back out. Below are four findings from a real cost audit of a Claude Code deployment on Amazon Bedrock, each backed by the actual numbers, and each a pattern worth checking for in any agentic deployment, not just this one.

---

## 1. Rising spend and falling cost can be the same story

Daily spend went *up* between the two halves of the audit period — from $179.66/day to $201.34/day, about +12%. Read on its own, that looks like the optimization work from a few weeks earlier stopped paying off. It's the opposite:

| | Period 1 (Jun 15–30, 16 days) | Period 2 (Jul 1–9, 9 days) | Change |
|---|---|---|---|
| Daily average cost | $179.66 | $201.34 | +12% |
| Tokens moved per day | 244.8 MTok | 386.2 MTok | +58% |
| Output tokens per day | 2.57 MTok | 5.49 MTok | **+113%** |
| Cost per MTok of output | $69.88 | $36.71 | **−47%** |

Spend rose 12%. Output more than doubled. The two numbers together mean cost per unit of real work fell 47% — the daily bill went up because more than twice the work got done, at a rate that got cheaper and 2X more efficient. A mid-period model upgrade (Sonnet 4.6 → Sonnet 5, priced about 33% cheaper on cache reads) is a big part of why the *rate* dropped; the higher bill is just more throughput riding on top of that lower rate.

---

## 2. The cheapest model on paper isn't always the cheapest model in practice

Haiku 4.5's rate card is about 3x cheaper than Sonnet 4.6's, token type for token type ($1.00 vs $3.00 per MTok input, $0.10 vs $0.30 per MTok cache read, and so on). Routing more work to it should have been close to free money. It wasn't: once actual usage was tallied up, Haiku's real cost per token came out almost the same as Sonnet's.

| Model | Price per token (rate card) | Actual spend over the period | Actual tokens moved | Real cost per token (blended) |
|---|---|---|---|---|
| Sonnet 4.6 | baseline | $2,486.28 | 3,561.8 MTok | $0.698/MTok |
| Haiku 4.5 | ~3x cheaper | $138.18 | 208.0 MTok | $0.665/MTok |

A 3x rate-card discount should have produced a blended rate around $0.23/MTok. Instead it landed at $0.665 — almost no discount survived contact with real usage.

**Why: caching rewards reuse, and Haiku's calls weren't getting reused.** Every time Claude is called with a big chunk of repeated context — a system prompt, project instructions, file contents — that context can be cached instead of resent in full. The first call pays a "cache write" fee to store it. Every later call that reuses the same cached context pays a much cheaper "cache read" fee instead of the full price. The catch: if a cached chunk of context only gets read back once or twice before being discarded, the expensive write fee never gets paid off — you end up paying almost as if there were no cache at all. Look at how many times each model's cache actually got reused per write, and the blended rate lines up with it directly:

| Model | Cache reused per write | Real cost per token |
|---|---|---|
| Sonnet 4.6 | 13.0 times | $0.698/MTok |
| Haiku 4.5 | **1.8 times** | $0.665/MTok |

Sonnet's cache paid for itself — reused 13 times per write, so the write fee was a rounding error spread across 13 cheap reads. Haiku's cache barely got reused at all, so it was paying the expensive write fee almost as often as the cheap read fee, which erased nearly all of its rate-card advantage.

**Where it was coming from: one automated test-grading tool**, `coding-harness/evals/runner/fixture-workflow.js`, line 446. On its busiest day, July 4, this one script fired **2,156 separate Haiku calls** — a brand-new call for every grading category, on every test case, on every run:

```js
// starts a fresh call per category, every single time —
// the reason nothing ever gets reused
return parallel(DIMENSIONS.map((dim) => () =>
  agent(..., { model: 'haiku', schema: GRADE_RESULT_SCHEMA })))
```

Each of those 2,156 calls re-sent almost identical instructions from scratch and paid the full cache-write fee for it, because each one was a brand-new call with nothing to reuse. That single script accounted for 93% of that day's entire Haiku bill, and its own cache-read-to-write ratio was 0.73:1 — worse than reading everything fresh every time.

**The fix isn't to drop Haiku — it's to stop starting from scratch every time.** Grading a test result against a rubric really is a small, cheap task Haiku should handle. The fix is combining all the grading categories for one test case into a *single* call that returns every category's grade at once, instead of firing a separate call per category. Applied to the actual tool and verified with a live fixture run: Haiku calls per test case dropped from **17 to 2**, with the same pass/fail results as before the change — no data lost in the larger, combined payload, and no change to which model does the work.

**Not every case of "Haiku isn't reusing its cache" needs a fix, though.** A different set of Haiku calls in the same account — 12 long, one-shot research tasks — showed the same low reuse-per-write on paper, but for a good reason: each individual call ran 22 to 92 turns and built up 800K to 5.6 million cache-read tokens *within itself*. There was no shared context between separate tasks to reuse in the first place, so there was nothing to fix.  
**The lesson: when a model's real cost doesn't match its rate card, check why the cache isn't being reused before assuming the model is the problem.** Sometimes it's one bad calling pattern (fixable, as above); sometimes it's just independent tasks that never had anything to share (not a problem at all).

---

## 3. Rank cost anomalies by cost per output, not raw dollars

Three days in the period stood out for spending the most money. Ranked by raw dollars, they look like three versions of the same problem. Ranked by cost per unit of finished output, they split into two very different groups:

| Day | What happened | Total cost | Output produced | Cost per MTok of output |
|---|---|---|---|---|
| Jun 25 | Multi-session editorial/document audit across ~6 independent sessions | $505.65 | 5.99 MTok | **$84.48 — worst day in the period** |
| Jul 4 | Scoped, well-tiered multi-agent build across 11 sessions | $360.72 | 11.93 MTok | **$30.24 — best day in the period** |
| Jul 9 | High-volume day, largest single-day spend in the period | $378.63 | 11.53 MTok | $32.85 — second-best day in the period |

The two highest-dollar days in the whole period, Jul 4 and Jul 9, are also two of the *best*-value days once output is accounted for — both moved a huge amount of work through the system, and the price per unit of that work was low. Jun 25 spent noticeably less in raw dollars than either of them, and was still the worst day by far on a per-output basis. Sorting by raw spend alone would flag the two best days in the dataset as the top problems, and let the actual worst day slip in behind them.  
**Divide cost by output before deciding a high-spend day needs investigating** — otherwise your busiest good days and your worst bad day are indistinguishable on a dashboard that only shows dollars.

---

## 4. A rule an agent can't act on gets bypassed — even when it's followed correctly

Weeks earlier, an uncompacted multi-agent fan-out had spiked cache costs 21x in a single day. The fix at the time was a written rule: compact the conversation before launching a large batch of agents. Jun 25 (see #3) looked, on first read of the billing data, like a repeat of exactly that failure.

It wasn't, once the actual session logs were checked. It was six separate, independent sessions, not one runaway batch, and automatic compaction (a setting that shrinks the conversation once it crosses 60% of the context window) was firing correctly the whole time. But one of those sessions had something more useful than a false alarm in it, two days earlier on Jun 23: the agent's own reasoning, saved verbatim in its transcript, in the moment right before it launched a multi-agent workflow:

> *"Before I finalize and run this, I should check if I need to call `/compact` first, since the instructions mention doing that before heavy fan-outs."*

The agent remembered the rule. It weighed it. Then it launched the workflow anyway — no compaction step in between. That workflow went on to run 20 agents, 77 tool calls, and moved roughly 5 million tokens. Checking the full list of tools available to that agent explained why it happened: `/compact` is a command a person types into the terminal. There was no tool the agent could call to trigger it itself, and it didn't fall back to just asking a person to run it either.

**A rule that tells an agent to do something it has no way to do doesn't fail occasionally — it's unenforceable by design.** An agent that reasons about the rule perfectly will still bypass it every time, because remembering a rule and having a way to act on it are two different things. The fix isn't a stronger-worded version of the same instruction. It's changing what the instruction asks for: instead of "run this command before a big fan-out," ask for "tell the user you recommend this, and wait for them to confirm before proceeding" — something the agent can actually do. Check every instruction you give an agent against its actual toolset. An instruction it structurally cannot carry out only looks like a safeguard.

---

## The checklist

- Divide cost by output, not by day — a rising bill and a falling one can represent the same underlying result, in either direction.
- If a cheap model's real (blended) rate doesn't match its rate card, check its cache read-to-write ratio before blaming the model — a bad calling pattern and a bad model choice look identical on a bill but need completely different fixes.
- Rank high-spend days by cost-per-output before investigating them — raw dollars flag your busiest good days as often as your worst bad one.
- Check every rule you write for an agent against what it can actually do. "Recommend this and wait for confirmation" holds up where "just do it" doesn't, if the agent has no tool to "just do it" with.

---

*Data: AWS Cost Explorer (Amazon Bedrock), account-level billing detail, cross-referenced against local Claude Code session and sub-agent transcripts. Period: June 15 – July 9, 2026.*
