---
name: using-tilebox-cli
description: "Use the Tilebox CLI for datasets, workflows, docs search, automations, parallel task runners. Use when a task involves tilebox commands. How to handle authentication, JSON output, agent-context schemas, pagination, installation, or upgrades."
license: MIT
metadata:
  author: tilebox
---

# Using Tilebox CLI

Use this skill whenever interacting with the `tilebox` command-line tool. Prefer machine-readable output and command schema discovery so automation remains robust.

## Core Rules For Agents

- Prefer `--json` for commands that return data or status.
- Use `tilebox agent-context <command path> --output-schema` before relying on a command's output shape.
- Pass authentication via `TILEBOX_API_KEY` unless the user explicitly asks to use `--api-key`.
- Use `--api-url` only when targeting a non-default API environment.
- For paginated commands, read `next_cursor` from JSON output and pass it back as `--cursor` until it is empty.
- Use `tilebox agent-context <command>` when behavior is unclear.

## Authentication And API URL

The CLI authenticates with either:

```bash
export TILEBOX_API_KEY=...
tilebox dataset list --json
```

or per command:

```bash
tilebox dataset list --api-key "$TILEBOX_API_KEY" --json
```

The default API is `https://api.tilebox.com`. Override it for staging or local environments:

```bash
# a staging env
tilebox --api-url https://api.tilebox.dev dataset list --json
```

If auth is missing, commands return a validation-style usage error. Do not print or log API keys.

## JSON Output

Use `--json` by default in agent workflows:

```bash
tilebox dataset list --json
tilebox job list --last 7d --json
tilebox job get <job-id> --json
```

Human output may be a table or rich TUI. JSON output is stable for automation and easier to parse.

## Combine JSON Output With `jq`

Use `jq` for quick field extraction, filtering, and shell pipelines. Keep `tilebox` responsible for structured output and `jq` responsible for selecting the fields you need. Prefer keeping intermediate and final output as JSON objects or arrays.

Examples:

```bash
# List dataset slugs
tilebox dataset list --json | jq '[.[].slug]'

# Extract a submitted job ID
JOB_ID=$(tilebox job submit --name <job-name> --task <task-name> --input '{}' --json | jq -r '.id')

# Inspect failed jobs from a query response
tilebox job list --last 7d --state failed --json | jq '{jobs: [.jobs[] | {id, state, name}]}'

# Page through commands manually by reading next_cursor
tilebox job logs <job-id> --limit 100 --json | jq -r '.next_cursor'

# Read automation storage location IDs and locations
tilebox automation storage-locations --json | jq '{storage_locations: [.storage_locations[] | {id, type, location}]}'
```

Use `jq -e` when a script should fail if a required value is missing:

```bash
tilebox job get <job-id> --json | jq -e '.state == "completed"'
```

## Discovering Commands And Output Schemas

Use `agent-context` to inspect available commands, arguments, flags, descriptions, and output schemas.
It always returns JSON; do not add `--json` to `agent-context` commands.

Describe the whole CLI:

```bash
tilebox agent-context
```

Describe one command:

```bash
tilebox agent-context job list --output-schema
```

Typical workflow:

1. Run `tilebox agent-context <command path> --output-schema`.
2. Read required args/flags and the JSON output schema.
3. Run the command with `--json`.
4. Parse fields according to the schema.

## Searching Tilebox Docs

Use `tilebox docs search` to browse and retrieve relevant excerpts from `docs.tilebox.com` without leaving the CLI. It is useful when you need current product documentation, conceptual guidance, examples, or SDK/API details before choosing command flags or implementation details.

```bash
tilebox docs search "dataset schema custom fields"
tilebox docs search "query datasets temporal extent spatial extent"
tilebox docs search "workflow job retry logs spans"
```

Search with natural-language phrases that include the product area and the exact concept, command, SDK type, or error you care about. Prefer a focused query over a broad one:

```bash
# Good: scoped to a feature and expected terminology
tilebox docs search "dataset query spatial extent GeoJSON Polygon"

# Too broad: likely to return mixed concepts
tilebox docs search "query"
```

Use docs search when:

