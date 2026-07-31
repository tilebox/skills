# Canonical STAC 1.1 Normalization

Normalize every accepted source to STAC 1.1 semantics before mapping it into Tilebox fields. This applies to valid older STAC, mixed-generation provider records, and non-STAC adapters. Semantic preservation is the goal; byte-identical source JSON is not.

## Normalization Order

For each record:

1. Parse and validate the declared source format.
2. Apply the source- and version-specific normalization defined by the conversion recipe.
3. Convert relative href and Link paths to absolute paths or URIs against the verified retrieval or self URL. Do not fetch each target as part of this step.
4. Normalize STAC core fields and extension layouts to 1.1 semantics.
5. Build semantic Links, Assets, Bands, Storage, and Authentication.
6. Map remaining canonical Item properties into the approved generated fields.
7. Pass semantic assets through the language-specific compilation path.

Keep raw source paths and normalization rules in the conversion recipe. Dataset field annotations identify canonical output paths, not provider provenance.

## Core Item Fields

Handle STAC core deliberately:

- `type` canonicalizes to `Feature`; do not store it as a dataset field.
- The canonical semantic model uses `stac_version` 1.1.0.
- Derive `stac_extensions` from populated canonical fields; do not preserve obsolete extension URLs as datapoint values.
- `geometry` becomes the required spatiotemporal Geometry field. Validate source GeoJSON before conversion.
- `bbox` is derivable from geometry. Report contradictions instead of silently choosing one.
- `datetime` normally supplies Tilebox `time`.
- When an Item has only `start_datetime` and `end_datetime`, use `start_datetime` for the required Tilebox `time` and retain both interval bounds as generated fields.
- STAC `id` is a provider identity, not the generated Tilebox datapoint UUID. Preserve it in an approved field such as `stac_id`.
- Source `collection` informs collection analysis but does not automatically become a Tilebox collection or datapoint field.

Do not preserve source catalog navigation merely to recreate the old graph. Tilebox becomes the catalog after import. Define source-collection deduplication and routing in `source-discovery-and-recipe.md`.

## Unified STAC 1.1 Bands

STAC 1.1 uses one `bands` array on an Asset. Common fields are unprefixed while extension fields retain their prefixes:

```json
{
  "bands": [
    {
      "name": "B04",
      "data_type": "uint16",
      "nodata": 0,
      "unit": "1",
      "eo:common_name": "red",
      "eo:center_wavelength": 0.665,
      "eo:full_width_half_max": 0.038,
      "raster:sampling": "area",
      "raster:scale": 0.0001,
      "raster:offset": -0.1,
      "raster:spatial_resolution": 10
    }
  ]
}
```

Older Items often use parallel `eo:bands` and `raster:bands` arrays. Merge them by index only when alignment is valid or an approved provider rule establishes it. Preserve the resulting source band order. Stop when lengths or meanings conflict.

Do not decide solely from `stac_version`: mixed-generation sources may declare STAC 1.0 while already publishing a unified `bands` array. Prefer a valid unified array when present.

Band `name` is optional in STAC 1.1 but must be non-empty when present. Do not invent names for unnamed altimeter, SAR, or container bands unless the provider defines one.

## Asset And Band Inheritance

Preserve the source's semantic Asset and Band values. Band values override inheritable Asset values. Different effective Band values remain distinct.

Do not move Band identity such as name, description, or spectral EO metadata to unrelated top-level fields. Do not flatten Asset- or Band-scoped unsupported metadata into Item properties because doing so loses association.

## Extension Conventions

Use the current generated `datasets.stac.v1` messages as the source of truth for supported fields. These conventions guide normalization; inspect the current protobuf files before implementing a converter.

### EO

- Keep `eo:cloud_cover` and `eo:snow_cover` as Item properties and generated top-level fields.
- Keep common name, wavelength, width, and solar illumination on Bands.
- EO common names are closed. Reject unknown values rather than inventing enum values.
- Hyperspectral bands may omit common names; preserve wavelengths and descriptions.

### Raster

- Map the complete STAC `DataType` vocabulary, not protobuf reflection kinds.
- Preserve explicit numeric nodata, scale, offset, unit, sampling, and spatial resolution when represented.
- Sampling is the closed `area` or `point` vocabulary.
- Special string nodata, unsupported statistics, and packed bit semantics require current protobuf support or an Asset/Band-scoped extension decision.

### Projection

Normalize older numeric `proj:epsg: 32635` to `proj:code: "EPSG:32635"`. Numeric strings need an explicit source/version normalization rule. Validate shape as Y/X, bbox as four 2D or six 3D values, and transform as six 2D or nine 3D values.

Canonical JSON null becomes protobuf absence. Unsupported Asset-level WKT2, PROJJSON, geometry, or centroid cannot be moved to Item fields without losing scope.

### View

Use typed View fields where the message supports the Asset/Band scope. Stable Item-level fields such as off-nadir, sun azimuth, or sun elevation can become generated fields. If unsupported fields occur inside an Asset, stop and consider extending View.

### File

