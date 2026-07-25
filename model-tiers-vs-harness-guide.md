# Model Tiers vs. Harness 

You've got a task that's not going well. Before you reach for a bigger model, read this. Most of the time, the fix isn't a smarter model — it's a better-structured prompt.

This guide gives you a three-step process. Fix your harness first. Raise the effort level on the same model second. Escalate the model tier last, only when you have proof it's warranted.

---

## The core idea

There are three separate levers for making an AI task succeed, in the order you should reach for them:

1. **Harness** — how you structure the task: decomposition, context, verification loops, retrieval, tool access and the prompts.
2. **Effort** — how much the model deliberates before answering, on the same model you're already using. Claude calls this the `effort` parameter (`low` to `max`); OpenAI calls it `reasoning.effort` (`minimal` to `xhigh`). Same model, same price per token — just more or less thinking.
3. **Model tier** — how capable the model itself is: Haiku → Sonnet → Opus → Fable.

People default to jumping straight to lever 3 when lever 1 or lever 2 is the actual fix. That's expensive and it doesn't fix what's broken.

Anthropic's own model-selection docs put it plainly. *"Tuning effort is often a better lever than switching models."*

OpenAI's reasoning guidance makes the identical case for its own effort parameter. Google Cloud's routing guidance says the same about model size. Three vendors agree: fix the setup, then tune the deliberation, before you pay for a bigger model.

---

## Step 1: Fix your harness first

Before you touch the model dropdown, check these. Each one is a common failure that looks like "the model isn't smart enough" but is actually a setup problem.

**Did you decompose the task?**
Can you write a checklist of independently-verifiable sub-steps? If yes, you don't have one hard problem — you have several small ones stapled together. Break it up. A model that fails a 10-step task in one shot often nails each step on its own.

**Did the model have the right context?**
A wrong file path, missing docs, or no access to the right data isn't a reasoning failure. It's a retrieval failure. No model tier fixes a model that's missing information. Give it the right context instead.

**Is there a verification loop?**
Check for a deterministic check — tests, lint, a schema, a build. If one exists and you're not running it, that's your gap. Add the check instead of upgrading the model.

**Is the work wide, not deep?**
A sweep across files, a batch of PRs, N similar reviews — that's throughput, not reasoning. Fan the work across Sonnet or Haiku in parallel. Don't pay frontier-model rates on items that don't need them.

**Try again with a plan and a verifier?**
Rerun it with an explicit plan and a fresh-context pass that checks the first attempt. If that fixes it, the model wasn't the issue.

If you've done all five and the task still fails the same way, move to Step 2.

---

## Step 2: Raise the effort level — same model, same price

Before you change models, check whether you've told this one to think harder. Claude's `effort` parameter and OpenAI's `reasoning.effort` control how much a model deliberates before answering, independent of which model you picked. Same model, same per-token price, no new cache namespace to warm up. Sonnet 5 and Opus 5 default to `high` already, so check that first. If a task is running below default — a subagent set to `low`, a workflow tuned down for cost — raise it one notch. Re-run before you touch the model tier. Move to `high`, or `xhigh` for long tool-heavy agentic work. If that clears it, you're done at the rate you were already paying. If it doesn't, you've ruled out "didn't think long enough," and it's time to move to Step 3.

**One exception: check Opus 5 before pushing Sonnet 5 further.** If Sonnet 5 is already at `xhigh` and still falling short, don't assume the next lever is squeezing more effort out of it. Independent benchmarks (Artificial Analysis) show Opus 5 at `high` effort can beat both Sonnet 5 and Opus 4.8 on quality at a lower cost per task than Sonnet 5 does at its own ceiling. Try Opus 5 at `high` there before assuming you need `xhigh` on a bigger tier — sometimes the cheaper next step is a tier up, not more effort on the same one.

**Why "overthinking" is a real risk.** Anthropic's own guidance on Opus 4.7 warns that pushing effort to `max` can lead to overthinking. The quality gain is often small; the added cost is not. Overthinking is what happens when a model keeps deliberating past the point where more thought helps. It starts second-guessing a correct answer, inventing edge cases that aren't there, or padding a simple output with caveats nobody asked for. That's a side effect of forcing more reasoning tokens onto a task that didn't need them. Model tier doesn't carry the same risk. Tier raises the ceiling on what the model can reason about. It doesn't force the model to reason longer on every task the way `max` effort can.

**Sometimes the fix is context window, not tier or effort.** Some models ship in more than one context-window size — Sonnet 5's 1M-token version alongside its standard window, for example. A model that can't hold the whole codebase, conversation history, or document set in view at once isn't failing at reasoning. It's failing at retrieval. Reach for the larger context window before you reach for a bigger tier. It's usually cheaper. It's also the more precise fix when the real problem is information falling out of view, not the model being under-powered.

---

## Step 3: Escalate the model tier — with evidence

The right process depends on one thing: how often this task will run.

