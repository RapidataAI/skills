---
name: rapidata
description: Explains how to use the Rapidata API to get real and fast human annotations for your data. Use when writing code that creates labeling tasks, compares models, collects human feedback, or integrates with the Rapidata Python SDK.
---

# Rapidata Python SDK

Rapidata connects you with distributed human labelers worldwide for fast, high-quality data annotation. The SDK lets you create labeling tasks, manage annotator audiences, and retrieve results programmatically.

## Before you start: check the skill is up to date

This skill is pinned to **Rapidata SDK v3.21.0**. Run this check **once at the start of a Rapidata task** (not on every call) to confirm the user's runtime matches the skill:

```bash
python -c "import rapidata; print(rapidata.__version__)" 2>/dev/null \
  || pip show rapidata 2>/dev/null | awk -F': ' '/^Version:/{print $2}'
```

Compare the output to the pinned version above:

- **Installed > pinned** — this skill is **outdated**. The SDK may have new features, renamed methods, or changed signatures that this skill does not document.
  1. First, try to update the plugin automatically. Claude Code installs plugins from this repo's `main` branch, so either:
     - Re-run the install command to pull the latest: `/install-plugin https://github.com/RapidataAI/skills`, **or**
     - Use the plugin manager: `/plugin` → `rapidata-sdk-plugin` → update.
  2. Tell the user clearly:
     > ⚠️ The Rapidata skill is pinned to v3.21.0 but v{installed} is installed — the skill docs may be out of date. I've suggested updating the plugin; if the update isn't available yet, I'll proceed with the documented API and flag any surprises.
  3. Proceed using the documented API. If you hit an unexpected error (missing attribute, changed signature), stop and tell the user the skill is likely the cause — don't guess at the new API.

- **Installed < pinned** — the user's runtime is older than this skill. Suggest `pip install -U rapidata` so the runtime matches.

- **Match, or rapidata not installed** — proceed normally. (If not installed, the installation section below is the first step anyway.)

## Installation & Authentication

```python
pip install -U rapidata

from rapidata import RapidataClient
client = RapidataClient()  # Credentials resolved: env vars → ~/.config/rapidata/credentials.json → browser login

# Or pass credentials directly
client = RapidataClient(client_id="...", client_secret="...")
# Credentials are saved to ~/.config/rapidata/credentials.json

# Environment variables (useful for headless/container deployments):
# RAPIDATA_CLIENT_ID, RAPIDATA_CLIENT_SECRET — authenticate without a browser
# RAPIDATA_ENVIRONMENT — override the API endpoint (default: rapidata.ai)
# RAPIDATA_TOKEN_FILE — read a shared access token from this file (see below)
# Empty values are treated as unset and fall through to the next resolution layer.
```

### Sharing a token across many workers (distributed training)

When Rapidata is queried from a large distributed job (e.g. a ranking flow hit from hundreds or thousands of GPU workers), don't let every worker authenticate on its own — all their tokens expire at the same instant and the simultaneous re-auth looks like a coordinated burst that gets rate-limited. Instead, authenticate **once** and share the token via a file:

```python
from rapidata import RapidataClient

# Coordinator (holds the client credentials): keep a shared token file fresh
coordinator = RapidataClient(leeway=300)  # renew the token 5 min before expiry
coordinator.maintain_token_file("/shared/rapidata_token.json").join()
# maintain_token_file() writes the file immediately, then keeps rewriting it
# atomically from a background thread (every 60s by default), creating the
# directory if needed. .join() blocks forever — drop it if the coordinator
# also does other work (e.g. rank 0 both trains and refreshes the token).

# Workers (never see the client secret): point the SDK at that file
client = RapidataClient(token_file="/shared/rapidata_token.json")
# or set RAPIDATA_TOKEN_FILE and construct with no arguments.
# The SDK reads the token at startup and re-reads the file whenever the
# in-memory token is within 60s of expiry (configurable via leeway).
```

The token file contains a bearer token — write it only to storage your job alone can access. To roll your own file writer, export the current token with `coordinator.get_token()` (cheap to call at any frequency; only contacts the auth server once the token is within `leeway` of expiry), write it atomically, and keep the absolute `expires_at` field so workers know when to re-read.

The file is just one transport. To move the token over any transport (key-value store, RPC, secret manager, message queue), pair `coordinator.get_token()` with `worker.set_token(fresh_token)`, which injects a fresh token into a running worker client (effective from its next request) without reconstructing it. This supports both a **push** system (the coordinator distributes a fresh token to every worker before the old one expires) and a **pull** system (each worker periodically fetches the current token from your own endpoint). Whatever the transport, pass the complete token object around and keep its absolute `expires_at` field. On older SDKs without `token_file`, pass the token dict directly with `RapidataClient(token=json.load(f))` — but the SDK never re-reads it, so each worker must call `set_token()` (or construct a new client) when the token expires.

## Core Concepts

- **Job Definition**: A reusable configuration template for a labeling task (task type, instruction, datapoints, answer options)
- **Audience**: A group of annotators selected and qualified for **one specific task** — via qualification examples and/or recruitment filters chosen to match that task. The whole point is to put the *right* people on *that* task. An audience trained for one task is **not** meant to be reused on a different, unrelated task: its qualification examples define what "good" means for the original task only, so reusing it elsewhere silently loses the quality it was built for. Reusing the same audience for repeated or scheduled runs of the **same** task is exactly right; for a different task, create a new audience. Three kinds:
  - **global** — the generic baseline pool for tasks that need no special qualification; instant, no setup. Use `client.audience.get_audience_by_id("global")`.
  - **curated** — pre-trained on a domain (e.g. alignment via `aud_MU1GZYoESyO`).
  - **custom** — trained with your own task-specific qualification examples (`client.audience.create_audience(...)` + `add_*_example(...)` + `start_recruiting()`). ⚠️ Recruiting is **explicit**: a custom audience recruits nobody until you add **≥3 qualification examples** *and then* call `audience.start_recruiting()`. Adding examples does **not** start recruiting on its own; assign a job before recruiting has started and it can never receive responses — `assign_job` logs a warning, and the waiting methods (`get_results()`, `display_progress_bar()`) raise instead of blocking forever. Use `"global"` when you don't need task-specific qualification.
