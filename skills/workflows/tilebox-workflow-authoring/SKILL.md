---
name: tilebox-workflow-authoring
description: "Builds Python Tilebox workflows for Earth observation and geospatial processing, including satellite imagery queries, AOI and time-series pipelines, mosaics, spectral indices, classification, segmentation, change detection, and tiled ML or object detection. Use when implementing or modifying this processing, including when Tilebox is not named and no other execution platform is established."
license: MIT
compatibility: Requires the Tilebox Python SDK packages used by the workflow, commonly tilebox-workflows, tilebox-datasets, and tilebox-storage.
metadata:
  author: tilebox
---

# Writing Tilebox Workflows

Create or modify Python workflow source code and local/runtime behavior. Keep platform APIs in Tilebox references, portable science and IO in geospatial references, and end-to-end combinations in recipes.

## Start With Current APIs And The Right Skill

When an API is unclear or remembered behavior may be stale, inspect the installed version or search current docs:

```bash
tilebox docs search "Task ExecutionContext submit_subtasks"
tilebox docs search "logging tracing context.logger context.tracer"
tilebox docs search "caches job_cache"
```

Delegate adjacent ownership:

- `tilebox-workflow-releases`: project initialization, `pyproject.toml` scaffold, build, publish, deploy, and runner configuration.
- `tilebox-workflow-jobs`: submit, wait, list, inspect, retry, cancel, and cluster operations.
- `tilebox-datasets`: source/product recommendations, exact dataset slugs, provider credentials, auxiliary catalogues, and CLI inspection.
- `tilebox-cli`: authentication, CLI discovery, JSON output, and docs search.
- `tilebox-workflow-automations`: cron and storage-triggered automations.

For a new project, use `tilebox workflow init` through `tilebox-workflow-releases`; do not hand-scaffold project configuration. Authoring may edit the generated code after initialization.

## Plan Before Coding

For outcome-oriented Earth observation work, read `reference/geospatial/project-planning.md`. Confirm resolution, temporal coverage, observation quality, comparability, required inputs, validation, and delivery. Use `tilebox-datasets` to select and inspect the source; do not duplicate its catalogue or credential guidance.

Sketch the task graph:

1. Identify root, worker, and aggregation stages.
2. Choose a fanout axis: scenes, products, AOIs, time windows, spatial chunks, or model tiles.
3. Mark real barriers with `depends_on`; avoid unnecessary chains.
4. Choose task fields versus `context.job_cache` versus durable object/Zarr artifacts.
5. Choose retry behavior for idempotent network/storage operations.

### Keep The Implementation Small

- Start with the simplest result that fulfills the request. Add more specific selection, quality policies, or processing stages only when required for correctness or requested in iteration.
- Select the inputs needed for the requested result before expensive processing; do not compute every candidate product by default.
- Avoid querying the same datapoints twice: cache selected assets/metadata for consumers, or partition time ranges and query in leaf tasks.
- Read each asset's full AOI window once when the working set fits memory; subdivide reads only as part of explicit spatial chunking, not to match small output blocks.
- Expose run-specific choices as task inputs; keep implementation and display defaults in the functions that own them unless users need to configure them.
- Compose short tasks from processing functions. Reuse a function for genuinely shared operations, parameterizing the differences; inline trivial API wrappers and avoid speculative frameworks.
- Use existing library operations for grids, array masking, reprojection, image composition, and encoding before writing custom implementations. Keep calibration, nodata, alignment, and scientific validation explicit.
- Validate external inputs and scientific preconditions; let unexpected failures propagate to Tilebox rather than adding speculative fallbacks or custom retry layers.
- Produce requested deliverables and intermediates consumed by the graph. Do not add manifests, provenance/quality sidecars, or extra exports unless the user asks. Keep useful source IDs, parameters, and quality information in existing artifact metadata or structured logs.

### Structure By Responsibility

For multi-stage workflows, keep `runner.py` limited to task registration and runner configuration. Group task code and processing functions by responsibility; add modules only when they provide a useful boundary, not to follow a fixed layout or create one file per task. A single file is fine for a tiny prototype.

