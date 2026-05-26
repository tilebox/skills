---
name: working-with-tilebox-automations
description: "Works with Tilebox workflow automations using the tilebox CLI. Use when listing or inspecting automations, checking storage locations, or submitting one-off cron/storage automation task triggers."
license: MIT
compatibility: Requires the tilebox CLI, and a Tilebox API Key ($TILEBOX_API_KEY) or `--api-key`.
metadata:
  author: tilebox
---

# Working With Tilebox Automations

Use this skill for `tilebox automation` and for one-off automation task submissions with `tilebox job submit --automation`. Use `--json` for agent workflows.

## Refresh Docs And CLI Metadata

Check exact installed flags and schemas before relying on memory:

```bash
tilebox agent-context automation --output-schema
tilebox agent-context job submit --output-schema
```

Relevant docs concepts:

- Automations submit workflow jobs automatically when external trigger conditions are met.
- Supported trigger types are cron schedules and storage events; dataset event triggers are on the roadmap.
- Automation tasks have normal task identifiers, versions, and input parameters, plus a `trigger` attribute describing the event.
- Python can register automations; Go registration is not supported yet. The CLI currently inspects automations/storage locations and submits one-off automation-triggered jobs.
- The Tilebox Console can create, edit, delete, and inspect automations.

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

## Storage Locations

To discover valid storage location UUIDs:

```bash
tilebox automation storage-locations --json
```

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

## Register Persistent Automations With Python

Persistent automations are registered with the Python SDK. Define an automation task class (`CronTask` or `StorageEventTask`), then call the matching `create_*_automation` method on `client.automations()`.

Cron automation example:

```python
from tilebox.workflows import Client, ExecutionContext
from tilebox.workflows.automations import CronTask


class SendHeartbeat(CronTask):
    message: str

    def execute(self, context: ExecutionContext) -> None:
        context.logger.info(
            "Cron task triggered",
            message=self.message,
            trigger_time=self.trigger.time,
        )


client = Client()
automations = client.automations()

cron_automation = automations.create_cron_automation(
    "send-heartbeat",
    SendHeartbeat(message="hello"),
    cron_schedules=[
        "15 * * * *",   # every hour at minute 15
        "45 18 * * *",  # every day at 18:45
    ],
)
print(cron_automation)
```

Storage event automation example:

```python
from tilebox.workflows import Client, ExecutionContext
from tilebox.workflows.automations import StorageEventTask, StorageEventType


class LogObjectCreation(StorageEventTask):
    head_bytes: int

    def execute(self, context: ExecutionContext) -> None:
        if self.trigger.type == StorageEventType.CREATED:
            path = self.trigger.location
            data = self.trigger.storage.read(path)
            context.logger.info(
                "Object created",
                path=path,
                size_bytes=len(data),
                preview=data[: self.head_bytes].hex(),
            )


client = Client()
automations = client.automations()
storage_locations = automations.storage_locations()

# Choose the registered storage location you want to watch.
gcs_bucket = storage_locations[0]

storage_automation = automations.create_storage_event_automation(
    "log-text-object-creations",
    LogObjectCreation(head_bytes=20),
    triggers=[
        (gcs_bucket, "**.txt"),  # match .txt files anywhere in the storage location
    ],
)
print(storage_automation)
```

After registering, verify from the CLI:

```bash
tilebox automation list --json
tilebox automation get <automation-id> --json
```

Make sure an eligible task runner is running for the automation task identifier/version and target cluster; otherwise triggered jobs remain queued until a runner is available.

## Debug After Submitting

The submit command returns a job ID. Then use the job tools:

```bash
tilebox job get <job-id> --json
tilebox job logs <job-id> --sort desc --limit 100 --json
tilebox job spans <job-id> --json
```

Use `--wait` on submit only when a compatible runner is known to be available and expected to execute the task.
