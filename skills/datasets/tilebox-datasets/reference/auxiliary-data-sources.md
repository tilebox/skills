# Auxiliary Data Sources

Use this reference for supporting data that is not indexed as a Tilebox dataset: elevation and terrain, weather and climate reanalysis, permanent-water or land masks, population, land cover, and similar global or regional grids.

## Decide Whether The Input Is Actually Auxiliary

- Product-coupled quality data such as Sentinel-2 SCL, Landsat QA bands, SAR incidence angle, and per-product masks belong to the selected source acquisition. Read and process them with that product.
- Independent grids such as DEMs, ERA5 weather fields, climatologies, and static land/water context are auxiliary sources. Select and access them separately.
- Treat an auxiliary variable as a scientific input: verify its meaning, units, reference surface or datum, spatial/temporal support, update policy, and license rather than choosing by filename alone.

## Prefer Cloud-Native Global Grids

For large global or multidimensional arrays, strongly prefer a suitable Zarr or Icechunk representation because a task can select only its AOI, time range, variables, and levels. Confirm that chunking fits the dominant access pattern; a cloud-native format with hostile chunks can still create excessive requests.

Open lazily, subset first, and compute only the bounded task region. Do not download an entire global store. Reproject or interpolate deliberately, preserve units and coordinates, and use nearest-neighbor for categorical masks. Keep auxiliary reads and the consuming product operation in the same task when their fanout axes match; write an intermediate only when a later task needs a different rechunking or fanout axis.

## Credentials-Free Terrain: Mapterhorn

