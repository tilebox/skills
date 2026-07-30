# Tilebox Storage Access

Use the async asset API from `tilebox.storage.aio`. `Client.resolve(asset)` is synchronous and network-free; reads are async. Resolution chooses a compatible declared location, configures anonymous access when no authentication is referenced, and reuses stores with the same policy. Do not probe locations or pass removed per-call `alternate`, `storage_scheme`, or `authentication_scheme` arguments.

```python
from tilebox.storage.aio import Client

storage = Client()
resolved = storage.resolve(asset)
payload = await storage.read_bytes(asset, max_bytes=1_000_000)
async for chunk in storage.iter_bytes(asset):
    consume(chunk)
path = await storage.download(asset, destination, overwrite=False)
geotiff = await storage.open_geotiff(asset)  # Opens; does not read pixels.
```

Equivalent module functions are available from `tilebox.storage.aio`. `iter_bytes(...)` returns an async iterator and is consumed with `async for`, not `await`. Reuse one client so resolved stores and connections are reused.

Use `AssetAccessPolicy` at client construction only when location-scheme preference must be constrained:

```python
from tilebox.storage.aio import AssetAccessPolicy, Client

storage = Client(
    policy=AssetAccessPolicy(preferred_schemes=("s3", "https")),
)
```

For bounded COG reads:

```python
from tilebox.storage.geotiff import window_from_bounds

window = window_from_bounds(
    geotiff,
    bounds=(west, south, east, north),
    crs="EPSG:4326",
    require_fully_contained=True,
)
chunk = await geotiff.read(window=window)
```

The CRS describes the supplied bounds. The API does not apply scale/offset, masks, alignment, reprojection, or resampling. Bound concurrency with a semaphore or bounded batches; do not open thousands of assets at once.

Public source access does not provide output storage. Legacy `open_data.copernicus.*` products are a temporary exception: when their datapoints are not asset-compatible, use the deprecated `CopernicusStorageClient` path documented by `tilebox-datasets`; keep credentials in the runtime environment and validate one product before fanout.
