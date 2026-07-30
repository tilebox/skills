# Object Storage With obstore

Prefer `obstore` for object-store access unless the project already standardizes on a different client or a provider-specific SDK is required.

`obstore` provides a typed sync/async Python interface to S3, GCS, Azure, local files, and S3-compatible APIs such as R2 or MinIO. It is a good default for high-throughput geospatial processing, Zarr stores, COG inputs, and streaming IO.

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

For public S3-compatible buckets, configure unsigned access explicitly where supported by the store/auth settings. For private buckets, prefer environment-provided credentials or credential providers appropriate for the execution environment.

## Key Design

Use deterministic keys derived from run ID, product ID, stage, and chunk coordinates:

```text
runs/{run_id}/cube/
runs/{run_id}/mosaic/y={y0}-{y1}_x={x0}-{x1}.tif
runs/{run_id}/stats/{product}/{chunk_key}.bin
```

Retryable work should overwrite the same key or use an atomic commit/rename pattern when the sink supports it. Avoid append-only outputs unless the sink has idempotency or deduplication.

## Prefer obstore Over HTTP For Storage

Use `niquests` for non-storage APIs. Use `obstore` for object storage, range-readable geospatial inputs, streaming object IO, and Zarr/COG integration.

References: https://developmentseed.org/obstore/latest/
