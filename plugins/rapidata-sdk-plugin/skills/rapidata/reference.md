# Rapidata SDK — Full API Reference

## Job Definition Parameters

The new job-definition API exposes **classification** and **comparison** publicly. Ranking / locate / free-text / select-words / draw are currently available through the legacy order API (see the "Legacy Order API" section below).

### Common parameters (classification & comparison)

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | str | Job identifier (not shown to labelers) |
| `instruction` | str | Task description shown to labelers |
| `datapoints` | list | Data to label (URLs or local paths) |
| `data_type` | `"media"` \| `"text"` | `"media"` (default, covers image/video/audio) or `"text"` |
| `responses_per_datapoint` | int | Responses per item (default 10) |
| `contexts` | list[str] \| None | Text context per datapoint |
| `media_contexts` | list[str] \| None | Reference media per datapoint |
| `confidence_threshold` | float \| None | Confidence-based early stopping threshold (0-1); cannot combine with `quorum_threshold` |
| `quorum_threshold` | int \| None | Quorum-based early stopping: stop when this many responses agree; cannot combine with `confidence_threshold` |
| `settings` | `Sequence[RapidataSetting] \| None` | Display/behavior settings |
| `private_metadata` | `list[dict[str, str]] \| None` | Hidden metadata per datapoint |

### Classification-specific

| Parameter | Type | Description |
|-----------|------|-------------|
| `answer_options` | list[str] | Categories to choose from |

### Comparison-specific

| Parameter | Type | Description |
|-----------|------|-------------|
| `datapoints` | list[list[str]] | Pairs: `[["a1.jpg","b1.jpg"], ...]` |
| `a_b_names` | list[str] \| None | Custom labels for results, e.g. `["Model A","Model B"]` |

### Ranking (via `client.order.create_ranking_order` or `client.flow.create_ranking_flow`)

| Parameter | Type | Description |
|-----------|------|-------------|
| `datapoints` | list[list[str]] | Groups: `[["img1","img2","img3"], ...]` |
| `comparison_budget_per_ranking` | int | Total comparisons per ranking group |
| `responses_per_comparison` | int | Responses per individual comparison (default 1) |
| `random_comparisons_ratio` | float | Ratio of random vs targeted comparisons (0-1, default 0.5) |

## Demographic Filters

All filters are importable from the top-level `rapidata` package.

```python
from rapidata import (
    CountryFilter, LanguageFilter, UserScoreFilter,
    AgeFilter, GenderFilter, DeviceFilter, CampaignFilter, CustomFilter,
    AgeGroup, Gender, DeviceType,
    NotFilter, OrFilter, AndFilter,
)

audience.update_filters([
    CountryFilter(country_codes=["US", "CA", "GB"]),                  # 2-letter ISO codes (uppercased)
    LanguageFilter(language_codes=["en", "fr"]),                      # 2-letter ISO language codes
    UserScoreFilter(lower_bound=0.5, upper_bound=0.99),               # 0.0–1.0
    AgeFilter(age_groups=[AgeGroup.AGE_25_34, AgeGroup.AGE_35_44]),
    GenderFilter(genders=[Gender.MALE, Gender.FEMALE]),
    DeviceFilter(device_types=[DeviceType.MOBILE, DeviceType.DESKTOP]),
])

# Combine filters with logic operators
combined = OrFilter([filter1, filter2])
audience.update_filters([NotFilter(combined)])
```

### Filter signatures

| Filter | Signature | Works on audiences? |
|--------|-----------|---------------------|
| `CountryFilter` | `(country_codes: list[str])` | yes |
| `LanguageFilter` | `(language_codes: list[str])` | yes |
| `UserScoreFilter` | `(lower_bound: float = 0.0, upper_bound: float = 1.0, dimension: str \| None = None)` | yes |
| `AgeFilter` | `(age_groups: list[AgeGroup])` | orders only |
| `GenderFilter` | `(genders: list[Gender])` | orders only |
| `DeviceFilter` | `(device_types: list[DeviceType])` | orders only |
| `CampaignFilter` | `(campaign_ids: list[str])` | orders only |
| `CustomFilter` | `(identifier: str, values: list[str])` | orders only |
| `NotFilter` | `(filter: RapidataFilter)` | both |
| `OrFilter` | `(filters: list[RapidataFilter])` | both |
| `AndFilter` | `(filters: list[RapidataFilter])` | both |

Note: `AgeFilter`, `GenderFilter`, `DeviceFilter`, `CampaignFilter`, and `CustomFilter` cannot currently be attached to audiences with `audience.update_filters(...)` — use them as `filters=[...]` on `client.order.create_*_order(...)` instead.

## Selections (Order API only)

Selections control which rapids (validation, labeling, demographic) are presented during a session. Pass them to `client.order.create_*_order(..., selections=[...])`.

