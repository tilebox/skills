---
name: tilebox-ingesting-datasets
description: "Designs and implements Tilebox scene-catalog ingestors from STAC, provider XML or JSON, object-storage prefixes, and COG file trees. Use when onboarding a new metadata source, designing its canonical schema and collections, normalizing records to STAC 1.1 semantics, authoring Python or Go converters, constructing Assets, Links, Storage, or Authentication messages, or validating and ingesting converted datapoints."
license: MIT
compatibility: Requires the tilebox CLI and a Tilebox API key for schema operations and ingestion. Converter implementation uses tilebox-python or tilebox-go.
metadata:
  author: tilebox
---

# Ingesting Datasets Into Tilebox

Design and implement repeatable converters for discrete scenes, acquisitions, products, and other records that fit Tilebox's temporal or spatiotemporal catalog model. STAC is the primary and best-documented input, but provider XML/JSON, APIs, buckets, and file trees are supported when their evidence can establish the same canonical semantics.

## Operating Contract

Follow these rules for every new ingestor:

1. **Check suitability before designing a schema.** Tilebox catalog datasets represent discrete records with stable identity, time, and usually geometry. Do not manufacture scene records from chunks or slices of an analysis-ready multidimensional cube.
2. **Discover before creating.** Inspect representative records, source collections, file layouts, and known variants before proposing fields or Tilebox collections.
3. **Normalize source semantics to canonical STAC 1.1.** Older STAC and non-STAC inputs may require source-specific adapters, but the documented conversion recipe must define one canonical semantic model.
4. **Keep dedicated structures dedicated.** Use the well-known Geometry, Links, Assets, Authentication, and Storage messages. Follow the STAC Authentication extension for access metadata whenever its model applies. Map remaining stable Item properties to typed top-level dataset fields.
5. **Prefer typed values.** Do not use JSON strings, `google.protobuf.Value`, or `Struct` to bypass uncertain source types unless the canonical field itself is specified as arbitrary JSON.
6. **Never guess through ambiguity or silently drop metadata.** Stop, show evidence, propose alternatives, and persist the user's decision in the conversion recipe.
7. **Keep source normalization in the converter.** Every converter validates its source and owns provider-specific parsing and STAC 1.1 normalization.
8. **Create and inspect schemas through the Tilebox CLI.** Include schema discovery and creation as explicit workflow steps rather than putting them inside the ingestion program.
9. **Validate before bulk ingestion.** Query representative samples back from Tilebox and compare their canonical semantics before enabling a full run.

## Choose The Relevant References

Read only the references needed for the source and implementation language. Use Python by default. Use Go only when the user explicitly requests Go or the work is in an existing Go converter project.

| Reference | Use when |
| --- | --- |
| `reference/source-discovery-and-recipe.md` | Always. Decide whether the source fits Tilebox, inspect representative variation, design collections, and persist decisions. |
| `reference/canonical-stac-1.1.md` | Always. Normalize STAC core fields, extensions, links, media types, roles, and cloud locations. |
| `reference/non-stac-and-file-tree-sources.md` | The source is XML, a non-STAC API, an object-store prefix, a COG directory, or another file tree. |
| `reference/stac-protobuf-mapping.md` | Design the Tilebox schema and map canonical values into well-known messages or generated fields. |
| `reference/python-converter.md` | Default implementation path. Use tilebox-python and its high-level semantic Assets API. |
| `reference/go-converter.md` | The user explicitly requests Go, or an existing converter project is written in Go. Generate the dataset type and ingest it with tilebox-go. |
| `reference/asset-profile-wire-format.md` | Low-level Asset profile optimization and wire encoding for Go only. Do not read for Python: `AssetCollection` handles this automatically. |
| `reference/validation.md` | Test source validation, canonical semantics, compiled messages, query-back behavior, and full ingestion. |
| `reference/examples/earth-search-sentinel-2.md` | Handle legacy Band arrays, source-priority alternates, and useful queryables. |
| `reference/examples/optical-and-hyperspectral.md` | Match optical/hyperspectral edge cases such as scoped repairs, overlapping collections, or container Bands. |
| `reference/examples/sar.md` | Match SAR/altimetry edge cases such as empty hrefs, cross-directory Assets, and Band-scoped frequencies. |
| `reference/examples/authenticated-assets.md` | Model authenticated locations, Storage, and Authentication, including Copernicus Data Space patterns. |

## Workflow

### Phase A — Discover And Decide

