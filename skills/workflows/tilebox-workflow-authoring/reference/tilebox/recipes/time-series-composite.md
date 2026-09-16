# Time-Series Composite Recipe

Follow the portable [selection, alignment, compositing, and styling rules](../../geospatial/processing/time-series.md).

## Task Graph And Artifacts

Use only the stages the product needs. When a bounded period mosaic fits one worker, fan out by period and read its selected scenes there; add scene intermediates and reducers only when scale or reuse requires them.

1. A root task accepts AOI/time/output configuration, queries metadata, applies selection/grouping, and caches shared metadata once. It adds progress totals and submits workers only for the selected scenes or periods.
2. Each worker reads only its bounded AOI, assesses AOI-level quality, masks/calibrates, aligns to the fixed grid, writes a deterministic scene/period key when a rendezvous is required, and completes one unit of the parent's progress indicator.
3. Keep an aligned observation in memory when source and output fanout match. If compositing changes the axis (for example scene workers followed by spatial output chunks), initialize one stable Zarr schema and write deterministic non-overlapping regions.
4. If reducers are needed, submit the reducer batch with one shared dependency on all source-worker handles, not a separate dependency group per scene/chunk. Use an individual producer dependency only for a single downstream consumer, not repeated paired stages. Reducers read valid aligned observations, write deterministic frame/chunk keys, and report a separate `composites` indicator.
5. A final task depends on every required frame/chunk, checks ordering/dimensions/completeness, assembles periodic COGs or encodes the requested visual artifact, and publishes metadata.

Derive keys from immutable inputs such as grid ID, period, scene ID, and processing version; retries must overwrite/reuse the same region. Do not pass frames or manifests as task fields. Store source timestamps/IDs, style, grid, and missing-period decisions in artifact metadata or logs. Create a separate manifest only when the user asks.

Declare image/video encoders in `pyproject.toml`; never assume a system codec. Use a fixed frame size/pixel format and explicitly represent acquisition timing. Interactive applications are separate consumers of published COG/Zarr data—workflow tasks never emit app source or bundles.
