# Sentinel-1 And SAR Patterns

Use this reference for Sentinel-1 or other synthetic-aperture radar workflows, especially flood mapping, maritime and ship detection, surface-change analysis, and deformation. SAR is cloud-tolerant and works day or night, but geometry and preprocessing differences can easily create false change.

## Choose The Product For The Outcome

- **GRD** products are detected, projected amplitude products commonly used for flood extent, backscatter change, and maritime detection.
- **SLC** products preserve phase and are required for interferometry, coherence, and deformation workflows.
- Choose acquisition mode, polarization, spatial resolution, and coverage from the analysis requirements.

Do not design an interferometric/deformation workflow around GRD data. Do not assume a ship or flood algorithm transfers unchanged between sensors, modes, resolutions, or polarizations.

Inspect current Tilebox collection names, fields, product locations, and sample datapoints before coding. Confirm whether source assets are raw, calibrated, terrain-corrected, analysis-ready, linear power, amplitude, or decibels.

## Keep Acquisitions Comparable

For change analysis, prefer acquisitions with compatible:

- Relative orbit and pass direction.
- Acquisition mode and product type.
- Polarization.
- Spatial resolution and target grid.
- Incidence-angle range and viewing geometry.
- Preprocessing chain and calibration convention.
- Seasonal and environmental conditions where practical.

Opposite look directions, different orbits, terrain shadows, layover, incidence-angle differences, and inconsistent preprocessing can look like surface change. Preserve these attributes in output provenance when useful.

Reprojection to a common map grid aligns output coordinates, but it does not remove SAR viewing-geometry or radiometric differences. For quantitative pixel-level time-series or change analysis, strongly prefer the same relative orbit/track, pass direction, mode, and polarization. Different geometries can be combined only when the method explicitly models or normalizes incidence angle and look-direction effects, masks shadow/layover, and is validated for that use; simple regridding alone is insufficient.

## Preprocess Deliberately

Use an established processor or documented analysis-ready source for the required operations. Depending on source and outcome, preprocessing may include orbit updates, border or thermal-noise handling, radiometric calibration, terrain correction, speckle treatment, conversion between linear and decibel units, and reprojection.

Rules:

- Apply the same processing chain to every acquisition in a comparison.
- Keep track of whether values represent amplitude, intensity/power, sigma nought, gamma nought, or another convention.
- Perform ratios, differences, thresholds, and averaging in the intended numeric domain; do not mix linear and decibel values accidentally.
- Treat speckle filtering as an explicit tradeoff because it can remove small or narrow targets.
- Use a suitable DEM and terrain masks in areas where slope, shadow, or layover matters. A cloud-native option is Earth Data Hub's [Copernicus DEM Global 30m Zarr](https://earthdatahub.destine.eu/collections/copernicus-dem/datasets/GLO-30); guide users through DestinE registration and API-key setup with the `tilebox-datasets` auxiliary-data reference before authenticated access.
- Never apply optical cloud-mask or reflectance assumptions to SAR data.

## Flood Mapping Pattern

A common flood workflow is:

1. Select compatible pre-event and event acquisitions, ideally with the same relative orbit, direction, mode, and polarization.
2. Apply a consistent calibration and terrain-correction chain.
3. Align both acquisitions to one grid.
4. Compute event backscatter, temporal change, or a model-derived water probability.
5. Suppress likely false positives using permanent-water context, terrain slope, radar shadow/layover information, connected-component or morphology rules, and minimum mapping units as appropriate.
6. Validate against representative reference data or manual samples.
7. Publish extent or probability, acquisition metadata, thresholds/model version, and limitations.

Open water often has low radar backscatter, but wind, vegetation, urban structure, wet soil, smooth non-water surfaces, and radar geometry can violate that assumption. Label unvalidated results as candidate or estimated flood extent rather than ground truth. Depth estimation requires additional terrain/hydrologic methods and should not be inferred from a simple water mask.

## Maritime And Ship Detection Pattern

Ships may appear as bright targets against relatively dark open water, but coastlines, platforms, waves, sea ice, sidelobes, and processing artifacts also create detections.

A common pipeline is:

1. Select suitable mode, polarization, resolution, and incidence-angle range.
2. Calibrate and terrain-correct consistently where required by the chosen method.
3. Apply a land/coast mask appropriate to the requested near-shore behavior.
4. Fan out inference or detector windows with enough overlap for edge targets.
5. Convert pixel detections to georeferenced geometries.
6. Merge overlap duplicates with deterministic NMS or another documented rule.
7. Publish scores, detector/model version, acquisition metadata, and known false-positive conditions.

Check whether the source resolution can resolve the target vessel sizes. A successful detector cannot recover objects below the information content of the imagery.

## Interferometry And Deformation

Interferometric workflows require SLC pairs and additional constraints such as compatible orbit geometry, baseline, phase handling, coregistration, interferogram formation, filtering, unwrapping, atmospheric/terrain corrections, and a reference point or area. Use a proven InSAR processing stack rather than implementing these steps casually from amplitude examples.

Treat deformation as a specialized workflow. If exact processing requirements, reference data, or validation are unavailable, narrow the task or ask for domain guidance rather than presenting an unverified displacement product.

## Tilebox Task Graph

Useful fanout axes include acquisition, burst/swath, AOI, raster chunk, and detector tile. Keep source product IDs, orbit/polarization metadata, object keys, and chunk coordinates in compact task inputs. Store aligned intermediates in object storage or Zarr only when a later stage needs a cross-task rendezvous or different chunking axis.

For pairwise change, use a root task to select compatible acquisitions, parallel preprocessing tasks per acquisition when needed, parallel change/inference tasks per chunk, and an aggregation task for deduplication, validation summaries, and final COG/vector outputs. Keep preprocessing and analysis together in memory when they share the same bounded task axis.

## Verification

- Confirm product type, units/domain, polarization, orbit, pass direction, and preprocessing chain.
- Inspect representative coast, urban, mountain, open-water, and no-data regions relevant to the AOI.
- Verify alignment before interpreting differences.
- Check thresholds or model behavior across more than one image when possible.
- Preserve source IDs, acquisition times, geometry metadata, algorithm parameters, and confidence/limitations with the output.
