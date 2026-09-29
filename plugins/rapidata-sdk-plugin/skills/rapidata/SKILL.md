---
name: rapidata
description: Explains how to use the Rapidata API to get real and fast human annotations for your data. Use when writing code that creates labeling tasks, compares models, collects human feedback, or integrates with the Rapidata Python SDK.
---

# Rapidata Python SDK

The full guide ships inside the `rapidata` package and always matches the installed version. Do not write Rapidata code from memory or by reading the installed source.

1. Install or upgrade the SDK:

   ```bash
   pip install -U "rapidata>=3.25.10"
   uv add "rapidata>=3.25.10"   # in a uv project; or: uv pip install -U "rapidata>=3.25.10"
   ```

2. Print the guide and read ALL of its output before writing any Rapidata code:

   ```bash
   python -m rapidata skill
   ```

   It names companion guides (`python -m rapidata skill reference`, `examples`, `flows-for-preference-data`). Print those when the task needs them.

3. Check authentication before the first `RapidataClient()`:

   ```bash
   python -m rapidata status   # exit 0: authenticated; exit 1: not logged in
   python -m rapidata login    # browser login; show the user the URL it prints (waits up to 5 minutes)
   ```
