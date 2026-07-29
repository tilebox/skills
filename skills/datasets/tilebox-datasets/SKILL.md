---
name: tilebox-datasets
description: "Discovers, selects, inspects, queries, and manages Tilebox datasets and external auxiliary grids. Use when choosing Earth observation or supporting DEM, weather, climate, QA, or land-mask data from a requested target product—even if no dataset is named—or when evaluating coverage, resolution, collections, schemas, datapoints, storage access, credentials, dataset creation, or schema updates."
license: MIT
compatibility: Requires the tilebox CLI, and a Tilebox API Key ($TILEBOX_API_KEY) or `--api-key`.
metadata:
  author: tilebox
---

# Managing Tilebox Datasets

Use this skill to choose Tilebox datasets from target-product requirements and for operational and design work with datasets: inspection, querying, schema design, creation/update, documentation, collection management, datapoint lookup, and generated types. Prefer the CLI for live catalog inspection and operations.

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

## Select A Dataset From The Target Product

When the user describes an Earth observation result but does not name a dataset, own the selection instead of asking the user to choose from unfamiliar slugs:

1. Translate the target product into modality, product level, bands or polarizations, spatial resolution, time range, revisit, latency, cloud tolerance, QA, and validation requirements.
2. Read `reference/earth-observation-product-selection.md` to identify a default and meaningful alternatives.
3. Read `reference/open-data-dataset-catalog.md` for exact Tilebox slugs, payload providers, formats, authentication, and limitations.
4. Verify candidates against the live catalog with `tilebox dataset list --json` and `tilebox dataset get <slug> --json`.
5. Query a small metadata sample over the requested AOI and time range. Confirm collections, fields, coverage, asset locations, and access metadata rather than trusting remembered schemas.
6. Classify source access as public unsigned, provider-authenticated, requester-pays, or restricted. Read the linked provider guide before coding data access.
7. Select one default, explain why it fits, and mention alternatives only when their tradeoffs are useful. Do not silently combine sensors or providers.

For ordinary optical multispectral surface-reflectance work, default to `open_data.aws_earth.sentinel2` when the user did not specify a source and Sentinel-2 resolution, coverage, revisit, and cloud limitations are suitable. Its Element 84 AWS Earth Search L2A payloads are public COGs and require no source-provider credentials. Landsat 8/9 can be suggested as an optional addition for a longer historical record, but do not create a joint Sentinel-2/Landsat pipeline by default because harmonizing grids, resolution, spectral response, radiometry, and styling adds meaningful complexity.

Do not use that optical default when the outcome needs cloud-penetrating SAR, deformation, active-fire or temperature measurements, finer object-level resolution, a longer pre-Sentinel-2 archive, or an explicitly requested provider/product layout.

## Select Auxiliary Data Outside Tilebox

Workflows often need supporting data that Tilebox does not index, such as a DEM, weather or climate reanalysis, permanent-water or land masks, population, or another global grid. Treat these as auxiliary inputs rather than inventing a Tilebox dataset slug. Read `reference/auxiliary-data-sources.md`, research the current catalogues, and select by variable semantics, spatial/temporal resolution, coverage, update cadence, vertical/reference conventions, chunking, license, and access requirements.

Strongly prefer an analysis-ready Zarr or Icechunk source for large global or multidimensional auxiliary grids when its chunking fits the workflow. Subset lazily to the task's AOI, time, variables, and levels instead of downloading the full store. Credentials-free access is still preferable when scientifically equivalent, but do not sacrifice product correctness for convenience.

Distinguish external global auxiliaries from product-coupled layers. A Sentinel-2 Scene Classification Layer, Landsat QA band, SAR incidence-angle layer, or similar per-acquisition data normally belongs to the selected source product and should be read with that product rather than looked up in a separate global catalogue.

Before authoring, tell the user when an auxiliary provider account or subscription is required and guide setup using direct links in the auxiliary reference. Configure keys through the local or runner secret environment, validate one bounded read, and remember that local credentials do not automatically reach remote runners.

### Dataset Selection And Provider References

