# Source Discovery And Conversion Recipe

Complete discovery before creating a Tilebox dataset or converter. The output is a reviewable recipe that turns source variation and policy choices into deterministic implementation rules.

## Start With Suitability

A source fits when its natural records are discrete scenes, acquisitions, observations, or products with stable identity, trustworthy time, meaningful asset boundaries, and authoritative geometry for spatial records.

Do not manufacture records from chunks, time slices, tiles, or variable groups in an analysis-ready Zarr or Icechunk cube. Recommend direct analytical access unless an upstream acquisition inventory or small set of genuine products serves a concrete discovery, access-control, lineage, or orchestration need. Stop when no stable identity, time, record boundary, or useful catalog use case exists.

## Classify The Source

Choose one discovery path:

| Source | Primary evidence | Additional reference |
| --- | --- | --- |
| STAC API, catalog, or Item JSON | STAC schemas, source collections, Items, Queryables, extension versions | `canonical-stac-1.1.md` |
| Provider XML or non-STAC JSON/API | Provider schema/docs, representative records, sidecars, product specification | `non-stac-and-file-tree-sources.md` |
| Object-store prefix or file tree | Stable grouping rules, file formats, raster headers, sidecars, object metadata | `non-stac-and-file-tree-sources.md` |

Do not assume the provider's catalog collections should become Tilebox collections. Do not assume one file equals one record. Establish those boundaries from product semantics.

## Build A Representative Discovery Matrix

Inspect enough records to cover expected variation, including where applicable:

- earliest, latest, and recent records;
- each product type and processing level;
- each mission, platform, instrument, or constellation generation;
- source collections and records duplicated across them;
- each asset layout and media format;
- single-band and multiband assets;
- optional, missing, null, empty, zero, and false values;
- different projections, dateline/polar geometries, and geometry types;
- public, signed, requester-pays, provider-authenticated, and alternate locations;
- intended whole-file and windowed/range-read access patterns for COGs;
- known malformed records and provider version transitions; and
- deletion, replacement, or revision behavior.

Record each fixture's source identity and retrieval URL or object prefix so relative references and discoveries remain reproducible.

## Inventory Each Record

For each representative record, inventory:

- identity and revision/version fields;
- acquisition datetime or interval;
- geometry and bbox consistency;
- source collection membership;
- all properties with observed types and null/missing behavior;
- assets, keys, hrefs, roles, media types, titles, descriptions, and bands;
- links and their relation, request behavior, and persistence value;
- Storage and Authentication registries or equivalent provider configuration;
- source and extension schema versions;
- relative-reference bases;
- object-store endpoints, regions, buckets/containers, requester-pays, and authentication requirements; and
- source defects or values that contradict the declared schema.

Compare values against the relevant specification, not only neighboring records. One convenient record does not establish a contract.

## Decide Identity And Revisions

Define:

- the stable source identity used for deduplication;
- the top-level field that preserves it, such as `stac_id` or `product_id`;
- whether that field receives the `primary_title` role;
- how provider revisions, reprocessing, and corrections behave; and
- whether a changed record is skipped, rejected, versioned, or ingested as a new product.

Tilebox's generated datapoint UUID is not source identity. Never derive identity from mutable URLs, list order, modification time, or undocumented field combinations.

## Decide Time And Geometry

Use STAC `datetime` or an equivalent documented acquisition timestamp. For interval-only records, use `start_datetime` as Tilebox `time` and preserve both interval bounds as generated fields.

For geometry:

- validate source GeoJSON before conversion;
- compare bbox with geometry and report contradictions;
- derive footprints from authoritative raster bounds only when the source lacks geometry and the recipe approves the method;
- transform footprints to geographic longitude/latitude with an explicit axis-order rule; and
- preserve antimeridian and polar behavior rather than replacing a footprint with an oversized bounding rectangle.

## Design Tilebox Collections

Research:

- whether the same source identity appears under multiple source collections;
- whether all candidate records share one schema;
- expected user queries, permissions, cadence, retention, and availability lifecycle; and
- stable semantic partition keys such as product type, processing level, mission, or OFFL/NRT availability.

Prefer one collection unless a stable partition serves a concrete operational or user-facing need. Names must be short, deterministic, and semantic, such as `all`, `l2a`, `sentinel-1-grd`, `offl`, or `nrt`. Do not derive names from source URLs or transient catalog titles.

Persist a deterministic `normalized record -> Tilebox collection(s)` routing function. Choose deliberately between:

1. **Canonical routing:** deduplicate repeated source discovery and ingest each source identity once into one canonical Tilebox collection. Use this when source collections are overlapping browsing/index views; retain useful memberships as fields.
2. **Mirrored routing:** ingest the same source identity into multiple Tilebox collections when those distinct collection views are useful Tilebox partitions in their own right.

