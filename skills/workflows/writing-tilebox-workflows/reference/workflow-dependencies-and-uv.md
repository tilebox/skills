# Workflow Dependencies And uv

All runtime dependencies for a Tilebox workflow project must be declared in `pyproject.toml` so `uv sync` can recreate the environment on runners.

Do not rely on ad-hoc `pip install`, notebook state, undeclared local packages, or packages installed manually on one machine.

## Basic Rules

- Put runtime dependencies in `[project].dependencies`.
- Put lint/type/test tools in dependency groups such as `[dependency-groups].dev`.
- Keep Python version constraints realistic for all intended runners.
- Run `uv sync` after dependency changes.
- Validate imports and runner startup after sync.
- Avoid adding heavy dependencies when a smaller library fits.

## Geospatial Defaults

Add only what the workflow uses, but prefer:

- `obstore` for object storage.
- `async-geotiff` for COG/GeoTIFF reads.
- `zarr` for direct Zarr writes.
- `xarray` for labeled reads from Zarr/NetCDF-like data.
- `odc-geo` for common geospatial reprojection/grid handling.
- `rasterio` when writing, warping, or reading non-GeoTIFF GDAL formats.
- `niquests` for non-storage HTTP APIs.

## PyTorch And Architecture-Specific Wheels

PyTorch requires care across CPU/GPU platforms and architectures. Do not add a naive dependency and assume `uv sync` will work everywhere.

Use `tool.uv.sources` and explicit indexes with platform markers appropriate for the workflow's intended runners. Example shape:

```toml
[tool.uv.sources]
torch = [
  { index = "pytorch-cu128", marker = "sys_platform == 'linux' and platform_machine == 'x86_64'" },
  { index = "pytorch-cpu", marker = "sys_platform != 'linux' or platform_machine != 'x86_64'" },
]
torchvision = [
  { index = "pytorch-cu128", marker = "sys_platform == 'linux' and platform_machine == 'x86_64'" },
  { index = "pytorch-cpu", marker = "sys_platform != 'linux' or platform_machine != 'x86_64'" },
]

[[tool.uv.index]]
name = "pytorch-cu128"
url = "https://download.pytorch.org/whl/cu128"
explicit = true

[[tool.uv.index]]
name = "pytorch-cpu"
url = "https://download.pytorch.org/whl/cpu"
explicit = true
```

Adjust CUDA version, CPU fallback, OS markers, and architecture markers to match the deployment target. Verify on each target runner class.

## Runtime Assets

Large assets such as model checkpoints, static rasters, lookup tables, and calibration files should not be passed through task inputs or `job_cache`.

Prefer:

- lazy download into a deterministic runner-local cache,
- private object storage accessed via `obstore`,
- or explicit package data only when intentionally bundled.

Validate checksums or file sizes before using cached assets.

## Verification

After dependency edits, run the narrowest useful checks:

```bash
uv sync
uv run python -c "import your_workflow_module"
```

For release projects, also run the relevant Tilebox workflow build/release validation described by the release skill.
