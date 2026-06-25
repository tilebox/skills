# Cloud Optimized GeoTIFF Outputs

If the user asks for raster output and does not specify a format, produce Cloud Optimized GeoTIFFs (COGs) by default. Use Zarr for distributed workflow intermediates and datacubes when it makes sense; for small compact intermediates, `context.job_cache` or pickled cache values can also be enough. Use COGs for final 2D/3D raster products intended for cloud access, visualization, or broad geospatial compatibility.

## COG Is Appropriate For

- final raster products
- RGB/RGBA visualizations
- single-scene or mosaic rasters
- per-band scientific rasters with shared dtype/grid
- HTTP range-readable distribution

Prefer Zarr instead for high-dimensional datacubes, many variables with mixed dtypes/nodata semantics, frequent partial updates, or analysis-first multidimensional outputs.

## Requirements

A valid COG needs:

- internal tiling
- internal overviews for large rasters
- range-read friendly layout
- correct CRS and transform
- correct nodata/mask/alpha metadata
- appropriate compression

Do not call a plain tiled GeoTIFF a COG without validation.

## Writing Tools

Use one of:

- GDAL COG driver
- `rio-cogeo`
- Rasterio/GDAL write followed by COG conversion

`async-geotiff` is read-only; do not use it for writing.

If the COG should be embedded directly in a web map, make that an explicit output requirement and create a web-optimized COG aligned to the Web Mercator tile grid where appropriate. Some tools expose this as a web-optimized flag, such as `rio-cogeo --web-optimized`, or GDAL COG options such as `TILING_SCHEME=GoogleMapsCompatible`.

## Compression Defaults

- General scientific integer rasters: `ZSTD` or `DEFLATE`, with predictor 2 when appropriate.
- Floating-point rasters: `ZSTD`/`DEFLATE`, predictor 3 when supported and beneficial.
- RGB visualization: JPEG/WebP can be appropriate if lossy output is acceptable; otherwise use lossless options.
- Floating-point lossy tolerance: consider LERC/LERC_ZSTD only when controlled error is acceptable and compatibility is understood.

Use 256 or 512 internal tiles unless access patterns justify another size. Match web tile size only when the product is intentionally web-optimized.

## Resampling For Overviews

- Continuous values: average, bilinear, or another semantically appropriate method.
- Categorical/classes/QA/masks: nearest or mode.
- RGB visualization: average/bilinear can be acceptable after visual scaling.

## Nodata And Masks

- Set nodata/mask metadata explicitly.
- Avoid lossy compression with nodata sentinels that can bleed into valid pixels.
- Prefer internal masks/alpha for visualization products when appropriate.
- Validate that nodata remains correct after conversion.

## Validate

Run a COG validator when possible:

```bash
rio cogeo validate output.tif
```

Also inspect CRS, transform, bounds, dtype, band descriptions, nodata/masks, overviews, and file size.

References: Cloud-Native Geospatial COG guide, cogeo.org developer guide, GDAL COG/GTiff docs, rio-cogeo docs.
