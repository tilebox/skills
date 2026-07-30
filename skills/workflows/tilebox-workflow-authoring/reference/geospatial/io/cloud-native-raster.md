# Cloud-Native Raster IO

Use range-readable GeoTIFF/COG access to avoid downloading whole rasters. `async-geotiff` with an `obstore` store is well suited to read-only windows and overviews.

## Best Uses

- Read-only COG/GeoTIFF windows and overview reads.
- Object storage and HTTP sources that support efficient ranges.
- Integration with `obstore` stores for S3, GCS, Azure, and compatible backends.

It is not a general warp/write layer.

```python
from async_geotiff import GeoTIFF, Window

geotiff = await GeoTIFF.open(object_key, store=store)
window = Window(col_off=x0, row_off=y0, width=x1 - x0, height=y1 - y0)
chunk = await geotiff.read(window=window)
array = chunk.data
mask = chunk.mask
```

`async-geotiff` can remain the efficient read layer when a later step reprojects or resamples the bounded array with `odc.geo`, Rasterio, or rioxarray; preserve the chunk CRS and transform when handing it off. Use Rasterio/GDAL or another suitable library for the actual warp, writing, update mode, JP2/HDF/NetCDF/GRIB, or other unsupported operations. Read windows or overviews rather than full rasters when partial access saves bytes. Download the whole file when it is not cloud optimized, most bytes are needed, a library requires a local path, range requests are poor, or repeated reads justify a local cache.

Bounds are `(west, south, east, north)`. Transform AOIs into the raster CRS before calculating pixel windows; never label longitude/latitude coordinates with a projected raster CRS. Keep compute near remote storage when possible, avoid unnecessary sidecar listings, log keys/window bounds/byte sizes for slow-read diagnosis, and bound concurrency to avoid connection or memory exhaustion.

References: async-geotiff documentation and GDAL/Rasterio VSI documentation.
