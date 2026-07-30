# Tilebox Assets

`AssetCollection` decodes canonical assets from exactly one selected datapoint:

```python
from tilebox.datasets.assets import AssetCollection

assets = AssetCollection.from_datapoint(datapoint)
red = assets["red"]
```

Use semantic keys such as `red`, `green`, `blue`, `scl`, or product-specific keys exposed by the inspected datapoint. Treat explicit metadata as authoritative: locations, media type, raster shape/grid, nodata, scale, offset, units, and band semantics. Do not reconstruct or normalize hrefs.

Automatic discovery normally finds the relevant xarray variables by type. When a datapoint contains ambiguous fields, use the unified `fields=` escape hatch and specify only the required variable names:

```python
assets = AssetCollection.from_datapoint(
    datapoint,
    fields={"assets": "other", "storage": "second"},
)
```

Apply scale and offset explicitly after reading:

```python
raster = asset.raster
scale = raster.scale if raster is not None and raster.scale is not None else 1.0
offset = raster.offset if raster is not None and raster.offset is not None else 0.0
values = raw.astype("float32") * scale + offset
```

If a workflow intentionally overrides a field, keep the override near asset selection and document why; do not mutate shared catalog assumptions. Byte access is owned by [storage access](storage-access.md).
