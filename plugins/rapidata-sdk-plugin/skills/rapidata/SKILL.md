---
name: rapidata
description: Explains how to use the Rapidata API to get real and fast human annotations for your data. Use when writing code that creates labeling tasks, compares models, collects human feedback, or integrates with the Rapidata Python SDK.
---

# Rapidata Python SDK

Rapidata connects you with distributed human labelers worldwide for fast, high-quality data annotation. The SDK lets you create labeling tasks, manage annotator audiences, and retrieve results programmatically.

## Before you start: check the skill is up to date

This skill is pinned to **Rapidata SDK v3.9.8**. Run this check **once at the start of a Rapidata task** (not on every call) to confirm the user's runtime matches the skill:

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
     > ⚠️ The Rapidata skill is pinned to v3.9.8 but v{installed} is installed — the skill docs may be out of date. I've suggested updating the plugin; if the update isn't available yet, I'll proceed with the documented API and flag any surprises.
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
# Empty values are treated as unset and fall through to the next resolution layer.
```

## Core Concepts

- **Job Definition**: A reusable configuration template for a labeling task (task type, instruction, datapoints, answer options)
- **Audience**: A group of pre-screened labelers. Can be global (fast, baseline quality), curated (pre-trained on domains like "alignment"), or custom (trained with your qualification examples)
- **Job**: A running instance of a job definition assigned to an audience
- **Flow**: Lightweight continuous ranking without full job setup
- **MRI/Benchmark**: Compare and rank AI models on leaderboards

**Client entry points:**
- `client.job` — create job definitions (classification, comparison)
- `client.audience` — create and find audiences
- `client.flow` — continuous ranking flows
- `client.mri` — model ranking insights / benchmarks
- `client.order` — legacy order API (still supported; currently the way to run ranking, locate, free-text, select-words, draw jobs)

## New API: Job Definitions + Audiences (Recommended)

The new API currently exposes **classification** and **comparison** as job definitions. For ranking, continue to use either the Flow API (recommended for continuous ranking) or the legacy `client.order.create_ranking_order(...)`. For locate / free text / select words / draw, use the legacy order API.

### Step-by-step workflow

```python
from rapidata import RapidataClient

client = RapidataClient()

# 1. Get or create an audience
audience = client.audience.find_audiences("alignment")[0]  # curated
# or: audience = client.audience.create_audience(name="My Evaluators")

# 2. Create a job definition
job_def = client.job.create_classification_job_definition(
    name="Animal Classification",
    instruction="What animal is in this image?",
    answer_options=["Cat", "Dog", "Bird"],
    datapoints=["https://example.com/img1.jpg", "img2.jpg"],
    responses_per_datapoint=10,
)

# 3. Preview (opens browser to see what labelers will see)
job_def.preview()

# 4. Assign to audience (starts labeling)
job = audience.assign_job(job_def)

# 5. Monitor and get results
job.display_progress_bar()
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
    media_contexts=["optional_reference.jpg"],
    confidence_threshold=0.99,      # Optional: confidence-based early stopping
    # quorum_threshold=7,           # Alternative: quorum-based early stopping (cannot use both)
    settings=[NoShuffleSetting()],         # Keep answer order
    private_metadata=[{"id": "abc"}],
)
```

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
    media_contexts=["reference.jpg"],
    a_b_names=["Model A", "Model B"],
    confidence_threshold=0.99,       # Optional: confidence-based early stopping
    # quorum_threshold=7,            # Alternative: quorum-based early stopping (cannot use both)
    settings=[AllowNeitherBothSetting()],   # Allow "Neither" or "Both" options
)
```

### Ranking

Ranking is available via the **legacy order API** or via **continuous ranking flows** (see below). The new job-definition API does not currently expose a public ranking constructor.

```python
# Legacy order API for ranking
order = client.order.create_ranking_order(
    name="Image Quality Ranking",
    instruction="Rank these images by quality",
    datapoints=[["img1.jpg", "img2.jpg", "img3.jpg"]],
    comparison_budget_per_ranking=50,
    responses_per_comparison=1,
    data_type="media",
    random_comparisons_ratio=0.5,
    contexts=["Optional context"],
).run()

order.display_progress_bar()
results = order.get_results()
```

### Custom Audiences

Create an audience trained on your specific task. Minimum 3 examples required before recruiting starts.

**Important:** Every qualification example and its associated truth must be manually and thoroughly reviewed by a human before use. If an example has a wrong or ambiguous truth value, the qualification process will filter out good labelers who answer correctly while letting through bad labelers who happen to match the incorrect answer — completely inverting quality control. Always verify that each example has a clear, unambiguous correct answer.

```python
audience = client.audience.create_audience(name="Expert Evaluators")

# Add classification examples
audience.add_classification_example(
    instruction="Rate image quality",
    answer_options=["Poor", "Good", "Excellent"],
    datapoint="example.jpg",
    truth=["Excellent"],
    context="Optional context",
    data_type="media",
    explanation="This image is excellent due to its high resolution and sharp focus.",  # Shown to labelers who answer incorrectly
)

# Add comparison examples
audience.add_compare_example(
    instruction="Which image follows the prompt better?",
    datapoint=["good.jpg", "bad.jpg"],
    truth="good.jpg",
    context="A cat on a chair",
    data_type="media",
    explanation="The first image clearly shows a cat sitting on a chair as described.",  # Shown to labelers who answer incorrectly
)

# Inspect the examples currently on the audience
examples_df = audience.get_examples(amount=10, page=1)
```

