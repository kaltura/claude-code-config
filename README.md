# Claude Code Config

Opinionated Claude Code defaults for cost-aware, model-tiered development on Amazon Bedrock.

## What's here

| File | Deploy to | Purpose |
|------|-----------|---------|
| `CLAUDE.md` | `~/.claude/CLAUDE.md` | Global guidance loaded into every session: working style, scope discipline, subagent model tiers, memory rules |
| `settings.json` | `~/.claude/settings.json` | User settings: plugin marketplaces, teammate mode, permission prompt config |
| `managed-settings.opusplan.json` | `/Library/Application Support/ClaudeCode/managed-settings.json` | Model defaults, effort level, env vars, Bedrock model ID pins, deny rules |

## Why

**Default Claude Code sessions inherit the main-loop model for every subagent**, silently multiplying cost on frontier sessions. This config:

- Pins **Sonnet 4.6** as the everyday default (cheap, capable, adaptive thinking)
- Makes **Opus 4.8**, **Fable 5**, **Haiku 4.5**, and an **opusplan** hybrid available one `/model` keystroke away — escalating to frontier reasoning is deliberate, never accidental
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
```

## Before you deploy

**Not on Bedrock?** Remove the `ANTHROPIC_DEFAULT_*_MODEL` env vars from `managed-settings.opusplan.json`. Bare aliases (`sonnet`, `opus`, etc.) will resolve against the Anthropic API directly.

**Telemetry:** `settings.json` includes OTLP env vars pointed at `http://127.0.0.1:10198`. Remove or update if you don't have a local collector. The `OTEL_EXPORTER_OTLP_HEADERS` field is where an auth token would go — never commit a value there.

**Plugins:** the `enabledPlugins` and `extraKnownMarketplaces` entries in `settings.json` are personal. Remove them or replace with your own.
