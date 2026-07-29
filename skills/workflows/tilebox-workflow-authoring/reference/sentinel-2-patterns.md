# Sentinel-2 Patterns

Use this reference for Sentinel-2 workflows. If the user does not specify a source and Sentinel-2 fits the requested product, default to Tilebox dataset `open_data.aws_earth.sentinel2`, collection `L2A`. It indexes Element 84 AWS Earth Search public COG assets, which require no source-provider credentials and support efficient window/range reads.

Use `open_data.copernicus.sentinel2_msi` only when the user explicitly requests Copernicus Data Space, needs original archive/SAFE/JP2 products, or another Copernicus-specific requirement outweighs the simpler public COG path. Tilebox indexes metadata and asset locations for both sources; it does not host the source imagery bytes.

For target-product selection and provider setup, use the companion `tilebox-datasets` skill and its product matrix, open-data catalog, AWS Earth Search guide, or Copernicus Data Space guide.

## Collections And Products

- L1C collections: top-of-atmosphere products.
- L2A collections: surface reflectance products with scene classification.
- `open_data.aws_earth.sentinel2` exposes an `L2A` collection with public COG assets. Verify the live collection and schema before coding.
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

Query `open_data.aws_earth.sentinel2` with collection `L2A` for the default path. Inspect sample datapoints for canonical STAC assets, storage profiles, S3/HTTPS locations, band keys, scale/offset, nodata, and QA conventions. Do not reconstruct paths from product IDs when explicit asset locations are available.

## Default AWS Earth Search COG Pattern

For `open_data.aws_earth.sentinel2`:

1. Query `L2A` metadata by AOI, time, and suitable queryable quality fields such as cloud cover.
2. Read canonical asset locations from the returned metadata.
3. **TODO(storage API):** Update the concrete accessor/store choice and COG read pattern after the in-progress Tilebox Python storage API design is finalized.
4. Apply scale/offset and masks from inspected metadata deliberately.
5. Build a source grid from raster CRS/transform and reproject to a shared target grid when needed. Keep bounded read, masking, calibration, normalization, and matching visualization/analysis in one task when possible; write an intermediate only for a later rechunk or different fanout axis.

Do not ask the user to create AWS credentials for this public source. If a client attempts signed access, configure unsigned/anonymous access instead. Public input access does not provide output storage; use local output only for suitable notebook/local execution and require shared storage for remote or distributed work.

## Authenticated Copernicus Archive Alternative

For `open_data.copernicus.sentinel2_msi` archive products:

1. Explain that the Tilebox API key covers metadata, not Copernicus product downloads.
2. Guide the user through Copernicus Data Space registration and S3 credential generation using the direct links in the `tilebox-datasets` Copernicus provider guide.
3. Read credentials from the local/runner environment and pass them to `CopernicusStorageClient`; never hardcode, log, or serialize them in task inputs.
4. Locate desired bands and SCL inside the selected product.
5. Read JP2 with Rasterio/GDAL (`JP2OpenJPEG`) or use the provider-specific storage client for required whole/partial product access.
6. Validate one authenticated product read before bulk work.

`async-geotiff` is for GeoTIFF/COG reads; it is not the JP2 reader. Credentials configured locally do not automatically reach remote runners.

## Reprojection And Resampling

Prefer `odc.geo` reprojection when it fits the data model. Rasterio/GDAL remains appropriate for JP2, warping, writing, or operations unsupported by the COG-native read path.

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