**Audience methods:**
- `audience.assign_job(job_definition)` — start a job
- `audience.find_jobs(name="filter", amount=10, page=1)` — find assigned jobs
- `audience.update_filters([...])` — apply demographic filters
- `audience.update_name("New Name")` — rename
- `audience.get_examples(amount=10, page=1)` — list qualification examples (returns DataFrame)
- `audience.delete()` — delete the audience

**Job / Job Definition methods:**
- `job_def.preview()` — open browser preview of what labelers see
- `job_def.update_dataset(datapoints=..., data_type=..., contexts=..., media_contexts=..., private_metadata=...)` — replace the datapoints on an existing job definition
- `job_def.delete()` — delete a job definition and all its revisions
- `job.display_progress_bar(refresh_rate=5)` — blocking progress bar
- `job.get_status()` — current status string
- `job.get_results()` — blocks until Completed/Failed, returns `RapidataResults`
- `job.view()` — open the job's details page in the browser
- `job.delete()` — delete a running job

## Legacy Order API

Still fully supported. Creates and runs tasks in a single step (no separate audience/job definition). Available task types: `create_classification_order`, `create_compare_order`, `create_ranking_order`, `create_free_text_order`, `create_select_words_order`, `create_locate_order`, `create_draw_order`.

```python
order = client.order.create_classification_order(
    name="Image Classification",
    instruction="What's in the image?",
    answer_options=["Cat", "Dog", "Bird"],
    datapoints=["img1.jpg", "img2.jpg"],
    responses_per_datapoint=10,
    validation_set_id="validation_id",
).run()

order.display_progress_bar()
results = order.get_results()
```

**Order methods:** `run(after=None)`, `pause()`, `unpause()`, `delete()`, `get_status()`, `display_progress_bar()`, `get_results(preliminary_results=False)`, `preview()`, `view()`. `run(after=...)` accepts another `RapidataOrder` (or its id) so the new order only starts after the prior one finishes.

**Migration to new API:** Replace `create_classification_order()` / `create_compare_order()` with `create_classification_job_definition()` / `create_compare_job_definition()`, replace validation sets with audience examples, use `audience.assign_job()` instead of `.run()`. Other task types are only available via the order API for now.

## Settings

Settings control how a task is rendered and behaves for labelers. All settings are importable from the top-level `rapidata` package.

```python
from rapidata import (
    NoShuffleSetting, AllowNeitherBothSetting, MarkdownSetting,
    MuteVideoSetting, FreeTextMinimumCharactersSetting, FreeTextMaxCharactersSetting,
    SwapContextInstructionSetting, PlayPercentageVideoSetting,
    OriginalLanguageOnlySetting, NoMistakeOptionSetting, DisableAutoloopSetting,
    NoInstructionDisplaySetting, KeyboardNumericSetting,
    LocateMaxPointsSetting, LocateMinPointsSetting,
    ComparePanoramaSetting, CompareEquirectangularSetting,
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
settings=[CustomSetting(key="my_flag", value="on")]              # Rapid-level flag (default); use target="campaign" for campaign-level
```

**Note on `FreeTextMinimumCharactersSetting` / `FreeTextMaxCharactersSetting`:** use these with caution. Free-text responses already pass through a reasonableness check by default, so tightening the bounds is usually unnecessary and will reject otherwise valid answers. Only set them when the question genuinely demands a specific length (e.g. a single word, or a full paragraph).

## Key Gotchas

1. **Always preview first** — call `.preview()` before assigning to catch instruction issues early
2. **Use `NoShuffleSetting()` for Likert scales** — ordered answer options get shuffled by default
3. **Min 3 audience examples** — custom audiences need at least 3 qualification examples before recruiting
4. **25-second time limit** — labelers have ~25 seconds per task; keep instructions concise
5. **Responses may exceed `responses_per_datapoint`** — concurrent labelers can cause slight overflow
6. **Two early stopping strategies, mutually exclusive** — `confidence_threshold` (statistical, weighted by labeler trust scores) or `quorum_threshold` (stops when N responses agree); cannot use both at once
7. **Early stopping only for unambiguous tasks** — both strategies work best when there's a clear correct answer
8. **Failed uploads don't block job creation** — a `FailedUploadException` is raised but the job is still created with successful datapoints; access the partial object via `e.job_definition` or `e.order`
9. **QR code printed on job/order creation** — when a job definition is created or an order enters preview, a terminal QR code linking to the campaign preview is printed automatically so you can open it on a phone; suppress it with `rapidata_config.logging.silent_mode = True`

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
    time_to_live=300,  # Stop after 5 minutes
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

## Additional Resources

- For complete API reference, all parameters, filters, results format, error handling, flows, and MRI: see [reference.md](reference.md)
- For full end-to-end code examples and common patterns: see [examples.md](examples.md)
