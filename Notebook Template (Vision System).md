---
title: Notebook Template — Vision System
type: reference
tags:
  - project/vision
  - reference
  - cv
  - ml
created: 2026-04-21
---

# Notebook Template — Vision System

Canonical cell structure for any new notebook in `Cresento Website/yolov model/notebooks/`. **Copy these cells verbatim** — they handle the footguns that have bitten this project repeatedly. See [[Vision System (Veo Pipeline)]] for the overall pipeline.

> [!danger] Footguns this template exists to prevent
> - `pillow==11.*` pin with `--force-reinstall`. Colab ships pillow 12.x which has a `_Ink` import regression that surfaces misleadingly as `from ultralytics import YOLO` failing.
> - Do NOT upgrade torch. Colab already ships torch 2.10+ which satisfies everything. Forcing `-U "torch>=2.5"` breaks torchvision and pulls in CUDA 13.
> - Colab-vs-local path detection with `try: from google.colab import drive`. Notebooks must run unchanged on Colab and locally.
> - Read class IDs dynamically from `model.names` — never hardcode `CLASS_GK = 1`. Different fine-tuned checkpoints have different orderings.
> - Merge `goalkeeper`→`player` class before ByteTrack, then restore by bbox match after. ByteTrack IDs break if GK flips class mid-track.

---

## Cell 1 — Install (copy verbatim)

```python
#@title 1. Install
# Pillow 12.x has an internal import regression (_Ink from _typing). Pin to 11.x.
# --force-reinstall is mandatory because Colab's fresh runtime ships pillow 12.1.1
# which satisfies a naive <12 constraint, so pip won't downgrade without force.
# bitsandbytes is needed for int8 quantization of Gemma 4 on T4.
!pip install -q --force-reinstall "pillow==11.*"
!pip install -q -U transformers accelerate einops pycocotools bitsandbytes
!pip install -q ultralytics supervision opencv-python scikit-learn scipy

# Sanity check — flags torch/torchvision mismatch or stale pillow.
import torch, torchvision, PIL
print(f'torch={torch.__version__}, torchvision={torchvision.__version__}, pillow={PIL.__version__}')
if not PIL.__version__.startswith('11.'):
    print('\n⚠️  pillow not on 11.x. Runtime → Restart session, then skip this cell.')
tv_required = getattr(torchvision.version, 'torch_version', None)
if tv_required and tv_required not in torch.__version__:
    print('\n⚠️  torch/torchvision mismatch. Runtime → Disconnect and delete runtime.')
```

**If imports still fail after running this:**
- `_Ink` error → Runtime → Restart session (pip downgrade is on disk, but broken pillow is still in memory)
- torch/torchvision mismatch → Runtime → Disconnect and delete runtime (pip upgrades from a previous session persist across Restart)

Pip warnings about `gradio requires pillow<12.0` and `cuml-cu12 / cudf-cu12 / cuda-toolkit` are Colab's RAPIDS libs — harmless, we don't import any of them.

---

## Cell 2 — Configuration & Imports (copy verbatim, customize the `VIDEO_PATH` line)

```python
#@title 2. Configuration & Imports
import os, sys, json, re, math, time
from pathlib import Path
from collections import defaultdict, Counter, deque

import cv2
import numpy as np
import torch
from PIL import Image
from sklearn.cluster import KMeans
from ultralytics import YOLO
import supervision as sv

# ---------- Colab vs Local ----------
try:
    from google.colab import drive
    IN_COLAB = True
    drive.mount('/content/drive')
    PROJECT_ROOT = Path('/content/drive/MyDrive/Coding/yolov-world-model')
except ImportError:
    IN_COLAB = False
    PROJECT_ROOT = Path(r'D:\Downloads\React App\Cresento Website\yolov model')

# ---------- Paths ----------
MODELS_DIR = PROJECT_ROOT / 'models'
CLIPS_DIR  = PROJECT_ROOT / 'data' / 'clips'
OUTPUT_DIR = PROJECT_ROOT / 'outputs'
OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

# Model fallback chain — first existing wins. Add new fine-tuned weights to the front.
for candidate in ['yolo11m_roboflow_kcc_finetuned.pt',
                  'yolo11m_4class_finetuned.pt',
                  'yolov8x-worldv2.pt']:
    MODEL_PATH = MODELS_DIR / candidate
    if MODEL_PATH.exists():
        break
else:
    MODEL_PATH = 'yolo11m.pt'   # ultralytics will auto-download
print(f'Model: {MODEL_PATH}')

# Pick a specific clip, or fall back to the most recent one in CLIPS_DIR.
clip_files = sorted(CLIPS_DIR.glob('*.mp4'), key=lambda p: p.stat().st_mtime, reverse=True)
VIDEO_PATH = CLIPS_DIR / '<your-clip-name>.mp4'
if not VIDEO_PATH.exists() and clip_files:
    VIDEO_PATH = clip_files[0]
print(f'Video: {VIDEO_PATH}')

# ---------- Inference constants ----------
CONF_THRESH = 0.15
IMGSZ       = 640
BATCH_SIZE  = 20
SEED        = 42
np.random.seed(SEED); torch.manual_seed(SEED)

# Pre-loading all frames into RAM only works for short clips. A full match at
# 1080p30 needs ~1 TB of RAM and ~80 min of YOLO inference on T4 — Colab will
# OOM-kill the session. Trim to a manageable slice for iteration; set
# MAX_SECONDS = None for full clip only once the pipeline is validated.
MAX_SECONDS = 30

GRASS_HUE_TOL = 28

DEVICE = 'cuda' if torch.cuda.is_available() else 'cpu'
print(f'Project root : {PROJECT_ROOT}')
print(f'GPU          : {torch.cuda.is_available()}')
if torch.cuda.is_available():
    print(f'GPU name     : {torch.cuda.get_device_name(0)}')
```

