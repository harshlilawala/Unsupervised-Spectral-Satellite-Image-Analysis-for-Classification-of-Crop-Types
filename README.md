# Unsupervised Crop-Type Classification with Sentinel-1 SAR (K-means & SLIC)

This repository explores **unsupervised spectral analysis** of Sentinel‑1 SAR imagery for **crop‑type classification** in a German agricultural landscape. 

The approach leverages Sentinel‑1’s **cloud‑penetrating, all‑weather** capability to support robust, year‑round crop monitoring in precision agriculture.

---

## Table of Contents

- [Data Requirements](#data-requirements)
- [Quick Start](#quick-start)
- [Notebook Walkthrough](#notebook-walkthrough)
  - [1) Load & Prepare Labels](#1-load--prepare-labels)
  - [2) Load & Prepare SAR Features](#2-load--prepare-sar-features)
  - [3) Pixel‑level K‑means](#3-pixellevel-kmeans)
  - [4) Object‑level SLIC + Region Features + K‑means](#4-objectlevel-slic--region-features--kmeans)
  - [5) Evaluation](#5-evaluation)
- [Configuration](#configuration)
- [Results & Discussion](#results--discussion)
- [Common Issues](#common-issues)
- [Cite / Acknowledge](#cite--acknowledge)
- [License](#license)

---




## Data Requirements

The notebook expects the following local files (paths can be adjusted in the first cells):

- `labels.geojson` — vector polygons with a `crop_id` attribute (integer codes) covering the area of interest.
- `vv.npy`, `vh.npy` — arrays containing VV/VH **backscatter** (or log-scaled) time slices or channels.
  - In the notebook, the first channel is used: `vv[:, :, :, 0]` and `vh[:, :, :, 0]`. If your arrays differ, adjust indexing accordingly.
  - The workflow applies a log10 transform and multiplies by the rasterized label mask.

**Coordinate consistency:** `labels.geojson` is normalized to image space using the total bounds of all geometries and the target image shape, then rasterized with `rasterio`. Ensure your polygons and arrays describe the same spatial extent.

---

## Quick Start

### 1) Environment (conda)
```bash
conda create -n s1-unsup python=3.10 -y
conda activate s1-unsup

# Core scientific stack
pip install numpy scipy scikit-image scikit-learn matplotlib tqdm seaborn

# Geospatial and IO
pip install rasterio geopandas shapely

# CV/Feature extraction
pip install opencv-python
```

> If you hit installation issues for `rasterio`/`geopandas`, prefer prebuilt wheels:
```bash
pip install --only-binary=:all: rasterio geopandas shapely
```

### 2) Run the notebook
```bash
jupyter lab
# open K_means_SLIC_Prog.ipynb and run cells top-to-bottom
```

Place `labels.geojson`, `vv.npy`, and `vh.npy` in the same directory (or edit paths in the notebook).

---

## Notebook Walkthrough

### 1) Load & Prepare Labels
- Reads `labels.geojson` with **GeoPandas**.
- Normalizes polygon coordinates to the image grid using dataset bounds and the target raster shape.
- Rasterizes polygons to an integer **mask** using `rasterio.features.rasterize` with `value_column='crop_id'`.

**Key libs:** `geopandas`, `shapely`, `rasterio`

### 2) Load & Prepare SAR Features
- Loads `vv.npy` and `vh.npy` (assumes shape like `[rows, cols, bands, time]` or similar).
- Uses the first slice: `vv[:, :, :, 0]`, `vh[:, :, :, 0]`.
- Applies `log10` scaling with small epsilon, multiplies by the mask.
- Assembles a `(N_pixels × 2)` feature matrix `[VH, VV]` for clustering (NaN/Inf handling included).

**Key libs:** `numpy`, `opencv (cv2)` for later processing

### 3) Pixel‑level K‑means
- **KMeans** (`sklearn.cluster.KMeans`) on `[VH, VV]`.
- Produces a label image for each slice.
- Compares predicted labels to `crop_id` mask via **confusion matrices** (`sklearn.metrics.confusion_matrix`).
- Provides a combined confusion matrix across time slices and a simple **accuracy %** (sum of diagonal / total).

**Key libs:** `scikit-learn`, `seaborn` for heatmap plotting

### 4) Object‑level SLIC + Region Features + K‑means
- Splits image into **grids**; runs **SLIC superpixels** (`skimage.segmentation.slic`) per grid to get compact regions.
- Extracts **edges** (Canny; thresholds derived from image stats).
- Intersects edges with SLIC to emphasize segment boundaries.
- Reassembles grid results back to full‑image maps.
- Finds **contours** (`cv2.findContours`) of coherent regions.
- For each region, computes multiple descriptors:
  - **Intensity histogram** (`cv2.calcHist`)
  - **HOG** (Histogram of Oriented Gradients)
  - **GLCM** contrast (`skimage.feature.graycomatrix`, `graycoprops`)
- Concatenates descriptors and clusters regions with **KMeans** (object‑level labeling).
- Visualizes the final region labels as a map.

**Key libs:** `scikit-image`, `opencv-python`, `scikit-learn`

### 5) Evaluation
- **Confusion matrix** per pixel for the K‑means pipeline and an aggregated matrix across slices.
- Overall **accuracy percentage** (naive; suitable for relative comparison).
- Visual inspection of region‑level labels for SLIC pipeline.

> Tip: If class IDs are not aligned with K‑means labels, remap clusters to majority‑vote crop IDs before scoring to get a more meaningful accuracy. The notebook includes a mapping step using majority assignment per cluster ID.

---

## Configuration

Common parameters (edit directly in the notebook cells):

- `n_clusters`: number of unsupervised clusters (default 10).
- `grid_size`: grid size for SLIC tiling (e.g., 25).
- `slic` params: `n_segments`, `compactness`, `sigma`.
- Canny thresholds: computed automatically from the median intensity (`auto_canny_threshold`); tweak `sigma` if edges are too weak/strong.

---

## Results & Discussion

The experiments show that:
- **Unsupervised K‑means** on VV/VH can separate major crop groups reasonably well in a cloud‑agnostic manner, though class mixing occurs when backscatter signatures overlap.
- **SLIC + region descriptors** captures **shape/texture** differences and can refine boundaries, improving interpretability for heterogeneous fields.
- Trade‑offs:
  - Pixel‑level is **fast & simple** but sensitive to speckle and within‑field variability.
  - Object‑level is **richer & cleaner** but requires more parameters and compute.

The methodology supports precision‑ag workflows by enabling **label‑efficient** monitoring without supervised training.

---

## Common Issues

- **Import errors** for `rasterio/geopandas/shapely`: use prebuilt wheels; on Windows consider `conda-forge`.
- **Array shape mismatches**: verify the indexing used for `vv`/`vh` and that image shapes match the rasterized mask.
- **Label alignment**: ensure `crop_id` values in `labels.geojson` are integers and cover the area of interest.
- **Memory**: large arrays can exhaust RAM; consider tiling or downsampling.

---

## Cite / Acknowledge

If this work helps your research or application, please cite the repository and acknowledge Sentinel‑1 mission data. You may also cite the core libraries: **NumPy**, **SciPy**, **scikit‑learn**, **scikit‑image**, **Rasterio**, **GeoPandas**, **Shapely**, and **OpenCV**.



---

## License

This project is released under the **Swansea University License**
---

### Maintainer

- Harsh Lilawala — lilawalaharsh@gmail.com