```python
from rapidata import (
    LabelingSelection, ValidationSelection, ConditionalValidationSelection,
    DemographicSelection, CappedSelection, ShufflingSelection, EffortSelection,
    RapidataRetrievalMode,
)

LabelingSelection(amount=5)                                        # 5 labeling rapids per session
LabelingSelection(amount=5, retrieval_mode=RapidataRetrievalMode.Sequential)
ValidationSelection(validation_set_id="...", amount=1)
ConditionalValidationSelection(
    validation_set_id="...",
    thresholds=[0.5, 0.8],
    chances=[1.0, 0.2],
    rapid_counts=[2, 1],
    dimensions=["global"],          # `dimension` (singular) is deprecated
)
DemographicSelection(keys=["age", "gender"], max_rapids=1)
CappedSelection(selections=[...], max_rapids=10)
ShufflingSelection(selections=[...])
EffortSelection(effort_budget=60)    # seconds of effort per session
```

`RapidataRetrievalMode` options: `Shuffled` (default), `Sequential`, `Random`.

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

Flow items have a different result shape — see the Flows section.

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

## Settings Reference

All settings inherit from `RapidataSetting` and are importable from `rapidata`.

| Class | Constructor | Effect |
|-------|-------------|--------|
| `NoShuffleSetting` | `(value: bool = True)` | Disable shuffling of answer options (Likert scales) |
| `AllowNeitherBothSetting` | `(value: bool = True)` | Comparison: allow "Neither" / "Both" answers |
| `MarkdownSetting` | `(value: bool = True)` | Render markdown in text |
| `MuteVideoSetting` | `(value: bool = True)` | Start videos muted |
| `FreeTextMinimumCharactersSetting` | `(value: int)` — must be ≥ 1 | Min chars for free-text tasks. Use with caution — see note below the table |
| `FreeTextMaxCharactersSetting` | `(value: int = 1024)` — must be ≥ 1 | Max chars for free-text tasks. Use with caution — see note below the table |
| `SwapContextInstructionSetting` | `(value: bool = True)` | Swap positions of context and instruction |
| `PlayPercentageVideoSetting` | `(percentage: int = 95)` — 0–95 | Require labelers to watch N% of video |
| `OriginalLanguageOnlySetting` | `(value: bool = True)` | Skip translation; show task in original language |
| `NoMistakeOptionSetting` | `(value: bool = True)` | Hide the "mark as mistake" option |
| `DisableAutoloopSetting` | `(value: bool = True)` | Disable automatic media looping |
| `NoInstructionDisplaySetting` | `(value: bool = True)` | Hide instruction from task screen |
| `KeyboardNumericSetting` | `(value: bool = True)` | Open numeric keyboard on mobile |
| `LocateMaxPointsSetting` | `(value: int = 3)` — ≥ 1 | Locate tasks: max points per labeler |
| `LocateMinPointsSetting` | `(value: int)` | Locate tasks: min points per labeler |
| `ComparePanoramaSetting` | `(value: bool = True)` | Render comparison media as 360° panorama |
| `CompareEquirectangularSetting` | `(value: bool = True)` | Render comparison media as equirectangular VR |
| `CustomSetting` | `(key: str, value: str, target: "rapids" \| "campaign" = "rapids")` | Pass a custom key/value through to the backend; `target` controls whether the flag is applied at the rapid level (`"rapids"`) or campaign level (`"campaign"`) |

**Note on `FreeTextMinimumCharactersSetting` / `FreeTextMaxCharactersSetting`:** use these with caution. Free-text responses already pass through a reasonableness check by default, so tightening the bounds is usually unnecessary and will reject otherwise valid answers. Only set them when the question genuinely demands a specific length (e.g. a single word, or a full paragraph).

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
    job_def = e.job_definition          # Partially created object (or e.order for the legacy API)
    print(f"Failed: {len(e.failed_uploads)}")
    for reason, dps in e.failures_by_reason.items():
        print(f"  {reason}: {len(dps)} datapoints")
    for fu in e.detailed_failures:
        print(fu.item, fu.error_message, fu.error_type)
    # Decide whether to proceed
    if len(e.failed_uploads) <= len(datapoints) * 0.1:
        job = audience.assign_job(job_def)
```

**Properties:** `failed_uploads` (list[Datapoint] — backward-compatible), `detailed_failures` (list[FailedUpload[Datapoint]]), `failures_by_reason` (dict[str, list[Datapoint]]), `job_definition`, `order`, `dataset`.

**Recovery docs:** https://docs.rapidata.ai/3.x/error_handling/

## Ranking Flows (Continuous Ranking)

Lightweight continuous ranking without full job/audience setup:

```python
# Create flow
flow = client.flow.create_ranking_flow(
    name="Image Quality Ranking",
    instruction="Which image looks better?",
    max_response_threshold=100,       # Target responses per flow item (default 100)
    min_response_threshold=50,        # Minimum acceptable responses; item is Incomplete if TTL expires below this
    # validation_set_id="...",        # Optional: validation-set id to interleave validation rapids
    # settings=[...],                 # Optional: flow-wide RapidataSettings
)

# Add items to rank
flow_item = flow.create_new_flow_batch(
    datapoints=["img1.jpg", "img2.jpg", "img3.jpg"],
    context="Generated by Model X",
    data_type="media",                # "media" (default) or "text"
    private_metadata=[...],           # Optional
    accept_failed_uploads=False,      # If True, proceed even if some uploads fail
    time_to_live=300,                 # Stop after N seconds
)

