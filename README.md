# Tilebox Skills

Skills to help AI coding agents work effectively with Tilebox APIs, workflows, and the `tilebox` CLI.

Skills follow the [Agent Skills](https://agentskills.io/) format. Each skill is a directory containing a `SKILL.md` file with skill metadata and instructions.

## Install

### Amp / Agent Skills

Clone or install this repository into an agent skills location, then start a new agent thread so skills are discovered.

```bash
git clone https://github.com/tilebox/skills ~/.config/agents/skills/tilebox
```

For local development, keep this repository checked out and copy or symlink the skill directories into your agent skills directory as needed.

## Skills

### Core

| Skill | Purpose | When to Use |
| --- | --- | --- |
| `using-tilebox-cli` | General `tilebox` CLI usage, authentication, JSON output, agent context, pagination, install, and upgrade guidance | Any task that uses the Tilebox CLI or needs command/output schema discovery |

### Workflows

| Skill | Purpose | When to Use |
| --- | --- | --- |
| `managing-tilebox-jobs` | Manage workflow jobs with `tilebox job`: submit, list, inspect, wait, logs, spans, retry, and cancel | Any task involving workflow job operations, debugging, or root task submission |

## Quick Start

| You Say | Skill Used |
| --- | --- |
| "Use the Tilebox CLI to list datasets" | `using-tilebox-cli` |
| "Find the output schema for `tilebox job list`" | `using-tilebox-cli` |
| "Submit a workflow job from the CLI" | `managing-tilebox-jobs` |
| "Check why this Tilebox job failed" | `managing-tilebox-jobs` |
| "Get logs and spans for this workflow job" | `managing-tilebox-jobs` |
| "Retry failed tasks for this job" | `managing-tilebox-jobs` |

## Repository Structure

```text
tilebox-skills/
├── README.md
└── skills/
    ├── core/
    │   └── using-tilebox-cli/
    │       └── SKILL.md
    └── workflows/
        └── managing-tilebox-jobs/
            └── SKILL.md
```

## Resources

- [Tilebox Docs](https://docs.tilebox.com)
- [Tilebox CLI repository](https://github.com/tilebox/cli)
- [Tilebox Go SDK](https://github.com/tilebox/tilebox-go)
- [Agent Skills](https://agentskills.io/)
