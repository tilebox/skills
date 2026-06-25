# Landsat 8/9 Patterns

Use this reference for Landsat 8/9 Operational Land Imager / Thermal Infrared Sensor (OLI/TIRS) workflows. If the user does not specify a different source, prefer the USGS Landsat Collection 2 cloud archive exposed by Tilebox; Landsat Collection 2 Level-2 products are delivered as Cloud Optimized GeoTIFFs, so prefer `async-geotiff` with `obstore` for reads when applicable.

## Tilebox Datasets And Collections

Tilebox exposes separate datasets for Landsat 8 and Landsat 9:

- `open_data.usgs.landsat8_oli_tirs`
- `open_data.usgs.landsat9_oli_tirs`

Common collections:

- `L1`: Level-1 products.
- `L2_SR`: Level-2 Surface Reflectance datapoints.
- `L2_ST`: Level-2 Surface Temperature datapoints.

Query both Landsat 8 and 9 when the workflow wants the combined OLI/TIRS time series. Query `L2_SR` for reflectance workflows and `L2_ST` for temperature workflows; a product with correction `L2SP` includes both SR and ST, while `L2SR` indicates surface reflectance only.

## Tilebox Metadata Fields

Sample Tilebox datapoints include fields such as:

- `granule_name`, e.g. `LC09_L2SP_088238_20250601_20250602_02_T2_SR`
- `location`, e.g. `s3://usgs-landsat/collection02/level-2/standard/oli-tirs/.../<ProductID>`
- `platform`: `LANDSAT_8` or `LANDSAT_9`
- `instrument`: `OLI_TIRS`
- `collection_category`: usually `T1` or `T2`
- `collection_number`: `2`
- `correction`: `L2SP` or `L2SR`
- `cloud_cover` and `cloud_cover_land`
- `wrs_path`, `wrs_row`, `wrs_type`
- `proj_epsg`, `proj_shape`, `proj_transform`
- `thumbnail` and `overview` URLs

Use `location` as the object prefix for product assets. The product ID is the final path component without the `_SR`/`_ST` suffix from `granule_name`.

## Product Assets And Band Names

For Landsat 8/9 Collection 2 Level-2 Surface Reflectance, common COG assets are:

- `SR_B1`: coastal aerosol
- `SR_B2`: blue
- `SR_B3`: green
- `SR_B4`: red
- `SR_B5`: near infrared (NIR)
- `SR_B6`: SWIR 1
- `SR_B7`: SWIR 2
- `QA_PIXEL`: pixel quality bit field
- `QA_RADSAT`: radiometric saturation bit field
- `SR_QA_AEROSOL`: aerosol QA bit field

For Surface Temperature products, common assets include:

- `ST_B10`: surface temperature
- `ST_QA`: surface temperature uncertainty
- `ST_CDIST`: distance from cloud
- `ST_ATRAN`, `ST_EMIS`, `ST_EMSD`, `ST_TRAD`, `ST_URAD`, `ST_DRAD`: intermediate ST layers

USGS asset file names are usually `<ProductID>_<asset>.TIF`, for example `<ProductID>_SR_B4.TIF` or `<ProductID>_QA_PIXEL.TIF` under the datapoint `location` prefix.

## Reading COG Assets

Because Collection 2 assets are COGs, prefer `async-geotiff` and `obstore` for band/window reads:

```python
from async_geotiff import GeoTIFF, Window
from obstore.store import S3Store

store = S3Store(bucket="usgs-landsat", prefix="collection02/level-2/standard/oli-tirs")
gtiff = await GeoTIFF.open(product_relative_key, store=store)
tile = await gtiff.read(window=Window(col_off=x0, row_off=y0, width=w, height=h))
arr = tile.data
```

If code starts from a Tilebox `location` such as `s3://usgs-landsat/collection02/.../<ProductID>`, split it into bucket, prefix, and asset filename. Keep object-store region/data locality in runner deployment rather than task code.

Use Rasterio/GDAL/odc-geo instead when you need reprojection/warping, writing, or APIs not available in `async-geotiff`.

## Grid And Reprojection

Landsat Collection 2 Level-2 products are typically delivered on a 30 m UTM or Polar Stereographic grid, with projection metadata available in Tilebox as `proj_epsg`, `proj_shape`, and `proj_transform`.

Avoid reprojection when all selected scenes/assets are already aligned for the analysis. If combining scenes across WRS path/row, UTM zones, or sensors creates misaligned grids, choose a target grid early and use `odc.geo` reprojection. If only the final delivery product needs Web Mercator or another output CRS, compute on the native/aligned grid first and reproject as the final publication step.

## Scale Factors And Dtype

Collection 2 Level-2 Surface Reflectance uses unsigned 16-bit integer values with fill value `0`, valid range `1..65455`, and scale/offset:

```text
reflectance = DN * 0.0000275 - 0.2
```

Collection 2 Surface Temperature uses unsigned 16-bit integer values with fill value `0` and scale/offset:

```text
temperature_kelvin = DN * 0.00341802 + 149.0
```

Preserve integer dtype in Zarr intermediates when possible and apply scale factors intentionally at analysis/output boundaries. Do not apply reflectance and temperature scaling interchangeably.

## QA_PIXEL Cloud Mask Pattern

For Landsat 8/9 `QA_PIXEL`, common bits include:

- bit 0: fill
- bit 1: dilated cloud
- bit 2: high-confidence cirrus
- bit 3: high-confidence cloud
- bit 4: high-confidence cloud shadow
- bit 5: high-confidence snow
- bit 6: clear
- bit 7: water

Example valid-data mask:

```python
fill = (qa & (1 << 0)) != 0
dilated_cloud = (qa & (1 << 1)) != 0
cirrus = (qa & (1 << 2)) != 0
cloud = (qa & (1 << 3)) != 0
shadow = (qa & (1 << 4)) != 0
snow = (qa & (1 << 5)) != 0

valid = ~(fill | dilated_cloud | cirrus | cloud | shadow | snow)
```

Adjust the mask for the application: water may be valid for aquatic workflows, snow may be valid for snow/ice workflows, and `QA_RADSAT` should be checked when saturation affects the analysis. Keep QA arrays as integer dtype and use nearest/mode resampling only.

## Common Workflow Shape

1. Query Landsat 8 and/or Landsat 9 Tilebox datasets for `L2_SR` or `L2_ST` over an AOI/time interval.
2. Filter by `collection_category`, `cloud_cover`/`cloud_cover_land`, WRS path/row, or platform when needed.
3. Use `location` to construct COG asset keys for required bands and QA layers.
4. Fan out Tilebox tasks by scene, band/product, or spatial chunk.
5. Read COG windows with `async-geotiff`/`obstore` when no reprojection is needed; otherwise use `odc.geo`/Rasterio path.
6. Write shared intermediates to Zarr when tasks need to rendezvous.
7. Emit final raster products as COGs unless another output format is requested.

## References

- USGS Landsat Collection 2 Level-2 Science Products.
- USGS Landsat Collection 2 Quality Assessment Bands.
- Landsat 8-9 Collection 2 Level 2 Science Product Guide.
- Landsat Cloud Optimized GeoTIFF Data Format Control Book.
