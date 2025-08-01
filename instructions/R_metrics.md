# README: R Script for CHM and Terrain Metric Extraction

## Overview

This script processes tiled Lidar point clouds to generate high-resolution raster products essential for forest structure monitoring and ecosystem change analysis. It is part of the **OpenForest4D** pipeline, which supports repeatable forest metrics generation across timepoints. The script is built with `lidR`, `terra`, `sf`, and `future`, and is highly customizable and parallelized.

---

## What Metrics Are Generated and Why

Each of the following geospatial products is derived from airborne Lidar and provides a different structural insight:

### 1. **Digital Surface Model (DSM)**

The DSM represents the elevation of the highest surface hit by the laser, including canopy tops, buildings, and other features. It is critical for understanding forest canopy architecture and is generated using TIN-based interpolation over first returns.

### 2. **Digital Terrain Model (DTM)**

The DTM reflects the bare earth elevation by interpolating ground-classified returns. It is necessary for slope, elevation, and hydrological analysis, and serves as a reference for normalization.

### 3. **Canopy Height Model (CHM)**

The CHM is obtained by subtracting the DTM from the DSM. It represents the vertical structure of vegetation and is used to assess canopy density, growth, biomass, and forest disturbances.

### 4. **Rumple Index**

The Rumple Index is a metric of surface roughness and structural complexity, defined as the ratio of 3D surface area to 2D ground area. High values typically indicate mature or complex canopy layers.

### 5. **Canopy Cover (First Returns > 1m)**

Canopy cover quantifies the proportion of first returns above a certain height (e.g., 1m) within each grid cell. It is a proxy for forest openness and ecosystem light availability.

### 6. **Density >2m**

This metric estimates the proportion of all Lidar points in a cell that fall above 2 meters. It highlights the vertical distribution of vegetation and is often used in habitat modeling.

### 7. **Hillshade (optional)**

A shaded relief visualization based on slope and aspect derived from the raster (DSM/CHM/DTM). Useful for visual inspection and 3D effect rendering.

---

## Code Explanation

This R script defines a function `process_chm_pipeline()` that automates the extraction of the above metrics for all tiles in a given Lidar folder. Below is a breakdown of how it works.

### 1. **Parallel Setup and Folder Initialization**

The script uses the `future` package to run tiles in parallel (multisession plan). It creates separate output folders for each metric (DTM, DSM, CHM, etc.) under the specified `output_dir`.

### 2. **Lidar Catalog Configuration**

A LAScatalog object is created from the `base_dir`. Tiling and buffer settings are configured for efficient chunk-wise processing with minimal edge artifacts.

### 3. **Tile-wise Metric Computation**

Each tile is read and processed via `catalog_apply()`. For each tile, the following steps are executed:

#### a. **DSM Generation**

TIN-based canopy interpolation is used via `rasterize_canopy()` to estimate the highest points. DSMs can optionally be adjusted using a geoid correction raster.

#### b. **DTM Generation**

Ground returns are interpolated using `rasterize_terrain()` to compute the bare-earth surface. Like DSM, it can also incorporate geoid adjustments.

#### c. **Normalization**

Raw LAS points are normalized (heights above ground) using `normalize_height()`. This step is essential for computing CHM and other height-dependent metrics.

#### d. **CHM Computation**

Using the normalized points, the CHM is rasterized. It is masked using DSM to remove any unclassified or noisy edge values.

#### e. **Hillshade (Optional)**

If enabled, a shaded relief image is generated using slope and aspect calculations from the raster.

#### f. **Rumple Index**

If enabled, the Rumple Index is computed using surface points and a sliding grid window. It is written as a raster using `writeRaster()`.

#### g. **Canopy Cover**

Computed as the proportion of first-return points above 1m in each grid cell. It requires normalized LAS and is output per tile.

#### h. **Density >2m**

Proportion of all points above 2m in each cell. Useful for understanding vertical complexity across strata.

