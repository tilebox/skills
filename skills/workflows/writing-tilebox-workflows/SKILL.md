---
name: writing-tilebox-workflows
description: "Guides writing Python Tilebox workflow code: task classes, task graphs, dataset queries, storage clients, caches, progress labels, logging, tracing, retries, dependencies, runners, and job submission. Use when creating or modifying Tilebox workflow source code."
license: MIT
compatibility: Requires the Tilebox Python SDK packages used by the workflow, commonly tilebox-workflows, tilebox-datasets, and tilebox-storage.
metadata:
  author: tilebox
---

# Writing Tilebox Workflows

Use this skill when creating or modifying Python Tilebox workflow code. Keep the scope to workflow source code and local/runtime iteration.

## Refresh Current APIs First

When encountering errors that could be due to unclear, or outdated remembered APIs, check the current docs or local package version for the exact API surface you are using:

For example:

```bash
tilebox docs search "Task ExecutionContext submit_subtasks"
tilebox docs search "logging tracing context.logger context.tracer"
tilebox docs search "caches job_cache"
```

Use these companion skills when the task crosses into operations:

- `using-tilebox-cli` for CLI discovery, authentication, JSON output, and docs search.
- `managing-tilebox-jobs` for submitting, listing, waiting on, debugging, retrying, or canceling jobs.
- `managing-tilebox-datasets` for dataset schema inspection and CLI datapoint queries.
- `working-with-tilebox-automations` for cron or storage-triggered workflow automations.

## Start With A Small Architecture Plan

For non-trivial workflows, sketch the task graph before coding:

1. Identify the root task and each worker/aggregation stage.
2. Choose the fanout axis: time windows, scenes/granules, AOIs, chunks, or products.
3. Mark real barriers with `depends_on`; avoid unnecessary sequential chains.
4. Decide what data is passed as task inputs versus stored in `context.job_cache` or external object storage.
5. Choose retry counts for network, storage, or provider operations.

Prefer this shape for scalable workflows:

```diagram
╭──────────────╮
│ Root/Stage   │
│ orchestrator │
╰──────┬───────╯
       │ submit_subtasks([...])
       ▼
╭────────╮  ╭────────╮  ╭────────╮
│Worker  │  │Worker  │  │Worker  │
╰───┬────╯  ╰───┬────╯  ╰───┬────╯
    ╰───────────┼───────────╯
                ▼ depends_on=worker_handles
          ╭────────────╮
          │ Aggregator │
          ╰────────────╯
```

## Define Tasks As Typed Python Classes

Inherit from `Task`; task fields are serializable input parameters. `Task` automatically applies dataclass behavior.

```python
from tilebox.workflows import ExecutionContext, Task


class ProcessScene(Task):
    scene_id: str
    cloud_threshold: float = 20.0

    @staticmethod
    def identifier() -> tuple[str, str]:
        return "tilebox.com/example/ProcessScene", "v1.0"

    def execute(self, context: ExecutionContext) -> None:
        context.current_task.display = f"ProcessScene({self.scene_id})"
        context.logger.info(
            "Started scene processing",
            scene_id=self.scene_id,
            cloud_threshold=self.cloud_threshold,
        )
```

Task identifier rules:

- Default identifier is the class name with version `v0.0`; fine for prototypes.
- For stable workflows, define `identifier()` as a `staticmethod` or `classmethod`.
- Return `(name, version)`, where version matches `vX.Y`.
- Keep the major version compatible for existing jobs; bump the major version for breaking input/behavior changes.
- Minor versions are forward-compatible: a runner with `v1.5` can execute a task submitted as `v1.3`, but not the reverse.

Input design:

- Keep inputs compact: IDs, time windows, AOI bounds, chunk coordinates, small config values, cache keys, and object prefixes.
- Do not pass large arrays, manifests, dataframes, xarray datasets, binary data, or thousands of URLs as task parameters.
- Pass source identifiers or object-store locations, not local file paths between tasks.
- Use typed fields and defaults instead of unpacking unstructured dictionaries unless the payload is naturally dynamic.

## Submit Subtasks, Dependencies, Optional Work, And Retries

Use `ExecutionContext` from inside `execute()` to build the job graph dynamically.

