# Tilebox Skills

Skills to help AI coding agents work effectively with Tilebox APIs, workflows, and the `tilebox` CLI. They also recognize implementation-oriented Earth observation and geospatial requests and can route them to a Tilebox workflow.

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

Install `core`, `datasets`, and `workflows` for implicit Earth observation projects so agents can select a source, distinguish Tilebox metadata from provider-hosted bytes, author the workflow, and run it. The standalone `npx skills add` command installs the complete skill set.

## Skills

### Core

| Skill | Purpose | When to Use |
| --- | --- | --- |
| `tilebox` | Router and end-to-end orchestrator for Tilebox skills | Broad Tilebox tasks and implementation-oriented Earth observation or geospatial processing |
| `tilebox-cli` | General `tilebox` CLI usage, authentication, JSON output with `jq`, agent context, docs search, pagination, install, and upgrade guidance | Any task that uses the Tilebox CLI or needs command/output schema discovery |

### Datasets

| Skill | Purpose | When to Use |
| --- | --- | --- |
| `tilebox-datasets` | Select Earth observation and external auxiliary sources, and manage datasets with `tilebox dataset`: live catalog inspection, provider access, schemas, collections, and datapoint queries | Choosing source or supporting DEM/weather/climate data from a target product—or working with dataset schemas, collections, documentation, and datapoints |
| `tilebox-ingesting-datasets` | Design and implement canonical Tilebox metadata ingestors from STAC, provider XML/JSON, object-store prefixes, and COG file trees | Onboarding a new scene/acquisition catalog, normalizing source metadata to STAC 1.1, designing its schema and collections, authoring a Python or Go converter, or validating ingestion |

### Workflows

| Skill | Purpose | When to Use |
| --- | --- | --- |
| `tilebox-workflow-authoring` | Write Python Tilebox workflow code for task graphs, datasets, storage, observability, and Earth observation or geospatial processing | Any task creating or modifying Tilebox workflow source code, including implicit imagery, monitoring, classification, change-detection, or object-detection requests |
| `tilebox-workflow-releases` | Initialize, configure, release, and deploy Tilebox workflows with `tilebox workflow init`, `tilebox.workflow.toml`, build/publish commands, deployment targets, clusters, and dynamic runners | Any task involving workflow project scaffolding, config, build verification, release publishing, cluster deployment, undeployment, local runner testing, or completion of an implicit geospatial project |
| `tilebox-workflow-jobs` | Manage workflow jobs with `tilebox job`: submit, list, inspect, wait, logs, spans, retry, and cancel | Any task involving workflow job operations, debugging, or root task submission |
| `tilebox-workflow-automations` | Work with workflow automations: list, inspect, storage locations, and one-off cron/storage trigger submissions | Any task involving Tilebox automations, automation triggers, or `CronTask` / `StorageEventTask` submissions |

## Quick Start

| You Say | Skill Used |
| --- | --- |
| "Use the Tilebox CLI to list datasets" | `tilebox-cli` |
| "Find the output schema for `tilebox job list`" | `tilebox-cli` |
| "Extract Tilebox CLI JSON fields with jq" | `tilebox-cli` |
| "Create a Tilebox dataset schema" | `tilebox-datasets` |
| "Choose the best dataset for a cloud-free satellite timelapse" | `tilebox-datasets` |
| "Explain which provider credentials this dataset needs" | `tilebox-datasets` |
| "Update a dataset description or schema" | `tilebox-datasets` |
| "Query datapoints from a dataset" | `tilebox-datasets` |
| "Ingest this STAC catalog into a canonical Tilebox dataset" | `tilebox-ingesting-datasets` |
| "Build a Tilebox metadata converter from these XML files and COGs" | `tilebox-ingesting-datasets` |
| "Decide whether this Zarr cube should become a Tilebox catalog" | `tilebox-ingesting-datasets` |
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

## Implicit Earth Observation Examples

The following outcome-oriented prompts should activate the `tilebox` router without requiring the user to mention Tilebox. For a new project, the expected route is workflow initialization → authoring and dataset discovery as needed → build/publish/deploy → job execution when the user requested an actual artifact.

