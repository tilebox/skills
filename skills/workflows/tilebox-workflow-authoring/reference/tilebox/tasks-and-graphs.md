# Tasks And Dynamic Graphs

## Typed Tasks And Versions

Inherit from `Task`; its fields are serializable inputs and `Task` supplies dataclass behavior.

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
```

- The default class-name identifier and `v0.0` are fine for prototypes.
- Stable tasks define `identifier()` as a staticmethod or classmethod returning `(name, version)`, where version is `vX.Y`.
- A runner at `v1.5` can execute a task submitted at `v1.3`, but not the reverse. Keep the major version for compatible fixes and bump it for breaking input or behavior changes.
- Every task used by a submitted job must be registered with the runner, and submitter/runner identifiers must agree.

The complete serialized input for every root or submitted task is limited to **2048 bytes**, including all fields and values. Pass compact values, IDs, small configuration, cache keys, and object prefixes. Do not pass arrays, large GeoJSON, manifests, dataframes, xarray datasets, large binary payloads, thousands of URLs, credentials, clients, open files, or local paths. Put compact job-scoped bytes such as metadata or reduction summaries in `context.job_cache`; put large or durable payloads in object storage or Zarr and pass only a compact key. See [state and artifacts](state-and-artifacts.md). Prefer typed fields and defaults to unstructured dictionaries.

## Choose Native Value Types Before Custom Shapes

Use the simplest idiomatic type that completely expresses the field. Before defining a representation, check whether Python, Tilebox, or the processing library already provides that value type. Do not recreate an established type as a custom dataclass, named tuple, or ad hoc positional tuple/list/dictionary merely to make it serializable. A plain container is still right when it is the conventional Python model for the concept; for example, use `tuple[datetime, datetime]` for an ordinary time range.

Task serialization supports these values directly; no registration or manual conversion is needed:

| Category | Supported values and when to use them |
| --- | --- |
| Plain Python | Primitives, typed containers, unions, optional values, enums, nested dataclasses, `datetime`, `date`, `time`, `timedelta`, `UUID`, `Decimal`, `bytes`, `Path`/`PurePath`, and `ZoneInfo`. Use these for generic concepts, while remembering that a machine-local path is not a portable task input. |
| Tilebox query values | `TimeInterval`, `IDInterval`, and `SpatialFilter`. Use them when their query behavior or explicit endpoint/filter semantics matter; they are not required replacements for ordinary Python ranges and geometry. |
| Shapely | Geometry subclasses such as points, lines, polygons, multi-geometries, and collections. Use for geometry without an associated CRS. |
| ODC Geo | `Geometry`, `BoundingBox`, `CRS`, `GeoBox`, `GeoboxTiles`, `Index2d`, `Shape2d`, `Resolution`, `XY`, and `AnchorEnum`. Use these directly for CRS-aware geometry, grids, and tiling rather than recreating their structure. |
| pyproj and affine | `pyproj.CRS` and `affine.Affine`. Use them when they are the native values of the processing code. |
| Raster windows | `rasterio.windows.Window` and `async_geotiff.Window`. Use the type consumed by the raster API instead of wrapping its offsets and shape in another input model. |
| Protobuf | `google.protobuf.message.Message` subclasses, including nested messages. Use them when the surrounding API already exposes protobuf values. |

For example, ordinary Python time and Shapely geometry remain natural root inputs, while ODC's own types describe an ODC tiling plan without a parallel custom model:

```python
from datetime import datetime

from odc.geo import Index2d
from odc.geo.geobox import GeoboxTiles
from shapely import Geometry
from tilebox.workflows import ExecutionContext, Task


class BuildMosaic(Task):
    time_range: tuple[datetime, datetime]
    aoi: Geometry


class ProcessTile(Task):
    tiles: GeoboxTiles
    tile_index: Index2d

    def execute(self, context: ExecutionContext) -> None:
        tile = self.tiles[self.tile_index]
        output_region = self.tiles.roi[self.tile_index]
        # Read, process, and write this tile.
```

Likewise, pass the window implementation used by the raster API instead of copying its four values into another shape:

```python
from async_geotiff import Window


class ReadRasterTile(Task):
    asset_key: str
    window: Window
```

Use timezone-aware datetimes. Use `odc.geo.Geometry` when the CRS must travel with the geometry. Use Tilebox `TimeInterval` when start/end inclusivity is explicit or the object itself is useful to a dataset query. A custom dataclass remains appropriate for a real workflow-specific concept that combines or adds semantics to existing values; it should not merely rename the fields of one supported type.

Codec integrations are lazy. The workflow must still declare every optional package it imports; see [dependencies and packaging](dependencies-and-packaging.md). Serialization support also does not make large or stateful values suitable task inputs.

## Synchronous And Asynchronous Execution

`execute` may be synchronous or asynchronous. Use `async def execute(...) -> None` when the task awaits `tilebox.storage.aio`, an async HTTP client, or other async IO. Await calls directly instead of putting `asyncio.run(...)` inside a synchronous task; the workflow executor detects and awaits the result. Exceptions from async execution follow the same task failure and retry path as exceptions from synchronous execution.

```python
from niquests import AsyncSession


