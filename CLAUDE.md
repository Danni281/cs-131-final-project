# CLAUDE.md

Standalone development log and working context for this repo. Written so a
fresh session on ANY machine (including the Windows GPU box with no prior
chat history) can pick up exactly where things stand. Update at the end of
every session.

## Project one-liner

Real-time perspective correction for selfie video. MediaPipe Face Mesh gives
478 (x, y, z) landmarks per frame; we treat z as a depth signal, build a
dense per-pixel depth map, and apply a true pinhole perspective re-projection
to make a close-range selfie look like it was shot from farther away with a
longer lens. Then we add temporal smoothing so it does not flicker on video.
Course: Stanford CS 131, Spring 2026. Solo project (Daoyuan Chi, Danni281).

## Current status, one paragraph

Phases 0 through 8 plus 8.5 are DONE and committed. The milestone PDF is
written and was submitted (report/milestone.pdf, 3 pages). The ML extension
(Depth Anything V2 as a learned depth source) is now RUN and committed: it
was executed on the Windows + RTX 5070 Ti laptop on 2026-05-31. Results live
in results/cmdp_eval_ml.{png,json} and results/ml_compare_*.png. What remains
is the demo slides (due 6/2) and the 4-page CVPR final report (due 6/6); both
can now cite the ML numbers in "Key results to cite" below.

## What to do next (the actual TODO)

1. ~~On the Windows 5070 Ti box: run the ML extension end to end.~~ DONE
   2026-05-31. The runbook had several never-run bugs (cu126 torch lacks
   sm_120 for this Blackwell laptop GPU; the CMDP curl/redirect dance was
   wrong; experiment A pointed at a gitignored capture; cmdp_eval.py used
   `sys` without importing it) -- all now fixed in
   `report/RUN_ON_WINDOWS_GPU.md` and `src/cmdp_eval.py`. Committed outputs:
   results/cmdp_eval_ml.{png,json}, results/ml_compare_cmdp_subject0.png,
   results/ml_compare_K36K-060208_2.png.
2. **Back on any box**: pull, integrate the ML figures into the report.
3. **Write the 4-page CVPR report** (not started). Section skeleton + the
   citation list are below.
4. **Demo slides** (not started), 4-minute talk.

## Course deadlines

- Milestone PDF (2 pages): Fri 5/22  -> DONE, submitted
- Demo slides: Tue 6/2  -> not started
- Demo day: 6/1 (virtual) or 6/3 (in-person)
- Final report (4 pages, CVPR template): Sat 6/6  -> not started

## Environment

### Mac (this repo's primary dev box)
- Python 3.12 venv at `.venv/`. Activate: `source .venv/bin/activate`.
- No torch. The live pipeline + all non-ML eval run here.

### Windows + RTX 5070 Ti (ML only)
- See `report/RUN_ON_WINDOWS_GPU.md` for the full setup.
- Python 3.12 venv, `pip install -r requirements.txt`, then torch **cu128**
  wheels (NOT cu126) + transformers + accelerate + pillow.
- The 5070 Ti here is a **Laptop GPU**, Blackwell, compute capability sm_120.
  cu126 wheels have no sm_120 kernels: torch.cuda.is_available() returns True
  but the first real GPU op raises "no kernel image is available". cu128 (and
  cu130) wheels include sm_120. Driver 592.02 / CUDA 13.1.
- Cloned to `C:\Users\danie\dev\cs-131-final-project` (deliberately OUTSIDE
  OneDrive so OneDrive does not sync/corrupt `.git`). Installed Python 3.12
  via `winget install Python.Python.3.12`.
- `git clone https://github.com/Danni281/cs-131-final-project.git` to start;
  this machine has Claude Code, the Windows box may not, so this file is the
  handoff.

### Why Python 3.12 and pinned deps
Tried 3.14 first. The only mediapipe wheel for 3.14 is 0.10.35, a stripped
"tasks-only" build with no `mp.solutions.face_mesh`. Downgraded to 3.12 and
pinned `mediapipe==0.10.21` (last release with the legacy solutions API) plus
`opencv-python` compatible with `numpy<2` (mediapipe 0.10.21 needs old numpy).
See `requirements.txt`.

## File map