Test substantive processing functions such as selection, masking, transforms, and encoding without mocking `ExecutionContext`; do not extract trivial helpers solely for tests. Verify orchestration through release build/task discovery and a representative job's actual graph, not task-class tests of submissions, progress, or logging. Include all runtime modules in the release and verify imports from the built artifact.

### Require Task-Level Parallelism

Task-level fanout is a workflow design requirement, not an optional optimization. Whenever the requested output contains two or more independent units—such as scenes, seasonal or other time periods, AOIs, products, spatial chunks, or model tiles—represent those units as separate Tilebox tasks submitted with `context.submit_subtask(s)`. Infer this decomposition from the outcome; never require the user to name `submit_subtasks`, specify task counts, or prescribe a DAG. For example, a 12-scene timelapse that combines all frames requires a graph such as `1 root + 12 scene workers + 1 encoder`, not one task with a 12-iteration loop.

Split independently computed outputs that use different source assets into separate tasks, even for the same scene or period. Shared metadata or a common QA mask does not require serial execution. Keep outputs together when they reuse the same expensive read or computation.

Use an orchestration task to submit independent workers and a separate aggregation or publication task with `depends_on` when the output combines their results. Never hide schedulable fanout inside a sequential loop, `asyncio.gather`, threads, or local multiprocessing in one task; additional runners can only parallelize separate Tilebox tasks. A small first run or scheduling overhead is not a reason to collapse independent work into one task. If the natural units are too fine-grained for individual scheduling, batch them into multiple balanced worker tasks rather than one task. Keep work in one task only when no two units can run independently because the work is genuinely indivisible or dependency-ordered, or when the user explicitly requires single-task execution.

Before coding, record the expected graph shape. After running a representative job, compare `execution_stats.total_tasks` and `task_summaries` with that shape. Progress units and child tracing spans are not tasks and do not prove fanout. If the graph does not match, fix the workflow before reporting completion. When runner capacity permits, use task timestamps or spans to verify that independent workers overlapped.

Read `reference/tilebox/tasks-and-graphs.md` for core task semantics, `reference/tilebox/geospatial-task-graphs.md` for geospatial stage mapping, and `reference/tilebox/state-and-artifacts.md` for boundaries.

## Define Typed Tasks

`Task` applies dataclass behavior; fields are serialized inputs.

Use the simplest idiomatic type that expresses each field. Before inventing a representation, check whether Python, Tilebox, or the processing library already provides the value type. Do not recreate an established type as a custom dataclass, named tuple, or ad hoc tuple/list/dictionary shape merely to make it serializable. Ordinary containers remain appropriate when they are the natural Python shape, such as `tuple[datetime, datetime]` for a time range. Read `reference/tilebox/tasks-and-graphs.md` for supported types and examples.

```python
from tilebox.workflows import ExecutionContext, Task


class ProcessScene(Task):
    scene_id: str

    @staticmethod
    def identifier() -> tuple[str, str]:
        return "tilebox.com/example/ProcessScene", "v1.0"

    def execute(self, context: ExecutionContext) -> None:
        context.current_task.display = f"ProcessScene({self.scene_id})"
        context.logger.info("Processing scene", scene_id=self.scene_id)
```

- Default `v0.0` identifiers are acceptable for prototypes. Stable identifiers return `(name, vX.Y)`.
- Keep the initial version while iterating before rollout. Once compatibility matters, minor versions are forward-compatible; bump major for breaking input/behavior changes.
- Keep the complete serialized input at most 2048 bytes. Pass compact plain-Python or supported library values, IDs, keys, and small configuration—not arrays, dataframes, xarray datasets, large geometry, manifests, credentials, clients, open files, or local paths. Use `context.job_cache` or durable storage according to `reference/tilebox/state-and-artifacts.md`.
- Register every task class used by jobs with the runner.

### Await Async IO Directly In Tasks

