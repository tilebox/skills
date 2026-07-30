# Sentinel-1 SAR Recipe

Use `tilebox-datasets` to select and inspect the current source. Confirm product type, preprocessing, units, polarization, orbit, and pass direction from the portable [SAR product guidance](../../geospatial/products/sentinel-1-sar.md) before building the graph.

For pairwise change:

1. A root task queries compact identifiers for compatible pre-event/event acquisitions.
2. Submit one preprocessing task per acquisition only when the source is not already consistently analysis-ready.
3. Store aligned outputs only if chunked comparison needs a rendezvous.
4. Fan out change or inference by bounded spatial chunk with identical target grids.
5. Aggregate compact validation summaries and publish deterministic COG/vector outputs.

Use stage handles as barriers rather than pairwise chains. Acquisition workers preserve orbit, direction, polarization, incidence geometry, units, and preprocessing metadata. Chunk tasks receive only product/object IDs, target-grid/chunk coordinates, and small algorithm parameters; deterministic keys include the pair/grid/chunk and processing version.

For ship detection, fan out overlapping detector tiles, convert detections to georeferenced geometries, and merge duplicates deterministically. For flood mapping, include terrain, permanent-water, coast/land, or shadow/layover context as required. Ask `tilebox-datasets` to select and inspect each auxiliary catalogue and establish access; this recipe receives only selected durable identifiers/keys and never embeds provider setup.

Keep product IDs, geometry metadata, chunk coordinates, and object keys in task inputs. Preserve algorithm/model version and limitations in outputs.
