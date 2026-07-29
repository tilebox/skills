# Earth Observation Product Selection

Use this reference when the user names a target product but not a dataset. Start from observation requirements, choose one default source, validate it against the live Tilebox catalog and a small AOI/time sample, and mention alternatives only when their tradeoffs matter.

## Selection Rules

1. Identify the required modality: optical multispectral, SAR, thermal, ocean/land color, atmospheric, elevation, or another observation family.
2. Determine required product level, bands or polarizations, spatial resolution, revisit, history, latency, cloud tolerance, QA, and validation.
3. Choose one default from the matrix below and inspect its entry in `open-data-dataset-catalog.md`.
4. Verify the exact slug, collections, fields, coverage, and asset access through the live Tilebox API.
5. Reject a default when its resolution, temporal coverage, modality, access requirements, or product semantics do not fit.
6. Do not silently combine sensors. Offer multi-source processing only when its added history, cadence, or robustness justifies harmonization work.

## Product Matrix

| Target product | Observation requirements | Default | Alternatives | Choose an alternative when | Avoid the default when |
| --- | --- | --- | --- | --- | --- |
| Cloud-free optical timelapse, mosaic, natural color, or false color | Optical surface reflectance, cloud/QA information, repeat acquisitions | `open_data.aws_earth.sentinel2` `L2A` | USGS Landsat 8/9 `L2_SR`; `open_data.copernicus.sentinel2_msi` when Copernicus archive products are explicitly needed | A longer Landsat record, 30 m continuity, or a specific archive layout matters | Fine object/building detail is required, clouds prevent suitable observations, or the target predates the archive |
| NDVI or related vegetation-index time series | Red/NIR and sometimes red-edge/SWIR surface reflectance, consistent scaling and masks | `open_data.aws_earth.sentinel2` `L2A` | Landsat 8/9 `L2_SR` | Longer history matters more than 10–20 m Sentinel-2 detail; a joint series is explicitly worth harmonizing | The requested variable is not measurable from reflectance alone |
| Crop, forest, land-cover, burn-scar, or surface-change mapping | Multispectral reflectance, QA, suitable seasonal/event observations | `open_data.aws_earth.sentinel2` `L2A` | Landsat 8/9 `L2_SR`; SAR as complementary evidence | Historical baseline, cloud tolerance, or complementary structure/moisture information matters | The request means active fire, deformation, or cloud-obscured event mapping rather than optical surface change |
| Long historical optical record | Stable multi-decade archive and comparable surface-reflectance products | Appropriate `open_data.usgs.landsat*` collections; use Landsat 8/9 `L2_SR` for modern work | Sentinel-2 as a recent higher-resolution addition | Recent 10–20 m detail is more important than archive length | The user expects Sentinel-2 resolution or a credentials-free source |
| Flood or open-water extent under clouds | Cloud-tolerant event/baseline observations, comparable orbit/polarization and terrain handling | Suitable GRD collection in `open_data.copernicus.sentinel1_sar` | Sentinel-2 or Landsat as clear-sky context; another live SAR dataset when appropriate | Clear optical imagery exists and visual interpretation is sufficient, or another SAR archive better fits the period | A simple low-backscatter threshold cannot handle terrain, vegetation, urban areas, wind, or permanent water without validation |
| Surface deformation or interferometry | Compatible SAR SLC pairs, orbit/baseline/phase processing | A suitable SLC collection in `open_data.copernicus.sentinel1_sar` | Historical ASF SAR sources when the period and processing stack fit | Historical mission coverage is required | Only GRD/amplitude products are available or no validated InSAR processing path exists |
| Ship or maritime-object detection | Suitable SAR or high-resolution optical data, target-resolving pixels, coast/land masks, detector validation | Choose a suitable live SAR collection after checking target size and product resolution | Optical imagery as supporting context; ASF datasets when the indexed mission and period fit | Target size, sea state, coastline, archive period, or model requirements favor another source | Sentinel-2 or another source cannot resolve the target sizes reliably |
| Active fire, fire radiative power, or land-surface temperature | Thermal observations and the correct derived product | Relevant collections in `open_data.copernicus.sentinel3_slstr` such as FRP or LST, after live inspection | Landsat `L2_ST` for different spatial/temporal needs; optical imagery for later burn scars | Finer spatial detail or post-event surface mapping matters more than rapid/coarser thermal observation | The user requested a burn scar or visual timelapse rather than active heat |
| Ocean/land color or water-quality proxy | Suitable spectral bands, atmospheric correction, water/land product semantics | Relevant collection in `open_data.copernicus.sentinel3_olci` after live inspection | Sentinel-2 for smaller inland/coastal areas when its bands and processing fit | Higher spatial detail matters more than OLCI coverage/cadence | The requested quantity cannot be inferred reliably from available reflectance |
| Atmospheric composition | Mission-specific atmospheric products and vertical/column semantics | Relevant collection in `open_data.copernicus.sentinel5p_tropomi` after live inspection | Other atmospheric datasets when explicitly available and better suited | The requested constituent, cadence, or resolution requires another instrument | The user expects local surface concentrations or fine spatial detail unsupported by the product |

## Sentinel-2 Is The Optical Default, Not A Universal Default

Choose `open_data.aws_earth.sentinel2` without asking the user to select a source when all are true:

- The outcome uses ordinary optical multispectral surface reflectance.
- Sentinel-2's 10/20/60 m bands, temporal coverage, and cloud limitations are sufficient.
- The user did not request a provider, product layout, longer historical archive, or different modality.
- Credentials-free public COG access is beneficial for the intended workflow.

For a satellite timelapse of a house, explain that Sentinel-2 can show the surrounding 3×3 km area, vegetation, land cover, and larger changes, but generally not fine roof-level detail.

Landsat 8/9 can be offered as an optional addition for a longer or denser multi-mission sequence. Do not enable it by default: align grids and resolution, harmonize reflectance and spectral response, normalize styling, and preserve sensor provenance before mixing frames.

## Selection Summary

Before authoring, communicate:

- Selected Tilebox dataset and collection.
- Why its modality, resolution, history, and access model fit.
- What Tilebox indexes versus where product bytes live.
- Source format and authentication/requester-pays requirements.
- Live metadata sample and source access actually validated.
- Relevant limitations and optional alternatives.