| Reference | Use when |
| --- | --- |
| `reference/earth-observation-product-selection.md` | Map a target product to observation requirements, a default dataset, alternatives, and counterexamples. |
| `reference/open-data-dataset-catalog.md` | Look up verified Tilebox open-data slugs and compare providers, formats, authentication, costs, and limitations. |
| `reference/auxiliary-data-sources.md` | Select DEM, weather/climate, masks, or other supporting grids outside Tilebox, favoring suitable Zarr/Icechunk sources and guiding provider access. |
| `reference/providers/aws-earth-search.md` | Use the default public Sentinel-2 L2A COG source from Element 84 AWS Earth Search. |
| `reference/providers/copernicus-data-space.md` | Access `open_data.copernicus.*` product bytes with a Copernicus account and S3 credentials. |
| `reference/providers/usgs-landsat.md` | Access `open_data.usgs.*` Landsat bytes in the AWS requester-pays bucket. |
| `reference/providers/alaska-satellite-facility.md` | Access `open_data.asf.*` SAR product bytes with NASA Earthdata Login credentials. |

## Separate Metadata From Product Bytes

Tilebox open-data datasets index structured metadata and asset locations; Tilebox usually does not host the source imagery bytes. A Tilebox API key does not automatically grant access to Copernicus, USGS/AWS, ASF, commercial providers, private source buckets, or an output bucket.

Keep three access layers explicit:

1. **Tilebox API:** metadata queries and workflow/job operations via the Tilebox API key.
2. **Source storage:** provider-hosted imagery bytes, which may be public or require separate credentials.
3. **Output storage:** local disk for suitable notebook/local work, or user-controlled shared storage for remote or distributed work. Do not assume Tilebox-hosted output storage exists.

For every selected source, tell the user what Tilebox provides, where the bytes live, their format, whether source credentials or requester-pays charges apply, and what access was actually validated. Never ask the user to paste secrets into chat or place them in task inputs, `job_cache`, logs, source control, or release artifacts.

## Inspect Existing Datasets

Listing and inspecting existing datasets:

```bash
tilebox dataset list --json
tilebox dataset get <dataset-slug> --json
```

Use `dataset get` before selection or schema changes to understand current fields, field descriptions, collection counts, time ranges, and whether any collection contains data.

## Schema Design

Choose the dataset kind:

- `temporal` (`telemetry`): required fields are `time`, `id`, and `ingestion_time`.
- `spatiotemporal` (`catalog`): required fields are `time`, `id`, `ingestion_time` and `geometry`.

Custom schema rules:

- Field names must be `snake_case` and valid code identifiers.
- Supported field types are `string`, `bytes`, `bool`, `int64`, `uint64`, `float64`, `Duration`, `Timestamp`, `UUID`, and `Geometry`.
- Set `"queryable": true` on custom fields that should support server-side filtering.
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
      "queryable": true,
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

`tilebox dataset query` always emits JSON. Agents should still pass `--json` consistently. Use it for quick inspection and scripts.

```bash
# Query all collections in the last 7 days
tilebox dataset query <dataset-slug> --last 7d --limit 100 --json

# Query specific collections over a time range
tilebox dataset query <dataset-slug> \
  --collections raw,processed \
  --after 2026-05-01 \
  --before 2026-06-01 \
  --limit 100 \
  --json

# Query datapoints intersecting a WKT polygon
tilebox dataset query <dataset-slug> \
  --last 7d \
  --spatial-extent 'POLYGON((-109.05 41,-109.05 37,-102.05 37,-102.05 41,-109.05 41))' \
  --limit 100 \
  --json

# Query datapoints intersecting a GeoJSON polygon or multipolygon file
tilebox dataset query <dataset-slug> \
  --after 2026-05-01 \
  --before 2026-06-01 \
  --spatial-extent-file colorado.geojson \
  --limit 100 \
  --json

# Continue pagination
tilebox dataset query <dataset-slug> --last 7d --limit 100 --cursor <next_cursor> --json
```

### Filter Queryable Fields

