---
name: managing-tilebox-datasets
description: "Manage Tilebox datasets with the tilebox CLI. Use when creating datasets, designing or updating schemas, documenting datasets, managing collections, querying and filtering datapoints, or generating dataset types."
license: MIT
compatibility: Requires the tilebox CLI, and a Tilebox API Key ($TILEBOX_API_KEY) or `--api-key`.
metadata:
  author: tilebox
---

# Managing Tilebox Datasets

Use this skill for operational and design work with Tilebox datasets: schema design, dataset creation/update, markdown documentation, collection management, datapoint queries with filtering, datapoint lookup, and generated types. Prefer the CLI for inspection and operations; consult docs and SDKs for ingestion.

## Refresh CLI Metadata

Check exact installed flags and schemas before relying on memory:

```bash
tilebox agent-context dataset --output-schema
```

Relevant docs concepts:

- Datasets are strongly typed containers; every datapoint in a dataset follows the dataset schema.
- Dataset kinds add required fields automatically. Do not include required fields in the custom schema.
- Custom field descriptions and example values power automatic schema documentation.
- Existing fields cannot be removed or changed after data has been ingested. New fields can be added because fields are optional.
- Empty datasets are the exception: if all collections are empty, the schema can be freely edited.

## Inspect Existing Datasets

Listing and inspecting existing datasets:

```bash
tilebox dataset list --json
tilebox dataset get <dataset-slug> --json
```

Use `dataset get` before schema changes to understand current fields, field descriptions, collection counts, time ranges, and whether any collection contains data.

## Schema Design

Choose the dataset kind:

- `temporal` (`telemetry`): required fields are `time`, `id`, and `ingestion_time`.
- `spatiotemporal` (`catalog`): required fields are `time`, `id`, `ingestion_time` and `geometry`.

Custom schema rules:

- Field names must be `snake_case` and valid code identifiers.
- Supported field types are `string`, `bytes`, `bool`, `int64`, `uint64`, `float64`, `Duration`, `Timestamp`, `UUID`, and `Geometry`.
- Set `"repeated": true` for array fields.
- Include `description` and `example_value` for every field whenever possible; this improves generated dataset documentation.
- Treat reordering, renaming, removing, or changing field types as breaking unless the dataset is empty.

Example `schema.json`:

```json
{
  "kind": "spatiotemporal",
  "fields": [
    {
      "name": "scene_id",
      "type": "string",
      "description": "Provider scene identifier.",
      "example_value": "S2A_MSIL2A_20260521T104031_N0511_R008_T32TQM_20260521T132145"
    },
    {
      "name": "cloud_cover",
      "type": "float64",
      "description": "Cloud cover percentage for the scene.",
      "example_value": "12.5"
    },
    {
      "name": "asset_urls",
      "type": "string",
      "repeated": true,
      "description": "URLs for assets associated with the scene.",
      "example_value": "[\"s3://bucket/path/B04.tif\"]"
    }
  ]
}
```

## Create A Dataset

Use files for non-trivial schemas and markdown documentation:

```bash
tilebox dataset create \
  --name "Processed Scenes" \
  --code-name processed_scenes \
  --summary "Processed Sentinel scenes" \
  --schema-file schema.json \
  --description-file README.md \
  --json
```

Inline schema is useful for small tests:

```bash
tilebox dataset create \
  --name "Scenes" \
  --code-name scenes \
  --summary "Processed scenes" \
  --schema '{"kind":"temporal","fields":[{"name":"scene_id","type":"string","description":"Scene identifier","example_value":"S2A_001"}]}' \
  --json
```

Input rules:

- `--schema` and `--schema-file` are mutually exclusive; one is required.
- `--description` and `--description-file` are mutually exclusive.
- `--schema-file -` reads schema JSON from stdin.
- `--description-file -` reads markdown documentation from stdin.
- Do not read both schema and description from stdin in one command.

## Add Markdown Documentation

The dataset `description` is larger markdown documentation, not just a short summary. Use it for context that belongs next to the schema:

- Dataset purpose and ownership.
- Source systems and ingestion cadence.
- Collection naming conventions.
- Field semantics, units, enum-like values, and nullability expectations.
- Query examples and known caveats.

Update documentation from a file:

```bash
tilebox dataset update <dataset-slug> --description-file README.md --json
```

Update summary separately when only the short overview changes:

```bash
tilebox dataset update <dataset-slug> --summary "New short summary" --json
```

## Update A Schema Safely

Schema updates replace the full custom schema. Always start from the current schema source file or reconstruct it from `tilebox dataset get` before editing.

Safe on non-empty datasets:

- Add new custom fields.
- Update metadata such as name, summary, and markdown description.

Only safe when all collections are empty:

- Remove custom fields.
- Rename fields.
- Change field types or repeated-ness.
- Change dataset code name.

Inspect collection counts before breaking changes:

```bash
tilebox dataset collection list --dataset <dataset-slug> --json | jq -r '.[] | [.name, .count] | @tsv'
```