Prefer canonical routing when source collections differ only by time or alternate labels. Do not reproduce “By Year” collections because Tilebox already has a temporal index. See the Wyvern pattern in `examples/optical-and-hyperspectral.md`.

## Infer Fields And Types

For standard STAC properties, identify both the source extension version and the current stable canonical version. Inventory deprecations and replacements before proposing fields. Use the modern target JSON Schema as the authoritative source for scalar, repeated, and enum types; do not use README field tables or observed JSON to override it. Use the source schema to interpret legacy input, and use representative non-null values to verify conformance, integer ranges, and source defects. Null values do not establish type; neither do empty arrays. Require stable element/object shapes and use only the dedicated messages permitted by `stac-protobuf-mapping.md`. Stop on incompatible shapes rather than stringifying conflicts or selecting the first observation.

Inventory decimal JSON strings that may be semantically numeric across representative records and variants. Convert them to the narrowest valid Tilebox integer only when documentation or consistent evidence establishes integer semantics; retain digit-only identifiers and codes as strings. Record accepted ranges and reject non-decimal or out-of-range values.

Use modern canonical STAC 1.1 JSON paths for schema mapping even when the source is older STAC or non-STAC. Record original source paths separately. Do not mirror deprecated properties into the Tilebox schema when a canonical replacement exists; document whether each legacy property is transformed, split, used only for validation, or omitted as redundant.

## Encode Source Normalization Without Guessing

Cleaning legacy or malformed metadata is normal converter work and needs no per-record warning when meaning is clear. Every normalization rule must still be:

- source- and version-specific;
- supported by provider documentation or representative evidence;
- deterministic; and
- covered by fixtures, including a nearby unsupported shape that must fail.

Numeric EPSG strings and singleton role strings can be repaired under an exact source/version condition. Empty href reconstruction and geometry contradictions require stronger evidence because they change identity or footprint semantics. Stop only when interpretations remain ambiguous, information must be invented, or information would be lost; never turn one provider repair into a global heuristic.

## Conversion Recipe Template

Persist a human-readable recipe next to the converter. Use this structure and remove sections that do not apply:

```yaml
source:
  name: <provider and product>
  kind: stac | xml | json-api | object-tree
  endpoints: [<catalog, API, or prefix>]
  declared_versions: [<schemas and versions>]
  representative_records:
    - id: <source id>
      location: <retrieval URL or object prefix>
      reason: <variation represented>

suitability:
  natural_record: <scene/acquisition/product>
  user_need: <discovery/access/lineage/orchestration>
  decision: ingest | abort | catalog-upstream-inventory

identity:
  source_key: <path or documented composite>
  tilebox_field: <stac_id/product_id/...>
  revision_policy: <skip/reject/version/new product>

time:
  source_paths: [<paths>]
  tilebox_time_rule: <rule>
  interval_fields: [start_datetime, end_datetime] # use [] when the source is not interval-only

geometry:
  source: <path or derivation>
  validation: <rules>
  normalization: <none or source-specific rule>

schema:
  kind: temporal | spatiotemporal
  fields:
    - name: <field>
      type: <protobuf scalar/message/enum>
      repeated: false
      canonical_pointer: </properties/...>
      source_paths: [<raw provider paths>]
      schema_evidence: <extension JSON Schema ref/version or provider contract>
      queryable: false
      roles: []
      normalization: <rule>

collections:
  names: [<approved names>]
  deduplication_key: <source key>
  routing_rule: <deterministic one-or-many collection rule>
  duplicate_across_collections: false | <documented condition>
  alternate_group_fields: [<fields retained instead of collection views>]

links:
  retain: [<relations/rules>]
  drop: [<source navigation relations>]

assets:
  key_rule: <stable key rule>
  location_rule: <resolution rule>
  cloud_alternates: <lossless patterns only>
  cog_delivery: <range-capable | download-only | not applicable; representative evidence>
  roles_and_bands: <mapping rule>

normalization_rules:
  - name: <source/version-specific rule>
    condition: <exact condition>
    evidence: <docs or fixtures>

unsupported:
  - path: <path>
    decision: extend protobuf | generated item field | reject | explicit omission
```

The recipe is a contract, not an exploratory notebook. Include final decisions and enough evidence to review them; keep discarded experiments elsewhere.

## Ask With Evidence

When blocked, show representative records, original and canonical paths, observed values/types/frequency, relevant specifications, the proposed decision, alternatives, and consequences. After the user decides, update the recipe and add a fixture; later runs use that decision instead of asking again.

## Phase A Is Complete When

The source is suitable, representative variation is covered, identity and routing are deterministic, every field has a stable type and canonical path, normalization rules are scoped and tested, and the persisted recipe contains no unresolved consequential decisions.
