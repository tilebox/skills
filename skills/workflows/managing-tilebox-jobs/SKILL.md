---
name: managing-tilebox-jobs
description: "Manages Tilebox workflow jobs with the tilebox job CLI. Use when submitting jobs, listing jobs, inspecting state, listing clusters, creating clusters, waiting for job completion, reading job logs/spans, retrying failed work, or canceling jobs."
license: MIT
compatibility: Requires the tilebox CLI, and a Tilebox API Key ($TILEBOX_API_KEY) or `--api-key`.
metadata:
  author: tilebox
---

# Managing Tilebox Jobs

Use this skill for operational work with `tilebox job` and `tilebox cluster`. For agents, use `--json` on every job command unless explicitly producing human output.

## Refresh CLI Metadata

Check exact installed flags and schemas before relying on memory:

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

- Start work: `tilebox job submit --name ... --task ... --input ... --json`.
- Find jobs: `tilebox job list --last 7d --json` or filter with `--state`, `--task-state`, `--name`.
- Inspect one job: `tilebox job get <job-id> --json`.
- Wait for completion/failure/cancel: `tilebox job wait <job-id> --json`.
- Inspect job log messages: `tilebox job logs <job-id> --sort desc --limit 100 --json`.
- Inspect job traces/spans when debugging timing: `tilebox job spans <job-id> --sort asc --json`.
- Retry eligible failed tasks after fixing the cause: `tilebox job retry <job-id> --json`.
- Stop pending/running work: `tilebox job cancel <job-id> --json`.

Use `tilebox agent-context job <subcommand> --output-schema` when a command's arguments or output shape are unclear. `agent-context` always returns JSON; do not add `--json` to it.

## Submit Jobs

Basic form:

```bash
tilebox job submit \
  --name <job-name> \
  --task <task-identifier-name> \
  --version v0.0 \
  --input '<json-or-plain-text>' \
  --json
```

Important flags:

- `--name`: required job name.
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
tilebox job submit --name process-scene --task ProcessScene --input S2A_001 --json
tilebox job submit --name process-count --task ProcessCount --input 5 --json
tilebox job submit --name process-count --task ProcessCount --input '"5"' --json
tilebox job submit --name structured --task tilebox.com/process_scene --version v1.0 --input '{"scene_id":"S2A_001","other_arg":3}' --json
tilebox job submit --name from-file --task ProcessScenes --input-file scenes.json --json
cat scenes.json | tilebox job submit --name from-stdin --task ProcessScenes --input-file - --json
```

For Python `CronTask` or `StorageEventTask` submissions, use the `working-with-tilebox-automations` skill. Those require `--automation` to construct the automation trigger wrapper.

## Python Task Identifiers And Input

Python `Task` classes default to identifier `<ClassName>@v0.0` unless they define an explicit `identifier()` method. Match the exact task name and version registered by the runner.

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

tilebox job submit --name process-scenes --task ProcessScenes --input-file task-input.json --json
```

## List, Inspect, Wait

```bash
tilebox job list --last 7d --limit 100 --json
tilebox job list --state failed --after 2026-05-01 --before 2026-06-01 --json
tilebox job list --name landsat --task-state failed,failed_optional --json
tilebox job get <job-id> --json
tilebox job wait <job-id> --stalled-timeout 5m --json
```

For paginated list output, keep filters and sort unchanged and pass `next_cursor` to `--cursor` until it is empty.

In `job get`, inspect `state`, `execution_stats`, `task_summaries`, and `progress` first.

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

## Debugging Flow

1. `tilebox job get <job-id> --json` to check state and task counts.
2. If failed, inspect failed task summaries and recent logs.
3. Use spans if timing, ordering, or runner/runtime attributes matter.
4. Retry only after code, data, credentials, or infrastructure are fixed.
5. Cancel if the job should stop instead of being retried.
