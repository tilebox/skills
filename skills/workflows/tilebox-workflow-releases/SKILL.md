---
name: tilebox-workflow-releases
description: "Initializes local Tilebox workflow projects, discovers and adapts owned/shared templates, links checkouts to remote workflows, and builds, publishes, deploys, and tests releases. Use for workflow init --template, cloned or unlinked projects, tilebox.workflow.toml, release artifacts, deployment targets, dynamic runners, or completing an implicit Earth observation project."
license: MIT
compatibility: Requires a current tilebox CLI and uv for scaffold dependency sync, release validation, and runners. Plain local init needs no API key; remote operations, linking, and template downloads require Tilebox authentication.
metadata:
  author: tilebox
---

# Releasing And Deploying Tilebox Workflows

Use this skill to initialize workflow projects, turn workflow code changes into immutable releases, and deploy those releases to one or more Tilebox clusters. Use `tilebox-workflow-authoring` for task code and this skill for project scaffold, config, publish, deploy, and runner iteration.

Use `tilebox-cli` first to ensure the CLI is up to date and inspect command schemas. The scoped-project release baseline is CLI >=0.14 and Tilebox Python packages >=0.68; check availability and installed capabilities rather than assuming these releases are already available. Regularly check and update project packages using authoring's `reference/tilebox/dependencies-and-packaging.md`, especially after cloning, copying a template, or resuming an older project.

Always submit workflow-scoped jobs for these projects, including development and smoke tests: use `--workflow .` or an exact remote reference. Running `tilebox runner start` locally still requires scoped jobs. Unscoped submissions are only for an explicitly requested direct Python runner or a Python script executed as a direct runner for testing; follow `tilebox-workflow-jobs` for that exception. Missing links or deployments are prerequisites to resolve, not reasons to drop scope.

## Agent Release Loop

For routine iteration, do the smallest safe loop:

1. Inspect the existing project and local link. For a new project, discover a named source workflow when the user asks to adapt one, then use `init --template`; otherwise initialize the embedded scaffold. Resolve an unlinked checkout before publishing using the linking policy below.
2. Edit workflow code and ensure changed files are covered by `[build].include` and not excluded.
3. Optional local verification: `tilebox workflow build-release --json`.
4. Publish: `tilebox workflow publish-release --json`.
5. Deploy the new release. For quick first results, omit `--cluster` to use the API default cluster instead of asking the user to choose one.
6. If the user requested an actual processed artifact, use `tilebox-workflow-jobs` to submit with `--workflow .` and verify a representative job after deployment; deployment alone does not create the artifact.
7. If no compatible runner is active and local execution is appropriate, start a dynamic runner for the same cluster; omitting `--cluster` also uses the API default cluster.

Prefer a specific release ID for production-like targets; use `--latest` for dev iteration only when that is acceptable.

For failed existing jobs caused by a compatible task bug, prefer deploying the fixed release and retrying the failed job over submitting a fresh job from scratch.

## Deployment Intent For Implicit Geospatial Projects

When the `tilebox` router accepts an implementation-oriented Earth observation or geospatial prompt through its implicit project route, local initialization is the first step. Treat remote creation, publication, and deployment as separate effects; perform them within the user's authorization and the host agent's approval rules.

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

```bash
tilebox workflow init --name "Scene QA" --json
tilebox workflow init ./scene-qa --name "Scene QA" --description "Processes scenes" --json
```

Plain init is local: it writes the scaffold, declares `tilebox-datasets>=0.68` and `tilebox-workflows>=0.68`, runs `uv sync`, and records new-workflow intent. It neither creates a remote workflow nor needs API credentials. The default destination is the current directory. `--name` derives the portable project slug; omit it only when the directory already has a suitable capability name. Init refuses to overwrite existing scaffold files.

