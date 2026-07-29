# Alaska Satellite Facility

Use this provider guide for product bytes referenced by `open_data.asf.*` Tilebox datasets.

## Metadata And Payload Access

- Tilebox metadata queries require the Tilebox API key but not an ASF account.
- Alaska Satellite Facility hosts the actual SAR products.
- Downloads generally require NASA Earthdata Login authentication.
- The current Tilebox catalog includes `open_data.asf.ers_sar` with `ERS-1` and `ERS-2` collections. Discover other `open_data.asf.*` datasets live rather than inventing slugs.

Direct authoritative resources:

- [NASA Earthdata Login overview and registration](https://www.earthdata.nasa.gov/data/earthdata-login)
- [Earthdata Login registration](https://urs.earthdata.nasa.gov/users/new)
- [ASF programmatic authentication methods](https://docs.asf.alaska.edu/asf_search/basics/)

## User Setup

1. Ask the user to register for or sign in to NASA Earthdata Login.
2. Discover missions, product types, and acquisitions through Tilebox metadata, which mirrors the relevant ASF catalogue information. Do not make an additional ASF Vertex search request by default; it is unnecessary for normal selection and can be slow or unstable.
3. Request product bytes directly from ASF using the locations in the selected Tilebox datapoints.
4. Do not ask the user to paste credentials or tokens into chat. Ask them to configure credentials in the local process or runner environment.
5. `ASFStorageClient` currently accepts ASF/Earthdata user and password values explicitly; generated code should read them from documented environment variables rather than hardcoding them.
6. Validate authentication by downloading one small representative product or quicklook before bulk work.

Example application-level convention:

```python
import os
from pathlib import Path

from tilebox.storage import ASFStorageClient

storage = ASFStorageClient(
    user=os.environ["EARTHDATA_USERNAME"],
    password=os.environ["EARTHDATA_PASSWORD"],
    cache_directory=Path("./data"),
)
```

These environment variable names are a project convention. ASF tooling may also support Earthdata tokens or `.netrc`; use the authentication mechanism supported by the selected Tilebox storage-client version and runner environment.

## Dataset And Processing Fit

Do not choose an ASF dataset only because the target uses SAR. Verify mission, product type, acquisition period, mode, polarization, geometry, and processing requirements. `open_data.asf.ers_sar` is a historical ERS source, not a drop-in Sentinel-1 substitute.

## Deployment Boundary

Credentials configured on a notebook or local machine do not automatically reach remote runners. Configure the target runner environment using its approved secret mechanism and test one authenticated product read there. Never put credentials in task inputs, `job_cache`, source control, release artifacts, or logs.