```python
class ProcessScenes(Task):
    scene_ids: list[str]

    def execute(self, context: ExecutionContext) -> None:
        context.current_task.display = f"ProcessScenes(n={len(self.scene_ids)})"

        workers = context.submit_subtasks(
            [ProcessScene(scene_id) for scene_id in self.scene_ids],
            max_retries=3,
        )
        context.submit_subtask(PublishSummary(), depends_on=workers)
```

Patterns:

- Use `context.submit_subtask(task)` for one child task.
- Use `context.submit_subtasks([...])` for homogeneous batches; it returns handles you can pass to `depends_on`.
- `depends_on` takes a list of submitted task handles and waits for successful completion.
- Use `optional=True` for non-critical branches whose failure should not fail the whole job.
- Use `max_retries` for flaky network, object storage, and provider API calls.
- Keep dependency shapes simple. Prefer stage-level barriers over wiring thousands of pairwise dependencies.

Avoid fine-grained DAGs that create many unique dependency shapes, such as long chains or `B[i]` depending only on `A[i]` for thousands of `i`. If the fanout is large, use orchestrator/stage tasks that submit homogeneous batches and stage barriers.

## Design Tasks To Be Retryable

Tasks can be retried after failures, including after a bug fix has been released to the same cluster. Write task execution to be re-entrant when practical so a failed large workflow can resume from failed tasks instead of requiring a fresh job from the beginning.

Retryable task rules:

- Keep side effects safe if the same task input is executed more than once.
- Write outputs to deterministic keys or paths derived from task inputs.
- Prefer overwrite-safe writes, existing-output checks, or atomic commit/rename patterns over append-only side effects.
- Do not emit duplicate external records, notifications, or database rows unless the sink has an idempotency key or deduplication strategy.
- Keep task input schemas stable. Retrying an old failed job uses the originally submitted task inputs.
- For backward-compatible bug fixes, keep the task identifier unchanged and bump only the minor version so newer runners can execute older submitted tasks.
- Bump the major version for breaking input or behavior changes; do not expect a major-version change to repair already-submitted tasks.

This retry-and-resume pattern assumes the task input parameters remain compatible and the workflow shape/dependency graph expected by the failed job has not changed drastically.

## Add Progress Labels

Set `context.current_task.display` to a concise human-readable label. This label appears in job visualization and makes large graphs easier to debug.

```python
class ComputeChunk(Task):
    product_id: str
    x0: int
    x1: int
    y0: int
    y1: int

    def execute(self, context: ExecutionContext) -> None:
        context.current_task.display = f"Chunk[{self.x0}:{self.x1},{self.y0}:{self.y1}]"
        # compute the chunk
```

Good labels include the runtime dimension that distinguishes tasks:

- `DownloadImages(n=24)`
- `DownloadImage('S2A_001')`
- `LocalStats[0:2048,0:2048]`
- `CombineStats n_pixels=12345678`

Set the label after computing useful values, but before expensive work starts.

## Track Progress For Meaningful Fanout

Use Tilebox progress indicators when a task submits a list of subtasks large enough that users benefit from knowing how many units are done. Progress indicators use a `done` / `total` model: `context.progress("name").add(n)` increases the total work, and `context.progress("name").done(n)` increases completed work. The job's progress is the sum of task-level progress reports, and Tilebox avoids double-counting retried tasks by only considering the last execution of a task.

For fanout workflows, use this rule of thumb:

- Call `progress("name").add(n)` in the task that submits `n` subtasks.
- Call `progress("name").done(1)` in each subtask after its represented unit of work completed successfully, usually at the end of `execute()`.
- Make the added total and completed count match: if the parent adds `n`, exactly `n` successful subtasks should collectively call `done(1)` for that indicator.
- Use named indicators for distinct stages such as `download`, `process`, `upload`, or `finalize`.
- Do not call `progress("name").add(n)` and `progress("name").done(n)` in the same task for subtask completion progress; that advances total and done together, so the indicator never shows useful in-progress state.

Example fanout progress pattern:

```python
class ProcessScenes(Task):
    scene_ids: list[str]

    def execute(self, context: ExecutionContext) -> None:
        context.progress("process-scenes").add(len(self.scene_ids))
        context.submit_subtasks([
            ProcessScene(scene_id) for scene_id in self.scene_ids
        ])


class ProcessScene(Task):
    scene_id: str

    def execute(self, context: ExecutionContext) -> None:
        # process this scene successfully first
        context.progress("process-scenes").done(1)
```