Initialization starts with a minimal `runner.py`. For non-trivial workflows, follow `tilebox-workflow-authoring` to group tasks and processing into coherent modules, leaving registration and runner configuration in the entrypoint. Include the full package tree in the build. Declare optional libraries imported by the code and rerun `uv sync`.

Plain init needs `uv` on `PATH`. Fix a missing installation rather than hand-scaffolding. Template init is different: it downloads files without installing dependencies or executing source.

### Local Project And Remote Identity

Keep these identities separate:

| Identity | Meaning |
| --- | --- |
| `[workflow].slug` in `tilebox.workflow.toml` | Portable, non-unique project slug. Commit it with code; it is not a remote destination. Optional `name` and `description` supply first-publish metadata. |
| Remote workflow `reference` | Address returned by the API, possibly with a suffix; shared references include a namespace, such as `tilebox:example`. Never reconstruct it from the project slug or display name. |
| Remote workflow `id` | Immutable UUID used to scope task execution across releases. |
| `.tilebox/workflows.json` | Checkout-local destination state beside the TOML, keyed by project slug. Keep it ignored, not in Git or release artifacts. |

Local state has three meanings: `intent: "new"` allows first publish to create a workflow; a saved `organizationSlug`/`workflowReference` selects an existing workflow; no entry means unlinked. An optional `apiUrl` records a non-default endpoint. Let CLI commands manage this file; do not copy a colleague's state, hand-edit a destination, or delete a link to request a fork. There is one destination per project slug per checkout.

The CLI finds the nearest TOML by searching the current/given directory and parents. Remote project operations validate the link against the active organization and API. New intent cannot be used by `get`, deploy, or scoped submission before first publication. A missing/inaccessible saved workflow fails rather than creating a replacement or submitting unscoped work.

Init JSON contains `directory` and `project_slug`; `workflow_slug` is only present when linked, not for a fresh project. Workflow get/list/create output uses `reference`, not the old `slug` field. Inspect each command with `agent-context ... --output-schema` before parsing it.

## Adapt An Owned Or Shared Workflow

For “Adapt the seasonal RGB time-lapse workflow,” search existing workflows before writing a replacement. Inspect both owned and shared catalogs; use names, summaries, descriptions, and project slugs to find candidates, then inspect their releases and task/input contract. An exact reference supplied by the user can be inspected directly.

```bash
tilebox workflow list --limit 100 --json
tilebox workflow list --shared --limit 100 --json
tilebox workflow get <source-reference> --json
```

Page each catalog through `next_cursor`, preserving `--shared` where applicable. Use the exact returned `reference`, including namespace/suffix. Do not invent `tilebox:seasonal-rgb…`, assume a specific public slug exists, or conclude absence from the first page. If several plausible candidates remain, present the differences and ask which source the user means. If none is found, distinguish no match from missing authentication/access before offering a fresh implementation.

```bash
# SOURCE_REFERENCE is the exact owned or shared reference discovered above.
tilebox workflow init ./rgb-timelapse --template "$SOURCE_REFERENCE" --name "RGB Timelapse" --json
```

Template init copies the latest published release into an independent local project with new intent. It never links the copy to the source, updates the source, publishes, or deploys. It preserves source/package/task names, dependencies, and `uv.lock`; it reconstructs build rules and runner configuration from release metadata with root `"."`, without copying deployment targets. Scope isolates the eventual copy, so do not rename task identifiers to avoid collisions.

Without a destination, template init uses the source project slug or a slug derived from `--name`. An explicit destination, including `.`, must be absent or empty. A different directory alone does not rename the project. `--name`/`--description` customize the new project; `--template` cannot be combined with `--slug`. There is no template release-selection flag: record the returned source reference, release ID, and artifact digest when reporting what was copied.

Review downloaded code, dependencies, build rules, and storage/output defaults before `uv sync`, imports, or release validation. Then check/update Tilebox dependencies, adapt the code, build, and publish/deploy within authorization. Template init does not upgrade dependencies. If the source has no usable published release or download validation fails, report that cause; do not silently use another template or a blank scaffold.

