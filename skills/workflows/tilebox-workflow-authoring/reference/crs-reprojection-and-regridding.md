# CRS, Reprojection, And Regridding

Prefer `odc.geo` reprojection for Tilebox geospatial workflows when it fits the data model. Use Rasterio/GDAL, rioxarray, xESMF, or other tools when the data format or operation is better suited to them.

## Concepts

- CRS/projection: how coordinates map to Earth.
- Source grid: original raster transform, shape, CRS, resolution.
- Target grid: desired output shape, transform, CRS, resolution.
- Reprojection/regridding: mapping values from a source grid to a chosen target grid. This may include changing CRS, changing resolution/alignment, or using conservative/area-weighted methods for climate/model data.

## Target Grid Design

Avoid reprojection when the source data is already aligned across time/products and the native grid supports the analysis. Compute on the aligned source grid first; if a web or delivery output needs a different CRS, reproject as a final publication step.

If source data is not already aligned, choose a target grid early so all downstream tasks write/read consistent regions. Choose target CRS/resolution intentionally:

- EPSG:3857 for web visualization.
- Native CRS to preserve source geometry and avoid unnecessary resampling.
- Local projected CRS for regional analysis.
- Equal-area CRS for area-weighted statistics.
- A reference product grid when aligning multiple products.

For AOIs supplied in lon/lat, transform bounds to the target CRS before building the grid. Use `Transformer.from_crs(..., always_xy=True)` when using pyproj directly.

## odc.geo Pattern

The common Tilebox pattern is:

1. Read source array and create a source `GeoBox` from shape, transform, and CRS.
2. Wrap the array with spatial metadata.
3. Load or build a target `GeoBox`.
4. Reproject with `dataset.odc.reproject(...)`.

Sketch:

```python
from odc.geo.geobox import GeoBox
from odc.geo.xr import wrap_xr
from rasterio.enums import Resampling

src_grid = GeoBox(shape=arr.shape, affine=src.transform, crs=src.crs)
da = wrap_xr(xr.DataArray(arr, dims=("y", "x")), gbox=src_grid)
out = da.odc.reproject(how=target_grid, resampling=Resampling.nearest, dst_nodata=0)
```

Check exact APIs against installed `odc-geo` and xarray versions.

## Resampling Choices

- Nearest: categorical rasters, QA bands, masks, bit fields.
- Mode: categorical downsampling when supported.
- Bilinear/cubic/lanczos: continuous imagery when interpolation is acceptable.
- Average: aggregation/downsampling of continuous values.
- Conservative/area-weighted: climate/model grids where totals/fluxes should be preserved.

Do not blindly use one resampling method for every variable. If variables have different semantics, reproject them separately or group by compatible semantics.

## When Not To Use odc.geo

Use another library when:

- the operation is pure COG window reading (`async-geotiff`),
- the source format needs GDAL-specific handling,
- the workflow needs complex warping options,
- the data is a curvilinear/rotated climate grid,
- conservative regridding is required,
- or existing project code already uses a reliable specialized path.

## Pitfalls

- EPSG:4326 degrees are not meters.
- Antimeridian and polar AOIs need special care.
- Reprojecting masks with bilinear/cubic corrupts classes.
- Reprojecting integer data with a float nodata can fail or silently cast.
- Target bounds should be snapped when repeated jobs must align exactly.