### 4. **Output and Logging**

Each metric is saved with year-specific filenames and stored in separate folders. Logs are printed per tile to track failures, missing data, or metrics that were skipped.

---

## How to Use This Script

### 1. **Prepare The Inputs**

* Ensure that the Lidar data is tiled and normalized if needed.
* Place tiles in `base_dir`.
* Ensure that there is a geoid file if vertical corrections are needed (in `.gtx` or `.tif`).

### 2. **Run the Pipeline**

```r
process_chm_pipeline(
  base_dir = "/path/to/tiled/Lidar",
  output_dir = "/path/to/save/metrics",
  year = 2020,
  generate_dtm = TRUE,
  generate_dsm = TRUE,
  generate_chm = TRUE,
  generate_hillshade = TRUE,
  overwrite = TRUE,
  geoid_path = "/path/to/geoid.gtx",
  compute_rumple = TRUE,
  compute_canopy_cover = TRUE,
  compute_density_2m = TRUE,
  metric_res = 10,
  size = 1000,
  buffer = 20
)
```

* **`base_dir`**: This is the directory containing the pre-tiled Lidar data in `.las` or `.laz` format. The pipeline will read all tiles from this folder and process them in parallel. The tiling is assumed to be consistent in size and coordinate system, ensuring smooth edge handling when metrics are computed.

* **`output_dir`**: This is where all raster outputs will be saved. The function creates subfolders inside this path for each type of derived product (e.g., DTM, CHM, Rumple). Having a centralized output location helps organize results and makes downstream analysis (e.g., differencing or mosaicking) easier.

* **`year`**: This numeric tag is added to every output raster file’s name (e.g., `tile_2020_dtm.tif`). It helps identify which year the raster corresponds to-especially useful when processing multi-temporal Lidar datasets for change detection.

* **`generate_dtm`, `generate_dsm`, and `generate_chm`**: These Boolean flags control whether to compute the Digital Terrain Model, Digital Surface Model, and normalized Canopy Height Model. Enabling them allows the function to extract foundational 3D structure metrics from the raw point clouds.

* **`generate_hillshade`**: When set to `TRUE`, the script will automatically generate hillshade versions of the DSM, CHM, and DTM rasters. Hillshades are useful for visual inspection, reports, and quick verification of elevation structure in a GIS viewer.

* **`overwrite`**: If set to `TRUE`, the pipeline will overwrite any existing raster files in the output directory. This ensures the most recent computations are saved and avoids confusion with stale results, especially when debugging or rerunning the pipeline with different settings.

* **`geoid_path`**: Can optionally supply a geoid correction raster in `.gtx` or `.tif` format here. If provided, the script will apply vertical adjustment by resampling the geoid to match the raster resolution and aligning the elevation values to orthometric height (NAVD88, EGM96, etc.). (See below) 

* **`compute_rumple`, `compute_canopy_cover`, and `compute_density_2m`**: These flags determine whether to compute the structural metrics derived from the CHM or point cloud. Rumple Index quantifies surface roughness, Canopy Cover measures the proportion of first returns above a certain height, and Density >2m calculates the fraction of returns above 2 meters. These metrics are especially useful in forestry, ecology, and disturbance studies.

* **`metric_res`**: This sets the resolution (in meters) for the derived metrics like Rumple, Canopy Cover, and Density >2m. A typical value is 10, which means each output cell represents a 10×10 meter grid area. Adjust this depending on the study area size and Lidar point density.

* **`size` and `buffer`**: `size` defines the tile dimension (e.g., 1000 meters square), and `buffer` adds a margin around each tile (e.g., 20 meters) to prevent edge effects during interpolation and smoothing. Buffers ensure smoother transitions between neighboring tiles and avoid artifacts at tile boundaries.


This function can be called multiple times to process different years or regions.

---

### **Geoid Correction File** (`.gtx` format) – *optional but recommended*

