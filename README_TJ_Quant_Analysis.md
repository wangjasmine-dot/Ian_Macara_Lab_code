# Instructions for `TJ Quant Analysis v4`

## Overview

`TJ Quant Analysis v4` is a Jupyter notebook that quantifies tight junction (TJ) network structure from fluorescence microscopy images. It processes `.tif` images of tight junction markers (e.g., ZO-1-mScarlet-G), binarizes and skeletonizes the junction network, and uses a grid-based intersection approach to measure uninterrupted segment lengths as a proxy for junction continuity. Results are exported as CSV files and bar graphs.

---

## Requirements

Install the following Python packages before running the notebook:

```bash
pip install numpy opencv-python matplotlib scikit-image scipy pandas
```

| Package        | Purpose                                           |
|---------------|---------------------------------------------------|
| `numpy`        | Numerical array operations                        |
| `opencv-python`| Image I/O utilities                               |
| `matplotlib`   | Plotting analysis results                         |
| `scikit-image` | Image filtering, thresholding, morphological thinning |
| `scipy`        | Spatial indexing (KD-tree for intersection search)|
| `pandas`       | Saving results to CSV                             |

---

## Input Files

### 1. Control Folder
A folder containing control `.tif` images used to automatically compute the binarization threshold.

- All files must be in **TIFF format** (`.tif` extension).
- Images should represent the same staining channel as the experimental images.
- The threshold is computed as the mean pixel intensity plus 0.5× the standard deviation across all control images after Gaussian blurring.

**Example path:**
```
/Users/apple/Desktop/Ian_lab/data/CTRL
```

### 2. Image Folder
A folder containing experimental `.tif` images to be analyzed.

- All files must be in **TIFF format** (`.tif` extension).
- Images can be grayscale or RGB (RGB images are automatically converted to grayscale).

**Example path:**
```
/Users/apple/Desktop/Ian_lab/data/TJlength_analysis
```

### 3. Output Folder
A folder where all results (plots and CSV files) will be saved. It will be created automatically if it does not exist.

**Example path:**
```
/Users/apple/Desktop/Ian_lab/data/TJlength_analysis/Output
```

---

## Configuration

Before running, update the following variables in **Cell 9** of the notebook:

```python
# Path to folder containing control .tif images
control_folder = "/path/to/your/CTRL"

# Path to folder containing experimental .tif images
image_folder = "/path/to/your/TJlength_analysis"

# Path to output folder (will be created if it does not exist)
output_folder = "/path/to/your/TJlength_analysis/Output"
```

In **Cell 10**, update paths and the zoom/magnification setting:

```python
# Path to the segment_lengths.csv produced by Cell 9
segment_lengths_path = "/path/to/your/TJlength_analysis/Output/segment_lengths.csv"

# Choose the zoom level matching your images
zoom_level = "40x_1x"   # or "40x_2x"
```

### Grid Parameters (in `generate_grid`, Cell 8)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `nx`       | `80`    | Number of vertical grid lines |
| `ny`       | `80`    | Number of horizontal grid lines |

### Preprocessing Parameters (in `preprocess_image`, Cell 8)

| Parameter        | Default | Description |
|-----------------|---------|-------------|
| `sigma`          | `2`     | Gaussian blur sigma for preprocessing |
| `size_threshold` | `50`    | Minimum region area (pixels) to keep after binarization |

### Pixel Size Settings (in Cell 10)

| Zoom Level | Pixel Size (µm/pixel) |
|-----------|----------------------|
| `40x_1x`   | 0.31                 |
| `40x_2x`   | 0.16                 |

Update `zoom_level` in Cell 10 to match your microscope settings.

---

## How It Works

