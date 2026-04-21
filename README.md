# Rapidata SDK — Claude Code Plugin

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that teaches Claude how to use the [Rapidata Python SDK](https://docs.rapidata.ai) for human annotation tasks.

When installed, Claude can write working Rapidata code for classification, comparison, ranking, benchmarks, and more — without you needing to look up the API docs.

## Install

In Claude Code:

```
/install-plugin https://github.com/RapidataAI/skills
```

## What it does

The plugin provides Claude with knowledge of the Rapidata SDK, covering:

- **Classification** — label images or text with categories
- **Comparison** — show two options to humans, get a preference
- **Ranking** — order multiple items by human judgment
- **Custom Audiences** — train annotators on your specific task before they start
- **Flows** — lightweight continuous ranking without full job setup
- **Benchmarks (MRI)** — compare AI models on human-evaluated leaderboards
- **Audience Filtering** — target annotators by country, language, age, device, etc.

## How it works

The plugin is three markdown files that get loaded into Claude's context when relevant:

- `SKILL.md` — core concepts, task types, and common patterns
- `reference.md` — full API reference (parameters, filters, result formats, error handling)
- `examples.md` — runnable code examples for every task type

These live in `plugins/rapidata-sdk-plugin/skills/rapidata/`.

## Version

The plugin version tracks the Rapidata SDK version (currently 3.9.2). A GitHub Actions workflow automatically syncs the version when a new SDK release is published.

## Repo structure

```
.claude-plugin/
  marketplace.json          # marketplace metadata
.github/workflows/
  sync-sdk-version.yml      # auto-sync plugin version to SDK releases
plugins/rapidata-sdk-plugin/
  .claude-plugin/
    plugin.json             # plugin name + version
  skills/rapidata/
    SKILL.md                # main skill definition
    reference.md            # API reference
    examples.md             # code examples
```