`--filter` accepts a CQL2 Text expression over queryable dataset fields. Before constructing one, discover the fields the dataset exposes:

```bash
tilebox dataset get <dataset-slug> --json \
  | jq '[.fields[] | select(.queryable == true) | {name, type, description, exampleValue}]'
```

Map the user's intent to an exact queryable field and choose operators from its type. Do not invent field names or assume every schema field is queryable.

- Strings support exact equality and `IS NULL` / `IS NOT NULL`; prefix, substring, pattern, and ordering comparisons are unsupported.
- Boolean fields support equality/inequality with `TRUE` or `FALSE` and null checks.
- Numeric fields support `=`, `<>`, `<`, `<=`, `>`, and `>=`.
- All comparisons, `NOT`, `AND`, and `OR` use SQL/CQL2 three-valued logic: comparisons against null or missing values are unknown, `NOT unknown` remains unknown, and only true results match.
- Include null values explicitly when the user's intent requires them, for example `cloud_cover < 5 OR cloud_cover IS NULL`.
- Parentheses, nested `AND` / `OR` expressions, and `NOT` are supported.
- Repeating `--filter` combines the filters with explicit `AND`.

Examples:

```bash
# Exact string and numeric comparison
tilebox dataset query open_data.aws_earth.sentinel2 --last 5d \
  --filter "cloud_cover < 5 AND platform = 'sentinel-2c'" \
  --json

# Nested logic that deliberately includes missing cloud cover
tilebox dataset query open_data.aws_earth.sentinel2 --last 5d \
  --filter "(cloud_cover < 5 OR cloud_cover IS NULL) AND platform = 'sentinel-2c'" \
  --json

# Equivalent explicit AND across repeated flags
tilebox dataset query open_data.aws_earth.sentinel2 --last 5d \
  --filter "cloud_cover < 5" \
  --filter "platform = 'sentinel-2c'" \
  --json
```

If the user asks for unsupported string matching such as a prefix or substring, explain the limitation and use an exact match only if it preserves their intent. When null handling is ambiguous and materially changes results, clarify whether missing values should be included.

Extract fields with `jq`:

```bash
tilebox dataset query <dataset-slug> --last 7d --limit 10 --json | jq '.datapoints'
tilebox dataset query <dataset-slug> --last 7d --limit 10 --json | jq -r '.next_cursor'
tilebox dataset query <dataset-slug> --last 7d --limit 10 --json | jq -r '.datapoints[] | [.id, .time] | @tsv'
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
tilebox dataset query open_data.aws_earth.sentinel2 \
  --collections L2A \
  --last 14d \
  --spatial-extent '{"type":"Polygon","coordinates":[[[-109.05,41],[-109.05,37],[-102.05,37],[-102.05,41],[-109.05,41]]]}' \
  --limit 50 \
  --json
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
  --limit 100 \
  --json
```

### Filter With The Python SDK

The Python SDK accepts one typed `filter=` expression on dataset-level and collection-level queries. Build it with `field()`; combine expressions with `&`, `|`, and `~`, not Python `and`, `or`, or `not`. Parenthesize comparisons when combining them.

```python
from tilebox.datasets import field

data = dataset.query(
    temporal_extent=(start, end),
    filter=(field("cloud_cover") < 5) & (field("platform") == "sentinel-2c"),
)

data_including_missing = dataset.query(
    temporal_extent=(start, end),
    filter=((field("cloud_cover") < 5) | field("cloud_cover").is_null())
    & (field("platform") == "sentinel-2c"),
)
```

Use `field("name").is_null()` and `.is_not_null()` for null checks. A comparison against a null or missing field evaluates to unknown and is excluded, including `!=` and negated comparisons; explicitly OR with `.is_null()` to include it. The Python API builds a typed expression rather than parsing CQL2 text, and exposes a single `filter=` argument, so compose multiple conditions into one expression.

Python SDK queries return notebook-friendly xarray results and also support temporal extents, spatiotemporal geometry filters, automatic pagination, progress bars, and `skip_data=True` for fast existence/count probes.

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
