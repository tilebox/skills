# NumPy And SciPy Raster Patterns

Use NumPy/SciPy for bounded in-task array math after Tilebox has partitioned the work into appropriate subtasks.

## Shape Discipline

Know the shape before computing:

- Rasterio: `(band, y, x)`.
- Image display: `(y, x, band)`.
- Feature matrix for ML/statistics: `(n_samples, n_features)`.
- xarray: named dimensions; inspect dims before `.to_numpy()`.

Convert explicitly:

```python
image = rasterio_arr.transpose(1, 2, 0)  # band,y,x -> y,x,band
features = image.reshape(-1, image.shape[-1])
```

## Dtype Discipline

- Keep QA/class rasters as integers.
- Use `float32` for many model inputs and continuous derived products unless precision requires `float64`.
- Prevent integer overflow before band math.
- Do not assign `NaN` into integer arrays.
- Clip before casting to unsigned integer outputs.

## Masked Math

```python
with np.errstate(divide="ignore", invalid="ignore"):
    ndvi = (nir - red) / (nir + red)
ndvi = np.where(valid & np.isfinite(ndvi), ndvi, np.nan)
```

For categorical counts:

```python
counts = np.bincount(classes[valid].ravel(), minlength=n_classes)
```

## Local Statistics For Reductions

For global statistics, compute compact per-chunk results and combine them with Tilebox aggregation tasks:

- valid pixel count
- sum / sum of squares
- min / max
- histograms
- class counts

## SciPy Patterns

Use `scipy.ndimage` for bounded local operations:

- `binary_dilation` / `binary_erosion` for mask buffers
- `label` for connected components
- convolution/filtering for local neighborhoods
- interpolation only when semantics support it

Consider halo/overlap between chunks if local operations depend on neighboring pixels.

## Avoid

- Python loops over pixels.
- Accidental `float64` expansion of large arrays.
- Chained expressions that allocate many full-size temporaries.
- Object dtype arrays.
- Global array loads inside a worker task.
- Dropping masks during `.astype(...)`.