Most Lidar point clouds from USGS 3DEP and other sources are referenced to an **ellipsoidal vertical datum**—typically the WGS84 ellipsoid or NAD83. However, real-world elevation (what we perceive as “height above sea level”) is defined relative to a **geoid**, which approximates mean sea level and accounts for Earth's gravitational irregularities.

#### Why this matters:

* **Ellipsoid height ≠ orthometric height**
  Without correction, your Digital Surface Models (DSM), Digital Terrain Models (DTM), and derived CHM will be **offset vertically**, often by **20 to 50 meters** depending on location.

* **Consistency across years**
  Applying geoid correction ensures that metrics like CHM are directly comparable across time points and regions—even if the original Lidar datasets used different vertical datums.

* **Required for scientific accuracy**
  When conducting ecological analysis, carbon stock estimation, or forest structure change detection, **absolute elevation matters**. Vertical bias can distort CHM values and affect downstream statistics.

#### How it’s applied:

* A geoid grid (e.g., `geoid_18_CONUS_save32612.gtx`) stores the vertical separation between the ellipsoid and the geoid in meters for every location.
* During CHM processing, the script **adds this correction raster** to the DSM and DTM layers using bilinear resampling to apply a smooth, spatially-aware adjustment.
* The correction is done **after rasterization** but before differencing DSM and DTM.

#### How to obtain or generate it:

* You can download geoid models from [NOAA NGS Geoid Downloads](https://geodesy.noaa.gov/GEOID/)
* Convert the `.tif` geoid raster into a `.gtx` format aligned with your projected coordinate system using:

  ```bash
  gdalwarp -overwrite -s_srs EPSG:4326 -t_srs EPSG:32612 -r bilinear \
    -of GTX geoid_18_CONUS.tif geoid_18_CONUS_save32612.gtx
  ```

**Note:** Even though this step is optional, it is *highly recommended* for vertical alignment across datasets and accurate CHM calculation.


## Sample Outputs

Each metric will be saved as GeoTIFFs under a subfolder like:

```
/output_dir/
  ├── DTM_tiles/
  ├── DSM_tiles/
  ├── CHM_normalized_tiles/
  ├── Rumple_tiles/
  ├── Canopy_Cover_tiles/
  └── Density_Tiles/
```

These files can be loaded directly into QGIS or ArcGIS.

---


## Requirements

* **R (≥ 4.1.0)**
* **Required Packages:**

  ```r
  install.packages("lidR")
  install.packages("terra")
  install.packages("sf")
  install.packages("future")
  install.packages("geometry")
  ```

These packages support high-performance point cloud processing, raster manipulation, spatial transformations, and parallel computation.

---

## What’s Next: Change Detection and Visualization

After generating CHM and related raster products for multiple time points (e.g., 2012 and 2018), you can proceed with:

### **Raster Differencing for Forest Change Detection**

Use the included **raster differencing script** to compare structural metrics across years:

* Compute **CHM differencing** (`CHM_2018 - CHM_2012`) to identify canopy loss or regrowth.
* Compare other structural metrics such as Rumple Index or Canopy Cover before and after disturbance.
* Automatically handle tile alignment, NoData values, and edge artifacts.

 Refer to the `differencing_script.ipynb` file for step-by-step usage.

### **Load Outputs into QGIS for Exploration**

All raster products (e.g., `CHM_normalized_tiles`, `DTM_tiles`, `Rumple_tiles`) are saved as **GeoTIFFs**, making them ready for direct import into **QGIS** or **ArcGIS**.

In QGIS, you can:

* Overlay DSM, DTM, and CHM for visual inspection.
* Apply color ramps or hillshades to analyze structure.
* Use raster calculator to create custom visualizations.

---

These next steps enable full **scientific interpretation**, **monitoring**, and **communication** of changes in forest canopy structure using Lidar-derived data.