Apply a schema update:

```bash
tilebox dataset update <dataset-slug> --schema-file schema.json --json
```

Combine schema and docs updates when they describe the same change:

```bash
tilebox dataset update <dataset-slug> \
  --schema-file schema.json \
  --description-file README.md \
  --summary "Updated dataset summary" \
  --json
```

## Manage Collections

Collections partition datapoints within a dataset. They are commonly used for products, sources, processing levels, tenants, or logical streams.

```bash
tilebox dataset collection list --dataset <dataset-slug> --json
tilebox dataset collection get <collection-name> --dataset <dataset-slug> --json
tilebox dataset collection create <collection-name> --dataset <dataset-slug> --if-not-exists --json
tilebox dataset collection delete <collection-name> --dataset <dataset-slug> --if-missing-ok --json
```

Use idempotent flags in automation:

- `--if-not-exists` for create.
- `--if-missing-ok` for delete.

Before deleting a collection, confirm intent unless the user explicitly requested deletion. Deleting a collection removes that logical collection from the dataset. A collection must be empty before it can be deleted.

## Query Datapoints With The CLI

`tilebox dataset query` always emits JSON. Use it for quick inspection and scripts.

```bash
# Query all collections in the last 7 days
tilebox dataset query <dataset-slug> --last 7d --limit 100

# Query specific collections over a time range
tilebox dataset query <dataset-slug> \
  --collections raw,processed \
  --after 2026-05-01 \
  --before 2026-06-01 \
  --limit 100

# Query datapoints intersecting a WKT polygon
tilebox dataset query <dataset-slug> \
  --last 7d \
  --spatial-extent 'POLYGON((-109.05 41,-109.05 37,-102.05 37,-102.05 41,-109.05 41))' \
  --limit 100

# Query datapoints intersecting a GeoJSON polygon or multipolygon file
tilebox dataset query <dataset-slug> \
  --after 2026-05-01 \
  --before 2026-06-01 \
  --spatial-extent-file colorado.geojson \
  --limit 100

# Continue pagination
tilebox dataset query <dataset-slug> --last 7d --limit 100 --cursor <next_cursor>
```

Extract fields with `jq`:

```bash
tilebox dataset query <dataset-slug> --last 7d --limit 10 | jq '.datapoints'
tilebox dataset query <dataset-slug> --last 7d --limit 10 | jq -r '.next_cursor'
tilebox dataset query <dataset-slug> --last 7d --limit 10 | jq -r '.datapoints[] | [.id, .time] | @tsv'
```

Temporal filters:

- Use `--last <duration>` for relative windows such as `7d`, `12h`, or `1Y3M`.
- Use `--after` and `--before` for explicit RFC3339 timestamps or `YYYY-MM-DD` dates.
- Do not combine `--last` with `--after` or `--before`.

Spatial filters:

- Use `--spatial-extent` for inline WKT or GeoJSON.
- Use `--spatial-extent-file` for a WKT or GeoJSON file.
- The query geometry must be a `Polygon` or `MultiPolygon`; GeoJSON `Feature` wrappers are accepted when their geometry is a polygon or multipolygon.
- Do not combine `--spatial-extent` with `--spatial-extent-file`.
- Coordinates are longitude/latitude for geographic datasets; keep polygon rings closed.
- Spatial filters can be combined with `--collections`, `--last`, `--after`, `--before`, `--limit`, and `--cursor`.

Example inline GeoJSON query:

```bash
tilebox dataset query <dataset-slug> \
  --collections S2A_S2MSI2A \
  --last 14d \
  --spatial-extent '{"type":"Polygon","coordinates":[[[-109.05,41],[-109.05,37],[-102.05,37],[-102.05,41],[-109.05,41]]]}' \
  --limit 50
```

Example WKT file query:

```bash
cat > area.wkt <<'EOF'
MULTIPOLYGON(((-109.05 41,-109.05 37,-102.05 37,-102.05 41,-109.05 41)))
EOF

tilebox dataset query <dataset-slug> \
  --after 2026-05-01T00:00:00Z \
  --before 2026-06-01T00:00:00Z \
  --spatial-extent-file area.wkt \
  --limit 100
```

For notebook-friendly xarray results or ingestion workflows, use the Python SDK query APIs and consult docs. The Python SDK supports collection-level or dataset-level queries, temporal extents, spatiotemporal geometry filters, automatic pagination, progress bars, and `skip_data=True` for fast existence/count probes.

## Find A Datapoint By ID

Use `find` when you know the datapoint UUID and want the decoded datapoint from any collection in a dataset:

```bash
tilebox dataset find <dataset-slug> <datapoint-id> | jq '.'
```

## Generate Go Types

Generate Go protobuf types when Go code should query strongly typed datapoints:

```bash
tilebox dataset generate --slug <dataset-slug> --out ./protogen --package tilebox.v1 --json
```

Check generated files into version control when they are used by application code.