- **Job**: A running instance of a job definition assigned to an audience
- **Flow**: Lightweight continuous ranking without full job setup
- **MRI/Benchmark**: Compare and rank AI models on leaderboards

**Client entry points:**
- `client.job` — create job definitions (classification, comparison, locate, draw, select words, free text, ranking)
- `client.audience` — create and find audiences
- `client.flow` — continuous ranking flows
- `client.mri` — model ranking insights / benchmarks
- `client.signals` — run a labeling job on a repeating schedule
- `client.context` — shorten over-long datapoint contexts against a specific question
- `client.billing` — read the current billing period's cost and remaining credit (organization-level)

## New API: Job Definitions + Audiences (Recommended)

The new API currently exposes **classification**, **comparison**, **locate**, **draw**, **select words**, **free text**, and **ranking** as job definitions. For continuous ranking without full job setup, the Flow API is still recommended.

### Step-by-step workflow

```python
from rapidata import RapidataClient

client = RapidataClient()

# 1. Get an audience. Default: the global pool — ready to go, zero setup.
audience = client.audience.get_audience_by_id("global")
# Curated domain pool (e.g. alignment): client.audience.get_audience_by_id("aud_MU1GZYoESyO")
# Only if you need task-specific qualification: client.audience.create_audience(name="My Evaluators")
#   — but recruiting is explicit: you MUST add >=3 qualification examples (add_*_example) AND
#   then call audience.start_recruiting() BEFORE assign_job. Adding examples does not start
#   recruiting; a job assigned before recruiting starts can never receive responses.
#   See "Custom Audiences".

# 2. Create a job definition
job_def = client.job.create_classification_job_definition(
    name="Animal Classification",
    instruction="What animal is in this image?",
    answer_options=["Cat", "Dog", "Bird"],
    datapoints=["https://example.com/img1.jpg", "img2.jpg"],
    responses_per_datapoint=10,
)

# 3. Assign to audience (starts labeling)
job = audience.assign_job(job_def)

# 4. Watch responses come in (opens browser on the running job)
job.view()

# 5. Monitor and get results
progress = job.get_progress()   # Non-blocking snapshot: state, completion_percentage, recruiting
job.display_progress_bar()      # Or block on a live progress bar
results = job.get_results()
df = results.to_pandas()
```

### Classification

Select one category from multiple options.

```python
from rapidata import NoShuffleSetting

job_def = client.job.create_classification_job_definition(
    name="Image Classification",
    instruction="What animal is in this image?",
    answer_options=["Cat", "Dog", "Bird", "Other"],
    datapoints=["img1.jpg", "img2.jpg"],
    data_type="media",              # "media" (default) or "text"
    responses_per_datapoint=10,
    contexts=["Optional text context per datapoint"],
    media_contexts=[["optional_reference.jpg"]],
    confidence_threshold=0.99,      # Optional: confidence-based early stopping
    # quorum_threshold=7,           # Alternative: quorum-based early stopping (cannot use both)
    settings=[NoShuffleSetting()],         # Keep answer order
    failure_tolerance=0.01,         # Optional: fraction of datapoints allowed to fail the upload
    private_metadata=[{"id": "abc"}],
)
```

`failure_tolerance` (available on every `create_*_job_definition`, defaults to `rapidata_config.upload.failureTolerance` = `0.0`, i.e. strict) is the fraction of datapoints allowed to fail uploading while the job definition is still created. Above the tolerance **no job definition is created at all** — see gotcha 8.

### Comparison

Compare two items and choose the better one.

```python
from rapidata import AllowNeitherBothSetting

job_def = client.job.create_compare_job_definition(
    name="Image Comparison",
    instruction="Which image is higher quality?",
    datapoints=[["img_a1.jpg", "img_b1.jpg"], ["img_a2.jpg", "img_b2.jpg"]],
    data_type="media",
    responses_per_datapoint=10,
    contexts=["Prompt that generated these"],
    media_contexts=[["reference.jpg"]],
    a_b_names=["Model A", "Model B"],
    confidence_threshold=0.99,       # Optional: confidence-based early stopping
    # quorum_threshold=7,            # Alternative: quorum-based early stopping (cannot use both)
    settings=[AllowNeitherBothSetting()],   # Allow "Neither" or "Both" options
)
```

### Ranking

Ranking is available via the **new job definition API** or via **continuous ranking flows** (see below).

```python
# New job definition API for ranking
job_def = client.job.create_ranking_job_definition(
    name="Image Quality Ranking",
    instruction="Which image looks better?",
    datapoints=[["img1.jpg", "img2.jpg", "img3.jpg"]],  # outer list = independent rankings
    comparison_budget_per_ranking=50,
    responses_per_comparison=1,
    random_comparisons_ratio=0.5,
    contexts=["Optional context"],
)
job = audience.assign_job(job_def)
job.display_progress_bar()
results = job.get_results()
```

**Small rankings are matched exhaustively.** A ranking with **more than 10 datapoints** is matched adaptively (Elo-style) within `comparison_budget_per_ranking`, and `random_comparisons_ratio` applies as usual. A ranking with **10 or fewer datapoints** instead compares every unique pair, spreading the budget evenly across pairs (rounded down to a multiple of the pair count; every pair is compared at least once even if the budget is smaller than the pair count) — here `random_comparisons_ratio` has **no effect**.

### Locate

Ask labelers to tap on the points in a datapoint that match your instruction. Results are the set of tapped coordinates per datapoint. No `answer_options`, `a_b_names`, `data_type`, `confidence_threshold`, or `quorum_threshold`.

