# Gazely

Gazely is a user experience evaluation tool that combines gaze estimation, mouse activity tracking, and screen recording. Its web interface supports calibration, session recording, playback, screenshot heatmaps, and review of potential friction events.

The frontend runs in a browser, while a Python backend captures the camera, screen, and mouse on the same computer. The current implementation is designed for local testing and stores session data as files.

## Features

- Grid-based gaze calibration with validation on separate targets.
- Webcam-based gaze estimation using Ridge Regression, Kalman filtering, and EMA smoothing.
- Primary-screen recording alongside gaze, mouse movement, clicks, and scrolling.
- Video playback with gaze and mouse markers synchronized to session timestamps.
- Screenshot heatmaps grouped by page appearance, with fixed smoothing and PNG export.
- Detection of potential rage clicks and gaze–mouse mismatches.
- Calibration accuracy reports and gaze frame quality information.

## Technology

| Layer | Components |
| --- | --- |
| Frontend | React, React Router, Tailwind CSS |
| Build and assets | Vite, TypeScript, Lucide React, Onest |
| API | FastAPI, Uvicorn |
| Gaze estimation | Local EyeTrax package, MediaPipe, OpenCV, scikit-learn |
| Numerical computing | NumPy, SciPy |
| Screen and mouse capture | MSS, pynput, screeninfo |
| Storage | CSV, JSON, PKL, MP4, JPG; PNG heatmap exports |

The application interface is written in JavaScript/JSX. TypeScript is also part of the frontend build process. Dependencies are listed in [requirements.txt](requirements.txt), [api/requirements.txt](api/requirements.txt), [eyetrax/pyproject.toml](eyetrax/pyproject.toml), and [frontend/package.json](frontend/package.json).

## Requirements

- A desktop computer with a webcam, primary display, and mouse or touchpad. The repository's web workflow was developed on Windows.
- A Python version compatible with MediaPipe, OpenCV, and scikit-learn. EyeTrax package metadata specifies Python ≥3.9; compatibility with a particular Python version still depends on dependency availability.
- Node.js ≥22.12 for the Vite version used in this repository.
- Internet access to install dependencies and download the face landmark model on first use.
- H.264 encoding support in OpenCV to save screen recordings in web mode.

Use one primary display. Keep the resolution, display scaling, camera position, and participant position consistent between calibration and recording.

## Installation

Run the following commands from the repository root. These examples use Windows PowerShell.

### Backend

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r requirements.txt -r api/requirements.txt mss
```

`requirements.txt` installs EyeTrax from the local `eyetrax/` directory. MSS is required for screen recording and is installed explicitly above because it is not yet listed in the requirements files.

On first use, the application downloads `face_landmarker.task` to the user cache. To use an existing local model, set the `EYETRAX_FACE_LANDMARKER_MODEL` environment variable to its file path before starting the backend.

### Frontend

```powershell
cd frontend
npm.cmd ci
```

Using `npm.cmd` in PowerShell avoids invoking the `npm.ps1` shim.

## Running the web app

Start the backend and frontend in separate terminals.

**Terminal 1 — from the repository root:**

```powershell
.\.venv\Scripts\python.exe -m uvicorn api.server:app --host 127.0.0.1 --port 8888 --reload
```

**Terminal 2 — from the repository root:**

```powershell
cd frontend
npm.cmd run dev
```

Open the address printed by Vite, usually `http://localhost:5173`. The frontend proxies `/api` and `/data` requests to the backend on port `8888`. API documentation is available at `http://127.0.0.1:8888/docs`.

### Usage

1. **Calibrate:** start calibration, follow the targets in the fullscreen window, and review the preview. Press **Enter** to save calibration, **R** to retry, or **Esc** to cancel.
2. **Record:** open the website to evaluate and start recording. The latest saved calibration is reused; if none is available, calibration preparation runs first.
3. Wait for the **valid gaze** count to increase before starting the task. Keep your face visible to the camera.
4. Use the website on the primary display. Switching tabs or applications can also appear in the desktop recording.
5. Return to **Record**, select **Stop & save session**, and wait for saving to finish.
6. Open **Sessions** to review playback, screenshot heatmaps, friction events, and calibration reports. Select **Download PNG** to export a screenshot with its heatmap.

### Serving the frontend build

To serve the frontend from the local backend:

```powershell
cd frontend
npm.cmd run build
cd ..
.\.venv\Scripts\python.exe -m uvicorn api.server:app --host 127.0.0.1 --port 8888
```

Open `http://127.0.0.1:8888`. The backend serves `frontend/dist/` if that directory exists when the backend loads. Use in-app navigation; the backend does not currently provide an SPA fallback for direct access to every frontend route.

## Command-line mode

The CLI records gaze and mouse activity without the dashboard:

```powershell
.\.venv\Scripts\python.exe main.py
```

Complete calibration and review the preview. In CLI mode, pressing **Enter** in the preview starts recording, while **Ctrl+C** in the terminal stops and saves the session. Unlike web mode, this CLI workflow does not record screen video.

Example configuration:

```powershell
.\.venv\Scripts\python.exe main.py --camera 0 --grid-rows 5 --grid-cols 5 --settle 1.2 --capture 2 --ema-alpha 0.25
```

| Argument | Default | Purpose |
| --- | --- | --- |
| `--camera` | `0` | Camera index |
| `--grid-rows`, `--grid-cols` | `4`, `4` | Number of calibration target rows and columns |
| `--settle` | `1.0` | Settling time per target, in seconds |
| `--capture` | `1.5` | Sample capture time per target, in seconds |
| `--ema-alpha` | `0.25` | Weight assigned to the previous EMA output |
| `--model` | `ridge` | Choose `ridge`, `svr`, `elastic_net`, or `tiny_mlp` |
| `--model-file` | Unset | Load a local model; validation and preview still run |
| `--output` | `data/sessions` | CLI output directory |

