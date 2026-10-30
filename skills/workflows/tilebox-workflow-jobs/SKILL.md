---
name: tilebox-workflow-jobs
description: "Manages Tilebox workflow jobs with the tilebox job CLI. Use when submitting jobs, listing jobs, inspecting state, listing clusters, creating clusters, waiting for job completion, reading job logs/spans, retrying failed work, or canceling jobs."
license: MIT
compatibility: Requires the tilebox CLI, and a Tilebox API Key ($TILEBOX_API_KEY) or `--api-key`.
metadata:
  author: tilebox
---

# Managing Tilebox Jobs

Use this skill for operational work with `tilebox job` and `tilebox cluster`. For agents, use `--json` on every job command unless explicitly producing human output.

## Refresh CLI Metadata

Use `tilebox-cli` to ensure the CLI is current (scoped-project baseline >=0.14), including managed runners. In Python workflow projects, regularly check/update Tilebox packages (baseline >=0.68) using authoring's dependency guidance. Check release availability and exact installed flags/schemas before relying on memory:

```bash
tilebox agent-context job --output-schema
tilebox agent-context cluster --output-schema
```

Relevant docs concepts:

- Tilebox Workflows is a parallel processing engine for tasks across clusters.
- A submitted job starts a trace; each task run creates a span.
- Task logs are correlated with job, task, runner, service, trace, and span metadata.
- Logs emitted inside an active span also appear as span events in trace views.

## Command Choice

- Start work: `tilebox job submit --workflow . --name ... --task ... --input ... --json` from a linked project, or use an explicit remote reference.
- Find jobs: `tilebox job list --last 7d --json` or filter with `--workflow <reference>`, `--state`, `--task-state`, `--name`.
- Inspect one job: `tilebox job get <job-id> --json`.
- Wait for completion/failure/cancel: `tilebox job wait <job-id> --json`.
- Inspect job log messages: `tilebox job logs <job-id> --sort desc --limit 100 --json`.
- Inspect job traces/spans when debugging timing: `tilebox job spans <job-id> --sort asc --json`.
- Retry eligible failed tasks after fixing the cause: `tilebox job retry <job-id> --json`.
- Stop pending/running work: `tilebox job cancel <job-id> --json`.

Use `tilebox agent-context job <subcommand> --output-schema` when a command's arguments or output shape are unclear. `agent-context` always returns JSON; do not add `--json` to it.

## Select The Execution Scope

A task execution address is its workflow UUID (or unscoped), name, and version. Workflow references select that UUID; task names remain portable. Do not prefix `--task` with a workflow reference or put a workflow ID into the input payload. Cluster selects where to run; workflow selects which implementation may execute the task. Neither substitutes for the other.

| Submission selector | Behavior |
| --- | --- |
| `--workflow .` | Resolve the nearest local project's saved link. New/unlinked, stale, inaccessible, or wrong-organization/API links fail. Use `tilebox-workflow-releases` to publish/link first. |
| `--workflow <reference>` | Resolve an owned or shared workflow reference, independent of the current directory. Use the exact API-returned reference. |
| Omitted `--workflow` | Unscoped, even inside a workflow project. The CLI warns in a project but does not infer scope. |
| `--workflow ''` | Deliberately unscoped; suppresses the project warning. Use only for the direct-runner exceptions below. |

**Always submit workflow-scoped jobs when working with a workflow project**, including scaffolded, cloned, and template-derived projects. Use `--workflow .` or the exact remote reference; in Python pass `workflow=reference_or_workflow`. This is required for development, smoke tests, and local managed runners too, not just production.

The only exceptions are when the user explicitly requests a direct Python runner, or when you execute a Python script as a direct runner for testing. Those submissions must be unscoped: use CLI `--workflow ''` or omit Python's `workflow` argument (or pass `None`). A Python script that only submits jobs to a deployed workflow is not a direct runner. Running `tilebox runner start` locally is also managed execution and still requires scoped jobs.

Do not switch to a direct runner or unscoped submission merely because linking, publication, deployment, or scope resolution failed. Resolve the prerequisite or report the blocker. Legacy aliases support existing work; their availability is not permission to submit new unscoped project jobs. New CLI-created workflows disable those aliases, so an omitted selector can leave work queued despite a healthy deployed runner. Matching task names in another workflow cannot execute this workflow's scoped work.

Confirm the selected workflow's release is deployed on the target cluster, inspect its discovered tasks, and match name/version and serialized input. Submission does not pin a release; a compatible newer release of the same workflow can execute queued work. Inspect `workflow get` and `cluster get` when the intended implementation is unclear.

## Submit Jobs

Basic form:

```bash
tilebox job submit \
  --workflow . \
  --name <job-name> \
  --task <task-identifier-name> \
  --version v0.0 \
  --input '<json-or-plain-text>' \
  --json
```

Important flags:

- `--workflow`: `.` for the linked local project or an explicit remote reference; omission stays unscoped.
- `--name`: required job name. This identifies one execution and may include useful run-specific context such as location, period, or cadence. Do not reuse an instance-specific job name as the workflow name or slug.
- `--task`: required task identifier name.
- `--version`: defaults to `v0.0`.
- `--input`: inline JSON or plain text. Valid JSON passes through; non-JSON text becomes a JSON string.
- `--input-file`: read input from a file; use `-` for stdin.
- `--cluster`: optional cluster slug; omit for the default cluster.
- `--max-retries`: root task retry count, default `0`.
- `--wait`: submit and then wait like `tilebox job wait <new-job-id>`.

