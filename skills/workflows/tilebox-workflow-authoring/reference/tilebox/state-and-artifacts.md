# State And Artifacts

Choose boundaries by size, lifetime, and sharing:

- Task fields: compact plain-Python or supported library values, IDs, keys, and small configuration; serialized total must be at most 2048 bytes.
- `context.job_cache`: compact job-scoped bytes such as metadata and reduction summaries.
- Object storage/Zarr: large arrays, rasters, manifests, and cross-task rendezvous.
- Local files: task-local scratch or explicitly runner-local cache only.

Pass keys rather than arrays, pandas/xarray objects, large geometry, manifests, credentials, clients, open files, or local paths. Serialization support for a value type does not change this boundary. Use deterministic keys derived from job/stage/source/chunk. Retried tasks should overwrite safely, check an existing valid output, or atomically commit.

For Zarr rendezvous, create arrays once, then let workers write deterministic non-overlapping regions. A later stage may read with a different fanout axis. See the generic [Zarr mechanics](../geospatial/io/zarr.md).

Large immutable runtime artifacts such as model weights belong in accessible private storage or a deterministic runner-local cache. Lazy-load, verify checksum/size, and keep correctness independent of a warm cache.

Public input access and output storage are separate. Local output is fine for a small local run; distributed or remote work that shares artifacts requires durable shared storage. A Tilebox API key does not imply an output bucket.
