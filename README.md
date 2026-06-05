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

Add this repository as a Claude Code plugin marketplace, then install the Tilebox plugin.

```text
/plugin marketplace add tilebox/skills
/plugin install tilebox@tilebox-skills
```

## Skills

### Core

| Skill | Purpose | When to Use |
| --- | --- | --- |
| `using-tilebox-cli` | General `tilebox` CLI usage, authentication, JSON output with `jq`, agent context, docs search, pagination, install, and upgrade guidance | Any task that uses the Tilebox CLI or needs command/output schema discovery |

### Datasets

| Skill | Purpose | When to Use |
| --- | --- | --- |
| `managing-tilebox-datasets` | Manage datasets with `tilebox dataset`: schema design, create/update, markdown docs, collections, datapoint query/find, and generated types | Any task involving Tilebox dataset creation, schema changes, collection management, documentation, or datapoint access |

### Workflows

| Skill | Purpose | When to Use |
| --- | --- | --- |
| `managing-tilebox-jobs` | Manage workflow jobs with `tilebox job`: submit, list, inspect, wait, logs, spans, retry, and cancel | Any task involving workflow job operations, debugging, or root task submission |
| `releasing-tilebox-workflows` | Release and deploy Agentic Workflows with `tilebox.workflow.toml`, build/publish release commands, deployment targets, clusters, and dynamic runners | Any task involving workflow project config, build verification, release publishing, cluster deployment, undeployment, or local runner testing |
| `writing-tilebox-workflows` | Write Python Tilebox workflow code: task classes, task graphs, dataset queries, storage clients, caches, progress labels, logging, tracing, retries, dependencies, runners, and job submission | Any task creating or modifying Tilebox workflow source code |
| `working-with-tilebox-automations` | Work with workflow automations: list, inspect, storage locations, and one-off cron/storage trigger submissions | Any task involving Tilebox automations, automation triggers, or `CronTask` / `StorageEventTask` submissions |

## Quick Start

| You Say | Skill Used |
| --- | --- |
| "Use the Tilebox CLI to list datasets" | `using-tilebox-cli` |
| "Find the output schema for `tilebox job list`" | `using-tilebox-cli` |
| "Extract Tilebox CLI JSON fields with jq" | `using-tilebox-cli` |
| "Create a Tilebox dataset schema" | `managing-tilebox-datasets` |
| "Update a dataset description or schema" | `managing-tilebox-datasets` |
| "Query datapoints from a dataset" | `managing-tilebox-datasets` |
| "Submit a workflow job from the CLI" | `managing-tilebox-jobs` |
| "Check why this Tilebox job failed" | `managing-tilebox-jobs` |
| "Get logs and spans for this workflow job" | `managing-tilebox-jobs` |
| "Retry failed tasks for this job" | `managing-tilebox-jobs` |
| "Publish this workflow release and deploy it to dev" | `releasing-tilebox-workflows` |
| "Edit tilebox.workflow.toml deployment targets" | `releasing-tilebox-workflows` |
| "Start a dynamic runner locally for this workflow cluster" | `releasing-tilebox-workflows` |
| "Write a Python Tilebox workflow" | `writing-tilebox-workflows` |
| "Add progress labels and structured logs to workflow tasks" | `writing-tilebox-workflows` |
| "Design a Tilebox task graph with subtasks and dependencies" | `writing-tilebox-workflows` |
| "List workflow automations" | `working-with-tilebox-automations` |
| "Submit a CronTask or StorageEventTask once" | `working-with-tilebox-automations` |
| "Find automation storage locations" | `working-with-tilebox-automations` |

## Repository Structure

```text
tilebox-skills/
├── .claude-plugin/
│   └── marketplace.json
├── AGENTS.md
├── README.md
└── skills/
    ├── core/
    │   └── using-tilebox-cli/
    │       └── SKILL.md
    ├── datasets/
    │   └── managing-tilebox-datasets/
    │       └── SKILL.md
    └── workflows/
        ├── managing-tilebox-jobs/
        │   └── SKILL.md
        ├── releasing-tilebox-workflows/
        │   └── SKILL.md
        ├── writing-tilebox-workflows/
        │   └── SKILL.md
        └── working-with-tilebox-automations/
            └── SKILL.md
```

## Resources

- [Tilebox Docs](https://docs.tilebox.com)
- [Tilebox CLI repository](https://github.com/tilebox/cli)
- [Tilebox Go SDK](https://github.com/tilebox/tilebox-go)
- [Agent Skills](https://agentskills.io/)
