---
name: managing-tilebox-jobs
description: "Manages Tilebox workflow jobs with the tilebox job CLI. Use when listing, submitting, inspecting, waiting for, debugging logs/spans, retrying, or canceling workflow jobs."
---

# Managing Tilebox Jobs

Use this skill for Tilebox workflow job operations through `tilebox job`. Always prefer `--json` for agent workflows.

## Command Overview

```bash
tilebox job submit   # submit a workflow root task as a job
tilebox job list     # query jobs by time, state, task state, name, cursor
tilebox job get      # inspect one job in detail
tilebox job wait     # wait for terminal job state
tilebox job logs     # query job log records
tilebox job spans    # query job trace spans
tilebox job retry    # reschedule eligible failed tasks
tilebox job cancel   # cancel a running or pending job
```

Authentication uses `TILEBOX_API_KEY` or `--api-key`. Use `--api-url` only for non-default environments.

## Choosing The Right Command

- Need recent jobs or a job ID: `tilebox job list --last 7d --json`.
- Need full state, task summaries, progress, and execution stats: `tilebox job get <job-id> --json`.
- Need to block until completion/failure/cancel: `tilebox job wait <job-id> --json`.
- Need task/application messages: `tilebox job logs <job-id> --json`.
- Need timing, spans, attributes, or trace structure: `tilebox job spans <job-id> --json`.
- Need to rerun failed tasks after fixing code/resources: `tilebox job retry <job-id> --json`.
- Need to stop queued work from being picked up: `tilebox job cancel <job-id> --json`.
- Need to start new work: `tilebox job submit --name ... --task ... --input ... --json`.

## Listing Jobs

List recent jobs:

```bash
tilebox job list --last 7d --json
```

Filter by job state:

```bash
tilebox job list --state running --json
tilebox job list --state failed --after 2026-05-01 --before 2026-06-01 --json
```

Filter by task state and name:

```bash
tilebox job list --name landsat --task-state failed,failed_optional --json
```

Pagination:

```bash
tilebox job list --last 7d --limit 100 --json
tilebox job list --last 7d --limit 100 --cursor <next_cursor> --json
```

Keep all filters and `--sort` unchanged between pages. Stop when `next_cursor` is empty.

## Inspecting And Waiting

Get details:

```bash
tilebox job get <job-id> --json
```

Read these fields first:

- `state`: `submitted`, `running`, `started`, `completed`, `failed`, or `canceled`.
- `execution_stats.total_tasks` and `execution_stats.tasks_by_state`.
- `task_summaries`: IDs, display names, states, parent IDs, start/stop times.
- `progress`: workflow progress indicators.

Wait for terminal state:

```bash
tilebox job wait <job-id> --json
```

Use a longer stalled timeout if runners may take time to pick up submitted/started jobs:

```bash
tilebox job wait <job-id> --stalled-timeout 5m --json
```

`wait` stops with `timeout: true` when a job remains `submitted` or `started` longer than the stalled timeout. Running jobs are waited on indefinitely.

## Logs And Spans

Latest logs:

```bash
tilebox job logs <job-id> --sort desc --limit 50 --json
```

Include runner attributes when diagnosing environment/runtime issues:

```bash
tilebox job logs <job-id> --include-runner-attributes --json
```

Spans for timing and trace analysis:

```bash
tilebox job spans <job-id> --sort asc --limit 100 --json
tilebox job spans <job-id> --include-runner-attributes --json
```

Logs and spans are paginated. Use `next_cursor` with `--cursor` for the next page.

Use logs for application messages and errors. Use spans for duration, ordering, parent/child timing, and attributes.

## Submitting Jobs

Basic form:

```bash
tilebox job submit \
  --name <job-name> \
  --task <task-identifier-name> \
  --version v0.0 \
  --input '<json-or-plain-text>' \
  --json
```

Flags:

- `--name`: required job name.
- `--task`: required task identifier name.
- `--version`: task identifier version, default `v0.0`.
- `--input`: inline input. Valid JSON passes through. Non-JSON text becomes a JSON string.
- `--input-file`: read input from a file. Use `-` for stdin.
- `--cluster`: optional cluster slug. Omit it for the default cluster.
- `--max-retries`: root task retry count, default `0`.
- `--wait`: after submission, wait until the job completes, fails, is canceled, or stalls before running. This has the same practical effect as running `tilebox job wait <new-job-id>` afterwards with the default stalled timeout.

