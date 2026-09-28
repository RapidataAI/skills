# Rapidata SDK — Claude Code Plugin

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that teaches Claude how to use the [Rapidata Python SDK](https://docs.rapidata.ai) for human annotation tasks.

When installed, Claude can write working Rapidata code for classification, comparison, ranking, benchmarks, and more — without you needing to look up the API docs.

## Install

In Claude Code:

```
/plugin marketplace add RapidataAI/skills
/plugin install rapidata-sdk-plugin@rapidata-sdk-marketplace
```

Pull the latest version later with `/plugin marketplace update rapidata-sdk-marketplace`.

Not using Claude Code? `npx skills add RapidataAI/skills` installs the same skill for Cursor, Codex, Copilot, Gemini CLI and others. With the SDK installed, `python -m rapidata skill` prints the full guide directly, and `python -m rapidata skill --install --agent claude|cursor|codex|generic` writes it into your project.

## What it does

The plugin points Claude at the guide bundled with the SDK, which covers:

- **Classification** — label images or text with categories
- **Comparison** — show two options to humans, get a preference
- **Ranking** — order multiple items by human judgment
- **Custom Audiences** — train annotators on your specific task before they start
- **Flows** — lightweight continuous ranking without full job setup
- **Benchmarks (MRI)** — compare AI models on human-evaluated leaderboards
- **Audience Filtering** — target annotators by country, language, age, device, etc.

## How it works

The skill in this repo is a short pointer. It tells the agent to install or upgrade the `rapidata` package, run `python -m rapidata skill`, and read the full guide that ships with the SDK before it writes any code. The guide therefore always matches the SDK version that is actually installed.

The full guide (`SKILL.md` plus the `reference`, `examples` and `flows-for-preference-data` companions) lives in the SDK repo at [`src/rapidata/_skill/`](https://github.com/RapidataAI/rapidata-python-sdk/tree/main/src/rapidata/_skill). **Edit it there, not here.**

## Version

`plugin.json`'s version tracks the latest Rapidata SDK release. On every stable SDK release, `sync-sdk-version.yml` bumps it; the skill text itself does not change per release.

## Repo structure

```
.claude-plugin/
  marketplace.json          # marketplace metadata
.github/workflows/
  sync-sdk-version.yml      # bump plugin version on SDK releases
plugins/rapidata-sdk-plugin/
  .claude-plugin/
    plugin.json             # plugin name + version
  skills/rapidata/
    SKILL.md                # pointer to `python -m rapidata skill`
```
