# Geospatial Raster Fundamentals

Use this reference before turning geospatial rasters into plain arrays, writing derived rasters, or combining products on a shared grid.

## Raster Data Is Array Values Plus Spatial Metadata

A NumPy array alone is not a geospatial raster. Preserve or reconstruct:

- CRS / projection
- affine transform
- bounds / extent
- pixel resolution
- width, height, band count
- dtype
- nodata value or valid-data mask
- scale/offset and units
- band names/descriptions
- time/product metadata when relevant

Common shape conventions:

- Rasterio reads/writes band-first arrays: `(band, y, x)`.
- Image display code often uses channel-last arrays: `(y, x, band)`.
- xarray/rioxarray often uses named dimensions such as `("band", "y", "x")` or `("time", "y", "x")`.
- Zarr arrays should declare dimension names when the chosen Zarr version/API supports them.

When converting arrays between conventions, keep the transform and CRS paired with the correct `y, x` dimensions.

## Pixel Semantics Matter

Choose operations based on what values mean:

- Continuous measurements: reflectance, temperature, elevation, precipitation.
- Categorical classes: land cover, scene classification, QA classes.
- Bit fields: QA masks, cloud flags, validity flags.
- Visualization values: RGB/RGBA bytes after scaling/stretching.

Categorical and QA rasters must not be interpolated with bilinear/cubic resampling. Preserve integer dtype for class codes and bit fields.

## Nodata And Valid Zero Are Different

Do not assume `0` is invalid. It is product-specific.

- Some reflectance products use `0` as fill/nodata.
- Many scientific variables can have valid zeros.
- Float data may use `NaN` where downstream tools support it.
- Integer rasters need a sentinel value, an internal mask, or an alpha band.

Always set nodata/mask metadata when writing, not just array values.

## Georeferencing Pitfalls

- Longitude/latitude order is usually x/y. Use `always_xy=True` with pyproj transformations where appropriate.
- EPSG:4326 degrees are not meters.
- Pixel resolution changes with latitude in geographic CRS.
- Bounds in one CRS cannot be used directly as bounds in another CRS without transformation.
- A bbox transformed by corners only may be insufficient for highly curved projections or antimeridian/polar cases.
- Reprojecting changes the grid; it is not just changing the CRS tag.

Tip: when testing or prototyping new workflows, use a non-square grid/AOI first. Different `y` and `x` sizes surface axis swaps, width/height mixups, and row/column mistakes much earlier than square test data.

## Before Writing A Derived Raster

Check:

1. Does the output array shape match the target grid?
2. Are dimensions ordered the way the writer expects?
3. Is CRS correct and explicit?
4. Is transform correct for the target grid?
5. Is dtype intentional and large enough?
6. Is nodata/mask explicit and valid for the dtype?
7. Are scale/offset/units preserved or updated?
8. Are band names/descriptions meaningful?

References: GDAL raster data model, Rasterio transforms/windows/masks docs, rioxarray and odc-geo geospatial metadata docs.
