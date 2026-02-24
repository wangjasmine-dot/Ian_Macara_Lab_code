# Instructions for `Tricell_Tracker_Notebook_v3`

## Overview

`Tricell_Tracker_Notebook_v3` is a Jupyter notebook for automated detection and tracking of tricellular junctions (vertices where three cells meet) across time-lapse fluorescence microscopy TIFF stacks. It implements a three-step interactive pipeline: (1) automatic detection on frame 0, (2) optional manual correction of detections, and (3) multi-frame tracking using local search with Kalman filtering and Hungarian assignment. An optional additional step supports laser ablation recoil analysis with bi-exponential fitting.

---

## Requirements

Install the following Python packages before running the notebook:

```bash
pip install numpy opencv-python matplotlib scikit-image scipy pandas pillow ipywidgets tifffile imageio
```

| Package        | Purpose                                                    |
|---------------|------------------------------------------------------------|
| `numpy`        | Numerical array operations                                 |
| `opencv-python`| Gaussian blur, image processing                            |
| `matplotlib`   | Plotting overlays and track time series                    |
| `scikit-image` | Ridge enhancement (Sato), thresholding, skeletonizing, morphology |
| `scipy`        | Distance transforms, Hungarian assignment, curve fitting   |
| `pandas`       | Saving tracking results to CSV                             |
| `pillow`       | Reading/writing TIFF frames and overlay images             |
| `ipywidgets`   | Interactive manual adjustment UI                           |
| `tifffile`     | Reading multi-page TIFF stacks                             |
| `imageio`      | Image I/O for ablation analysis section                    |

---

## Input Files

### TIFF Stack
A **multi-frame TIFF file** where each frame is one time point in the time-lapse.

- Frames should contain fluorescence signal from a tight junction marker (e.g., ZO-1-mScarlet-G).
- Images can be grayscale or RGB (automatically converted to float 0–1 range).
- The file is selected interactively via a file dialog when the first cell is run.

**Example path:**
```
/Users/apple/Desktop/Ian_lab/data/20250910_MMEC_NT1c5-ZO1-CB650-TJ.tif
```

---

## Configuration

Update the following parameters in **Cell 3** (Parameters section):

```python
# PIXEL_UM: physical pixel size in micrometers
PIXEL_UM = 0.25           # µm per pixel (update for each experiment)

# FRAME_INTERVAL_SEC: time interval between frames in seconds
FRAME_INTERVAL_SEC = 1204.738   # seconds per frame (update for each acquisition)

# OUT_DIR: output directory for all results (set automatically from STACK_PATH, or override here)
OUT_DIR = "/path/to/your/output_folder"
```

### Detection Parameters

| Parameter          | Default | Description |
|-------------------|---------|-------------|
| `SIGMA_UM`         | `0.5`   | Gaussian blur sigma in micrometers |
| `THRESHOLD_MODE`   | `"mean"`| Thresholding mode: `"mean"`, `"otsu"`, or `"p<N>"` (e.g., `"p90"`) |
| `MIN_INTENSITY`    | `0.0`   | Minimum intensity below which pixels are zeroed before binarization |
| `USE_RIDGE`        | `True`  | Whether to apply ridge (Sato) enhancement before thresholding |
| `MIN_OBJ_PIX`      | `25`    | Minimum object size in pixels to retain after binarization |
| `MIN_ARM_UM`       | `2.0`   | Minimum arm length (µm) required for a junction to be detected |
| `MIN_ANGLE_DEG`    | `30`    | Minimum angle between junction arms (degrees) |
| `CIRCLE_UM`        | `1.5`   | Radius (µm) of annotation circle drawn on overlays |
| `RADIUS_EXPAND_UM` | `1.0`   | Expansion radius (µm) used for center refinement |

### Tracking Parameters

| Parameter          | Default | Description |
|-------------------|---------|-------------|
| `MAX_SEARCH_UM`    | `5.0`   | Maximum search radius per frame (µm) |
| `MAX_MISS_FRAMES`  | `3`     | Number of consecutive missed frames before a track is lost |

---

## How It Works

### Step 1 — Detect Frame 0
1. The TIFF stack is loaded and all frames are converted to float [0, 1].
2. Frame 0 is preprocessed: Gaussian blur → optional ridge (Sato) enhancement → thresholding → binarization → cleaning → skeletonization.
3. A **multi-pass junction detector** finds skeleton branch points (degree ≥ 3 nodes). It traces each arm and filters detections by minimum arm length and inter-arm angle.
4. Detected junctions are annotated on an overlay image (`overlay_0000.png`) and stored as `pts0_px` in memory.

### Step 2 — Manual Adjust (Frame 0)
5. An interactive widget displays the frame-0 overlay. You can:
   - **Deselect by ID** — remove spurious detections by entering their IDs.
   - **Add coordinates** — paste pixel coordinates of missed junctions.
   - **Preview** the current selection.
   - **Save** — writes `pts0_manual_adjusted.csv` and `overlay_0000_manual.png`.

### Step 3 — Track Frames 1..N
6. For each subsequent frame, the detector re-runs detection in a local window around each track's last known position (search radius = `MAX_SEARCH_UM`).
7. A Kalman filter predicts each track's next position. A Hungarian (linear sum assignment) algorithm matches predictions to detections.
8. Tracks that miss detections for more than `MAX_MISS_FRAMES` consecutive frames are marked as lost.
9. Per-frame overlays are saved and all track data is written to CSV files.