```python
from rapidata import Box

job_def = client.job.create_locate_job_definition(
    name="Artifact Detection",
    instruction="Tap on any visual glitches or errors in the image.",
    datapoints=["img1.jpg", "img2.jpg"],
    responses_per_datapoint=35,
    contexts=["Optional text context"],
    media_contexts=[["optional_reference.jpg"]],
    settings=[LocateMaxPointsSetting(5)],
    private_metadata=[{"id": "abc"}],
)
```

For locate qualification examples, pass `truths` as a `list[Box]` (import `Box` from `rapidata`); coordinates are image ratios (0.0–1.0):

```python
audience.add_locate_example(
    instruction="Tap on any visual glitches or errors in the image.",
    datapoint="example_with_artifact.jpg",
    truths=[Box(x_min=0.44, y_min=0.42, x_max=0.58, y_max=0.63)],
    context="Optional context",
    explanation="The artifact is in the highlighted region.",  # Shown to labelers who answer incorrectly
    settings=[...],  # Optional: match job settings so labelers qualify on the same UI
)
```

### Draw

Ask labelers to draw (color in) regions of an image that match your instruction. Results are the set of drawn lines per datapoint. No `answer_options`, `a_b_names`, `data_type`, `confidence_threshold`, or `quorum_threshold`.

```python
job_def = client.job.create_draw_job_definition(
    name="Artifact Drawing",
    instruction="Color in any visual glitches or errors in the image.",
    datapoints=["img1.jpg", "img2.jpg"],
    responses_per_datapoint=35,
    contexts=["Optional text context"],
    media_contexts=[["optional_reference.jpg"]],
    private_metadata=[{"id": "abc"}],
)
```

For draw qualification examples, pass `truths` as a `list[Box]` (import `Box` from `rapidata`); coordinates are image ratios (0.0–1.0):

```python
audience.add_draw_example(
    instruction="Color in any visual glitches or errors in the image.",
    datapoint="example_with_artifact.jpg",
    truths=[Box(x_min=0.44, y_min=0.42, x_max=0.58, y_max=0.63)],
    explanation="The artifact is within the highlighted region.",  # Shown to labelers who answer incorrectly
    settings=[...],  # Optional: match job settings so labelers qualify on the same UI
)
```

### Select Words

Ask labelers to select the words from a sentence that match your instruction (e.g., words not depicted in an image). Each datapoint is paired with a `sentence` split by spaces. No `contexts`, `media_contexts`, `data_type`, `answer_options`, `a_b_names`, `confidence_threshold`, or `quorum_threshold`.

```python
job_def = client.job.create_select_words_job_definition(
    name="Image-Text Alignment",
    instruction="Select the words not correctly depicted in the image.",
    datapoints=["img1.jpg", "img2.jpg"],
    sentences=["A cat on a red couch [No_mistakes]", "A blue car in the rain [No_mistakes]"],
    responses_per_datapoint=15,
    private_metadata=[{"id": "abc"}],
)
```

For select words qualification examples, pass `truths` as a `list[int]` of 0-based word indices to select:

```python
audience.add_select_words_example(
    instruction="Select the words not correctly depicted in the image.",
    datapoint="image.jpg",
    sentence="a white cat on a sunny beach [No_mistakes]",
    truths=[1],  # 0-based word indices; e.g. index 1 = "white" if the cat is black
    explanation="The cat in the image is black, not white.",  # Shown to labelers who answer incorrectly
    settings=[...],  # Optional: match job settings so labelers qualify on the same UI
)
```

### Free Text

Ask labelers to answer your instruction with free-form text. No `answer_options`, `a_b_names`, `confidence_threshold`, or `quorum_threshold`. Note: free text answers cannot be graded against a ground truth, so audiences cannot be trained with free text qualification examples.

```python
job_def = client.job.create_free_text_job_definition(
    name="Prompt Collection",
    instruction="What would you like to ask an AI? Please spell out the question.",
    datapoints=["image.jpg"],
    responses_per_datapoint=15,
    contexts=["Optional text context"],
)
```

### Custom Audiences

Create an audience trained on your specific task.

⚠️ **Recruiting is explicit — you must start it yourself.** A custom audience recruits nobody until you (1) add **≥3 qualification examples** with `add_*_example(...)` and (2) call `audience.start_recruiting()`. Adding examples does **not** start recruiting; until `start_recruiting()` is called the audience stays in its `Created` state. A job assigned to such an audience is still created (with a warning), but it can never receive responses — `display_progress_bar()` / `get_results()` raise an error explaining that nobody graduated and nobody is being recruited. Call `start_recruiting()` **once**, after all examples are added and reviewed, before `assign_job`. If you don't need task-specific qualification, skip all of this and use the ready-to-go global pool with no setup: `client.audience.get_audience_by_id("global")`.

**Important:** Every qualification example and its associated truth must be manually and thoroughly reviewed by a human before use. If an example has a wrong or ambiguous truth value, the qualification process will filter out good labelers who answer correctly while letting through bad labelers who happen to match the incorrect answer — completely inverting quality control. Always verify that each example has a clear, unambiguous correct answer.

