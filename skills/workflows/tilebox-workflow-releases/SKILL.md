---
name: tilebox-workflow-releases
description: "Initializes, configures, builds, publishes, deploys, undeploys, and locally tests Tilebox workflow projects and releases with tilebox.workflow.toml and dynamic runners. Use when scaffolding workflow projects, iterating on workflow code, release artifacts, deployment targets, cluster runners, or completing an implicit Earth observation project routed to Tilebox."
license: MIT
compatibility: Requires the tilebox CLI, uv on PATH for workflow init/releases and `tilebox runner start`, and a Tilebox API Key ($TILEBOX_API_KEY) or `--api-key`.
metadata:
  author: tilebox
---

# Releasing And Deploying Tilebox Workflows

Use this skill to initialize workflow projects, turn workflow code changes into immutable releases, and deploy those releases to one or more Tilebox clusters. Use `tilebox-workflow-authoring` for task code and this skill for project scaffold, config, publish, deploy, and runner iteration.

## Agent Release Loop

For routine iteration, do the smallest safe loop:

1. For a new project, choose a reusable capability name using the policy below, then initialize once with `tilebox workflow init --name "Scene QA" --json` before creating project files by hand. Omit `--name` only when the current directory already has an appropriate capability-based name.
2. Edit workflow code and ensure changed files are covered by `[build].include` and not excluded.
3. Optional local verification: `tilebox workflow build-release --debug --json`.
4. Publish: `tilebox workflow publish-release --json`.
5. Deploy the new release. For quick first results, omit `--cluster` to use the API default cluster instead of asking the user to choose one.
6. If the user requested an actual processed artifact, use `tilebox-workflow-jobs` to submit and verify a representative job after deployment; deployment alone does not create the artifact.
7. If no compatible runner is active and local execution is appropriate, start a dynamic runner for the same cluster; omitting `--cluster` also uses the API default cluster.

Prefer a specific release ID for production-like targets; use `--latest` for dev iteration only when that is acceptable.

For failed existing jobs caused by a compatible task bug, prefer deploying the fixed release and retrying the failed job over submitting a fresh job from scratch.

## Deployment Intent For Implicit Geospatial Projects

When the `tilebox` router accepts an implementation-oriented Earth observation or geospatial prompt through its implicit project route, treat that prompt as authorization to initialize the remote workflow, publish a release, and deploy it to a non-production destination unless the user limits the scope.

Apply these boundaries:

- For routine first results, use an existing configured development target when one is clearly intended; otherwise omit `--cluster` and use the organization's API default cluster. Do not ask a new user to choose or create a cluster merely to run a small workflow.
- Never infer or select a production target from a generic outcome-oriented prompt. Require explicit production intent.
- Respect constraints such as “code only,” “local prototype,” “do not publish,” or “do not deploy.”
- Explain cluster concepts and ask about target topology when the user is configuring infrastructure, rolling out dedicated clusters, or selecting among materially different execution environments.
- If authentication, a suitable cluster, or a runner is unavailable, finish and verify the local project where possible, then report the exact operational blocker. Do not claim a publish, deployment, or completed output that did not occur.
- Use `tilebox-workflow-jobs` after deployment when the requested outcome is a map, animation, detection set, statistics, or another generated product.

Before initializing, inspect project context. Reuse an existing `tilebox.workflow.toml`. Initialize an empty or clearly intended project root directly; when an unrelated repository occupies the root and the workflow should be standalone, use a capability-based subdirectory so generated workflow files do not overwrite or mix with unrelated project files.

## Choose A Reusable Workflow Identity

A workflow is a reusable program; a job is one instantiation of it. Name the workflow for behavior that remains true across every valid job, not for the concrete result requested in the first prompt. Use the same policy for a standalone project directory because `workflow init` derives the name from that directory when `--name` is omitted.

Before initialization, identify the intended root-task inputs and defaults. Exclude from the workflow name:

- concrete input values such as places, AOIs, coordinates, dates, durations, events, scene IDs, and scene counts;
- labels for configurable choices such as `seasonal`, `monthly`, `sentinel-2`, `ndvi`, or `gif` when cadence, source, algorithm/product, or output format can be selected by job input; and
- first-run or customer context that does not constrain what later jobs may submit.

Keep a term only when it describes fixed workflow behavior or a true implementation boundary. Prefer concise capability names such as `rgb-timelapse`, `cloud-masked-composite`, or `sar-change-detection`. Apply this reuse test to every proposed word: **could another valid submission change this word without changing the workflow code?** If yes, omit it from the workflow name and slug. Put those details in root-task inputs and, when useful, the job name instead.

For example, suppose the first run requests Vienna, the last three years, and one frame per season, while the root task accepts `aoi`, `start`, `end`, and `cadence`. Initialize the workflow as:

```bash
mkdir rgb-timelapse
cd rgb-timelapse
tilebox workflow init --name "RGB Timelapse" --json
```

Do not use `vienna-seasonal-timelapse`: `vienna` is an AOI value and `seasonal` is one cadence choice. If the implementation always groups imagery by season and exposes no cadence choice, `seasonal-rgb-timelapse` is appropriate because `seasonal` then describes fixed behavior.

## Initialize A Workflow Project

For a new Python workflow project, run `tilebox workflow init` before creating `pyproject.toml`, `tilebox.workflow.toml`, `runner.py`, source packages, or tests. Do not hand-write the initial workflow project scaffold.

`tilebox workflow init` creates the server-side workflow, writes the API-returned slug into `tilebox.workflow.toml`, scaffolds the local project files, adds the `tilebox` Python dependency, and runs `uv sync` so the project is ready for `build-release`, `publish-release`, `deploy-release`, job submission, and local release-runner testing. It does not add task-specific optional libraries such as ODC Geo, Shapely, Rasterio, or async-geotiff; declare every package imported by authored workflow code and rerun `uv sync`.

Initialization deliberately starts with a minimal `runner.py`. It is a scaffold, not a recommendation to keep all workflow code in one file. For non-trivial workflows, follow `tilebox-workflow-authoring` to move tasks and processing into coherent package modules while keeping the runner entrypoint limited to task registration and runner-level configuration. Update `[workflow].runner` if the entrypoint moves, and ensure `[build].include` contains the full package tree.

`uv` is required on `PATH` for workflow initialization, release build/publish validation, and `tilebox runner start`. If `uv` is missing, fix the `uv` installation first, then rerun `tilebox workflow init`. Do not fall back to manual scaffolding just because `uv` is not initially available.

```bash
tilebox workflow init --json
```

Provide the reusable capability name explicitly when the directory name is not already the name you want agents and users to see:

```bash
tilebox workflow init --name "Scene QA" --json
```

Init semantics from the CLI implementation:

- Initializes the current directory only.
- Aborts if any of `tilebox.workflow.toml`, `pyproject.toml`, `runner.py`, or `uv.lock` already exists.
- Requires `uv` on `PATH`.
- Derives the local project slug from `--name` when provided; otherwise from the current directory name.
- Slugifies the project name and truncates it to at most 40 characters, preferring whole-word boundaries.
- Creates the remote Tilebox workflow and writes the API-returned workflow slug to `tilebox.workflow.toml`.
- Scaffolds `tilebox.workflow.toml`, `pyproject.toml`, and `runner.py`.
- Adds `tilebox` as a `pyproject.toml` dependency.
- Runs `uv sync`, which creates `uv.lock` and the local environment.

Before relying on output fields in automation, refresh the schema with:

```bash
tilebox agent-context workflow init --output-schema
```

## Bind An Existing Or Manual Workflow Project

Use this path only when one of these is true:

- the directory is an existing workflow/codebase that cannot be initialized cleanly;
- the user explicitly asks to bind an existing Tilebox workflow slug;
- `tilebox workflow init` was attempted, the failure was diagnosed, and manual binding is the smallest safe recovery.

