# Earth Search Sentinel-2 Edge Case

Use this as a reusable pattern for legacy per-Asset Band metadata and source-declared alternate locations, not as a complete Earth Search converter specification.

## Evidence

- source collection: `sentinel-2-c1-l2a`
- source Item: [`S2B_T35SLA_20260730T090532_L2A`](https://earth-search.aws.element84.com/v1/collections/sentinel-2-c1-l2a/items/S2B_T35SLA_20260730T090532_L2A), declaring STAC 1.0
- Tilebox dataset: `open_data.aws_earth.sentinel2`

Pin repairs to the observed source collection/version and recheck representative Items when that evidence changes.

## Approved Reusable Rules

### Merge Legacy Band Arrays By Index

The Item uses legacy `eo:bands` and `raster:bands`. For each Asset:

1. normalize both arrays to STAC 1.1 Band fields;
2. when both exist, require compatible lengths and merge entries by index;
3. preserve source order; and
4. reject conflicts unless the recipe defines precedence.

This applies to spectral, visual, and mask Assets. In particular, `visual` must retain its physical source order `B04`, `B03`, `B02` (red, green, blue). Never sort Bands or move metadata between Assets based on similar names.

### Mirror Source Location Priority

The source publishes HTTPS as primary and S3 as an alternate on applicable Assets, so the conversion preserves that order. Let the Python semantic Asset API compile shared Storage records; do not manipulate profiles or derive one href from another with lossy string replacement.

Location priority is source evidence, not a universal HTTPS-first policy. If another supported source shape reverses it, preserve that order unless the dataset recipe deliberately says otherwise.

### Retain Provenance, Not Navigation Noise

Keep useful source-Item, service, license, and documentation Links. Treat `self`, `parent`, `collection`, and `root` as resolution context unless they remain useful to Tilebox consumers.

## Queryables In The Live Dataset

- `stac_id`
- `platform`
- `cloud_cover`
- `nodata_pixel_percentage`

These demonstrate useful categories—stable lookup/routing fields and common quality filters—not a list to copy. Keep queryability narrow and evaluate it using the mapping guidance.

## Stable Query-Back Check

```bash
tilebox dataset query open_data.aws_earth.sentinel2 \
  --filter "stac_id = 'S2B_T35SLA_20260730T090532_L2A'" \
  --after 2026-07-30T09:10:00Z \
  --before 2026-07-30T09:11:00Z \
  --limit 1 \
  --json
```

Validate source identity/time/geometry, exact Asset keys and resolved hrefs, merged Band semantics and order, registry references, retained Links, and the four queryable fields. Do not assert serialized profile indices, protobuf field order, or formal STAC output.