```python
audience = client.audience.create_audience(
    name="Expert Evaluators",
    # target_accuracy=0.8,   # Optional: fraction of qualification tasks (0-1) a labeler must get right (server default 0.75)
    # min_tasks=12,          # Optional: qualification tasks before the accuracy verdict is trusted (server default 10)
    # max_tasks=30,          # Optional: cap on admission-trial tasks before a verdict is forced (default: no cap)
)

# Add classification examples
audience.add_classification_example(
    instruction="Rate image quality",
    answer_options=["Poor", "Good", "Excellent"],
    datapoint="example.jpg",
    truth=["Excellent"],
    context="Optional context",
    data_type="media",
    explanation="This image is excellent due to its high resolution and sharp focus.",  # Shown to labelers who answer incorrectly
    settings=[NoShuffleSetting()],  # Optional: match job settings so labelers qualify on the same UI
)

# Add comparison examples
audience.add_compare_example(
    instruction="Which image follows the prompt better?",
    datapoint=["good.jpg", "bad.jpg"],
    truth="good.jpg",
    context="A cat on a chair",
    data_type="media",
    explanation="The first image clearly shows a cat sitting on a chair as described.",  # Shown to labelers who answer incorrectly
    settings=[AllowNeitherBothSetting()],  # Optional: match job settings so labelers qualify on the same UI
)

# Add locate examples (requires: from rapidata import Box)
audience.add_locate_example(
    instruction="Tap on any visual glitches or errors in the image.",
    datapoint="example_with_artifact.jpg",
    truths=[Box(x_min=0.44, y_min=0.42, x_max=0.58, y_max=0.63)],
    context="Optional context",
    explanation="The artifact is in the highlighted region.",  # Shown to labelers who answer incorrectly
    settings=[...],  # Optional: match job settings so labelers qualify on the same UI
)

# Add draw examples (requires: from rapidata import Box)
audience.add_draw_example(
    instruction="Color in any visual glitches or errors in the image.",
    datapoint="example_with_artifact.jpg",
    truths=[Box(x_min=0.44, y_min=0.42, x_max=0.58, y_max=0.63)],
    explanation="The artifact is within the highlighted region.",  # Shown to labelers who answer incorrectly
    settings=[...],  # Optional: match job settings so labelers qualify on the same UI
)

# Add select words examples
audience.add_select_words_example(
    instruction="Select the words not correctly depicted in the image.",
    datapoint="image.jpg",
    sentence="a white cat on a sunny beach [No_mistakes]",
    truths=[1],  # 0-based word indices to select
    explanation="The cat in the image is black, not white.",  # Shown to labelers who answer incorrectly
    settings=[...],  # Optional: match job settings so labelers qualify on the same UI
)

# Inspect the examples currently on the audience
examples_df = audience.get_examples(amount=10, page=1)

# Once >=3 examples are added and reviewed, start recruiting. This is required and explicit:
# adding examples does not start it, and a job assigned before this can never get responses.
audience.start_recruiting()

# Watch the funnel fill up (graduated / distilling / dropped / inactive)
metrics = audience.get_recruiting_metrics()
print(metrics.graduated, metrics.distilling)

# Now the audience recruits against the examples; assign a job as usual.
job = audience.assign_job(job_def)
```

**Managing audiences (`client.audience`):**
- `client.audience.create_audience(name, filters=None, target_accuracy=None, min_tasks=None, max_tasks=None)` — create a custom audience. The last three set the admission bar for qualification: `target_accuracy` (0–1, server default `0.75`), `min_tasks` (server default `10`), `max_tasks` (no cap by default). Supplying only some of them is fine — the SDK fills in the defaults. Raises `ValueError` for an accuracy outside 0–1, `min_tasks < 1`, or `max_tasks < min_tasks`.
- `client.audience.get_audience_by_id(audience_id)` — fetch by id; pass `"global"` for the ready-to-go global audience
- `client.audience.find_audiences(name="", amount=10, page=1)` — list your audiences (newest first), optionally filtered by name

**Audience methods:**
- `audience.start_recruiting()` — begin recruiting/onboarding annotators against the audience's qualification examples. **Required and explicit for custom audiences**: call it once, after adding ≥3 examples and before `assign_job` — adding examples never starts recruiting on its own, and a job assigned before recruiting has started can never receive responses. Calling it more than once is a no-op; a backend failure raises `RapidataError` rather than being swallowed. Returns the audience, so it chains. Not needed for the global/curated pools.
- `audience.get_recruiting_metrics()` — snapshot of the audience's recruiting funnel as a `RecruitingMetrics` (`graduated` = eligible to work now, `distilling` = still qualifying, `dropped`, `inactive`; one bucket per annotator). All zeros before `start_recruiting()` has pulled anyone in, and for curated audiences.
- `audience.assign_job(job_definition)` — start a job. Never blocks on funds: the job is always created, but if its estimated cost exceeds your account balance a cost warning is logged (with the estimate, your balance, and the shortfall) and the job may pause partway until you top up. A warning is also logged if the audience has no graduated annotators yet.
- `audience.find_jobs(name="filter", amount=10, page=1)` — find assigned jobs
- `audience.update_filters([...])` — set the recruitment filters on this audience (audience-supported filters only — see below)
- `audience.filter([filters])` — derive a filtered subset of this audience without re-onboarding labelers; supports `CountryFilter`, `LanguageFilter`, `DemographicFilter`, `AgeFilter`, `GenderFilter`, and `DeviceFilter`, plus the `AndFilter`/`OrFilter`/`NotFilter` combinators (also via `&` / `|` / `~`); returns a `RapidataFilteredAudience` (exposes `assign_job`, `find_jobs`, `delete` only — no `add_classification_example`, `update_filters`, or nested `.filter()`)
- `audience.update_name("New Name")` — rename
- `audience.get_examples(amount=10, page=1)` — list qualification examples (returns DataFrame)
- `audience.delete()` — delete the audience

**Audience-supported filters:** `update_filters(...)` (recruitment filters) accepts `CountryFilter` and `LanguageFilter`; `.filter(...)` (deriving a filtered audience from graduates) additionally accepts `DemographicFilter` (age/gender/occupation), `AgeFilter`, `GenderFilter`, and `DeviceFilter`. Both accept the `AndFilter`/`OrFilter`/`NotFilter` combinators. `UserScoreFilter`, `CampaignFilter`, and `CustomFilter` are **not supported on audiences** and raise `NotImplementedError`.