[Mapterhorn](https://mapterhorn.com/data-access/) is a credentials-free elevation source optimized for terrain visualization. It publishes a global Copernicus GLO-30 baseline with higher-resolution regional sources where available as 512-pixel, lossless WebP terrain tiles in the standard XYZ Web Mercator grid. Elevation is encoded in metres using Terrarium RGB. No account or API key is required.

Prefer Mapterhorn for quick terrain visualization, hillshade, contours, and bounded local processing when avoiding provider setup matters more than using a single analysis-native grid. For Python AOI reads, the simplest path is usually the anonymous [XYZ endpoint and TileJSON](https://mapterhorn.com/data-access/#single-tile-http-endpoint), not direct PMTiles access:

1. Choose a zoom that matches the needed effective resolution and current local coverage, then enumerate the XYZ tiles intersecting the AOI.
2. Fetch `https://tiles.mapterhorn.com/{z}/{x}/{y}.webp`, decode each pixel as `R * 256 + G + B / 256 - 32768` metres, and mosaic the tiles on their EPSG:3857 bounds.
3. Crop to the exact AOI and reproject once to the workflow's target grid. Preserve the selected zoom, source version, and [required attribution](https://mapterhorn.com/attribution/) with the result.

This does not require downloading the full PMTiles archive. For larger or reusable extracts, use Mapterhorn's documented `pmtiles extract --bbox=...` flow; PMTiles uses HTTP range requests, but extracting, decoding, mosaicking, and reprojection is still more involved than selecting an AOI from a well-chunked Zarr or COG.

Do not treat Mapterhorn as a datum-harmonized scientific DEM. It combines source DEMs and DTMs with different native resolutions and inherited vertical references, is resampled to Web Mercator, and is primarily built for interactive maps. For terrain correction, hydraulic modeling, absolute-elevation comparison, or other datum-sensitive analysis, prefer an analysis-oriented Zarr/COG source such as Copernicus DEM and verify its vertical datum. Use Mapterhorn for such analysis only after confirming that the contributing source, resolution, attribution, and vertical reference over the AOI meet the requirement.

## Recommended Catalogues

Research all three catalogues when the user has not specified a source. Offer one default based on scientific and operational fit rather than combining mirrors automatically.

| Catalogue | Access model | Useful examples | Selection notes |
| --- | --- | --- | --- |
| [DestinE Earth Data Hub](https://earthdatahub.destine.eu/catalogue) | Most data requires a DestinE account and Earth Data Hub API key; some restricted Climate DT data requires upgraded access | Copernicus DEM GLO-30/GLO-90; ERA5 and ERA5-Land Zarr v3; CMIP6 and Climate DT grids | Strong default candidate for global Zarr auxiliaries when its quota, variables, resolution, and chunking fit |
| [Earthmover Marketplace](https://app.earthmover.io/marketplace) | Requires an Arraylake account, organization, and subscription; listings may be free or paid | Free and daily-updated ERA5 in Icechunk, plus weather/climate and Earth-observation listings | Prefer when Icechunk versioning/chunking or update cadence justifies an account/subscription |
| [Source Cooperative](https://source.coop/products) | Current public products are readable without an account through standard URLs; restricted or direct-cloud access can require authentication | Public Zarr, COG, GeoParquet, PMTiles, land cover, buildings, fields, weather, and other community products | Inspect each product's format, documentation, maintenance, and access mode; do not assume every product is a global array |

These catalogues change. Re-open the live product page before implementation and validate one bounded read rather than relying on remembered URLs, variables, groups, or chunks.

## DestinE Earth Data Hub Setup

Useful direct resources:

- [Catalogue](https://earthdatahub.destine.eu/catalogue)
- [Getting started, registration, API keys, and authenticated Zarr access](https://earthdatahub.destine.eu/getting-started)
- [DestinE/Earth Data Hub sign-in and registration](https://earthdatahub.destine.eu/login)
- [Earth Data Hub quota and API-key settings](https://earthdatahub.destine.eu/quota-api-keys#my-personal-access-tokens)
- [Copernicus DEM collection](https://earthdatahub.destine.eu/collections/copernicus-dem)
- [Copernicus DEM Global 30m Zarr](https://earthdatahub.destine.eu/collections/copernicus-dem/datasets/GLO-30)
- [Copernicus DEM Global 90m Zarr](https://earthdatahub.destine.eu/collections/copernicus-dem/datasets/GLO-90)
- [ERA5 collection](https://earthdatahub.destine.eu/collections/era5)

Guide the user to:

1. Register with the DestinE Platform through the getting-started flow.
2. Create or retrieve a Standard API Key from Earth Data Hub account settings. Explain that most datasets require authentication and the service currently applies a monthly request quota.
3. Store the key outside code, task inputs, logs, and release artifacts. Prefer the provider's documented `.netrc` setup or an approved runner-secret mechanism over embedding the key in a URL.
4. Install compatible Xarray/Zarr tooling and validate the provider's public test store first when useful.
5. Open the selected store lazily, then read one small AOI/variable/time slice before scaling out. Never attempt to download the complete global store.

Copernicus DEM is a digital surface model: GLO-30 and GLO-90 represent elevations including surface features and use metres above the geoid. Confirm that a DSM, its resolution, and its vertical reference fit the terrain operation. ERA5 and ERA5-Land are reanalyses, not direct local observations or forecasts; choose variables, levels, cadence, and spatial resolution accordingly.

## Earthmover Marketplace Setup

Useful direct resources:

- [Marketplace](https://app.earthmover.io/marketplace)
- [Marketplace data-user and subscription guide](https://docs.earthmover.io/marketplace/data-users)
- [Arraylake authentication and API-key guidance](https://docs.earthmover.io/setup/org-access)

Guide the user to:

1. Create or sign in to an Arraylake account and select or create an organization.
2. Subscribe to the chosen listing. Free listings can be subscribed to immediately; paid listings require terms and access from the provider.
3. Authenticate interactively for local work or create a narrowly scoped API key for an automated runner. Keep tokens in the runner secret environment, not workflow code or task inputs.
4. Open the subscribed read-only repository through Arraylake/Icechunk and validate one bounded Xarray/Zarr selection.

Earthmover currently offers ERA5 listings with different update cadences and access terms. Compare free versus paid freshness, included variables/levels, temporal coverage, and chunking before choosing.

## Source Cooperative Access

Useful direct resources:

- [Product catalogue](https://source.coop/products)
- [Source Cooperative documentation](https://docs.source.coop/)

Do not require account setup for a current public Source product: public data are designed for standard URL access and can be read anonymously. Inspect the product's Access Data instructions and repository documentation. If a product is restricted, a direct-cloud credential is required, or anonymous access is insufficient for the workload, guide the user through the access flow shown by that product and configure credentials through the runner's secret mechanism.

Source Cooperative hosts many formats, not only Zarr. Prefer a Zarr/Icechunk product for multidimensional global-grid access when suitable; use COG, GeoParquet, or another cloud-native format when it better matches the data model and task access pattern.

## Selection Checklist

Before authoring a workflow with auxiliary data, record:

1. Variable semantics, units, product type, and whether it is static, observed, modeled, forecast, or reanalysis data.
2. Spatial/temporal resolution, coverage, update cadence, CRS/grid, vertical levels or datum, and expected uncertainty.
3. Format, dimensions/groups, chunks, compression, fill values, and bounded-read pattern.
4. Account, subscription, quota, license, and credential requirements for local and remote runners.
5. Alignment, resampling, interpolation, and masking rules relative to the primary imagery.
6. One validated bounded read over a representative AOI/time/variable selection.
