# Tiling, Windowing, And Tilebox Parallelism

Use Tilebox tasks as the outer parallel execution model for geospatial workflows. If work should happen across many scenes, granules, AOIs, product files, spatial chunks, temporal windows, or model tiles, express that with `context.submit_subtask(s)` and dependencies.

## Distinguish Similar Chunk Concepts

- **Tilebox task chunk**: a scheduling unit for distributed workflow execution.
- **Raster window**: a `(row_off, col_off, height, width)` read/write region in a raster.
- **GeoTIFF internal tile/block**: physical storage layout for efficient range reads.
- **Web map tile**: XYZ/TMS tile pyramid, commonly Web Mercator.
- **Zarr chunk**: storage and compute chunk for multidimensional arrays.
- **Array-library chunk**: in-task lazy/eager array partitioning; not the workflow scheduler.

Align these when useful, but do not treat them as the same abstraction.

## Choose The Tilebox Fanout Axis First

Common fanout choices:

- One task per scene/granule for independent source products.
- One task per product/band when assets are separate files.
- One task per AOI for independent regions.
- One task per time window for period aggregates.
- One task per spatial chunk for mosaics, local filters, statistics, or inference.
- One task per model tile for fixed-size ML inference.
- Recursive task trees for global reductions that can be represented as compact mergeable summaries.

Keep task inputs compact: chunk bounds, object keys, IDs, CRS/resolution, and small config values. Store large arrays and shared intermediates in Zarr/object storage.

## Chunk Size Rules Of Thumb

- Large enough to amortize task scheduling, object-store, and open-file overhead.
- Small enough to fit memory after all temporary arrays are allocated.
- Prefer chunks matching model input size for ML inference.
- Prefer chunks compatible with output Zarr chunking for region writes.
- For windowed COG reads, align with internal blocks when practical.
- Avoid millions of tiny tasks and avoid single tasks that read whole products unnecessarily.

## Windowed IO Pattern

For COG/GeoTIFF reads, prefer `async-geotiff` when it supports the operation:

```python
from async_geotiff import GeoTIFF, Window

gtiff = await GeoTIFF.open(object_key, store=store)
tile = await gtiff.read(window=Window(col_off=x0, row_off=y0, width=x1 - x0, height=y1 - y0))
arr = tile.data
```

For Rasterio/GDAL readers, especially when using non-GeoTIFF formats or APIs requiring GDAL:

```python
from rasterio.windows import Window

window = Window(col_off=x0, row_off=y0, width=x1 - x0, height=y1 - y0)
arr = src.read(window=window)
```

Use Rasterio/GDAL for reprojection, writing, or non-GeoTIFF formats.

## Tree Reduction Pattern

For global statistics, compute compact local results per chunk, store them in `context.job_cache` or object storage, then combine them through aggregation tasks. This avoids loading all pixels into one task.

Good reduction payloads are compact summaries whose size does not grow with pixel count:

- counts
- sums or sums of powers
- histograms
- class counts
- min/max ranges
- small sketches or approximations when exact results are not required
- any other mergeable statistics specific to the analysis

Avoid storing large arrays, per-pixel records, or workflow-specific bulky state in `job_cache`.

## Progress Indicators

Use progress indicators for large fanout. Add totals in the task that submits work and mark completion in the task that actually completes the represented unit. Make the unit explicit: granules, products, chunks, tiles, or output regions.
