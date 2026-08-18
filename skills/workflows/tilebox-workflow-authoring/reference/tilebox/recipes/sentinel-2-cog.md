# Sentinel-2 COG Recipe

Use `tilebox-datasets` to confirm the source and live schema. The established default for ordinary public L2A COG work is the AWS Earth Search-backed source; this recipe assumes one already-selected datapoint with semantic assets.

RGB is typically 10 m while SCL is typically 20 m. Build a bounded window on each grid, read the bands concurrently, and reproject SCL to the RGB window/grid with nearest-neighbor resampling:

```python
import asyncio

import numpy as np
from rasterio.enums import Resampling
from rasterio.warp import reproject
from tilebox.datasets.assets import AssetCollection
from tilebox.storage.aio import Client
from tilebox.storage.geotiff import window_from_bounds

assets = AssetCollection.from_datapoint(datapoint)
rgb_assets = [assets["red"], assets["green"], assets["blue"]]
scl_asset = assets["scl"]
storage = Client()

rgb_geotiffs, scl_geotiff = await asyncio.gather(
    asyncio.gather(*(storage.open_geotiff(asset) for asset in rgb_assets)),
    storage.open_geotiff(scl_asset),
)
reference = rgb_geotiffs[0]
if any(
    (item.crs, item.transform, item.width, item.height)
    != (reference.crs, reference.transform, reference.width, reference.height)
    for item in rgb_geotiffs[1:]
):
    raise ValueError("RGB assets must share a pixel grid")

rgb_window = window_from_bounds(
    reference, bounds=bounds, crs="EPSG:4326", require_fully_contained=True
)
scl_window = window_from_bounds(
    scl_geotiff, bounds=bounds, crs="EPSG:4326", require_fully_contained=True
)
rgb_chunks, scl_chunk = await asyncio.gather(
    asyncio.gather(*(item.read(window=rgb_window) for item in rgb_geotiffs)),
    scl_geotiff.read(window=scl_window),
)

channels = []
for asset, chunk in zip(rgb_assets, rgb_chunks, strict=True):
    raster = asset.raster
    scale = raster.scale if raster is not None and raster.scale is not None else 1.0
    offset = raster.offset if raster is not None and raster.offset is not None else 0.0
    channels.append(chunk.data[0].astype(np.float32) * scale + offset)
rgb = np.stack(channels, axis=-1)

scl_on_rgb = np.zeros(rgb.shape[:2], dtype=scl_chunk.data[0].dtype)
reproject(
    source=scl_chunk.data[0],
    destination=scl_on_rgb,
    src_transform=scl_chunk.transform,
    src_crs=scl_geotiff.crs,
    dst_transform=rgb_chunks[0].transform,
    dst_crs=reference.crs,
    resampling=Resampling.nearest,
    src_nodata=0,
    dst_nodata=0,
)
valid = np.isin(scl_on_rgb, [2, 4, 5, 6, 11])
masked_rgb = np.where(valid[..., None], rgb, np.nan)
```

The positive valid-class list is application-specific. Inspect current SCL definitions and decide whether water, snow/ice, dark pixels, or unclassified pixels are valid. SCL class `0` is treated as nodata during reprojection. Also combine source read masks/nodata when present.

The temporary legacy Copernicus archive exception is documented under [storage access](../storage-access.md): use it only when explicitly required and hand provider credential setup to `tilebox-datasets`.