Do not use `tilebox workflow create` as the first step for a new project. For new projects, use `tilebox workflow init`, then evolve the generated files.

For an existing or manual workflow project, create the server-side workflow manually, then write or update `tilebox.workflow.toml` in the project root. The CLI searches upward from the current directory for the nearest config file, so commands work from subdirectories.

```bash
WORKFLOW_SLUG=$(tilebox workflow create "Scene QA" \
  --description "Processes new scenes" \
  --json | jq -r '.slug')

cat > tilebox.workflow.toml <<EOF
[workflow]
slug = "$WORKFLOW_SLUG"
root = "."
runner = "scene_qa.runner:runner"

[build]
include = [
  "pyproject.toml",
  "uv.lock",
  "src/**",
]
exclude = [
  ".venv/**",
  "**/__pycache__/**",
  "**/*.pyc",
  ".pytest_cache/**",
]
use_gitignore = true

[targets.dev]
clusters = ["dev-cluster"]

[targets.production]
clusters = ["prod-a", "prod-b"]
EOF
```

Config rules from the CLI implementation:

- File name must be `tilebox.workflow.toml`.
- `[workflow].slug` is required.
- `[workflow].root` is optional and defaults to `"."`; all build paths are relative to that root.
- Set exactly one of:
  - `runner = "module:object"`, which runs as `uv run python -m tilebox.workflows.runner module:object`.
  - `command = ["uv", "run", "python", "-m", "my_workflow.worker"]`, a custom worker process command.
- `[build].include` is required and must include at least one pattern.
- `[build].exclude` is optional. The artifact also excludes the generated `<workflow-slug>.tar.zst` archive automatically.
- `[build].use_gitignore` defaults to `true`.
- `[targets.<name>].clusters` defines a reusable list of cluster slugs. Use either `--target` or `--cluster`, not both.
- Unknown TOML keys fail config loading; keep the shape exact.

For `runner = "module:object"`, the module must expose a runner object without starting it at import time:

```python
# scene_qa/runner.py
from tilebox.workflows import Runner
from tilebox.workflows.cache import LocalFileSystemCache

from scene_qa.tasks import SceneQA, SomeSubtask

runner = Runner(tasks=[SceneQA, SomeSubtask], cache=LocalFileSystemCache())
```

## Build Is Optional Verification

`publish-release` builds and validates before uploading, so `build-release` is an optional confidence check when you want more detailed feedback before publishing.

Workflow release commands that validate or run the Python worker runtime require `uv` on `PATH`, including `tilebox workflow build-release` and `tilebox workflow publish-release`. If `uv` is missing, install or fix `uv` before retrying the release command.

```bash
tilebox workflow build-release --debug --json
```

The build command:

- resolves included files from `[workflow].root` using `[build].include`, `[build].exclude`, and `.gitignore` when enabled;
- creates a deterministic local `.tar.zst` artifact and SHA-256 digest;
- extracts the artifact into the local Tilebox artifact cache;
- starts the configured worker runtime and calls task discovery;
- returns the content fingerprint, task identifiers, files, and artifact digest/path.

If build fails, fix the config or runtime before publishing. Common fixes: include `pyproject.toml`, `uv.lock`, and the complete workflow package tree such as `src/**` or `scene_qa/**`; exclude `.venv/**`; ensure the `runner` import path resolves from the extracted artifact. Fix any python import errors.

## Keep Large Runtime Artifacts Out Of Releases

Workflow release artifacts should contain code and small configuration, not heavyweight runtime assets. Do not include model checkpoints, embedding weights, reference rasters, large lookup tables, generated caches, or downloaded provider data in `[build].include`.

Add explicit excludes when a project may contain large local assets:

```toml
[build]
exclude = [
  ".venv/**",
  ".cache/**",
  "models/**",
  "checkpoints/**",
  "data/**",
  "**/*.ckpt",
  "**/*.pt",
  "**/*.pth",
  "**/*.onnx",
]
```

