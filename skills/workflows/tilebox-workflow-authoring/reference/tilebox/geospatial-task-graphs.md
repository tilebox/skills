# Mapping Geospatial Work To Tilebox Tasks

Task-level parallelism is required for independent work. Choose the outer fanout axis from the outputs and workload: scene/granule, product, AOI, time window, spatial chunk, or model tile. Use `context.submit_subtask(s)` and stage barriers for distributed work; avoid millions of tiny tasks or thousands of unique pairwise dependencies.

Distinguish task scheduling units from raster windows, GeoTIFF blocks, web tiles, Zarr chunks, and array-library chunks. Align them when useful, but do not conflate them.

When a task input is an ODC tiling plan, use `GeoboxTiles` with `Index2d`; when it is a Rasterio or async-geotiff read window, use that library's `Window`. Do not create parallel dataclass or tuple/dictionary representations solely for task serialization.

Keep bounded read, masking, calibration, reprojection, and matching analysis in one task when they share a natural axis and fit memory. Add an external intermediate only for a cross-task rendezvous, rechunk, different fanout axis, or working set that cannot remain local.

For small multi-output mosaics, use one worker per independently computed output TIFF. A before/after RGB and NBR workflow has four mosaic workers, followed by delta and overlay tasks with dependencies on their inputs. Each mosaic worker may loop over a few source bands and granules in memory. Account for reads, masks, reprojection buffers, and output arrays at the intended runner concurrency. Output-level fanout is still required; this is not permission to put all mosaics in one task.

For larger areas or input sets, distribute source processing across granule tasks, splitting oversized granules into bounded windows. Follow them with one aggregation task per output chunk, so neither a source worker nor a reducer must hold the entire mosaic. For a mean mosaic, combine valid per-pixel sums and counts, then divide where the count is nonzero; keep zero-count pixels as nodata. Do not average partial means without their counts. If the product uses another compositing rule, preserve that rule when distributing the work.

Chunk sizing should amortize scheduling/open costs, fit all temporary arrays in memory, match model input where fixed, and align with Zarr/output blocks when practical. Use halos for neighborhood operations and crop before writing.

For global statistics, workers emit compact mergeable summaries (counts, sums, min/max, histograms, sketches). Combine them through a bounded tree or aggregation stage rather than gathering pixels. Use deterministic outputs and dependencies for stage completion.

Add progress totals in the task submitting a meaningful fanout and mark one completion in the worker that finishes that unit. Generic raster windows and algorithms live under `../geospatial/`; mixed graph examples live under `recipes/`.
