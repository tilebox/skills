# Non-STAC And File-Tree Sources

Convert provider XML, non-STAC JSON APIs, object-store prefixes, and file trees by first constructing the same semantic record that a canonical STAC 1.1 Item would express. A source does not need to publish STAC, but it must provide authoritative evidence for each inferred value.

The adapter may use internal typed models, but it must establish the same identity, time, geometry, properties, Assets, Links, and access metadata as the canonical STAC model. It need not serialize intermediate STAC JSON.

## Evidence Order

Use evidence in this order:

1. provider product specification or machine-readable schema;
2. explicit source metadata and sidecars;
3. embedded file-format metadata;
4. stable, documented filename and directory conventions;
5. object metadata supplied by the provider; and
6. user-approved project configuration.

Do not infer scientific semantics from a filename or folder merely because the pattern looks familiar. File modification time is not acquisition time. An object owner, endpoint, or bucket name is not a platform, mission, or license. A `.tif` suffix alone does not prove a Cloud Optimized GeoTIFF.

## Provider XML Or Sidecar Metadata

Identify and validate the sidecar schema/version, namespaces, units, and required elements. Map values to canonical paths while retaining raw XML paths as recipe evidence. Resolve relative files against the verified product root. Preserve useful XML as a metadata Asset or provenance Link.

Stop when namespace/version changes alter element meaning, coordinates lack a known CRS/axis order, timestamps lack an interpretable timezone, units are unclear, or a file inventory cannot be associated with the correct product.

## Non-STAC JSON And Provider APIs

Record API version, authentication, revision/deletion behavior, and stable pagination/checkpoints. Build semantics from authoritative detail records rather than search summaries. Provider pagination/navigation controls ingestion and does not become datapoint Links; retain only meaningful documentation, provenance, license, or service links.

Do not use a list endpoint's transient cursor, rank, or URL as product identity. Do not infer a complete record from a search summary when a detail endpoint exists.

## Object-Store Prefixes And File Trees

Define the natural record boundary first: for example one authoritative manifest or provider-documented acquisition directory. Files belonging to one acquisition normally become Assets on one datapoint. Derive identity, time, roles, and Band meaning from metadata/product evidence, never modification timestamps or familiar filenames alone.

### List Robustly

Paginate bounded prefixes, preserve exact key bytes/case, ignore ordering, and group by one documented deterministic rule. Decide how versions, delete markers, temporary files, and incomplete uploads behave; reject orphan files and conflicting manifests.

Do not scan an unbounded bucket during tests. Use bounded prefixes and recorded fixtures.

### Inspect Raster Assets

Inspect format, CRS/axis order, transform/bounds/shape, Band count/order, data values, masks, embedded metadata, and consistency with sidecars. Validate block layout/overviews before claiming COG.

Use a COG validator or authoritative library check before assigning the COG media type. A valid GeoTIFF that is not cloud optimized uses the GeoTIFF media type instead.

Embedded bounds can supply a footprint only under an approved rule. Transform raster corners to WGS84 with rotation and axis order handled; prefer authoritative valid-data geometry when available.

### Construct Asset Keys And Roles

Use provider-defined layer identifiers or a documented collision-safe key mapping. Assign roles from product semantics, not file extensions; preserve meaningful custom roles.

### Construct Band Metadata

Use specifications, sidecars, and embedded descriptions to determine Band values. Verify logical order against physical container order.

Do not invent EO common names for hyperspectral bands or infer wavelengths from display colors. Do not treat subdatasets inside HDF5, NetCDF, or Zarr as reconstructible from a Band name unless the format/product contract provides the internal path.

### Construct Locations, Storage, And Authentication

Keep href, Storage, and Authentication separate.

For S3, GCS, Azure, R2, and other object stores, use exact provider configuration and object keys. Generate an HTTPS/native-cloud alternate only when translation is lossless and the service endpoint is known. Do not convert signed URLs, CDN links, download APIs, versioned URLs, transformed URLs, or ambiguous percent-encoding.

Pass absolute hrefs and applicable schemes to the SDK. Resolving a relative path is not an object request; Python's `AssetCollection` owns generated field/profile construction. Record evidence for every handcrafted value in the recipe and leave optional values absent when evidence is insufficient.

## Source Validation Assertions

Fail records with ambiguous manifests/files, missing required members, invalid time/geometry, unexpected grid/Band order, unresolved hrefs, mismatched media types, or missing registry references. Unknown files must follow an approved rule or be reported.

Log product identity and source location with failures, but never log credentials or signed query strings.

## Abort Conditions

Stop when grouping is undocumented; identity/time depends on mutable object metadata; geometry lacks a known CRS; sidecars disagree; uploads are incomplete; filenames cannot establish semantics; hrefs are unresolved; or access behavior cannot be represented without loss.

Recommend the smallest next step: obtain the provider schema, catalog the upstream acquisitions instead, add a manifest, define a project configuration, extend the relevant protobuf, or use the analytical cube directly.