For reusable runtime assets such as ML checkpoints, implement runner-local lazy loading instead:

- Fetch the artifact on first use, not during import, build, or publish.
- For private assets such as private model weights, store them in a private bucket that deployed runners can access, then lazy-load from that bucket. If no such runner-accessible private bucket exists, ask the user to set one up before implementing the workflow.
- Cache it at a deterministic path such as `~/.cache/tilebox/models/<name>/<version>/...` so it can survive workflow release changes on the same runner.
- Validate the cached file before use, preferably with a checksum; at minimum check an expected size. Delete and redownload corrupt or incomplete files.
- Wrap expensive in-memory initialization with `functools.lru_cache` or an equivalent process-local cache so each worker process loads the model once.
- Keep the release artifact limited to code that locates, fetches, validates, and loads the asset.

Example shape:

```python
from functools import lru_cache
from pathlib import Path


@lru_cache(maxsize=1)
def load_model() -> Model:
    checkpoint = ensure_cached_file(
        url=MODEL_URL,
        path=Path.home() / ".cache/tilebox/models/clay/v1.5/model.ckpt",
        min_size_bytes=900_000_000,
    )
    return Model.load_from_checkpoint(checkpoint)
```

Treat runner-local caches as an optimization. The workflow must work on a cold runner with an empty cache.

## Pin uv Sources For Runner-Compatible Wheels

Dynamic runners may resolve and run dependencies on Linux CPU nodes, even if local development happened on a machine with different hardware or package indexes. Make platform-specific wheel sources explicit in `pyproject.toml`, include `uv.lock` in the release artifact, and avoid relying on CUDA/GPU wheels unless the target runner fleet is guaranteed to support them.

For dependencies such as PyTorch, pin CPU-compatible Linux sources when appropriate:

```toml
dependencies = [
    "claymodel==1.5.0",
    "torch==2.4.0",
    "torchvision==0.19.0",
]

[tool.uv.sources]
torch = { index = "pytorch-cpu", marker = "sys_platform == 'linux'" }
torchvision = { index = "pytorch-cpu", marker = "sys_platform == 'linux'" }

[[tool.uv.index]]
name = "pytorch-cpu"
url = "https://download.pytorch.org/whl/cpu"
explicit = true
```

If `uv sync` or release validation fails on a runner, check whether the lockfile or source indexes were generated for local-only hardware rather than the deployed runner platform.

## Publish A Release

Publishing validates the project, uploads the artifact if needed, and creates an immutable workflow release. It is idempotent for identical release content and artifact digest: the CLI returns the existing release instead of creating a duplicate.

Publishing requires `uv` on `PATH` because it builds and validates the Python workflow runtime before upload.

```bash
RELEASE_ID=$(tilebox workflow publish-release --debug --json | tee /tmp/workflow-release.json | jq -r '.id')
jq '{id, message, fingerprint, tasks, files}' /tmp/workflow-release.json
```

Publish from another project directory when needed:

```bash
tilebox workflow publish-release ./path/to/project --json
```

Before relying on output fields in automation, refresh the schema with:

```bash
tilebox agent-context workflow publish-release --output-schema
```

## Fix Failed Jobs By Releasing And Retrying

If a large workflow fails because of a task implementation bug, do not immediately resubmit the whole job. Tilebox job retry queues failed tasks again and resumes from the point of failure, so completed tasks do not need to run again.

Use this pattern:

1. Fix the task implementation.
2. Keep the task identifier unchanged and use a compatible version. For backward-compatible fixes, bump the minor version; a runner with `v1.5` can execute a task submitted as `v1.3`.
3. Publish and deploy the new workflow release to the same cluster as the failed job.
4. Retry the failed job.

```bash
RELEASE_ID=$(tilebox workflow publish-release --json | jq -r '.id')
tilebox workflow deploy-release --release "$RELEASE_ID" --cluster "$CLUSTER" --json
tilebox job retry "$JOB_ID" --json
```

