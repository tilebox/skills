# Python Converter

Use tilebox-python's semantic Assets API as the default converter implementation. The converter authors complete semantic assets with absolute location paths/URIs and metadata; `AssetCollection` validates them and produces the `Assets`, `Storage`, and `Authentication` fields.

## Check API Availability

The semantic authoring API is currently present in the WIP tilebox-python source under `tilebox.datasets.assets`. Before implementation, verify the project dependency exposes:

```python
from tilebox.datasets.assets import Asset, AssetCollection, AssetLocation, Band
```

If a released dependency does not expose these types yet, use the approved WIP/pinned tilebox-python version for the ingestion project or wait for the release. Do not work around the missing API by copying SDK internals into the converter.

## Author Semantic Assets

Use:

- `AssetLocation`: an absolute href plus applicable Storage and Authentication schemes;
- `Band`: semantic common, EO, Raster, Classification, and SAR metadata;
- `Asset`: key, primary/alternate locations, media type, roles, Bands, and Asset-scoped metadata; and
- `AssetCollection`: one Item's validated mapping of asset keys to semantic Assets.

Construct complete semantic values from the normalized source:

```python
from tilebox.datasets.assets import Asset, AssetCollection, AssetLocation, Band

asset = Asset(
    key=asset_key,
    primary=AssetLocation(
        href=primary_href,
        storage_schemes=storage_schemes,
        authentication_schemes=authentication_schemes,
    ),
    alternates={
        "s3": AssetLocation(
            href=s3_href,
            alternate_name="S3",
            storage_schemes=storage_schemes,
        )
    },
    media_type=media_type,
    roles=roles,
    bands=(
        Band(
            name=band_name,
            data_type=data_type,
            nodata=nodata,
            eo=eo_properties,
            raster=raster_properties,
        ),
    ),
    projection=projection,
)
```

Use current generated protobuf constructors for typed EO, Raster, Projection, SAR, Classification, Storage, and Authentication values. Making an href absolute is path normalization, not a network request.

## Compile Into Dataset Fields

Compile all Assets belonging to one datapoint together:

```python
asset_collection = AssetCollection.from_assets(semantic_assets)
asset_fields = asset_collection.to_fields()

point = {
    "time": acquisition_time,
    "geometry": geometry,
    "stac_id": source_id,
    "cloud_cover": cloud_cover,
    "links": links_message,
    **asset_fields,
}
```

`to_fields()` returns `assets` and includes non-empty `storage` and `authentication` registries when locations reference schemes. The output is a flat field mapping containing generated protobuf-py messages.

If Links or another field references schemes not used by Assets, pass those registries to `to_fields()`:

```python
asset_fields = asset_collection.to_fields(
    storage=link_only_storage,
    authentication=link_only_authentication,
)
```

Prefer canonical `assets`, `storage`, and `authentication` field names. For an approved nonstandard schema, pass a `fields` mapping to both `to_fields()` and `from_datapoint()`.

## What AssetCollection Owns

`AssetCollection.from_assets(...).to_fields(...)` owns:

- validation of asset keys, locations, roles, media types, registries, and Bands;
- known/custom role and media-type representation;
- STAC Asset-to-Band inheritance while preserving effective values;
- omission of Band metadata where no semantic Band exists;
- combination and validation of Storage and Authentication definitions; and
- deterministic construction of the dataset fields.

The converter must not recreate generated Asset/registry messages, compute href prefixes, move fields for storage optimization, depend on internal ordering, or copy private SDK functions.

The converter still owns source normalization. It must merge valid legacy `eo:bands` and `raster:bands` into semantic Bands, convert relative href paths to absolute hrefs, choose roles and alternates, and construct typed metadata before calling the compiler. Do not request every Asset while normalizing its href.

## Keep The Datapoint Flat

Python ingestion accepts a flat mapping from dataset field names to scalar values, geometries, or generated protobuf messages:

Correct:

```python
point = {
    "time": acquisition_time,
    "geometry": geometry,
    "assets": assets_message,
    "storage": storage_message,
    "providers": provider_messages,
    "cloud_cover": 12.5,
}
```

Do not add a nested `properties` mapping or represent message fields as dictionaries. Repeated message fields are lists of generated message instances.

## Ingest By Collection

Compute the approved collection route before ingestion and group points accordingly. Use the collection's SDK ingestion API:

```python
ids = collection.ingest(
    points_for_this_collection,
    allow_existing=True,
    show_progress=True,
)
```

Use `allow_existing=True` by default for ingestors so retries and restarts are idempotent. Given the exact same ingestion request payload, Tilebox derives the same datapoint ID and deduplicates it in the backend.

This default applies only to an identical payload. If a source record changes under the same source identity, follow the recipe's revision policy rather than treating the changed payload as a successful idempotent retry.

The SDK validates and serializes the generated dataset message.

## Resolve Queried Assets

Use the same semantic API for round-trip validation and consumers:

```python
from tilebox.datasets import iter_datapoints
from tilebox.datasets.assets import AssetCollection

for datapoint in iter_datapoints(data):
    resolved = AssetCollection.from_datapoint(datapoint)
    red = resolved["red"]
    assert red.primary.href.startswith("https://")
    assert red.alternates["s3"].href.startswith("s3://")
```

`from_datapoint()` accepts exactly one scalar xarray datapoint. Select one time index or use `iter_datapoints` for query results. It reconstructs semantic hrefs, Bands, Storage, and Authentication regardless of how the SDK encoded them.

## Python Test Boundaries

Test three layers separately:

1. source record to semantic `Asset`/`Band` values;
2. semantic collection through `to_fields()` and `from_datapoint()` back to semantic equality; and
3. full datapoint ingestion and query-back.

Assert semantic values rather than the SDK's internal encoding. That encoding may evolve while preserving semantics. Only an SDK-compatibility test should assert the binary representation.
