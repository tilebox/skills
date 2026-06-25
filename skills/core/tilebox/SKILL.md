---
name: tilebox
description: "Tilebox router. Use when a task involves Tilebox CLI commands, datasets, datapoints, schema design, workflow task code, geospatial raster/grid/projection patterns, workflow releases, deployments, jobs, clusters, logs, traces, or automations. Automatically routes to the specific Tilebox skill or combination of skills based on the task."
license: MIT
metadata:
  author: tilebox
---

# Tilebox Skills Router

Use this router first for broad Tilebox requests, then switch to the most specific skill or skills for the actual work. This skill is an index; it does not replace the detailed operational guidance in the routed skills.

## Context Detection

Check the user's request and nearby project files to determine the Tilebox surface area:

| Signal | Route to |
| --- | --- |
| `tilebox` shell commands, auth, `--json`, `agent-context`, or docs search | CLI usage |
| Dataset schemas, collections, datapoints, generated types, or dataset docs | Dataset operations |
| Python task classes, task graphs, `ExecutionContext`, subtasks, caches, logs, tracing, or geospatial processing code | Workflow code authoring |
| Raster IO, COGs, Zarr, object storage, CRS, reprojection, regridding, masks, nodata, tiling, windowing, Sentinel-2, Landsat, or geospatial ML inference | Workflow code authoring |
| `tilebox.workflow.toml`, release artifacts, deployment targets, clusters, or dynamic runners | Workflow releases and deployment |
| Job submission, job state, retries, cancellation, logs, spans, or cluster inspection | Workflow job operations |
| `CronTask`, `StorageEventTask`, automation storage locations, or `--automation` submissions | Workflow automations |

When multiple areas apply, route to all relevant skills. Prefer the skill for the first concrete action as the primary guide, then use the companion skills for the parts of the task they own.

---

## By Task

**Operating Tilebox from the CLI** → Use `tilebox-cli`
- Authentication and API URL handling
- Machine-readable `--json` output
- `tilebox agent-context` schema discovery
- Docs search, pagination, installation, and upgrades

**Managing datasets** → Use `tilebox-datasets`
- Dataset creation, updates, and documentation
- Schema design and generated dataset types
- Collection management
- Datapoint queries, filters, and lookup

**Writing workflow task code** → Use `tilebox-workflow-authoring`
- Python task classes and task graphs
- Subtasks, dependencies, retries, and fanout patterns
- Dataset queries, storage clients, and caches from workflow code
- Progress labels, structured logs, tracing, and local code iteration
- Geospatial workflow patterns: raster metadata, grids, CRS/projections, reprojection/regridding, COG/Zarr outputs, object-storage raster IO, nodata/masks, tiling/windowing, Sentinel-2, Landsat, and tiled ML inference

**Releasing and deploying workflows** → Use `tilebox-workflow-releases`
- `tilebox workflow init` and project scaffolding
- `tilebox.workflow.toml` build and deployment configuration
- Release build, publish, deploy, and undeploy operations
- Dynamic runners and local release-runner testing

**Managing workflow jobs** → Use `tilebox-workflow-jobs`
- Job submission, listing, inspection, and waiting
- Logs, spans, traces, and execution debugging
- Retrying failed work or canceling running work
- Cluster listing and creation for job execution

**Working with workflow automations** → Use `tilebox-workflow-automations`
- Listing and inspecting workflow automations
- Finding automation storage locations
- Submitting one-off cron or storage-triggered automation tasks
- Building `CronTask` and `StorageEventTask` job submissions

## Common Multi-Skill Routes

Use multiple skills when the task spans design, code, release, and operations:

| User task | Use these skills |
| --- | --- |
| Create a dataset and use it from workflow code | `tilebox-datasets` + `tilebox-workflow-authoring` |
| Scaffold a new workflow project and write task code | `tilebox-workflow-releases` + `tilebox-workflow-authoring` |
| Publish a workflow change and retry a failed job | `tilebox-workflow-releases` + `tilebox-workflow-jobs` |
| Debug a failed workflow task from logs and fix the code | `tilebox-workflow-jobs` + `tilebox-workflow-authoring` |
| Build geospatial raster processing, reprojection, grid alignment, COG/Zarr output, or satellite-data workflow code | `tilebox-workflow-authoring` + `tilebox-datasets` when dataset queries or schemas are involved |
| Add a cron or storage-triggered workflow and test it once | `tilebox-workflow-automations` + `tilebox-workflow-jobs` |
| Use CLI command schemas while doing dataset, job, release, or automation work | `tilebox-cli` + the relevant domain skill |

If a task needs more than one skill, keep each skill's boundary clear: use the domain skill for product behavior and `tilebox-cli` for command mechanics, JSON output, auth, pagination, and schema discovery.

## Quick Navigation

If you know your task, you can directly access:

- `/tilebox-cli` - CLI commands, auth, JSON output, docs search, and command schemas
- `/tilebox-datasets` - Dataset schemas, collections, datapoints, docs, and generated types
- `/tilebox-workflow-authoring` - Python workflow task code, task graphs, subtasks, caches, logs, tracing, and geospatial raster/grid/projection patterns
- `/tilebox-workflow-releases` - Workflow init, release build/publish, deployment, targets, and runners
- `/tilebox-workflow-jobs` - Job submission, monitoring, logs, spans, retries, cancellation, and clusters
- `/tilebox-workflow-automations` - Workflow automations, cron/storage triggers, and one-off automation submissions

Or describe what you need and route to the right Tilebox skill or skills before acting.