- `agent-context` tells you the CLI shape, but you need conceptual docs or examples.
- You need SDK or API behavior that may not be obvious from CLI help.
- You want to confirm current docs terminology before writing user-facing documentation.

Do not use docs search for command output schemas; use `tilebox agent-context <command path> --output-schema` for that.

## Pagination

Some commands return paginated results with a `next_cursor` field. Pass this as `--cursor` to fetch the next page of results. Loop until `next_cursor` is empty. For example:

```bash
tilebox job list --last 7d --limit 100 --json
tilebox job list --last 7d --limit 100 --cursor <next_cursor> --json
```

Keep the same filters and sort order across pages. Only change `--cursor`.

## Installing The CLI

The public installer downloads a released binary, verifies checksums, and installs to `$HOME/.local/bin` by default:

```bash
curl -fsSL https://cli.tilebox.com/install.sh | sh
```

Customize the install directory:

```bash
curl -fsSL https://cli.tilebox.com/install.sh | TILEBOX_INSTALL_DIR="$HOME/bin" sh
```

Install a specific version:

```bash
curl -fsSL https://cli.tilebox.com/install.sh | TILEBOX_VERSION=0.3.1 sh
```

Ensure the install directory is on `PATH`, then verify:

```bash
tilebox --version
tilebox --help
```

## Updating The CLI

Use the built-in upgrade command for released binaries installed on `PATH`:

```bash
tilebox upgrade --json
```

Install a specific release:

```bash
tilebox upgrade --version 0.3.1 --json
```

Force reinstall:

```bash
tilebox upgrade --force --json
```

Notes:

- `tilebox upgrade` requires `sh` and `curl`.
- It is not supported for dev builds or Windows.
- If the binary was installed in a custom directory, set `TILEBOX_INSTALL_DIR` when needed.

## Useful Command Families

The current CLI exposes these top-level command families. Run `tilebox agent-context` after CLI changes to refresh the list.

| Family | Purpose | Useful Commands |
| --- | --- | --- |
| `automation` | Inspect workflow automations and storage locations. | `tilebox automation list`, `tilebox automation get <automation-id>`, `tilebox automation storage-locations` |
| `cluster` | Manage workflow compute clusters. | `tilebox cluster list`, `tilebox cluster get <cluster-slug>`, `tilebox cluster create <name>`, `tilebox cluster delete <cluster-slug>` |
| `dataset` | Create, update, inspect, query, find datapoints, and generate types for datasets. | `tilebox dataset list`, `tilebox dataset get <dataset-slug>`, `tilebox dataset create`, `tilebox dataset update <dataset-slug>`, `tilebox dataset query <dataset-slug>`, `tilebox dataset find <dataset-slug> <datapoint-id>`, `tilebox dataset generate --slug <dataset-slug>` |
| `dataset collection` | Manage collections within a dataset. | `tilebox dataset collection list --dataset <dataset-slug>`, `tilebox dataset collection get <name> --dataset <dataset-slug>`, `tilebox dataset collection create <name> --dataset <dataset-slug>`, `tilebox dataset collection delete <name> --dataset <dataset-slug>` |
| `job` | Submit, monitor, debug, retry, wait for, and cancel workflow jobs. | `tilebox job submit`, `tilebox job list`, `tilebox job get <job-id>`, `tilebox job wait <job-id>`, `tilebox job retry <job-id>`, `tilebox job cancel <job-id>`, `tilebox job logs <job-id>`, `tilebox job spans <job-id>` |
| `docs` | Search Tilebox documentation from the CLI. | `tilebox docs search "<query>"` |
| `parallel` | Run a shell command multiple times in parallel. | `tilebox parallel -n <count> -- <command> [args...]` |
| `upgrade` | Upgrade or reinstall the Tilebox CLI. | `tilebox upgrade`, `tilebox upgrade --version <version>`, `tilebox upgrade --force` |
| `agent-context` | Describe command metadata and output schemas for agents. | `tilebox agent-context`, `tilebox agent-context job list --output-schema` |

## Safety And Verification

- For destructive actions, such as `cluster delete`, confirm intent unless the user explicitly asked for the action.
- When a command fails, read the error text first. Validation errors usually name the exact flag or argument to fix. Otherwise refer to the `agent-context` for the command.