## Use Structured Logs And Custom Spans

Tilebox automatically correlates task logs with job, task, runner, trace, and span metadata. Log through `context.logger` inside tasks.

```python
class PublishOutput(Task):
    output_key: str

    def execute(self, context: ExecutionContext) -> None:
        log = context.logger.bind(output_key=self.output_key)
        log.info("Publishing output")

        try:
            with context.tracer.span("publish-output") as span:
                span.set_attribute("output_key", self.output_key)
                # upload or publish data
                log.info("Output published", format="cog")
        except Exception as error:
            log.exception("Output publication failed")
            raise
```

Logging rules:

- Prefer structured fields (`scene_id=...`, `chunk=...`) over string-only messages.
- Use `logger.bind(...)` for attributes shared by several records in one task.
- Use `logger.exception(...)` inside `except` blocks, then re-raise.
- Use `context.tracer.span("name")` around expensive or failure-prone phases such as download, compute, and publish.
- Record attributes on spans for dimensions you will filter by later.

For local development, configure console logging in the runner entrypoint, not inside task classes:

```python
import logging

from tilebox.workflows import Client
from tilebox.workflows.observability.logging import configure_console_logging

configure_console_logging(level=logging.DEBUG)

client = Client(name="example-runner")
client.configure_logging(level=logging.DEBUG, runner_level=logging.INFO)
runner = client.runner(tasks=[ProcessScenes, ProcessScene, PublishSummary])
runner.run_forever()
```

## Query Datasets Deliberately

For dataset-driven workflows, inspect the dataset and collections before coding against fields:

```bash
tilebox dataset get <dataset-slug> --json
tilebox dataset query <dataset-slug> --collections <collection> --last 7d --limit 5
```

The field names in `tilebox dataset query` output and dataset schemas correspond to variables/coordinates returned on the Python `xarray.Dataset`. Use the CLI for quick schema and sample-data inspection, then write Python code against those names.

Python query pattern:

```python
import xarray as xr
from shapely import Polygon
from tilebox.datasets import Client as DatasetClient
from tilebox.datasets.data import TimeInterval


def load_sentinel2(aoi: Polygon, start: str, end: str) -> xr.Dataset:
    dataset = DatasetClient().dataset("open_data.copernicus.sentinel2_msi")
    interval = TimeInterval(start=start, end=end)

    return dataset.query(
        collections=["S2A_S2MSI2A", "S2B_S2MSI2A", "S2C_S2MSI2A"],
        temporal_extent=interval,
        spatial_extent=aoi,
        show_progress=True,
    )
```

Dataset rules:

- Prefer `dataset.query(collections=[...])` when querying multiple collections at once. If `collections` is omitted, all collections in the dataset are queried.
- Scope queries with explicit collection names, IDs, or objects when the workflow expects specific products; do not rely on positional collection ordering.
- Use Shapely geometries (`Polygon`, `MultiPolygon`) for `spatial_extent`, not bbox tuples.
- Use `skip_data=True` only for fast probes; it omits many fields required for downstream processing.
- Do not hardcode assumptions about `location` or provider path formats. Inspect schema examples and sample datapoints.

## Choose Storage Access Based On Data Format

Tilebox datasets index metadata; they usually do not host open-data product bytes. Prefer Tilebox storage clients when they cover the provider and the task needs whole files or provider-specific path/auth behavior.

Use storage clients for:

- Whole-file products such as JP2, classic GeoTIFF, HDF5, NetCDF, and product directories.
- Provider-specific auth, requester-pays, path normalization, quicklooks, caching, or listings.
- Workflows that know exact assets and can download only needed bands/QA files.

Use cloud-native reads directly for COG, Zarr, or cloud-optimized NetCDF when partial spatial/temporal reads materially reduce bytes transferred.

Example storage-client pattern:

```python
from pathlib import Path

from tilebox.storage import CopernicusStorageClient


storage = CopernicusStorageClient(
    access_key,
    secret_access_key,
    Path("s2-data"),
)
storage.download(scene_datapoint, show_progress=True)
```

