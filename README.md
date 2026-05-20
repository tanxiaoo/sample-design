# MOLCA / MOLCOPA Validation Sample Design

This project contains the notebooks and data used to build 2024 validation samples from 2019 photo-interpreted reference points and MOLCA 2024 raster coverage.

The full sample-selection pipeline is:

1. Start from the 2019 reference points.
2. Filter out points that do not fall on valid MOLCA 2024 coverage.
3. Re-extract the 2024 MOLCA class value at each kept point.
4. Compute a per-class target sample size with a Cochran-style formula.
5. If a class has too many points, subsample it evenly across tiles.
6. If a class has too few points, add new points from tiles that contain that class but do not already contribute existing samples.
7. Export the final validation sample as GeoJSON and CSV.

The workflow is split into two notebooks:

1. [filter_samples_by_molca2024.ipynb](filter_samples_by_molca2024.ipynb) filters the 2019 reference points to keep only samples that fall on valid MOLCA 2024 coverage and re-samples the 2024 class value at each kept point.
2. [sample_design_molcopa2024.ipynb](sample_design_molcopa2024.ipynb) builds the final validation sample by applying a Cochran-based target per class, subsampling surplus classes, and generating new points for deficit classes.

Both notebooks are region-agnostic. To run a different region, edit the `REGION` value in the first configuration cell and keep the rest of the notebook unchanged.

## Repository Layout

```text
config/                         Class legend files used by both notebooks
docs/                           Sampling plan and implementation notes
MOLCA/                          Region-specific MOLCA 2024 rasters and tile indexes
Samples_2019/                   Original 2019 photo-interpreted reference points
Samples_2024/                   Filtered points and final 2024 validation samples
Sentine-2_UTM_Tiling_Grid_By_Region/  Supporting tiling-grid geopackages
requirements.txt                Python dependencies
```

## What the Notebooks Produce

### `filter_samples_by_molca2024.ipynb`

Input:

- `Samples_2019/<region>_static_area_2019_*.geojson`
- `MOLCA/<region>_2024_UTM/molcx_<region>_2024_s2_exist.gpkg`
- `MOLCA/<region>_2024_UTM/Tiffs/`
- `config/MOLCA_legend.csv`

Output:

- `Samples_2024/<region>_static_area_2024_molca_filtered.geojson`

This notebook keeps only points that land inside a valid MOLCA 2024 tile and on a non-nodata pixel, then stores the sampled 2024 class and label.

### `sample_design_molcopa2024.ipynb`

Input:

- the filtered GeoJSON produced by `filter_samples_by_molca2024.ipynb`
- the same MOLCA tile index and GeoTIFF directory used above
- `config/MOLCA_legend.csv`

Outputs:

- `Samples_2024/<region>_validation_2024_final.geojson`
- `Samples_2024/<region>_validation_2024_final.csv`

This notebook computes a per-class target sample size, balances surplus classes by tile, and fills deficit classes by generating additional points in unused tiles that already contain the target class.

## Cochran Target Per Class

The notebook uses a Cochran-based target to estimate the minimum sample size per class:

$$
n_{cochran} = \left\lceil \frac{p(1-p)}{(E/Z)^2} \right\rceil
$$

With the current notebook values:

- $p = 0.90$ expected accuracy
- $E = 0.05$ allowable error
- $Z = 1.96$ for 95% confidence
- buffer = 15% to account for photo-interpretation discards

That gives:

- $n_{cochran} = 139$ samples per class
- buffered target $n_{target} = 160$ samples per class

So the notebook aims for 160 validation points for each class that appears in the filtered MOLCA 2024 set. This is the number used to decide whether a class is surplus or deficit.

## How the Point Selection Works

### Surplus classes

If a class already has at least 160 points, the notebook reduces it to the target while keeping the spatial distribution as even as possible across tiles.

The selection logic is:

1. Group points by `Tile_ID`.
2. Compute an ideal per-tile cap from the target and the number of tiles that contain that class.
3. Take up to that many points from each tile using random sampling within the tile.
4. If the target is still not reached, take one more point at a time from tiles that still have unused points, preferring tiles with more remaining candidates.

This avoids concentrating the selected points in only a few tiles.

### Deficit classes

If a class has fewer than 160 points, the notebook keeps all existing points and generates the missing ones.

The new points are chosen from tiles that:

1. contain the target class in the MOLCA raster,
2. do not already have existing samples for that class,
3. are ordered spatially so the chosen tiles are spread out.

For each selected tile, the notebook picks a random pixel of the target class, converts that pixel center to WGS84, and stores it as a new validation point. These points are tagged as `source = "new_2024"` so they can be separated from reused 2019 points.

## Output Logic

The final table combines:

- reused 2019 points from surplus classes that were kept after subsampling,
- reused 2019 points from deficit classes,
- newly generated 2024 points for deficit classes.

Each final sample gets a sequential `sample_id` and is exported in:

- GeoJSON for GIS use,
- CSV with longitude and latitude for Collect Earth or CEO upload.

## Requirements

Install the Python packages listed in [requirements.txt](requirements.txt). The current environment uses:

- numpy
- pandas
- geopandas
- rasterio
- shapely
- fiona
- pyproj
- ipykernel

## How to Run

1. Open the workspace in VS Code.
2. Ensure the Python environment has the dependencies from [requirements.txt](requirements.txt).
3. Run [filter_samples_by_molca2024.ipynb](filter_samples_by_molca2024.ipynb) for the desired region.
4. Run [sample_design_molcopa2024.ipynb](sample_design_molcopa2024.ipynb) using the filtered output from the previous step.

## Notes

- The notebooks are written for Africa, Amazon_Extension, and Siberia, using the same control flow and region-specific file naming.
- `Samples_2024/` stores both intermediate and final outputs so the pipeline can be rerun without moving files around.
- The implementation plan in [docs/MOLCOPA_sampling_plan.md](docs/MOLCOPA_sampling_plan.md) describes the intended logic behind the final validation-sample construction.