**Job / Job Definition methods:**
- `job_def.preview()` — open browser preview of what labelers see
- `job_def.update_dataset(datapoints=..., data_type=..., contexts=..., media_contexts=..., private_metadata=...)` — replace the datapoints on an existing job definition
- `job_def.estimated_cost` / `job.estimated_cost` — a `CostEstimate` (`estimated_cost`, `datapoint_count`, `required_responses`) for running to completion; available on a job definition *before* assigning it. The estimate is priced shortly after creation, so the first read blocks briefly until it's ready (raises `TimeoutError` if still unavailable after a few minutes), then caches. It's an estimate, not the final bill — early stopping can lower the actual cost.
- `job_def.delete()` — delete a job definition and all its revisions
- `job.display_progress_bar(refresh_rate=5)` — blocking progress bar
- `job.get_status()` — current status string
- `job.get_progress()` — non-blocking snapshot: a `JobProgress` with `state` (same value as `get_status()`), `completion_percentage` (0–100) and `recruiting` (a `RecruitingMetrics`, or `None` for curated audiences)
- `job.get_results()` — blocks until Completed/Failed (auto-regenerates if `StaleResults`), returns `RapidataResults`. If the job needs manual review (`ManualApproval`) or runs out of funds mid-run (`SpendLimited`) — neither state completes on its own — it raises an informative error naming the state instead of blocking; top up or wait for a reviewer, then call it again. It also raises up front when the job's audience **can never produce responses** (nobody graduated *and* nobody is being recruited); an audience that is merely still distilling does not raise.
- `job.view()` — open the job's details page in the browser
- `job.delete()` — delete a running job

## Context Management

Datapoint contexts have a **400-character maximum**; the backend rejects longer ones. An over-long context is therefore **always** shortened against the task instruction before upload — this cannot be turned off — and a warning reports how many contexts were shortened:

```python
job_def = client.job.create_classification_job_definition(
    name="Outfit check",
    instruction="Does the main character wear the right clothing?",
    answer_options=["Yes", "No"],
    datapoints=["scene.jpg"],
    contexts=["<a very long, detailed scene description ...>"],
)
```

A context tuned to the question focuses the labeler even when it already fits the limit. Set `rapidata_config.upload.contextShortening = True` to have **every** context shortened, not just the over-long ones:

```python
from rapidata import rapidata_config

rapidata_config.upload.contextShortening = True
```

Shorten contexts directly without creating a job via `client.context`:

```python
# Single context
short = client.context.shorten_context(
    context="<a very long description ...>",
    question="Does the main character wear the right clothing?",
)

# Batch: (context, question) pairs, order preserved; sent as concurrent batches of 10
shortened = client.context.shorten_contexts([
    (context_a, question_a),
    (context_b, question_b),
])
```

`ContextManager` is also importable directly: `from rapidata import ContextManager`.

## Migration from the removed Order API

The order-based API (`client.order`, `RapidataOrderManager`, `RapidataOrder`) was **removed** in v3.21.0 — `client.order` no longer exists, and `from rapidata import RapidataOrder` / `RapidataOrderManager` will fail. The job-definition + audience model is now the only supported path.

To migrate: replace `create_classification_order()` / `create_compare_order()` (and the other `create_*_order()` methods) with the matching `create_*_job_definition()` on `client.job`, replace validation sets with audience qualification examples, and use `audience.assign_job(job_def)` instead of `.run()`. Classification, comparison, locate, draw, select words, free text, and ranking are all available via the job definition API.

## Settings

Settings control how a task is rendered and behaves for labelers. All settings are importable from the top-level `rapidata` package.

Most settings only apply to specific task types. If you add a setting that the job's task type does not support, the SDK logs a non-fatal warning and still sends the flag — it is never dropped and no error is raised. Ranking jobs are treated as Compare for this check.

```python
from rapidata import (
    NoShuffleSetting, AllowNeitherBothSetting, MarkdownSetting,
    MuteVideoSetting, FreeTextMinimumCharactersSetting, FreeTextMaxCharactersSetting,
    SwapContextInstructionSetting, PlayPercentageVideoSetting,
    OriginalLanguageOnlySetting, NoMistakeOptionSetting, DisableAutoloopSetting,
    NoInstructionDisplaySetting, KeyboardNumericSetting,
    LocateMaxPointsSetting, LocateMinPointsSetting,
    ComparePanoramaSetting, CompareEquirectangularSetting,
    ClassifyEquirectangularSetting,
    CustomSetting,
)

settings=[NoShuffleSetting()]                             # Keep answer options in order (use for Likert scales)
settings=[AllowNeitherBothSetting()]                      # Comparison: allow "Neither"/"Both"
settings=[MarkdownSetting()]                              # Render markdown in text
settings=[MuteVideoSetting()]                             # Start videos muted
settings=[FreeTextMinimumCharactersSetting(50)]           # Min text length for free-text tasks (use with caution — see note below)
settings=[FreeTextMaxCharactersSetting(500)]              # Max text length for free-text tasks (default 1024) (use with caution — see note below)
settings=[SwapContextInstructionSetting()]                # Swap the positions of context and instruction
settings=[PlayPercentageVideoSetting(percentage=95)]      # Require labelers to watch N% of video before answering (0-95)
settings=[OriginalLanguageOnlySetting()]                  # Do not translate the task
settings=[NoMistakeOptionSetting()]                       # Hide the "mark as mistake" option
settings=[DisableAutoloopSetting()]                       # Disable automatic looping of media
settings=[NoInstructionDisplaySetting()]                  # Hide instruction on the task screen
settings=[KeyboardNumericSetting()]                       # Open numeric keyboard on mobile
settings=[LocateMaxPointsSetting(5)]                      # Locate task: max number of points (default 3)
settings=[LocateMinPointsSetting(1)]                      # Locate task: min number of points
settings=[ComparePanoramaSetting()]                       # Render comparison media as 360° panorama
settings=[CompareEquirectangularSetting()]                # Render comparison media as equirectangular VR
settings=[ClassifyEquirectangularSetting()]               # Render classification media as equirectangular 360° view
settings=[CustomSetting(key="my_flag", value="on")]              # Rapid-level flag (default); use target="campaign" for campaign-level
```

**Note on `FreeTextMinimumCharactersSetting` / `FreeTextMaxCharactersSetting`:** use these with caution. Free-text responses already pass through a reasonableness check by default, so tightening the bounds is usually unnecessary and will reject otherwise valid answers. Only set them when the question genuinely demands a specific length (e.g. a single word, or a full paragraph).