## Link An Existing Or Cloned Project

`workflow link` selects an existing owned remote workflow without changing source, TOML metadata, remote metadata, or deployment. It does not download that workflow's code.

```bash
tilebox workflow link <owned-reference> --json
# Only when deliberately changing an existing destination:
tilebox workflow link <other-owned-reference> --replace --json
```

Repeated linking to the same destination is safe. Replacing new intent with an existing destination does not need `--replace`; changing a saved destination, organization, or API does. Shared workflows may be template sources or explicit execution targets, but cannot be linked as publishing destinations unless owned by the active organization. `init --slug <owned-reference>` scaffolds and links a new directory; it does not copy that remote workflow's implementation.

### Publishing A Colleague's Checkout

A Git clone normally lacks the ignored link file. Before publishing:

1. Inspect the nearest TOML, local state, and active account (`tilebox account get --json`). Preserve valid links and explicit new intent.
2. If unlinked, list owned workflows with pagination. Compare the portable project slug, name, description, repository context, and release task/build metadata. These identify candidates, not proof of ownership or identity.
3. If the user supplied an unambiguous destination, link it. Otherwise explain the likely match(es) and ask whether to update one or create an independent workflow. A generic “publish this clone” is not a reason to silently create a duplicate or choose a name match.
4. For an existing destination, run `workflow link <reference>`, inspect `workflow get`, then publish. For an explicitly chosen independent destination, use `workflow create` followed by `workflow link` as below; keep the existing project/source intact.

```bash
WORKFLOW_REFERENCE=$(tilebox workflow create "Scene QA" \
  --description "Processes new scenes" --json | jq -er '.reference')
tilebox workflow link "$WORKFLOW_REFERENCE" --json
```

Creation is a remote write; get authorization for that choice first. Do not rerun init over an existing project, replace its portable slug with a remote reference, fabricate `intent: "new"`, or assume an `unlink` command exists. If a link is stale or belongs to another organization/API, resolve that mismatch explicitly before `--replace`.

The CLI temporarily auto-links unlinked legacy projects only when the TOML slug exactly equals an existing remote reference (including suffix). Do not use that compatibility path as the agent's matching policy; check the intended destination before invoking a command that could auto-link.

## Project Configuration

For an existing project, preserve its portable slug and edit its build/runner configuration as needed. This example is a shape reference, not a command to overwrite an existing config:

```toml
[workflow]
slug = "scene-qa"
name = "Scene QA"
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
```

Config rules from the CLI implementation:

- File name must be `tilebox.workflow.toml`.
- `[workflow].slug` is required and remains portable; optional `name` and `description` are creation metadata, not instructions to update a linked workflow.
- `[workflow].root` is optional and defaults to `"."`; all build paths are relative to that root.
- Set exactly one of:
  - `runner = "module:object"`, which runs as `uv run python -m tilebox.workflows.runner module:object`.
  - `command = ["uv", "run", "python", "-m", "my_workflow.worker"]`, a custom worker process command.
- `[build].include` is required and must include at least one pattern.
- `[build].exclude` is optional. Builds always exclude `.git`, `.tilebox`, `.venv`, and `.tilebox-artifact` at any depth, case-insensitively, plus the generated archive. Still exclude secrets, downloaded data, and large assets explicitly.
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
tilebox workflow build-release --json
```

For CLI diagnostics, set `TILEBOX_DEBUG=1`; build/publish no longer accept `--debug`. This is separate from workflow task log severity.

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

Publishing builds and validates first. For new intent it then creates a scoped-only remote workflow, saves the returned reference in the local link, uploads the artifact, and publishes an immutable release. For a linked project it reuses the validated destination. Local build alone never creates a workflow.

Identical release content and artifact digest are idempotent within the destination workflow. Remote workflow creation is not guaranteed exactly once: after an ambiguous creation/network failure, inspect owned workflows and recover with `workflow link <reference>` before retrying. Once the link is saved, upload/publication retries reuse it. Do not delete it to retry. TOML name/description changes do not rename an already linked remote workflow.

Publishing requires `uv` on `PATH` because it builds and validates the Python workflow runtime before upload.

```bash
RELEASE_ID=$(tilebox workflow publish-release --json | tee /tmp/workflow-release.json | jq -r '.id')
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
3. Publish and deploy the new release to the same workflow and cluster as the failed job. A new workflow UUID cannot serve the old workflow's queued tasks even with identical task names.
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

