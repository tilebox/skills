---
name: working-with-tilebox-automations
description: "Works with Tilebox workflow automations using the tilebox CLI. Use when listing or inspecting automations, checking storage locations, or submitting one-off cron/storage automation task triggers."
---

# Working With Tilebox Automations

Use this skill for `tilebox automation` and for one-off automation task submissions with `tilebox job submit --automation`. Use `--json` for agent workflows.

## What The CLI Supports

Read-only automation discovery:

```bash
tilebox automation list --json
tilebox automation get <automation-id> --json
tilebox automation storage-locations --json
```

One-off triggered automation job submissions:

```bash
tilebox job submit --name <job-name> --task <task-name> --automation cron --json
tilebox job submit --name <job-name> --task <task-name> --automation storage --trigger <storage-location-uuid>:/path --json
```

Use `tilebox agent-context automation --output-schema` or `tilebox agent-context job submit --output-schema` when you need exact output schemas or flag metadata. `agent-context` always returns JSON; do not add `--json` to it.

## Inspect Existing Automations

List automations first when you need IDs, trigger definitions, or task prototypes:

```bash
tilebox automation list --json
```

Inspect one automation for full details:

```bash
tilebox automation get <automation-id> --json
```

Important fields:

- `id`, `name`, `disabled`
- `prototype.identifier.name` and `prototype.identifier.version`
- `prototype.cluster_slug`, `prototype.max_retries`, `prototype.input`
- `cron_triggers[].schedule`
- `storage_event_triggers[].storage_location` and `glob_pattern`

## Storage Locations

For storage automations, discover valid storage location UUIDs with:

```bash
tilebox automation storage-locations --json
```

Use the returned `storage_locations[].id` in one-off storage trigger submissions. Keep the trigger path local to that storage location and start it with `/`:

```bash
tilebox job submit \
  --name storage-once \
  --task ProcessStorage \
  --automation storage \
  --trigger 019e4f3c-4646-7312-b8fe-2e7fa83c1546:/incoming/object.tif \
  --json
```

The CLI looks up the storage location by UUID and constructs a `created` storage event trigger payload for the runner.

## Submit One-Off Cron Automation Tasks

Use this for Python `CronTask` classes when you want to run the task once as if the cron fired.

Trigger now:

```bash
tilebox job submit --name cron-once --task ProcessCron --automation cron --json
```

Trigger at an explicit time:

```bash
tilebox job submit \
  --name cron-at-time \
  --task ProcessCron \
  --automation cron \
  --trigger 2026-05-21T12:00:00Z \
  --json
```

`--trigger` must be an RFC3339/RFC3339Nano timestamp. If omitted, the CLI uses the current UTC time.

## Submit One-Off Storage Automation Tasks

Use this for Python `StorageEventTask` classes when you want to run the task once as if an object was created in a storage location.

```bash
tilebox job submit \
  --name storage-once \
  --task ProcessStorage \
  --automation storage \
  --trigger <storage-location-uuid>:/path/in/location \
  --json
```

Rules:

- `--trigger` is required.
- Format is `<storage-location-uuid>:/path/in/location`.
- The path must start with `/`.
- Event type is always `created`.
- The CLI fetches the storage location from the Tilebox API before submitting.

## Automation Task Input

`--input` and `--input-file` still describe the task's normal argument payload. The CLI wraps that payload with the automation trigger event before submitting.

For Python automation task classes, argument serialization follows normal `serialize_task` rules:

- One field: submit the value directly.
- Multiple fields: submit a JSON object keyed by field names.
- One list field: submit a JSON array.

Examples:

```bash
# Single string field
tilebox job submit --name cron-scene --task ProcessCronScene --automation cron --input S2A_001 --json

# Single integer field; submits number 5, not string "5"
tilebox job submit --name cron-count --task ProcessCronCount --automation cron --input 5 --json

# Multiple fields
tilebox job submit --name cron-structured --task ProcessCron --automation cron --input '{"scene_id":"S2A_001","other_arg":3}' --json

# One list field
tilebox job submit --name storage-scenes --task ProcessStorageScenes --automation storage --trigger <storage-location-uuid>:/batch.json --input '["S2A_001","S2B_002"]' --json
```

If unsure, ask Python to show the args bytes:

```python
from tilebox.workflows.automations import CronTask, StorageEventTask

class ExampleCronTask(CronTask):
    name: str
    value: int

assert ExampleCronTask("test", 42)._serialize_args() == b'{"name": "test", "value": 42}'
```

## Relationship To Persistent Automations

`tilebox job submit --automation ...` submits a one-off job with a synthetic trigger event. It does not create or update a persistent automation.

For persistent automation definitions, use the SDK/API. The current CLI can list and inspect automations and storage locations, but it does not create, update, enable/disable, or delete automations.

## Debug After Submitting

The submit command returns a job ID. Then use the job tools:

```bash
tilebox job get <job-id> --json
tilebox job logs <job-id> --sort desc --limit 100 --json
tilebox job spans <job-id> --json
```

Use `--wait` on submit only when a compatible runner is known to be available and expected to execute the task.
