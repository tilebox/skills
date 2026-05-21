---
name: using-tilebox-cli
description: "Uses the Tilebox CLI for datasets, workflows, docs search, and automation. Use when a task involves tilebox commands, authentication, JSON output, agent-context schemas, pagination, installation, or upgrades."
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

## Pagination

Some commands return paginated results with a `next_cursor` field, including:

- `tilebox job list --json`
- `tilebox job logs <job-id> --json`
- `tilebox job spans <job-id> --json`

Loop until `next_cursor` is empty:

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

- `tilebox dataset list|get|generate` — dataset discovery and Go type generation.
- `tilebox cluster list|get|create|delete` — workflow cluster management.
- `tilebox job submit|list|get|wait|logs|spans|retry|cancel` — workflow job operations.
- `tilebox docs search` — search Tilebox documentation from the CLI.
- `tilebox parallel` — run a command multiple times in parallel.

## Safety And Verification

- For destructive actions, such as `cluster delete`, confirm intent unless the user explicitly asked for the action.
- When a command fails, read the error text first. Validation errors usually name the exact flag or argument to fix. Otherwise refer to the `agent-context` for the command.