Keep downloads inside the task that consumes the files. Do not pass downloaded local paths to later tasks; pass product IDs or object-store keys instead.

## Use Cache And External Storage For Shared State

`context.job_cache` is a job-scoped key-value store shared by tasks in one job. Values are bytes.

```python
import pickle


class LoadMetadata(Task):
    def execute(self, context: ExecutionContext) -> None:
        metadata = ...
        context.job_cache["metadata"] = pickle.dumps(metadata)


class SelectProducts(Task):
    def execute(self, context: ExecutionContext) -> None:
        metadata = pickle.loads(context.job_cache["metadata"])
        products = select_products(metadata)
        context.job_cache["products"] = "\n".join(products).encode()
```

Cache rules:

- Use `job_cache` for compact intermediate data shared within one job.
- Prefix keys by product, stage, or task when multiple branches write similar values.
- Store large manifests or large intermediates in object storage and pass a small key/prefix to tasks.
- Treat local filesystem caches as development/local-runner state unless the runner environment guarantees shared access.
- Do not commit large model weights or static runtime artifacts into workflow source, pass them through task inputs, or store them in `job_cache`.
- For reusable runner-local assets such as ML checkpoints, fetch lazily into a deterministic cache path under `~/.cache/tilebox/...`, validate before use, and redownload if incomplete or corrupt.
- For private runtime assets, lazy-load from a private bucket that deployed runners can access; if none exists, ask the user to set one up first.
- Cache expensive in-memory objects such as loaded models with `functools.lru_cache` when safe, but keep the workflow correct on a cold runner with an empty cache.

Runner cache examples:

```python
from tilebox.workflows.cache import LocalFileSystemCache

runner = client.runner(tasks=[ProcessScenes, ProcessScene], cache=LocalFileSystemCache())
```

## Run And Submit For Iteration

Runner entrypoint pattern:

```python
from tilebox.workflows import Client

from my_workflow import ProcessScene, ProcessScenes, PublishSummary


client = Client(name="example-runner")
runner = client.runner(tasks=[ProcessScenes, ProcessScene, PublishSummary])
runner.run_forever()
```

Use `runner.run_all()` for notebooks or scripts that should drain currently available work and return. Use `runner.run_forever()` for long-running runner processes.

Python job submission pattern:

```python
from tilebox.workflows import Client

job = Client().jobs().submit(
    "process-scenes",
    ProcessScenes(scene_ids=["S2A_001", "S2B_002"]),
    max_retries=1,
)
print(job.id)
```

For CLI submission, use the `managing-tilebox-jobs` skill so the payload matches Python task serialization rules.

## Verification Checklist

Before considering workflow-code changes complete:

1. Ensure every task class used by submitted jobs is registered with the runner.
2. Ensure task identifiers and versions match between submitter and runner.
3. Check task inputs are serializable and compact.
4. Check large or cross-task data uses `job_cache` or object storage instead of task arguments.
5. If the workflow uses large runtime artifacts such as model weights, ensure they are fetched lazily into a runner-local cache and excluded from the release artifact.
6. If the task may be retried after a fix, confirm execution is re-entrant/idempotent and the task input schema remains compatible with existing failed jobs.
7. Add `current_task.display` labels for high-fanout tasks.
8. Add progress indicators for sizable fanout where the total and completed subtask counts are useful to users.
9. Add structured logs for start, selected counts, skipped/empty cases, and output locations.
10. Add custom spans around expensive I/O, compute, and publish phases when debugging or performance matters.
11. Run the narrowest local check available: unit tests for pure helpers, import/type checks for task modules, or a small submitted job against a known runner.

## Reference Patterns From Examples

The public `github.com/tilebox/examples` workflows demonstrate these proven patterns:

- Hello-world workflow: minimal `Task`, `submit_subtask`, `submit_subtasks`, `current_task.display`, local runner, and job display.
- Sentinel-2 download workflow: staged metadata loading, filtering, selection, provider storage download, `depends_on`, `max_retries`, and `LocalFileSystemCache`.
- Cron automation workflow: `CronTask`, default fields, trigger time windows, dataset queries, and automation retries.
- Hyperspectral PCA workflow: recursive/scalable fanout, chunk-level display labels, `logger.bind`, `job_cache` keys, and optional cloud-backed runner cache.
