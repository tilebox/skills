# Mapping Canonical STAC Into A Tilebox Dataset

Tilebox does not store a complete STAC Item in one monolithic protobuf message. The generated dataset type is the aggregate: required dataset fields hold time and geometry, well-known messages hold dedicated STAC structures, and generated top-level fields hold the remaining canonical Item properties.

## Field Placement

Use this baseline mapping:

| Canonical STAC path | Tilebox representation |
| --- | --- |
| `/properties/datetime`, or `/properties/start_datetime` for an interval-only Item | required dataset `time` |
| `/properties/start_datetime` and `/properties/end_datetime` on an interval-only Item | generated Timestamp fields preserving both bounds |
| `/geometry` | required spatiotemporal `geometry` |
| `/links` | `datasets.stac.v1.Links` |
| `/assets` | `datasets.stac.v1.Assets` |
| `/properties/auth:schemes` | `datasets.stac.v1.Authentication` |
| `/properties/storage:schemes` | `datasets.stac.v1.Storage` |
| `/properties/providers` | repeated `datasets.stac.v1.Provider` |
| `/properties/processing:software` | `datasets.stac.v1.ProcessingSoftware` |
| other stable `/properties/...` leaves | generated typed top-level fields |

STAC `id` normally becomes an explicit generated string such as `stac_id`; it is not the Tilebox datapoint UUID. Source collection membership informs routing and is retained as a field only when the approved dataset contract requires it.

## Inspect Current Contracts

Refresh CLI support before creating the schema:

```bash
tilebox agent-context dataset create --output-schema
```

Current schema fields support scalar and Tilebox well-known types, fully qualified `datasets.stac.v1` message and enum names, repetition, and these annotations:

- `description`
- `example_value`
- `source_json_pointer`
- `queryable`
- `json_schema_ref`
- `roles`, currently including `primary_title`

The protobuf repository is the exact source of message names, fields, enum values, validation, and presence. Relevant packages are:

- `apis/datasets/stac/v1` for Assets, Links, Storage, Authentication, EO/Raster/Projection/View/File/Classification, SAR, Satellite, Product, and Processing;
- `apis/datasets/v1/dataset_type.proto` for fields and annotations; and
- Tilebox SDK generated modules for language-specific constructors.

Do not copy pseudo-protobuf from a guide when current generated types are available.

## Minimal Schema Example

This example shows identity, a dedicated message, a repeated message, and a queryable scalar. Add only fields required by the recipe.

```json
{
  "kind": "spatiotemporal",
  "fields": [
    {
      "name": "stac_id",
      "type": "string",
      "description": "Stable identifier of the source STAC Item.",
      "example_value": "S2B_T35SLA_20260730T090532_L2A",
      "source_json_pointer": "/id",
      "queryable": true,
      "roles": ["primary_title"]
    },
    {
      "name": "assets",
      "type": "datasets.stac.v1.Assets",
      "description": "Canonical STAC 1.1 Assets with semantic locations and Bands.",
      "source_json_pointer": "/assets"
    },
    {
      "name": "providers",
      "type": "datasets.stac.v1.Provider",
      "repeated": true,
      "description": "Organizations that captured, processed, or host the product.",
      "source_json_pointer": "/properties/providers"
    },
    {
      "name": "cloud_cover",
      "type": "float64",
      "description": "Cloud cover over the scene as a percentage from 0 to 100.",
      "example_value": "0.057545",
      "source_json_pointer": "/properties/eo:cloud_cover",
      "queryable": true
    }
  ]
}
```

Omit unused optional messages. Examples must use final canonical values, not raw provider fragments. Create the reviewed schema through `tilebox dataset create`, then inspect the stored contract:

```bash
tilebox dataset get <dataset-slug> --json
```

## Canonical Source JSON Pointers

`source_json_pointer` is an RFC 6901 pointer to the field's position in the normalized modern STAC 1.1 semantic model. It is not a pointer into raw provider XML, a deprecated extension property, legacy extension JSON, or an API response. When a converter modernizes a legacy property, annotate the field with the modern canonical pointer and record the legacy input path separately in the recipe.

Examples:

```text
/assets                                      -> assets
/links                                       -> links
/properties/eo:cloud_cover                   -> cloud_cover
/properties/s2:nodata_pixel_percentage       -> nodata_pixel_percentage
/properties/storage:schemes                  -> storage
```

Apply RFC 6901 escaping to `~` and `/`; colons remain literal. Put the canonical pointer on the field and record different raw source paths and transformations in the recipe.

## Field Descriptions And Examples

Describe final meaning, units, ranges, and representation. Keep provider defects and conversion rationale in the recipe. Asset descriptions must state the actual primary/alternate policy.

## Readable Field Names

Remove the leading `properties` segment, join nested segments with underscores, and normalize invalid characters to lowercase snake case. Preserve standard STAC extension prefixes exactly as published instead of expanding them: for example, `proj:code` becomes `proj_code` and `sat:orbit_state` becomes `sat_orbit_state`. Name fields by the extension that owns the property, not by the dataset's application domain.

The only standard-extension naming exceptions are `eo:cloud_cover` to `cloud_cover` and `eo:snow_cover` to `snow_cover`. These frequently queried Item fields stay concise; every other EO property retains the `eo_` prefix. Their `source_json_pointer` values remain `/properties/eo:cloud_cover` and `/properties/eo:snow_cover`.

