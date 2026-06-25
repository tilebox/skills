# Sentinel-2 Patterns

Use this reference for Sentinel-2 workflows, especially Copernicus archive products. If the user does not specify a different source, prefer the Copernicus archive/product layout used by Tilebox examples. There are COG alternatives for Sentinel-2 in public clouds; use those when the user asks for them or when COG-native access is clearly the better fit.

## Collections And Products

- L1C collections: top-of-atmosphere products.
- L2A collections: surface reflectance products with scene classification.
- Copernicus archive assets are commonly JP2 files inside SAFE-like product structures.
- Common L2A bands include B02, B03, B04, B08 at 10m and B05, B06, B07, B8A, B11, B12 at 20m.
- SCL is the Scene Classification Layer, commonly at 20m.

Check exact collection names and schema via Tilebox dataset inspection before coding.

## Query And Selection

Typical metadata filters:

- time interval
- AOI geometry
- collection names
- cloud cover
- processing baseline
- product ID/location

For multiple collections, query each collection or use the dataset API pattern supported by the current Tilebox SDK, then concatenate/sort by time.

## Copernicus JP2 Read Pattern

For Copernicus archive JP2 assets:

1. Locate the product paths for desired bands and SCL.
2. Read the JP2 asset with Rasterio/GDAL (`JP2OpenJPEG`) or a project-specific storage client.
3. Build a source grid from the asset transform/CRS.
4. Reproject to a shared target grid.
5. Write to Zarr as the workflow rendezvous.

`async-geotiff` is for GeoTIFF/COG reads; it is not the JP2 reader.

## COG Alternatives

Some Sentinel-2 mirrors expose COG assets. For COG sources:

- prefer `async-geotiff` with `obstore` for reads,
- use windowed reads where possible,
- avoid downloading full scenes when only an AOI/chunk is needed,
- confirm band naming, scale, nodata, and QA conventions because they may differ from Copernicus JP2 products.

## Reprojection And Resampling

Prefer `odc.geo` reprojection for the Copernicus JP2-to-target-grid pattern.

Choose resampling by variable:

- SCL/classes/QA: nearest.
- Reflectance bands: nearest is conservative and preserves original samples; bilinear/cubic may be appropriate for visualization or some analysis if explicitly chosen.

Do not use one method blindly across all products unless it is intentional.

## Masking And Mosaics

Common cloud-free mosaic shape:

1. Query matching granules.
2. Initialize a Zarr cube `(time, y, x)` per band/SCL.
3. Fan out product-to-Zarr tasks.
4. Fan out spatial mosaic chunks.
5. Build valid masks from SCL and data availability.
6. Reduce over time, e.g. a quantile or median of valid observations.
7. Emit final raster output as COG unless another format is requested.

SCL valid classes are product-specific. Document the selected classes in code/comments.

## Scale And Dtype

Sentinel-2 reflectance products often use integer scaled reflectance. Preserve integer dtype in intermediate Zarr when possible. Apply scale factors intentionally when creating visualization COGs or analysis outputs.

Do not generalize Sentinel-2 nodata, scale, band order, or class logic to other sensors.