Only use `--wait` when a runner is known to be available and expected to pick up and execute the submitted task. If runners are not deployed, are on the wrong cluster, or may not register the submitted task identifier, prefer submitting without `--wait`, then inspect with `job get`, `job logs`, or `job spans`.

Examples:

```bash
tilebox job submit --name process-scene --task ProcessScene --input S2A_001 --json
tilebox job submit --name process-count --task ProcessCount --input 5 --json
tilebox job submit --name process-count --task ProcessCount --input '"5"' --json
tilebox job submit --name structured --task tilebox.com/process_scene --version v1.0 --input '{"scene_id":"S2A_001","other_arg":3}' --json
tilebox job submit --name from-file --task ProcessScenes --input-file scenes.json --json
cat scenes.json | tilebox job submit --name from-stdin --task ProcessScenes --input-file - --json
tilebox job submit --name process-and-wait --task ProcessScene --input S2A_001 --wait --json
```

### Python Task Identifier Rules

For Python tasks deriving from `tilebox.workflows.Task`:

- If the task class defines `identifier()` as a static/class method returning `(name, version)`, use those exact values.
- If no explicit identifier is set, the default name is the Python class name and the default version is `v0.0`.

Default example:

```python
class ProcessScenes(Task):
    scene_ids: list[str]
```

Submit with:

```bash
tilebox job submit --name process-scenes --task ProcessScenes --input-file scenes.json --json
```

Explicit identifier example:

```python
class ProcessScenes(Task):
    scene_ids: list[str]

    @staticmethod
    def identifier() -> tuple[str, str]:
        return ("tilebox.com/process_scenes", "v1.2")
```

Submit with:

```bash
tilebox job submit --name process-scenes --task tilebox.com/process_scenes --version v1.2 --input-file scenes.json --json
```

### Python Task Input Serialization Rules

The Python workflow SDK serializes task input with `serialize_task(task)` and reads it with `deserialize_task(TaskClass, bytes)`. Match that byte shape when using the CLI.

For a task with **no serializable fields**, omit input or use `{}`:

```bash
tilebox job submit --name empty --task EmptyTask --json
```

For a task with **one field**, submit the field value directly as JSON:

```python
class ProcessScene(Task):
    scene_id: str
```

```bash
tilebox job submit --name process-scene --task ProcessScene --input S2A_001 --json
```

The CLI sees `S2A_001` is not valid JSON and submits the JSON string `"S2A_001"`.

For a single integer field:

```python
class ProcessCount(Task):
    count: int
```

```bash
tilebox job submit --name process-count --task ProcessCount --input 5 --json
```

The CLI sees `5` is valid JSON and submits the number `5`. To submit the string `"5"`, use `--input '"5"'`.

For a single list field:

```python
class ProcessScenes(Task):
    scene_ids: list[str]
```

Create `scenes.json` as a JSON array, not an object:

```json
["S2A_001", "S2B_002"]
```

Submit:

```bash
tilebox job submit --name process-scenes --task ProcessScenes --input-file scenes.json --json
```

For a task with **multiple fields**, submit a JSON object keyed by dataclass field names:

```python
class ProcessScene(Task):
    scene_id: str
    other_arg: int
```

```bash
tilebox job submit --name process-scene --task ProcessScene --input '{"scene_id":"S2A_001","other_arg":3}' --json
```

### Producing And Verifying Input With Python Serialization

When in doubt, ask Python to produce the exact bytes using the same SDK code as the runner, then verify those bytes with `deserialize_task` before submitting.

Inline example:

```bash
/path/to/.venv/bin/python - <<'PY' > task-input.json
from test import ProcessScenes
from tilebox.workflows.task import serialize_task, deserialize_task

task = ProcessScenes(["S2A_001", "S2B_002"])
payload = serialize_task(task)

# Optional round-trip check: this should reconstruct the same task shape that
# the runner will execute.
round_trip = deserialize_task(ProcessScenes, payload)
assert round_trip.scene_ids == task.scene_ids

print(payload.decode())
PY

tilebox job submit --name process-scenes --task ProcessScenes --input-file task-input.json --json
```

Use this pattern when field count is ambiguous, when nested dataclasses are involved, or when a task has an explicit identifier in code and you want to confirm the submitted input shape.

