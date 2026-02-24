# Instructions for `Displacement Tracking v08`

## Overview

`Displacement Tracking v08` is a Jupyter notebook that performs automated displacement tracking and analysis for tissue ablation experiments. It processes time-lapse TIFF stacks of epithelial monolayers, detects ablation events through frame correlation analysis, estimates tissue displacement using optical flow, and tracks the separation of selected features (e.g., tricellular junctions) over time. The notebook also supports batch comparison between experimental groups (e.g., NT vs. Pals1 KO).

---

## Requirements

Install the following Python packages before running the notebook:

```bash
pip install numpy tifffile opencv-python matplotlib ipywidgets scipy pandas
```

| Package        | Purpose                                        |
|---------------|------------------------------------------------|
| `numpy`        | Numerical array operations                     |
| `tifffile`     | Reading multi-page TIFF stacks                 |
| `opencv-python`| Image I/O, grayscale conversion, optical flow  |
| `matplotlib`   | Plotting displacement and tracking results     |
| `ipywidgets`   | Interactive verification UI within the notebook|
| `scipy`        | Statistical analysis of tracking data          |
| `pandas`       | Exporting results to CSV                       |

---

## Input Files

### TIFF Stack
A **multi-frame TIFF file** where each frame is a time point in the ablation experiment.

- Each frame should be **2D grayscale** (or color images that will be automatically converted).
- Frames are ordered chronologically; the ablation event is detected automatically from frame correlations.

**Example path:**
```
/Users/apple/Desktop/Ian_lab/python_script/ablation assay python/20250918_NT1c5-ZO1-CB650-TJ_72h_001.tif
```

---

## Configuration

All configuration is done interactively within the notebook:

1. **Select TIFF file** — A file dialog opens to let you select your TIFF stack.
2. **Verify ablation frame** — An interactive UI shows frame-to-frame correlations; confirm the ablation frame and any obscured frames.
3. **Select junctions to track** — Click on the image to mark two tricellular junctions whose separation will be tracked.

### Optional Parameters (in the code cells)

| Parameter             | Description                                                     |
|----------------------|-----------------------------------------------------------------|
| `pre_ablation_frame`  | Frame index just before ablation (detected automatically)       |
| `first_nonobscured_frame` | First frame after ablation debris clears                   |
| Optical flow settings | Controlled via `cv2.calcOpticalFlowFarneback` parameters        |

---

## How It Works

1. **Load and preprocess stack:** The TIFF stack is loaded and converted to 8-bit grayscale if needed.
2. **Frame correlation analysis:** Mean-centered normalized cross-correlations between successive frames are computed. A sharp drop in correlation identifies the ablation event.
3. **Interactive verification:** A UI widget displays the correlation plot and lets you confirm the ablation frame, obscured frames, and junction positions to track.
4. **Displacement field estimation:** Dense optical flow (Farneback method) is computed between each pair of successive frames to build a displacement field.
5. **Feature tracking:** Two selected junction positions are tracked across frames using the displacement field. The Euclidean distance between the two tracked points is computed at each frame.
6. **Distance change calculation:** The distance change relative to the first post-ablation frame is exported.
7. **Batch comparison (optional):** Cell 4 compares feature separation between NT and Pals1 KO groups across multiple experiments and replicates. Cell 5 calculates the initial rate of feature separation.

---

## Outputs

All outputs are saved to an automatically created export directory named `<filename>_exports/` in the same directory as the notebook.

### Per-Sample Files

| File | Description |
|------|-------------|
| `<filename>_tracking_results.csv` | Frame-by-frame tracking positions and distance change |
| `displacement_overlay_*.png` | Overlay images showing displacement vectors per frame |
| `figures/*.pdf` | Collection of analysis figures in PDF format |

### CSV Columns (tracking results)

| Column | Description |
|--------|-------------|
| `Frame` | Absolute frame index in the original stack |
| `Relative Frame` | Frame index relative to the ablation event (0 = ablation frame) |
| `Junction 1 X`, `Junction 1 Y` | Tracked position of junction 1 (pixels) |
| `Junction 2 X`, `Junction 2 Y` | Tracked position of junction 2 (pixels) |
| `Distance` | Euclidean distance between the two junctions (pixels) |
| `Distance Change` | Distance change relative to the first post-ablation frame (pixels) |

### Batch Comparison Outputs (Cells 4–5)

| File | Description |
|------|-------------|
| `feature_separation_comparison.png` | Bar graph comparing mean distance change between groups |
| `initial_rate_comparison.png` | Bar graph comparing the initial rate of feature separation |

---

## Running the Notebook

Open `Displacement Tracking v08.ipynb` in Jupyter Notebook or JupyterLab and run the cells **in order**:

1. **Cell 1** — Load libraries and define helper functions (run first after kernel restart).
2. **Cell 1 (continued)** — Select the TIFF file via the file dialog, analyze frame correlations, and complete the interactive verification UI (click "✓ Confirm" when done).
3. **Cell 2** — Continue analysis after verification: estimate displacement fields, create overlays, and track features.
4. **Cell 4** *(optional)* — Batch comparison between NT and Pals1 KO groups. Update the `notebook_dir` path and sample naming patterns to match your data.
5. **Cell 5** *(optional)* — Calculate the initial rate of feature separation for each sample group.

> **Note:** Cell 2 must be run **after** clicking "✓ Confirm" in Cell 1's interactive UI. If you rerun the kernel, restart from Cell 1.

---

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| `FileNotFoundError` on TIFF | Wrong path or file moved | Rerun the file selection cell and pick the correct file |
| Ablation frame not detected | Flat correlation curve | Check if images are sufficiently different between frames; manually set `pre_ablation_frame` |
| Feature tracking drifts | High noise or low contrast | Inspect overlay images; adjust optical flow parameters |
| Batch analysis finds no export folders | Wrong `notebook_dir` path or no `*_exports` directories | Update `notebook_dir` to the folder containing all `*_exports` subfolders |
| `KeyError: 'Distance Change'` in batch cell | CSV column name mismatch | Check the CSV header in one of your export files and update the column name in the code |

---

## Notes

- The notebook is designed for time-lapse images of epithelial monolayers stained with tight junction markers (e.g., ZO-1-mScarlet-G).
- The optical flow method assumes relatively small frame-to-frame displacements. Very fast tissue recoil may reduce tracking accuracy immediately after ablation.
- The batch comparison cells (4–5) assume a specific file naming convention used in the Ian Macara Lab. Update the `*_pattern` variables to match your own file naming scheme.
