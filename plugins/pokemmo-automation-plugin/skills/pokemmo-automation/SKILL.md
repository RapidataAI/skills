---
name: pokemmo-automation
description: Use when working on local PokeMMO automation in this workspace, especially gym runners, farming bots, battle-log parsing, movement verification, shiny safety, and DeepScanner/GameMonitor integrations.
---

# PokeMMO Automation

Use this skill for the local PokeMMO scripts in `/Users/michael/Documents/github/pokemmo`.

## Runtime

- Prefer `/opt/homebrew/opt/python@3.12/bin/python3.12` for live runs so Quartz is available.
- Before launching a runner, verify no old `mega_gym_run.py`, `sail_from_kanto.py`, or `farming_bot.py` process is still active.
- When a user says `stop`, stop the automation process only. Do not assume that means dismissing an in-game battle.

## Battle Safety

- Do not trust one source for shiny detection.
- For farm scripts, scan the current battle's GameMonitor battle-log slice for explicit encounter lines like `The wild Shiny ...` or `A wild Shiny ...`.
- Do not trigger on generic chat lines containing `shiny`.
- If a shiny-like encounter is suspected, fail closed: stop inputs immediately and leave the game untouched.

## Gym Battle Handling

- Post-victory handling should prefer `X` to dismiss UI; avoid blind `A` mashing after leader fights.
- Confirm battle completion with an overworld movement test when possible instead of trusting only reward text.
- Treat battle logs as authoritative for victory markers such as `Player defeated ...` and reward lines, but verify control before moving on.

## Movement

- Log expected start coordinates before long movement segments.
- Verify segment endpoints after each run; only use `walk_to` as recovery after a real miss on the same map.
- Be careful with map transitions like Pokecenter exits where coordinates can jump across spaces; a large coord jump can be a valid transition, not a miss.
- Cut/fly verification steps consume movement. Adjust the following route segment so the check tile is not double-counted.

## Farming Bots

- Party level tracking should sync from DeepScanner party summaries at startup and after battles.
- Identical species in the same party need slot labels such as `Blastoise #1` and `Blastoise #2`; do not collapse them into one key.
- EXP lines can arrive during battle wind-down. Count EXP there too, not only in the main battle loop.
- World chat can contaminate GameMonitor battle logs. Filter aggressively when parsing encounter-critical events.
