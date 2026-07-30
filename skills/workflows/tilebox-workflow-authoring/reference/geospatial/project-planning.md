# Earth Observation Project Planning

Translate an outcome into feasible data and validation requirements before choosing libraries or implementation architecture.

## Define The Outcome

- Product: raster, detections, statistics, image/video, mosaic, datacube, or alert.
- AOI and time: geometry, event/baseline windows, recurrence, and missing-period behavior.
- Scale: ground resolution, output CRS/grid, and geographic extent.
- Quality: uncertainty, confidence, false-positive tolerance, and validation.
- Delivery: COG, GeoParquet, Zarr, image/video, or another requested format.

Keep run-specific AOI, time, event, and output choices as root inputs rather than hardcoding them. Supply concrete coordinates, compact geometry, timestamps, and identifiers at execution time; store a large geometry separately and pass its key. Resolve named public places before execution and never guess a private AOI.

Keep interactive applications separate: processing publishes COG or Zarr; a separately built application consumes it.

Do not generate HTML, CSS, JavaScript, frontend bundles, web servers, or UI scaffolding in the processing pipeline. If both processing and a viewer are requested, deliver independently maintained components with separate dependencies and lifecycles.

## Distinguish Similar-Sounding Outcomes

- Wildfire can mean active fire, smoke, progression, burn scar, or burn severity.
- Flood can mean open-water extent, change from a baseline, depth estimation, or affected assets.
- Vegetation monitoring can mean greenness, crop type, stress, phenology, biomass, or disturbance.
- Change detection can mean visual comparison, categorical transition, continuous difference, or detected objects.
- Ship monitoring can mean presence, count, tracks, vessel class, or anomalous behavior.
- Disaster damage can mean visible structural change, accessibility, affected land cover, or a rapid qualitative overview.

## Generic Data Families

| Family | Strengths | Important limitations |
| --- | --- | --- |
| Optical multispectral | Natural color, indices, land cover, vegetation, burn scars, and many visual products | Clouds, illumination, seasonality, and spatial resolution |
| SAR | Day/night and cloud-tolerant acquisition, water/surface-change sensitivity, maritime work, and deformation with suitable products | Speckle, geometry effects, preprocessing consistency, and less intuitive interpretation |
| Thermal | Active fire, heat, and surface temperature | Coarser resolution, atmospheric effects, and revisit limits |
| Hyperspectral | Material discrimination and detailed spectral analysis | Volume, coverage, calibration, and model/data availability |
| Elevation/terrain | Orthorectification support, slope masks, drainage context, and terrain products | Acquisition date, resolution, vertical datum, and surface-versus-terrain differences |

## Feasibility Gates

1. The source resolves the target at the required scale.
2. Suitable temporal coverage and cadence exist.
3. Clouds, haze, snow, shadows, viewing geometry, seasonality, and gaps do not invalidate the comparison.
4. Observations can be aligned and normalized sufficiently.
5. Required bands, polarizations, quality layers, model weights, and calibration metadata exist.
6. Representative results can be inspected or measured. Without ground truth, label outputs as candidates or estimates.

Do not present a visual proxy as a direct physical measurement. An index is not automatically yield, water depth, burn severity, or structural damage.

## Product Expectations

- Environmental/agriculture: preserve dates, formulas, scale factors, masks, compositing rules, and units.
- Disaster response: include before/event/after windows, latency, confidence, and confounders.
- Maritime/object detection: emit georeferenced geometries, scores, model version, and overlap-deduplication details.
- Classification/segmentation: include class definitions, confidence, validation, and caveats.
- Imagery/time series: preserve stable styling, provenance, timestamps, and output grid.
- Urban/infrastructure: distinguish observed physical change from inferred construction, damage, or use.

Successful processing proves the pipeline ran, not that the scientific result is valid.
