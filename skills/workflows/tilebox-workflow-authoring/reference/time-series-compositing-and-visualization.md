# Time-Series Compositing And Visualization

Use this reference for imagery timelapses, periodic composites, cloud-free mosaics, before/after comparisons, and other products where multiple acquisitions must look and align consistently over time.

## Establish The Sequence

Define:

- AOI and output grid.
- Start/end dates and expected cadence.
- Sensor, product level, bands, and quality layers.
- Frame selection rule when several scenes cover the same period.
- Missing-period behavior: omit, hold, interpolate only when scientifically justified, or render an explicit no-data frame.
- Final product: frames, contact sheet, MP4/WebM/GIF, periodic COGs, Zarr cube, or an interactive-ready manifest.

Sort by acquisition time from source metadata, not filenames. Preserve source timestamps and product IDs in the output manifest.

## Make Every Frame Spatially Comparable

Choose one target CRS, extent, resolution, transform, width, and height for the sequence. Reproject every acquisition to that grid before frame rendering or pixel-wise comparison.

- Use nearest-neighbor resampling for classes, QA, masks, and labels.
- Choose continuous-band resampling deliberately and document it.
- Apply the same AOI clipping and nodata conventions to every frame.
- Detect empty or partial-coverage frames instead of silently stretching missing data.
- For before/after products, verify registration around stable features before interpreting pixel differences as change.

## Keep Styling Stable Across Time

Do not calculate an independent display stretch for every frame unless the purpose is to visualize relative contrast within each acquisition. Per-frame stretching can make unchanged surfaces flicker and can hide or exaggerate temporal change.

For a stable visual sequence:

1. Apply product scale/offset and masks consistently.
2. Derive fixed visualization limits from a representative sample, a baseline period, robust sequence-wide percentiles, or documented domain limits.
3. Reuse the same band mapping, limits, gamma, colormap, and alpha/nodata treatment for every frame.
4. Keep legends and class colors fixed.
5. Record visualization parameters in the manifest.

If atmospheric, seasonal, or illumination normalization is needed, treat it as an explicit processing stage and preserve enough provenance to explain the transformation.

## Select And Composite Observations

Common period-selection strategies include:

- Lowest-cloud or highest-quality valid observation.
- Median or another robust statistic over valid pixels.
- Most recent valid observation per pixel.
- Domain-specific quality ranking.
- A before/event/after set selected for comparable geometry and season.

Do not call a product cloud-free unless invalid pixels were masked and the remaining spatial/temporal coverage was checked. Document how observations from different dates are combined within one composite.

## Tilebox Task Graph

A scalable sequence commonly uses:

1. A root task that queries scenes and stores a compact manifest.
2. One task per scene/product to read, mask, normalize, and reproject it to a deterministic intermediate key or Zarr region.
3. One task per period or output chunk to composite valid observations when needed.
4. One task per frame to render from aligned data using shared visualization parameters.
5. An aggregation task depending on all required frames to validate ordering, encode the final animation/video, and write the output manifest.

Add progress totals for selected scenes, composites, and rendered frames. Keep frame and intermediate keys deterministic so retries overwrite or reuse the same locations safely.

## Encode And Publish Visual Outputs

Declare encoder and image dependencies in `pyproject.toml`; do not assume an undeclared system encoder exists on every runner. Choose a format based on the user's delivery needs and runtime support:

- MP4 for broadly compatible video when the selected codec/runtime supports it.
- WebM when web delivery and its codec support are appropriate.
- GIF only for short, low-color previews; it is inefficient for long photographic sequences.
- PNG/WebP/JPEG frames when the consumer assembles or serves them separately.

Use a fixed frame size and pixel format. Make frame duration or timestamps explicit; irregular acquisition intervals should not silently appear evenly spaced unless that presentation choice is intentional.

Publish at least:

- Final animation/video or frame prefix.
- Thumbnail or representative frame when useful.
- Ordered frame manifest with timestamps and source IDs.
- AOI, CRS/grid, visualization parameters, missing-frame decisions, and output checksums or sizes where practical.

## Verification

Before considering the product complete:

1. Confirm frames are ordered correctly and have identical dimensions.
2. Inspect stable landmarks for spatial jitter.
3. Check for display flicker caused by inconsistent stretches or masks.
4. Verify timestamps, labels, legends, and no-data rendering.
5. Decode or open the final artifact, not only the intermediate frames.
6. Confirm the manifest references every rendered frame and source product.
7. State limitations from clouds, missing periods, mixed sensors, seasonal effects, or insufficient spatial resolution.
