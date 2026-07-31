# Optical And Hyperspectral Edge Cases

These are approved reusable rules for equivalent source conditions, not provider branches to copy into unrelated converters.

## Satellogic: Narrow Numeric-String Projection Repair

Evidence: [representative EarthView metadata](https://satellogic-earthview.s3.us-west-2.amazonaws.com/data/json/zone=10N/region=547925_5255414/date=2022-08-06/20220806_192834_SN23_10N_547925_5255414_metadata.json)

Some records publish `proj:epsg` as a wholly numeric string such as `"32610"`, although the declared Projection extension expects a number. A source/version-scoped repair may normalize this to `EPSG:32610`; reject other strings. Test numeric, supported numeric-string, and invalid-string cases. This is not a general Projection coercion.

Generate native S3 alternates only when canonical AWS URLs expose the provider, bucket, region, and exact key losslessly. Use source Asset keys rather than deriving keys from titles.

## Wyvern: Hyperspectral Bands And Overlapping Collections

Evidence: [catalog browser](https://opendata.wyvern.space), [STAC root](https://wyvern-odp.com/catalog.json), and a [representative Dragonette Item](https://wyvern-odp.com/product-type/extended/wyvern_dragonette-003_20250319T000402_d5ebab06_l2a/wyvern_dragonette-003_20250319T000402_d5ebab06_l2a.json)

### Preserve Hyperspectral Band Semantics

Dragonette L2A uses aligned legacy EO/Raster arrays for its 31-Band COG. Merge each index into one semantic Band while preserving order and wavelength-specific metadata. Missing closed EO common names are valid; never invent one.

Observed malformed `roles: "metadata"` may be repaired to `roles: ["metadata"]` only under the verified source/version condition. Reject other unexpected shapes and never iterate a role string into characters.

Custom `wyvern-data.com` or `wyvern-odp.com` domains do not prove S3/GCS locations; retain HTTPS unless authoritative configuration supplies a native object href.

A tested Wyvern COG was downloadable through `wyvern-data.com`, but that route did not support byte-range reads and no public native bucket was available. Keep the COG media type when its internal layout is valid, but report the HTTPS route as download-only and warn that `StorageClient.open_geotiff()` window reads will fail. Do not synthesize an S3/GCS alternate. Revalidate representative COGs if the provider changes its delivery service.

### Canonicalize Navigation Collections Deliberately

The source's **By Year**, **By Application**, and **By Spectral Range** trees overlap, so one Item can appear in several collections. In `open_data.wyvern.dragonette`, ingest each stable product identity once into a product-oriented collection:

- `Standard VNIR`
- `Extended VNIR`
- `Surface Reflectance`

Preserve application memberships as repeated `industries`, for example `["energy", "forestry"]`. Do not create year collections because Tilebox already indexes time.

Apply this pattern when collections are alternate indexes, not distinct acquisitions. If source collections are useful independent Tilebox partitions, deliberate duplication can still be correct; record that routing choice explicitly.

## Tanager: Mixed STAC Generations And Container Bands

Evidence: [representative Tanager Item](https://www.planet.com/data/stac/tanager-core-imagery/coastal-water-bodies/20250225_153627_16_4001/20250225_153627_16_4001.json)

This source declares STAC core 1.0 while already publishing valid unified `bands`. Use that array directly; do not split and re-merge it because of the core version.

Some HDF5 Assets contain hundreds of Band Objects. Preserve every difference and the original order. If `Band.name` does not encode an internal subdataset path required by consumers, stop and ask whether the model must be extended—never assume the name reconstructs a subdataset URI.

Canonical `storage.googleapis.com/open-cogs/...` hrefs should gain exact `gs://open-cogs/...` alternates when no query state or transformation would be lost. Null-only sampled properties do not establish a schema type; inspect more Items and omit the field until stable non-null evidence exists.
