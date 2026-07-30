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
2. Select one datapoint, decode its canonical assets with `AssetCollection.from_datapoint(...)`, and inspect the available semantic keys.
3. Open required COG assets with `tilebox.storage.aio.Client` or the module-level `open_geotiff(...)`. The resolver prefers compatible asset locations, configures public S3 access unsigned, and reuses stores; do not construct S3 paths or stores manually.
4. Apply scale/offset and masks from inspected metadata deliberately.
5. Build a source grid from raster CRS/transform and reproject to a shared target grid when needed. Keep bounded read, masking, calibration, normalization, and matching visualization/analysis in one task when possible; write an intermediate only for a later rechunk or different fanout axis.

Small one-band pattern, given one selected datapoint and AOI bounds `(west, south, east, north)`:

```python
import numpy as np

from tilebox.datasets.assets import AssetCollection
from tilebox.storage.aio import open_geotiff
from tilebox.storage.geotiff import window_from_bounds

assets = AssetCollection.from_datapoint(datapoint)
asset = assets["red"]
geotiff = await open_geotiff(asset)
window = window_from_bounds(
    geotiff, bounds, crs="EPSG:4326", require_fully_contained=True
)
chunk = await geotiff.read(window=window)

raster = asset.raster
scale = raster.scale if raster is not None and raster.scale is not None else 1.0
offset = raster.offset if raster is not None and raster.offset is not None else 0.0
red = chunk.data[0].astype(np.float32) * scale + offset
```

Small concurrent RGB pattern using the same bounds:

```python
import asyncio

import numpy as np

from tilebox.datasets.assets import AssetCollection
from tilebox.storage.aio import open_geotiff
from tilebox.storage.geotiff import window_from_bounds

assets = AssetCollection.from_datapoint(datapoint)
rgb_assets = [assets["red"], assets["green"], assets["blue"]]
geotiffs = await asyncio.gather(*(open_geotiff(asset) for asset in rgb_assets))

reference = geotiffs[0]
for geotiff in geotiffs[1:]:
    if (
        geotiff.crs != reference.crs
        or geotiff.transform != reference.transform
        or geotiff.width != reference.width
        or geotiff.height != reference.height
    ):
        raise ValueError("RGB assets do not share the same pixel grid")

window = window_from_bounds(
    reference, bounds=bounds, crs="EPSG:4326", require_fully_contained=True
)
chunks = await asyncio.gather(
    *(geotiff.read(window=window) for geotiff in geotiffs)
)

channels = []
for asset, chunk in zip(rgb_assets, chunks, strict=True):
    raster = asset.raster
    scale = raster.scale if raster is not None and raster.scale is not None else 1.0
    offset = raster.offset if raster is not None and raster.offset is not None else 0.0
    channels.append(chunk.data[0].astype(np.float32) * scale + offset)

rgb = np.stack(channels, axis=-1)
```

GeoTIFF support is included in `tilebox-storage`. The new client is async-only and is imported from `tilebox.storage.aio`; there is no `tilebox.storage.Client`. `open_geotiff(...)` does not read pixels, align bands, or apply catalog scale/offset, nodata, or masks. Check `chunk.mask` and SCL explicitly. Before stacking RGB or indices, verify that opened assets share CRS, transform, width, and height, or reproject them deliberately.

Cloud-mask one native 20m image band with the 20m Scene Classification Layer. SCL classes 3, 8, 9, and 10 represent cloud shadow, medium-probability cloud, high-probability cloud, and thin cirrus respectively:

```python
import asyncio

import numpy as np

from tilebox.datasets.assets import AssetCollection
from tilebox.storage.aio import open_geotiff
from tilebox.storage.geotiff import window_from_bounds

assets = AssetCollection.from_datapoint(datapoint)
image_asset = assets["swir16"]
scl_asset = assets["scl"]
image_geotiff, scl_geotiff = await asyncio.gather(
    open_geotiff(image_asset), open_geotiff(scl_asset)
)

if (
    image_geotiff.crs != scl_geotiff.crs
    or image_geotiff.transform != scl_geotiff.transform
    or image_geotiff.width != scl_geotiff.width
    or image_geotiff.height != scl_geotiff.height
):
    raise ValueError("Reproject SCL to the image grid with nearest resampling")

window = window_from_bounds(
    image_geotiff, bounds=bounds, crs="EPSG:4326", require_fully_contained=True
)
image_chunk, scl_chunk = await asyncio.gather(
    image_geotiff.read(window=window), scl_geotiff.read(window=window)
)

raster = image_asset.raster
scale = raster.scale if raster is not None and raster.scale is not None else 1.0
offset = raster.offset if raster is not None and raster.offset is not None else 0.0
image = image_chunk.data[0].astype(np.float32) * scale + offset
scl = scl_chunk.data[0]
cloud_or_shadow = np.isin(scl, [3, 8, 9, 10])
cloud_masked = np.where(cloud_or_shadow, np.nan, image)
```

Use the product's actual SCL class definitions and adapt the excluded classes to the application—for example, also exclude class 11 when snow or ice is invalid. Do not apply SCL directly to a differently sized band; align it to the image grid with nearest-neighbor resampling first.

Do not ask the user to create AWS credentials for this public source. Public input access does not provide output storage; use local output only for suitable notebook/local execution and require shared storage for remote or distributed work.

## Authenticated Copernicus Archive Alternative

For `open_data.copernicus.sentinel2_msi` archive products:

1. Explain that the Tilebox API key covers metadata, not Copernicus product downloads.
2. Guide the user through Copernicus Data Space registration and S3 credential generation using the direct links in the `tilebox-datasets` Copernicus provider guide.
3. Read explicitly documented environment variables and pass them to the deprecated `CopernicusStorageClient`; never hardcode, log, or serialize credentials in task inputs.
4. Use `CopernicusStorageClient.download(...)` with the selected datapoint, then locate desired bands and SCL inside the downloaded product and read JP2 files with Rasterio/GDAL (`JP2OpenJPEG`).
5. Treat this provider client as a temporary compatibility exception: the legacy Copernicus datasets are not yet compatible with `AssetCollection`.
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