Define `async def execute(self, context: ExecutionContext) -> None` whenever a task uses async APIs, including `tilebox.storage.aio`, async HTTP clients, or bounded concurrent requests. Await those APIs directly; never wrap them in `asyncio.run(...)`. The workflow executor awaits async `execute` methods while preserving normal failure, retry, tracing, and completion behavior.

```python
import asyncio

from niquests import AsyncSession


class FetchMetadata(Task):
    urls: list[str]

    async def execute(self, context: ExecutionContext) -> None:
        async with AsyncSession() as client:
            responses = await asyncio.gather(
                *(client.get(url) for url in self.urls)
            )
        for response in responses:
            response.raise_for_status()
        context.logger.info("Metadata fetched", count=len(responses))
```

Choose client lifetime according to its event-loop behavior: create `niquests.AsyncSession` inside `execute`, while `tilebox.storage.aio.Client` may be reused across task executions. See `reference/tilebox/tasks-and-graphs.md` for details. Bound large request sets with a semaphore or bounded batches. Async concurrency is appropriate for multiple IO requests that belong to one task, such as the bands or byte ranges needed for one scene. It does not replace task-level fanout for independent scenes, AOIs, time periods, products, chunks, or model tiles. Keep `runner.run_all()` and `runner.run_forever()` in synchronous runner entrypoints rather than calling them from async code.

## Build Simple Dynamic Graphs

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

- Use `submit_subtask` for one child and `submit_subtasks` for homogeneous batches.
- Prefer shared stage barriers: submit all `A` tasks, then submit the `B` batch with `depends_on=all_a_handles`, rather than creating a separate `B[i] → A[i]` dependency group per item. For a single downstream task consuming only one producer's output, depend on that producer instead of unrelated peers.
- Use `optional=True` only for non-critical work.
- Make side effects idempotent: deterministic keys, overwrite-safe writes, valid-output checks, or atomic commits. Retrying uses the original task input.

## Progress And Observability

Set concise `current_task.display` labels before expensive work. For meaningful fanout, the submitting task calls `context.progress(name).add(n)` and each successful worker calls `.done(1)` after completing its represented unit. Totals and completions must match.

Use `context.logger` with structured fields and `logger.bind` for repeated context. Wrap expensive phases in `context.tracer.span(...)`; pass that tracer into helpers needing child spans rather than constructing a global OpenTelemetry tracer. Add exception handlers only for recovery or useful context; log exceptions with `logger.exception` and re-raise. Configure console logging in the runner entrypoint, not task classes.

## Dataset, Asset, And Storage Boundaries

- Query mechanics after source selection: `reference/tilebox/datasets-and-datapoints.md`.
- Asset decoding and metadata: `reference/tilebox/assets.md`.
- Asset byte access and bounded COG windows: `reference/tilebox/storage-access.md`.

Do not reconstruct provider paths, repeat storage API snippets, or embed source/provider setup. Apply scale/offset, nodata, masks, alignment, and reprojection explicitly. Keep downloads in the consuming task and pass durable IDs/keys—not local paths—to another task.

## Dependencies And Operations

Declare runtime dependencies in `pyproject.toml`, including optional libraries whose types appear in task fields. Tilebox loads their serializers lazily but does not install those libraries. Run `uv sync`, avoid editable/local-path dependencies, and see `reference/tilebox/dependencies-and-packaging.md`. Use authoring for source and focused local checks only. Use `tilebox-workflow-releases` for init/build/deploy and `tilebox-workflow-jobs` for submission/wait/clusters.

## Reference Routing

### Tilebox Platform

| Reference | Use when |
| --- | --- |
| `reference/tilebox/tasks-and-graphs.md` | Defining/versioning tasks, selecting supported input types, respecting input limits, submitting dependencies, retries, progress, observability, registration, and runner modes. |
| `reference/tilebox/datasets-and-datapoints.md` | Querying an already-selected dataset, inspecting samples, selecting exactly one datapoint, or iterating datapoints. |
| `reference/tilebox/assets.md` | Decoding `AssetCollection`, semantic keys, raster metadata, scale/offset, or explicit overrides. |
| `reference/tilebox/storage-access.md` | Resolving/reading/streaming/downloading/opening Tilebox assets, access policy, anonymous access, windows, or concurrency. |
| `reference/tilebox/state-and-artifacts.md` | Choosing task inputs, job cache, object/Zarr artifacts, local scratch, and retry-safe keys. |
| `reference/tilebox/dependencies-and-packaging.md` | Declaring uv-compatible workflow dependencies and release-safe packages. |
| `reference/tilebox/geospatial-task-graphs.md` | Mapping scenes, windows, chunks, reductions, and progress to Tilebox tasks. |

