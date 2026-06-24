# Cloud-Native Raster IO

Prefer `async-geotiff` for GeoTIFF and Cloud Optimized GeoTIFF reads when applicable. It integrates with `obstore`, supports window/overview reads, exposes CRS/transform metadata, handles masks, and avoids a GDAL dependency for read-only COG access.

It is fine to use `async-geotiff` from otherwise synchronous Tilebox task code by running the async call in the task's local execution context. Keep Tilebox subtasks as the workflow-level parallelism.

## async-geotiff Is Best For

- COG/GeoTIFF reads.
- Windowed reads from object storage.
- Overview reads for lower-resolution previews or statistics.
- Read-only pipelines that do not need warping/resampling.
- Integrating with `obstore` stores for S3/GCS/Azure/S3-compatible backends.

Sketch:

```python
from async_geotiff import GeoTIFF, Window
from obstore.store import S3Store

store = S3Store(bucket="sentinel-cogs", region="us-west-2", skip_signature=True)
gtiff = await GeoTIFF.open("path/to/image.tif", store=store)
tile = await gtiff.read(window=Window(col_off=x0, row_off=y0, width=w, height=h))
arr = tile.data
mask = tile.mask
```

Check the current API before copying exact calls; prefer docs/local package introspection for version-specific details.

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
