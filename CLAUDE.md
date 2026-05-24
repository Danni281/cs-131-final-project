# CLAUDE.md

Development log + working context for Claude sessions on this repo. Keep this
short, factual, and current. Update at the end of every session.

## Project one-liner

Real-time perspective correction for selfie video using MediaPipe Face Mesh
landmarks, a Delaunay piecewise-affine warp, and temporal smoothing.
Course: Stanford CS 131, Spring 2026. Solo project.

## Course deadlines

- Milestone PDF (2 pages): Fri 5/22
- Demo slides: Tue 6/2
- Demo day: 6/1 (virtual) or 6/3 (in-person)
- Final report (4 pages, CVPR template): Sat 6/6

## Plan-to-week mapping

Plan has 9 phases; proposal has 4 weeks. Phases are reordered vs the original
plan to honor TA feedback ("get the real-time pipeline stable BEFORE adding
temporal filtering"):

- Week 1 (-> 5/22 milestone): phases 0, 1, 2, 3
- Week 2 (-> 5/29): phases 4, 5, **7** (real-time first)
- Week 3 (-> 6/3 demo): phase 6 (EMA + Kalman)
- Week 4 (-> 6/6 report): phase 8 + writing

## Phase status

| # | Phase | State | Notes |
|--:|-------|-------|-------|
| 0 | Webcam + FPS counter | done | folded into phase 1 entry point |
| 1 | Face Mesh overlay + save key | done | `src/main.py`, captures land in `captures/` |
| 2 | Landmark-ratio metrics + CSV | todo | IPD/face_w, nose_w/face_w, nose-chin/face_w, ear-ear/face_h |
| 3 | Single-frame correction (milestone) | todo | Delaunay + cv2.remap, scale=0.92 to start, boundary fixed |
| 4 | Boundary alpha blending | todo | |
| 5 | Per-frame baseline video | todo | record 10s clips: still, talking, head-turn |
| 6 | Temporal smoothing | todo | EMA alpha=0.7 first, then per-landmark Kalman variant |
| 7 | Real-time optimization | todo | precompute triangulation, reuse map arrays, target 30 FPS live |
| 8 | Evaluation script | todo | FPS, latency, ratio deltas, frame-to-frame L2, plots, JSON |

## Decisions / non-obvious choices

- **Python 3.14** works for mediapipe 0.10.35; no need to downgrade.
- `refine_landmarks=False` in `make_face_mesh()` — saves cost; iris not needed
  for perspective correction.
- Frame is `cv2.flip`'d horizontally for selfie ergonomics — remember to undo
  before saving raw for ground-truth comparisons in phase 8.
- Started 2D-only. Proposal mentioned MediaPipe's z-coords as a 3D scaffold;
  revisit if uniform shrink underperforms (phase 3 stretch).

## Open questions / TODO before milestone

- Connect phase-2 metrics to phase-3 correction strength (TA explicitly flagged
  "define your distortion metrics with more detail"). The 0.92 scale should
  become a function of measured ratios, not a constant.
- Decide on a 3D scaffold or commit to 2D radial shrink with justification.

## Repo

- GitHub: https://github.com/Danni281/cs-131-final-project (private)
- Local: `~/Desktop/cs-131-final-project`
- Venv: `.venv/` (Python 3.14)

## How to run right now

```bash
cd ~/Desktop/cs-131-final-project
source .venv/bin/activate
python src/main.py
```

## Citations to keep handy (for report)

Full annotated list lives in `~/Downloads/compass_artifact_wf-242c1d3f-da6e-416a-8b5e-4b581a107666_text_markdown.md`.
Trim target: 13 refs across 4 buckets (portrait correction / landmark detection /
warping / temporal stability). Anchors: Fried 2016, Shih 2019, Alhawwary 2026,
Kartynnik 2019 (MediaPipe), Goshtasby 1986, Casiez 2012 (1 Euro), Bonneel 2015.
