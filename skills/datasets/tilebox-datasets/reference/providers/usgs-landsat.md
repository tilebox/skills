# USGS Landsat On AWS

Use this provider guide for product bytes referenced by `open_data.usgs.*`, especially Landsat Collection 2.

## Access Model

- Tilebox indexes Landsat metadata and asset locations.
- USGS Landsat Collection 2 Level-1, Level-2, and Level-3 products are stored in the AWS `usgs-landsat` S3 bucket in `us-west-2`.
- The bucket is requester-pays. An AWS account and credentials are required, and requests or data transfer may create charges.
- `USGSLandsatStorageClient` uses the configured AWS credential chain; do not hardcode AWS access keys.

Direct authoritative resources:

- [Create an AWS account](https://aws.amazon.com/resources/create-account/)
- [USGS Landsat Commercial Cloud Data Access](https://www.usgs.gov/landsat-missions/landsat-commercial-cloud-data-access)
- [USGS Landsat requester-pays introduction](https://www.usgs.gov/software/introduction-landsat-cloud-access-direct-requester-pays)
- [AWS requester-pays documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/RequesterPaysBuckets.html)
- [AWS CLI credential and configuration files](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

## User Setup

1. Explain requester-pays before access and obtain the user's agreement to possible AWS charges.
2. If needed, ask the user to create or select an AWS account and configure credentials using an approved AWS mechanism such as a named profile, environment credentials, or workload identity.
3. Do not ask the user to paste AWS secret values into chat.
4. Configure `us-west-2` and requester-pays behavior in the chosen client/tool.
5. Validate access with a small listing or object read before bulk processing.

Diagnostic shape:

```bash
aws s3 ls s3://usgs-landsat/collection02/ \
  --region us-west-2 \
  --request-payer requester
```

Use `open_data.usgs.landsat8_oli_tirs` or `open_data.usgs.landsat9_oli_tirs` with `L2_SR` for modern surface-reflectance workflows and `L2_ST` for appropriate surface-temperature work. Inspect the exact live collection and fields before coding.

## When To Recommend Landsat

Recommend Landsat when its historical record, 30 m continuity, thermal products, or user requirement justifies requester-pays setup. For a simple recent optical timelapse or vegetation product where Sentinel-2 is suitable, prefer credentials-free `open_data.aws_earth.sentinel2` and present Landsat as an optional extension.

## Deployment Boundary

Local AWS credentials do not automatically reach remote runners. Use the target environment's approved identity/secret mechanism and validate requester-pays access there. Never place AWS credentials in task inputs, `job_cache`, source control, release artifacts, or logs.
