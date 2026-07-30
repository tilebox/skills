# Masking, Nodata, And QA

Use this reference when handling validity masks, nodata values, alpha bands, QA bit fields, cloud masks, or class rasters.

## Mask Types

- Nodata sentinel: a value marked invalid by raster metadata.
- Valid-data mask: often nonzero/255 means valid in GDAL/Rasterio.
- NumPy masked array: `True` in the mask means invalid.
- Alpha band: visual transparency and sometimes validity.
- `NaN`: useful for float arrays, not integer arrays.
- QA bit field: packed flags that need bit operations.
- Classification layer: integer classes such as Sentinel-2 SCL.

Be explicit about mask polarity when converting between APIs.

## Choosing Nodata

- Float data: `NaN` can be appropriate if downstream tools support it.
- Signed integer data: use a sentinel outside the valid range when possible.
- Unsigned/byte data: consider an internal mask if all values may be valid.
- Categorical data: reserve and document a code.
- QA/masks: preserve integer dtype and metadata.

Do not use `0` as nodata unless the product definition makes it safe.

## QA Bitmask Pattern

```python
cloud = (qa & (1 << cloud_bit)) != 0
shadow = (qa & (1 << shadow_bit)) != 0
valid = ~(cloud | shadow)
```

Keep QA arrays integer. Use nearest resampling only.

## Scene Classification Pattern

For class rasters, name the accepted classes and keep the class layer separate from derived masks:

```python
valid = scl.isin([2, 4, 5, 6, 11])
```

The class meanings and valid list are product-specific.

## Morphology

Use SciPy for local mask refinement on bounded arrays:

```python
from scipy import ndimage

cloud_buffer = ndimage.binary_dilation(cloud_mask, iterations=2)
small_objects = ndimage.label(valid_mask)
```

Keep morphology operations bounded to the processing chunk, and consider halos/overlap if edge effects matter.

## Reprojection And Masks

- Reproject categorical masks with nearest/mode.
- Set `dst_nodata` explicitly.
- Check dtype before using `NaN`.
- Validate after reprojection: count valid pixels, inspect min/max/classes, and verify expected nodata regions.

## Writing Outputs

- Set nodata in metadata or write an internal mask.
- Preserve alpha only when it is truly a display/validity channel.
- Avoid lossy compression for masks/classes.
- Document mask provenance in attrs/tags when derived from QA/cloud/shadow/snow/water logic.
