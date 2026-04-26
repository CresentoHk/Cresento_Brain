---
title: "Proposal — Falcon Perception + Gemma for role + team assignment"
type: proposal
tags:
  - project/vision
  - proposal
  - cv
  - ml
  - exploration
created: 2026-04-21
status: simplified — see 2026-04-21 pivot at top
---

> [!info] 2026-04-21 pivot — Falcon-only
> The original plan below (Gemma palette + YOLO detection + ByteTrack + RoleAssigner + Falcon correction) hit sustained practical issues on Colab T4:
> - Gemma 4 E4B doesn't fit T4; bitsandbytes quantization is silently ignored by Gemma 4's `trust_remote_code` multimodal path
> - Gemma 4 E2B fits but adds complexity for marginal gain over direct grounded detection
> - Every added model/stage added a stale-cell / VRAM-residue / version-mismatch failure mode
>
> **Current approach in [`tracking_falcon_gemma_roles.ipynb`](Cresento%20Website/yolov%20model/notebooks/tracking_falcon_gemma_roles.ipynb) is Falcon-only.** Five fixed grounded-detection queries per keyframe:
> ```
> player wearing white jersey       → team_a
> player not wearing white jersey   → team_b
> goalkeeper                         → goalkeeper
> referee                            → referee
> ball                               → ball
> ```
> No YOLO, no Gemma, no ByteTrack, no palette, no rolling vote. Queries are hardcoded strings that get edited per clip to match the actual kit colors. Output is a keyframe-only annotated video (boxes "pop" at each keyframe interval).
>
> Tradeoffs:
> - ✅ Drastically simpler — ~150 lines total, 5 cells end-to-end
> - ✅ Only one model loaded at a time (~3 GB VRAM, fits T4 trivially)
> - ✅ Works without any of the stale-cell / VRAM / quantization footguns
> - ❌ No per-frame detections — only keyframes. Smooth tracking is future work.
> - ❌ Query strings are clip-specific and manual
> - ❌ Falcon throughput caps real-time playback speed
>
> If smooth per-frame tracking matters later, re-introduce YOLO + ByteTrack for per-frame bboxes and use Falcon's keyframe output as the role-label source, Hungarian-matched onto tracks. That's the original plan below, executed after the Falcon-only version is validated.

---

# Proposal — Falcon Perception + Gemma for role + team assignment

Replace the H1–H5 KMeans / dual-mask jersey-color stack in [[Vision System (Veo Pipeline)|the Gen-2 notebook pipeline]] with a two-stage VLM flow: **Gemma once per clip to derive the team/role palette, Falcon Perception (or the YOLO-World fallback) at keyframes to re-label roles and teams using natural-language prompts built from that palette.**

> [!warning] Status: exploring
> No code written. Needs a Colab feasibility spike before committing — specifically Falcon Perception VRAM / throughput on T4 (free tier) and A100 (Pro). The Gemma half is cheap enough to ship standalone regardless.

> [!info] Runs on Colab
> All of this must fit the lead's personal Colab account. **T4 (16 GB VRAM) on free, A100 40 GB on Pro when available.** Any plan that needs an H100 is out of scope.

---

## Why

The Gen-2 pipeline ships a fine-tuned YOLOv11m that already emits `{ball, goalkeeper, player, referee}` — so this is not a "we have no classes" problem. The pain is two levels up:

1. **Goalkeeper / referee misclassification** at inference time. YOLO learns *appearance* but goalkeeper is really a *positional + kit-distinct* concept. Same for referees vs. coaching staff in kit-adjacent colors.
2. **Team color assignment is fragile.** Five documented failure hypotheses (H1–H5) in [`debug_jersey_color_instrumented.ipynb`](Cresento%20Website/yolov%20model/notebooks/debug_jersey_color_instrumented.ipynb): white jerseys contaminated by warm stadium backgrounds, KMeans picking shadow/skin, grass hue drifting 10–15° per frame, BGR-vs-LAB color-space tradeoffs. Each new venue is a new debugging session.

A VLM that reads the image once ("team A wears red, goalkeeper wears green, referee in black") bypasses every one of H1–H5. The question is whether the compute fits on Colab.

---

## Architecture

