# Validation And Ingestion

Validate four layers independently: source assumptions, canonical semantics, compiled protobuf messages, and Tilebox query-back. A converter is not ready because one happy-path Item ingested successfully.

## Fixture Matrix

Turn every discovery-matrix dimension and normalization rule into a small recorded fixture. Add malformed near-neighbors, collection boundaries, geometry edge cases, alternate access modes, and explicit missing/null/empty/zero/false cases. Retain source identity and retrieval context, but remove secrets and signed query strings.

## Layer 1 — Source Validation

Assert the recipe's source versions, required fields, shapes, units, Band alignment, href bases/resolvers, XML/API versions, and file-tree grouping before normalization.

For extension properties, validate legacy input against its source schema before normalization. For decimal strings normalized to integers, cover zero, representative bounds, non-decimal input, and overflow. For deprecated closed values such as processing levels or timeliness categories, test every accepted source value and reject a nearby unknown value rather than inventing a canonical mapping.

Test that an unexpected future source shape fails with record ID, source path, and actionable evidence. Do not silently accept a provider change.

For every source-specific normalization rule, test both the exact accepted condition and a nearby unexpected condition that must still fail.

## Layer 2 — Canonical Semantics

Compare the normalized record with the recipe's canonical semantic model. Focus on identity/time/geometry, legacy Band merge and order, nested scope, exact Links and alternates, open values, and rejection of unsupported nested metadata.

Assert extension output types against the modern target JSON Schema, especially where README tables or observed encodings disagree. For every deprecated source property, assert the modern canonical value and pointer and the absence of a mirrored legacy schema field. When a legacy field is redundant, test its consistency check and prove it is not emitted.

Validate the canonical model semantically. Do not require a formal STAC export from Tilebox; Tilebox does not currently expose a STAC output interface.

## Layer 3 — SDK Compilation

### Python

Round-trip semantic assets through the high-level API:

```python
compiled = AssetCollection.from_assets(semantic_assets).to_fields()
resolved = AssetCollection.from_datapoint(scalar_datapoint(**compiled))
assert resolved == AssetCollection.from_assets(semantic_assets)
```

Use the project's scalar-xarray helper or a real queried datapoint. Assert resolved hrefs, schemes, roles, media types, Bands, and explicit values. Do not assert the SDK's internal representation in converter tests.

### Go

Validate every invariant in `asset-profile-wire-format.md`, then reconstruct semantic Assets with a test decoder. Assert deterministic output from shuffled source maps and equivalent input orderings where order is not semantic.

For Go, `asset-profile-wire-format.md` is authoritative. For both languages, assert registry resolution, primary href presence, effective Band values, open values, and protobuf validation.

## Layer 4 — Tilebox Query-Back

Create/resolve the approved collections and ingest a bounded sample. Query by source identity where that field is queryable:

```bash
tilebox dataset query <dataset-slug> \
  --filter "stac_id = '<representative-id>'" \
  --last 10Y \
  --limit 1 \
  --json
```

Use explicit `--after` and `--before` instead when the sample is outside a practical relative range. Verify exactly one expected result unless the approved revision or multi-collection policy says otherwise.

Compare collection, identity, time, geometry, every populated property, Links, resolved Assets/locations, registries, Band order/effective values, checksums/sizes/presence, custom values, and approved query filters.

Assert that the queried-back messages preserve the expected canonical STAC 1.1 semantics even though they are not exported as STAC Items.

## Asset Accessibility

Metadata correctness does not prove payload access. Treat locations as paths during conversion, then separately test one bounded metadata/range request or storage-client open for each promised access mode. Confirm authentication, requester-pays/custom endpoints, alternate selection, and which modes were tested; avoid full downloads.

For each COG location intended for direct access, test one representative object per distinct published route and relevant Asset family with a tiny byte-range request or bounded storage-client window open. For HTTP(S), require `206 Partial Content` with a valid `Content-Range`; `Accept-Ranges: bytes` on a HEAD response is only a hint. Abort immediately if a server ignores the Range header and starts a full response. Run these probes only during discovery/validation, never in the converter or for every Item.

If the file is a valid COG but no published location supports ranges, keep the COG media type and warn that whole-file reads may work while windowed reads will not. Record the route as download-only in the recipe and dataset documentation. If windowed access is an intended use case, ask whether to proceed with that limitation, obtain a provider-supported route, or mirror the file into suitable storage; never synthesize an unverified bucket URI.

Never place credentials in datapoints, fixtures, logs, conversion recipes, or job inputs.

## Collection Routing And Deduplication

Test routing as a pure function across every boundary and overlapping source membership. Source ordering/pagination must not affect the result; unknown partitions fail rather than creating collections, and every target collection exists before ingestion.

## Idempotency And Revisions

Run the same sample twice under the approved policy. Confirm duplicate behavior and returned created/existing counts or IDs. Then test a changed record with the same source identity.

Use `allow_existing=True` for normal idempotent retries of the exact same ingestion payload; Tilebox derives the same datapoint ID and deduplicates it. Do not use it to hide a changed-payload revision conflict. The recipe must decide whether a changed source record is rejected, assigned a versioned identity, or ingested as a new product.

## Bulk Ingestion Controls

After sample validation, checkpoint by stable source token/identity, group by collection, use SDK batching (at most 8192 datapoints per service request), and bound concurrency/retries. Distinguish transport failures from deterministic validation errors; report discovered/converted/skipped/existing/created/failed counts and quarantine bad records with source ID and collection.

Do not enable a complete backfill until source, canonical, SDK, and query-back tests pass for every fixture family.
