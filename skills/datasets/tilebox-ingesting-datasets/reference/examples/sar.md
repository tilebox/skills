# SAR And Altimetry Edge Cases

Apply these reusable rules only when discovery establishes equivalent product semantics and source behavior.

## Umbra: Empty Asset Hrefs

Evidence: [representative Umbra Item](https://s3.us-west-2.amazonaws.com/umbra-open-data-catalog/stac/2025/2025-03/2025-03-24/451caf11-e082-4e84-ada9-01609128028f/451caf11-e082-4e84-ada9-01609128028f.json)

The sampled STAC Item contains empty Asset hrefs. Tilebox requires a concrete primary location, so reject the record unless the Item, declared Links, or an authoritative provider path contract establishes it. A resolver must be source/version-scoped and fixture-backed; never guess filenames from sibling Assets.

The converter must never make storage, list-object, HEAD, or range requests. If metadata does not establish a clear layout, ask the user how to proceed. Offer a small bounded discovery probe only with permission, record its result in the recipe, and keep it out of per-Item conversion.

Preserve typed SAR/Satellite scope, normalize deprecated `sar:product_type` into the open Product value, and use the exact CPHD media type `application/vnd.cphd` when applicable rather than `application/octet-stream`.

## Capella: Cross-Directory Assets

Evidence: [Capella catalog](https://capella-open-data.s3.us-west-2.amazonaws.com/stac/catalog.json)

Preview and thumbnail Assets may reside in a sibling GEO directory while data belongs to SLC, GEC, SICD, SIDD, CPHD, or another product. Preserve each concrete href; never force all Assets under the Item's data directory or one shared leaf prefix.

Keep provider Product types as open strings unless the current protobuf explicitly defines a closed value. Preserve exact roles and custom media types such as `application/vnd.cphd`. Discover CSI/layout variants before defining any source resolver.

## General SAR Scope And Vocabularies

Validate closed SAR values such as polarization against the current protobuf. Keep instrument mode and Product type open where their specifications are open. Frequency, polarization, resolution, looks, and similar metadata must remain at Item, Asset, or Band scope as published; do not lift them merely to reuse a convenient field.

Stop when a closed value is unknown or polarization/Band frequency order is ambiguous.

## Altimetry Is Not SAR

Sentinel-3/Sentinel-6-style products can legitimately carry distinct C- and Ku-band frequency metadata in `Band.sar`, including unnamed Bands. Preserve that Band-scoped order and metadata.

Do not relabel `altm:instrument_mode`, `altm:instrument_type`, legacy `band_width`, or other altimetry metadata as SAR. If current messages cannot preserve the original scope, ask whether to add generated typed fields, extend the protobuf, or exclude the affected product family.
