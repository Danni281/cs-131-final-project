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
| 2 | Landmark-ratio metrics + CSV | done | `src/metrics.py`. refine_landmarks=True (iris). CSV per run in `metrics_logs/`. Press `m` to toggle HUD. |
| 3 | Single-frame correction (milestone) | done | `src/warp.py`. **NEW DEFAULT: `dense`** (`dense_perspective_correct`) — treats MediaPipe (x,y,z) as a sparse depth signal, builds a dense per-pixel depth map by barycentric interpolation across the Delaunay mesh, then applies true pinhole perspective re-projection per pixel: `scale = α(t+1)/(t+α)` with `t = z(u,v)/d_old`. Smooth depth = smooth warp = no sparse-landmark artifacts. ~44ms on 720p. Controlled by `--alpha` (1=off, 2=double virtual camera distance, ∞=orthographic). Sparse-landmark modes (nose, perspective, uniform) kept for ablation. `--auto-strength` still works for the nose/perspective sparse modes. |
| 4 | Boundary alpha blending | done | `warp.make_alpha_mask` fills FACE_OVAL polygon, erodes by `--feather/2`, Gaussian-blurs (sigma=feather/2). `warp.blend` does `alpha*corrected + (1-alpha)*raw`. Default feather=30. Adds ~12ms at default; feather=80 adds ~90ms (kernel grows). |
| 5 | Per-frame baseline video | done | `src/process_clip.py record` + `run --smooth none` |
| 6 | Temporal smoothing | done | `src/smoothing.py` with EMA, vectorized per-landmark Kalman, **1-Euro Filter (Casiez 2012)** as the third method. CLI: `--smooth {ema,kalman,oneeuro}`. **Ablation on a 10s clip** (300 frames @30 FPS): mean frame-to-frame landmark L2 jitter `none 4.07 -> ema 3.28 (-19%), kalman 3.69 (-9%), oneeuro 3.40 (-16%)`. EMA strongest at rest but laggiest; 1-Euro the best lag-jitter tradeoff (matches Casiez 2012's claim). Plot: `results/phase6_jitter.png`, JSON: `results/phase6_jitter.json`. `src/process_clip.py` runs the offline pipeline; `src/jitter_compare.py` builds the comparison plot. |
| 7 | Real-time optimization | todo | precompute triangulation, reuse map arrays, target 30 FPS live |
| 8 | Evaluation script | todo | FPS, latency, ratio deltas, frame-to-frame L2, plots, JSON |
| 8.5 | CMDP cross-subject evaluation | done | `src/cmdp_eval.py`. CMDP (Burgos-Artizzu 2014) = 51 subjects × 7 distances (2-16 ft). For each (subject, distance): run MediaPipe, compute `nose_w/face_w`, apply dense correction, re-measure. Use 16ft as per-subject GT. **Results (51 subjects, 347 images, alpha=2.0):** raw `nose_w/face_w` inflates from 0.279 (16ft) to 0.301 (2ft) — clear perspective signal. Correction closes 20% of the raw->GT gap at 2ft, 39% at 3ft, 93% at 6ft, and overshoots at 8-12ft. Alpha sweep (1.5/2.0/3.0) shows higher alpha = bigger close-range gap reduction. Plot: `results/cmdp_eval.png`. JSON: `results/cmdp_eval.json`. |

## Decisions / non-obvious choices

- **Python 3.12** required. Tried 3.14 first: the only mediapipe wheel
  available for 3.14 is 0.10.35, which is a stripped-down "tasks-only" build
  with no `mp.solutions.face_mesh` and a broken `tasks` namespace. Downgraded
  to 3.12 and pinned `mediapipe==0.10.21` (last release with the legacy
  solutions API we use). `opencv-python` pinned compatible with `numpy<2`
  because mediapipe 0.10.21 needs the older numpy.
- `refine_landmarks=False` in `make_face_mesh()` — saves cost; iris not needed
  for perspective correction.
- Frame is `cv2.flip`'d horizontally for selfie ergonomics — remember to undo
  before saving raw for ground-truth comparisons in phase 8.
- Started 2D-only. Proposal mentioned MediaPipe's z-coords as a 3D scaffold;
  revisit if uniform shrink underperforms (phase 3 stretch).
- Phase 2 landmark indices (in `src/metrics.py`):
  iris 468/473, nose tip 4, chin 152, forehead 10, alar 49/279,
  face edges 234/454, ear proxies 127/356. Pre-Phase-2 saved
  `_landmarks.npy` files have only 468 points (no iris) so
  `compute_metrics` returns None on them by design.

## Open questions / TODO before milestone

- ~~**Connect phase-2 metrics to phase-3 correction strength** (TA
  explicitly flagged "define your distortion metrics with more
  detail").~~ DONE — `--auto-strength` derives per-frame strength from
  the measured nose_w/face_w ratio vs a portrait-distance reference
  (0.22). Strength is no longer a hand-tuned constant; it's a function
  of measured distortion. Adapts as the user moves toward/away from
  the camera. Burgos-Artizzu 2014 (CMDP) supports the "landmark ratios
  encode distance" premise; cite in report.
- ~~Decide on a 3D scaffold or commit to 2D radial shrink with
  justification.~~ DONE — tried three approaches in sequence:
  (1) uniform shrink: puffy cheeks (proposal-style "start simple",
  doesn't look good).
  (2) full-face depth re-projection (symmetric and asymmetric): visible
  peripheral artifacts because radial scaling over-displaces brow/
  forehead landmarks that are far from face center AND have very
  negative z. Sparse 2D landmarks can't faithfully reproduce a 3D
  perspective transform; Fried 2016 needed a 3DMM for this reason.
  (3) **nose-only restriction**: displace only NOSE_REGION (~20
  landmarks); everything else pinned. Subtle, clean, defensible. This
  is the default and what the milestone visuals use. Discuss the
  attempted broader approaches as a methodology story in the report.
- Phase 4 blending intentionally keeps the hard-boundary warp under the
  hood — the alpha mask only hides the discontinuity at the head outline.
  A future refactor could distribute the displacement across the boundary
  band instead, but the proposal explicitly describes the alpha-mask
  approach, so we shipped that.
- Warp at 41ms/frame is ~24 FPS warp-only; combined pipeline will land below
  the 25 FPS target. Phase 7 (precompute triangulation, cache map arrays
  across frames) is what fixes this.

## Repo

- GitHub: https://github.com/Danni281/cs-131-final-project (private)
- Local: `~/Desktop/cs-131-final-project`
- Venv: `.venv/` (Python 3.12 — see decisions above)

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
