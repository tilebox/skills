# Dependencies And Packaging

All runtime dependencies must be declared in `pyproject.toml` so `uv sync` can recreate the runner environment. Do not rely on ad-hoc `pip install`, notebook state, undeclared local packages, or packages installed manually on one machine.

## Basic Rules

- Put runtime dependencies in `[project].dependencies` and lint/type/test tools in groups such as `[dependency-groups].dev`.
- Keep Python constraints realistic for every intended runner.
- Do not add editable or local-path dependencies such as `my-lib @ file://...`, `my-lib @ ../my-lib`, or a uv source with `path`/`editable`.
- Keep local modules in the released source tree; publish/version reusable libraries or vendor a small stable helper.
- Run `uv sync`, validate imports, and verify runner startup. Avoid heavy packages when a smaller library fits.

## Keep Tilebox Packages Current

Check versions when starting/resuming work, after cloning or template initialization, before relying on new APIs, and before publishing after substantial changes. Plain `uv sync` respects the existing lockfile; it does not upgrade locked packages. Template init preserves the source dependencies and lockfile unchanged.

For scoped tasks and the local-project release, target `tilebox-workflows>=0.68` and the other Tilebox packages used by the project at >=0.68, alongside CLI >=0.14. Confirm those releases are available; these minimums describe the planned release baseline. Use current stable releases rather than pinning forever to the minimum. If unavailable, report the version blocker and use development/prerelease packages only when the user has selected them.

Inspect `pyproject.toml`, `uv.lock`, and installed distributions. Check available updates without changing the environment:

```bash
uv pip list --python .venv/bin/python --outdated
```

Adjust the interpreter path for the project/OS. Update only the Tilebox packages already used, respecting intentional pins and constraints. For a project using workflows and datasets:

```bash
# If existing constraints do not yet allow the required baseline:
uv add 'tilebox-workflows>=0.68' 'tilebox-datasets>=0.68'
# Refresh locked versions even when existing constraints already allow them:
uv lock --upgrade-package tilebox-workflows --upgrade-package tilebox-datasets
uv sync
```

Include `--upgrade-package tilebox-storage` or the `tilebox` meta-package when present; do not add unused packages just to upgrade them. Inspect the resulting constraint/lock changes, resolve compatibility issues, run focused tests/import checks and release build validation, and keep the updated lockfile with the code. Do not broadly upgrade unrelated scientific dependencies or silently override a deliberate version pin. Updating your local environment does not update an already published release: publish and deploy a new release within authorization. Use `tilebox-cli` for CLI and managed-runner upgrades.

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

## Optional Task-Input Types

Tilebox tasks support many common geospatial library values directly. Codec integrations load lazily so workflows that do not use those libraries stay small. Lazy loading does not install a package: declare every package imported by workflow source explicitly, even when another dependency happens to install it transitively.

| Imported type | Runtime dependency | Useful for |
| --- | --- | --- |
| `odc.geo.*` | `odc-geo>=0.5` with no upper bound | CRS-aware geometry, grids, and tiling |
| `shapely.*` | `shapely>=2` | Geometry without an associated CRS |
| `pyproj.CRS` | `pyproj>=3.4` | CRS parsing and transformations |
| `affine.Affine` | `affine>=2` | Raster grid transforms |
| `rasterio.windows.Window` | `rasterio` | Rasterio window reads and writes |
| `async_geotiff.Window` | `async-geotiff` | Async GeoTIFF window reads |

For example, source that imports ODC Geo and Shapely should declare both:

```toml
dependencies = [
  "tilebox-workflows>=0.68",
  "odc-geo>=0.5",
  "shapely>=2",
]
```

After changing task field types or dependencies, run `uv sync` and import every task module in the synchronized environment. Release build validation must discover the same modules successfully.

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
