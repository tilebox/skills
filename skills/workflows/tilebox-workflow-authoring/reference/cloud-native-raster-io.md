# Cloud-Native Raster IO

For GeoTIFF or COG assets returned by a Tilebox dataset, prefer the asset-aware API in `tilebox.storage.aio`. It resolves canonical locations into reusable `obstore` stores, then opens the native `async-geotiff.GeoTIFF` without reading pixels. Use direct `async-geotiff` and `obstore` construction only when an input is not represented by a Tilebox asset.

It is fine to use `async-geotiff` from otherwise synchronous Tilebox task code by running the async call in the task's local execution context. Keep Tilebox subtasks as the workflow-level parallelism.

## async-geotiff Is Best For

- COG/GeoTIFF reads.
- Windowed reads from object storage.
- Overview reads for lower-resolution previews or statistics.
- Read-only pipelines that do not need warping/resampling.
- Integrating with `obstore` stores for S3/GCS/Azure/S3-compatible backends.

GeoTIFF support is included in `tilebox-storage`. Bounds use the order `(west, south, east, north)`: west is minimum longitude/x, south is minimum latitude/y, east is maximum longitude/x, and north is maximum latitude/y. For example, given one selected datapoint, an asset key, and a WGS84 AOI over New York:

```python
from tilebox.datasets.assets import AssetCollection
from tilebox.storage.aio import Client as StorageClient
from tilebox.storage.geotiff import window_from_bounds

bounds = (-74.05, 40.68, -73.90, 40.83)  # west, south, east, north
assets = AssetCollection.from_datapoint(datapoint)
storage = StorageClient()
gtiff = await storage.open_geotiff(assets[asset_key])
window = window_from_bounds(
    gtiff,
    bounds=bounds,
    crs="EPSG:4326",
    require_fully_contained=True,
)
tile = await gtiff.read(window=window)
arr = tile.data
mask = tile.mask
```

The `crs` argument describes the supplied bounds; `window_from_bounds(...)` transforms them into the GeoTIFF CRS. Never pass longitude/latitude bounds with the raster CRS as their label. When calculating windows without this helper, transform the AOI into the raster CRS first.

The new `Client` exists only in `tilebox.storage.aio`. `resolve(...)` is synchronous and network-free; all data reads are async. Public locations without authentication references are configured anonymously. `AssetCollection.from_datapoint(...)` accepts exactly one datapoint, so select one result or use `tilebox.datasets.iter_datapoints(...)` first. The client does not automatically apply scale/offset, nodata, masks, alignment, reprojection, or resampling.

## Use Rasterio/GDAL When Needed

Use Rasterio/GDAL/odc/rioxarray instead of async-geotiff when you need:

- reprojection or warping
- resampling
- COG/GeoTIFF writing
- JP2, HDF, NetCDF, GRIB, or other GDAL drivers
- update mode
- features not exposed by async-geotiff

## Remote IO Rules

- Prefer reading windows/overviews, not full rasters, for chunked tasks.
- Assume large remote raster workflows need runners deployed near the object storage region/provider. Do not try to solve data locality in task code; choose storage locations and runner deployment targets so reads are already close to the data.
- Avoid listing sidecar files around each open when the backend permits skipping directory reads.
- Keep object keys and auth out of task outputs unless intentionally public.
- Log byte sizes, window bounds, and object keys for debugging slow reads.
- Wrap larger reads and writes in task subtraces, e.g. `with context.tracer.span("read-window"):` or `with context.tracer.span("write-output"):`.

## COG Reads Versus Whole-File Downloads

Use windowed COG reads when partial spatial access saves bytes. Download whole files when:

- the format is not cloud-optimized,
- most of the file will be read anyway,
- downstream libraries require local paths,
- source servers handle range requests poorly,
- or repeated reads benefit from a runner-local cache.

References: https://developmentseed.org/async-geotiff/latest/ and GDAL/Rasterio VSI docs.
