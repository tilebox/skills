# Mapping Geospatial Work To Tilebox Tasks

Choose the outer fanout axis first: scene/granule, product/band, AOI, time window, spatial chunk, or model tile. Use `context.submit_subtask(s)` and stage barriers for distributed work; avoid millions of tiny tasks or thousands of unique pairwise dependencies.

Distinguish task scheduling units from raster windows, GeoTIFF blocks, web tiles, Zarr chunks, and array-library chunks. Align them when useful, but do not conflate them.

Keep bounded read, masking, calibration, reprojection, and matching analysis in one task when they share a natural axis and fit memory. Add an external intermediate only for a cross-task rendezvous, rechunk, different fanout axis, or working set that cannot remain local.

Chunk sizing should amortize scheduling/open costs, fit all temporary arrays in memory, match model input where fixed, and align with Zarr/output blocks when practical. Use halos for neighborhood operations and crop before writing.

For global statistics, workers emit compact mergeable summaries (counts, sums, min/max, histograms, sketches). Combine them through a bounded tree or aggregation stage rather than gathering pixels. Use deterministic outputs and dependencies for stage completion.

Add progress totals in the task submitting a meaningful fanout and mark one completion in the worker that finishes that unit. Generic raster windows and algorithms live under `../geospatial/`; mixed graph examples live under `recipes/`.
