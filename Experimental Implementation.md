---
title: Experimental Implementation
type: project
tags:
  - project/experimental
  - python
  - analysis
created: 2026-04-11
status: reference
location: "experimental_implementation/"
---

# 🧪 Experimental Implementation

Python scratchpad for IMU data analysis and rest detection. Used to prototype algorithms before they get hardened into [[StatsEngine Cross-Platform|StatsEngine]] on the apps.

> [!info] Lab notebook, not a product
> This is exploratory code. The outputs (PNG profile plots) are decision artifacts — they answer questions like "what does player X's intensity profile look like across a session?".

---

## Layout

```
experimental_implementation/
├── plot_session.py           (per-session intensity plot)
├── rest_detector.py          (detect rest periods within a session)
├── data/                     (input session CSVs)
├── requirements.txt
├── diagnostic_intensity_profiles.png
├── explore_profiles.png
├── final_alexNL.png
├── final_anson.png
├── final_Keenan.png
├── final_maxE.png
├── final_nathanC.png
└── final_Toi_Wing.png
```

The `final_*.png` files are per-player intensity profile renders — keep them; they're easier to compare against than re-running the script.

---

## Why it exists

Wearable IMU data is noisy. Before adding a new metric to [[StatsEngine Cross-Platform|StatsEngine]] (which runs on three platforms and ships to coaches), you want to **prototype on real data** in a fast loop. This is that fast loop.

Typical workflow:
1. Pull a session CSV out of `CSV data/` (or export from Firestore)
2. Run `plot_session.py` / `rest_detector.py` on it
3. Tune the algorithm
4. Once happy, port to the three [[StatsEngine Cross-Platform|StatsEngine]] implementations

---

## Related

- [[StatsEngine Cross-Platform]] — where prototypes graduate to
- [[Data Pipeline]] — how the input CSVs get produced
- [[Vision System (Veo Pipeline)]] — sibling Python project
