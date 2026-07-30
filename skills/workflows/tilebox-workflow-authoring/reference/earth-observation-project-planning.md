# Earth Observation Project Planning

Use this reference when a user describes an Earth observation outcome rather than a specific processing pipeline. Translate the requested map, animation, detection set, index, or monitoring product into feasible data requirements and a Tilebox task graph before choosing libraries or writing substantial code.

## Define The Outcome First

Establish the smallest output that satisfies the request:

- **Product type:** raster map, vector detections, statistics, report, image, animation, video, mosaic, datacube, or alert.
- **AOI:** geometry, administrative region, asset boundary, or collection of independent AOIs.
- **Time:** event date, before/after windows, recurrence, or complete time-series interval.
- **Scale:** required ground resolution, output CRS/grid, and whether the product is regional or local.
- **Quality:** acceptable uncertainty, confidence thresholds, false-positive tolerance, and validation expectations.
- **Delivery:** COG, GeoParquet, Zarr, image/video, object-store prefix, dataset, or another requested format; add a manifest only when requested or useful enough to justify it. For an interactive map or web app, the workflow delivery is COG or Zarr data rather than application files.

Keep run-specific AOI, time, event, and output choices as root-task inputs instead of hardcoding them. Concrete values can be supplied at job submission after authoring. Submit technical values such as coordinates, small GeoJSON, timestamps, or identifiers; store a large geometry separately and submit its compact key. Resolve named public places before submission and never guess a private AOI.

## Choose Output Storage For The Execution Mode

Source imagery storage and workflow output storage are separate decisions. Public source access does not provide a destination for generated products.

| Execution mode | Output guidance |
| --- | --- |
| Notebook or small single-process prototype | A local output folder is appropriate when the user can retrieve the result from that machine. |
| Small workflow executed by one task on a local runner | Local output can be acceptable, but state that it is runner-local and not automatically durable. |
| Several independently scheduled tasks sharing frames, rasters, or other intermediates | Use shared object storage or another shared rendezvous; do not pass local paths between tasks. |
| Remote or multiple runners | Require user-controlled shared durable storage and the corresponding credentials before execution. |
| Transition from local to remote | Warn that local paths will not be shared or preserved and guide the user through output-storage setup before starting remote runners. |

Until Tilebox-hosted output storage exists, do not imply that a Tilebox API key provides an output bucket. For simple quickstarts, prefer a local result over forcing the user to configure a bucket. Keep the workflow structure honest: if fanout tasks must exchange large data, local-only output is no longer a safe simplification.

## Keep Interactive Applications Outside The Workflow

When the requested result includes a scrollable or zoomable map, dashboard, or small web application, split the implementation at the data boundary. Workflow tasks process and publish the COG or Zarr product. Build the application separately and configure it to read the workflow's durable output URL or object key as its input.

Do not make workflow tasks write HTML, CSS, JavaScript, frontend bundles, or application scaffolding. Do not embed a web server or UI build step in the task graph. If the user requests both processing and a viewer, deliver two separate components with independent dependencies and lifecycles; the web app consumes the workflow output but is not itself a workflow output.

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

When the user did not name a source, use the companion `tilebox-datasets` product-selection matrix and open-data catalog to choose one default, inspect it live, and explain provider access. For ordinary optical multispectral surface-reflectance work, prefer public `open_data.aws_earth.sentinel2` when its resolution, history, coverage, and cloud limitations fit; do not use operational convenience to override scientific requirements.

| Data family | Strengths | Important limitations |
| --- | --- | --- |
| Optical multispectral | Natural-color products, spectral indices, land cover, vegetation, burn scars, and many visual products | Clouds, illumination, seasonal effects, and spatial resolution |
| SAR | Day/night and cloud-tolerant acquisition, water and surface-change sensitivity, maritime applications, and deformation with suitable products | Speckle, geometry effects, preprocessing consistency, and less intuitive interpretation |
| Thermal | Active-fire, heat, and surface-temperature applications | Coarser resolution, atmospheric effects, and revisit limitations |
| Hyperspectral | Material discrimination and detailed spectral analysis | Data volume, coverage, calibration, and model/data availability |
| Elevation and terrain | Typically auxiliary global-grid data for orthorectification support, slope masks, drainage context, and terrain-derived products | Acquisition date, resolution, vertical datum, and surface-versus-terrain differences |

Inspect current Tilebox datasets, collections, schemas, and sample datapoints before relying on field names, asset layouts, spatial coverage, or temporal availability. For DEM, weather/climate, land masks, and other auxiliary global grids that are not indexed by Tilebox, use the `tilebox-datasets` auxiliary-data guide and prefer suitable Zarr or Icechunk sources. Check licensing, credentials, and model-weight access before designing around a private or restricted source.

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
2. Fan out by scene, AOI, time window, or another natural source axis. Within each task, keep the cloud-native read, masking, calibration, normalization, reprojection, and any matching visualization or analysis in memory when the bounded working set fits.
3. Write an intermediate only when a later stage needs a different fanout or rechunking scheme, or the data cannot safely remain task-local. For example, build monthly RGB mosaics by first fanning out over timestamps and writing spatially chunked RGB observations, then fanning out over spatial chunks to average valid pixels in parallel.
4. Fan out later analysis by spatial chunk, model tile, or independent AOI when that axis differs from source acquisition.
5. Aggregate results into the requested mosaic, time series, detections, statistics, or video.
6. Publish deterministic final outputs. Add a compact manifest with source IDs, timestamps, parameters, CRS/grid, and output locations only when the user requests one or it is easy and materially useful.
7. Validate at least one representative output and record limitations.

Avoid intermediate cache writes between operations that consume the same product and share a task boundary. When a cross-task rendezvous or rechunk is necessary, use object storage or Zarr for large intermediates and pass only compact IDs, object keys, cache keys, AOI references, or chunk coordinates in task inputs. Make writes deterministic and retry-safe.

## Output Expectations By Family

- **Environmental and agriculture:** preserve source dates, index formulas, scale factors, masks, compositing rules, and units.
- **Disaster response:** include before/event/after windows, acquisition latency, confidence, and known confounders.
- **Maritime and object detection:** emit georeferenced geometries, scores, model/version metadata, and overlap-deduplication details.
- **Urban and infrastructure:** distinguish observed physical change from inferred construction, damage, or use.
- **General imagery products:** preserve stable styling, source provenance, timestamps, and output grid details.
- **Classification and segmentation:** include class definitions, probabilities or confidence where available, and validation metrics or caveats.

Successful execution proves that the pipeline ran; it does not by itself prove that the Earth observation result is scientifically valid.