If the serialized bytes are not UTF-8 JSON because the single field is a protobuf message, the current CLI JSON input mode is not appropriate; submit from SDK code instead.

### Automation Task Input: CronTask And StorageEventTask

Python automation task classes have two different serialization shapes. Be careful not to confuse them.

- `CronTask` and `StorageEventTask` inherit from `Task`, so their **argument payload** follows normal `serialize_task` rules.
- Their `_serialize_args()` method returns only those normal task arguments. This is what automation prototypes store when creating cron or storage-event automations.
- Their `_serialize()` method is different: it requires a trigger event and returns a binary protobuf `Automation` wrapper containing `trigger_event` plus `args`.
- When an automation-triggered job runs, the runner calls the task class `_deserialize(...)`, unwraps the trigger event, deserializes `args`, and sets `task.trigger`.

Implication for `tilebox job submit`: the CLI's `--input` and `--input-file` are JSON-oriented and should not be used to manually submit `CronTask` or `StorageEventTask` subclasses as if they were regular tasks. A runner for those classes expects the binary automation wrapper, not just the args JSON. Use the automation API/SDK to create or trigger automations, or submit a non-automation `Task` wrapper if you need manual CLI-triggered work.

Confirmed examples from the Python serializer:

```python
from tilebox.workflows.automations import CronTask, StorageEventTask
from tilebox.workflows.task import serialize_task, deserialize_task

class ExampleCronTask(CronTask):
    name: str
    value: int

class SingleArgCronTask(CronTask):
    scene_id: str

class ExampleStorageEventTask(StorageEventTask):
    name: str
    value: int

assert ExampleCronTask("test", 42)._serialize_args() == b'{"name": "test", "value": 42}'
assert SingleArgCronTask("S2A_001")._serialize_args() == b'"S2A_001"'
assert ExampleStorageEventTask("test", 42)._serialize_args() == b'{"name": "test", "value": 42}'

# _serialize_args() is equivalent to normal task serialization for the args.
payload = serialize_task(ExampleCronTask("test", 42))
assert payload == ExampleCronTask("test", 42)._serialize_args()
assert deserialize_task(ExampleCronTask, payload) == ExampleCronTask("test", 42)
```

For actual triggered automation payloads, `_serialize()` is binary and includes trigger metadata:

```python
from datetime import datetime, timezone
from uuid import UUID
from tilebox.workflows.data import StorageEventType, StorageLocation, StorageType

triggered_cron = ExampleCronTask("test", 42).once(
    datetime(2026, 5, 21, 12, 0, tzinfo=timezone.utc)
)
cron_wire_payload = triggered_cron._serialize()  # binary Automation protobuf, not JSON

storage_location = StorageLocation(
    UUID("019e4f3c-4646-7312-b8fe-2e7fa83c1546"),
    "/tmp/tilebox",
    StorageType.FS,
)
triggered_storage = ExampleStorageEventTask("test", 42).once(
    storage_location,
    "FM171/apid.json",
    StorageEventType.CREATED,
)
storage_wire_payload = triggered_storage._serialize()  # binary Automation protobuf, not JSON
```

If you are creating automations from Python, pass task instances to the automation client and let the SDK use `_serialize_args()`:

```python
client.automations().create_cron_automation(
    "nightly-process",
    ExampleCronTask("test", 42),
    "0 0 * * *",
)

client.automations().create_storage_event_automation(
    "process-new-files",
    ExampleStorageEventTask("test", 42),
    (storage_location, "*.json"),
)
```

## Retry And Cancel

Retry a failed job after fixing task code, data, credentials, or infrastructure:

```bash
tilebox job retry <job-id> --json
```

The response includes `rescheduled_tasks`.

Cancel a running or pending job when the work should stop:

```bash
tilebox job cancel <job-id> --json
```

Cancellation prevents queued tasks from being picked up. Tasks already running may finish.

## Debugging Workflow

1. `tilebox job get <job-id> --json` to understand state and task counts.
2. If failed, inspect `task_summaries` for failed task display names.
3. Query recent logs: `tilebox job logs <job-id> --sort desc --limit 100 --json`.
4. Query spans if timing/order matters: `tilebox job spans <job-id> --json`.
5. If the fix is deployed and retry is appropriate, run `tilebox job retry <job-id> --json`.
6. If the job should not continue, run `tilebox job cancel <job-id> --json`.