```
  Stage A — once per clip (cheap, always runs)
  ──────────────────────────────────────────────
  Sample 3–5 frames spanning the clip
       │
       ▼
  Gemma 3 4B vision (Colab T4, ~8 GB VRAM fp16, ~3 GB int4)
       │
       ▼
  JSON palette (cached alongside calibration.json):
    { team_a: { jersey, shorts, gk_jersey },
      team_b: { jersey, shorts, gk_jersey },
      referee: { jersey } }

  Stage B — every frame (existing, unchanged)
  ──────────────────────────────────────────────
  YOLOv11m 4-class + supervision.ByteTrack  →  boxes + stable track IDs

  Stage C — every N frames (NEW, optional)
  ──────────────────────────────────────────────
  Falcon Perception, prompt built from Stage A palette:
    "Detect: goalkeeper in green, player in red, player in blue,
     referee in black, ball"
       │
       ▼
  Role-corrected + team-labeled boxes
       │
       ▼
  Hungarian match against ByteTrack IDs → overwrite team_id / class_id
  on the track (not the frame)

  Downstream (existing, unchanged)
  ──────────────────────────────────────────────
  PlayerBallAssigner (70 px) → possession HUD → minimap render
```

Crucially, Stage C **corrects tracks, not frames.** Falcon runs at keyframes only. ByteTrack still owns continuity.

---

## What each pain point maps to

| Current failure (H-series) | Addressed by |
|---|---|
| GK mis-classed as player (YOLO) | Stage C (Falcon grounded on "goalkeeper in green") |
| Referee confused with staff | Stage C |
| H1 — white jersey contaminated by warm background | Stage A (VLM reads palette directly, no masking) |
| H2 — KMeans picks shadow/skin | Stage A |
| H3 — grass hue drifts across frames | Stage A (bypassed; no grass detection needed) |
| H5 — BGR vs LAB team separation | Stage A (bypassed; no color-space clustering) |
| Players lost across long occlusions | **Not addressed.** Still a tracker-level problem. Needs BoT-SORT or a re-id embedder, separate work. |

---

## Colab sizing — revised 2026-04-21 (Gemma 4 E4B + sequential loading)

**Model choices locked in:**
- **Gemma 4 E4B-it** (`google/gemma-4-E4B-it`) — 4.5B effective / 8B params with embeddings
- **Falcon Perception 600M** (`tiiuae/Falcon-Perception`)
- **YOLOv11m 4-class** fine-tuned

All three coexisting in VRAM on T4 (~16 GB) is not feasible — Gemma 4 E4B fp16 alone is ~16 GB. The solution: **sequential loading**. None of the three models need to coexist, so we free each one before the next loads.

| Stage | In VRAM | Peak (T4) |
|---|---|---|
| Section 3 — batched detection | YOLO only | ~3 GB |
| Section 4 — palette priming | Gemma 4 E4B int8 | ~10 GB |
| Section 5+ — keyframe correction | Falcon | ~3 GB |

Gemma is loaded with int8 quantization (via `bitsandbytes`) on any GPU with < 30 GB VRAM, and at fp16 on A100/H100. Decision is automatic via a VRAM probe in the config cell:

```python
GEMMA_QUANT_CONFIG = (BitsAndBytesConfig(load_in_8bit=True)
                      if 0 < _total_vram < 30e9 else None)
```

Same notebook runs on free T4 and Pro A100 with no manual edits.

YOLO is freed at the end of Section 3 (`del model, tracker; torch.cuda.empty_cache()`). Gemma is freed after Section 4 via the existing `UNLOAD_GEMMA_AFTER_PRIMING = True` flag.

**Falcon throughput on T4** (600M multimodal, autoregressive decode):
- Image encoding: ~50–100 ms
- Short structured response (~50 tokens): ~0.5–1 s
- **~1–2 fps**, which is real-time at keyframe intervals of 1–2 s
- A100 roughly 5× faster; not required

**Framing:** Phase 0 (is Falcon feasible?) largely collapses into Phase 1 — there's no VRAM risk.

### Streaming frame-by-frame (mandatory)

Do not materialize the whole video as a tensor. Bounded state, per-frame `cv2.VideoCapture` read, expensive models gated by `frame_idx % keyframe_interval`:

