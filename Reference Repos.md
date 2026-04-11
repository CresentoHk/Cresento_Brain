---
title: Reference Repos
type: reference
tags:
  - reference
  - ml
  - cv
created: 2026-04-11
location: "Random Repos/, unravelsports-main/"
---

# 📚 Reference Repos

Third-party / open-source soccer-analytics projects vendored into the workspace as reference. Most are inputs to or inspirations for the [[Vision System (Veo Pipeline)|Veo vision pipeline]].

> [!info] Read-only
> These are NOT actively developed in this workspace. Don't edit them — pull updates from upstream if needed.

---

## In `Random Repos/`

| Repo                            | What it is                                                          |
| ------------------------------- | ------------------------------------------------------------------- |
| `PassNet-master/`               | Pass detection neural net for soccer                                |
| `Tactic_Zone-master/`           | Tactical pattern extraction from tracking data                      |
| `ball-action-spotting-master/`  | Ball-action spotting (event detection in match video)               |
| `sn-spotting-main/`             | SoccerNet spotting models                                           |
| `ball_action_results (1)/`      | Pretrained results / outputs                                        |
| `model-006-0.864002.pth`        | Trained checkpoint (filename suggests 0.864 metric)                 |
| `ball_finetune_long_004-...zip` | Fine-tuning artifacts                                               |
| `config.json`                   | Shared config (likely for one of the repos above)                   |

Most of these have their own README inside their folder. Look there before guessing.

---

## In `unravelsports-main/`

A third-party Python sports analytics library — graph neural networks (GNN) for tracking data. Notable because the [[Vision System (Veo Pipeline)|Veo pipeline]] mirrors its column conventions in `veo_loader.py`:

```
period_id, frame_id, timestamp,
x, y, z,
player_id (alias 'id'), team_id,
vx, vy, vz, v,
ax, ay, az, a
```

Sticking to the unravelsports schema means you can plug Veo output into unravelsports models with no glue code.

---

## CSV data

Not strictly a "repo" but related: `CSV data/` contains raw player session CSV files (20 Hz IMU data) used by:
- [[Experimental Implementation]] for prototyping
- Manual analysis when debugging metric formulas

These are real player sessions — handle accordingly.

---

## Other top-level extras

| Path                            | What it is                                                          |
| ------------------------------- | ------------------------------------------------------------------- |
| `Veo Videos/`                   | The actual [[Vision System (Veo Pipeline)\|Veo pipeline]]           |
| `React app crashlytics/`        | Android crash logs from the [[Cresento React Native App\|RN app]]   |
| `Nordic/`                       | Nordic SDK toolchain & S112 SoftDevice headers                      |
| `Nordic_Test_Project/`          | Minimal Zephyr RTOS BLE test (~99 lines), pre-Arduino spike         |

---

## Related

- [[Vision System (Veo Pipeline)]]
- [[AI Coach (text_ai_coach)]]
- [[Experimental Implementation]]
