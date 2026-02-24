# Instructions for `cell_height_change_quant`

## Overview

`cell_height_change_quant` is a Python script that quantifies cell height changes over time from multi-slice TIFF microscopy images. It processes fluorescence images (e.g., ZO-1 tight junction marker), detects the apical surface of cells using skeletonized binary masks, and computes protein displacement (a proxy for cell height) across time points represented as spatial columns in each image slice.

---

## Requirements

Install the following Python packages before running the script:

```bash
pip install numpy opencv-python matplotlib scikit-image scipy tifffile pandas openpyxl
```

| Package        | Purpose                                      |
|---------------|----------------------------------------------|
| `numpy`        | Numerical array operations                   |
| `opencv-python`| Image I/O utilities                          |
| `matplotlib`   | Plotting displacement graphs                 |
| `scikit-image` | Image filtering, thresholding, skeletonizing |
| `scipy`        | Spatial indexing, Gaussian smoothing         |
| `tifffile`     | Reading multi-page TIFF files                |
| `pandas`       | Saving results to Excel                      |
| `openpyxl`     | Excel file writing backend for pandas        |

---

## Input Files

### 1. Control Folder
A folder containing control `.tif` images used to automatically compute the binarization threshold.

- All files must be in **TIFF format** (`.tif` extension).
- Images should represent the same staining channel as the experimental image.
- The threshold is computed as the mean pixel intensity across all control images after Gaussian blurring.

**Example path:**
```
/Users/apple/Desktop/Ian_lab/python_script/cell_height_v2/control_png
```

### 2. Experimental Image
A **multi-slice TIFF** file where:
- Each **page/slice** corresponds to one measurement (e.g., one cell or one field of view).
- The **x-axis** (image width) represents **time** (mapped to 0–20 hours by default).
- The **y-axis** (image height) represents **vertical position** within the cell (apical at top, basal at bottom).

**Example path:**
```
/Users/apple/Desktop/Ian_lab/python_script/cell_height_v2/250813-EpH4 Zo1mSG NT or KOPals1+Veh_CNO3_H2O Overnight 001 - Denoised-Slices-Slices_crop_crop.tif
```

> **Note:** The image should be pre-processed and re-saved through Fiji (ImageJ) to ensure correct slice ordering and time displacement representation.

---

## Configuration

Before running, update the following variables near the top of the script:

```python
# Path to folder containing control .tif images
control_folder = "/path/to/your/control_png"

# Path to the multi-slice TIFF experimental image
image_path = "/path/to/your/experimental_image.tif"

# Folder where results (Excel files + graphs) will be saved
save_folder = "/path/to/your/results/output_folder"
```

### Optional Parameters

| Parameter        | Default | Description                                              |
|-----------------|---------|----------------------------------------------------------|
| `sigma`          | `1`     | Gaussian blur sigma for preprocessing                    |
| `size_threshold` | `20`    | Minimum region area (pixels) to keep after binarization  |
| `pixel_size_um`  | `0.16`  | Physical pixel size in micrometers                       |
| `min_valid_y`    | `20`    | Minimum y-coordinate (pixels) for valid apical detection |
| `total_hours`    | `20.0`  | Total time span (hours) mapped across image width        |
| `interval`       | `1`     | Grid line spacing (pixels) for intersection counting     |

---

## How It Works

1. **Threshold computation:** Gaussian-blurred control images are averaged to compute a global binarization threshold.
2. **Preprocessing per slice:** Each slice is Gaussian-blurred, binarized using the global threshold, small regions are removed, and the result is skeletonized.
3. **Grid-based intersection:** A fine 1-pixel-interval grid is overlaid on the binary skeleton. Intersection points between the skeleton and vertical grid lines are identified using a KD-tree for efficiency.
4. **Displacement calculation:** For each vertical grid line (time point):
   - The basal y-position is fixed from the rightmost vertical line.
   - The apical y-position is estimated as the 10th percentile of skeleton y-coordinates along that line.
   - **Cell height (protein displacement)** = `(basal_y − apical_y) × pixel_size_um`
5. **Smoothing:** Displacements are smoothed with a 1D Gaussian filter (sigma=5).
6. **Feature detection:** The script detects the earliest local minimum in the smoothed displacement curve (using first and second derivatives), and identifies the preceding local maximum as the "starting time." The time difference between these two points is reported.

---

## Outputs

All outputs are saved to `save_folder`.

### Per-Slice Excel Files
**Filename:** `slice_<i>_prt_displacement.xlsx`

| Column                                | Description                              |
|---------------------------------------|------------------------------------------|
| `Time (hr)`                           | Time mapped from pixel x-position       |
| `Protein Displacement Raw (µm)`       | Raw per-column cell height estimate      |
| `Protein Displacement Smoothed (µm)` | Gaussian-smoothed cell height estimate   |

### Per-Slice Graphs
**Filename:** `slice_<i>_prt_displacement.png`

- Light gray line: raw displacement
- Colored line: smoothed displacement
- Red dotted vertical lines + red dots: detected starting time and local minimum time
- Red dashed horizontal line: average displacement across all time points

### Summary Excel File
**Filename:** `average_prt_displacements_summary.xlsx`

| Column                              | Description                                              |
|-------------------------------------|----------------------------------------------------------|
| `Slice Index`                       | Index of each processed slice                            |
| `Average Protein Displacement (µm)` | Mean cell height across all time points in that slice    |
| `t_first_point (hr)`               | Time of the earliest local minimum in displacement       |
| `t_first_zero (hr)`                | Time of the preceding local maximum (starting time)      |
| `Time to first d/dt=0 (hr)`        | Time difference between starting time and minimum time   |

---

## Running the Script

```bash
python cell_height_change_quant
```

> **Note:** The script file `cell_height_change_quant` has no `.py` extension. If your system requires it, rename the file to `cell_height_change_quant.py` and run `python cell_height_change_quant.py` instead.

Or open the file in your preferred Python IDE (e.g., VS Code, PyCharm, Spyder) and run it directly.

---

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| `FileNotFoundError` on control folder | Wrong path | Update `control_folder` path in the script |
| `FileNotFoundError` on image | Wrong path or filename | Update `image_path` with the correct absolute path |
| All displacements are `NaN` | Threshold too high / image too dark | Check control images; try adjusting `sigma` or inspect image intensities |
| No local minimum detected | Flat or noisy displacement curve | Increase smoothing sigma or inspect per-slice graphs for issues |
| Shape mismatch warning (`⚠️ Skipping slice`) | Unexpected image dimensions | Ensure slices are 2D grayscale; re-export from Fiji if needed |

---

## Notes

- The script assumes the **rightmost column** of each slice marks the basal surface.
- The x-axis of the image is treated as **linearly mapping to time** (0 to `total_hours`). Adjust `total_hours` if your experiment uses a different duration.
- The script prints debug information for slice 0 (first slice) to assist with parameter tuning.