To record gaze only:

```powershell
.\.venv\Scripts\python.exe eyetrax/gaze_logger.py
```

Only load pickle model files from trusted sources.

## Session data

Each session is stored in `data/sessions/<session_id>/`.

| File | Contents |
| --- | --- |
| `calibration_report.json` | Configuration, validation errors, session timestamps, and valid frame counts |
| `gaze_model.pkl` | Calibrated gaze model |
| `gaze_log.csv` | Timestamps and valid gaze coordinates after filtering |
| `gaze_quality.csv` | Raw gaze, filtered gaze, validity, and frame rejection reasons |
| `mouse_session.json` | Timestamped mouse movement, click, and scroll events; available in gaze + mouse mode |
| `screen.mp4` | H.264 primary-screen recording configured at 10 FPS; available in web mode |
| `friction_events.json` | Potential rage clicks and gaze–mouse mismatches |
| `page_snapshots/` | JPG screenshots and a display-interval manifest, generated from video when screenshot heatmaps are requested |

Web-mode calibration is stored in `data/latest_calibration/`. Webcam video is not saved as a primary session output. The session data directory is ignored by Git.

Screenshots are grouped by visual screen similarity, rather than website URL or DOM. Gaze from matching display intervals is combined on the corresponding screenshot. Different scroll positions may produce separate screenshots; very brief appearances may be ignored as transitions.

## Methods and interpreting results

- **Calibration:** a default 4 × 4 grid followed by validation on five different targets. Validation samples are not used to retrain the regressor.
- **Gaze estimation:** facial landmark features are mapped to screen coordinates using Ridge Regression by default.
- **Filtering:** Kalman filtering and EMA reduce jitter. Smoothing does not remove systematic bias; a larger EMA weight can also increase lag.
- **Rage clicks:** candidates are detected from at least three clicks within a one-second window.
- **Gaze–mouse mismatch:** each click is paired with the nearest gaze sample within ±0.5 seconds; a candidate is flagged when the distance exceeds 150 pixels.
- **Screenshot heatmaps:** gaze density is rendered using a Gaussian kernel with fixed smoothing.

`mean_error_px`, `median_error_px`, and `p90_error_px` measure raw gaze error relative to validation targets. Lower values indicate predictions closer to the targets during that stage, rather than guaranteed accuracy throughout the session. Filter noise is estimated from the same validation pass, so the report is not an independent accuracy test of the filtered output.

Review potential friction events and interface recommendations alongside the video, tracking quality, and task context. The absence of mismatch events in a session without valid gaze does not establish that the interface has no usability issues.

## Repository structure

```text
api/
  server.py                API endpoints and session process management
  page_snapshots.py        Screen screenshot grouping
  _scripts/                Web-mode calibration and recording
eyetrax/
  src/eyetrax/             Gaze estimator, models, filters, and workflows
frontend/
  src/pages/               Home, Calibrate, Record, Sessions, session details
  src/components/          Playback, heatmaps, and interface components
  src/assets/              Gazely logos
fusion/
  live_fusion.py           CLI entry point and friction heuristics
mouse-tracker/             Mouse tracking components
tests/                     Python tests
main.py                    Gaze + mouse CLI entry point
start.py                   Windows local launcher
```

`start.py` installs API dependencies and starts the backend. It also builds the frontend if `frontend/dist/` does not exist. EyeTrax, MSS, and frontend dependencies must already be installed. Rebuild after frontend changes because the launcher does not rebuild an existing `dist` directory.

## Testing

Run Python tests from the repository root:

```powershell
.\.venv\Scripts\python.exe -m pip install pytest
.\.venv\Scripts\python.exe -m pytest -q
```

Screenshot and recording tests can also run without a camera using the standard library:

```powershell
.\.venv\Scripts\python.exe -m unittest discover -s tests -p test_session_playback.py -v
.\.venv\Scripts\python.exe -m unittest discover -s tests -p test_page_snapshots.py -v
```

Frontend timeline and heatmap tests:

```powershell
cd frontend
node --test src/components/playbackTimeline.test.js src/components/heatmapDensity.test.js
npm.cmd run build
```

Automated tests use simulated inputs for device-dependent behavior. Webcam access, fullscreen behavior, real-world synchronization, and accuracy on the intended participants and devices still require manual testing.

## Troubleshooting

| Problem | What to check |
| --- | --- |
| Frontend cannot connect to the API | Confirm that the backend is running on port `8888` and the proxy configuration matches |
| Camera unavailable | Close other applications using the camera, check camera permissions, or select another camera index in CLI mode |
| Valid gaze count stays at zero | Check tracking quality messages, lighting, face position, and calibration; gaze data that was never recorded cannot be recovered |
| Video unavailable | Check `video_error` in the session report and OpenCV H.264 encoder support |
| Page screenshots unavailable | Confirm that the session has finished and contains readable video |
| Built interface has not updated | Run `npm.cmd run build` again; restart the backend if the `dist/` directory was newly created |

Storage and the dashboard can move to a server by adding session uploads and access management while retaining local recording. Moving recording into the browser also requires adapting camera, screen, and mouse acquisition, calibration, and synchronization.

## Attribution

Gaze estimation uses [EyeTrax](https://github.com/ck-zhang/eyetrax) code included in the `eyetrax/` directory. Its license is available in [eyetrax/LICENSE](eyetrax/LICENSE). The Onest font license is included in [frontend/public/Onest-OFL.txt](frontend/public/Onest-OFL.txt).