---

## Cell 3 — Load Model, Read Frames, Run Detection + ByteTrack (canonical)

```python
#@title 3. Load Model, Read Frames, Run Detection + ByteTrack
assert VIDEO_PATH and Path(VIDEO_PATH).exists(), f'No video at {VIDEO_PATH}'

model = YOLO(str(MODEL_PATH))
CLASS_NAMES = model.names
CLASS_NAMES_INV = {v: k for k, v in CLASS_NAMES.items()}
PLAYER_CLS = CLASS_NAMES_INV.get('player')
GK_CLS     = CLASS_NAMES_INV.get('goalkeeper')
print(f'Classes: {CLASS_NAMES}')

# Read frames into memory (trimmed by MAX_SECONDS — see config cell)
cap = cv2.VideoCapture(str(VIDEO_PATH))
fps = cap.get(cv2.CAP_PROP_FPS) or 30
max_frames = int(fps * MAX_SECONDS) if MAX_SECONDS else None
frames = []
while True:
    ret, f = cap.read()
    if not ret: break
    frames.append(f)
    if max_frames and len(frames) >= max_frames: break
cap.release()
print(f'Loaded {len(frames)} frames @ {fps:.0f} fps'
      f'{" (trimmed — set MAX_SECONDS=None for full clip)" if max_frames else ""}')

# Batched detection + ByteTrack.
# Critical: merge goalkeeper→player before tracker, then restore GK by bbox match.
# Dropping this regresses track-ID stability on keepers.
tracker = sv.ByteTrack()
tracks = {'players': [], 'goalkeepers': [], 'referees': [], 'ball': []}

detections_list = []
for i in range(0, len(frames), BATCH_SIZE):
    batch = frames[i:i + BATCH_SIZE]
    detections_list.extend(model.predict(batch, conf=CONF_THRESH, imgsz=IMGSZ, verbose=False))

for frame_num, detection in enumerate(detections_list):
    det_sv = sv.Detections.from_ultralytics(detection)
    gk_mask = np.array([CLASS_NAMES.get(int(c), '') == 'goalkeeper' for c in det_sv.class_id])
    original_classes = det_sv.class_id.copy()
    if PLAYER_CLS is not None and GK_CLS is not None:
        det_sv.class_id[gk_mask] = PLAYER_CLS
    tracked = tracker.update_with_detections(det_sv)

    for group in ('players', 'goalkeepers', 'referees', 'ball'):
        tracks[group].append({})

    for det in tracked:
        bbox = det[0].tolist()
        cls_id = int(det[3]); track_id = int(det[4])
        cls_name = CLASS_NAMES.get(cls_id, '')
        if cls_name in ('player', 'goalkeeper'):
            is_gk = any(
                np.allclose(od, det[0], atol=1.0) and CLASS_NAMES.get(int(oc), '') == 'goalkeeper'
                for od, oc in zip(det_sv.xyxy, original_classes)
            )
            dest = 'goalkeepers' if is_gk else 'players'
            tracks[dest][frame_num][track_id] = {'bbox': bbox}
        elif cls_name == 'referee':
            tracks['referees'][frame_num][track_id] = {'bbox': bbox}
        elif cls_name == 'ball':
            tracks['ball'][frame_num][track_id] = {'bbox': bbox}
```

---

## What gets built on top

After cells 1–3, every subsequent cell iterates over `frames` (pre-loaded BGR ndarrays) and `tracks` (per-frame dicts grouped by role). No more `cv2.VideoCapture`, no more `model.predict` calls — detection is done, you just consume the results.

For the Falcon + Gemma role pipeline specifically, see [`tracking_falcon_gemma_roles.ipynb`](Cresento%20Website/yolov%20model/notebooks/tracking_falcon_gemma_roles.ipynb) — sections 4 onward show the palette priming, goal detector, role assigner, and render loop built on top of this canonical base.

---

## Sequential model loading (critical for T4 VRAM budget)

**YOLO, Gemma, and Falcon never need to coexist in VRAM.** They run in sequence:

| Stage | Model needed | Peak VRAM on T4 |
|---|---|---|
| Section 3 — batched detection | YOLO only | ~3 GB |
| Section 4 — palette priming | Gemma only (YOLO freed) | ~10 GB (E4B int8) |
| Section 5+ — keyframe correction | Falcon only (Gemma freed) | ~3 GB |

Enforce this by freeing models after each stage:

```python
# End of Section 3 — free YOLO before Gemma loads
del model, tracker
torch.cuda.empty_cache()
print(f'yolo unloaded — vram: {torch.cuda.memory_allocated()/1e9:.2f} GB')
```

```python
# End of Section 4 — free Gemma after palette
if UNLOAD_GEMMA_AFTER_PRIMING:
    del gemma_model, gemma_proc
    torch.cuda.empty_cache()
```

## Gemma 4 on T4 — use E2B, not E4B

Gemma 4 comes in two dense-model sizes:

| Variant | Effective params | With embeddings | fp16 VRAM | Fits T4? |
|---|---|---|---|---|
| **E2B** | 2.3B | 5.1B | ~6 GB | ✅ yes, with room for inference |
| E4B | 4.5B | 8B | ~12 GB | ❌ loads but OOMs during inference |

**bitsandbytes quantization is NOT reliably applied** to Gemma 4's multimodal `trust_remote_code` path (tested 2026-04-21). Passing `BitsAndBytesConfig(load_in_4bit=True, ...)` to `AutoModelForMultimodalLM.from_pretrained` still loads E4B at ~12 GB — same as fp16 with partial offload. The quant config seems silently ignored.

**On T4, just use E2B at fp16. No quantization needed.** Set:
```python
GEMMA_MODEL_ID = 'google/gemma-4-E2B-it'
GEMMA_QUANT_CONFIG = None
```

E4B becomes viable again on A100+ where fp16 fits comfortably. Quality difference is modest on the palette-JSON task.

```python
# Gemma loader:
gemma_proc = AutoProcessor.from_pretrained(GEMMA_MODEL_ID)
gemma_model = AutoModelForMultimodalLM.from_pretrained(
    GEMMA_MODEL_ID,
    dtype='auto',
    device_map='auto',
    quantization_config=GEMMA_QUANT_CONFIG,   # int8 on T4, None on A100
)
```

This makes the same notebook run on free T4 and Pro A100 without edits.

---

## Common mistakes

- **Hardcoding class IDs** — always derive from `CLASS_NAMES_INV` because fine-tuned weights differ in class ordering.
- **Re-reading the video** in downstream cells — `frames` is already in memory; use it.
- **Re-running YOLO** in downstream cells — `detections_list` and `tracks` are already populated; consume them.
- **Passing `Path` objects to cv2** — wrap with `str()`. OpenCV's Python bindings accept both on recent versions but older wheels don't.
- **Forgetting to restart after a failed install** — `pip install` changes disk, but running Python keeps the old modules loaded. Always Restart after a pillow/torch fix.
- **Duplicate cells after pasting an update** — when pasting a cell update into Colab, **delete the old cell first**, don't just add the new one below it. Colab runs cells top-to-bottom; a stale cell earlier in the notebook referencing an old variable (`YOLO_WEIGHTS`, `yolo`, etc.) will fire first and `NameError` before your new cell even runs. Telltale sign: two cells in a row with the same section header `## 3. ...`.
- **Pre-loading all frames for a long clip** — the canonical Section 3 reads every frame into `frames = []` before detection. This only works for clips up to a few minutes on Colab (RAM cap: 12 GB). A 90-minute full match at 1080p30 needs ~1 TB of RAM and ~80 min of YOLO inference — the session will OOM-kill. Always set `MAX_SECONDS = 30` (or similar) while iterating. Only bump it up once the pipeline is validated end-to-end.
- **Falcon Perception OOM at 1920×1080** — Falcon's KV cache scales with image tokens. At native 1080p a single frame allocates ~5 GB of KV cache, which OOMs on T4 even with other models unloaded. Always pass `max_dimension=640` (and `max_new_tokens=512`) to `falcon_model.generate()`. Falcon's default `max_dimension=1024` is also too high on T4.
- **Falcon Perception FlexAttention fails on T4** — T4 (Turing, compute capability 7.5) has 64 KB of SRAM per SM. Falcon's `torch.compile`'d FlexAttention kernel needs 80 KB, so the kernel launch fails with `InductorError: out of resource: triton_tem_fused_flex_attention_0 Required: 81920 Hardware limit: 65536`. Pass `compile=False` to `falcon_model.generate()` on any pre-Ampere GPU. Detect via `torch.cuda.get_device_capability(0) < (8, 0)`. Eager FlexAttention is ~2–3× slower but actually runs. A100/L4/H100 (Ampere+) can use `compile=True`.
- **bitsandbytes weights don't fully free on `del`** — `del gemma_model; torch.cuda.empty_cache()` often leaves several GB of VRAM occupied when Gemma was loaded with `BitsAndBytesConfig(load_in_8bit=True)`. Belt-and-suspenders cleanup before Falcon loads:
  ```python
  import gc
  gc.collect()
  torch.cuda.empty_cache()
  torch.cuda.synchronize()
  ```
  If allocated VRAM after this is still > 1 GB when no model should be loaded, only `Runtime → Restart session` will fully reclaim it.

---

## Related

- [[Vision System (Veo Pipeline)]] — the overall pipeline
- [[Proposal - Falcon Perception + Gemma Role and Team Assignment]] — the proposal this template supports
