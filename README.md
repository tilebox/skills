# Tilebox Skills

Skills to help AI coding agents work effectively with Tilebox APIs, workflows, and the `tilebox` CLI.

Skills follow the [Agent Skills](https://agentskills.io/) format. Each skill is a directory containing a `SKILL.md` file with skill metadata and instructions.

## Install

### Agent Skills

Install these agent skills with the standalone skills installer, then start a new agent thread so skills are discovered.

```bash
npx skills add tilebox/skills
```

### Claude Code Plugin Marketplace

Add this repository as a Claude Code plugin marketplace, then install the Tilebox plugin groups you need.

```text
/plugin marketplace add tilebox/skills
/plugin install core@tilebox-skills
/plugin install datasets@tilebox-skills
/plugin install workflows@tilebox-skills
```

## Skills

### Core

| Skill | Purpose | When to Use |
| --- | --- | --- |
| `tilebox` | Router for Tilebox skills | Broad Tilebox tasks where the right specialized skill is not obvious |
| `tilebox-cli` | General `tilebox` CLI usage, authentication, JSON output with `jq`, agent context, docs search, pagination, install, and upgrade guidance | Any task that uses the Tilebox CLI or needs command/output schema discovery |

### Datasets

| Skill | Purpose | When to Use |
| --- | --- | --- |
| `tilebox-datasets` | Manage datasets with `tilebox dataset`: schema design, create/update, markdown docs, collections, datapoint query/find, and generated types | Any task involving Tilebox dataset creation, schema changes, collection management, documentation, or datapoint access |

### Workflows

| Skill | Purpose | When to Use |
| --- | --- | --- |
| `tilebox-workflow-authoring` | Write Python Tilebox workflow code: task classes, task graphs, dataset queries, storage clients, caches, progress labels, logging, tracing, retries, dependencies, runners, and job submission | Any task creating or modifying Tilebox workflow source code |
| `tilebox-workflow-releases` | Initialize, configure, release, and deploy Tilebox workflows with `tilebox workflow init`, `tilebox.workflow.toml`, build/publish commands, deployment targets, clusters, and dynamic runners | Any task involving workflow project scaffolding, config, build verification, release publishing, cluster deployment, undeployment, or local runner testing |
| `tilebox-workflow-jobs` | Manage workflow jobs with `tilebox job`: submit, list, inspect, wait, logs, spans, retry, and cancel | Any task involving workflow job operations, debugging, or root task submission |
| `tilebox-workflow-automations` | Work with workflow automations: list, inspect, storage locations, and one-off cron/storage trigger submissions | Any task involving Tilebox automations, automation triggers, or `CronTask` / `StorageEventTask` submissions |

## Quick Start

| You Say | Skill Used |
| --- | --- |
| "Use the Tilebox CLI to list datasets" | `tilebox-cli` |
| "Find the output schema for `tilebox job list`" | `tilebox-cli` |
| "Extract Tilebox CLI JSON fields with jq" | `tilebox-cli` |
| "Create a Tilebox dataset schema" | `tilebox-datasets` |
| "Update a dataset description or schema" | `tilebox-datasets` |
| "Query datapoints from a dataset" | `tilebox-datasets` |
| "Submit a workflow job from the CLI" | `tilebox-workflow-jobs` |
| "Check why this Tilebox job failed" | `tilebox-workflow-jobs` |
| "Get logs and spans for this workflow job" | `tilebox-workflow-jobs` |
| "Retry failed tasks for this job" | `tilebox-workflow-jobs` |
| "Initialize a new Tilebox workflow project" | `tilebox-workflow-releases` |
| "Publish this workflow release and deploy it to dev" | `tilebox-workflow-releases` |
| "Edit tilebox.workflow.toml deployment targets" | `tilebox-workflow-releases` |
| "Start a dynamic runner locally for this workflow cluster" | `tilebox-workflow-releases` |
| "Write a Python Tilebox workflow" | `tilebox-workflow-authoring` |
| "Add progress labels and structured logs to workflow tasks" | `tilebox-workflow-authoring` |
| "Design a Tilebox task graph with subtasks and dependencies" | `tilebox-workflow-authoring` |
| "List workflow automations" | `tilebox-workflow-automations` |
| "Submit a CronTask or StorageEventTask once" | `tilebox-workflow-automations` |
| "Find automation storage locations" | `tilebox-workflow-automations` |

## Repository Structure

```text
tilebox-skills/
├── .claude-plugin/
│   └── marketplace.json
├── AGENTS.md
├── README.md
└── skills/
    ├── core/
    │   ├── tilebox/
    │   │   └── SKILL.md
    │   └── tilebox-cli/
    │       └── SKILL.md
    ├── datasets/
    │   └── tilebox-datasets/
    │       └── SKILL.md
    └── workflows/
        ├── tilebox-workflow-authoring/
        │   └── SKILL.md
        ├── tilebox-workflow-automations/
        │   └── SKILL.md
        ├── tilebox-workflow-jobs/
        │   └── SKILL.md
        └── tilebox-workflow-releases/
            └── SKILL.md
```

## Resources

- [Tilebox Docs](https://docs.tilebox.com)
- [Tilebox CLI repository](https://github.com/tilebox/cli)
- [Tilebox Go SDK](https://github.com/tilebox/tilebox-go)
- [Agent Skills](https://agentskills.io/)
