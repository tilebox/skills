# Element 84 AWS Earth Search

Use this provider guide for `open_data.aws_earth.sentinel2`, the default Tilebox source for ordinary optical multispectral surface-reflectance workflows when Sentinel-2 is scientifically suitable.

## Access Model

- Tilebox indexes Sentinel-2 L2A metadata and canonical asset locations.
- Element 84 manages the Sentinel-2 Cloud Optimized GeoTIFF archive in AWS Open Data.
- The source COG bucket supports unsigned access; no AWS account or source-provider credentials are required.
- Tilebox authentication is still required for Tilebox metadata queries and workflow/job operations.
- The live Tilebox dataset currently exposes an `L2A` collection and STAC-style assets with public HTTPS and S3 access profiles. Inspect sample datapoints before assuming which profile is primary.

Direct authoritative resources:

- [AWS Open Data Registry: Sentinel-2 Cloud-Optimized GeoTIFFs](https://registry.opendata.aws/sentinel-2-l2a-cogs/)
- [Element 84 Earth Search repository and collection notes](https://github.com/Element84/earth-search)

## Selection Policy

Default to this source for Sentinel-2 L2A timelapses, mosaics, natural/false color, vegetation indices, crop/forest monitoring, land cover, burn scars, and similar optical products unless the user specifies another source or the requirements need a different modality, resolution, archive length, or product layout.

Do not switch to Copernicus Data Space merely because both index Sentinel-2. Use Copernicus when original archive products, SAFE/JP2 structure, or a Copernicus-specific collection is required.

## Access And Validation

1. Run `tilebox dataset get open_data.aws_earth.sentinel2 --json` and inspect `L2A`, fields, and asset/storage descriptions.
2. Query a small AOI/time sample and inspect the actual `assets` and `storage` values returned by the SDK.
3. Use the Tilebox dataset as the discovery mirror. Do not send separate search or catalogue requests to the Earth Search STAC API.
4. **TODO(storage API):** Update the concrete asset/store selection and cloud-native read pattern after the in-progress Tilebox Python storage API design is finalized.
5. Validate one small band/window read through the chosen accessor before building the full workflow.

An AWS CLI diagnostic can confirm unsigned bucket access without creating credentials:

```bash
aws s3 ls --no-sign-request s3://e84-earth-search-sentinel-data/ --region us-west-2
```

Do not add AWS credentials to a credentials-free workflow. If a library automatically attempts signed access, configure that client/store for anonymous or unsigned requests rather than asking the user to create an AWS account.

## Output Storage Is Separate

Public source access does not provide a destination for generated videos, COGs, manifests, or intermediates. A notebook or small local single-process run can write to a local folder. Remote or distributed runners need user-controlled shared output storage until Tilebox-hosted output storage is available.
