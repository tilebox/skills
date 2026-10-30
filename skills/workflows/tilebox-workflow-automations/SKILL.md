---
name: tilebox-workflow-automations
description: "Manages scoped Tilebox workflow automations with the CLI and Python SDK. Use when creating, updating, pausing, deleting, listing, or inspecting cron/storage automations, checking storage locations, selecting task scope, or submitting one-off triggers."
license: MIT
compatibility: Requires the tilebox CLI, and a Tilebox API Key ($TILEBOX_API_KEY) or `--api-key`.
metadata:
  author: tilebox
---

# Working With Tilebox Automations

Use this skill for `tilebox automation` and for one-off automation task submissions with `tilebox job submit --automation`. Use `--json` for agent workflows.

## Refresh Docs And CLI Metadata

Use `tilebox-cli` to ensure the CLI is up to date (scoped-project baseline >=0.14). In workflow projects, regularly check/update Tilebox Python packages (baseline >=0.68) using authoring's dependency guidance. Check release availability and installed flags/schemas before relying on memory:

```bash
tilebox agent-context automation --output-schema
tilebox agent-context job submit --output-schema
```

Relevant docs concepts:

- Automations submit workflow jobs automatically when external trigger conditions are met.
- Supported trigger types are cron schedules and storage events; dataset event triggers are on the roadmap.
- Automation prototypes store workflow scope separately from task name/version and input parameters. At execution, the task also has a `trigger` attribute describing the event.
- The CLI and SDK can manage persistent automations. One-off `job submit --automation` calls do not create persistent definitions.
- The Tilebox Console can create, edit, delete, and inspect automations.

## What The CLI Supports

Read-only automation discovery:

```bash
tilebox automation list --json
tilebox automation get <automation-id> --json
tilebox storage-location list --json
```

One-off triggered automation job submissions:

```bash
tilebox job submit --workflow . --name <job-name> --task <task-name> --automation cron --json
tilebox job submit --workflow . --name <job-name> --task <task-name> --automation storage --trigger <storage-location-uuid>:/path --json
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
- `prototype.workflow_id` (optional; absent means unscoped). Resolve references with `workflow get` when comparing scope.
- `prototype.cluster_slug`, `prototype.max_retries`, `prototype.input`
- `cron_triggers[].schedule`
- `storage_event_triggers[].storage_location` and `glob_pattern`

## Submit One-Off Cron Automation Tasks

Use this for Python `CronTask` classes when you want to run the task once as if the cron fired.

Trigger now:

```bash
tilebox job submit --workflow . --name cron-once --task ProcessCron --automation cron --json
```

Trigger at an explicit time:

```bash
tilebox job submit \
  --workflow . \
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
  --workflow . \
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
tilebox storage-location list --json
```

Use `storage-location list`, not the removed `automation storage-locations`. Inspect each location's `id`, `name`, and typed `reference` before choosing a trigger destination; a workflow's local link does not select storage or configure notification delivery.

## Automation Task Input

`--input` and `--input-file` still describe the task's normal argument payload. The CLI wraps that payload with the automation trigger event before submitting.

For Python automation task classes, argument serialization follows normal `serialize_task` rules:

- One field: submit the value directly.
- Multiple fields: submit a JSON object keyed by field names.
- One list field: submit a JSON array.

Examples:

```bash
# Single string field
tilebox job submit --workflow . --name cron-scene --task ProcessCronScene --automation cron --input S2A_001 --json

# Single integer field; submits number 5, not string "5"
tilebox job submit --workflow . --name cron-count --task ProcessCronCount --automation cron --input 5 --json

# Multiple fields
tilebox job submit --workflow . --name cron-structured --task ProcessCron --automation cron --input '{"scene_id":"S2A_001","other_arg":3}' --json

# One list field
tilebox job submit --workflow . --name storage-scenes --task ProcessStorageScenes --automation storage --trigger <storage-location-uuid>:/batch.json --input '["S2A_001","S2B_002"]' --json
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

