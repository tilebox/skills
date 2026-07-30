# Time-Series Composite Recipe

Follow the portable [selection, alignment, compositing, and styling rules](../../geospatial/processing/time-series.md).

## Task Graph And Artifacts

1. A root task accepts AOI/time/output configuration, queries compact scene identifiers, applies a coarse scene-quality prefilter, groups periods, calls `progress("scenes").add(n)`, and submits scene workers.
2. Each scene worker reads only its bounded AOI, assesses AOI-level quality, masks/calibrates, aligns to the fixed grid, writes a deterministic scene/period key when a rendezvous is required, then calls `progress("scenes").done(1)`.
3. Keep an aligned observation in memory when source and output fanout match. If compositing changes the axis (for example scene workers followed by spatial output chunks), initialize one stable Zarr schema and write deterministic non-overlapping regions.
4. A barrier task submits period/output-chunk reducers after scene handles succeed. Reducers read only valid aligned observations, write deterministic frame/chunk keys, and report a separate `composites` indicator.
5. A final task depends on every required frame/chunk, checks ordering/dimensions/completeness, assembles periodic COGs or encodes the requested visual artifact, and publishes metadata.

Derive keys from immutable inputs such as grid ID, period, scene ID, and processing version; retries must overwrite/reuse the same region. Do not pass frames or manifests as task fields. Store source timestamps/IDs, style, grid, and missing-period decisions in artifact metadata or an optional manifest only when requested or materially useful.

Declare image/video encoders in `pyproject.toml`; never assume a system codec. Use a fixed frame size/pixel format and explicitly represent acquisition timing. Interactive applications are separate consumers of published COG/Zarr data—workflow tasks never emit app source or bundles.