## Key Gotchas

1. **Watch early responses** — after assigning, call `job.view()` to open the running job in the browser and check that labelers understand the instruction as intended
2. **Use `NoShuffleSetting()` for Likert scales** — ordered answer options get shuffled by default
3. **Custom audiences need ≥3 examples AND an explicit `start_recruiting()`** — recruiting never begins on its own. Add **≥3 qualification examples** (`add_*_example(...)`), then call `audience.start_recruiting()` **once**, before `assign_job`. Skip either step and the audience recruits nobody: the job is still created (with a warning), but it can never receive responses, so `get_results()` / `display_progress_bar()` raise an error saying the audience can never produce responses. Use `client.audience.get_audience_by_id("global")` when you don't need task-specific qualification.
   - **One audience = one task.** A custom audience is qualified for the specific task its examples describe. Reusing it for a different, unrelated task is a misuse — the qualification no longer applies and the quality guarantee is lost. Reuse it only for repeated/scheduled runs of the *same* task; spin up a new audience for a new task.
4. **25-second time limit, 250-character instructions** — labelers have ~25 seconds per task; keep instructions concise. Instructions (and the `target` of locate/draw tasks) are capped at **250 characters** — a longer one raises `ValueError: instruction is <n> characters; maximum is 250` when the job definition or qualification example is created
5. **Responses may exceed `responses_per_datapoint`** — concurrent labelers can cause slight overflow
6. **Two early stopping strategies, mutually exclusive** — `confidence_threshold` (statistical, weighted by labeler trust scores) or `quorum_threshold` (stops when N responses agree); cannot use both at once
7. **Early stopping only for unambiguous tasks** — both strategies work best when there's a clear correct answer
8. **Failed uploads abort job-definition creation** — job definitions are created atomically: if more than `failure_tolerance` of the datapoints fail to upload, **no job definition is created** (`e.job_definition` is `None`) and at least one datapoint must always succeed. Fix the failing datapoints and call `e.retry()`, which re-uploads only the failed ones into the *same* dataset and finishes creating the definition; it raises `FailedUploadException` again if failures remain, so it can be looped. Within tolerance, the definition is created and a warning reports how many failed. Inspect failures via `e.failures_by_reason`, `e.failures_by_stage` (grouped by remote-URL ingestion stage — only `internal` is a Rapidata-side fault), and each `FailedUpload`'s `stage` / `http_status`
9. **Preview link printed on job creation** — when a job definition is created, a dashboard preview link is printed automatically (QR-code previews were removed in v3.21.0); suppress it with `rapidata_config.logging.silent_mode = True`
10. **Context length limit is 400 characters** — the backend rejects contexts longer than 400 characters, so an over-long context is **always** shortened against the task instruction before upload (not optional; a warning reports how many were shortened). Set `rapidata_config.upload.contextShortening = True` to shorten *every* context, or use `client.context.shorten_context()` / `client.context.shorten_contexts()` to shorten manually.
11. **Jobs can pause for manual review or funds** — `assign_job` always creates the job, but if its estimated cost exceeds your account balance it logs a cost warning and the job may pause until you top up. A job can also enter manual review (`ManualApproval`) or become spend-limited (`SpendLimited`) mid-run; since neither state completes on its own, `get_results()` raises an informative error naming the state instead of blocking — top up or wait for a reviewer, then retry.

## Ranking Flows (Continuous Ranking)

Lightweight continuous ranking without full job/audience setup:

```python
# Create flow
flow = client.flow.create_ranking_flow(
    name="Image Quality Ranking",
    instruction="Which image looks better?",
    max_response_threshold=100,       # Target responses per flow item (default 100)
    min_response_threshold=50,        # Minimum acceptable responses; item is Incomplete if TTL expires below this
    # validation_set_id="...",        # Optional: run a validation set alongside the flow
    # settings=[NoShuffleSetting()],  # Optional: flow-wide settings
)

# Preheat for low-latency responses (call ~5 minutes before time-sensitive batches)
client.flow.preheat()

# Add items to rank
flow_item = flow.create_new_flow_batch(
    datapoints=["img1.jpg", "img2.jpg", "img3.jpg"],
    context="Generated by Model X",
    # context_assets=["reference.jpg"],  # Optional: 1–10 image/video/audio paths/URLs shown alongside instruction
    time_to_live=300,  # Seconds until expiry (45–3600; defaults to 3600 when omitted)
)

# Get results (flow items have their own result shape, not RapidataResults)
results = flow_item.get_results()         # Blocks until complete; returns FlowItemResult(datapoints, total_votes)
status = flow_item.get_status()           # Non-blocking check
matrix = flow_item.get_win_loss_matrix()  # Pandas DataFrame (blocks until complete)
count = flow_item.get_response_count()

# Tune a flow after creation
flow.update_config(
    instruction="New instruction",
    starting_elo=1000,
    min_responses=40,
    max_responses=120,
)

# Manage flows
all_flows = client.flow.find_flows(name="", amount=10, page=1)
flow = client.flow.get_flow_by_id("flow_id")
items = flow.get_flow_items(amount=10, page=1)
flow.delete()
```

Note: `RapidataFlowItem` does **not** have `display_progress_bar()` — poll with `get_status()` or just call `get_results()` to block.

## Model Ranking Insights (MRI / Benchmarks)

Compare and rank AI models on leaderboards. Supports images, videos, audio, and text.

