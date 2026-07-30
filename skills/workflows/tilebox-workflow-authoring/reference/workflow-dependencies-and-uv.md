# Workflow Dependencies And uv

All runtime dependencies for a Tilebox workflow project must be declared in `pyproject.toml` so `uv sync` can recreate the environment on runners.

Do not rely on ad-hoc `pip install`, notebook state, undeclared local packages, or packages installed manually on one machine.

## Basic Rules

- Put runtime dependencies in `[project].dependencies`.
- Put lint/type/test tools in dependency groups such as `[dependency-groups].dev`.
- Keep Python version constraints realistic for all intended runners.
- Do not add editable installs or local path dependencies. Avoid dependency declarations such as `my-lib @ file://...`, `my-lib @ ../my-lib`, `[tool.uv.sources].my-lib = { path = "..." }`, or `{ path = "...", editable = true }`.
- If workflow code needs local modules, keep them inside the workflow source tree and include them in the workflow release artifact; if it needs a reusable library, publish/version that library in a package index or vendor the needed source into the workflow project.
- Run `uv sync` after dependency changes.
- Validate imports and runner startup after sync.
- Avoid adding heavy dependencies when a smaller library fits.

## No Editable Or Local Path Dependencies

Workflow releases must resolve dependencies from the release artifact and package indexes on the runner. Editable installs and local path dependencies depend on files outside the released project or on development-machine paths, so they will fail to resolve during release validation or on `tilebox runner start`.

Do not use these patterns in workflow `pyproject.toml`:

```toml
dependencies = [
  "shared-lib @ file:///Users/alice/dev/shared-lib",
  "shared-lib @ ../shared-lib",
]

[tool.uv.sources]
shared-lib = { path = "../shared-lib", editable = true }
```

Use one of these instead:

- publish the dependency to a package index and pin a normal version constraint;
- move the source under the workflow project, for example `src/my_workflow/shared_lib/`, and include it via `[build].include`;
- copy a small, stable helper module into the workflow package when publishing a separate library would be overkill.

## Geospatial Defaults

Add only what the workflow uses, but prefer:

- `obstore` for object storage.
- `tilebox-storage` for asset-aware COG/GeoTIFF reads; it includes the async storage client and compatible `async-geotiff` dependency.
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
