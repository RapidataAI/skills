---
name: rapidata
description: Explains how to use the Rapidata API to get real and fast human annotations for your data. Use when writing code that creates labeling tasks, compares models, collects human feedback, or integrates with the Rapidata Python SDK.
---

# Rapidata Python SDK

Rapidata connects you with distributed human labelers worldwide for fast, high-quality data annotation. The SDK lets you create labeling tasks, manage annotator audiences, and retrieve results programmatically.

## Installation & Authentication

```python
pip install -U rapidata

from rapidata import RapidataClient
client = RapidataClient()  # First use opens browser for login

# Or use credentials directly
client = RapidataClient(client_id="...", client_secret="...")
# Credentials are saved to ~/.config/rapidata/credentials.json
```

## Core Concepts

- **Job Definition**: A reusable configuration template for a labeling task (task type, instruction, datapoints, answer options)
- **Audience**: A group of pre-screened labelers. Can be global (fast, baseline quality), curated (pre-trained on domains like "alignment"), or custom (trained with your qualification examples)
- **Job**: A running instance of a job definition assigned to an audience
- **Flow**: Lightweight continuous ranking without full job setup
- **MRI/Benchmark**: Compare and rank AI models on leaderboards

**Client entry points:**
- `client.job` — create job definitions
- `client.audience` — create and find audiences
- `client.flow` — continuous ranking flows
- `client.mri` — model ranking insights / benchmarks
- `client.order` — legacy order API (still supported)

## New API: Job Definitions + Audiences (Recommended)

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

Order multiple items by preference (uses pairwise comparisons internally).

```python
job_def = client.job.create_ranking_job_definition(
    name="Image Quality Ranking",
    instruction="Rank these images by quality",
    datapoints=[["img1.jpg", "img2.jpg", "img3.jpg"]],
    comparison_budget_per_ranking=50,
    responses_per_comparison=1,
    data_type="media",
    random_comparisons_ratio=0.5,
    contexts=["Optional context"],
)
```

### Custom Audiences

Create an audience trained on your specific task. Minimum 3 examples required before recruiting starts.

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
)

# Add comparison examples
audience.add_compare_example(
    instruction="Which image follows the prompt better?",
    datapoint=["good.jpg", "bad.jpg"],
    truth="good.jpg",
    context="A cat on a chair",
    data_type="media",
)
```

**Audience methods:**
- `audience.assign_job(job_definition)` — start a job
- `audience.find_jobs(name="filter", amount=10)` — find assigned jobs
- `audience.update_filters([...])` — apply demographic filters
- `audience.update_name("New Name")` — rename

## Legacy Order API

Still supported. Creates and runs tasks in a single step (no separate audience/job definition).

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

**Migration to new API:** Replace `create_*_order()` with `create_*_job_definition()`, replace validation sets with audience examples, use `audience.assign_job()` instead of `.run()`.

## Settings

```python
from rapidata import NoShuffleSetting, AllowNeitherBothSetting, Markdown, AlertOnFastResponseSetting, FreeTextMinimumCharactersSetting

settings=[NoShuffleSetting()]                         # Keep answer options in order (use for Likert scales)
settings=[AllowNeitherBothSetting()]                  # Comparison: allow "Neither"/"Both"
settings=[Markdown()]                                 # Render markdown in text
settings=[AlertOnFastResponseSetting()]               # Alert if labeler answers too quickly
settings=[FreeTextMinimumCharactersSetting(min=50)]   # Min text length for free text
```

## Key Gotchas

1. **Always preview first** — call `.preview()` before assigning to catch instruction issues early
2. **Use `NoShuffle()` for Likert scales** — ordered answer options get shuffled by default
3. **Min 3 audience examples** — custom audiences need at least 3 qualification examples before recruiting
4. **25-second time limit** — labelers have ~25 seconds per task; keep instructions concise
5. **Responses may exceed `responses_per_datapoint`** — concurrent labelers can cause slight overflow
6. **Two early stopping strategies, mutually exclusive** — `confidence_threshold` (statistical, weighted by labeler trust scores) or `quorum_threshold` (stops when N responses agree); cannot use both at once
7. **Early stopping only for unambiguous tasks** — both strategies work best when there's a clear correct answer
8. **Failed uploads don't block job creation** — a `FailedUploadException` is raised but the job is still created with successful datapoints

## Additional Resources

- For complete API reference, all parameters, filters, results format, error handling, flows, and MRI: see [reference.md](reference.md)
- For full end-to-end code examples and common patterns: see [examples.md](examples.md)