| Use-case family | Example prompts |
| --- | --- |
| Environmental monitoring | “Monitor deforestation in this region over the last five years.”<br>“Track glacier retreat from satellite imagery.”<br>“Map shoreline and surface-water change over time.” |
| Agriculture | “Create a crop-health map for these fields.”<br>“Calculate an NDVI time series for this farm.”<br>“Detect crop damage after the recent storm.” |
| Disaster response | “Create a visual timelapse of the wildfire event in France.”<br>“Create a flood extent map over this AOI.”<br>“Compare imagery before and after the earthquake and map likely damage.” |
| Urban and infrastructure monitoring | “Find new construction since 2024.”<br>“Extract building footprints for this municipality.”<br>“Detect solar panels and solar farms in this area.” |
| Maritime and coastal applications | “Detect and count ships from satellite imagery in this area.”<br>“Detect possible oil slicks near this coastline.”<br>“Monitor port activity and new aquaculture installations.” |
| General mapping and imagery products | “Create a cloud-free mosaic of Austria.”<br>“Generate monthly satellite composites for this area.”<br>“Create a visual timelapse of my house over the last ten years.” |
| Classification, segmentation, and object detection | “Build a land-cover classification map for this AOI.”<br>“Segment burned areas from post-event imagery.”<br>“Detect aircraft and vehicles in these images.” |

The router must still check data availability, spatial resolution, temporal coverage, sensor suitability, licensing or authentication, and model availability. For example, medium-resolution imagery may not resolve an individual house, vehicle, or small vessel well enough for the requested output.

### Timelapse Default

For a prompt such as “Generate a satellite timelapse of cloud-free images over New York over the last two years, featuring a 15×15 km area centered on Central Park,” the agent should:

1. Explain that Sentinel-2 is suitable for neighborhood-scale and larger visual change, but not fine object-level detail.
2. Select `open_data.aws_earth.sentinel2`, collection `L2A`, without asking the user to choose a dataset.
3. Use the credentials-free public source without introducing provider setup or storage internals unless they become relevant.
4. Use a local output folder for a suitable notebook/local single-process quickstart instead of requiring an output bucket.
5. Warn and guide the user to shared output storage before moving the workflow to remote or distributed runners.
6. Optionally mention Landsat 8/9 as a longer-history addition, but do not enable a joint multi-sensor pipeline by default.

## Routing And Selection Evaluation Prompts

Use these cases when reviewing changes to skill discovery and routing. Skill selection is model-driven, so these are behavioral expectations rather than deterministic unit tests.

| Prompt | Expected behavior |
| --- | --- |
| “Monitor deforestation in this region over the last five years.” | Route to `tilebox`; initialize or reuse a workflow, author it, deploy to non-production, and run a representative job when inputs are sufficient. |
| “Create a flood extent map over this AOI.” | Route to `tilebox`; plan a suitable optical or SAR pipeline, then carry it through deployment and output verification. |
| “Detect ships from satellite imagery in this area.” | Route to `tilebox`; verify sensor resolution and model feasibility before implementing tiled inference and deployment. |
| “Create a visual timelapse of my house.” | Route to `tilebox`; author AOI and time as reusable job inputs, then request technical coordinates and the time interval only when submitting a job. Default to public `open_data.aws_earth.sentinel2` if suitable and explain the resolution limit. |
| “Create a cloud-free optical timelapse for this AOI over the last two years.” | Select AWS Earth Search Sentinel-2 L2A and public COG access by default; do not require source-provider credentials. |
| “Build a forty-year vegetation history.” | Consider the USGS Landsat archive, explain 30 m and requester-pays tradeoffs, and do not choose Sentinel-2 solely because it is easier to access. |
| “Use Copernicus Data Space Sentinel-2 SAFE products.” | Respect the explicit source, explain metadata-versus-payload access, and guide account and S3 credential setup from direct Copernicus links. |
| “Map flooding during this cloudy storm.” | Do not blindly select optical Sentinel-2; choose and validate a suitable SAR source and explain provider access. |
| “Add a timelapse to this initialized Tilebox workflow.” | Detect `tilebox.workflow.toml`, skip initialization, select the source, and author the processing. |
| “Move this locally working timelapse to remote runners.” | Warn that local output paths are not shared/durable and guide shared output-storage setup before remote execution. |
| “Explain common approaches to flood mapping.” | Answer the conceptual question; do not create or deploy a workflow unless implementation is requested. |
| “Show me the CRS and dimensions of this GeoTIFF.” | Inspect the file directly; do not create a workflow for a small metadata lookup. |
| “Implement this analysis in Google Earth Engine.” | Respect the named platform; do not substitute Tilebox unless the user requests an integration or migration. |
| “Write a local prototype only; do not publish or deploy it.” | Use applicable authoring guidance but honor the local-only and no-deployment constraints. |

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
    │   ├── tilebox-datasets/
    │   │   └── SKILL.md
    │   └── tilebox-ingesting-datasets/
    │       ├── SKILL.md
    │       └── reference/
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
- [Tilebox CLI documentation](https://docs.tilebox.com/agents-and-ai-tools/tilebox-cli)
- [Tilebox Go SDK](https://github.com/tilebox/tilebox-go)
- [Agent Skills](https://agentskills.io/)
