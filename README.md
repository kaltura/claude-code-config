# Claude Code Config

Opinionated Claude Code defaults for cost-aware, model-tiered development on Amazon Bedrock.

## What's here

| File | Deploy to | Purpose |
|------|-----------|---------|
| `CLAUDE.md` | `~/.claude/CLAUDE.md` | Global guidance loaded into every session: working style, scope discipline, subagent model tiers, memory rules |
| `settings.json` | `~/.claude/settings.json` | User settings: plugin marketplaces, teammate mode, permission prompt config |
| `managed-settings.opusplan.json` | `/Library/Application Support/ClaudeCode/managed-settings.json` | Model defaults, effort level, env vars, Bedrock model ID pins, deny rules |
| `git-hooks/commit-msg` | `~/.claude/git-hooks/commit-msg` + `git config --global core.hooksPath ~/.claude/git-hooks` | Strips the `noreply@anthropic.com` email from Claude's `Co-Authored-By` commit trailer, keeping just the model name |

## Why

**Default Claude Code sessions inherit the main-loop model for every subagent**, silently multiplying cost on frontier sessions. This config:

- Pins **Sonnet 4.6** as the everyday default (cheap, capable, adaptive thinking)
- Makes **Opus 5**, **Fable 5**, **Haiku 4.5**, and an **opusplan** hybrid available one `/model` keystroke away — escalating to frontier reasoning is deliberate, never accidental
- Tiers subagents by value in `CLAUDE.md` so mechanical tasks run on Haiku and only judge/synthesis work runs on Opus
- Pins exact Bedrock inference-profile IDs (on Bedrock, bare aliases like `opus` resolve to older model versions without these pins)

## Install

```bash
git clone https://github.com/kaltura/claude-code-config.git
cd claude-code-config

cp CLAUDE.md ~/.claude/CLAUDE.md
cp settings.json ~/.claude/settings.json
cp managed-settings.opusplan.json \
  "/Library/Application Support/ClaudeCode/managed-settings.json"

cp -r git-hooks ~/.claude/git-hooks
git config --global core.hooksPath ~/.claude/git-hooks
```

## Before you deploy

**Not on Bedrock?** Remove the `ANTHROPIC_DEFAULT_*_MODEL` env vars from `managed-settings.opusplan.json`. Bare aliases (`sonnet`, `opus`, etc.) will resolve against the Anthropic API directly.

**Telemetry:** `settings.json` includes OTLP env vars pointed at `http://127.0.0.1:10198`. Remove or update if you don't have a local collector. The `OTEL_EXPORTER_OTLP_HEADERS` field is where an auth token would go — never commit a value there.

**Plugins:** the `enabledPlugins` and `extraKnownMarketplaces` entries in `settings.json` are personal. Remove them or replace with your own. The ones registered here:

| Plugin | Repo | What it does |
|--------|------|--------------|
| `message-timestamps` | [zoharbabin/claude-code-message-timestamps](https://github.com/zoharbabin/claude-code-message-timestamps) | Adds a timestamp to every user prompt submission |
| `avatar-presentation-skill` | [zoharbabin/avatar-presentation-skill](https://github.com/zoharbabin/avatar-presentation-skill) | Skill for generating avatar-driven presentations |
| `kalt-ai-plugins` | [kaltura/kalt-ai-plugin-marketplace](https://github.com/kaltura/kalt-ai-plugin-marketplace) | Kaltura internal plugin marketplace |

**Attribution:** `settings.json` sets `attribution.commit` and `attribution.pr` to hide the `noreply@anthropic.com` email, keeping just `Co-Authored-By: Claude Code`. A known Claude Code bug ([anthropics/claude-code#45137](https://github.com/anthropics/claude-code/issues/45137)) can make the commit-trailer setting get ignored, so `git-hooks/commit-msg` is a backup that rewrites the trailer after the fact. `core.hooksPath` is a global git setting: only install it if you don't already rely on per-repo hooks, since it replaces `core.hooksPath` for every repo on the machine that doesn't set its own. There's no equivalent backup for PR descriptions, since `gh pr create` doesn't go through git hooks.

**`claudeMdExcludes`:** `settings.json` excludes `/opt/homebrew/CLAUDE.md` from auto-loading. This is a personal filesystem-layout fix, not a general recommendation: on Apple Silicon, `brew --prefix` is `/opt/homebrew`, and Homebrew commits its own contributor `CLAUDE.md`/`AGENTS.md` at that repo's root. Claude Code walks up the directory tree and loads every ancestor `CLAUDE.md` it finds, so any project nested under `/opt/homebrew` (e.g. `/opt/homebrew/var/www/...`) picks up Homebrew's Ruby/Sorbet/RuboCop guidance by accident. Excluding the one file also suppresses `AGENTS.md`, since it's only pulled in via an `@AGENTS.md` import inside that `CLAUDE.md` — excluding the importer means the import never fires. If your projects don't live under `/opt/homebrew`, remove this key entirely.

## Further reading

The data and reasoning behind the defaults in this repo:

- [`ai-cost-efficiency-learnings.md`](./ai-cost-efficiency-learnings.md) — how one deployment cut daily spend 86% through model tiering and cache discipline
- [`model-tiers-vs-harness-guide.md`](./model-tiers-vs-harness-guide.md) — when to fix your prompt/harness, when to raise effort, and when to actually escalate model tier
- [`lessons-from-re-auditing-my-own-cost-report.md`](./lessons-from-re-auditing-my-own-cost-report.md) — four lessons from auditing whether those savings held up over time, including a caching pitfall that quietly erased a cheaper model's rate-card advantage