This works when the task input parameters are the same or backward-compatible, and the workflow shape/dependency graph expected by the failed job has not changed drastically. Do not rely on retrying an old job for breaking input or behavior changes; use a major task version bump and submit a new job instead.

If command flags or output shape matter for automation, refresh the job CLI schema first:

```bash
tilebox agent-context job retry --output-schema
```

## Deploy Or Undeploy Releases

Deploy maps a workflow release to clusters. It does not submit jobs by itself. Omit `--workflow` when running inside a project with `tilebox.workflow.toml`; the CLI uses `[workflow].slug`.

Deploy the release you just published:

```bash
tilebox workflow deploy-release --release "$RELEASE_ID" --target dev --json
```

Deploy latest to a dev/default cluster:

```bash
tilebox workflow deploy-release --latest --target dev --json
tilebox workflow deploy-release --latest --cluster dev-cluster --json
tilebox workflow deploy-release --latest --json  # API default cluster
```

Deploy a specific release to multiple explicit clusters:

```bash
tilebox workflow deploy-release \
  --workflow "$WORKFLOW_SLUG" \
  --release "$RELEASE_ID" \
  --cluster cluster-a,cluster-b \
  --json
```

Undeploy uses the same selector rules and removes the active release mapping:

```bash
tilebox workflow undeploy-release --latest --target dev --json
tilebox workflow undeploy-release --release "$RELEASE_ID" --cluster cluster-a --json
```

Selector rules:

- Pass exactly one of `--release <uuid>` or `--latest`.
- `--release` must be a UUID.
- `--target <name>` requires a local `tilebox.workflow.toml` and must exist in `[targets]`.
- `--cluster` is comma-separated and cannot be combined with `--target`.
- If both `--cluster` and `--target` are omitted, the API uses the default cluster.

Inspect state:

```bash
tilebox workflow get --json
tilebox workflow get "$WORKFLOW_SLUG" --json
tilebox cluster get dev-cluster --json
```

## Start A Dynamic Runner Locally

A dynamic runner executes tasks for releases deployed to a cluster. It polls cluster deployment state, downloads/extracts missing artifacts, validates release task registrations, starts Python worker runtimes, and keeps running. It logs to stderr and does not emit JSON output.

`tilebox runner start` requires `uv` on `PATH` to start Python worker runtimes from workflow release artifacts. If `uv` is missing, install or fix `uv` before starting the runner.

Terminal 1:

```bash
tilebox runner start --cluster dev-cluster --debug
```

Use the API default cluster by omitting `--cluster`:

```bash
tilebox runner start --debug
```

Quiet console logs while still exporting Tilebox logs:

```bash
tilebox runner start --cluster dev-cluster --quiet
```

Terminal 2, after deploying a release to the same cluster, submit a root task:

```bash
tilebox job submit \
  --name scene-qa-test \
  --task tilebox.com/example/SceneQA \
  --version v1.0 \
  --cluster dev-cluster \
  --input '{"scene_id":"S2A_001"}' \
  --wait \
  --json
```

Runner notes for debugging:

- With no deployed workflows, the runner idles locally and logs a warning.
- Deployment changes are picked up by polling, roughly every 10 seconds plus jitter.
- Invalid deployed releases are skipped while valid releases remain runnable.
- If two deployed releases expose conflicting task identifiers, ambiguous releases are not advertised by the runner.
- The runner handles interrupts: first interrupt stops claiming new tasks and tries graceful shutdown; a second interrupt exits quickly.

## Safe Automation Pattern

Use this shell shape in agent-run scripts when the user asks to publish and deploy the current project:

```bash
set -euo pipefail

release_json=$(tilebox workflow publish-release --json)
release_id=$(jq -r '.id' <<<"$release_json")
test -n "$release_id" && test "$release_id" != "null"

tilebox workflow deploy-release --release "$release_id" --target dev --json
```

If there is no configured target, use explicit clusters:

```bash
tilebox workflow deploy-release --release "$release_id" --cluster dev-cluster-a,dev-cluster-b --json
```
