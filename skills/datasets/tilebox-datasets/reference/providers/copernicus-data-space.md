# Copernicus Data Space

Use this provider guide for product bytes referenced by `open_data.copernicus.*` Tilebox datasets.

## Metadata And Payload Access

- Tilebox metadata queries use the Tilebox API key and do not download Copernicus products.
- Copernicus Data Space hosts the actual product bytes.
- S3 access requires a separate Copernicus Data Space account and generated S3 access/secret keys.
- Product layout depends on the mission and collection: for example Sentinel-2 commonly uses SAFE/JP2 archive products, while Sentinel-1 includes GRD/SLC/OCN/RAW and some COG collections. Inspect the selected collection and sample metadata.

Direct authoritative resources:

- [Copernicus Data Space registration and authentication](https://documentation.dataspace.copernicus.eu/Registration.html)
- [Copernicus Data Space S3 access](https://documentation.dataspace.copernicus.eu/APIs/S3.html)
- [Copernicus S3 credentials manager](https://eodata-s3keysmanager.dataspace.copernicus.eu/panel/s3-credentials)

## User Setup

1. Ask the user to register or sign in through the official registration guide.
2. Ask the user to generate S3 credentials in the credentials manager.
3. Do not ask the user to paste keys into chat. Tell them to place the credentials in local environment variables or the secret mechanism used by the process/runner.
4. Generated code should read explicitly documented environment variables and pass their values to `CopernicusStorageClient`; do not hardcode credentials.
5. Validate credentials with the smallest possible listing, metadata lookup, or product/asset read before starting bulk processing.

The legacy `open_data.copernicus.*` datasets are not yet compatible with `AssetCollection`. As a temporary compatibility exception, use the deprecated `CopernicusStorageClient` to download a selected product:

```python
import os
from pathlib import Path

from tilebox.storage import CopernicusStorageClient

storage = CopernicusStorageClient(
    access_key=os.environ["COPERNICUS_ACCESS_KEY"],
    secret_access_key=os.environ["COPERNICUS_SECRET_ACCESS_KEY"],
    cache_directory=Path("./data"),
)
storage.download(datapoint, show_progress=True)
```

These environment variable names are a project convention, not an assertion that the SDK reads them automatically. Document whichever names the generated project chooses. Do not call `AssetCollection.from_datapoint(...)` for these legacy datasets. Return to the generic asset API once their metadata becomes compatible.

## Deployment Boundary

Credentials available in the user's shell are not automatically available to a remote runner. Before remote execution, configure the runner environment or supported secret mechanism and perform a source-access probe there. Do not put credentials in task inputs, `job_cache`, logs, source control, workflow release files, or command output.