Only a provider- or source-specific namespace may be dropped when the dataset context leaves the result unambiguous. For example, `umbra:collect_id` may become `collect_id` and `s1:slice_number` may become `slice_number`. Never drop a standard namespace merely because the dataset uses only one extension. The canonical `source_json_pointer` always retains the namespace, including a dropped provider prefix.

Compare names against every generated, required, dedicated, and reserved field. Stop on collisions; never append hashes, numbers, or order-dependent suffixes automatically.

For example, `/properties/view:azimuth` and `/properties/view_azimuth` both propose `view_azimuth`. Show both paths and ask whether to rename explicitly, keep a typed object, exclude a field, split datasets, or abort.

## Type And Shape Rules

For a standard STAC extension property, use the JSON Schema for the modern canonical extension version as the authoritative target type contract. Use older schemas only to interpret legacy input before normalization. Do not substitute an extension README's field table or infer a different type only because sampled records happen to use one representation. Use representative non-null values to verify conformance, presence, width, and source defects. JSON Schema `number` normally maps to `float64`; `integer` maps to an integer type selected by the range rules below; closed strings map to an available Tilebox STAC enum; open strings remain strings. Preserve stable repeated element types and explicit presence for zero, false, and empty values. Null or missing becomes unset; null-only non-standard fields remain omitted until their type is known. Reject incompatible shape changes.

Choose integer widths deliberately before creating the dataset:

- Default to `int32` when the source contract guarantees the value fits a signed 32-bit integer. Common examples include raster width/height or shape, bits per sample, band/count fields, row/column indices, and other bounded product dimensions.
- Use `int64` only when the documented or observed domain can exceed the signed 32-bit range; do not select it merely because Python uses `int` or a source JSON Schema says `integer` without narrower bounds.
- Use `uint64` only when unsigned 64-bit semantics are part of the source contract, not as a substitute for choosing an appropriate signed width.
- Record the range evidence in the conversion recipe when the width is not obvious from the field's specification.

This choice also affects command-line JSON. Standard ProtoJSON renders `int64` and `uint64` values as JSON strings to preserve precision, including repeated values, while `int32` values render as JSON numbers. Never change semantic meaning solely for presentation, but prefer `int32` for naturally bounded values so consumers do not receive avoidable stringified integers.

Some providers serialize semantically numeric integer properties as decimal JSON strings, including values they consider 64-bit. Normalize those values back to a Tilebox integer rather than preserving the transport representation as `string`. Choose `int32`, `int64`, or `uint64` from the documented and observed domain, not from the fact that the JSON token was quoted. Record the evidence and parsing rule in the recipe, and reject non-decimal or out-of-range values. Do not apply this rule to identifiers or codes whose canonical semantics are strings merely because they contain digits.

Do not use JSON strings, arbitrary Structs, or `google.protobuf.Value` to force a mixed field into the schema.

## STAC Message Boundaries

At dataset field scope, use STAC-specific structured messages only for `Assets`, `Links`, repeated `Provider`, `ProcessingSoftware`, `Storage`, and `Authentication`. Geometry remains the required Tilebox spatial field. STAC enum types may be used for individual generated fields.

Do not use Item-level extension property-group messages such as `SARProperties`, `SatelliteProperties`, `ProductProperties`, or analogous convenience containers as dataset fields. Flatten each property the collection actually publishes into its own namespace-preserving generated field, with the type defined by the extension JSON Schema. Do not exhaustively add every property offered by an extension when the source does not publish it.

This rule does not authorize flattening nested metadata. Asset- and Band-scoped values stay associated with their Asset or Band and use the supported nested messages there; unsupported nested metadata follows the stop conditions below.

## Queryable Fields

Make stable source identity queryable when users need direct record lookup. Add common, stable scalar filters only when a realistic use case and type are clear, such as cloud or nodata percentage, platform, product/processing identifier, orbit value, or projection code.

Inspect a source STAC API's `/queryables` endpoint when available, but treat it as provider evidence rather than a list to copy. Select only fields that remain useful, stable, and correctly typed in the canonical Tilebox schema.

Default large messages, repeated values, Assets, Links, registries, arbitrary structures, and low-value high-cardinality values to non-queryable. Queryable fields support typed filtering; they are not guaranteed secondary indexes.

Use `json_schema_ref` when the STAC Queryables definition identifies the property through a stable `$ref`. Keep the canonical pointer and schema reference separate.

Assign `primary_title` to at most one stable human-readable identity field. Thumbnail selection remains an Asset role, not a field role.

## Item-Level Versus Nested Metadata

Item-level extension properties become generated fields unless they are one of the dedicated STAC structures listed above. Asset/Band values remain inside Assets/Bands; Link access values remain on Links and their registries; Storage and Authentication remain dedicated messages.

The presence of an Asset-level grouping message does not mean the same property at Item scope should be forced into it. Conversely, an unsupported nested field cannot be moved to a top-level field without losing which Asset or Band it described.

## Collection And Schema Boundaries

Use separate datasets when record families cannot share one stable schema. Collection design and routing belong to `source-discovery-and-recipe.md`. Create approved collections before ingestion and regenerate Go types after compatible schema updates.

On non-empty datasets:

- add compatible fields when needed;
- update descriptions and documentation deliberately;
- never retype, rename, remove, reorder, or renumber existing fields; and
- create a new dataset for incompatible wire changes.

Treat field type choices as permanent once the first datapoint has been ingested. Do not rely on deleting datapoints or collections to make a later retype possible: deployed backends may retain ingestion history and reject the breaking update even when every current collection is empty. If a retype is rejected, create a replacement dataset with the corrected schema rather than restoring the old data and retrying.

Inspect collection counts and the current schema before every update.
