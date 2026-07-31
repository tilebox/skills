# Asset Profile Wire Format

This is an advanced Go-only interoperability reference while Go lacks a high-level semantic Assets builder. Python converters must use `AssetCollection`, must not read this reference for ordinary conversion, and must not construct these structures.

The compact profile tables are a public protobuf representation, but profile optimization is not source-normalization logic. Keep direct construction isolated and test semantic reconstruction.

## Three Separate Layers

```text
Authentication.schemes <--- auth_refs -------+
                                             |
Storage.schemes        <--- storage_refs ----+--- AssetAccessProfile
                                                    |
                                                    | access_profile_index
                                                    v
                                             AssetLocation.href
```

- `AssetLocation.href` identifies a concrete resource after reconstruction.
- `StorageScheme` describes where/how the object service is hosted.
- `AuthenticationScheme` describes how permission is obtained.
- `AssetAccessProfile` deduplicates shared access metadata and an Item-local href prefix.

A profile represents a complete access mode, not only a protocol. Two HTTPS locations with different authentication or storage requirements need different profiles.

## Access Profiles

Each `AssetAccessProfile` contains:

- `alternate_key` for alternate locations such as `s3`;
- optional default alternate display name;
- `base_href` for exact byte-prefix compression;
- `storage_refs`; and
- `auth_refs`.

Every `AssetLocation.access_profile_index` must reference an existing profile. Index zero is valid and must retain explicit presence where required.

Reconstruct the full href by byte-concatenating `base_href` and the selected suffix. Do not run URI reference resolution, slash normalization, percent decoding, path cleaning, or case normalization during reconstruction.

Resolve source-relative hrefs before compression. Compression never gives a relative source href its base.

## Href Presence

Presence is semantic:

- a primary location must have a present href;
- an alternate with absent href reuses the primary location's suffix under its own profile;
- an alternate with present empty href identifies exactly its profile's `base_href`; and
- an alternate with present non-empty href supplies its own suffix or an absolute href when the base is empty.

Do not collapse absent and empty. Protobuf builders use pointers/explicit presence for these fields.

Example:

```text
profile 0:
  base_href = https://bucket.s3.us-west-2.amazonaws.com/path/item/
  storage_refs = [earth-search]

profile 1:
  alternate_key = s3
  default_alternate_name = S3
  base_href = s3://bucket/path/item/
  storage_refs = [earth-search]

asset red:
  primary = profile 0, href present "B04.tif"
  alternate = profile 1, href absent
```

The alternate reconstructs with the primary suffix as `s3://bucket/path/item/B04.tif`.

## Prefix Selection

Choose a common prefix only when reconstruction is byte-exact. Prefer URI path boundaries so suffixes remain recognizable and no asset crosses provider, bucket/container, endpoint, authentication, or access-mode boundaries.

Use an empty base and complete href when no safe useful prefix exists. Compression is optional; semantic correctness is mandatory.

Build profiles deterministically, for example by a stable semantic tuple of:

```text
(alternate key, default alternate name presence/value, base href, storage refs, auth refs)
```

Preserve approved registry-ref ordering or sort it deterministically when order is semantically irrelevant. Reject conflicting definitions for the same registry key.

Validate that alternate keys materialized for one Asset are unique. Two locations cannot export to the same Alternate Assets map key.

## Band Profiles

`Assets.band_profiles` contains reusable non-empty normalized Band messages. Each `Asset.band_profile_indices` list preserves that Asset's source Band order and may reference the same profile more than once.

Before interning in Go:

1. Resolve effective inheritable values under STAC rules.
2. Lift a value to the Asset only when every Band has the same complete effective typed value, including presence.
3. Keep name, description, and spectral EO identity on Bands unless the canonical extension inheritance explicitly establishes an Asset-level value.
4. Omit fields from Bands only when the Asset supplies the same inherited value.
5. Reject a Band that becomes completely empty when omission would lose meaningful band cardinality.
6. Compare complete protobuf values using semantic protobuf equality, including explicit presence.
7. Reuse a profile only for exact equality; any difference creates another profile.

References have no overrides. Never add a project-local delta layer around a profile reference.

## Deterministic Band Ordering

Profile table order is independent of each Asset's Band order. Choose and document one domain-natural deterministic order, then remap all indices.

Useful semantic keys include:

- ascending center wavelength for spectral products;
- provider-defined natural Band order;
- stable Band name with deterministic tie-breakers; or
- a canonical protobuf representation when no domain order exists.

Do not depend on Go map iteration. After sorting profiles, remap every old index while preserving the sequence of indices on each Asset.

For Sentinel-2, a wavelength-based order naturally yields B01, B02, B03, B04, B05, B06, B07, B08, B8A, B09, B11, and B12 while visual Assets can still preserve their red-green-blue Band order through references.

## Registries

Every `storage_ref` and `auth_ref` must resolve to a key in the datapoint's corresponding registry. Merge definitions by exact key and semantic equality. The same key with different definitions is an error.

Retain registry entries used only by Links or other fields by merging them into the final Storage/Authentication messages even when no Asset location references them.

Do not add Storage or Authentication schemes that the source or approved project configuration does not establish. Do not place credentials, tokens, signed query values, or secrets in registry messages.

## Known And Custom Values

Use known enums only for exact values represented by the current API. Preserve open media types, roles, Storage types, and Authentication types through their custom string paths when supported.

Unknown values from a closed vocabulary are errors. Do not assign `UNSPECIFIED` or an approximate enum merely to pass validation.

## Validation Invariants

Before placing compiled Assets on a datapoint, verify:

- every Asset key is non-empty and unique;
- every Asset has a primary location with present href;
- every location references an existing access profile;
- every Asset Band index references an existing non-empty profile;
- no duplicate alternate key materializes for one Asset;
- every registry ref resolves;
- exact href reconstruction succeeds for primary and alternate locations;
- explicit empty, absent, zero, false, and enum values retain intended presence;
- Band effective values match the canonical semantic model;
- output is deterministic for equivalent input; and
- protobuf validation succeeds.

Golden tests should reconstruct canonical semantic Assets and compare those values. Do not treat a particular compression ratio or profile index as the core correctness criterion.
