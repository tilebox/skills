# Element 84 AWS Earth Search

Use this provider guide for the credentials-free `open_data.aws_earth.sentinel2` and `open_data.aws_earth.sentinel1` datasets. They are the default Tilebox sources for ordinary Sentinel-2 L2A optical and Sentinel-1 GRD amplitude/backscatter workflows when those products are scientifically suitable.

## Access Model

- Tilebox indexes Sentinel-2 L2A and Sentinel-1 Level-1 GRD metadata and canonical asset locations.
- Element 84 publishes both catalogs through Earth Search with public AWS product assets.
- The source buckets support unsigned access and are not requester-pays; no AWS account or source-provider credentials are required.
- Tilebox authentication is still required for Tilebox metadata queries and workflow/job operations.
- `open_data.aws_earth.sentinel2` exposes `L2A`. `open_data.aws_earth.sentinel1` exposes only `GRD`, with polarization COGs plus SAFE manifest and product/calibration/noise metadata. Both provide public S3 and HTTPS locations. Inspect sample datapoints before assuming asset keys or which profile is primary.

Direct authoritative resources:

- [AWS Open Data Registry: Sentinel-2 Cloud-Optimized GeoTIFFs](https://registry.opendata.aws/sentinel-2-l2a-cogs/)
- [AWS Open Data Registry: Sentinel-1](https://registry.opendata.aws/sentinel-1/)
- [Element 84 Earth Search repository and collection notes](https://github.com/Element84/earth-search)

## Selection Policy

Default to this source for Sentinel-2 L2A timelapses, mosaics, natural/false color, vegetation indices, crop/forest monitoring, land cover, burn scars, and similar optical products unless the user specifies another source or the requirements need a different modality, resolution, archive length, or product layout.

Default to `open_data.aws_earth.sentinel1` for Sentinel-1 GRD flood/change mapping, backscatter analysis, and maritime workflows unless the user specifies another source or needs another product. Prefer it over `open_data.copernicus.sentinel1_sar` whenever GRD is sufficient. Do not use it for interferometry, coherence, or deformation: those require compatible SLC phase products, which this dataset does not contain.

Do not switch to Copernicus Data Space merely because both providers index the same mission. Use Copernicus when original Sentinel-2 SAFE/JP2 products, Sentinel-1 SLC/OCN/RAW, or another Copernicus-specific collection or layout is required.

## Access And Validation

1. Run `tilebox dataset get <slug> --json` and inspect the expected `L2A` or `GRD` collection, fields, and asset/storage descriptions.
2. Query a small AOI/time sample and inspect the actual `assets` and `storage` values returned by the SDK.
3. Use the Tilebox dataset as the discovery mirror. Do not send separate search or catalogue requests to the Earth Search STAC API.
4. Convert one selected datapoint with `AssetCollection.from_datapoint(...)`, then pass the required asset to `tilebox.storage.aio`. The resolver selects a compatible S3 or HTTPS location and configures unsigned access automatically when the location has no authentication requirement.
5. Validate one small band/window read through the chosen accessor before building the full workflow.

With `tilebox-storage` installed, read COGs directly through the asset API. Given one selected datapoint and AOI bounds `(west, south, east, north)`:

```python
from tilebox.datasets.assets import AssetCollection
from tilebox.storage.aio import open_geotiff
from tilebox.storage.geotiff import window_from_bounds

assets = AssetCollection.from_datapoint(datapoint)
geotiff = await open_geotiff(assets["red"])
window = window_from_bounds(
    geotiff, bounds, crs="EPSG:4326", require_fully_contained=True
)
tile = await geotiff.read(window=window)
```

`AssetCollection.from_datapoint(...)` accepts exactly one datapoint; use `data.isel(time=index)` or `tilebox.datasets.iter_datapoints(data)` after a multi-result query. Inspect `list(assets)` rather than assuming every dataset uses the same keys. `open_geotiff(...)` opens metadata but does not read pixels or apply scale, offset, nodata, masks, or reprojection.

An AWS CLI diagnostic can confirm unsigned bucket access without creating credentials:

```bash
aws s3 ls --no-sign-request s3://e84-earth-search-sentinel-data/ --region us-west-2
```

Do not add AWS credentials to this credentials-free workflow or construct a signed S3 client manually. The asset resolver sees that the public location has no authentication reference and creates an unsigned store.

## Output Storage Is Separate

Public source access does not provide a destination for generated videos, COGs, manifests, or intermediates. A notebook or small local single-process run can write to a local folder. Remote or distributed runners need user-controlled shared output storage until Tilebox-hosted output storage is available.
