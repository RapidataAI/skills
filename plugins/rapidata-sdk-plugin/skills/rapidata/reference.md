# Rapidata SDK — Full API Reference

## Job Definition Parameters

### Common Parameters (all job types)

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | str | Job identifier (not shown to labelers) |
| `instruction` | str | Task description shown to labelers |
| `datapoints` | list | Data to label (URLs or local paths) |
| `data_type` | str | `"media"` (default) or `"text"` |
| `responses_per_datapoint` | int | Responses per item (default 10) |
| `contexts` | list[str] | Text context per datapoint |
| `media_contexts` | list[str] | Reference media per datapoint |
| `confidence_threshold` | float | Confidence-based early stopping threshold (0-1); cannot combine with `quorum_threshold` |
| `quorum_threshold` | int | Quorum-based early stopping: stop when this many responses agree; cannot combine with `confidence_threshold` |
| `settings` | list | Display/behavior settings |
| `private_metadata` | list[dict] | Hidden metadata per datapoint |

### Classification-specific

| Parameter | Type | Description |
|-----------|------|-------------|
| `answer_options` | list[str] | Categories to choose from |

### Comparison-specific

| Parameter | Type | Description |
|-----------|------|-------------|
| `datapoints` | list[list[str]] | Pairs: `[["a1.jpg","b1.jpg"], ...]` |
| `a_b_names` | list[str] | Custom labels for results, e.g. `["Model A","Model B"]` |

### Ranking-specific

| Parameter | Type | Description |
|-----------|------|-------------|
| `datapoints` | list[list[str]] | Groups: `[["img1","img2","img3"], ...]` |
| `comparison_budget_per_ranking` | int | Total comparisons per ranking group |
| `responses_per_comparison` | int | Responses per individual comparison |
| `random_comparisons_ratio` | float | Ratio of random vs targeted comparisons (0-1) |

## Demographic Filters

```python
from rapidata import (
    CountryFilter, LanguageFilter, UserScoreFilter,
    AgeFilter, GenderFilter, DeviceFilter,
    AgeGroup, Gender, DeviceType,
    NotFilter, OrFilter, AndFilter
)

audience.update_filters([
    CountryFilter(include=["US", "CA", "GB"]),
    LanguageFilter(include=["en", "fr"]),
    UserScoreFilter(min=0.5, max=0.99),
    AgeFilter(include=[AgeGroup.AGE_25_34, AgeGroup.AGE_35_44]),
    GenderFilter(include=[Gender.MALE, Gender.FEMALE]),
    DeviceFilter(include=[DeviceType.MOBILE, DeviceType.DESKTOP]),
])

# Combine filters with logic operators
combined = OrFilter([filter1, filter2])
audience.update_filters([NotFilter(combined)])
```

## Results Format

### Classification Results

```json
{
  "results": {
    "globalAggregatedData": { "Cat": 15, "Dog": 8 },
    "data": [
      {
        "originalFileName": "image1.jpg",
        "aggregatedResults": { "Cat": 15, "Dog": 8 },
        "summedUserScores": { "Cat": 9.5, "Dog": 4.2 },
        "confidencePerCategory": { "Cat": 0.989, "Dog": 0.011 },
        "detailedResults": [...]
      }
    ]
  }
}
```

### Comparison Results

```json
{
  "summary": { "A_wins_total": 5, "B_wins_total": 3 },
  "results": [
    {
      "context": "A small blue book...",
      "winner_index": 1,
      "winner": "model_b.jpg",
      "aggregatedResults": { "model_a.jpg": 3, "model_b.jpg": 5 },
      "aggregatedResultsRatios": { "model_a.jpg": 0.375, "model_b.jpg": 0.625 },
      "summedUserScores": { "model_a.jpg": 1.0, "model_b.jpg": 2.5 },
      "summedUserScoresRatios": { "model_a.jpg": 0.286, "model_b.jpg": 0.714 },
      "detailedResults": [
        {
          "votedFor": "model_b.jpg",
          "userDetails": {
            "country": "US", "language": "en",
            "userScores": { "global": 0.75 },
            "age": "25-34", "gender": "Female"
          }
        }
      ]
    }
  ]
}
```

### Key Result Fields

| Field | Meaning |
|-------|---------|
| `winner` | Most-voted option (weighted by userScores) |
| `aggregatedResults` | Raw vote counts |
| `aggregatedResultsRatios` | Vote percentages |
| `summedUserScores` | Total reliability scores per option |
| `summedUserScoresRatios` | Weighted voting proportions |
| `confidencePerCategory` | Confidence level per category (with early stopping) |
| `userScore` | 0-1 value indicating individual labeler reliability |

### Working with Results

```python
results = job.get_results()
df = results.to_pandas()
json_data = results.to_json()
```

## Early Stopping

Two mutually exclusive strategies are available for Classification and Comparison jobs. You cannot set both on the same job.

### Confidence Stopping

Stop collecting responses once a statistical confidence threshold (weighted by labeler trust scores) is reached.

```python
job_def = client.job.create_classification_job_definition(
    name="Animal Classification",
    instruction="What animal is in this image?",
    answer_options=["Cat", "Dog"],
    datapoints=["pet1.jpg", "pet2.jpg"],
    responses_per_datapoint=50,   # Maximum
    confidence_threshold=0.99,    # Stop at 99% confidence
)
```

