# Rapidata SDK — Code Examples

## Simple Classification

```python
from rapidata import RapidataClient

client = RapidataClient()
audience = client.audience.find_audiences("alignment")[0]

job_def = client.job.create_classification_job_definition(
    name="Image Classification",
    instruction="What's in this image?",
    answer_options=["Cat", "Dog", "Bird"],
    datapoints=["img1.jpg", "img2.jpg"],
)
job_def.preview()
job = audience.assign_job(job_def)
job.display_progress_bar()
results = job.get_results()
df = results.to_pandas()
```

## Classification with Likert Scale

```python
from rapidata import RapidataClient, NoShuffle

client = RapidataClient()
audience = client.audience.find_audiences("alignment")[0]

job_def = client.job.create_classification_job_definition(
    name="Quality Rating",
    instruction="How well does the video match the description?",
    answer_options=["1: Poor", "2: Fair", "3: Good", "4: Excellent"],
    datapoints=["video1.mp4", "video2.mp4"],
    contexts=["A cat playing piano", "A sunset over the ocean"],
    responses_per_datapoint=15,
    settings=[NoShuffle()],  # Critical for ordered scales
)
job = audience.assign_job(job_def)
```

## Comparison with Early Stopping

```python
from rapidata import RapidataClient

client = RapidataClient()
audience = client.audience.find_audiences("alignment")[0]

job_def = client.job.create_compare_job_definition(
    name="Model Comparison",
    instruction="Which image follows the prompt better?",
    datapoints=[
        ["flux_cat.jpg", "mj_cat.jpg"],
        ["flux_sunset.jpg", "mj_sunset.jpg"],
    ],
    contexts=["A cat on a chair", "A sunset over mountains"],
    responses_per_datapoint=50,
    confidence_threshold=0.99,
    a_b_names=["Flux", "Midjourney"],
)
job_def.preview()
job = audience.assign_job(job_def)
job.display_progress_bar()
results = job.get_results()
```

## Comparison Allowing "Neither" / "Both"

```python
from rapidata import RapidataClient, AllowNeitherBoth

client = RapidataClient()
audience = client.audience.find_audiences("alignment")[0]

job_def = client.job.create_compare_job_definition(
    name="Text Comparison",
    instruction="Which response is more helpful?",
    datapoints=[["response_a.txt", "response_b.txt"]],
    data_type="text",
    settings=[AllowNeitherBoth()],
)
```

## Custom Audience with Full Workflow

```python
from rapidata import RapidataClient

client = RapidataClient()

# Create and train audience (need min 3 examples)
audience = client.audience.create_audience(name="Image Quality Experts")

audience.add_compare_example(
    instruction="Which image follows the prompt better?",
    datapoint=["good_example.jpg", "bad_example.jpg"],
    truth="good_example.jpg",
    context="A cat on a chair",
    data_type="media",
)
audience.add_compare_example(
    instruction="Which image follows the prompt better?",
    datapoint=["clear.jpg", "blurry.jpg"],
    truth="clear.jpg",
    context="A sunset over mountains",
    data_type="media",
)
audience.add_compare_example(
    instruction="Which image follows the prompt better?",
    datapoint=["accurate.jpg", "inaccurate.jpg"],
    truth="accurate.jpg",
    context="A red sports car",
    data_type="media",
)

# Create and run job
job_def = client.job.create_compare_job_definition(
    name="Production Comparison",
    instruction="Which image follows the prompt better?",
    datapoints=[
        ["model_a_1.jpg", "model_b_1.jpg"],
        ["model_a_2.jpg", "model_b_2.jpg"],
    ],
    contexts=["A wizard casting a spell", "A futuristic city"],
    responses_per_datapoint=20,
    a_b_names=["Model A", "Model B"],
)
job_def.preview()
job = audience.assign_job(job_def)
job.display_progress_bar()
results = job.get_results()
```

## Audience with Demographic Filters

```python
from rapidata import (
    RapidataClient, CountryFilter, LanguageFilter,
    UserScoreFilter, AgeFilter, AgeGroup
)

client = RapidataClient()
audience = client.audience.create_audience(name="US English Evaluators")
audience.update_filters([
    CountryFilter(include=["US", "CA"]),
    LanguageFilter(include=["en"]),
    UserScoreFilter(min=0.5),
    AgeFilter(include=[AgeGroup.AGE_25_34, AgeGroup.AGE_35_44]),
])
# Add examples, then assign jobs as normal
```

## Ranking