class FetchMetadata(Task):
    url: str

    async def execute(self, context: ExecutionContext) -> None:
        async with AsyncSession() as client:
            response = await client.get(self.url)
            response.raise_for_status()
```

Runner tasks execute sequentially, but each async `execute` call currently gets a fresh event loop. Sequential execution alone does not make arbitrary async clients reusable across tasks. In particular, construct and close `niquests.AsyncSession` within one `execute` call because its connections are tied to that loop. `tilebox.storage.aio.Client` may instead be cached at runner scope: it caches reusable obstore stores rather than an event-loop-bound session. Follow the documented lifetime of other clients. Use bounded async concurrency for related IO requests within one task, but represent independently schedulable workflow units as subtasks. `runner.run_all()` and `runner.run_forever()` remain synchronous entrypoints and must not be called from a running event loop.

## Subtasks, Dependencies, Optional Work, And Retries

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

- `submit_subtask(task)` submits one child; `submit_subtasks([...])` submits a homogeneous batch and returns handles.
- `depends_on` accepts submitted-task handles and waits for successful completion.
- Use `optional=True` only for non-critical branches whose failure must not fail the job.
- Use `max_retries` for flaky network, storage, and provider operations whose effects are retry-safe.
- Prefer simple stage barriers over thousands of pairwise dependency shapes. Long chains and `B[i]` depending only on `A[i]` at large fanout create expensive fine-grained DAGs; use orchestrator/stage tasks and homogeneous batches instead.

Tasks may retry after a corrected release reaches the same cluster. Make execution re-entrant: derive deterministic output keys from inputs, use overwrite-safe writes, validate existing outputs, or commit atomically. External records and notifications need idempotency keys or deduplication. Retrying an old job uses its original inputs, so preserve input compatibility and avoid radically changing the expected graph. A compatible fix keeps the identifier major version and may bump the minor version; a major bump cannot repair already-submitted tasks.

## Displays And Progress

Set `current_task.display` after useful values are known but before expensive work:

```python
context.current_task.display = f"Chunk[{self.x0}:{self.x1},{self.y0}:{self.y1}]"
```

Useful labels expose the distinguishing runtime dimension: `DownloadImages(n=24)`, `DownloadImage('S2A_001')`, `LocalStats[0:2048,0:2048]`, or `CombineStats n_pixels=12345678`.

Progress indicators use `done / total`: `.add(n)` increases total work and `.done(n)` increases completed work. The parent that submits `n` units adds `n`; each successful worker calls `done(1)` only after its unit completes. Make totals match completions, and use separate names such as `download`, `process`, and `upload`. Do not add and complete subtask progress in the same parent; it never shows useful in-progress state. Tilebox uses the last execution of a retried task to avoid double-counting.

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
        # Process and publish this scene successfully first.
        context.progress("process-scenes").done(1)
```

## Structured Logs And Spans

Tilebox correlates task logs with job, task, runner, trace, and span metadata. Log through `context.logger` and prefer structured fields.

```python
class PublishOutput(Task):
    output_key: str

    def execute(self, context: ExecutionContext) -> None:
        log = context.logger.bind(output_key=self.output_key)
        log.info("Publishing output")
        try:
            with context.tracer.span("publish-output") as span:
                span.set_attribute("output_key", self.output_key)
                # Upload or publish data.
                log.info("Output published", format="cog")
        except Exception:
            log.exception("Output publication failed")
            raise
```

Use `logger.bind(...)` for fields shared by several records. Call `logger.exception(...)` only in an exception handler and re-raise. Add spans around expensive or failure-prone download, compute, and publish phases, with attributes useful for later filtering. Configure console logging in the runner entrypoint rather than task classes.

## Registration And Runner Modes

Register every reachable task class:

```python
from tilebox.workflows import Client

client = Client(name="example-runner")
runner = client.runner(tasks=[ProcessScenes, ProcessScene, PublishSummary])
runner.run_forever()
```

Use `runner.run_all()` in notebooks or scripts that should drain currently available work and return. Use `runner.run_forever()` for a long-running runner process. Project initialization, release build/deployment, runner configuration, and job submission/operations belong to the companion release and jobs skills.

## Proven Public Example Patterns

The public `github.com/tilebox/examples` workflows demonstrate:

- Hello world: minimal `Task`, single and batch subtask submission, display labels, a local runner, and job display.
- Sentinel-2 download: staged metadata loading, filtering and selection, dependencies, retries, and cache boundaries. Follow the current asset/storage references instead of copying legacy provider-access code.
- Cron automation: `CronTask`, default fields, trigger time windows, dataset queries, and automation retries.
- Hyperspectral PCA: recursive/scalable fanout, chunk labels, `logger.bind`, job-cache keys, and an optional cloud-backed runner cache.