```python
cap = cv2.VideoCapture(video_path)
fps = cap.get(cv2.CAP_PROP_FPS)

track_history = defaultdict(list)  # track_id -> rolling labels
track_colors  = defaultdict(list)  # track_id -> recent LAB samples
goal_cache    = RollingGoalDetector()

frame_idx = 0
while True:
    ret, frame = cap.read()
    if not ret: break

    goal_bbox = goal_cache.update(frame)                  # ~2 ms always
    result    = yolo(frame, conf=0.3)[0]                  # ~30 ms always
    det       = tracker.update_with_detections(result)

    for bbox, tid in zip(det.xyxy, det.tracker_id):
        track_colors[tid].append(mean_lab(frame, bbox))

    # grounded re-labeling at keyframes only
    if frame_idx % int(fps * 1.5) == 0:                   # every 1.5 s
        falcon_out = falcon.detect(frame, prompt=palette_prompt)
        apply_role_corrections(falcon_out, det, track_history)

    frame_idx += 1
cap.release()
```

### Model roles — finalized split

| Model | Runs | Job |
|---|---|---|
| Gemma 3 4B | Once per clip (3–5 frames) | Output palette JSON with role labels + color names |
| Falcon Perception 600M | Every ~1.5 s keyframe | Grounded detection with palette-derived prompts |
| OpenCV goal detector | Every frame | Goal bbox + which-half signal |
| YOLOv11m + ByteTrack | Every frame | Detections + stable track IDs |
| KMeans residual | Per-track (online) | Flag outliers as GK/referee candidates |

---

## Proposed rollout (three phases, each independently shippable)

### Phase 0 — Colab feasibility spike (~1 day)

- Boot Falcon Perception in a fresh Colab T4 notebook.
- Quantize to int4, load, run 10 frames of a KCC clip with a 6-role prompt.
- Record: VRAM peak, frames/sec, box quality vs YOLOv11m baseline.
- Kill-switch: if T4 int4 fails AND A100 gives < 2 fps, drop Falcon and proceed with YOLO-World in Stage C.

### Phase 1 — Gemma palette (ship regardless of Phase 0 outcome)

- New notebook: `gemma_palette_primer.ipynb`.
- Input: clip path + 3–5 sampled frames. Output: palette JSON, cached to `calibration.json` next to the existing homography.
- Integration point in [`tracking_team_and_2d_minimap.ipynb`](Cresento%20Website/yolov%20model/notebooks/tracking_team_and_2d_minimap.ipynb): wire `TeamAssigner` to read the palette first and use the H1–H5 path only as a fallback.
- Expected win: eliminates H1 and H2 for most clips, reduces H3/H5 to sanity checks.

### Phase 2 — grounded re-labeling (Falcon or YOLO-World)

- New notebook: `grounded_role_corrector.ipynb`.
- Every 0.5–1 s keyframe, run the grounded detector with a palette-derived prompt. Hungarian-match to live ByteTrack IDs. Update the per-track `team_id` / `class_id` by majority vote across keyframes.
- Expected win: GK and referee role correction, robust team assignment under weird lighting.

### Phase 3 — evaluation + decision

- Hand-label 200 frames across 3 KCC clips (team, role).
- Compare Gen-2 baseline vs Phase 1 vs Phase 2 accuracy on the held-out set.
- If Phase 1 alone closes most of the gap, Phase 2 is deferred.

---

## What this does NOT solve

- **Track-level player loss across long occlusions.** ByteTrack alone doesn't re-identify a player who leaves the frame for 3 seconds. That's a separate proposal. Note: the palette DOES let a returning player get the correct team label on their new track_id for free — only identity continuity is unsolved.
- **Ball occlusion / small-object loss.** VLMs are poor at small-object localization; Gemma / Falcon will not help find the ball when it's ~10–20 px or occluded. Fix is elsewhere: dedicated higher-res ball model, better interpolation, multi-scale inference.
- **Pitch keypoint drift on Veo's wide-angle edges.** Separate calibration concern.

---

## Future work — motion-signature re-identification

**Idea (owner: lead):** cross-reference IMU data from the shin-guard pads with per-player motion in video. At each instant, the pad reports speed / sprint / acceleration; the tracker reports the same from pitch coordinates. When a player returns to frame after an occlusion, match their video motion signature to the pad motion signature to recover identity across track breaks.

