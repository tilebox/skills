# Geospatial ML Recipe

Follow the portable [model tiling and normalization guidance](../../geospatial/processing/ml-inference.md).

1. A root task stores any large AOI/manifest externally, initializes the output grid/Zarr schema once, and submits compact model-tile inputs containing source/output keys, tile bounds, grid identity, and small model configuration.
2. Each inference task reads a bounded source COG/Zarr region, including halo, and computes grid-derived positional inputs.
3. Lazily load and process-cache the model; obtain weights from a verified runner-local cache or accessible object storage.
4. Crop halo outputs and map patch-grid outputs to deterministic non-overlapping Zarr regions, or emit per-tile vectors under deterministic keys. The same tile retry must target the same output.
5. A dependent aggregation task validates coverage, merges region outputs, deterministically deduplicates overlapping vectors, and publishes final COG/vector products plus grid, normalization, class, confidence, and model/version metadata.

Match fanout chunks to fixed model input size when possible; include halos without overlapping final raster writes. Add progress in the parent and complete it only after each tile is durably written. Never pass arrays, tensors, xarray datasets, large AOIs, or weights as Tilebox task fields or through `context.job_cache`.

Load models lazily with process caching. Fetch weights to a deterministic runner-local cache or accessible private object store, validate checksums/sizes, and remain correct on a cold runner. Declare architecture-safe ML dependencies and indexes in `pyproject.toml`; do not package large weights in the release by accident.
