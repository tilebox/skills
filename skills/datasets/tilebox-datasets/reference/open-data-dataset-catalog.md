# Tilebox Open-Data Dataset Catalog

Use this catalog as a curated shortlist, not as a substitute for the live Tilebox API. Always run `tilebox dataset list --json`, `tilebox dataset get <slug> --json`, and a small AOI/time query before relying on a slug, collection, field, asset layout, or coverage range.

Tilebox indexes metadata and asset locations for these datasets. The payload provider hosts the product bytes and may require separate credentials or requester-pays access.

## Recommended And Common Sources

| Tilebox dataset | Mission/product | Payload provider and common format | Access | Best for | Important limitations | Provider guide |
| --- | --- | --- | --- | --- | --- | --- |
| `open_data.aws_earth.sentinel2` | Sentinel-2 MSI L2A; live collection `L2A` | Element 84 AWS Earth Search; public COG assets with S3 and HTTPS locations | Public unsigned; no source-provider account | Default optical multispectral workflows, timelapses, mosaics, indices, vegetation, land cover, burn scars | Clouds; 10/20/60 m bands; validate scale/offset, SCL/QA, coverage, and target resolution | `providers/aws-earth-search.md` |
| `open_data.aws_earth.sentinel1` | Sentinel-1 SAR Level-1 GRD; live collection `GRD` | Element 84 AWS Earth Search; public polarization COGs and product metadata with S3 and HTTPS locations | Public unsigned; no source-provider account or requester-pays | Default Sentinel-1 amplitude/backscatter workflows, including flood/change mapping and maritime analysis when GRD fits | GRD only: no SLC phase for interferometry/coherence/deformation; validate mode, orbit, polarization, units, preprocessing, and geometry | `providers/aws-earth-search.md` |
| `open_data.copernicus.sentinel2_msi` | Sentinel-2 MSI archive collections | Copernicus Data Space; commonly mission-native SAFE/JP2 products | Copernicus account and generated S3 credentials | Explicit Copernicus archive/product-layout needs and whole-product access | Auth setup and larger product downloads; do not prefer over AWS COGs for simple optical workflows | `providers/copernicus-data-space.md` |
| `open_data.copernicus.sentinel1_sar` | Sentinel-1 SAR GRD, SLC, OCN, RAW, and some COG collections | Copernicus Data Space; product format depends on collection | Copernicus account and generated S3 credentials | SLC-based InSAR/coherence/deformation, OCN, RAW, or an explicitly required Copernicus product/layout | Legacy source; prefer credentials-free `open_data.aws_earth.sentinel1` when GRD is sufficient | `providers/copernicus-data-space.md` |
| `open_data.copernicus.sentinel3_slstr` | Sentinel-3 SLSTR including FRP, LST, WST, and radiance products | Copernicus Data Space; mission-native products | Copernicus account and generated S3 credentials | Active fire, fire radiative power, land/water surface temperature | Coarser spatial resolution and product-specific semantics | `providers/copernicus-data-space.md` |
| `open_data.copernicus.sentinel3_olci` | Sentinel-3 OLCI | Copernicus Data Space; mission-native products | Copernicus account and generated S3 credentials | Ocean/land color and broad-area spectral products | Atmospheric correction and product semantics; not a fine-resolution default | `providers/copernicus-data-space.md` |
| `open_data.copernicus.sentinel5p_tropomi` | Sentinel-5P TROPOMI | Copernicus Data Space; mission-native atmospheric products | Copernicus account and generated S3 credentials | Atmospheric composition and trace-gas products | Coarse footprint and column/product semantics; not local surface concentration | `providers/copernicus-data-space.md` |
| `open_data.usgs.landsat8_oli_tirs` | Landsat 8 OLI/TIRS; live `L1`, `L2_SR`, `L2_ST` collections | USGS Landsat Collection 2 in AWS S3; COG assets | AWS credentials; `usgs-landsat` is requester-pays | Modern long-running optical/thermal record, 30 m surface reflectance, surface temperature | Requester-pays charges, 30 m multispectral resolution, 16-day single-satellite revisit | `providers/usgs-landsat.md` |
| `open_data.usgs.landsat9_oli_tirs` | Landsat 9 OLI/TIRS; live `L1`, `L2_SR`, `L2_ST` collections | USGS Landsat Collection 2 in AWS S3; COG assets | AWS credentials; `usgs-landsat` is requester-pays | Modern Landsat surface reflectance and temperature, commonly paired with Landsat 8 | Requester-pays charges and 30 m multispectral resolution | `providers/usgs-landsat.md` |
| Other live `open_data.usgs.landsat*` datasets | Landsat 1–7 MSS/TM/ETM archives | USGS-hosted products; inspect the selected dataset and collection | Usually provider/AWS access requirements; verify before use | Historical optical records predating Landsat 8 | Sensors, bands, resolution, calibration, gaps, and product availability differ by mission | `providers/usgs-landsat.md` |
| `open_data.asf.ers_sar` | Historical ERS-1 and ERS-2 SAR collections | Alaska Satellite Facility product downloads | NASA Earthdata Login/ASF authentication | Historical ERS SAR analysis from 1991–2011 | Not a Sentinel-1 substitute; mission-specific geometry and processing | `providers/alaska-satellite-facility.md` |
| Other live `open_data.asf.*` datasets | ASF-indexed SAR missions as they become available | Alaska Satellite Facility | NASA Earthdata Login/ASF authentication | SAR archives when the mission, product, and period match | Never infer an ASF slug or product; discover and inspect it live | `providers/alaska-satellite-facility.md` |