Read `reference/source-discovery-and-recipe.md` and the applicable source reference. Produce a conversion recipe covering representative variation, canonical semantics, schema fields, identity, revisions, collection routing, and unsupported metadata. Do not create a dataset while consequential decisions remain unresolved.

### Phase B — Design And Create

Read `reference/stac-protobuf-mapping.md`, refresh `tilebox dataset create` metadata, then create the reviewed schema and collections through the CLI. Inspect the resulting contract before generating types or converter code. A non-empty dataset may gain compatible fields, but existing fields must never be retyped, renamed, removed, reordered, or renumbered.

### Phase C — Implement Conversion

Implement the recipe through the Python or Go reference. The converter validates source shape, applies only documented source/version normalizations, constructs canonical semantics, and routes datapoints; it does not discover or create schemas. Resolve relative hrefs as paths or URIs without requesting every object.

### Phase D — Validate And Ingest

Read `reference/validation.md`. Prove source, canonical, SDK, and Tilebox query-back behavior on bounded fixtures before enabling idempotent, observable bulk ingestion. For COGs, validate representative delivery routes for byte-range access when users need windowed reads. Tilebox has no STAC output interface, so compare reconstructed semantics rather than requiring a formal STAC export.

## Write Documentation For Public Users

Treat the dataset description as product documentation for people who want to discover and use the data, not as an ingestion report. Write it before handoff and publish it with `tilebox dataset update --description-file`.

Keep it concise and practical:

- explain the mission or source program, what the dataset contains, coverage or cadence caveats, and the license, citing authoritative source links;
- list available asset families, what each contains, and which product a user should choose for common workflows;
- state where the bytes live, whether credentials or requester-pays access are required, and which HTTPS, S3, or other locations are available; and
- include a small example that queries one datapoint, decodes it with `AssetCollection.from_datapoint(...)`, and reads or downloads an asset through `tilebox.storage.aio` when the dataset supports the canonical Assets API.

Do not expose converter mechanics, field-mapping decisions, deduplication rules, skipped helper records, normalization details, validation notes, or other implementation-only facts. Mention source-specific subsets or places only when they help users understand scientific content or coverage. Verify links, asset names, access requirements, and quantitative mission claims against live data and authoritative documentation.

## Suitability And Stop Conditions

A source fits when its natural records are discrete scenes, acquisitions, observations, or products with stable identity, trustworthy time, meaningful boundaries, and useful catalog metadata. An analysis-ready Zarr or Icechunk cube is normally consumed directly; catalog an upstream acquisition inventory or genuine product records only when that serves a concrete need.

When evidence supports one clear path, proceed without presenting unnecessary choices. Stop and ask only when authoritative evidence cannot determine:

- identity, time, geometry, record boundaries, deduplication, or collection routing;
- a stable field type/name or alignment of legacy Band arrays;
- the meaning of an unknown closed value or ambiguous source value;
- a valid representation for nested metadata or source authentication;
- an exact base/resolver for a relative or empty href; or
- a lossless cloud alternate or file-tree inference rule.

Show representative records, source and canonical paths, observed values/types, the proposed decision, alternatives, and consequences. For unsupported authentication, explicitly warn that it cannot be represented canonically. Persist the answer in the recipe.

## Ownership Boundaries

The source converter owns:

- discovery assertions and source validation;
- provider-specific parsing and normalization;
- canonical STAC 1.1 semantics;
- semantic Asset, Band, Link, Storage, and Authentication values;
- generated dataset property values;
- deduplication and collection routing; and
- observability and ingestion control.

The Python semantic Assets API owns applying STAC Asset/Band inheritance, validating the complete semantic Assets and their locations, and producing the `Assets`, `Storage`, and `Authentication` dataset fields. Python converters provide semantic values and do not recreate those generated messages by hand.

Go has no equivalent high-level API yet; isolate direct wire-format construction, follow the advanced reference, and cover it with golden round-trip tests.

## Expected Project Artifacts

A completed ingestion project contains:

- a documented conversion recipe with evidence and policy decisions;
- the Tilebox schema source and dataset documentation;
- a repeatable source converter and ingestion command;
- representative source fixtures, including malformed and normalized cases;
- canonical semantic, idempotency, collection-routing, and query-back tests; and
- operational guidance for credentials, revisions, retries, and full ingestion.

Do not build a generic schema-generator script. The agent performs discovery and schema design with the user; the committed program converts and ingests records under that documented contract.