**Why it's promising:**

- Both signals already exist. Shin-guard IMU is logged at 20 Hz; video tracking produces per-frame pitch-space velocity.
- Motion signatures are high-dimensional enough to disambiguate 22 players even with a few seconds of data. Sprint timing + acceleration bursts are strongly individual.
- It bypasses the need to train / host a visual re-id embedder (OSNet, CLIP-ReID), which would otherwise be a separate Colab model load.

**Why it's deferred:**

- Requires the pad-side wearable pipeline to be producing timestamped session data in a queryable form during the same session the Veo footage covers — that's a cross-project sync concern touching [[Firebase Backend]], [[StatsEngine Cross-Platform]], and the Veo pipeline.
- Needs a clock-sync mechanism between pad UNIX timestamps and video frame timestamps (probably a clap / visual trigger at match start).
- No value until Phase 1 (palette) and Phase 2 (role correction) are landed — there's no point re-identifying a player whose team label is wrong anyway.

**Not part of this proposal's scope.** Flagged here so it doesn't get lost; spin out as its own proposal when the Phase 1/2 work is stable.

---

## Risks / unknowns

- **Falcon model size unverified.** Repo summary doesn't state parameter counts cleanly — Phase 0 spike clarifies.
- **Colab runtime limits.** Long keyframe passes risk the 12-hour cap on free Colab. Keep per-clip inference under ~30 min by batching keyframes aggressively.
- **Palette drift mid-clip.** If a team changes kit at halftime (rare but it happens), Stage A's one-off palette misses it. Mitigation: re-sample Gemma at minute 45.
- **License / provider split.** Gemma is Google, Falcon is TII. Both are open-weights as of early 2026; verify license fit before any production usage beyond personal Colab experimentation.

## Colab install gotchas (learned 2026-04-21)

> [!danger] Do NOT force-upgrade `torch` on Colab
> The Falcon Perception README says `pip install "torch>=2.5"`. Taking that literally on Colab is a trap: `-U "torch>=2.5"` upgrades torch 2.10 → 2.11 and breaks the preinstalled torchvision/torchaudio (both pin to `torch==2.10`). You also pick up a CUDA 13 toolkit that conflicts with the installed RAPIDS libraries (`cudf-cu12`, `cuml-cu12`).
>
> **Fix:** Colab's preinstalled torch is already 2.10+, which satisfies `>=2.5`. Don't upgrade it.

> [!danger] Pin `pillow==11.*` (not `<12.2`)
> Pillow 12.x has an internal import regression where `PIL.ImageText` tries to `from ._typing import _Ink` but that symbol isn't exported. The bug is **across the entire 12.x line — 12.0, 12.1, 12.2 all break.** Initial diagnosis of "12.2 only" was wrong.
>
> Imports fail with `ImportError: cannot import name '_Ink' from 'PIL._typing'` — surfaces misleadingly as `from ultralytics import YOLO` because ultralytics imports PIL transitively.
>
> **Fix:** pin to the last stable 11.x:
> ```
> !pip install -q --force-reinstall "pillow==11.*"
> ```
> `--force-reinstall` is mandatory because Colab's fresh runtime ships pillow 12.1.1, and a plain `pip install "pillow<12"` won't downgrade an already-satisfying version on every pip version.
>
> **If you already hit the error mid-session:** Runtime → Restart session. The bad pillow stays loaded in memory even after pip downgrades on disk. (Restart is enough here — no factory reset needed.)

> [!warning] Restart vs Factory Reset
> **Runtime → Restart session** only clears Python imports. Previous `pip install` changes remain on disk.
>
> **Runtime → Disconnect and delete runtime** (factory reset) gives a completely fresh Colab image. Use this one when a previous cell force-upgraded torch/torchvision/CUDA and the mismatch is baked into `/usr/local/lib/python3.12/dist-packages/`. Restart alone won't recover.
>
> The install cell now prints a loud warning if torch and torchvision disagree, to catch this without a manual inspection.

> [!info] Ignore these pip warnings
> After a clean install on Colab, the pip resolver will still grumble:
> - `gradio requires pillow<12.0` — gradio is a Colab preinstall we don't import.
> - `cuml-cu12 / cudf-cu12 / cuda-python requires cuda-toolkit==12.*` — RAPIDS libs we don't import.
> Both are harmless.

