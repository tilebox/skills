# xarray And rioxarray

Use xarray for labeled multidimensional arrays and convenient reads from Zarr/NetCDF-like data. Use rioxarray or odc-geo when geospatial metadata and reprojection are needed. Keep Tilebox tasks as the workflow-level execution model.

## Good Uses Inside Tilebox Tasks

- Open a Zarr group and read the task's bounded region.
- Use named dimensions for readable indexing.
- Compute a local reduction over a task's chunk.
- Preserve coordinates/attrs when helpful.
- Convert a bounded result to NumPy for model inference or direct Zarr writes.

Pattern:

```python
cube = xr.open_zarr(store, consolidated=False)
subset = cube["B04"].isel(y=slice(y0, y1), x=slice(x0, x1))
arr = subset.load().to_numpy()
```

Always subset early. Avoid loading global arrays inside a worker task.

## Reading Versus Writing Zarr

- Reading: xarray is fine and often convenient.
- Writing: prefer the lower-level `zarr` library directly for workflow region writes and array initialization.

Use xarray `to_zarr` only when it materially simplifies metadata/schema creation and the write behavior is well understood.

## Geospatial Metadata

Use rioxarray/odc-geo to attach and preserve CRS/transform metadata. A plain xarray object does not guarantee geospatial correctness unless spatial metadata is explicitly present and understood by the chosen accessor.

For reprojection in Tilebox geospatial workflows, prefer `odc.geo` when it fits. Use rioxarray/Rasterio/GDAL when those APIs are better suited.

## Chunking Guidance

Distinguish storage chunks from Tilebox tasks. For cloud-backed Zarr:

- Choose Zarr chunks that match common task read/write regions.
- Avoid tiny chunks that create too many objects.
- Avoid huge chunks that force full-scene reads.
- Avoid repeated rechunking inside tasks.

## Anti-Patterns

- Passing xarray datasets as task inputs.
- Calling `.values`, `.load()`, or `.compute()` before subsetting.
- Building a second workflow scheduler for tile/scene fanout instead of Tilebox tasks.
- Writing overlapping Zarr regions from concurrent tasks without a clear safety model.
- Losing CRS/transform by converting to NumPy and writing output without metadata.

References: xarray IO docs, xarray parallel/lazy array docs, rioxarray docs, odc-geo docs.