```
src/main.py          live webcam app: capture, face mesh, metrics HUD,
                     side-by-side correction view. Keys: q quit, s save,
                     m toggle metrics, c toggle correction.
src/metrics.py       4 landmark ratios + auto_strength() + PORTRAIT_REFERENCE
src/warp.py          all warp modes. dense_perspective_correct is the default
                     and the project's contribution. Also build_depth_map,
                     make_alpha_mask, blend, and the sparse-mode ablations.
src/smoothing.py     EMA / Kalman / OneEuro smoothers, shared update(pts, t)
src/process_clip.py  offline: record a clip, run pipeline frame-by-frame
src/jitter_compare.py  Phase 6 plot from *_landmarks.npy logs
src/eval.py          Phase 8 end-to-end eval (latency/FPS/ratio/jitter)
src/cmdp_eval.py     Phase 8.5 CMDP cross-subject eval; --ml-depth adds the
                     Depth Anything V2 series
src/ml_depth.py      MLDepthEstimator: Depth Anything V2 via HF transformers
src/ml_compare.py    MediaPipe-depth vs ML-depth comparison figures
captures/            saved live snapshots (gitignored except .gitkeep)
clips/               recorded raw mp4s (gitignored)
results/             plots + per-run CSV/npy (gitignored EXCEPT the headline
                     PNGs: phase6_jitter.png, cmdp_eval.png, eval_default.png)
data/cmdp/           CMDP dataset. Annotations committed; images gitignored,
                     re-download per RUN_ON_WINDOWS_GPU.md
report/milestone.html / .pdf   the submitted milestone
report/RUN_ON_WINDOWS_GPU.md   the GPU runbook
```

## Phase status

| # | Phase | State | Notes |
|--:|-------|-------|-------|
| 0 | Webcam + FPS counter | done | folded into the main.py entry point |
| 1 | Face Mesh overlay + save key | done | `src/main.py`, captures in `captures/` |
| 2 | Landmark-ratio metrics + CSV | done | `src/metrics.py`. refine_landmarks=True (iris). CSV per run in `metrics_logs/`. `m` toggles HUD. |
| 3 | Single-frame correction (milestone) | done | `src/warp.py`. DEFAULT `dense` (`dense_perspective_correct`): MediaPipe (x,y,z) -> dense depth via barycentric over Delaunay -> per-pixel pinhole reprojection `scale = alpha(t+1)/(t+alpha)`, `t = z(u,v)/d_old`. Smooth depth = smooth warp = no sparse artifacts. Controlled by `--alpha` (1=off, 2=double distance, large=orthographic). Sparse modes (nose, perspective, uniform) kept for ablation. |
| 4 | Boundary alpha blending | done | `warp.make_alpha_mask` fills FACE_OVAL, erodes by feather/2, Gaussian blurs. `warp.blend` does alpha*corrected+(1-alpha)*raw. Default feather=30. |
| 5 | Per-frame baseline video | done | `src/process_clip.py record` then `run --smooth none` |
| 6 | Temporal smoothing | done | `src/smoothing.py`: EMA, vectorized per-landmark Kalman, 1-Euro (Casiez 2012). 10s/300-frame clip jitter: none 4.07 -> ema 3.28 (-19%), kalman 3.69 (-9%), oneeuro 3.40 (-16%). Plot `results/phase6_jitter.png`. |
| 7 | Real-time optimization | done | `--depth-downsample` (default 2), `--process-downsample` (default 1). 720p warp: ds1/ps1 42.7ms, ds2/ps1 38.4ms, ds2/ps2 24.1ms (~41 FPS warp-only). Live ~30 FPS default. |
| 8 | Evaluation script | done | `src/eval.py`: 5 variants on a clip; per-stage latency, FPS, pre/post ratio, jitter; JSON + `results/eval_default.png`. |
| 8.5 | CMDP cross-subject eval | done | `src/cmdp_eval.py`. 51 subjects x 7 distances. raw nose_w/face_w 0.279 (16ft) -> 0.301 (2ft). Correction closes 20% of raw->GT gap at 2ft, 39% at 3ft, 93% at 6ft, overshoots at 8-12ft. Plot `results/cmdp_eval.png`. |
| 9 (ML) | Depth Anything V2 depth source | done | Run 2026-05-31 on the 5070 Ti laptop (cu128 torch, ~6.6 min, 347/357 CMDP images). ML depth closes more of the raw->GT nose-ratio gap at close range (2ft 43% vs MP 38%, 3ft 59% vs 32%, 4ft 78% vs 33%); both overshoot past 6ft. `src/ml_depth.py`, `src/ml_compare.py`, `cmdp_eval.py --ml-depth`. `results/cmdp_eval_ml.*`. |

## The ML pivot (important context)

Original plan was to integrate Zhao 2019 "Learning Perspective Undistortion of
Portraits" as an ML comparison. **Zhao 2019 has no usable code** — the
official repo README still says "Code: coming soon" six years later, and the
community mirror (bearjoy730) is just the paper webpage with zero
implementation. DisCO 2024 has code but is a 4-dependency 3D-GAN stack
(EG3D + PTI + STIT + 3d-photo-inpainting) at ~30 s/image, too heavy.

**Pivot: Depth Anything V2** (Yang et al., NeurIPS 2024; HF
`depth-anything/Depth-Anything-V2-Small-hf`). Rather than run a competing
corrector, we swap our depth SOURCE: replace MediaPipe's sparse, jittery z
(interpolated over the mesh) with a learned dense depth field, and feed it to
the same `dense_perspective_correct` via the new `depth_override` parameter.
The report framing: "our pipeline's correction quality is bounded by depth
quality; here is MediaPipe z vs a foundation depth model on the same CMDP
benchmark." This is a cleaner, stronger story than running someone's
competitor, and it reuses 100% of our warp code.

