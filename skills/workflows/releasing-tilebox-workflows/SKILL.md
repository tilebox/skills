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