```python
from rapidata import RapidataClient

client = RapidataClient()
audience = client.audience.find_audiences("alignment")[0]

job_def = client.job.create_ranking_job_definition(
    name="Image Quality Ranking",
    instruction="Rank these images by visual quality",
    datapoints=[
        ["img1.jpg", "img2.jpg", "img3.jpg", "img4.jpg"],
    ],
    comparison_budget_per_ranking=50,
    responses_per_comparison=1,
    random_comparisons_ratio=0.5,
    contexts=["A photorealistic landscape"],
)
job = audience.assign_job(job_def)
job.display_progress_bar()
results = job.get_results()
```

## Continuous Ranking Flow

```python
from rapidata import RapidataClient

client = RapidataClient()

flow = client.flow.create_ranking_flow(
    name="Ongoing Quality Ranking",
    instruction="Which image looks better?",
)

# Submit batches over time
batch1 = flow.create_new_flow_batch(
    datapoints=["gen1.jpg", "gen2.jpg", "gen3.jpg"],
    context="Generated from prompt A",
    time_to_live=300,
)

batch1.display_progress_bar()
results = batch1.get_results()
matrix = batch1.get_win_loss_matrix()

# Later, submit more batches to the same flow
batch2 = flow.create_new_flow_batch(
    datapoints=["gen4.jpg", "gen5.jpg", "gen6.jpg"],
    context="Generated from prompt B",
    time_to_live=300,
)
```

## Model Benchmark (MRI)

```python
from rapidata import RapidataClient

client = RapidataClient()

benchmark = client.mri.create_new_benchmark(
    name="Text-to-Image Benchmark",
    prompts=[
        "A serene mountain landscape",
        "A futuristic city at night",
        "A wise wizard portrait",
    ],
)

leaderboard = benchmark.create_leaderboard(
    name="Prompt Adherence",
    instruction="Which image matches the description better?",
    show_prompt=True,
)

# Evaluate models (creates, uploads, and submits in one step)
benchmark.evaluate_model(
    name="DALL-E 3",
    media=["dalle_mountain.png", "dalle_city.png", "dalle_wizard.png"],
    prompts=["A serene mountain landscape", "A futuristic city at night", "A wise wizard portrait"],
)

benchmark.evaluate_model(
    name="Midjourney v6",
    media=["mj_mountain.png", "mj_city.png", "mj_wizard.png"],
    prompts=["A serene mountain landscape", "A futuristic city at night", "A wise wizard portrait"],
)

# Get leaderboard standings
standings = leaderboard.get_standings()
print(standings)
```

## Model Benchmark — Staged Submission

```python
from rapidata import RapidataClient

client = RapidataClient()

benchmark = client.mri.get_benchmark_by_id("benchmark_id")

# Add models without submitting
benchmark.add_model(
    name="DALL-E 3",
    media=["dalle_mountain.png", "dalle_city.png", "dalle_wizard.png"],
    prompts=["A serene mountain landscape", "A futuristic city at night", "A wise wizard portrait"],
)

benchmark.add_model(
    name="Midjourney v6",
    media=["mj_mountain.png", "mj_city.png", "mj_wizard.png"],
    prompts=["A serene mountain landscape", "A futuristic city at night", "A wise wizard portrait"],
)

# Inspect participants before submitting
for p in benchmark.participants:
    print(p.name, p.status)

# Submit all at once
benchmark.run()
```

## Handling Failed Uploads

```python
from rapidata import RapidataClient
from rapidata.rapidata_client.exceptions import FailedUploadException

client = RapidataClient()
audience = client.audience.find_audiences("alignment")[0]

datapoints = ["valid1.jpg", "broken_url", "valid2.jpg", "missing.jpg"]

try:
    job_def = client.job.create_classification_job_definition(
        name="With Failures",
        instruction="What's in this image?",
        answer_options=["Cat", "Dog"],
        datapoints=datapoints,
    )
except FailedUploadException as e:
    job_def = e.job_definition
    print(f"{len(e.failed_uploads)} of {len(datapoints)} failed to upload")
    for reason, failed in e.failures_by_reason.items():
        print(f"  {reason}: {len(failed)}")

    # Proceed if failure rate is acceptable
    if len(e.failed_uploads) / len(datapoints) < 0.1:
        job = audience.assign_job(job_def)
    else:
        print("Too many failures, aborting")
```

## Legacy Order API

```python
from rapidata import RapidataClient

client = RapidataClient()

# Classification order
order = client.order.create_classification_order(
    name="Image Classification",
    instruction="What's in the image?",
    answer_options=["Cat", "Dog", "Bird"],
    datapoints=["img1.jpg", "img2.jpg"],
    responses_per_datapoint=10,
).run()

order.display_progress_bar()
results = order.get_results()

# Comparison order
order = client.order.create_compare_order(
    name="Image Comparison",
    instruction="Which is better?",
    datapoints=[["a.jpg", "b.jpg"]],
    responses_per_datapoint=10,
).run()
```