## How the depth_override path works (so a fresh session understands warp.py)

`warp.correct(..., mode="dense", depth_override=HxW_float_array)` skips the
barycentric MediaPipe-z build entirely and uses the provided depth field. The
field is re-centered to face-plane = median internally. `MLDepthEstimator`
in `ml_depth.py` produces it: HF depth pipeline -> negate (DA-V2 returns
inverse depth, larger=closer; we flip to match MediaPipe where smaller=closer)
-> standardize and rescale to ~0.04*width std so the existing `--alpha` knob
stays in the same range. depth_source is recorded in the returned info dict
as "external" vs "mediapipe".

## Key results to cite in the report

- Phase 6 jitter: none 4.07 px -> EMA 3.28 (-19%), Kalman 3.69 (-9%),
  1-Euro 3.40 (-16%) mean frame-to-frame landmark L2. (`results/phase6_jitter.png`)
- Phase 7 latency: 720p dense warp 42.7 ms -> 24.1 ms with ds2/ps2,
  ~41 FPS warp-only, ~30 FPS end to end live.
- Phase 8.5 CMDP: raw nose_w/face_w 0.279 (16ft GT) to 0.301 (2ft);
  correction closes 20% (2ft) / 39% (3ft) / 93% (6ft) of the raw->GT gap;
  overshoots at 8-12ft which motivates --auto-strength. (`results/cmdp_eval.png`)
- ML (done 2026-05-31): MediaPipe-depth vs ML-depth (Depth Anything V2
  Small) gap-closed across 51 CMDP subjects, alpha=2.0, 347/357 images.
  GT (16ft mean nose_w/face_w) = 0.277. % of raw->GT gap closed,
  MediaPipe vs ML: 2ft 38% vs 43%, 3ft 32% vs 59%, 4ft 33% vs 78%,
  6ft 66% vs 151%, 8ft 93% vs 331%. Reading: ML depth corrects harder, so it
  wins at the close range where selfies live but overshoots past ~6ft -- the
  same fixed-alpha overshoot --auto-strength is meant to damp. ML inference
  ~10ms/frame on the 5070 Ti. (`results/cmdp_eval_ml.png`, `cmdp_eval_ml.json`,
  `results/ml_compare_cmdp_subject0.png` qualitative grid.)

## Method lineage (the methodology story for the report)

Phase 3 went through four formulations; keep all as `--mode` for ablation:
1. `uniform`: shrink central landmarks to face center by constant. Puffs cheeks.
2. `perspective` (depth-weighted radial): over-displaces brow/forehead
   (landmark 10 sits far forward in z), distorts head outline.
3. `nose`: displace only ~20 NOSE_REGION landmarks, pin the rest. Clean but
   limited. (auto_strength wires Phase-2 metrics into this mode's strength.)
4. `dense` (DEFAULT): per-pixel reprojection over a dense depth field. No
   sparse-landmark artifacts. This is the contribution. Sparse 2D landmark
   warps cannot reproduce a true 3D perspective transform; Fried 2016 needed
   a 3DMM to avoid exactly this, we use MediaPipe depth instead.

## Run commands

Mac live preview:
```bash
source .venv/bin/activate
python src/main.py --correct-on-start --alpha 2.0          # dense, default
python src/main.py --correct-on-start --smooth oneeuro     # + smoothing
```
Windows ML (after RUN_ON_WINDOWS_GPU.md setup), use backslashes:
```powershell
python src\ml_compare.py cmdp-grid --subject 0
python src\cmdp_eval.py --alpha 2.0 --ml-depth --out-prefix cmdp_eval_ml
```

## Report citation list (trim target ~13, 4 buckets)

Full annotated source notes were in a Downloads markdown artifact (Mac only);
the anchors to definitely cite:
- Portrait correction: Fried 2016, Shih 2019, Zhao 2019 (cite as "code never
  released"), DisCO 2024, Burgos-Artizzu 2014 (CMDP dataset + "ratios encode
  distance").
- Landmark detection: Kartynnik 2019 (MediaPipe Face Mesh).
- Warping: Goshtasby 1986 (piecewise affine), Bookstein 1989 (TPS).
- Temporal stability: Casiez 2012 (1-Euro), Bonneel 2015 (blind temporal
  consistency).
- Depth: Yang 2024 (Depth Anything V2) for the ML extension.

## Repo

- GitHub: https://github.com/Danni281/cs-131-final-project (private)
- Working tree clean and pushed through commit "ML extension. Depth Anything
  V2 as an alternative depth source".
- Commit/push only when work is in a good state; this file is the source of
  truth for cross-machine handoff.
