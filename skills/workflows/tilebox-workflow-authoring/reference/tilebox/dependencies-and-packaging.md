# Dependencies And Packaging

All runtime dependencies must be declared in `pyproject.toml` so `uv sync` can recreate the runner environment. Do not rely on ad-hoc `pip install`, notebook state, undeclared local packages, or packages installed manually on one machine.

## Basic Rules

- Put runtime dependencies in `[project].dependencies` and lint/type/test tools in groups such as `[dependency-groups].dev`.
- Keep Python constraints realistic for every intended runner.
- Do not add editable or local-path dependencies such as `my-lib @ file://...`, `my-lib @ ../my-lib`, or a uv source with `path`/`editable`.
- Keep local modules in the released source tree; publish/version reusable libraries or vendor a small stable helper.
- Run `uv sync`, validate imports, and verify runner startup. Avoid heavy packages when a smaller library fits.

## No Editable Or Local Paths

Development-machine paths are unavailable during release validation and runner startup. Do not use:

```toml
dependencies = [
  "shared-lib @ file:///Users/alice/dev/shared-lib",
  "shared-lib @ ../shared-lib",
]

[tool.uv.sources]
shared-lib = { path = "../shared-lib", editable = true }
```

Instead publish the dependency to an index with a normal version constraint, move it under the workflow package (for example `src/my_workflow/shared_lib/`) and include it in the release, or copy a small stable helper into the package.

## Geospatial Defaults

Add only what is used. Typical choices are `obstore` for external object storage, `tilebox-storage` for asset-aware GeoTIFF reads, `zarr` for direct region writes, `xarray` for labeled Zarr/NetCDF reads, `odc-geo` for grid/reprojection work, `rasterio` for writing/warping and other GDAL formats, and `niquests` for non-storage HTTP. Follow the relevant storage, Zarr, and grid references instead of repeating their setup here.

## PyTorch And Architecture-Specific Wheels

Use explicit uv indexes and platform markers matching intended CPU/GPU runners:

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

Adjust CUDA, fallback, OS, and architecture markers and verify each target runner class.

## Runtime Assets And Validation

Do not pass model checkpoints, static rasters, lookup tables, or calibration files through task inputs or job cache. Fetch them lazily into a deterministic runner-local cache, read them from accessible private object storage, or deliberately bundle small package data. Validate cached checksums or sizes. See [state and artifacts](state-and-artifacts.md) for runtime-state boundaries.

```bash
uv sync
uv run python -c "import your_workflow_module"
```

For release projects, also run the build/release validation prescribed by `tilebox-workflow-releases`; initialization, publishing, deployment, and runner configuration remain owned there.