- Decode lowercase hexadecimal multihashes to bytes; do not store the hexadecimal text in a bytes field.
- Preserve explicitly present file size, including zero.
- Keep `file:local_path` relative.
- Do not reinterpret unsupported byte-order, header, or format-specific fields.

### Classification

Map supported simple class fields directly. Packed classification bitfields need dedicated modeling; never approximate them as simple classes or Item properties.

### SAR, Satellite, Product, And Processing

- Use closed SAR enums for frequency band, polarization, and observation direction.
- Keep open instrument mode and product type strings as strings.
- Normalize deprecated `sar:product_type` into Product semantics.
- Preserve different C/Ku or other frequency values on their individual Bands.
- Keep Satellite orbit category separate from timestamped orbit state vectors.
- Keep Product acquisition type as its closed enum; product type and timeliness category remain open strings.
- Map `processing:software` as one typed software/version map, not one generated field per dynamic software name.

Do not reinterpret altimetry-specific properties as SAR merely because both mention bands or instrument modes.

### Storage And Authentication

Asset and Link hrefs identify resources, Storage describes the hosting service, and Authentication describes how permission is obtained. Keep these concerns separate.

Never place object paths in Storage. Never infer access from Storage alone. Never include secret values in Authentication metadata; schemes describe mechanisms, not credentials.

Follow the STAC Authentication extension whenever it can represent the source. Preserve its scheme registry and the exact Asset, alternate-location, and Link references. Inspect the current Authentication protobuf before mapping a flow. If the extension or protobuf cannot represent provider-required request semantics without loss, warn the user and ask how to proceed; do not invent custom authentication fields or references. An API extension may be the correct resolution.

## Open And Closed Vocabularies

Reject unknown values only when the relevant STAC specification closes the vocabulary. Use exact custom strings for open vocabularies.

### Media Types

Media types are open IANA strings. Preserve exact custom types such as `application/vnd.cphd` instead of downgrading them to `application/octet-stream`.

Do not claim COG solely from `.tif`. Validate the file layout or trust an authoritative source contract.

COG structure and delivery suitability are separate. A structurally valid COG remains a COG when its HTTPS server lacks byte-range support, but that location is download-only for practical purposes. Preserve the media type, document and warn about the access limitation, and never invent a native-cloud alternate to make windowed reads appear available.

### Asset Roles

Roles are open and order is not semantically meaningful. Use known compact values when exact and preserve meaningful provider roles as custom strings.

Thumbnail discovery comes from an Asset's `thumbnail` role. Do not add a separate thumbnail dataset field.

## Links

Retain meaningful provenance, canonical, derived-from, alternate, license, described-by, service, documentation, and domain links. Resolve relative links against the Item URL. Item-JSON alternates remain Links, not Alternate Assets.

Drop source graph and transport links such as `root`, `parent`, `child`, `item`, `collection`, `next`, and `prev`. Do not copy source `self` as Tilebox `self`.

A source self URL may become `canonical` when it identifies the same record or `derived_from` when conversion creates a derived representation. Do not duplicate data files as Links when they already exist as Assets.

If a Link needs unsupported methods, headers, or bodies, stop rather than dropping request semantics.

## Cloud-Native Alternates

Keep the source location. Add a native cloud alternate only when provider, bucket/container, endpoint, and exact object-key bytes are recoverable without query or transformation state.

Recognized safe shapes include:

```text
https://storage.googleapis.com/{bucket}/{key}
  -> gs://{bucket}/{key}

https://{bucket}.storage.googleapis.com/{key}
  -> gs://{bucket}/{key}

https://{bucket}.s3.{region}.amazonaws.com/{key}
  -> s3://{bucket}/{key}

https://s3.{region}.amazonaws.com/{bucket}/{key}
  -> s3://{bucket}/{key}
```

Keep AWS region and bucket in Storage. R2 needs a custom-S3 scheme containing the account endpoint. Keep Azure Blob HTTPS unless configuration proves ADLS DFS/ABFS availability.

Do not generate alternates from:

- signed, SAS, query-bearing, fragment-bearing, or version-selecting URLs;
- CDN, proxy, browser, static-website, download-API, or transformation URLs;
- S3 access points, Object Lambda, or websites;
- R2 public/custom domains; or
- ambiguous percent-encoding.

A native URI identifies an object; it does not prove access. Credentials, requester-pays, anonymous mode, and custom endpoints remain separate concerns.

## Source-Specific Normalization

Keep provider repairs in named source/version recipes and fixtures. Never:

- derive an Asset base from `providers[].url`;
- lowercase an Asset title to create a key;
- reconstruct a missing href because a sibling looks similar;
- coerce a malformed singleton or number globally; or
- discard a contradictory value to make validation pass.

Routine deterministic normalization needs no per-record warning. If a value is ambiguous or changes beyond the documented condition, fail with the source record ID and return to discovery.

## Unsupported Metadata

An unsupported stable Item property may become a generated typed field. Unsupported nested metadata cannot be flattened without losing association. Stop and choose one of:

- extend the well-known protobuf;
- reject affected records;
- split the product into a different approved representation; or
- explicitly omit the value after the user accepts the information loss.

Do not use arbitrary JSON as a discovery fallback. It weakens typing and hides schema disagreements.
