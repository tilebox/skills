# Time-Series Compositing And Visualization

## Establish The Sequence

Define the AOI and output grid; start/end dates and expected cadence; sensor, product level, bands, and quality layers; the frame rule when several scenes cover a period; missing-period behavior (omit, hold, explicitly render no data, or interpolate only when scientifically justified); and final frames, contact sheet, video, periodic COGs, or Zarr cube. Sort by acquisition metadata, not filenames. Preserve timestamps and product IDs in metadata when practical. A separate manifest is optional—use it when requested or materially useful, not automatically for a simple GIF.

For an interactive timeline or map, publish COG or Zarr for a separate application. Processing must not generate frontend code or host the application.

## Make Frames Comparable

Reproject every observation to one CRS, extent, transform, width, and height. Use nearest for classes/QA and intentional continuous resampling. Verify registration around stable features before interpreting change.

Detect empty/partial frames rather than stretching missing data. Keep visualization stable: apply masks and calibration consistently, derive fixed limits from a baseline, representative sample, documented domain limits, or robust sequence-wide percentiles, and reuse band mapping, limits, gamma, tone/color transformations, colormap, legends, class colors, alpha, and nodata behavior. Per-frame stretching can create flicker, conceal change, or create false apparent change. Treat atmospheric, seasonal, or illumination normalization as an explicit, documented stage.

## Select And Composite

Common choices are lowest-cloud/highest-quality observation, median or robust statistic over valid pixels, most-recent valid pixel, domain-specific quality ranking, or a before/event/after set with comparable geometry and season. For period composites, mask invalid pixels before reduction. Scene-level cloud fraction is only a prefilter: assess the requested AOI from the quality layer because a mostly cloudy scene can contain a clear AOI and vice versa. Never call a product cloud-free without checking remaining spatial/temporal coverage, and document how dates are mixed.

## Output Formats

Use MP4 for broadly compatible video when codecs are available, WebM for suitable web delivery, GIF only for short low-color previews, and PNG/WebP/JPEG when a consumer assembles frames. Fix frame dimensions and pixel format. Make durations/timestamps explicit; do not silently display irregular acquisitions at equal spacing unless intentional. Publish the requested artifact and only useful support: a representative frame, optional ordered manifest, grid/style/missing-frame metadata, and checksums or sizes where practical.

## Verification

1. Confirm ordering and identical dimensions.
2. Inspect stable landmarks for jitter.
3. Check flicker from inconsistent stretches or masks.
4. Verify timestamps, labels, legends, and no-data rendering.
5. Decode/open the final artifact, not only intermediate frames.
6. If present, verify the manifest covers every frame and source product.
7. State limitations from clouds, gaps, mixed sensors, seasonality, or resolution.
