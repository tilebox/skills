# Encoding, Compression, And Cloud Formats

Use this reference to choose output formats and encoding settings for geospatial workflow products.

## Format Decision Matrix

| Need | Prefer |
| --- | --- |
| Final 2D/3D raster product | COG |
| Distributed workflow intermediate | Zarr |
| Cloud-native multidimensional datacube | Zarr |
| Scientific interoperability with legacy/model tools | NetCDF |
| Vector/tabular geospatial output | GeoParquet/Parquet |
| Thumbnail, quicklook, preview image, or small static overview | PNG |
| Quick local scratch for one task | local temporary file or memory |

If the user simply asks for raster output, default to COG. If tasks need to rendezvous over shared array state, default to Zarr.

## Encoding Fields To Decide Explicitly

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

## Zarr

Use Zarr for shared workflow arrays and datacubes. Prefer direct `zarr` library writes and `obstore`-backed stores.

Choose chunks based on task read/write regions and common access patterns. Avoid both tiny chunks and huge chunks.

## COG

Use COG for final raster output. Choose compression based on dtype and semantics. Validate COG layout after writing.

## NetCDF

Use NetCDF only when the user or downstream tooling really requires NetCDF. If a workflow is choosing NetCDF only out of habit, ask whether the downstream tool can consume Zarr, COG, or another cloud-native format instead. Be careful with:

- HDF5 file locking in distributed environments
- chunk-based compression overhead
- limited cloud-native partial-write behavior compared with Zarr
- CF metadata expectations

## Compression Tradeoffs

- Faster compression may cost storage and transfer bytes.
- Stronger compression may cost CPU and slow reads.
- Lossy compression requires explicit user/product acceptance.
- Integer packing can save space but must preserve scale/offset metadata.
- Smooth continuous rasters often compress well; noisy data and embeddings may not.

## Anti-Patterns

- Changing dtype without recording scale/offset.
- Mixing variables with incompatible dtype/nodata in one GeoTIFF.
- Writing many tiny files/objects without a reason.
- Using NetCDF as a distributed partial-write rendezvous when Zarr fits better.
- Forgetting CRS/transform on final raster outputs.
