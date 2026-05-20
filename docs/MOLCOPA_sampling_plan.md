# MOLCOPA 2024 Validation Sampling — Implementation Plan

## Overview

Build a single Jupyter notebook (`sample_design_molcopa2024.ipynb`) that:
1. Takes the output of `filter_samples_by_molca2024.ipynb` (the filtered 2019 points with MOLCOPA 2024 class values)
2. Computes Cochran sample size per class
3. Subsamples surplus classes with spatial spread across tiles
4. Generates new random points for deficit classes spread across tiles
5. Exports the final validation dataset

**Reusable**: change `REGION` to run for Africa, Amazon, or Siberia.

---

## Cell 1 — Configuration (only cell to edit between regions)

```python
REGION = "Africa"  # "Africa", "Amazon_Extension", "Siberia"

PROJECT_DIR    = r"d:\06_Polimi\2025-2026\Land Cover\Validation\sample-design"
INPUT_FILTERED = rf"{PROJECT_DIR}\Samples_2024\{REGION.lower()}_static_area_2024_molca_filtered.geojson"
MOLCA_GPKG     = rf"{PROJECT_DIR}\MOLCA\{REGION}_2024_UTM\molcx_{REGION.lower()}_2024_s2_exist.gpkg"
MOLCA_TIFFS    = rf"{PROJECT_DIR}\MOLCA\{REGION}_2024_UTM\Tiffs"
LEGEND_CSV     = rf"{PROJECT_DIR}\config\MOLCA_legend.csv"
OUTPUT_FINAL   = rf"{PROJECT_DIR}\Samples_2024\{REGION.lower()}_validation_2024_final.geojson"
OUTPUT_CSV     = rf"{PROJECT_DIR}\Samples_2024\{REGION.lower()}_validation_2024_final.csv"

NODATA_CLASS   = 255
RANDOM_SEED    = 42

# --- Cochran parameters (adjust as needed) ---
P_EXPECTED     = 0.90   # expected accuracy of MOLCOPA
E_ALLOWABLE    = 0.05   # 5% allowable error
Z_ALPHA        = 1.96   # 95% confidence
BUFFER_PCT     = 0.15   # 15% preventive buffer for photo-interpretation discards
```

**Notes:**
- `INPUT_FILTERED` is the output GeoJSON from `filter_samples_by_molca2024.ipynb`
- It must contain columns: `molca_class_2024`, `Tile_ID`, `geometry`
- If `Tile_ID` is not in the filtered output, it will need to be re-joined (see Cell 2)

---

## Cell 2 — Load inputs

Load:
1. **Filtered 2019 points** — the output of `filter_samples_by_molca2024.ipynb`
2. **MOLCA tile index** — the gpkg with tile geometries and tif paths
3. **Legend** — class code to label mapping

If the filtered points don't have `Tile_ID`, re-do the spatial join with the tile index to assign each point to its tile.

Re-use the same pattern from `filter_samples_by_molca2024.ipynb` cells 2-4:
- `gpd.read_file(INPUT_FILTERED)`
- `gpd.read_file(MOLCA_GPKG)`
- Build `tif_path` column from tile index
- Spatial join points to tiles if `Tile_ID` not present