Deploy maps a workflow release to clusters. It does not submit jobs by itself. Omit `--workflow` inside a linked project to use its saved destination, not the portable TOML slug. Outside the project, pass the exact remote reference. A shared workflow can be deployed by explicit reference without linking your source project to it.

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
  --workflow "$WORKFLOW_REFERENCE" \
  --release "$RELEASE_ID" \
  --cluster cluster-a,cluster-b \
  --json
```

Undeploy removes the workflow's active mapping on the selected clusters. Omitting `--release` removes whichever release is deployed there; supplying it selects a specific release:

```bash
tilebox workflow undeploy-release --target dev --json
tilebox workflow undeploy-release --release "$RELEASE_ID" --cluster cluster-a --json
```

Selector rules:

- Deploy requires exactly one of `--release <uuid>` or `--latest`. Undeploy accepts optional `--release` and has no `--latest` flag.
- `--release` must be a UUID.
- `--target <name>` requires a local `tilebox.workflow.toml` and must exist in `[targets]`.
- `--cluster` is comma-separated and cannot be combined with `--target`.
- If both `--cluster` and `--target` are omitted, the API uses the default cluster.

Inspect state:

```bash
tilebox workflow get --json
tilebox workflow get "$WORKFLOW_REFERENCE" --json
tilebox cluster get dev-cluster --json
```

## Start A Dynamic Runner Locally

A dynamic runner executes tasks for releases deployed to a cluster. It polls cluster deployment state, downloads/extracts missing artifacts, validates release task registrations, starts Python worker runtimes, and keeps running. It logs to stderr and does not emit JSON output.

`tilebox runner start` requires `uv` on `PATH` to start Python worker runtimes from workflow release artifacts. If `uv` is missing, install or fix `uv` before starting the runner.

Terminal 1:

```bash
tilebox runner start --cluster dev-cluster --log-level debug
```

Use the API default cluster by omitting `--cluster`:

```bash
tilebox runner start --log-level debug
```

Quiet console logs while still exporting Tilebox logs:

```bash
tilebox runner start --cluster dev-cluster --quiet
```

Terminal 2, after deploying a release to the same cluster, submit a root task:

```bash
tilebox job submit \
  --workflow . \
  --name scene-qa-test \
  --task tilebox.com/example/SceneQA \
  --version v1.0 \
  --cluster dev-cluster \
  --input S2A_001 \
  --wait \
  --json
```

Runner notes for debugging:

- With no deployed workflows, the runner idles locally and logs a warning.
- Deployment changes are picked up by polling, roughly every 10 seconds plus jitter.
- Invalid deployed releases are skipped while valid releases remain runnable.
- Upgrade managed runner CLIs as well as the submitting CLI before using scoped deployments. Releases contain unbound name/version pairs; the managed runner binds them to the deployment's workflow UUID without rewriting source or artifacts.
- Different workflows may expose the same task name/major on one cluster. Two workflows exposing the same unscoped legacy alias still conflict; duplicate name/major pairs within one release are invalid even with different minors.
- New CLI-created workflows are scoped-only. Existing workflows may retain `legacy_task_aliases` for old jobs/automations; do not remove legacy compatibility to fix an unrelated issue. If incompatible/ambiguous deployments are skipped, inspect runner logs rather than falling back to unscoped submission.
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