Always scope workflow-project automation jobs and new persistent prototypes: use `--workflow .` from a linked project or `--workflow <reference>` for an explicit owned/shared execution target. This includes one-off trigger tests. Unscoped jobs are only for an explicitly requested direct Python runner or a Python script executed as a direct runner for testing, as described in `tilebox-workflow-jobs`; a local managed runner is not an exception. Omission stays unscoped even in a project. Failed local/reference resolution must not fall back to unscoped work. New projects must be published or explicitly linked first; use `tilebox-workflow-releases` for that lifecycle.

Copying a template or linking a checkout does not copy, migrate, or retarget persistent automations. Inspect each existing prototype before changing it. A copied workflow needs its own scoped prototype to execute the copy rather than the original. Do not put scope in task arguments or the trigger wrapper.

## Manage Persistent Automations With The CLI

Inspect `agent-context automation create`, `update`, and `delete` before writing. Creation is enabled by default and matching triggers may immediately submit jobs. Create disabled while preparing/testing unless immediate activation was requested; enable after checking scope, inputs, deployment, runner availability, and the user's intent.

```bash
tilebox automation create "Daily scene processing" \
  --workflow . --task ProcessCron --version v1.0 \
  --cron-trigger 'CRON_TZ=Europe/Vienna 0 7 * * *' \
  --input-file args.json --disabled --json

tilebox automation create "Process arrivals" \
  --workflow . --task ProcessStorage --version v1.0 \
  --storage-trigger '<storage-location-uuid>:incoming/**/*.tif' \
  --input-file args.json --disabled --json

tilebox automation update <automation-id> --workflow . --json
tilebox automation update <automation-id> --disabled=false --json
tilebox automation update <automation-id> --disabled --json
tilebox automation delete <automation-id> --json
```

Persistent automation changes are remote writes. Confirm deletion or retargeting unless explicitly authorized. Pausing stops future triggers, not jobs already submitted.

- Supply 1–32 triggers of one type. Repeat `--cron-trigger` or `--storage-trigger`; do not combine types. Cron uses UTC unless `CRON_TZ` selects a timezone. Storage notification delivery must already be configured.
- `--input`/`--input-file` supply ordinary task arguments; the server adds the event wrapper. Do not use `--automation` or one-off `--trigger` flags on `automation create`.
- Updates preserve omitted fields, including workflow scope. `--workflow <reference>` or `.` retargets future jobs; `--workflow ''` deliberately clears scope to unscoped. Never clear scope merely to repair a failed lookup.
- Supplying trigger flags on update replaces the entire trigger list. The CLI reads and merges the current definition; concurrent edits can overwrite one another. Fetch and verify the result.
- Existing unscoped prototypes keep needing legacy aliases until explicitly migrated. Updates do not change already-submitted jobs. Deploy compatible releases to the same workflow UUID for queued work and retries.

## Register Persistent Automations With Python

Define an automation task class (`CronTask` or `StorageEventTask`), then call the matching `create_*_automation` method on `client.automations()`. These examples create active remote automations; execute only when activation is intended. Pass `workflow=` as a remote reference or `Workflow` object; Python does not read the CLI's local link through `workflow="."`. Omission/`None` is unscoped.

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
    workflow="my-workflow-reference",
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

# Resolve the exact registered location selected with storage-location list.
gcs_bucket = client.storage_locations().find("<storage-location-uuid>")

storage_automation = automations.create_storage_event_automation(
    "log-text-object-creations",
    LogObjectCreation(head_bytes=20),
    triggers=[
        (gcs_bucket, "**.txt"),  # match .txt files anywhere in the storage location
    ],
    workflow="my-workflow-reference",
)
print(storage_automation)
```

After registering, verify from the CLI:

```bash
tilebox automation list --json
tilebox automation get <automation-id> --json
```

Verify `prototype.workflow_id` against the selected workflow's `id`, along with task name/version, input, triggers, disabled state, and cluster. Make sure an eligible runner advertises that scope on the target cluster; otherwise triggered jobs remain queued. For raw SDK/API updates, preserve the current prototype's scope explicitly: the backend replaces the prototype, whereas CLI patch-style omission preserves it.

## Debug After Submitting

The submit command returns a job ID. Then use the job tools:

```bash
tilebox job get <job-id> --json
tilebox job logs <job-id> --sort desc --limit 100 --json
tilebox job spans <job-id> --json
```

Use `--wait` on submit only when a compatible runner is known to be available and expected to execute the task.