- System calculates confidence using labeler userScores
- Stops when target confidence is reached
- Saves cost by collecting fewer responses when consensus is clear
- Best for unambiguous tasks with clear correct answers
- Not recommended for subjective preference tasks

### Quorum Stopping

Stop collecting responses once a fixed number of responses agree on the same answer.

```python
job_def = client.job.create_classification_job_definition(
    name="Animal Classification",
    instruction="What animal is in this image?",
    answer_options=["Cat", "Dog"],
    datapoints=["pet1.jpg", "pet2.jpg"],
    responses_per_datapoint=10,   # Maximum
    quorum_threshold=7,           # Stop when 7 responses agree
)
```

A datapoint stops when:
1. `quorum_threshold` responses agree on the same answer, **OR**
2. Quorum becomes mathematically impossible (e.g. votes are too split to reach the threshold), **OR**
3. `responses_per_datapoint` total votes are collected

- Simpler than confidence stopping — based on raw vote counts, not statistics
- Good when you want predictable cost bounds with early termination
- Best for unambiguous tasks with a clear correct answer

## Error Handling

### FailedUploadException

When datapoints fail to upload, the job is still created with the successful ones:

```python
from rapidata.rapidata_client.exceptions import FailedUploadException

try:
    job_def = client.job.create_classification_job_definition(
        name="My Job",
        instruction="...",
        answer_options=[...],
        datapoints=["valid.jpg", "missing.jpg", "valid2.jpg"],
    )
except FailedUploadException as e:
    job_def = e.job_definition          # Partially created object
    print(f"Failed: {len(e.failed_uploads)}")
    for reason, dps in e.failures_by_reason.items():
        print(f"  {reason}: {len(dps)} datapoints")
    # Decide whether to proceed
    if len(e.failed_uploads) <= len(datapoints) * 0.1:
        job = audience.assign_job(job_def)
```

**Properties:** `failed_uploads`, `failures_by_reason`, `detailed_failures`, `job_definition` (or `order` for legacy API).

## Ranking Flows (Continuous Ranking)

Lightweight continuous ranking without full job/audience setup:

```python
# Create flow
flow = client.flow.create_ranking_flow(
    name="Image Quality Ranking",
    instruction="Which image looks better?",
    max_response_threshold=100,   # Target responses per flow item (default 100)
    min_response_threshold=50,    # Minimum acceptable responses; item is Incomplete if TTL expires below this
)

# Add items to rank
flow_item = flow.create_new_flow_batch(
    datapoints=["img1.jpg", "img2.jpg", "img3.jpg"],
    context="Generated by Model X",
    time_to_live=300,  # Stop after 5 minutes
)

# Get results
results = flow_item.get_results()     # Blocks until complete
status = flow_item.get_status()       # Non-blocking check
matrix = flow_item.get_win_loss_matrix()  # Pandas DataFrame
count = flow_item.get_response_count()

# Manage flows
all_flows = client.flow.find_flows(amount=10)
flow = client.flow.get_flow_by_id("flow_id")
flow.delete()
```

## Model Ranking Insights (MRI / Benchmarks)

Compare and rank AI models on leaderboards. Supports images, videos, audio, and text.

```python
# Create benchmark
benchmark = client.mri.create_new_benchmark(
    name="AI Art Competition",
    prompts=["A serene mountain landscape", "A futuristic city"],
)

# Create leaderboard
leaderboard = benchmark.create_leaderboard(
    name="Realism",
    instruction="Which image is more realistic?",
    show_prompt=False,
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
    data_type="media",   # "media" (default) or "text"
)

# Upload additional media to the same participant
participant.upload_media(
    assets=["mountain_v3_extra.png"],
    identifiers=["A serene mountain landscape"],
    data_type="media",   # "media" (default) or "text"
)

# Submit individually or all at once
participant.run()       # Submit one participant
benchmark.run()         # Submit all unsubmitted participants

# List participants and their status
for p in benchmark.participants:
    print(p.name, p.status)

# Get results
standings = leaderboard.get_standings()  # Pandas DataFrame

# Find existing benchmarks
benchmarks = client.mri.find_benchmarks(name="AI Art", amount=10)
benchmark = client.mri.get_benchmark_by_id("benchmark_id")
```

## Configuration

```python
from rapidata import rapidata_config, logger

# Logging
rapidata_config.logging.level = "INFO"       # DEBUG, INFO, WARNING, ERROR, CRITICAL
rapidata_config.logging.log_file = "/path/to/log.txt"
rapidata_config.logging.silent_mode = False
rapidata_config.logging.enable_otlp = True   # OpenTelemetry tracing

# Upload tuning
rapidata_config.upload.maxWorkers = 25        # Concurrent upload threads
rapidata_config.upload.maxRetries = 3
rapidata_config.upload.cacheToDisk = True
rapidata_config.upload.batchSize = 1000

# Clear caches
client.clear_all_caches()
```

All config can also be set via environment variables with `RAPIDATA_` prefix (e.g., `RAPIDATA_maxWorkers=10`).

## Human Prompting Best Practices

- **Be concise** — labelers have ~25 seconds per task
- **Positive framing** — "Which is more realistic?" not "Which is less AI-generated?"
- **Distinct options** — "Poor / Acceptable / Excellent" not "Bad / Not Good / Fine / Good / Great"
- **Single criterion per task** — "What animal is in the image?" not "Does this contain a rabbit, dog, or cat?"
- **Use `NoShuffle()` for scales** — always for Likert or ordered answer options
