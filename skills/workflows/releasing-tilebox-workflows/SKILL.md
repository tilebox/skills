---
name: releasing-tilebox-workflows
description: "Creates, builds, publishes, deploys, undeploys, and locally tests Tilebox Agentic Workflow releases with tilebox.workflow.toml and dynamic runners. Use when iterating on workflow code, release artifacts, deployment targets, or cluster runners."
license: MIT
compatibility: Requires the tilebox CLI, and a Tilebox API Key ($TILEBOX_API_KEY) or `--api-key`.
metadata:
  author: tilebox
---

# Releasing Tilebox Workflows

Use this skill to turn workflow code changes into an immutable release and deploy that release to one or more Tilebox clusters. Use `writing-tilebox-workflows` for task code and this skill for project config, publish, deploy, and runner iteration.

## Agent Release Loop

For routine iteration, do the smallest safe loop:

1. Edit workflow code and ensure changed files are covered by `[build].include` and not excluded.
2. Optional local verification: `tilebox workflow build-release --debug --json`.
3. Publish: `tilebox workflow publish-release --json`.
4. Deploy the new release to a target or cluster.
5. If testing locally, use a testing cluster, deploy the release to that, and run a dynamic runner for that cluster and submit a job.

Prefer a specific release ID for production-like targets; use `--latest` for dev iteration only when that is acceptable.

For failed existing jobs caused by a compatible task bug, prefer deploying the fixed release and retrying the failed job over submitting a fresh job from scratch.

## Create Or Bind A Workflow Project

Create the server-side workflow, then write or update `tilebox.workflow.toml` in the project root. The CLI searches upward from the current directory for the nearest config file, so commands work from subdirectories.

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

```bash
tilebox workflow build-release --debug --json
```

The build command:

- resolves included files from `[workflow].root` using `[build].include`, `[build].exclude`, and `.gitignore` when enabled;
- creates a deterministic local `.tar.zst` artifact and SHA-256 digest;
- extracts the artifact into the local Tilebox artifact cache;
- starts the configured worker runtime and calls task discovery;
- returns the content fingerprint, task identifiers, files, and artifact digest/path.

If build fails, fix the config or runtime before publishing. Common fixes: include `pyproject.toml`, `uv.lock`, and `src/**`; exclude `.venv/**`; ensure the `runner` import path resolves from the extracted artifact. Fix any python import errors.

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
