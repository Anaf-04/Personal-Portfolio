# DAINVI (智能檢測系統 / Intelligent Inspection System)

DAINVI is an internal industrial visual-inspection and quality-control platform built by **Dalantia Technology (Hong Kong) Company Limited** (大藍蛇 Dalantia), a Hong Kong-based hard-tech and AI startup founded by **Dr. Chun Long Aaron LAM**. Dalantia builds AI-driven precision manufacturing and on-demand production platforms, including its flagship **FASTD (Factory-As-a-Service Technology by Dalantia)** brand. DAINVI supports that mission by giving factory floor staff a browser-based tool to monitor production cameras in real time and to build the labeled image datasets needed to train AI-based defect-detection models.

> ⚠️ This is an internal tool. The repository currently contains hardcoded credentials (Gmail OAuth tokens in `python/config_login.py` and `system/token.json`, a reCAPTCHA secret key in `python/bp_login.py`) and seeded test accounts with plaintext passwords in `database/python/create_database.py`. These must be rotated/removed before the repository is shared outside the team or made public.

## What it does

DAINVI is a locally-run Flask web application (it launches like a desktop app — it opens your default browser automatically) with four main capabilities, reflected in the sidebar of the UI:

- **Real-time detection** — streams live video from factory cameras (USB or network/RTSP/ISP sources), lets operators start/stop recordings, and take snapshots of parts on the line.
- **Data annotation** — a manual labeling workbench where users draw bounding boxes on captured images and tag them with defect types (e.g. `scratch`, `insPolishing`), building a labeled dataset for model training. Supports per-user/per-organization access control over image sets and exports labeled data in a YOLO-style train/val split.
- **Defect management** — browsing and managing the catalog of defect types and individual defect records.
- **Batch / workshop management** — organizing inspection work by production batch and workshop.
- **Virtual bot** — placeholder module for an assistant/automation feature.
- **Login & accounts** — email/password authentication with Google reCAPTCHA v2, organization-scoped users, and an (in-progress) email verification flow via the Gmail API.

The `/darknet` blueprint and the empty `darknet/` directory are the integration point for a **YOLO/Darknet** object-detection model: the snapshot → annotation pipeline exists today to produce training data, but no model weights or inference code are checked into this snapshot of the repository yet. The intent is for a trained Darknet/YOLO model to eventually perform automated defect detection on the live camera feed.

## Tech stack

### Backend
- **Python 3.12** / **Flask** — web framework and routing, organized as four Flask **blueprints**:
  | Blueprint | File | URL prefix | Responsibility |
  |---|---|---|---|
  | `app_main_bp` | `python/bp_main.py` | `/` | Serves the SPA shell (`index.html`), initializes/restores the database on startup, runs an hourly autosave backup thread |
  | `app_darknet_bp` | `python/bp_darknet.py` | `/darknet` | Camera management, MJPEG video streaming, recording, snapshotting, feeding snapshots into the annotation pipeline |
  | `app_data_annotation_bp` | `python/bp_data_annotation.py` | `/data_annotation` | Image preloading, bounding-box annotation storage, status tracking, dataset export |
  | `app_login_bp` | `python/bp_login.py` | `/login` | Authentication, reCAPTCHA verification, password reset, email verification |
- **OpenCV (`opencv-python`)** — camera capture and frame processing, wrapped in a background-thread `VideoCaptureAsync` class for non-blocking streaming.
- **simplejpeg** — fast JPEG encoding for the MJPEG live-view stream.
- **NumPy** — array/image manipulation.
- **SQLite3** (Python standard library, no ORM) — embedded database; see below.
- **Google APIs** (`google-auth`, `google-api-python-client`, `oauth2client`) — Gmail API integration for sending account-verification emails.
- **wcwidth** — used for pretty-printing SQL query results to the terminal during development.
- A single global `threading.Lock` (`python/shared_variables.py`) serializes all SQLite access across request threads, since Flask's dev server is multi-threaded but SQLite access is not.

### Frontend
- **Vanilla JavaScript (ES modules)** — no framework (no React/Vue/Angular), no bundler, no build step, no `package.json`. All logic lives in `static/assets/scripts/*.mjs` and is dynamically imported on navigation from a single SPA shell (`static/assets/pages/index.html` + `index.mjs`).
- **jQuery 3.7.1** — vendored locally for DOM/AJAX convenience.
- **Boxicons** and **Font Awesome** — icon fonts, loaded from CDN.
- **Google reCAPTCHA v2** — bot protection on login, loaded from Google's CDN.
- **Plain CSS** — hand-written stylesheets per page (`constants.css`, `index.css`, `login.css`, `real_time.css`, `data_annotation.css`, `reusable_styles.css`), no preprocessor.
- **Built-in i18n** — `language.mjs` implements a lightweight translation engine covering English, Traditional Chinese (繁體中文), and Bengali (বাংলা), the latter partially translated.

