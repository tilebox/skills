# Zarr As Workflow Rendezvous

Use Zarr as the preferred shared intermediate format for distributed geospatial workflows. It works well for multidimensional arrays, region writes, object storage, and handoff between Tilebox tasks.

When writing Zarr, prefer the lower-level `zarr` library directly. When reading, xarray is fine when labels, coordinates, or convenient slicing help.

## Recommended Pattern

1. A setup task creates the group and arrays once.
2. Worker tasks write deterministic, non-overlapping regions.
3. Aggregation/finalization tasks read regions or whole variables as needed.
4. Final raster products are emitted as COGs unless another output format is requested.

## Direct Zarr Writes

Prefer direct writes for predictable region updates:

```python
import zarr

group = zarr.open_group(store, mode="a")
arr = group["mosaic"]
arr[band, y0:y1, x0:x1] = data
```

Initialize arrays explicitly:

```python
zarr.create_array(
    store=store,
    name="mosaic",
    shape=(n_bands, n_y, n_x),
    chunks=(1, chunk_y, chunk_x),
    dtype="uint16",
    fill_value=0,
    dimension_names=("band", "y", "x"),
    compressors=compressor,
)
```

Check the installed Zarr version for exact compressor and metadata APIs.

## Reading With xarray

Use xarray for labeled reads and reductions:

```python
import xarray as xr

cube = xr.open_zarr(store, consolidated=False)
subset = cube["B04"].isel(y=slice(y0, y1), x=slice(x0, x1))
```

Always subset to the task's bounded region before loading/computing arrays.

## Region Write Rules

- Align Tilebox task chunks with Zarr chunks when practical.
- Avoid overlapping writes from concurrent tasks.
- Use deterministic region slices derived from task inputs.
- Use fill values for sparse regions.
- Keep schema/chunks/dimensions stable for a job.
- Store the group/prefix in task inputs, not large arrays.
- Use `obstore` stores for cloud-backed object storage.

## Good Uses

- Sentinel-2 time-by-y-by-x cubes.
- Cloud-free mosaic intermediates.
- Per-tile ML embeddings.
- Chunked statistics inputs.
- Shared arrays that many tasks write and later tasks read.

## Avoid

- Passing xarray/Zarr objects as task parameters.
- Creating the same Zarr arrays concurrently from many tasks.
- Tiny chunks that create excessive object counts.
- Huge chunks that force full-scene reads.
- Treating Zarr as the default final raster output when the user simply asked for a raster; use COG for that.

References: Zarr docs, xarray Zarr IO docs, obstore Zarr examples.
