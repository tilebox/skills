# Geospatial ML Inference Patterns

Use this reference for tiled model inference over rasters, embeddings, segmentation/classification, or change detection.

## Tilebox Fanout

Use one Tilebox task per model tile or spatial chunk. Task inputs should include:

- source Zarr/COG/object key
- output Zarr group/array or COG prefix
- chunk bounds
- CRS/resolution or enough information to reconstruct the target grid
- small model config values

Do not pass model weights, arrays, tensors, or xarray datasets as task inputs.

## Model Tile Design

- Match chunk size to model input size when fixed, e.g. 256x256 pixels.
- If the model emits patch embeddings, map input pixel chunks to output patch-grid slices deterministically.
- If neighborhood context matters, add halos and crop outputs before writing.
- Use deterministic output regions to make retries idempotent.

## Geospatial Context Features

Models may need:

- center lat/lon of each tile
- timestamp or representative time
- GSD/resolution
- CRS-aware position
- band wavelengths
- platform/sensor name

Compute these from the target grid and task chunk, not from global mutable state.

## Band Order And Normalization

- Validate band names/order against model metadata.
- Preserve integer/scaled values until the model preprocessing step.
- Convert to contiguous `float32` for PyTorch/NumPy model inputs when appropriate.
- Keep normalization constants in checked-in config or package data, not task inputs.

## Model Loading

Use lazy cached model loading:

```python
from functools import lru_cache

@lru_cache
def model():
    return load_model().to(device()).eval()
```

Optional runner preload is fine when startup latency is acceptable and memory/GPU capacity is known.

Large weights should be:

- fetched lazily into a deterministic runner-local cache, or
- stored in a private object bucket accessible to runners, or
- included only when intentionally part of the workflow project.

Do not store model weights in `job_cache` or task parameters.

## Outputs

Prefer Zarr for intermediate embeddings and dense model outputs. Write by deterministic region with the lower-level `zarr` library. Emit COGs for final raster products unless another output format is requested.

## Dependencies

Declare ML dependencies in `pyproject.toml`. For PyTorch, use uv sources/indexes and platform markers so `uv sync` works across intended CPU/GPU architectures.
