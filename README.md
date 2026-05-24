# cs-131-final-project

**Real-Time Perspective Correction for Selfie Video** — CS 131 Spring 2026 final project (Daoyuan Chi).

Selfies and webcam video are shot at short camera-to-face distances, which makes
noses and foreheads look enlarged. This project builds a real-time pipeline that
reduces that perspective distortion and stays temporally stable across frames.

## Status

| Phase | Description | State |
|------:|-------------|-------|
| 0 | Webcam capture loop + FPS counter | done |
| 1 | MediaPipe Face Mesh overlay + save key | done |
| 2 | Per-frame landmark-ratio metrics + CSV log | todo |
| 3 | Single-frame correction (Delaunay warp) — **milestone** | todo |
| 4 | Boundary blending mask | todo |
| 5 | Per-frame baseline video | todo |
| 6 | Temporal smoothing (EMA + Kalman) | todo |
| 7 | Real-time optimization (30 FPS live) | todo |
| 8 | Evaluation script | todo |

See [CLAUDE.md](CLAUDE.md) for the running development log.

## Setup

Requires **Python 3.12** (mediapipe 0.10.21 has no wheels for 3.13/3.14,
and newer mediapipe drops the Face Mesh `solutions` API).

```bash
brew install python@3.12         # if you don't have it
python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Run

```bash
python src/main.py              # default webcam, 1280x720
python src/main.py --camera 1   # second camera
```

Keys inside the window:
- `q` — quit
- `s` — save current frame + landmarks to `captures/`

Each `s` writes four files under `captures/`:
`<timestamp>_raw.png`, `<timestamp>_overlay.png`, `<timestamp>_landmarks.npy`,
`<timestamp>_meta.json`.

## Layout

```
src/main.py        # phase-1 capture + face mesh entry point
captures/          # saved frames + landmarks (gitignored)
requirements.txt
CLAUDE.md          # dev log for future sessions
```
