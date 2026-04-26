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
location: "Veo Videos/, Cresento Website/yolov model/notebooks/"
---

# 🎥 Vision System (Veo Pipeline)

A Python computer-vision pipeline that turns Veo match videos into pitch heatmaps, per-player tracking, team assignment, ball possession, and a live 2D minimap. Independent from the wearable / Firebase pipeline — runs in **Google Colab** (lead's personal account) or locally, outputs video + CSVs.

> [!info] Standalone today
> The vision pipeline is **not currently wired into Firebase**. It produces local files (heatmaps, tracking CSVs, rendered video) for analysis. Integration with the rest of the platform is a future step.

> [!info] Two generations live side-by-side
> - **Gen 1 (heatmap-first, `Veo Videos/`)** — YOLOv8n + homography + mplsoccer heatmaps. Schema-compatible with unravelsports. Documented in full below.
> - **Gen 2 (tracking-first, `Cresento Website/yolov model/notebooks/`)** — Fine-tuned YOLOv11m 4-class (ball/goalkeeper/player/referee) + ByteTrack + KMeans jersey-color team assignment + per-frame pitch-keypoint homography + live 2D minimap render. Jupyter-based, Colab-hosted. See [[#Gen 2 — the notebook pipeline]] below.
>
> Both coexist. The 4-class detector is the current state of the art; the Gen-1 heatmap stage still runs on its outputs.

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

## Gen 2 — the notebook pipeline

Located at `Cresento Website/yolov model/notebooks/`. Runs in **Google Colab** on the lead's account — any proposal must fit T4 (free) or A100 (Pro) VRAM budgets.

### Detection: fine-tuned YOLOv11m, 4 classes

Trained in [`finetune_yolov11m_4class.ipynb`](Cresento%20Website/yolov%20model/notebooks/finetune_yolov11m_4class.ipynb) on a 3-dataset fusion:

- **X_321_5star** (~9,368 images) — large general football set
- **Roboflow `football-players-detection` v19** (5× oversampled)
- **KCC_partial_2** CVAT annotations on Veo footage (5× oversampled)

Classes: `0: ball, 1: goalkeeper, 2: player, 3: referee`. Side referees merged into a single referee class; staff excluded. 100 epochs, imgsz=640, AdamW lr0=0.001, cosine LR.

Expected weight: `yolo11m_4class_finetuned.pt`. Base `yolov8x-worldv2.pt` (YOLO-World open-vocab) also present in `models/`.

### Tracking: supervision.ByteTrack

Standard ByteTrack, batched 20-frame inference. Goalkeepers are **merged into the player class for tracking** to keep IDs continuous across the rare frames where GK is mislabeled, then the original class is restored at render time (thicker ellipse, "GKn" label).

### Team assignment: dual-mask KMeans (`TeamAssigner`)

Five-hypothesis pipeline, documented exhaustively in the `debug_*` notebooks:

1. **H3 — Grass hue detection.** Sample HSV median of central strip (rows 50–75%, cols 25–75%) across 8 frames, sat > 25. Validity check only — not used as a mask.
2. **H1/H2/H4 — Per-player dominant color.** Top-half bbox crop → dual-mask remove (grass hue ±28°, warm background ±15°) → KMeans(k=2) → larger cluster = jersey. Fallback to mean if 5–30 pixels; skip if <5. Safety swap if selected hue sits within 12° of grass.
3. **H5 — Team-level clustering.** Collect 6D (dominant + secondary) vectors from ~20 evenly-spaced frames, bboxes ≥ 30×40 px, area-weighted ≤ 7 samples/player, drop >2σ outliers, KMeans(k=2) in **BGR** (LAB/HSV were tested; BGR chosen for silhouette/robustness balance).

Known failure modes (from [`debug_jersey_color_instrumented.ipynb`](Cresento%20Website/yolov%20model/notebooks/debug_jersey_color_instrumented.ipynb)):

- H1 contaminates white jerseys when stadium lighting is orange/yellow
- H2 picks shadow/skin under poor lighting
- H3 grass hue swings 10–15° frame-to-frame
- H5 BGR separation weaker than LAB for some kits

### Pitch keypoints + 2D minimap

Separate YOLOv11m trained in [`train_pitch_keypoints.ipynb`](Cresento%20Website/yolov%20model/notebooks/train_pitch_keypoints.ipynb) on the 32 FIFA-standard pitch landmarks (corners, goal/penalty areas, center circle). Per-frame homography: detected keypoints → 105×68 m pitch space. Minimap is a 525×340 px canvas, team 1 red, team 2 orange, GK cyan, referees yellow. Position smoothing α=0.15. Rendered side-by-side with tracked video.

### Ball possession

`PlayerBallAssigner`: closest-player-feet to ball center within **70 px**, assigns that player's team. Missing ball frames linearly interpolated + velocity-extrapolated. HUD shows rolling team 1/2 possession %.

### Entry point

[`tracking_team_and_2d_minimap.ipynb`](Cresento%20Website/yolov%20model/notebooks/tracking_team_and_2d_minimap.ipynb) — the current "master" notebook. Produces `tracked_[clip].mp4` with ellipses, track IDs, team colors, ball marker, possession HUD, and minimap panel.

### Experimental — Falcon-only 5-class pipeline

[`tracking_falcon_gemma_roles.ipynb`](Cresento%20Website/yolov%20model/notebooks/tracking_falcon_gemma_roles.ipynb) — current prototype. **Falcon Perception alone**, no YOLO / Gemma / ByteTrack. Five fixed grounded-detection queries per keyframe: `player wearing white jersey`, `player not wearing white jersey`, `goalkeeper`, `referee`, `ball`. Query strings get edited per clip to match actual kit colors. See [[Proposal - Falcon Perception + Gemma Role and Team Assignment]] for the pivot rationale.

Earlier design (Gemma palette + YOLO + ByteTrack + RoleAssigner + Falcon correction) is documented in the proposal note and preserved in git history — re-introduce only if per-frame smooth tracking becomes a hard requirement.

Falcon wired in via `AutoModelForCausalLM.from_pretrained('tiiuae/Falcon-Perception', trust_remote_code=True, dtype='auto', device_map={'':'cuda:0'})`. On Turing T4 pass `compile=False` + `max_dimension=640` to `generate()`. Requires PyTorch ≥ 2.5 (FlexAttention).

---

## 🛑 Hands off

- ❌ The column names in `veo_loader.py` / `Column` class — they match unravelsports and downstream code expects them
- ❌ The contents of `calibration.json` for the camera setups currently in use (regenerating means clicking through every video again)
- ❌ The 4-class label ordering (`0:ball, 1:goalkeeper, 2:player, 3:referee`) — baked into the fine-tuned weights and every downstream notebook's class-id math
- ❌ The GK-→-player class merge for ByteTrack, then restore at render — dropping this regresses track-ID stability on keepers

## Setting up a new notebook

See [[Notebook Template (Vision System)]] for the canonical install cell, Colab-vs-local config cell, and model + frames + detection + ByteTrack cell. Copy those three cells verbatim into any new notebook in `Cresento Website/yolov model/notebooks/` — they handle the pillow/torch/Colab footguns this project has run into repeatedly.

## Notebook path convention

All notebooks in `Cresento Website/yolov model/notebooks/` use a shared Colab-vs-local path block (see `tracking_falcon_gemma_roles.ipynb` §2 for the canonical version):

```python
try:
    from google.colab import drive
    IN_COLAB = True
    drive.mount('/content/drive')
    PROJECT_ROOT = Path('/content/drive/MyDrive/Coding/yolov-world-model')
except ImportError:
    IN_COLAB = False
    PROJECT_ROOT = Path(r'D:\Downloads\React App\Cresento Website\yolov model')

MODELS_DIR = PROJECT_ROOT / 'models'
CLIPS_DIR  = PROJECT_ROOT / 'data' / 'clips'
OUTPUT_DIR = PROJECT_ROOT / 'outputs'
```

Model weights live in `MODELS_DIR`, match clips in `CLIPS_DIR`, rendered outputs in `OUTPUT_DIR`. New notebooks should copy this block verbatim rather than hardcoding paths — lets the same file run on the lead's Colab account and locally on Windows without edits.

---

## 🧪 Open proposals

- [[Proposal - Falcon Perception + Gemma Role and Team Assignment]] — replace the H1–H5 KMeans stack with a VLM-primed palette + grounded-detection flow, sized for Colab

---

## Related

- [[Reference Repos]] — third-party model repos in `Random Repos/`
- [[AI Coach (text_ai_coach)]] — separate ML project
- [[Experimental Implementation]] — wearable-side data exploration