Don't build a benchmark for a single bug fix — that's slower than just doing the work. Build one for anything you'll run repeatedly. The setup cost pays for itself on the second run.

### One-off task: use the four signals as a judgment call

If this is a single task you're doing once, don't stop to benchmark it. Instead, check it against the four signals below. If one or more clearly applies, escalate now and move on. A wrong guess here just costs one extra task at a higher tier — it's not a repeated tax on every future run.

| Signal | What it looks like | Example | Why a bigger model helps |
|---|---|---|---|
| **Retries don't help** | You've re-run the task twice with a better plan and a fresh verifier, and it fails the same way both times. | Scheduled emails go out at the wrong time for some users. An agent blames a timezone-conversion bug on attempt one, then a server clock issue on attempt two. Both are wrong — the real bug is that the scheduler displays times in the recipient's timezone but fires the send job in the sender's, and it only shows up during daylight-saving transitions. Spotting it means holding both code paths in mind at once, not re-reading either one more carefully. | The fix isn't more context or a better plan — you already gave it both. What's missing is the reasoning depth to hold two explanations in mind at once and notice they contradict. That depth is what scales with model tier. |
| **The parts don't separate** | Splitting the task into pieces loses the thing that made it hard — the pieces only fail when combined. | A checkout total is wrong, but only for orders with a percentage discount, free shipping, and a tax-exempt state, all at once. Reviewed one at a time, the discount logic, shipping logic, and tax logic are each correct. The bug only exists in how the three combine and in what order they apply. | Decomposition — your Step 1 fix — destroys the signal here, since each piece checks out on its own. With no way to split the task without losing the bug, the only lever left is a model that can reason about all three parts together in one pass. |
| **Wrong is expensive and unchecked** | A mistake here costs real money or trust, no test/lint/schema would catch it, and the right answer depends on current best practices, not just in-the-moment logic. | You're deciding whether to upgrade password hashing by re-hashing on each user's next login, or all at once in one background job. Get it wrong and you either leave weak hashes sitting around for months, or lock out every user mid-migration if the background job has a bug. The right call depends on current security guidance and what's realistic for your migration setup — not something a model reasons its way to from first principles. | Tier alone isn't the fix here — no amount of extra reasoning depth substitutes for guidance the model was never trained on. The right combination is a strong model tier plus [Web Researcher MCP](https://zoharbabin.com/web-researcher-mcp/) for verified, cited-source research, followed by deep synthesis of your project's goals, structure, and the research findings. The tier is what makes that synthesis sound; the research is what keeps it from being a confident guess. |
| **Long task, low margin for error** | The task runs many steps end-to-end and needs to stay reliable across all of them, not just get one step right. | You're renaming a column across 40 million database rows, in batches, over several hours. If the job dies partway through, the data needs to stay in a state your app can still read — not half-renamed, with some code paths looking for the old column name and some for the new one. Getting the first 39 batches right and the 40th wrong isn't a small miss; it's a broken database. | A small per-step error rate compounds over dozens of steps into a near-certain failure somewhere in the chain. A model with a lower per-step error rate carries that reliability across the whole run, not just one step of it. |

None of these apply and you're still failing? Escalate one tier, try once, and see if it clears. That single try costs less than debating it further.

### Recurring task: build the benchmark once

Some tasks run many times. Think a prompt template in production, a recurring review pipeline, an agent handling a whole category of tickets. For those, the math flips: a one-time 20-minute test saves you from overpaying or under-delivering on every run after this one.

1. **Pull 5–10 real examples** of the task — actual prompts and data, not made-up cases.
2. **Run them on your current tier.** Apply the Step 1 fixes (decomposed, right context, verification loop in place) and raise effort to `high` or `xhigh`.
3. **Run the same examples on the next tier up**, at the same effort level.
4. **Compare against a pass/fail bar you set beforehand.** For example: tests pass, output matches the spec, a reviewer would approve it. If the bigger tier clears the bar consistently where the current tier doesn't, escalate the default for that task. If both tiers land in the same place, keep the cheaper one — you'd be paying more for the same answer, on every future run.

**One honest caveat:** the four signals are a fast filter for one-off calls, not a guarantee. Anthropic's own guidance is explicit: a real comparison beats a feeling. Lean on the benchmark whenever a task recurs enough to make one worth running.

---

## Which model, when

| Tier | Model | Use for |
|---|---|---|
| Quick / Routine | **Haiku 4.5** | File lookup, grep, diff, format/lint, JSON reshaping, docs edits. 200K context cap — not for large codebases or long sessions. |
| Standard / Deep | **Sonnet 5** | Implementation, bug fixes, tests, code review, research, architecture, security review. This is the right call for most of what you do. |
| Hardest Problems | **Opus 5** | Genuinely ambiguous, multi-domain synthesis, high-stakes decisions where a wrong answer is dangerous — after Steps 1 and 2 didn't fix it. |
| Novel / Unsolved | **Fable 5** | Opus 5 at `high` or `xhigh` effort has already failed on this exact task, with a run you can point to. Costs 2x Opus per token — reach for it on evidence, not a hunch. |

**The bar for Fable 5 got higher, not lower.** Opus 5 now beats Fable 5 on Artificial Analysis's blended Intelligence Index and on AA-Briefcase, and does it at roughly half Fable's per-token price — so tasks that used to need Fable's reasoning depth are increasingly won by Opus 5 at `xhigh` first. Fable's real remaining edge is narrower than "harder problems": it still leads on factual knowledge and hallucination rate (AA-Omniscience) and on frontier physics (CritPt), and Opus 5's hallucination rate on uncertain/factual questions runs meaningfully higher than Fable's. That's the actual trade when you're deciding between them — not raw capability, but whether the task rewards Fable's lower hallucination rate or punishes Opus 5's higher one.

The clearest real-world signal for Fable so far: Stripe used it to migrate a 50-million-line Ruby codebase in a single day. Their own team had estimated two months by hand. That's the shape of task where Fable earns its price. Not one hard function, but thousands of files that all need to change in a consistent way. The run is long enough that drift, not difficulty, is the real risk. A few other concrete patterns worth escalating for — and none of these have been benchmarked head-to-head against Opus 5 at `xhigh`, so treat them as Fable's documented niche, not proof Opus 5 can't also do them:

- **A codebase-wide migration or refactor** that has to stay internally consistent across hundreds or thousands of files in one continuous run. The Stripe case above.
- **A long autonomous agent session** — many hours, many tool calls — where the risk isn't any single step being wrong. It's losing track of the plan over the length of the run.
- **Dense, structured documents**, like financial filings or scientific figures, where Opus is misreading the structure, not just the content.
- **Multi-day or multi-session work that benefits from persistent memory.** Fable's gains compound when it writes notes to files and reads them back across runs. It doesn't start cold every session.
- **Factual accuracy matters more than raw reasoning depth** — research, citations, anything where a confident wrong answer is worse than a slower right one. This is where Fable's lower hallucination rate outweighs Opus 5's price and speed advantage.

If your task doesn't look like one of these, Opus 5 at `xhigh` effort is almost always the better spend — cheaper, and now the stronger model on most benchmarks besides factual reliability. **Mythos 5** is the same underlying model without Fable's cyber and bio safety classifiers. It's invitation-only through Anthropic's [Project Glasswing](https://anthropic.com/glasswing), for vetted cybersecurity defenders and biomedical researchers — not something you opt into for a regular coding task.

**Classify before the first token, not mid-run.** In an agentic loop, a model that's out of its depth won't tell you. It produces a plausible-looking answer and keeps going. By the time you notice, you've paid for the wrong-tier run *and* the retry. Decide the tier upfront, based on the signals above.

**Set the tier explicitly on every subagent you spawn.** Subagents inherit the parent model by default. If your main session is on Opus, every grep and file read a subagent does runs at Opus prices unless you say otherwise. Set `model: 'haiku'` for mechanical work, `model: 'sonnet'` for almost everything else, and reserve `opus`/`fable` for the agent that's actually doing the hard part.

**Fable 5 can refuse outright.** It runs safety classifiers that Opus doesn't. A request can come back with `stop_reason: "refusal"` instead of an answer. There's no automatic fallback to another model — you have to retry it yourself, on Opus or Sonnet. Expect this on anything security- or bio-adjacent, even benign work in those areas. Route that kind of task to Opus first, and only escalate to Fable if Opus itself falls short.

**Set effort per subagent too.** A mechanical Haiku subagent doing a file lookup doesn't need `high` effort — drop it to `low` for speed. Save `high` or `xhigh` for the agent doing the actual reasoning.

---

## The one-line version

**Decompose it. Give it the right context. Add a verification check. Try again.**

**Still failing? Raise the effort level on the same model before you switch models.**

**Still failing, one-off task? Check the four signals and make the call.**

**Still failing, recurring task? Run the 20-minute benchmark before you commit.**

---

*Sources: [Anthropic model selection docs](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model), [Anthropic effort parameter docs](https://platform.claude.com/docs/en/build-with-claude/effort), [OpenAI model guidance](https://developers.openai.com/api/docs/guides/latest-model), [OpenAI reasoning models guide](https://developers.openai.com/api/docs/guides/reasoning), [Google Cloud agentic architecture guidance](https://docs.cloud.google.com/architecture/choose-agentic-ai-architecture-components), [FrugalGPT (Stanford)](https://ar5iv.labs.arxiv.org/html/2305.05176), [METR time-horizon research](https://metr.org/time-horizons/). See also [`ai-cost-efficiency-learnings.md`](./ai-cost-efficiency-learnings.md) in this repo for the cost data behind the tiering recommendations.*