## Limited-Coverage Credentials-Free Sources

These are current examples, not an exhaustive list: discover the live catalog because additional credentials-free datasets may be added. The public datasets below require no source-provider credentials, but contain curated, opportunistic, sample, or time-limited coverage rather than dependable global acquisition histories. Use them when the user asks about the named provider, mission, or product and a live query confirms coverage. Do not select them as defaults for generic products such as cloud-free mosaics, arbitrary-AOI monitoring, or broad historical analysis.

| Tilebox dataset | Public product | Live collections | Selection note |
| --- | --- | --- | --- |
| `open_data.capella.sar` | Capella X-band SAR | `CPHD`, `CSI`, `GEC`, `GEO`, `SICD`, `SIDD`, `SLC` | Curated public acquisitions; choose the product deliberately. |
| `open_data.iceye.sar` | ICEYE SAR scenes and interferometric products | `SAR` | Public open-data selection, not systematic coverage. |
| `open_data.planet.tanager` | Planet Tanager-1 hyperspectral imagery | `hyperspectral` | Small public hyperspectral sample; verify place, time, and product level. |
| `open_data.satellogic.earth_view` | Satellogic EarthView 1 m optical tiles | `L1` | Worldwide sample limited to July–December 2022; not a current monitoring source. |
| `open_data.umbra.sar` | Umbra high-resolution X-band SAR | `SAR` | Public selected acquisitions; verify the requested AOI and date. |
| `open_data.wyvern.dragonette` | Wyvern Dragonette L2A hyperspectral imagery | `standard`, `extended` | Opportunistic open-data scenes, not a systematic global time series. |

## Access Classes

- **Public unsigned:** no source-provider credentials; still requires Tilebox authentication for Tilebox metadata/workflow operations. Prefer this for simple quickstarts when the product requirements fit.
- **Provider-authenticated:** the user must create an external account/credential and make it available to the process reading product bytes.
- **Requester-pays:** credentials and a billable cloud account are required; explain potential request and transfer charges before access.
- **Restricted/commercial:** do not assume entitlement. Explain the requirement and stop before source access until the user confirms access.

## Choosing Among Multiple Suitable Datasets

Rank candidates by scientific fit first, then operational friction:

1. Correct modality and product semantics.
2. Sufficient spatial resolution and coverage.
3. Suitable history, revisit, latency, and observation quality.
4. Required bands, polarizations, QA, calibration, and validation support.
5. Payload format and efficient access for the intended AOI.
6. Credential, requester-pays, licensing, egress, and deployment complexity.

Credentials-free access must not override scientific incompatibility. Conversely, do not require provider accounts or whole-product downloads when the public AWS Sentinel-2 L2A or Sentinel-1 GRD source fully satisfies the request.
