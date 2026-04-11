---
title: Vision System (Veo Pipeline)
type: project
tags:
  - project/vision
  - cv
  - ml
  - python
created: 2026-04-11
status: active
location: "Veo Videos/"
---

# 🎥 Vision System (Veo Pipeline)

A Python computer-vision pipeline that turns Veo match videos into pitch heatmaps and tracking data. Independent from the wearable / Firebase pipeline — runs locally on a workstation, outputs files.

> [!info] Standalone today
> The vision pipeline is **not currently wired into Firebase**. It produces local files (heatmaps, tracking CSVs) for analysis. Integration with the rest of the platform is a future step.

---

## Stack

- **Python**
- **OpenCV** — frame grabbing, drawing, calibration
- **NumPy / Polars** — math, dataframes
- **Matplotlib** + **mplsoccer** — pitch rendering
- **scipy** — heatmap smoothing
- **Ultralytics YOLO** (`yolov8n.pt`) — player detection / tracking
- **SAM** — alternate, heavier tracker

```bash
pip install opencv-python polars numpy matplotlib mplsoccer scipy ultralytics
```

---

## The four-stage pipeline

`run_analysis.py` orchestrates four stages:

```
 1. interactive_homography.py  → calibration.json (one-time, per-camera)
 2. sam3_tracker.py            → player tracks (YOLO by default)
 3. homography_converter.py    → pixel (x,y) → pitch metres (x,y)
 4. pitch_3d_recreation.py     → 2D and 3D heatmaps
```

### Step 1 — Calibration (interactive)

```bash
python run_analysis.py --calibrate --video video.mp4
```

Opens a window. Arrow keys to navigate. Press `C` to enter calibration mode, click pitch landmarks, press `S` to save. Homography is stored in `calibration.json`.

### Step 2 — Full analysis

```bash
python run_analysis.py --video video.mp4
```

Runs YOLO on the first 5 minutes (configurable), converts pixel positions to pitch coordinates via the saved homography, writes heatmaps to `output/`.

### Both at once

```bash
python run_analysis.py --calibrate --video video.mp4
```

---

## Layout

```
Veo Videos/
├── run_analysis.py              (entry — orchestrates the four stages)
├── interactive_homography.py    (manual click calibration)
├── homography_converter.py      (pixel → pitch transform)
├── sam3_tracker.py              (tracking, YOLO backend)
├── pitch_3d_recreation.py       (2D/3D heatmap rendering)
├── heatmap_processor.py
├── heatmap_3d_visualizer.py
├── event_overlay_video.py       (draws events back onto video)
├── side_by_side.py              (comparison renders)
├── veo_loader.py                (loads Veo data into Polars)
├── example_usage.py
├── calibration.json             (saved homography per camera)
├── yolov8n.pt                   (YOLO weights)
├── output/                      (heatmaps + analysis output)
├── event_clips/
├── tactic_zone_results/         (TacticZone exports)
├── Advanced Results FIltered/
└── Kcc vids/                    (input videos)
```

---

## Data model — column conventions

`veo_loader.py` mirrors the **unravelsports / KloppyPolarsDataset** schema:

```
period_id, frame_id, timestamp,
x, y, z,
player_id (alias 'id'), team_id,
vx, vy, vz, v,
ax, ay, az, a
```

Sticking to these names makes it easy to interop with `unravelsports` (see [[Reference Repos]]) and other open soccer-analytics libraries.

---

## Reference / inspiration repos

Located in `Random Repos/`:

- **PassNet** — pass detection model
- **TacticZone** — tactical pattern extraction
- **ball-action-spotting** — ball event spotting
- **sn-spotting** — SoccerNet spotting models
- `model-006-0.864002.pth`, `ball_finetune_long_004-...zip` — trained weights / checkpoints

See [[Reference Repos]] for the full list.

---

## Typical workflow

1. Drop a Veo `.mp4` into a working folder
2. Calibrate once per camera setup (`--calibrate`)
3. Run analysis — produces heatmaps + position CSVs
4. Use `event_overlay_video.py` to render events back onto video for review
5. (Future) Push aggregated stats into Firebase to merge with wearable data

---

## 🛑 Hands off

This is an exploratory pipeline. There's not much that's "frozen" — but:

- ❌ The column names in `veo_loader.py` / `Column` class — they match unravelsports and downstream code expects them
- ❌ The contents of `calibration.json` for the camera setups currently in use (regenerating means clicking through every video again)

---

## Related

- [[Reference Repos]] — third-party model repos in `Random Repos/`
- [[AI Coach (text_ai_coach)]] — separate ML project
- [[Experimental Implementation]] — wearable-side data exploration
