# Datasets And Datapoints

Source recommendation, exact dataset slugs, collection selection, provider credentials, and auxiliary catalogues belong to the companion `tilebox-datasets` skill. After selection, inspect the live schema and samples before coding.

```python
from tilebox.datasets import Client
from tilebox.datasets.query import TimeInterval

dataset = Client().dataset(dataset_slug)
data = dataset.query(
    collections=[collection],
    temporal_extent=TimeInterval(start=start, end=end),
    spatial_extent=aoi,
    show_progress=True,
)
```

Use Shapely geometry for `spatial_extent`. Name collections explicitly; omitted collections query all. Use `skip_data=True` only for metadata probes because it omits fields needed later. Inspect schema and a small result to confirm fields, canonical assets, scale/offset, nodata, and QA semantics.

Push categorical selection into the query when the live schema marks scalar or repeated enums queryable. Use exact public strings and compose one `filter=` expression:

```python
from tilebox.datasets import field

selected = dataset.query(
    temporal_extent=(start, end),
    filter=(field("product_type") == "L2")
    & field("use_cases").contains_any(["agriculture", "flood-monitoring"]),
)
```

This example requires those fields and values in the inspected schema; do not assume existing open-data datasets have migrated from legacy enum/string types. Repeated enums also support `.contains(value)` and `.contains_all(values)`. Missing/empty lists are empty sets, so negated membership includes them; missing scalars are excluded by inequality unless a null branch is added. Python returns repeated enums as one-dimensional object columns with a `list[str]` per row, not a padded extra dimension. The `tilebox-datasets` skill owns the full enum schema, query, and evolution contract.

Give each query one owner: query/select once and cache the required assets and metadata for downstream tasks, or partition the time range into non-overlapping intervals and query directly in the leaf tasks. Avoid querying datapoints in the parent just to pass IDs that children immediately query again. Use parent selection when grouping or ranking needs the full candidate set.

Asset decoding requires exactly one datapoint:

```python
from tilebox.datasets import iter_datapoints

for datapoint in iter_datapoints(data):
    process(datapoint)

one = data.isel(time=0)
```

Do not pass a multi-time dataset to `AssetCollection.from_datapoint`. Do not reconstruct provider paths from IDs when canonical assets are present.
