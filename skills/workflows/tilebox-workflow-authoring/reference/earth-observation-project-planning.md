# Earth Observation Project Planning

Use this reference when a user describes an Earth observation outcome rather than a specific processing pipeline. Translate the requested map, animation, detection set, index, or monitoring product into feasible data requirements and a Tilebox task graph before choosing libraries or writing substantial code.

## Define The Outcome First

Establish the smallest output that satisfies the request:

- **Product type:** raster map, vector detections, statistics, report, image, animation, video, mosaic, datacube, or alert.
- **AOI:** geometry, administrative region, asset boundary, or collection of independent AOIs.
- **Time:** event date, before/after windows, recurrence, or complete time-series interval.
- **Scale:** required ground resolution, output CRS/grid, and whether the product is regional or local.
- **Quality:** acceptable uncertainty, confidence thresholds, false-positive tolerance, and validation expectations.
- **Delivery:** COG, GeoParquet, Zarr, image/video, manifest, object-store prefix, dataset, or another requested format.

Ask only when a missing value materially changes feasibility or the workflow. Never guess a private AOI such as a home address. If a public event is named but its date or extent is ambiguous, inspect available authoritative metadata or ask a narrow question.

## Distinguish Similar-Sounding Outcomes

Clarify scientific intent before selecting a sensor or algorithm:

- Wildfire can mean active fire, smoke, fire progression, burn scar, or burn severity.
- Flood can mean open-water extent, change from a baseline, depth estimation, or affected assets.
- Vegetation monitoring can mean greenness, crop type, stress, phenology, biomass, or disturbance.
- Change detection can mean visual comparison, categorical transition, continuous difference, or detected objects.
- Ship monitoring can mean presence, count, tracks, vessel class, or anomalous behavior.
- Disaster damage can mean visible structural change, accessibility, affected land cover, or a rapid qualitative overview.

Do not present a visual proxy as a direct physical measurement. For example, a spectral index is not automatically crop yield, flood-water depth, burn severity, or structural damage.

## Select Data From Requirements

Choose sources based on the requested outcome rather than habit:

| Data family | Strengths | Important limitations |
| --- | --- | --- |
| Optical multispectral | Natural-color products, spectral indices, land cover, vegetation, burn scars, and many visual products | Clouds, illumination, seasonal effects, and spatial resolution |
| SAR | Day/night and cloud-tolerant acquisition, water and surface-change sensitivity, maritime applications, and deformation with suitable products | Speckle, geometry effects, preprocessing consistency, and less intuitive interpretation |
| Thermal | Active-fire, heat, and surface-temperature applications | Coarser resolution, atmospheric effects, and revisit limitations |
| Hyperspectral | Material discrimination and detailed spectral analysis | Data volume, coverage, calibration, and model/data availability |
| Elevation and terrain | Orthorectification support, slope masks, drainage context, and terrain-derived products | Acquisition date, resolution, vertical datum, and surface-versus-terrain differences |

Inspect current Tilebox datasets, collections, schemas, and sample datapoints before relying on field names, asset layouts, spatial coverage, or temporal availability. Check licensing, credentials, and model-weight access before designing around a private or restricted source.

## Apply Feasibility Gates

Before implementation, verify:

1. **Spatial resolution:** the target is large enough to be resolved. A medium-resolution source may support field or neighborhood analysis but not an individual vehicle, small vessel, or detailed building condition.
2. **Temporal coverage:** suitable before/event/after acquisitions exist and revisit frequency supports the requested cadence.
3. **Observation quality:** cloud, haze, snow, shadows, SAR geometry, seasonal change, or missing acquisitions do not invalidate the intended comparison.
4. **Comparability:** scenes can be aligned to a common CRS/grid and normalized sufficiently for the requested analysis.
5. **Algorithm inputs:** required bands, polarizations, QA layers, model weights, reference data, and calibration metadata are available.
6. **Validation:** there is a credible way to inspect or quantify representative results. If no ground truth exists, label outputs as candidates or estimates rather than verified facts.

If a gate fails, adapt the product, source, AOI, time window, or stated confidence. Do not hide the limitation behind a successful workflow run.

## Map The Product To Tilebox Stages

A common project shape is:

1. Query and select source scenes or products for the AOI and time range.
2. Fan out downloads or cloud-native reads by scene, asset, AOI, or time window.
3. Apply sensor-specific masking, calibration, normalization, and reprojection to a shared grid.
4. Fan out analysis by scene, spatial chunk, model tile, or independent AOI.
5. Aggregate scene/chunk results into the requested mosaic, time series, detections, statistics, or video.
6. Publish deterministic final outputs plus a compact manifest containing source IDs, timestamps, parameters, CRS/grid, and output locations.
7. Validate at least one representative output and record limitations.

Use object storage or Zarr for large intermediates and pass only compact IDs, object keys, cache keys, AOI references, or chunk coordinates in task inputs. Make writes deterministic and retry-safe.

## Output Expectations By Family

- **Environmental and agriculture:** preserve source dates, index formulas, scale factors, masks, compositing rules, and units.
- **Disaster response:** include before/event/after windows, acquisition latency, confidence, and known confounders.
- **Maritime and object detection:** emit georeferenced geometries, scores, model/version metadata, and overlap-deduplication details.
- **Urban and infrastructure:** distinguish observed physical change from inferred construction, damage, or use.
- **General imagery products:** preserve stable styling, source provenance, timestamps, and output grid details.
- **Classification and segmentation:** include class definitions, probabilities or confidence where available, and validation metrics or caveats.

Successful execution proves that the pipeline ran; it does not by itself prove that the Earth observation result is scientifically valid.
