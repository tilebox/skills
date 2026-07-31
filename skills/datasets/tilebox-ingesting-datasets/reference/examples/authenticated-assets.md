# Authenticated Asset Edge Case

Keep this separate because authenticated locations require conditional policy even though only one provider example is currently verified.

## Universal Policy

Use the STAC Authentication extension whenever it can represent the source. Preserve `auth:schemes` and each location's exact `auth:refs` in Tilebox Authentication. Storage describes the service; Authentication describes how access is granted; neither identifies the resource itself.

If the STAC extension or current Tilebox message cannot represent a required flow without loss, warn the user, explain the missing semantics, and ask how to proceed. Never invent a custom convention or persist credentials/signed URLs.

## Copernicus Data Space Layouts

Evidence: [STAC API documentation](https://documentation.dataspace.copernicus.eu/APIs/STAC.html) and [representative Sentinel-2 L2A Item](https://stac.dataspace.copernicus.eu/v1/collections/sentinel-2-l2a/items/S2B_MSIL2A_20260731T064629_N0512_R020_T47XNJ_20260731T084722)

The STAC 1.1 Item declares Authentication Extension 1.1 with:

- `s3`: type `s3`
- `oidc`: type `openIdConnect`, using the CDSE realm discovery URL

It demonstrates three distinct source layouts:

| Asset kind | Primary | Alternate |
| --- | --- | --- |
| most JP2/metadata files | `s3://eodata/...`, `auth:refs=[s3]`, Storage refs `cdse-s3` and `creodias-s3` | explicit OData `Nodes(...)/$value` HTTPS, `auth:refs=[oidc]` |
| Product archive | explicit OData `Products(...)/$value` HTTPS, `auth:refs=[oidc]` | none |
| thumbnail | explicit CREODIAS OData HTTPS without auth ref | `s3://eodata/...`, `auth:refs=[s3]`, with both Storage refs |

Preserve primary/alternate order per Asset and model each concrete location independently. Authentication and Storage references are independent and must both resolve where present.

OData `Nodes(...)` and `Products(...)` URLs cannot be generated from S3 prefixes. Preserve only explicit source hrefs unless provider documentation proves a lossless mapping. Stop and ask if exact locations cannot be associated with their access schemes or a required provider flow cannot be represented.