```python
# Create benchmark
benchmark = client.mri.create_new_benchmark(
    name="AI Art Competition",
    prompts=["A serene mountain landscape", "A futuristic city"],
    # identifiers=[...],        # Optional: stable ids for each prompt
    # prompt_assets=[...],      # Optional: reference media for each prompt
    # tags=[...],               # Optional: per-prompt tags — str, Tag(value, category=...), or a mix
    # origins=[...],            # Optional: per-prompt Origin(source) or plain source string
    # description=None,         # Optional: plain-text credit for the benchmark (max 2000 characters)
)

# Add prompts later if needed (one or many, matched up by index)
benchmark.add_prompts(
    prompts=["A quiet lake at dawn"],
    # identifiers=["dawn_lake"],   # Optional: stable id per prompt
    # prompt_assets=["ref.jpg"],   # Optional: reference media per prompt
    # tags=[["landscape"]],        # Optional: list of tag lists, one per prompt (str and/or Tag)
    # origins=["coco"],            # Optional: where each prompt came from (Origin or str)
)

# Tags carry an optional category; bare strings become Tag(value, category=None)
from rapidata import Tag, Origin

benchmark.add_prompts(
    identifiers=["garage_car"],
    prompts=["A car in a garage"],
    tags=[[Tag("vehicle", category="object"), "indoor"]],
    origins=[Origin("coco")],
)

# Re-tag / set the origin of an already-registered prompt (a field left None stays unchanged)
benchmark.update_prompt("garage_car", tags=["abstract", "surreal"], origin="wikiart")

print(benchmark.tags)             # Values only, aligned by index with prompts (categories dropped)
print(benchmark.structured_tags)  # list[list[Tag]] — preferred, keeps categories
print(benchmark.origins)          # list[Origin | None]

# Create leaderboard
leaderboard = benchmark.create_leaderboard(
    name="Realism",
    instruction="Which image is more realistic?",
    show_prompt=False,
    show_prompt_asset=False,
    inverse_ranking=False,
    # level_of_detail="high",            # "debug" (20) | "low" (2000) | "medium" (4000) | "high" (8000)
    #                                    #   | "very high" (16000), or a positive int response budget
    # min_responses_per_matchup=5,
    # audience_id="...",                 # Optional: id string, RapidataAudience, or RapidataFilteredAudience
    # settings=[...],
    # included_tags=["outdoor"],         # Optional: only collect matchups for prompts carrying one of these tags
    # excluded_tags=["nsfw"],            # Optional: skip prompts carrying one of these tags (always wins)
    # vote_aggregation=VoteAggregation.MAJORITY_VOTE,  # default; or VoteAggregation.ALL_VOTES — how a matchup's
    #                                    #   individual responses are aggregated (import VoteAggregation from rapidata)
    # skip_initial_run=False,            # Optional: when True, skip the initial run that evaluates the models already
    #                                    #   in the benchmark against each other — you start with no responses/standings.
    #                                    #   Later add_model still compares against the whole field; create-only (not readable back).
    # benchmarkDescription="...",        # Optional: description for a newly created benchmark (max 2000 chars; ignored if benchmark already exists)
)

# Evaluate a model (creates participant, uploads media, and submits in one step)
benchmark.evaluate_model(
    name="MyModel_v2",
    media=["mountain.png", "city.png"],
    prompts=["A serene mountain landscape", "A futuristic city"],
    data_type="media",   # "media" (default) or "text"
)

# Or add a model without submitting (for more control)
participant = benchmark.add_model(
    name="MyModel_v3",
    media=["mountain_v3.png", "city_v3.png"],
    prompts=["A serene mountain landscape", "A futuristic city"],
    data_type="media",
)

# Upload additional media to the same participant
uploaded, failed = participant.upload_media(
    assets=["mountain_v3_extra.png"],
    identifiers=["A serene mountain landscape"],
    data_type="media",
)
# Returns (identifiers uploaded, list[FailedUpload[SampleUpload]]). Each FailedUpload
# carries the media/identifier pair (.item, a SampleUpload), the reason, and a trace id,
# so a failed pair can be re-submitted directly. Raises ValueError if assets and
# identifiers differ in length.

# Recover a partial upload (server truth — works for any participant, incl. ones from
# benchmark.participants). add_model already runs this sweep automatically on failure.
missing = participant.missing_counts(identifiers)   # Counter[identifier -> samples still short]; empty == done
uploaded, still_failed = participant.retry_missing(  # re-sends assets for short identifiers
    assets=["mountain_v3_extra.png"],
    identifiers=["A serene mountain landscape"],
    data_type="media",
)
# Safe to call repeatedly (the backend rejects samples already held, so no duplication).

# Submit individually or all at once
participant.run()       # Submit one participant
benchmark.run()         # Submit all unsubmitted (CREATED) participants

# Faucet — configure a participant to auto-generate samples via Replicate
participant.set_faucet(
    model_owner="stability-ai",  # Replicate model owner (e.g. "stability-ai")
    model_name="sdxl",           # Model name (e.g. "sdxl")
    model_version=None,          # Optional: pin a specific version hash
    additional_inputs={"aspect_ratio": "16:9"},  # Optional: extra model inputs (not prompt/num_outputs)
)
participant.delete_faucet()      # Remove the faucet from the participant
participant.disable()             # Exclude from evaluation and standings (reversible)
participant.enable()              # Re-enable a previously disabled participant
participant.rename("New Name")    # Rename the participant
participant.get_elo()             # Aggregated Elo across all leaderboards (None if not yet computed)
participant.delete()              # Delete participant and its uploaded media (cannot be undone)

# Sample generation — trigger a batch generation run across participants with faucets
sample_gen = benchmark.generate_samples(
    samples_per_prompt=3,         # How many samples per prompt (1–16)
    participant_ids=None,          # Optional: restrict to specific participant ids
    prompt_identifiers=None,       # Optional: restrict to specific prompt identifiers
    tags=None,                     # Optional: restrict to prompts matching any of these tags
)
# sample_gen.id                       — generation request id
# sample_gen.total_count              — total items queued
# sample_gen.skipped_participant_ids  — participants without a configured faucet

# List participants and their status (p.faucet is None if no faucet is configured)
for p in benchmark.participants:
    print(p.name, p.status, p.faucet)

# Prompts — original language and English translation (aligned by index)
print(benchmark.prompts)          # As originally provided
print(benchmark.english_prompts)  # Server-side English translations, aligned by index
print(benchmark.description)      # Optional plain-text credit (None if not set)

# Get results
standings = leaderboard.get_standings()                    # Pandas DataFrame for one leaderboard
overall = benchmark.get_overall_standings(tags=None, leaderboard_ids=None)  # Aggregated ELO across all leaderboards
matrix_lb = leaderboard.get_win_loss_matrix()              # Pairwise wins/losses for one leaderboard
matrix_bm = benchmark.get_win_loss_matrix(                 # Pairwise wins/losses across leaderboards
    tags=None, participant_ids=None, leaderboard_ids=None, use_weighted_scoring=None,
)

# Access the jobs that ran for a leaderboard (one per run, most recent first)
for job in leaderboard.jobs:
    job_results = job.get_results()

# Update leaderboard config live — all mutation goes through update(); the old property
# setters (leaderboard.name = ..., .level_of_detail = ..., .min_responses_per_matchup = ...)
# were removed and now raise AttributeError. Only the arguments you pass are changed;
# omitted ones keep their stored value, and everything goes out in one PATCH request.
leaderboard.update(
    name="Realism (Updated)",                     # non-empty string
    level_of_detail="very high",                  # named level or a positive int budget (e.g. 5000)
    min_responses_per_matchup=7,                  # int >= 3 (bool rejected); takes effect for future evaluations
    vote_aggregation=VoteAggregation.MAJORITY_VOTE,  # re-counts already-collected responses (no re-evaluation)
)
# A no-argument update() sends an empty patch. Changing level_of_detail / min_responses_per_matchup
# only affects future evaluations; already-computed standings are not recomputed.

# Reading back the config (all read-only properties)
print(leaderboard.level_of_detail)   # A named level only on an exact budget match, otherwise "custom"
print(leaderboard.response_budget)   # The exact budget behind it, e.g. 5000
print(leaderboard.vote_aggregation)  # A VoteAggregation member (lazily fetched for leaderboards read from a listing)
print(leaderboard.included_tags, leaderboard.excluded_tags)  # Fixed at creation — create a new leaderboard to re-scope

# Open in browser
benchmark.view()
leaderboard.view()

# Find existing benchmarks
benchmarks = client.mri.find_benchmarks(name="AI Art", amount=10)
benchmark = client.mri.get_benchmark_by_id("benchmark_id")
```

