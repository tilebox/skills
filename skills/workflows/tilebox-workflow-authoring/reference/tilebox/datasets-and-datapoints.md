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

Asset decoding requires exactly one datapoint:

```python
from tilebox.datasets import iter_datapoints

for datapoint in iter_datapoints(data):
    process(datapoint)

one = data.isel(time=0)
```

Do not pass a multi-time dataset to `AssetCollection.from_datapoint`. Do not reconstruct provider paths from IDs when canonical assets are present.