1. **Threshold computation:** A global binarization threshold is derived from the control images as `mean + 0.5 × std` of Gaussian-blurred pixel intensities.
2. **Preprocessing per image:** Each experimental image is Gaussian-blurred, binarized using the global threshold, small regions below `size_threshold` are removed, and the result is morphologically thinned (skeletonized).
3. **Grid generation:** An 80×80 grid of evenly spaced vertical and horizontal lines is overlaid on the image.
4. **Segment length measurement:** For each grid line, the algorithm traverses the line pixel by pixel. It uses a KD-tree to find skeleton pixels within 1-pixel distance. Consecutive runs of non-skeleton pixels define **uninterrupted segments** — stretches where the junction network is present without a break. Segment lengths are recorded in pixels.
5. **Scaling:** In Cell 10, pixel-based segment lengths are converted to micrometers using the pixel size for your objective/zoom combination.
6. **Aggregation and plotting:** Average and median segment lengths are computed per image (and per frame, if applicable) and plotted as a bar graph.

---

## Outputs

All outputs are saved to `output_folder`.

### Per-Image Analysis Plots
**Filename:** `<image_name>_analysis.png`

Each plot contains two panels:
- **Left:** Original grayscale image.
- **Right:** Processed binary/thinned image with the 80×80 grid overlaid (lime green lines) and colored segments (alternating yellow/blue).

### Segment Lengths CSV
**Filename:** `segment_lengths.csv`

| Column | Description |
|--------|-------------|
| `Filename` | Name of the source image file |
| `Segment_ID` | Index of each segment within that image |
| `Segment_Length` | Uninterrupted segment length (pixels) |

### Average Segment Lengths CSV (per frame)
**Filename:** `average_segment_lengths_by_frame.csv`

| Column | Description |
|--------|-------------|
| `Filename` | Name of the source image file |
| `Frame` *(if present)* | Frame index |
| `Average_Scaled_Length` | Mean scaled segment length (µm) |
| `Median_Scaled_Length` | Median scaled segment length (µm) |
| `Segment_Count` | Total number of segments measured |

### Average Segment Lengths CSV (per file)
**Filename:** `average_segment_lengths_by_file.csv` *(only generated if a `Frame` column exists in the data)*

Same columns as above, but aggregated across all frames for each file.

### Bar Graph
Displayed interactively within the notebook (Cell 10). Shows average scaled segment length per image or frame.

---

## Running the Notebook

Open `TJ Quant Analysis v4.ipynb` in Jupyter Notebook or JupyterLab and run the cells **in order**:

1. **Cells 0–7** — Background/version notes (markdown only; no code to run).
2. **Cell 8** — Define all helper functions. **Run this first** after a kernel restart.
3. **Cell 9** — Set paths, compute threshold, process all images, and save `segment_lengths.csv` and per-image plots.
4. **Cell 10** — Load `segment_lengths.csv`, scale to micrometers, aggregate by image/frame, save summary CSVs, and display the bar graph.

---

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| `FileNotFoundError` on control folder | Wrong path | Update `control_folder` in Cell 9 |
| `FileNotFoundError` on image folder | Wrong path | Update `image_folder` in Cell 9 |
| All segment lengths are 0 | Threshold too high; all images appear blank after binarization | Check control images; inspect `global_threshold` value; lower `sigma` or adjust control images |
| Very short segments everywhere | Threshold too low; too much noise is retained | Increase `size_threshold` or check that control images represent true background |
| Bar graph is empty | No `.tif` files in `image_folder` | Ensure files end with `.tif` (lowercase) and are in the correct folder |
| Pixel size is wrong | Wrong `zoom_level` selected | Update `zoom_level` in Cell 10 to match your microscope settings |

---

## Notes

- Segment lengths are measured as **uninterrupted stretches without a junction skeleton pixel**, not the length of junction segments themselves. Longer uninterrupted lengths indicate less continuous or more fragmented junction networks.
- The 80×80 grid spacing is fixed relative to the image size. If your images have very different resolutions, consider adjusting `nx` and `ny` in `generate_grid`.
- The notebook currently supports `.tif` images only. Ensure all files use the `.tif` extension (not `.tiff`).
