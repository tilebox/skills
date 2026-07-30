# Geospatial ML Inference

## Model Tile Design

Partition rasters into model-sized tiles (for example 256 × 256 when fixed). If a model emits patch embeddings, deterministically map pixel chunks to output patch-grid slices. Add halos when neighborhood context matters and crop outputs before stitching. Map every input tile to a deterministic output region so retries are safe.

## Geospatial Context Features

Models may require tile-center latitude/longitude, timestamp, GSD/resolution, CRS-aware position, wavelengths, or platform/sensor. Compute these from the target grid and tile, never mutable global state.

## Band Order And Normalization

Validate names/order against model metadata plus units, scale/offset, timestamp, GSD, and normalization. Preserve integer/scaled values until preprocessing and convert bounded inputs to contiguous `float32` when appropriate. Keep normalization constants in checked-in configuration or package data.

## Model Loading

Load models lazily and cache them per process:

```python
from functools import lru_cache

@lru_cache
def model():
    return load_model().to(device()).eval()
```

Optional process preload is reasonable when startup latency, memory, and accelerator capacity are known. Fetch large weights into a deterministic local cache or accessible object storage and validate checksums/sizes; intentionally bundle only suitable assets. Do not serialize weights or arrays through orchestration messages.

## Outputs And Dependencies

Prefer Zarr for dense intermediate embeddings and model outputs, using deterministic region writes, and COG for final raster predictions unless another format is requested. Vector/object results must preserve georeferencing and deterministic overlap deduplication. Preserve model/version, class definitions, confidence, CRS/grid, and normalization. Declare ML dependencies explicitly; architecture-specific PyTorch wheels need package indexes and platform markers appropriate to every intended runtime.