Only use `--wait` when a compatible runner is known to be available and expected to execute the task. Otherwise submit without `--wait`, then inspect with `job get`, `job logs`, or `job spans`.

Examples:

```bash
tilebox job submit --workflow . --name process-scene --task ProcessScene --input S2A_001 --json
tilebox job submit --workflow . --name process-count --task ProcessCount --input 5 --json
tilebox job submit --workflow . --name process-count --task ProcessCount --input '"5"' --json
tilebox job submit --workflow <reference> --name structured --task tilebox.com/process_scene --version v1.0 --input '{"scene_id":"S2A_001","other_arg":3}' --json
tilebox job submit --workflow . --name from-file --task ProcessScenes --input-file scenes.json --json
cat scenes.json | tilebox job submit --workflow . --name from-stdin --task ProcessScenes --input-file - --json
```

For Python `CronTask` or `StorageEventTask` submissions, use the `tilebox-workflow-automations` skill. Those require `--automation` to construct the automation trigger wrapper.

## Python Task Identifiers And Input

Python `Task` classes default to the unbound name/version pair `<ClassName>`, `v0.0` unless they define an explicit `identifier()` method. The `name@version` display is not CLI syntax: pass name and version separately. Match the deployed task definition and select its workflow separately.

Input must match Python `serialize_task(task)` / `deserialize_task(TaskClass, bytes)`:

- No fields: omit input or submit `{}`.
- One field: submit the field value directly.
  - `scene_id: str` -> `--input S2A_001` submits JSON string `"S2A_001"`.
  - `count: int` -> `--input 5` submits JSON number `5`; use `--input '"5"'` for string `"5"`.
  - `scene_ids: list[str]` -> submit a JSON array, not an object.
- Multiple fields: submit a JSON object keyed by field names.

When unsure, produce the exact payload with Python:

```bash
/path/to/.venv/bin/python - <<'PY' > task-input.json
from test import ProcessScenes
from tilebox.workflows.task import serialize_task, deserialize_task

task = ProcessScenes(["S2A_001", "S2B_002"])
payload = serialize_task(task)
assert deserialize_task(ProcessScenes, payload).scene_ids == task.scene_ids
print(payload.decode())
PY

tilebox job submit --workflow . --name process-scenes --task ProcessScenes --input-file task-input.json --json
```

For Python submissions with current packages:

```python
from tilebox.workflows import Client

client = Client()
job = client.jobs().submit("process-scenes", ProcessScenes(["S2A_001", "S2B_002"]), workflow="my-workflow-reference")
```

Pass a remote reference or a `Workflow` object. Python does not resolve the CLI's `workflow="."` local-link shorthand. Omitted `workflow`/`None` submits unscoped roots, even when called from inside another task; only `context.submit_subtask(s)` inherits the executing parent's scope. Use `tilebox-workflow-authoring` for those task-graph semantics.

## List, Inspect, Wait

```bash
tilebox job list --last 7d --limit 100 --json
tilebox job list --workflow <reference> --last 7d --json
tilebox job list --state failed --after 2026-05-01 --before 2026-06-01 --json
tilebox job list --name landsat --task-state failed,failed_optional --json
tilebox job get <job-id> --json
tilebox job wait <job-id> --stalled-timeout 5m --json
```

For paginated list output, keep filters and sort unchanged and pass `next_cursor` to `--cursor` until it is empty.

`job list --workflow` accepts comma-separated remote references and matches jobs containing any task in any selected scope, not just roots. A job may contain multiple workflow scopes. Use an actual reference here, not `.`; resolve the local reference with `workflow get` first. Unscoped history does not match this filter, and shared workflows do not expose another organization's jobs. `--name` filters job names, not workflow identity.

In `job get`, inspect `state`, `execution_stats`, `task_summaries`, and `progress` first. Task attribution may include `workflow_reference`; absence alone does not prove unscoped execution because deleted/inaccessible workflows may lack display attribution.

## Logs, Spans, Retry, Cancel

```bash
tilebox job logs <job-id> --sort desc --limit 100 --json
tilebox job logs <job-id> --include-runner-attributes --json
tilebox job spans <job-id> --sort asc --limit 100 --json
tilebox job spans <job-id> --include-runner-attributes --json
tilebox job retry <job-id> --json
tilebox job cancel <job-id> --json
```

Use logs for application messages and errors. Use spans for timing, ordering, parent/child relationships, and attributes. Retry only after the underlying issue is fixed. Cancel when work should not continue; queued tasks will not be picked up, while already-running tasks may finish.

Retry preserves the original task inputs and scope. Deploy compatible fixes to that same workflow and cluster; publishing an independent copy does not repair the old job. Keep legacy aliases while old unscoped jobs or automation prototypes still need them.

## Debugging Flow

1. `tilebox job get <job-id> --json` to check state and task counts.
2. For queued work, compare workflow scope, name/major/minor compatibility, cluster deployment, and runner version/capabilities. For failed work, inspect failed task summaries and recent logs.
3. Use spans if timing, ordering, or runner/runtime attributes matter.
4. Retry only after code, data, credentials, or infrastructure are fixed.
5. Cancel if the job should stop instead of being retried.
