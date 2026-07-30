# Landsat 8/9 Collection 2 Product Semantics

## Products And Bands

Collection 2 Level-2 Surface Reflectance commonly includes SR_B1 coastal aerosol, SR_B2 blue, SR_B3 green, SR_B4 red, SR_B5 NIR, SR_B6 SWIR1, SR_B7 SWIR2, QA_PIXEL, QA_RADSAT, and SR_QA_AEROSOL. Surface Temperature products include ST_B10, ST_QA, ST_CDIST, and atmospheric/emissivity/radiance intermediate layers. Product correction labels distinguish reflectance-only from products containing reflectance and temperature; inspect metadata.

## Grid And Scale

Products are usually 30 m UTM or Polar Stereographic grids. Avoid reprojection when selected scenes/assets align. Across WRS path/row, projection zones, or sensors, choose a target grid early; compute natively when possible and reserve delivery reprojection for the end.

Surface Reflectance uses fill `0`, valid DN `1..65455`, and:

```text
reflectance = DN * 0.0000275 - 0.2
```

Surface Temperature uses fill `0` and:

```text
temperature_kelvin = DN * 0.00341802 + 149.0
```

Do not interchange these calibrations. Preserve integer storage when useful.

## Quality Masks

Common QA_PIXEL bits are fill (0), dilated cloud (1), high-confidence cirrus (2), high-confidence cloud (3), high-confidence shadow (4), high-confidence snow (5), clear (6), and water (7):

```python
invalid = ((qa & (1 << 0)) != 0) | ((qa & (1 << 1)) != 0)
invalid |= ((qa & (1 << 2)) != 0) | ((qa & (1 << 3)) != 0)
invalid |= ((qa & (1 << 4)) != 0) | ((qa & (1 << 5)) != 0)
valid = ~invalid
```

Adapt water/snow handling to the application and check QA_RADSAT where saturation matters. The clear bit alone is not a substitute for deliberately handling relevant flags. Keep QA integer and use nearest/mode resampling.

## Verification And References

Verify product level, sensor/platform, correction, grid, fill/range, scaling, and every QA assumption against current product metadata. Inspect clouds, shadows, snow/water, saturated pixels, edges, and no-data in representative outputs. Preserve WRS path/row, acquisition/provenance, calibration, masks, and target grid.

Authoritative references are the USGS Landsat Collection 2 Level-2 Science Products, Collection 2 Quality Assessment Bands, Landsat 8–9 Collection 2 Level-2 Science Product Guide, and Landsat Cloud Optimized GeoTIFF Data Format Control Book.
