# Dataset Enum Fields

Use schema-local enums for closed vocabularies in new datasets. Both scalar and repeated enums can be queried server-side when marked `queryable`. Legacy hardcoded/generated well-known enum types remain supported for existing datasets but are deprecated for new dataset fields. This does not replace enum members inside dedicated messages such as Assets or Bands; use those messages' current SDK contracts.

## Define And Inspect The Vocabulary

CLI schema input:

```json
{
  "kind": "temporal",
  "fields": [
    {
      "name": "product_type",
      "type": "enum",
      "enum_values": ["L1", "L2"],
      "queryable": true,
      "description": "Product type.",
      "example_value": "L1"
    },
    {
      "name": "use_cases",
      "type": "enum",
      "enum_values": ["agriculture", "flood-monitoring", "forestry"],
      "repeated": true,
      "queryable": true,
      "description": "Applicable uses of the product.",
      "example_value": "[\"agriculture\", \"flood-monitoring\"]"
    }
  ]
}
```

Save this as `schema.json` and use `tilebox dataset create --name "Scenes" --code-name scenes --schema-file schema.json --json`. Use the returned slug in subsequent commands.

- Declare 1–100 unique, nonempty public strings per field, each at most 128 Unicode code points. These limits apply to scalar and repeated fields, queryable or not.
- Preserve exact spelling, case, punctuation, and Unicode. Values such as `etm+`, `L2__NO2___`, and `01` need no protobuf-identifier normalization.
- Vocabulary order is part of schema evolution. The server assigns protobuf symbols and numbers; do not supply or guess them.
- Enum fields do not consume the two-queryable-string allowance. Arbitrary repeated strings or numbers do not gain queryability from enum support.
- Input uses `enum_values`; `tilebox dataset get --json` exposes `enumValues`, along with `type: "enum"`, `repeated`, and `queryable`. Discovery output is not directly reusable as schema input.

Inspect the actual contract before querying an existing dataset:

```bash
tilebox dataset get <dataset-slug> --json \
  | jq '[.fields[] | select(.type == "enum") | {name, enumValues, repeated, queryable}]'
```

## CLI Filters

Use exact public strings, never enum numbers or generated symbols. The following commands assume the example schema above and a time range containing data:

```bash
# Scalar equality; use OR for alternatives.
tilebox dataset query <dataset-slug> --last 7d \
  --filter "product_type = 'L1' OR product_type = 'L2'" --json

# Include missing scalar values explicitly.
tilebox dataset query <dataset-slug> --last 7d \
  --filter "product_type <> 'L1' OR product_type IS NULL" --json

# Contains all requested tags; additional tags are allowed.
tilebox dataset query <dataset-slug> --last 7d \
  --filter "A_CONTAINS(use_cases, ('agriculture', 'flood-monitoring'))" --json

# Contains any requested tag, combined with scalar equality.
tilebox dataset query <dataset-slug> --last 7d \
  --filter "product_type = 'L2'" \
  --filter "A_OVERLAPS(use_cases, ('agriculture', 'forestry'))" --json

# Includes empty/missing lists.
tilebox dataset query <dataset-slug> --last 7d \
  --filter "NOT A_CONTAINS(use_cases, 'agriculture')" --json
```

Scalar enums support `=`, `<>`, `IS NULL`, and `IS NOT NULL`, combined with `AND`, `OR`, and `NOT`. Ordering comparisons and numeric literals are invalid. Comparisons against missing scalars evaluate to unknown, so inequality and negated equality exclude them unless a null branch is added.

Repeated enums support `A_CONTAINS` (all) and `A_OVERLAPS` (any), with either a single string or a parenthesized list of strings. Do not use JSON brackets in CQL2. The CLI accepts 1–100 nonempty string arguments; empty `()` is unsupported. Scalar comparisons, null checks, set equality, and contained-by predicates are unsupported for repeated enums. Unknown vocabulary values are errors, not empty matches.

## Python Ingestion And Queries

For Python schema authoring, use the same `"type": "enum"`, `"enum_values": [...]`, and optional `"repeated": True` field dictionaries with `Client.create_or_update_dataset`. Other repeated Python field types use `list[T]` instead. In ingestion projects, create schemas separately through the CLI.

Python ingestion accepts declared public strings and lists of strings in flat records, column dictionaries, pandas DataFrames, and xarray datasets. It resolves the dataset's schema before encoding protobuf numbers:

