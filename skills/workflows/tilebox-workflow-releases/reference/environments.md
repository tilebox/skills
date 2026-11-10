# Managing Workflow Environments

Store workflow configuration and credentials as Tilebox environment variables, assign each entry to all clusters or selected clusters, and read them through Python's `os.environ`. Environment-capable `tilebox runner start` processes inject the resolved values automatically and replace worker runtimes when effective values change. No workflow-side secret fetcher, polling loop, release publication, or manual runner restart is needed for a value-only change.

Read this reference when configuring provider tokens/API keys, assigning variables to clusters, resolving inheritance/overrides, rotating credentials, or debugging missing/stale runtime values. Use `tilebox-workflow-authoring` for task code and `tilebox-workflow-jobs` for execution checks. These are Tilebox variables, not the agent host's own secrets or environment settings.

## Contents

- [Check capabilities and destination](#check-capabilities-and-destination)
- [Inheritance and shared entries](#inheritance-and-shared-entries)
- [Inspect inventory versus effective values](#inspect-inventory-versus-effective-values)
- [Create, rotate, and retarget](#create-rotate-and-retarget)
- [Secret entry and literal values](#secret-entry-and-literal-values)
- [Read values in workflow code](#read-values-in-workflow-code)
- [Automatic reload and verification](#automatic-reload-and-verification)

## Check Capabilities And Destination

Use `tilebox-cli` to verify authentication, active organization/API, and current CLI capabilities:

```bash
tilebox --version
tilebox agent-context variable --output-schema
tilebox agent-context runner start
tilebox account get --json
tilebox cluster list --json
```

The environment integration targets CLI 0.15; CLI 0.14 lacks `variable`. Check released availability and installed capabilities rather than assuming a version is available. Upgrade managed runner machines too: updating the submitting CLI does not upgrade an already-running runner. Use a development build only when the user selected that channel. If the backend lacks variable management or `GetClusterRunnerEnvironment`, report the compatibility blocker rather than silently falling back to unmanaged execution.

Before a mutation, inspect existing entries and their full scope. An all-clusters change can affect unrelated workflows and future clusters; use the requested cluster scope rather than silently choosing a global credential. Apply remote-write approval rules to creates, rotations, scope changes, and deletions.

## Inheritance And Shared Entries

An entry has a stable UUID, uppercase environment name, literal value, secret classification, optional description, and cluster scope. There are no named environment sets or separate secret bindings. Multiple entries can share a name under these rules:

- **All clusters** supplies a default across all current and future clusters in the active organization/namespace.
- **Selected clusters** overrides that default for its explicit cluster selection. Same-name selected entries must have disjoint selections; only one all-clusters default is allowed.
- Removing a selected override exposes the all-clusters default, if present. Without either, Tilebox supplies no managed value for that name.
- One entry assigned to several clusters is one shared value. Editing it affects every cluster receiving that entry, even when opened from one cluster's console page.
- Selecting every existing cluster is a fixed selection, not all-clusters scope. A retained entry with no assigned clusters applies nowhere; it never becomes global.

For example, `LOG_FORMAT=json` on all clusters and `LOG_FORMAT=text` on `dev` gives `dev` text and other clusters json. Deleting the dev entry restores json on dev. Changing the default affects clusters without an override; it does not replace dev's override.

These variables are cluster-scoped, not job-, workflow-, or release-scoped. Workflows executing on the same cluster receive the same managed environment. Use separate clusters when workflows need different values, but do not treat cluster assignment as an authorization boundary: runner credentials may permit resolving other clusters in the organization. Workflow execution scope does not isolate credentials.

## Inspect Inventory Versus Effective Values

List inventory to find stored entries; list with a cluster filter to see which entry wins for each name. These examples omit values and previews from agent output:

```bash
tilebox variable list --json \
  | jq '{variables: [.variables[] | {id, name, is_secret, scope}], next_cursor}'
tilebox variable list --cluster dev --json \
  | jq '{variables: [.variables[] | {id, name, is_secret, scope}], next_cursor}'
tilebox variable list --cluster '' --json \
  | jq '{variables: [.variables[] | {id, name, is_secret, scope}], next_cursor}'
```

`--cluster ''` selects only the default cluster. Omitting `--cluster` on **list** means inventory, whereas omitting it on **create** means all clusters. Page through `next_cursor` with the same filters; effective overrides are resolved before pagination. Returned entries retain their supplying UUID and full shared scope.

`get`, `update`, and `delete` accept a name or UUID. Without selectors, a name must identify exactly one inventory entry. With `--cluster`, they select an **explicit** entry containing every selected cluster, not an inherited default. Use `--all-clusters` to select the default explicitly, or use its UUID. Prefer the inspected UUID when the name is ambiguous.

## Create, Rotate, And Retarget

The following examples assume the named clusters exist and each operation is authorized. Explicit scope makes the intended impact visible:

```bash
# Non-secret default and a dev-only override.
tilebox variable create LOG_FORMAT json --all-clusters --json
tilebox variable create LOG_FORMAT text --cluster dev --json

# One secret shared by two clusters, from an existing protected file.
tilebox variable create PROVIDER_API_KEY --secret \
  --cluster dev --cluster batch --value-file /secure/provider-key.txt --json

# Rotate the inspected entry without revealing its old value.
tilebox variable update <variable-id> --value-file /secure/replacement-key.txt --json

# Replace the complete selection, retaining dev and adding analytics.
tilebox variable scope <variable-id> --cluster dev --cluster analytics --json

# Remove the whole inspected entry, not just one cluster assignment.
tilebox variable delete <variable-id> --json
```

- `create` never overwrites. For a conflict, inspect entries and update the intended UUID or choose a disjoint scope; do not delete a shared entry just to make creation succeed.
- `update --cluster dev` selects an entry; it neither changes its scope nor limits the write to dev. The same applies to `delete --cluster dev`: it deletes the whole shared entry.
- To override an inherited default only on dev, create a same-name entry for dev. If dev already belongs to a shared selected entry, it cannot also belong to another same-name selected entry; explicitly plan the scope split and its temporary fallback before changing it.
- `scope` requires a UUID and replaces the entire cluster selection. Repeat `--cluster` for every cluster to keep. To remove just one assignment, retain the others in `scope`; use `delete` when removing the entry entirely. `--all-clusters` and `--cluster` cannot be combined.
- Omitted update fields remain unchanged, including value and secret classification. Metadata-only changes need no value or reveal. `--description ''` clears a description; `--secret=false` deliberately makes the value readable without reveal.

## Secret Entry And Literal Values

Use `--secret` for credentials: CLI creation defaults to non-secret. Prefer an existing protected file or a secret-manager stream piped to `--value-file -`, with shell tracing disabled. Do not interpolate credentials into command arguments or ask the user to paste them into chat. A missing create value fails without prompting; `--value-file -` reads stdin until EOF, not an interactive hidden prompt.

For user entry, direct them to **Workflows → Environment → Add variable** in the intended team's console (`/workflows/environment`). Enter Name and Value, keep Secret enabled, and choose the intended clusters. To rotate, edit the entry and choose **Replace value**. Secret defaults on in the console but the entry textarea is visible while typing; avoid screenshots containing it. Cluster details show the effective environment; editing there still changes the original shared entry. Console bulk import accepts dotenv; the CLI has no `variable import` command.

Names match `^[A-Z_][A-Z0-9_]*$`. Values must be nonempty UTF-8 without NUL bytes. File/stdin input is one literal value, not a dotenv document; whitespace, trailing newlines, and multiline content are preserved. Tilebox does not expand `$OTHER_VARIABLE` or execute shell expressions. Avoid an accidental newline when preparing a token file.

All stored values are encrypted at rest. Secret classification controls disclosure in management reads, not access by workflow code: workers receive full values. Normal secret reads omit `value` and may include `secret_preview`; previews are display-only, never usable credentials. Mutation responses omit both. Avoid `--reveal` for configuration checks; console Copy also retrieves a full secret. Keep values and previews out of descriptions, task inputs, caches, release files, logs, and reports. Masking in the console does not redact secrets that workflow code prints.

## Read Values In Workflow Code

Managed values reach Python before workflow imports, so tasks and provider SDKs can use ordinary environment access:

```python
import os

from tilebox.workflows import ExecutionContext, Task


class CheckProviderConfiguration(Task):
    def execute(self, context: ExecutionContext) -> None:
        api_key = os.environ["PROVIDER_API_KEY"]
        # Pass api_key to the provider client here; never log its value.
        context.logger.info("Provider credential configured", configured=bool(api_key))
```

Document required names and read credentials when constructing the client that needs them, preferably inside task execution or lazy initialization. This keeps local import/task discovery independent of deployment-only credentials. Do not fetch secrets through the management API inside tasks or add an environment watcher. Libraries with their own environment conventions can read injected names directly; otherwise pass `os.environ[...]` explicitly.

Effective process precedence is inherited runner-host environment → resolved managed values → runner-owned bootstrap/session settings. Managed values override local values with the same name. Removing the last managed entry exposes any inherited host value; it does not guarantee the name disappears. Runner API credentials, API URL, runtime/session transport, and runner log settings remain runner-owned; do not try to rotate `TILEBOX_API_KEY` or override reserved names through `variable`.

Automatic injection applies to managed `tilebox runner start`, including one started locally. Standalone Python runners and notebooks need their environment configured separately. Local build/publish validation does not fetch cluster variables, and managed variables are not supplied to dependency installation (`uv sync`); arrange build/install authentication separately without bundling secrets into the release.

## Automatic Reload And Verification

The managed runner polls cluster deployments and resolved variables roughly every 10–12 seconds. A successful poll replaces the complete managed snapshot, including removals; an empty snapshot clears all managed overrides. Changes to effective names or values trigger runtime replacement. Metadata-only changes or an unchanged resolved name/value set do not.

The runner validates releases with the new environment before making them runnable. Idle superseded runtimes stop; already-reserved/running tasks finish on their original runtimes and values. Newly assigned work uses the validated replacement configuration. Old and new runtimes may coexist until old work drains. The runner stays up; existing Python processes are not modified in place. Environment-only changes reuse extracted artifacts and the prepared virtualenv rather than reinstalling dependencies.

Propagation is eventual, not a deadline or a per-job snapshot. A job spanning a rotation can have tasks using different credential versions. Keep the old provider credential valid until old tasks drain when uninterrupted rotation is required; a configuration save alone does not prove every runner has switched.

- Initial environment-fetch failure prevents runner initialization. A later poll failure retains the last published snapshot and retries; it does not clear values.
- If a release fails preparation/import/task-discovery validation under the new environment, that release is excluded from new work and retried on later polls; active tasks still drain. Validation does not execute task bodies, so it cannot prove provider credentials work.
- To debug missing/stale values, compare the effective entry's UUID and full scope for the actual execution cluster, then check runner version, poll errors, and release-validation errors. Check host fallback after deletion and whether the observed task started before refresh.
- Verify delivery with a workflow-scoped probe using `tilebox-workflow-jobs`: check an allowlisted name's presence or make one bounded authenticated provider request. Report success/failure, not the value, preview, or credential hash. Never dump `os.environ`.
- To verify reload, use an authorized non-sensitive test marker: start work before the change, change the marker, wait for successful refresh, and run new work. Confirm old work completes with its old marker while new work sees the new marker. A single post-change job proves delivery, not graceful replacement.

A credential-only fix needs no new release. After confirming the new runtime can access the provider, retry eligible failed jobs rather than resubmitting completed work. Environment refresh does not itself retry failed tasks.
