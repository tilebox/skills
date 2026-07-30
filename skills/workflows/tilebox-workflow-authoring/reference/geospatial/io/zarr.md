# Zarr Arrays

Use Zarr for multidimensional arrays, cloud-backed partial IO, and deterministic non-overlapping region writes. Prefer direct `zarr` writes for schema creation and region updates; use xarray for bounded labeled reads when its coordinates and dimensions help.

## Recommended Pattern

1. One setup stage creates groups and arrays.
2. Independent workers write deterministic, non-overlapping regions.
3. Finalization reads bounded regions or variables.
4. Publish a COG for a conventional final 2D raster unless another format is requested.

```python
import zarr

group = zarr.open_group(store, mode="a")
group["mosaic"][band, y0:y1, x0:x1] = data
```

Initialize schema, dimensions, chunks, dtype, fill values, and compressors once before concurrent writes. Check the installed Zarr version for exact compressor APIs. Choose storage chunks around common read/write regions, distinguish them from in-memory or lazy compute chunks, and avoid overlapping concurrent writes, tiny chunks, and chunks so large that every access reads a scene. Use deterministic slices and preserve CRS/grid metadata in attributes or coordinates.

Good uses include time-by-y-by-x cubes, mosaic intermediates, tiled embeddings, chunk-statistics inputs, and arrays written by many workers then read by later stages. Use fill values for sparse regions, keep schema stable, and use cloud object-store adapters where appropriate. Do not pass Zarr/xarray objects through orchestration inputs or let many workers race to create the same array.

```python
import xarray as xr

cube = xr.open_zarr(store, consolidated=False)
subset = cube["B04"].isel(y=slice(y0, y1), x=slice(x0, x1))
array = subset.load().to_numpy()
```

Subset before calling `.load()`, `.compute()`, `.values`, or `.to_numpy()`. Use xarray `to_zarr(...)` only when it materially simplifies metadata/schema creation and its write behavior is understood; prefer direct Zarr writes for deterministic concurrent regions. A plain xarray object is not geospatially complete unless its CRS, transform/grid, dimensions, and coordinate semantics are explicit; preserve them with suitable metadata or geospatial accessors before converting to NumPy.

COG is generally a better final format for a conventional 2D raster.