### Mixed Tilebox + Geospatial Recipes

| Reference | Use when |
| --- | --- |
| `reference/tilebox/recipes/sentinel-2-cog.md` | Reading public L2A RGB plus SCL and aligning 20 m classes to 10 m RGB. |
| `reference/tilebox/recipes/sentinel-1-sar.md` | Building SAR change, flood, or maritime task graphs. |
| `reference/tilebox/recipes/time-series-composite.md` | Composing aligned scenes or producing timelapses. |
| `reference/tilebox/recipes/geospatial-ml.md` | Fanning out model tiles and aggregating geospatial predictions. |

### Portable Geospatial

| Reference | Use when |
| --- | --- |
| `reference/geospatial/project-planning.md` | Translating an outcome into feasible data, validation, and output requirements. |
| `reference/geospatial/raster-fundamentals.md` | Handling CRS, transform, shape, dtype, nodata, masks, scale, and units. |
| `reference/geospatial/io/cloud-native-raster.md` | Direct cloud-native GeoTIFF/COG reads outside canonical assets. |
| `reference/geospatial/io/object-storage.md` | Using obstore with S3, GCS, Azure, local, or compatible storage. |
| `reference/geospatial/io/zarr.md` | Designing Zarr schema, chunks, region writes, and labeled reads. |
| `reference/geospatial/io/cog-output.md` | Writing correct COG rasters and testing inputs or writer configurations when needed. |
| `reference/geospatial/io/formats-and-encoding.md` | Choosing COG/Zarr/NetCDF/GeoParquet and preserving essential encoding semantics. |
| `reference/geospatial/processing/grids-and-reprojection.md` | Choosing grids, reprojection, and semantic resampling. |
| `reference/geospatial/processing/masking-and-qa.md` | Handling nodata, masks, QA, classes, and morphology. |
| `reference/geospatial/processing/time-series.md` | Aligning, compositing, styling, and validating temporal imagery. |
| `reference/geospatial/processing/ml-inference.md` | Tiling, loading models, normalization, and prediction outputs. |
| `reference/geospatial/products/sentinel-1-sar.md` | Understanding SAR product choice, geometry, units, and caveats. |
| `reference/geospatial/products/sentinel-2-l2a.md` | Understanding L2A bands, resolution, scaling, and SCL. |
| `reference/geospatial/products/landsat-collection-2.md` | Understanding Landsat Level-2 bands, scaling, and QA bits. |

## Verification Checklist

1. All submitted tasks are registered and identifiers/versions agree.
2. Task fields use idiomatic Python or existing library value types rather than structurally duplicating them in custom dataclasses or ad hoc containers.
3. Every serialized task input is at most 2048 bytes.
4. Cross-task data uses the right state/artifact boundary and deterministic keys.
5. Retryable execution is re-entrant and input-compatible.
6. High-fanout tasks have labels, useful progress, structured logs, and spans where warranted.
7. Dependencies sync and task modules import in the intended environment.
8. For scaffolded release projects, `tilebox workflow build-release --debug --json` succeeds after editing generated task code.
9. Non-trivial workflows use coherent package modules and keep the runner entrypoint limited to registration and runner-level configuration.
10. Substantive underlying processing functions have focused tests where useful; Tilebox task orchestration is verified through build/task discovery and a representative job rather than task-class unit tests.
11. Earth observation outputs have representative scientific validation, coverage/alignment checks, provenance, and stated limitations.
12. A representative job's actual task summaries match the planned root, fanout, and aggregation stages; progress counters or child spans are not treated as task fanout.