Canonical install cell for any notebook in this pipeline:

```python
!pip install -q -U transformers accelerate einops pycocotools "pillow<12.2"
!pip install -q ultralytics supervision opencv-python scikit-learn scipy
```

The `cudf-cu12 / cuml-cu12 / cuda-toolkit` dependency-resolver warnings that pip prints are safe to ignore — those are Colab's RAPIDS libs that we don't import.

---

## Sensor-fusion refinement — goal detection + KMeans outliers + Gemma as tie-breaker

**Insight (2026-04-21, lead):** Gemma doesn't need to be the primary classifier at all. Three cheap signals already in the pipeline (goal geometry, YOLO class head, KMeans cluster residuals) can vote on role, with Gemma reduced to a one-shot labeling call for the outlier colors.

### Signal 1 — goal detection via classical CV

Football goalposts are a near-ideal classical-CV target: white-painted, rectangular (3:1 aspect), two verticals + one horizontal, stationary across consecutive frames. Roughly 40 lines of OpenCV:

```python
def detect_goal(frame):
    hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
    white = cv2.inRange(hsv, (0,0,200), (180,40,255))
    edges = cv2.Canny(white, 50, 150)
    lines = cv2.HoughLinesP(edges, 1, np.pi/180, 80, minLineLength=60, maxLineGap=10)
    # accept "one vertical + one horizontal meeting at ~90°" as a partial goal
    return fit_goal_rect(verticals, horizontals)
```

