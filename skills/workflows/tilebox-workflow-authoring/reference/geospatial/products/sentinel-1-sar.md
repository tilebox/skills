# Sentinel-1 And SAR Product Semantics

## Product Choice

GRD products are detected projected amplitude products used for backscatter, flood, and maritime work. SLC preserves phase and is required for interferometry, coherence, and deformation. Choose mode, polarization, resolution, and coverage from the outcome. Do not design deformation around GRD or assume an algorithm transfers unchanged across sensors, modes, resolution, or polarization. Confirm units/domain, calibration, terrain correction, and whether data are raw or analysis-ready.

## Comparability And Preprocessing

For change analysis, prefer matching relative orbit, pass direction, mode, polarization, incidence-angle range, target grid, and preprocessing. Regridding does not remove viewing-geometry or radiometric differences.

Apply one documented preprocessing chain: as needed, orbit updates, border/thermal-noise handling, radiometric calibration, terrain correction, speckle treatment, unit conversion, and reprojection. Do not mix amplitude, intensity/power, sigma/gamma nought, linear, and decibel values accidentally; perform ratios, thresholds, differences, and averaging in their intended domain. Speckle filtering can remove small/narrow targets. Optical cloud-mask and reflectance assumptions do not apply.

Use a DEM and terrain masks where slope, radar shadow, or layover matter. Check resolution, acquisition date, vertical datum, and whether an elevation layer describes surface or bare terrain. Auxiliary-source selection and access belong outside this product-semantics reference.

## Flood And Maritime

Flood mapping should select compatible pre/event acquisitions, process consistently, align one grid, derive change/water probability, suppress false positives with permanent-water and terrain/geometry context plus suitable morphology/minimum units, and validate samples. Open water often has low backscatter, but wind, vegetation, urban structure, wet soil, smooth surfaces, shadow, and layover confound it. Do not infer water depth from a simple mask.

Ships may be bright, but coasts, platforms, waves, ice, sidelobes, and artifacts cause false positives. Maritime workflows should use appropriate land/coast masks, overlapping windows, georeferenced detections, and deterministic overlap deduplication. Confirm that source resolution can resolve target vessel sizes.

## Interferometry

InSAR needs compatible SLC pairs plus orbit/baseline constraints, coregistration, interferogram formation, filtering, unwrapping, atmospheric/terrain corrections, and a reference point/area. Use a proven processor; narrow the claim when requirements or validation are unavailable.

## Verification

- Confirm product, domain/units, polarization, orbit, pass, and processing chain.
- Inspect representative coast, urban, mountain, open-water, and no-data regions relevant to the AOI.
- Verify alignment before interpreting change and test thresholds/models across multiple images where possible.
- Preserve source times/IDs, geometry metadata, algorithm parameters, confidence, and limitations.
