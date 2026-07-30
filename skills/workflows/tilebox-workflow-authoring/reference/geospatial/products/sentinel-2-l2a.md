# Sentinel-2 L2A Product Semantics

## Products, Bands, And Grids

L2A is surface reflectance with a Scene Classification Layer (SCL); L1C is top-of-atmosphere reflectance. Archives may expose COG assets or JP2 files in SAFE-like products. Common L2A bands B02 (blue), B03 (green), B04 (red), and B08 (NIR) are 10 m; B05, B06, B07 (red edge), B8A (narrow NIR), B11 and B12 (SWIR), and SCL are commonly 20 m. Some bands use other resolutions. Inspect actual metadata rather than assuming all assets share a grid.

## Scale And Classification

Reflectance is commonly stored as scaled integers. Apply each band's declared scale/offset, preserve integer storage when useful, and do not generalize nodata or scaling to another source. Never stack different grids without deliberate alignment.

SCL is categorical. Common classes distinguish no-data/defective/dark pixels, cloud shadow, vegetation, bare soil, water, unclassified, medium/high cloud, cirrus, and snow/ice; exact definitions and handling must follow the product. Select positive valid classes for the application and reproject to the image grid with nearest-neighbor resampling. Snow, water, shadows, dark pixels, and unclassified pixels may be valid or invalid depending on the analysis.

## Reprojection, Masking, And Mosaics

Choose resampling by semantics: nearest for SCL/classes/QA; nearest conservatively preserves reflectance samples, while bilinear/cubic can suit deliberate visualization or analysis. Never apply SCL directly to a differently shaped band. Combine classification with source nodata/read masks.

Use scene cloud cover only as a prefilter; compute AOI-level validity from SCL for stringent selection. A common mosaic aligns observations, masks invalid pixels, and reduces valid observations by median/quantile or another documented rule. Preserve dates, selected classes, scale, processing baseline, target grid, and remaining coverage.