### Data / storage
- **SQLite** database file (`database/data/dainviapx`), schema defined in `database/python/create_database.py` and configured in `database/python/database_configuration.py`. Key tables:
  - `organization`, `user`, `user_additional_email`, `user_annotation_custom_defect` — identity and access model.
  - `annotation_image_set`, `annotation_image_set_access_org`, `annotation_image_set_access_user` — image-set catalog and access control.
  - `annotation_type`, `default_defect`, `record`, `annotation_status` — defect taxonomy and the actual bounding-box annotation data.
  - `recordings` — metadata for saved camera recordings.
- The database is **automatically backed up every hour** to `database/data_backup/` and restored from backup on startup if the primary file is missing.
- `database/python/config.json` stores the runtime camera registry (camera names mapped to device index or RTSP/ISP URL) — not a secrets file, but rewritten by both `python/config_real_time.py` and the `/darknet` blueprint's camera CRUD routes.

### Vision / ML (planned integration)
- **Darknet / YOLO** — the `darknet/` directory and `sys.path.append('./darknet')` in `run_browser.py` are scaffolding for a future object-detection model to run inference against the live camera feed for automated defect detection. The annotation workflow in `app_data_annotation_bp` exists specifically to produce the labeled training data (exported in YOLO label format) this model would be trained on.
- `dataset/` — intended output location for exported, labeled training data (currently empty).

## Project structure

```
DAINVI Browser- FINAL VERSION/
├── run_browser.py              # App entry point: builds Flask app, registers blueprints, launches browser
├── requirements.txt            # Python dependencies
├── python/
│   ├── bp_main.py              # Main blueprint: serves SPA shell, DB init + autosave
│   ├── bp_darknet.py           # Real-time camera streaming, recording, snapshots
│   ├── bp_data_annotation.py   # Manual annotation workflow + dataset export
│   ├── bp_login.py             # Auth, reCAPTCHA, password reset
│   ├── config_main.py          # Host/port config
│   ├── config_real_time.py     # Camera/recording paths (resets config.json on import)
│   ├── config_login.py         # Mail sender config (contains live credentials — do not commit)
│   ├── config_data_annotation.py # Annotation batch size, train/val export ratio
│   └── shared_variables.py     # Global DB access lock
├── database/
│   ├── python/
│   │   ├── create_database.py          # Schema + seed data
│   │   ├── database_configuration.py   # Table/column name constants
│   │   ├── database_funcs.py           # Query helpers
│   │   └── config.json                 # Runtime camera registry
│   ├── data/                            # Live SQLite DB file
│   └── data_backup/                     # Hourly autosave backup
├── static/
│   ├── assets/
│   │   ├── pages/index.html    # SPA shell template
│   │   ├── scripts/*.mjs       # ES module frontend (per-page logic, i18n, UI helpers)
│   │   └── stylesheets/*.css   # Per-page styles
│   ├── images/                 # UI assets, icons, captured defect photos
│   ├── snapshots/              # Camera snapshots taken during inspection
│   └── videos/                 # Saved camera recordings
├── darknet/                     # Placeholder for YOLO/Darknet model files (empty)
├── dataset/                     # Placeholder for exported labeled training data (empty)
└── system/                      # Bundled PowerShell + OAuth token cache (Windows runtime support)
```

## Running the app

```bash
pip install -r requirements.txt
python run_browser.py
```

This initializes (or restores) the SQLite database, starts the Flask dev server on `127.0.0.1:5100` (configurable in `python/config_main.py`), and automatically opens the app in your default browser. The app has also been run on an NVIDIA Jetson device (Linux/ARM) based on deployment paths referenced in `TODO.txt`, alongside its primary Windows development environment.

**Note:** This uses Flask's built-in development server (`app.run(...)`), not a production WSGI server — it's intended for local/on-site desktop-style use on the factory floor, not for public internet deployment.

## Known limitations / cleanup items

- No automated tests, linter, or CI configuration currently exist in this repository.
- Live-looking credentials are checked into `python/config_login.py` and `system/token.json`; these should be rotated and moved to environment variables or a secrets manager.
- `database/python/create_database.py` seeds test users with plaintext passwords and real-looking email addresses — fine for local development, but should not ship as-is.
- The Darknet/YOLO inference step is not yet implemented — the `darknet/` folder is empty and `bp_darknet.py` only handles camera I/O, not detection.