# Get results — flow items return FlowItemResult, NOT RapidataResults
result = flow_item.get_results()      # Blocks until completed/failed/stopped/incomplete
# result.datapoints: dict[str, int]   # asset → ELO
# result.total_votes: int

status = flow_item.get_status()       # Non-blocking check
matrix = flow_item.get_win_loss_matrix()  # Pandas DataFrame (blocks until completed)
count = flow_item.get_response_count()

# Query flow items
items = flow.get_flow_items(amount=10, page=1)

# Update a flow after creation
flow.update_config(
    instruction="New instruction",
    starting_elo=1000,
    min_responses=40,
    max_responses=120,
)

# Preheat for low-latency responses (call ~5 minutes before time-sensitive batches)
client.flow.preheat()

# Manage flows
all_flows = client.flow.find_flows(name="", amount=10, page=1)
flow = client.flow.get_flow_by_id("flow_id")
flow.delete()

# Delete other resources
job_def.delete()   # Deletes the job definition and all its revisions
job.delete()       # Deletes a running job
audience.delete()  # Deletes the audience
```

## Model Ranking Insights (MRI / Benchmarks)

Compare and rank AI models on leaderboards. Supports images, videos, audio, and text.

```python
# Create benchmark
benchmark = client.mri.create_new_benchmark(
    name="AI Art Competition",
    prompts=["A serene mountain landscape", "A futuristic city"],
    # identifiers=[...],        # Optional: stable ids for each prompt
    # prompt_assets=[...],      # Optional: reference media for each prompt
    # tags=[...],               # Optional: tags applied to the benchmark
)

# Add a prompt later if needed
benchmark.add_prompt(prompt="A quiet lake at dawn")

# Create leaderboard
leaderboard = benchmark.create_leaderboard(
    name="Realism",
    instruction="Which image is more realistic?",
    show_prompt=False,
    show_prompt_asset=False,
    inverse_ranking=False,
    # level_of_detail="high",            # "debug" | "low" | "medium" | "high" | "very high"
    # min_responses_per_matchup=5,
    # audience_id="...",                 # Optional: restrict to a specific audience
    # settings=[...],
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
participant.upload_media(
    assets=["mountain_v3_extra.png"],
    identifiers=["A serene mountain landscape"],
    data_type="media",
)

# Submit individually or all at once
participant.run()       # Submit one participant
benchmark.run()         # Submit all unsubmitted (CREATED) participants

# List participants and their status
for p in benchmark.participants:
    print(p.name, p.status)

# Get results
standings = leaderboard.get_standings()                    # Pandas DataFrame for one leaderboard
overall = benchmark.get_overall_standings(tags=None)       # Aggregated ELO across all leaderboards
matrix_lb = leaderboard.get_win_loss_matrix()              # Pairwise wins/losses for one leaderboard
matrix_bm = benchmark.get_win_loss_matrix(                 # Pairwise wins/losses across leaderboards
    tags=None, participant_ids=None, leaderboard_ids=None, use_weighted_scoring=None,
)

# Update leaderboard config live
leaderboard.name = "Realism (Updated)"
leaderboard.level_of_detail = "very high"
leaderboard.min_responses_per_matchup = 7

# Open in browser
benchmark.view()
leaderboard.view()

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
rapidata_config.logging.silent_mode = False  # also suppresses terminal QR codes printed on job/order creation
rapidata_config.logging.enable_otlp = True   # OpenTelemetry tracing

# Upload tuning
rapidata_config.upload.maxWorkers = 25        # Concurrent upload threads (warns above 200)
rapidata_config.upload.maxRetries = 3
rapidata_config.upload.cacheToDisk = True
rapidata_config.upload.cacheTimeout = 1.0
rapidata_config.upload.batchSize = 1000       # URLs per batch (100–5000)
rapidata_config.upload.batchPollInterval = 0.5

# Client-level maintenance
client.clear_all_caches()
client.reset_credentials()
```

`cacheLocation` (`~/.cache/rapidata/upload_cache`) and `cacheShards` (128) are immutable — don't try to assign them.

All config fields support environment-variable overrides with the `RAPIDATA_` prefix (e.g., `RAPIDATA_maxWorkers=10`, `RAPIDATA_DISABLE_OTLP=1`).

**Client authentication** is also resolved from environment variables: `RAPIDATA_CLIENT_ID` and `RAPIDATA_CLIENT_SECRET` are used before falling back to `~/.config/rapidata/credentials.json` and browser login. `RAPIDATA_ENVIRONMENT` overrides the API endpoint (default: `rapidata.ai`). Empty values are treated as unset and fall through to the next resolution layer.

## Human Prompting Best Practices

- **Be concise** — labelers have ~25 seconds per task
- **Positive framing** — "Which is more realistic?" not "Which is less AI-generated?"
- **Distinct options** — "Poor / Acceptable / Excellent" not "Bad / Not Good / Fine / Good / Great"
- **Single criterion per task** — "What animal is in the image?" not "Does this contain a rabbit, dog, or cat?"
- **Use `NoShuffleSetting()` for scales** — always for Likert or ordered answer options
