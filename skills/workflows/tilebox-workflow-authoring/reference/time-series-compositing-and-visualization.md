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

Sort by acquisition time from source metadata, not filenames. Preserve source timestamps and product IDs in output metadata when practical. A separate manifest is optional: recommend it when useful for provenance or downstream consumption, but do not require it for a simple requested artifact such as a GIF.

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
3. Reuse the same band mapping, limits, gamma correction, tone/color transformations, colormap, and alpha/nodata treatment for every frame.
4. Keep legends and class colors fixed.
5. Record visualization parameters in output metadata or an optional manifest when useful.

If atmospheric, seasonal, or illumination normalization is needed, treat it as an explicit processing stage and preserve enough provenance to explain the transformation.

## Select And Composite Observations

Common period-selection strategies include:

- Lowest-cloud or highest-quality valid observation.
- Median or another robust statistic over valid pixels.
- Most recent valid observation per pixel.
- Domain-specific quality ranking.
- A before/event/after set selected for comparable geometry and season.

Scene-level `cloud_cover` is only a coarse prefilter: the requested AOI can be clear inside a cloudy scene or cloudy inside a mostly clear scene. When low cloud is important, first filter to reasonably low scene-level cloud cover, then read the AOI and compute its cloud fraction from the product quality layer. Skip observations whose AOI cloud fraction is too high. For mosaics and composites, mask cloudy pixels before reduction so they do not contribute to the aggregate.

Do not call a product cloud-free unless invalid pixels were masked and the remaining AOI-level spatial/temporal coverage was checked. Document how observations from different dates are combined within one composite.

## Tilebox Task Graph

A scalable sequence commonly uses:

1. A root task that queries scenes and submits compact source identifiers and processing parameters.
2. One task per scene/product to read, mask, normalize, reproject, and render the frame in memory when the source fanout already matches the output frames.
3. If compositing requires a different chunking axis, write deterministic aligned intermediates, then fan out by period or output chunk to composite valid observations.
4. Render in the compositing task when its output already corresponds to one frame; create a separate frame-rendering task only when rendering needs a different fanout or reusable aligned input.
5. An aggregation task depending on all required frames to validate ordering and encode the final animation/video. Write a manifest only when requested or materially useful.

Add progress totals for selected scenes, composites, and rendered frames. Keep frame and intermediate keys deterministic so retries overwrite or reuse the same locations safely.

## Encode And Publish Visual Outputs

Declare encoder and image dependencies in `pyproject.toml`; do not assume an undeclared system encoder exists on every runner. Choose a format based on the user's delivery needs and runtime support:

- MP4 for broadly compatible video when the selected codec/runtime supports it.
- WebM when web delivery and its codec support are appropriate.
- GIF only for short, low-color previews; it is inefficient for long photographic sequences.
- PNG/WebP/JPEG frames when the consumer assembles or serves them separately.

Use a fixed frame size and pixel format. Make frame duration or timestamps explicit; irregular acquisition intervals should not silently appear evenly spaced unless that presentation choice is intentional.

Publish the requested visual product, plus only the supporting outputs that are useful:

- Final animation/video or frame prefix.
- Thumbnail or representative frame when useful.
- Optional ordered frame manifest with timestamps and source IDs when requested or useful.
- AOI, CRS/grid, visualization parameters, missing-frame decisions, and output checksums or sizes where practical, either in artifact metadata or the optional manifest.

## Verification

Before considering the product complete:

1. Confirm frames are ordered correctly and have identical dimensions.
2. Inspect stable landmarks for spatial jitter.
3. Check for display flicker caused by inconsistent stretches or masks.
4. Verify timestamps, labels, legends, and no-data rendering.
5. Decode or open the final artifact, not only the intermediate frames.
6. If a manifest was produced, confirm that it references every rendered frame and source product.
7. State limitations from clouds, missing periods, mixed sensors, seasonal effects, or insufficient spatial resolution.