## Signals (Scheduled Labeling)

A signal runs the same labeling job on a repeating schedule: bind a job definition to an audience and an interval, and Rapidata creates a new job on every tick.

```python
from rapidata import RapidataClient

client = RapidataClient()

audience = client.audience.get_audience_by_id("aud_MU1GZYoESyO")

job_def = client.job.create_compare_job_definition(
    name="Prompt Alignment Job",
    instruction="Which image follows the prompt more accurately?",
    datapoints=[["flux_book.jpg", "mj_book.jpg"]],
    contexts=["A small blue book sitting on a large red book."],
)

signal = client.signals.create_signal(
    name="Daily prompt alignment",
    audience=audience,           # also accepts id string
    job_definition=job_def,      # also accepts id string
    interval_hours=24,
    # revision_number=...,       # Optional: pin a specific job-definition revision
    # is_public=True,            # Optional: let others in your org read the signal
)

# Inspect jobs created by the signal
for job in signal.get_jobs(page_size=10):
    print(job, job.get_status())

# Fire one job immediately instead of waiting for the schedule
signal.trigger()
job = signal.wait_for_next_job(timeout=600)  # blocks until the job is created
print(job.get_results())

# Manage the signal
signal.pause()
signal.resume()
signal.update(name="Hourly prompt alignment", interval_hours=1)
signal.delete()

# Look signals up later
signal = client.signals.get_signal_by_id("signal_id")
signals = client.signals.find_signals(name="alignment")
```

**Signal properties:** `id`, `name`, `description`, `audience_id`, `job_definition_id`, `revision_number`, `interval_hours`, `next_run_at`, `last_run_at`, `is_paused`, `is_public`, `created_at`.

Note: `signal.pause()` only affects the scheduler — manual `trigger()` calls still fire on a paused signal.

## Billing

`client.billing` reads how much the current billing period has cost so far and how much credit is left. Billing is settled per **organization**, so the figures cover everything the organization spent — not only the jobs this client created.

```python
from rapidata import RapidataClient, BillingPeriod, RapidataBillingManager

client = RapidataClient()

# The billing period currently accruing cost. Raises RapidataError (status 404)
# if the organization has no active period (one only opens once there is something to bill).
period = client.billing.get_current_billing_period()   # -> BillingPeriod
```

`BillingPeriod` is a frozen dataclass; all amounts are US dollars rounded to the cent, and each read is a snapshot (fetch again for an up-to-date figure):

| Field | Description |
|---|---|
| `id` | The billing period's id. |
| `start_date` / `end_date` | When the period starts and ends (`datetime`). |
| `status` | `"Open"` while still accruing cost; otherwise one of `"Invoiced"`, `"Void"`, `"Reconciling"`, `"PendingReview"`, `"Closed"`. |
| `outstanding_cost` | Net cost accrued so far (`gross_cost` minus `discount`) — what the period would be invoiced for today. |
| `gross_cost` | Cost accrued so far, before discounts. |
| `discount` | Discounts applied to the period so far. |
| `response_count` | Number of billable responses collected in the period (`int`). |
| `credits` | Prepaid credit still available, or `None` when the organization is billed for usage rather than from a prepaid balance. An organization-level balance that carries across periods. |
| `effective_limit` | The most the organization may spend this period, or `None` when it spends without a cap. On a prepaid plan this is the total credit granted, and `credits` is what remains of it. |

## Additional Resources

- For complete API reference, all parameters, filters, results format, error handling, flows, and MRI: see [reference.md](reference.md)
- For full end-to-end code examples and common patterns: see [examples.md](examples.md)
