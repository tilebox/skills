# Object Storage With obstore

Prefer `obstore` for object-store access in workflow code unless the project already standardizes on a different client or a provider-specific SDK is required.

`obstore` provides a typed sync/async Python interface to S3, GCS, Azure, local files, and S3-compatible APIs such as R2 or MinIO. It is a good default for high-throughput geospatial workflows, Zarr stores, COG inputs, and streaming IO.

## Use Cases

- Open COG/GeoTIFF data through `async-geotiff`.
- Back Zarr stores used as workflow rendezvous.
- Stream downloads and uploads without loading whole objects into memory.
- List object prefixes without manual pagination.
- Use automatic multipart uploads for large outputs.
- Use a single abstraction across local, S3, GCS, Azure, and S3-compatible endpoints.

## Store Selection

Typical imports:

```python
from obstore.store import AzureStore, GCSStore, LocalStore, S3Store
```

Patterns:

```python
store = S3Store(bucket="my-bucket", prefix="workflow/run-123")
store = GCSStore(bucket="my-bucket", prefix="workflow/run-123")
store = LocalStore("/tmp/workflow-data")
```

For public S3-compatible buckets, configure unsigned access explicitly where supported by the store/auth settings. For private buckets, prefer environment-provided credentials or credential providers appropriate for the runner environment.

## Workflow Key Design

Use deterministic keys derived from job ID, product ID, stage, and chunk coordinates:

```text
jobs/{job_id}/cube/
jobs/{job_id}/mosaic/y={y0}-{y1}_x={x0}-{x1}.tif
jobs/{job_id}/stats/{product}/{chunk_key}.bin
```

Retryable tasks should overwrite the same key or use an atomic commit/rename pattern when the sink supports it. Avoid append-only outputs unless the sink has idempotency or deduplication.

## Shared State Boundaries

- Task inputs: small IDs, keys, chunk bounds, config.
- `context.job_cache`: compact metadata or small reduction results.
- `obstore`/Zarr/object storage: large arrays, derived rasters, manifests, ML outputs.
- Local filesystem: only for task-local scratch or runner-local caches unless all tasks run on the same machine.

## Prefer obstore Over HTTP For Storage

Use `niquests` for non-storage APIs. Use `obstore` for object storage, range-readable geospatial inputs, streaming object IO, and Zarr/COG integration.

References: https://developmentseed.org/obstore/latest/