Smooth across 5–10 frames (goal doesn't move); filter pitch sidelines by aspect ratio; net is grey so the low-saturation high-value mask isolates the posts cleanly.

**What this unlocks:**
- **Half awareness.** `half = "left" if goal_cx < w/2 else "right"` — tells you which GK should be on screen.
- **GK proximity prior.** The player closest to the goal bbox is almost certainly the goalkeeper. Independent signal, doesn't depend on color or YOLO class.

### Signal 2 — KMeans cluster outliers are GK + referee *by FIFA rule*

FIFA Law 4: GK kit must be distinguishable from both teams and the referee; referee kit must be distinguishable from both teams. So **the two largest KMeans clusters are by definition the teams**, and any track whose color sits far from both centroids is guaranteed to be GK or referee.

This is already being computed in H5 — the current code just doesn't extract the residual. Turning residuals into an explicit role-candidate set is essentially free.

```python
for tid, color in zip(track_ids, track_colors):
    d_a = np.linalg.norm(color - centroids[0])
    d_b = np.linalg.norm(color - centroids[1])
    if min(d_a, d_b) < TEAM_THRESHOLD:
        team = "a" if d_a < d_b else "b"
    else:
        role_candidates.add(tid)  # GK or referee — disambiguate next
```

### Signal 3 — disambiguation table

Combining signals for each role-candidate:

| Signal | Goalkeeper | Referee |
|---|---|---|
| Distance to goal bbox | very small, persistent | varies across pitch |
| Pitch-space motion pattern | lateral bursts, low mean speed | continuous running |
| YOLO class head (when it fires) | `goalkeeper` | `referee` |
| Expected count per match | exactly 2 | 1 main + up to 2 assistants |

Three of four agreeing is strong. No VLM call needed.

### Gemma's new role — labeling, not classifying

Gemma is no longer per-clip palette prep. It becomes a **one-shot outlier labeler**:

> "Here are 3–4 outlier jersey colors extracted from this match. For each (shown as a cropped patch or a numbered set-of-mark overlay), tell me if it is team_a_gk, team_b_gk, main referee, or assistant referee."

One call per clip, output is a color→role dictionary. After that, every per-track decision is classical distance math. Gemma may be skippable entirely for recurring fixtures where kit colors are already known.

### Revised architecture

```
Per-clip (once, optional):
  Gemma → label the outlier colors with role names

Per-frame (cheap, all classical):
  1. Goal detector (Hough)         → goal_bbox, half
  2. YOLOv11m 4-class              → detections
  3. ByteTrack                     → track IDs

Per-track (on birth, updated on drift):
  4. Mean LAB color
  5. KMeans(2) → team centroids
  6. Distance test:
     - near centroid A → team A
     - near centroid B → team B
     - outlier → role-candidate

Per-outlier (rare, ambiguous cases only):
  7. Goal-proximity + motion signature → GK vs ref
  8. Fallback: Gemma set-of-mark prompt
```

### Why this is better than pure Gemma

- **No per-frame VLM.** Gemma is a one-shot labeler, not a hot-path dependency.
- **Explainable.** Every role assignment comes with a reason (color outlier + near goal + lateral motion = GK).
- **Multiple signals vote.** No single point of failure.
- **Cheaper on Colab.** Goal detection is OpenCV-fast; the only model calls per frame are YOLO (already running) and optionally Gemma once per clip.
- **Degrades gracefully without Gemma.** If Gemma is unavailable, the pipeline still gives correct team labels and identifies GK/ref slots — just without the human-readable color name.

---

## Partial-palette problem — Veo's half-pitch focus

**The failure mode:** Veo's auto-camera often locks on one half during extended possession sequences. Gemma sampling 5 frames uniformly across the clip may never see the far-side goalkeeper. Palette returns `team_b_gk: null`. Without this, every far-side GK detection later in the clip gets classified against incomplete anchors and mislabeled as an outfield player.

**Two-layer fix.**

### Layer 1 — bias palette sampling toward wide shots

Don't sample frames uniformly in time. Pick frames where both halves are structurally likely to be in view:

- **Kickoffs** (first frame of each half) — Veo always pulls wide at kickoff, both GKs standing at their goals.
- **Max-detection heuristic** — rank frames by `len(det.xyxy)` and pick the top-K with a minimum time gap. Wide shots contain 15+ players; zoomed half-pitch shots rarely exceed 8–10. This correlates strongly with "both halves visible" without needing an explicit shot-classifier.
- Require ≥30 s gap between picked frames to avoid clustering on one zoomed sequence.

### Layer 2 — online palette expansion

Treat the palette as evolving state, not frozen JSON. The 4-class YOLO model *can* detect `goalkeeper` — use that as a trigger:

```python
# palette starts as Gemma's output; nulls are allowed
# palette_anchors = {"team_a_gk": [L,a,b] or None, "team_b_gk": [L,a,b] or None, ...}

for each detection:
    crop_lab = mean_lab(frame, bbox)
    nearest, dist = nearest_anchor(crop_lab, palette_anchors)

    # YOLO says GK, but color matches nothing known → surprise
    if cls == GOALKEEPER_CLASS and dist > UNKNOWN_GK_THRESHOLD:
        gk_surprise_buffer[tid].append(crop_lab)
        if len(gk_surprise_buffer[tid]) >= 10:  # ~0.3s sustained surprise
            new_anchor = np.mean(gk_surprise_buffer[tid], axis=0)
            if palette_anchors["team_b_gk"] is None:
                palette_anchors["team_b_gk"] = new_anchor
            elif palette_anchors["team_a_gk"] is None:
                palette_anchors["team_a_gk"] = new_anchor
            reclassify_track_history(tid, palette_anchors)
```

Three layers of defense this produces:

1. Gemma saw both GKs → palette complete from frame 0.
2. Gemma missed one → YOLO fires `goalkeeper` later with an unknown color → palette self-heals, prior frames of that track get retroactively corrected.
3. Neither ever sees the far GK → that player isn't in the clip, so no error to make.

This is a key design property: **the palette is best-effort-then-self-healing, not a hard contract.** It degrades gracefully instead of poisoning the whole clip.

---

## Decision criteria

Go/no-go on each phase:

- **Phase 1 ships** if Gemma palette accuracy on a 50-frame hand-label set ≥ 90%. Otherwise iterate the prompt before investing further.
- **Phase 2 ships** if Phase 0 shows Falcon at ≥ 2 fps on Colab Pro A100 OR YOLO-World hits equivalent accuracy at ≥ 10 fps on T4.
- **Proposal shelved** if Phase 1 alone gets team accuracy above 95% — Falcon's role correction becomes marginal and not worth the GPU.

---

## Related

- [[Vision System (Veo Pipeline)]] — current pipeline this extends
- [[Reference Repos]] — where to file Falcon Perception / YOLO-World notes if we vendor anything
- [tiiuae/falcon-perception on GitHub](https://github.com/tiiuae/falcon-perception) — source repo