Also load the `tile_classes` dict (which classes exist in each tile's raster) using the same scan loop from Cell 9 of the existing notebook — this is needed later for generating new points.

**Store in variables:**
- `kept` — GeoDataFrame of filtered points with `molca_class_2024` and `Tile_ID`
- `tiles` — GeoDataFrame of tile index with `Tile_ID`, `tif_path`, `geometry`
- `tile_classes` — dict: `{Tile_ID: set of class codes present in raster}`
- `label_map` — dict: `{class_code: label_string}`

---

## Cell 3 — Cochran sample size computation

```
n_cochran = ceil((P_EXPECTED * (1 - P_EXPECTED)) / (E_ALLOWABLE / Z_ALPHA)**2)
n_target  = n_cochran + ceil(n_cochran * BUFFER_PCT)
```

Print:
- Cochran per class: `n_cochran`
- With buffer: `n_target`
- Number of classes present in MOLCOPA 2024
- Total target: `n_target * num_classes`

---

## Cell 4 — Classify each class as surplus or deficit

For each class in `kept['molca_class_2024'].unique()`:
- Count existing points: `n_existing`
- If `n_existing >= n_target` → **surplus** (subsample down)
- If `n_existing < n_target` → **deficit** (need new points)

Print a summary table:
```
Class        Have  Target  Status      Action
Grassland      55     160  DEFICIT     keep 55, add 105 new
Cropland      194     160  SURPLUS     subsample to 160
...
```

Store result in a DataFrame for use in later cells.

---

## Cell 5 — Subsample surplus classes (spatially even)

**For each surplus class:**

Algorithm:
1. Group existing points by `Tile_ID` → get `points_per_tile` Series
2. Calculate `ideal_cap = floor(n_target / num_tiles_with_points)`
3. First pass: for each tile, keep `min(tile_count, ideal_cap)` points (randomly selected within tile)
4. Count how many points selected so far → `n_selected`
5. If `n_selected < n_target`: need `n_remaining = n_target - n_selected` more
6. Second pass: from tiles that still have unselected points, sort by number of unselected points descending. Take 1 additional random point from each tile until `n_remaining` is fulfilled.
7. Concatenate all selected points for this class.

**Within-tile random selection:** use `GeoDataFrame.sample(n, random_state=RANDOM_SEED)` on the subset of points in that tile.

Store all selected surplus points in a list.

---

## Cell 6 — Generate new points for deficit classes (spatially spread)

**For each deficit class:**

Step 1: Keep ALL existing points of this class.

Step 2: Determine how many new points needed: `n_new = n_target - n_existing`

Step 3: Find candidate tiles — tiles where this class exists in the raster BUT have no existing points of this class:
```python
tiles_with_class = {t for t, classes in tile_classes.items() if class_code in classes}
tiles_with_existing = set(kept[kept['molca_class_2024'] == class_code]['Tile_ID'].unique())
tiles_unused = tiles_with_class - tiles_with_existing
```

Step 4: Select which unused tiles to sample from — **spatially systematic**:
```python
# Get tile centroids for unused tiles
unused_tile_gdf = tiles[tiles['Tile_ID'].isin(tiles_unused)].copy()
unused_tile_gdf['centroid_x'] = unused_tile_gdf.geometry.centroid.x
unused_tile_gdf['centroid_y'] = unused_tile_gdf.geometry.centroid.y

# Sort by lat then lon for spatial ordering
unused_tile_gdf = unused_tile_gdf.sort_values(['centroid_y', 'centroid_x'])

# If n_new <= len(tiles_unused): select at regular intervals
if n_new <= len(tiles_unused):
    step = len(tiles_unused) / n_new
    indices = [int(i * step) for i in range(n_new)]
    selected_tiles = unused_tile_gdf.iloc[indices]
else:
    # Need multiple points per tile — use all unused tiles first
    # then cycle back for additional points
    selected_tiles = unused_tile_gdf  # will handle multiples below
```

Step 5: For each selected tile, randomly pick one pixel of the target class:
```python
with rasterio.open(tile_tif_path) as src:
    data = src.read(1)
    # Find all pixel positions where class matches
    rows, cols = np.where(data == class_code)
    if len(rows) == 0:
        continue  # skip tile (shouldn't happen but safety check)
    # Random select one pixel
    idx = rng.integers(0, len(rows))
    r, c = rows[idx], cols[idx]
    # Convert pixel row/col to geographic coordinates
    x, y = src.xy(r, c)  # returns center of pixel in the raster's CRS
    # If raster CRS is UTM, transform to WGS84
    if src.crs != CRS.from_epsg(4326):
        lon, lat = warp_transform(src.crs, CRS.from_epsg(4326), [x], [y])
        lon, lat = lon[0], lat[0]
    else:
        lon, lat = x, y
```

Step 6: Create a GeoDataFrame of new points with columns:
- `molca_class_2024`: the class code
- `molca_label_2024`: label from legend
- `Tile_ID`: which tile it came from
- `source`: "new_2024" (to distinguish from reused 2019 points)
- `geometry`: Point(lon, lat)

Store all new deficit points in a list.

---

## Cell 7 — Combine and tag all points

```python
# Tag existing points
surplus_selected['source'] = 'reused_2019'
deficit_existing['source'] = 'reused_2019'
deficit_new['source'] = 'new_2024'

# Combine
final = pd.concat([surplus_selected, deficit_existing, deficit_new], ignore_index=True)
final = gpd.GeoDataFrame(final, geometry='geometry', crs='EPSG:4326')
```

Standardize columns to:
- `sample_id`: sequential integer 1..N
- `molca_class_2024`: class code from MOLCOPA 2024
- `molca_label_2024`: class label
- `Tile_ID`: tile the point belongs to
- `source`: "reused_2019" or "new_2024"
- `geometry`: Point geometry in EPSG:4326

---

## Cell 8 — Validation summary

Print:
```
MOLCOPA 2024 VALIDATION SAMPLE DESIGN — {REGION}
================================================
Cochran parameters: p={P_EXPECTED}, E={E_ALLOWABLE}, Z={Z_ALPHA}
Cochran per class:  {n_cochran}
With 15% buffer:    {n_target} per class
Number of classes:  {num_classes}
Total target:       {n_target * num_classes}

Class           Reused  New   Total  Tiles_covered
Grassland(7)        55  105     160           131
Cropland(8)        160    0     160            40
...
TOTAL              754  366    1120

All points require fresh photo-interpretation for 2024.
```

Also print per-class tile coverage to confirm spatial spread.

---

## Cell 9 — Export

Save two formats:

1. **GeoJSON** — for GIS use:
```python
final.to_file(OUTPUT_FINAL, driver="GeoJSON")
```

2. **CSV with lat/lon** — for Collect Earth or CEO upload:
```python
export = final.copy()
export['longitude'] = export.geometry.x
export['latitude'] = export.geometry.y
export[['sample_id', 'longitude', 'latitude', 'molca_class_2024',
        'molca_label_2024', 'Tile_ID', 'source']].to_csv(OUTPUT_CSV, index=False)
```

---

## Key implementation notes

### Random seed
Use `numpy.random.default_rng(RANDOM_SEED)` throughout for reproducibility. Pass to all random operations (subsampling, pixel selection, tile selection).

### Tile_ID in filtered points
The current output of `filter_samples_by_molca2024.ipynb` may not include `Tile_ID`. Either:
- Add `Tile_ID` to the output of that notebook (recommended — add it in the spatial join step), OR
- Re-do the spatial join in this notebook's Cell 2

### UTM to WGS84 transformation
MOLCOPA tiles are in UTM (per the folder name `Africa_2024_UTM`). When generating new pixel coordinates with `src.xy()`, the coordinates will be in UTM. Use `rasterio.warp.transform` to convert to WGS84 (EPSG:4326) before storing in the GeoDataFrame. The existing notebook already imports `warp_transform` and handles this.

### Edge case: tiles with very few pixels of a class
When randomly selecting a pixel from a tile, some tiles may have only 1-2 pixels of a rare class (e.g., Built-up). This is fine — just select one of them randomly.

### Edge case: deficit class needs more points than unused tiles
For Wetland: 127 new points from 146 unused tiles — nearly all tiles used, 1 point each. Select 127 of 146 tiles by the spatial systematic method described above.

If a future region has a deficit class where `n_new > len(tiles_unused)`, allocate 1 point per unused tile first, then go back to tiles (including those with existing points) and add a second point, distributing as evenly as possible.

---

## Theoretical justification (for documentation/report)

**Sample size**: Cochran's equation (Cochran 1977) with:
- p = 0.90 (expected OA based on MOLCA precedent: 94-96% OA demonstrated in Bratic et al. 2023 and CCI HRLC PVP v4.0 Table 37; 0.90 used conservatively)
- E = 0.05 (5% allowable error)
- 95% confidence level (Z = 1.96)
- 15% preventive buffer (based on MOLCA discard rate of 12.4%, rounded up for temporal uncertainty)

**Sampling design**: Stratified random sampling (each MOLCOPA class = one stratum) with equal allocation across strata, following the methodology of Bratic et al. (2023) and the CCI HRLC PVP Section 5.2.

**Spatial spread**: Tile-based allocation ensures geographic coverage across the study region, following the principle of the TREES systematic sampling design used in the CCI HRLC project (PVP Section 5.2.2).

**References**:
- Cochran, W.G. (1977). Sampling Techniques, 3rd ed.
- Bratic, G., Oxoli, D., Brovelli, M.A. (2023). Map of Land Cover Agreement. Remote Sens. 15, 3774.
- CCI HRLC PVP v4.0 (2022). Product Validation Plan, CCI_HR LC_Ph1-PVP.
