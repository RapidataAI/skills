# Rapidata SDK skill

Points your coding agent at the guide that ships with the [Rapidata Python SDK](https://docs.rapidata.ai). The skill tells the agent to install the `rapidata` package and read `python -m rapidata skill` before writing any Rapidata code, so the guide always matches the installed SDK version.

## Install

Claude Code:

```
/plugin marketplace add RapidataAI/skills
/plugin install rapidata-sdk-plugin@rapidata-sdk-marketplace
```

Any other agent:

```bash
npx skills add RapidataAI/skills
```
