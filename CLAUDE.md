# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

PCTrack (Predict & Correct Tracking) is a real-time multi-object tracker for edge devices (Jetson Nano/TX2). It compensates for slow DNN detectors by predicting bounding boxes forward via a lightweight CNN, then correcting them with frame-difference analysis.

## Commands

```bash
# Run tracking (uses conda env "pctrack", default: Intersect, delay=30, method=ours)
python run.py

# Quick test run (10 windows instead of 30)
python test.py

# With arguments
python run.py --data OnRamp --delay 35 --method base

# Evaluate results
python eval.py --result_path runs/Intersect_ours/30/ --gt_path datasets/labels/Intersect_label/
```

## Architecture

### Data flow (per sliding window of `WINDOW`=delay frames)

```
load outdated labels → detectFeatures (Shi-Tomasi corners in box mask)
  → PDP/IDP (predict boxes forward via CNN or iterative optical flow)
  → [per frame] OpticalFlow → move_box → FDC (correct with frame diff)
  → det_newobj every 5 frames (DBSCAN on diff activation)
```

### Key modules

| File | Role |
|---|---|
| `run.py` | Entry point. Loads detection labels as "ground truth" (offline evaluation mode), loops over 30 windows of `WINDOW` frames. |
| `tracker.py` | Core `Tracker` class: feature detection, optical flow, box propagation, new-object detection. Also `frame_cache` for grayscale image window. |
| `corrector.py` | `fix_box()`: Sobel edge-cut + Pareto-frontier search to refine bounding boxes using frame difference. |
| `model/cnn.py` | `CNNPred`: 1D-conv model predicting future box positions from 3-frame trajectories. |
| `detector.py` | YOLOv5 wrapper (`Detector` class), not used in offline evaluation — for real-time deployment. |
| `utils/config.py` | CLI args (`--data`, `--delay`, `--method`, etc.), per-dataset params loaded from `data/{Dataset}.txt`, device selection, weight paths. |
| `utils/utils.py` | Box conversions (xywh/xyxy), IoU, `creat_mask`, `creat_matrix`, coordinate transforms, `smooth_points`, `linear_partition`. |
| `eval.py` | Precision/Recall/F1 computation against ground truth. |

### Config: two layers

1. **CLI args** (`utils/config.py`): `--data`, `--delay` (sets `WINDOW`), `--method` (`ours`/`base`), `--source`, `--label`, `--save`
2. **Per-dataset params** (`data/{Dataset}.txt`): `newobj_area`, DBSCAN thresholds, camera matrices (`P`, `P_inv`), CNN normalization (`mean`, `std`), `maxVars` for FDC triggering

CNN weights are loaded per-dataset from `weights/{Dataset}.pth`.

### Method variants

- **`ours`**: PDP (CNN prediction) + FDC (frame-diff correction) + new-object detection
- **`base`**: IDP (iterative optical flow only), no correction, no new objects

## Important patterns

- `self.prev_box`, `self.curr_box`, `self.pprev_box` are all xywh-format (center x, y, width, height) normalized to [0,1]. Conversion helpers are in `utils/utils.py`.
- `WINDOW` equals `--delay` (default 30). It represents both the detector latency and the sliding window size.
- Labels are read from text files (one per frame, `cls x y w h` per line, normalized coordinates).
- `cv2.goodFeaturesToTrack` can return `None` when no corners pass quality threshold. `cv2.calcOpticalFlowPyrLK` can return `None` with empty input points. Guard both.
- Frame indices in `run.py` use 1-based numbering (frame 000001.jpg = fid 1).