### Optional — Laser Ablation Recoil Analysis (Cell 26–27)
10. Two tracked vertices flanking the ablation site are selected.
11. A bi-exponential model `d(t) = Δx₁·(1 - e^(-t/τ₁)) + Δx₂·(1 - e^(-t/τ₂))` is fitted to the post-ablation recoil distance over time.
12. Mechanical parameters (displacement amplitudes Δx₁, Δx₂ and time constants τ₁, τ₂) are extracted and saved.

---

## Outputs

All outputs are saved to `OUT_DIR` (automatically set to `<stack_filename>_out/` unless overridden).

| File | Description |
|------|-------------|
| `overlay_0000.png` | Frame 0 with auto-detected junctions annotated |
| `overlay_0000_manual.png` | Frame 0 with manually adjusted junctions annotated |
| `pts0_manual_adjusted.csv` | Pixel coordinates of frame-0 junctions after manual adjustment |
| `overlay_<NNNN>.png` | Per-frame overlays with tracked junction IDs (frames 1..N) |
| `detections_vertices_with_IDs.csv` | Full tracking table (canonical output) |
| `tracks_per_frame.csv` | Same data in per-frame format |
| `tracks_summary.csv` | Per-track summary statistics |
| `tracks_summary_with_RMSD.csv` | Per-track summary including RMSD displacement metric |

### `detections_vertices_with_IDs.csv` / `tracks_per_frame.csv` Columns

| Column | Description |
|--------|-------------|
| `FRAME` | Frame index (0-based) |
| `TRACK_ID` | Unique integer ID assigned to each tracked junction |
| `x_um` | X position in micrometers |
| `y_um` | Y position in micrometers |
| `dx_um` | X displacement from previous frame (µm) |
| `dy_um` | Y displacement from previous frame (µm) |
| `step_um` | Step distance from previous frame (µm) |
| `cum_um` | Cumulative path length (µm) |
| `t_sec` | Time in seconds |
| `net_um` | Net displacement from frame 0 (µm) |
| `inst_speed_um_per_sec` | Instantaneous speed (µm/s) |

### `tracks_summary_with_RMSD.csv` Columns

| Column | Description |
|--------|-------------|
| `TRACK_ID` | Track identifier |
| `n_points` | Number of frames this track was detected |
| `total_time_sec` | Total tracked duration (seconds) |
| `total_path_um` | Total path length (µm) |
| `net_disp_um` | Net displacement from start to end (µm) |
| `avg_speed_um_per_sec` | Average speed (µm/s) |
| `RMSD` | Root mean square displacement (µm) |

---

## Running the Notebook

Open `Tricell_Tracker_Notebook_v3 copy.ipynb` in Jupyter Notebook or JupyterLab and run the cells **in order**:

1. **Cell 3** — Set paths and calibration parameters. A file dialog will open to select your TIFF stack. Update `PIXEL_UM` and `FRAME_INTERVAL_SEC` before running.
2. **Cell 5** — Import all required packages. **Run after every kernel restart** before running any other cells.
3. **Cells 7–9** — Define helper functions (preprocessing, detection, tracking). Run all of these after imports.
4. **Step 1 (Cell 12)** — Detect tricellular junctions on frame 0. Review `overlay_0000.png` to check detections.
5. **Step 2 (Cell 14)** — Interactive manual adjustment. Use the widget to deselect wrong points or add missed ones, then click **Save**.
6. **Step 3 (Cell 16)** — Track junctions across all subsequent frames.
7. **Cell 21** *(optional)* — Export and compute final track summaries.
8. **Cell 24** *(optional)* — Inspect individual tracks interactively.
9. **Cells 26–27** *(optional)* — Laser ablation recoil analysis. Requires `DT_PER_FRAME_MIN` to be set.

> **Run order summary:** Cell 3 → Cell 5 → Cells 7–9 → Step 1 → Step 2 → Step 3

---

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| No file selected | File dialog did not appear or was closed | Rerun Cell 3; ensure a display/GUI environment is available |
| `AssertionError: frames not found` | Cells run out of order | Run Cell 3 and Cell 5 first, then Step 1 |
| Zero junctions detected on frame 0 | Threshold too high or ridge enhancement disabled | Try `THRESHOLD_MODE = "otsu"` or decrease `MIN_INTENSITY`; set `USE_RIDGE = True` |
| Too many spurious detections | Threshold too low | Use Step 2 to manually deselect, or increase `MIN_OBJ_PIX` or `MIN_ARM_UM` |
| Tracks lost after a few frames | Junctions move faster than `MAX_SEARCH_UM` | Increase `MAX_SEARCH_UM` in the parameters cell |
| `PIXEL_UM not defined` error in ablation cell | Cell 3 not run or PIXEL_UM not set | Rerun Cell 3 and ensure `PIXEL_UM` is defined |
| Overlay images not saved | `OUT_DIR` path does not exist or is not writable | Check `OUT_DIR` path; the directory is created automatically if it doesn't exist |

---

## Notes

- `PIXEL_UM` and `FRAME_INTERVAL_SEC` must be updated for **each experiment** as they depend on the microscope settings used during acquisition.
- The notebook uses `tkinter` for the file dialog. In headless environments (e.g., remote servers without a display), replace the file dialog with a direct path assignment: `STACK_PATH = "/path/to/your/file.tif"`.
- Step 2 (manual adjustment) is optional but recommended for high-accuracy downstream analysis. If skipped, Step 3 will use the raw auto-detections from Step 1.
- The ablation recoil analysis (Cells 26–27) is an independent module and requires `DT_PER_FRAME_MIN` to be defined in addition to `PIXEL_UM`.