```python
from datetime import UTC, datetime
from tilebox.datasets import field

collection.ingest([
    {
        "time": datetime(2026, 10, 29, tzinfo=UTC),
        "product_type": "L2",
        "use_cases": ["agriculture", "flood-monitoring"],
    }
])

data = dataset.query(
    temporal_extent=("2026-10-29", "2026-10-30"),
    filter=(field("product_type") == "L2")
    & field("use_cases").contains_all(["agriculture", "flood-monitoring"]),
)

# Other membership expressions, composable with &, |, and ~:
field("use_cases").contains("agriculture")
field("use_cases").contains_any(["agriculture", "forestry"])
~field("use_cases").contains("agriculture")
```

Dataset and collection queries, sync and async, use the same expressions. Custom enum inputs must be public strings, not integers or legacy enum constants. The expression builder is schema-unaware; the server validates vocabulary membership, queryability, and enum-specific operators.

Scalar results contain public strings; detect missing values with xarray's `.isnull()`. Repeated results are one-dimensional object columns with a `list[str]` per row, including independent `[]` values for empty rows. There is no padded extra dimension. Extract a row with `data.use_cases.isel(time=0).item()`. Xarray `.isin()` does not search inside each list; prefer server-side membership filtering.

## Repeated Values And Presence

Missing and empty repeated fields both represent the empty set. Membership predicates never return null: a nonempty all/any request does not match an empty field, while its negation does. The SDK/API also supports empty requests: `.contains_all([])` matches every row and `.contains_any([])` matches none; the CLI cannot express these empty requests.

The server deduplicates repeated enum values before storage and ID calculation, retaining first-occurrence order. `['forestry', 'agriculture', 'forestry']` becomes `['forestry', 'agriculture']`. Membership ignores order, but reordering distinct values can change the payload and datapoint ID. Do not sort automatically unless the recipe defines a canonical order. Enums cannot preserve meaningful multiplicity; use another representation when duplicate occurrences matter.

Vocabulary and predicate limits are 100 entries. A raw ingestion list can contain more entries through duplicates; its stored distinct values still fit the vocabulary. Missing scalar values remain absent; explicit protobuf zero and undeclared numbers are invalid for custom enums. Decoded CLI/Go records may omit absent fields rather than emitting `null` or `[]`.

## Go And Low-Level Contracts

Go schema helpers are `field.Enum("product_type", "L1", "L2").Queryable()` and `field.Enum("use_cases", "agriculture", "flood-monitoring", "forestry").Repeated().Queryable()`. `field.ProtoEnum` is the legacy protobuf-enum helper; do not use it for new dataset fields.

Go ingestion still accepts protobuf messages. Use `datasets.NewDatapointDescriptor(dataset)` and its `EnumNumber(fieldName, publicValue)` lookup to obtain the number for each scalar or repeated member. Do not invent a string-map ingestion API. Annotation-aware `datasets.UnmarshalDatapoint` returns public strings; generated enum `.String()` returns protobuf symbols instead.

Go filter expressions use public strings:

```go
query.And(
    query.Field("product_type").Equal("L2"),
    query.Field("use_cases").ContainsAll("agriculture", "flood-monitoring"),
)
query.Field("use_cases").ContainsAny("agriculture", "forestry")
```

Prefer SDK conversion over implementing descriptor handling. When a low-level integration needs it, custom fields use `TYPE_ENUM`, no `type_name`, and `FieldAnnotation.enum_values` entries containing `value`. Resolve the returned message by `type_url`; pair field annotations by field declaration order, retaining empty placeholders. Pair vocabulary entries with nonzero descriptor enum members in declaration order and use their actual numbers, not `index + 1` or parsed symbol names.

## Vocabulary Updates

Schema updates replace the full custom schema. Retain every existing field and annotation that should remain.

- On a non-empty dataset, append values only. Existing values must remain an unchanged prefix; renaming, removal, reordering, or insertion changes existing identities and is rejected.
- On an empty dataset, vocabularies may be replaced or reordered. Emptiness uses collection counts, including soft-deleted collections that remain restorable; it is not a permanent "ever ingested" flag. Let the backend decide compatibility rather than deleting data to force it.
- On a non-empty dataset, existing field types, repetition, and queryability cannot change. New fields must be appended and non-queryable. Choose categorical filters before ingestion.
- Refresh Python dataset objects/descriptors and regenerate Go types after vocabulary updates. A stale schema cannot encode newly added values.
- Do not automatically replace a populated legacy enum/string field with a custom enum. That is a type change; use an approved replacement dataset and explicit conversion/reingestion when needed. Old enum numbers do not define the new vocabulary's numbers.

Unknown provider values require an explicit decision: correct a documented normalization, append a valid new value through a schema update, or reject/quarantine the record. Do not silently map them to zero, `UNSPECIFIED`, or an invented fallback.
