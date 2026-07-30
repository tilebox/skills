# Encoding, Compression, And Cloud Formats

Use this reference to choose geospatial output formats and encoding settings.

## Format Decision Matrix

| Need | Prefer |
| --- | --- |
| Final 2D raster or fixed-band image product | COG |
| Distributed workflow intermediate | Zarr |
| Cloud-native multidimensional datacube | Zarr |
| Scientific interoperability with legacy/model tools | NetCDF |
| Vector/tabular geospatial output | GeoParquet/Parquet |
| Thumbnail, quicklook, preview image, or small static overview | PNG |
| Quick local scratch | local temporary file or memory |

If the user simply asks for raster output, default to COG. For shared multidimensional array state, default to Zarr.

Follow [COG output](cog-output.md) for raster writer details and [Zarr arrays](zarr.md) for schema, chunk, and region-write guidance. Use NetCDF when the user or downstream tooling requires it rather than as a distributed partial-write rendezvous.

## Preserve Essential Encoding Semantics

- dtype
- fill value / nodata
- scale factor and offset
- units
- band or variable names
- dimension names
- chunks
- compressor/codec
- CRS/transform/geospatial metadata
- time/calendar encoding where relevant

Do not change dtype or integer packing without preserving scale/offset, combine variables with incompatible dtype/nodata semantics in one raster, or omit CRS/grid metadata from geospatial outputs. Lossy encoding requires explicit acceptance for the requested product.
