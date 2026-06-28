# PDF Password Remover — Design Spec

**Date:** 2026-06-27 (updated 2026-06-28)
**Status:** Implemented

## Summary

A **personal, local web application** that removes the password from a PDF the
user already knows the password to. The user uploads a PDF, enters its password,
and downloads a decrypted copy that opens without any password.

This is a *known-password decryption* tool — **not** a password cracker. There is
no brute-force/recovery functionality.

## Goals

- Decrypt a password-protected PDF using the **known** password and return a
  PDF that opens with no password.
- Polished, modern web UI: drag-and-drop, clear states, prominent
  download button.
- Runs entirely **locally and offline**. No uploaded data ever leaves the
  user's computer; no dependency contacts the internet.
- Configurable via a YAML file.

## Non-Goals (YAGNI)

- No password cracking / brute-force / recovery of unknown passwords.
- No user accounts, authentication, sessions, or database.
- No persistence — files are never written to disk.
- No batch/folder processing (single file per request).
- No public-internet deployment or multi-user hardening.
- No removal of *owner-only* permission restrictions as a separate feature
  (decrypting with the known password already produces an unrestricted PDF).

## Language & Stack

- **Language:** Python 3.10+
- **PDF engine:** `pikepdf` (>=9.0; wraps QPDF; handles RC4 40/128-bit, AES-128,
  AES-256). Pure-local C++ engine, no network use.
- **Web framework:** `Flask` (>=3.0)
- **Config parsing:** `PyYAML` (`yaml.safe_load`)
- **Testing:** `pytest` (>=8.0)
- **Packaging/tooling:** `pyproject.toml` + `uv.lock` (for `uv` users);
  `backend/requirements.txt` for the pip-based launcher scripts.

All dependencies operate fully offline. No CDN, telemetry, or network I/O.

## Architecture

Single-process Flask app, three independently testable layers:

1. **Core decryptor** (`backend/decryptor.py`) — pure function with no web
   knowledge. Input: PDF bytes + password (+ behavior flags). Output: decrypted
   PDF bytes. Raises typed errors. This is the only module that imports pikepdf.
2. **Config loader** (`backend/config.py`) — loads `config.yaml` with built-in
   defaults; missing file or missing keys deep-merge onto the defaults.
3. **Web layer** (`backend/app.py`) — Flask routes. Serves the static frontend,
   exposes the API endpoints, translates core errors into JSON responses, sets
   security headers.

The **frontend** (`frontend/`) is pure static assets (HTML/CSS/vanilla JS),
served by Flask. It performs the upload via `fetch`, receives the decrypted PDF
as a blob, and presents a result screen with a download button. It also does a
**fully local, in-browser pre-check** for whether the chosen PDF is encrypted at
all (see Frontend / UX).

## Project Structure

```
pdf-password-remover/
├── backend/
│   ├── __init__.py
│   ├── app.py                 # Flask routes + static serving + security headers
│   ├── decryptor.py           # core decrypt logic (pikepdf), typed errors
│   ├── config.py              # loads config.yaml, deep-merged onto defaults
│   ├── requirements.txt       # pikepdf, flask, pyyaml, pytest
│   └── tests/
│       ├── __init__.py
│       ├── test_decryptor.py  # core unit tests
│       └── test_app.py        # Flask route tests (test client)
├── frontend/
│   ├── index.html
│   ├── css/styles.css
│   └── js/app.js
├── config.yaml                # user-editable configuration
├── pyproject.toml             # project metadata + deps (uv)
├── uv.lock                    # locked dependency versions (uv)
├── run-mac-linux.sh           # launcher for macOS / Linux
├── run-windows.bat            # launcher for Windows
└── README.md
```

The launchers (`run-mac-linux.sh`, `run-windows.bat`) create a `.venv` on first
run, `pip install` from `backend/requirements.txt`, then start the app with
`python -m backend.app`. `pyproject.toml` / `uv.lock` are provided for users who
prefer `uv`.

## Configuration (`config.yaml`)

```yaml
server:
  host: 127.0.0.1        # localhost only — never expose on the network
  port: 5000
  debug: false

uploads:
  max_file_size_mb: 500
  allowed_extensions:
    - pdf

behavior:
  passthrough_unencrypted: true   # if PDF isn't encrypted, return it unchanged
  output_prefix: "unlocked-"      # output filename = <prefix><original name>

security:
  enable_csp: true                # send Content-Security-Policy: default-src 'self'

ui:
  min_loader_seconds: 2           # minimum time the "Unlocking…" loader stays visible
```

`backend/config.py` defines a matching `DEFAULTS` dict and deep-merges the user
file onto it, so the app runs even if `config.yaml` is absent or partial.

## Data Flow

1. Browser loads `/` → static frontend. The frontend fetches `GET /api/config`
   for the loader timing.
2. User selects/drops a PDF. The browser reads the file bytes locally and scans
   for the `/Encrypt` token:
   - **No `/Encrypt`** → show a "No password to remove" info modal and reset
     (no server round-trip).
   - **Encrypted** → user types the password and proceeds.
3. Frontend `POST /api/unlock` with multipart form: `file` + `password`.
4. Flask rejects early if `Content-Length` exceeds the configured cap (`413`).
5. Web layer reads bytes into memory (`BytesIO`), calls the core decryptor.
6. Core opens with pikepdf using the password, saves a decrypted copy to a
   `BytesIO`, returns the bytes.
7. Flask streams the bytes back with
   `Content-Disposition: attachment; filename="<prefix><name>.pdf"`.
8. Frontend turns the response into a blob URL and shows the download button.
   The loader popup is held for at least `ui.min_loader_seconds`.
9. Request ends; all in-memory bytes are released. Nothing persisted.

## API

### `GET /`
Serves `frontend/index.html`.

### `GET /css/<filename>` and `GET /js/<filename>`
Serve frontend CSS / JS assets.

### `GET /api/config`
Returns client-facing config as JSON: `{ "min_loader_seconds": <number> }`.

### `POST /api/unlock`
**Request:** `multipart/form-data` with:
- `file`: the PDF
- `password`: string (may be empty)

**Success (200):** body = decrypted PDF bytes,
`Content-Type: application/pdf`,
`Content-Disposition: attachment; filename="unlocked-<name>.pdf"`.

**Error (4xx):** JSON `{ "error": "<human-readable message>" }`:
- `400` — no file provided / not a `.pdf` / corrupt or invalid PDF / not
  password-protected (when passthrough is disabled).
- `401` — incorrect password.
- `413` — file exceeds `max_file_size_mb`.

## Core Decryptor Behavior

`decrypt(pdf_bytes, password, *, passthrough_unencrypted=True) -> bytes`

- **Encrypted + correct password** → returns decrypted PDF bytes.
- **Encrypted + wrong password** → raises `WrongPassword`.
- **Not encrypted** → if `passthrough_unencrypted` is true, return input
  unchanged; otherwise raise `NotEncrypted`.
- **Not a valid PDF / corrupt** → raises `InvalidPDF`.

Typed exceptions (`WrongPassword`, `NotEncrypted`, `InvalidPDF`) live in
`decryptor.py` and are mapped to HTTP responses by the web layer.

## Security & Privacy

- **Localhost binding:** server binds to `127.0.0.1` only. Not reachable from
  other devices.
- **No disk persistence:** uploaded and decrypted bytes live only in memory for
  the request lifetime.
- **Offline by construction:** no dependency makes network calls; the frontend
  bundles all assets locally (system-font stack, hand-written CSS, vanilla JS) —
  no CDN, fonts, or external scripts. The "is it encrypted?" pre-check runs
  entirely in the browser.
- **Content-Security-Policy:** `default-src 'self'` (when
  `security.enable_csp` is true) prevents the page from loading or contacting
  anything external — enforces the offline guarantee in the browser.
- **Filename sanitization:** original filename sanitized (basename only,
  non-`[\w\-_. ]` chars replaced with `_`) before being echoed in
  `Content-Disposition`.
- **Size cap:** requests above `max_file_size_mb` rejected with `413` via
  Flask's `MAX_CONTENT_LENGTH`.
- **Memory note:** a single 500 MB job may transiently use ~1–1.5 GB RAM
  (input + working copy + output). Acceptable for local single-user use;
  documented in the README.

## Frontend / UX

Single page branded **"PDF Unlocker"** with a header logo + tagline, footer
reminder that all processing is local, and two visual states on one screen plus
two modal overlays:

1. **Idle:** centered card, drag-and-drop zone ("Drop your PDF here / click to
   browse") with an inline icon, password field, "Unlock PDF" button. Selected
   filename shown once chosen. Inline error message area for retries.
2. **Result:** "✓ Your PDF is ready" with a large **Download** button (uses the
   blob URL) and an "Unlock another" reset link.

**Modals:**
- **Loader popup:** shown while unlocking, spinner + "Unlocking your PDF…", held
  on screen for at least `ui.min_loader_seconds` (default 2s) even if the
  request finishes sooner, so the user always sees it.
- **"No password to remove" info modal:** shown immediately (no server call)
  when the locally-scanned PDF has no `/Encrypt` token. Dismissing it resets the
  form, since there is nothing to unlock.

**Local encryption pre-check:** on file select/drop, the browser reads the bytes
and scans for the `/Encrypt` token that encrypted PDFs carry in their trailer.
Best-effort — if detection throws, the normal server unlock flow still runs.

Styling: clean modern card UI, soft shadows, accent color, responsive — all from
local CSS. No build step, no JS framework, no external resources. Static asset
links carry a `?v=` cache-busting query.

## Error Handling Summary

| Situation | Core | HTTP | User-facing message |
|---|---|---|---|
| Correct password | returns bytes | 200 | (download) |
| Wrong password | `WrongPassword` | 401 | "Incorrect password — please try again." |
| Not encrypted (passthrough on) | returns input | 200 | (download) |
| Not encrypted (passthrough off) | `NotEncrypted` | 400 | "This PDF is not password-protected." |
| Not encrypted (caught client-side) | — | — | "No password to remove" info modal (no request) |
| Invalid/corrupt PDF | `InvalidPDF` | 400 | "This doesn't look like a valid PDF." |
| Non-PDF extension / no file | — | 400 | "Please choose a PDF file." |
| Too large | — | 413 | "File exceeds the 500 MB limit." |

## Testing Strategy

- **Core (`test_decryptor.py`):** build small encrypted PDFs at test time with
  pikepdf, then assert: correct password decrypts and result opens without a
  password; wrong password raises `WrongPassword`; unencrypted input handled per
  flag; garbage bytes raise `InvalidPDF`.
- **Routes (`test_app.py`):** Flask test client — happy path returns
  `application/pdf` with a download disposition; wrong password returns `401`
  JSON; missing/non-PDF/corrupt file returns `400`; unencrypted passthrough
  returns `200`; `/` serves HTML; `GET /api/config` returns a numeric
  `min_loader_seconds`.
- **Offline check:** dependencies require no network; no external URLs present in
  frontend assets.

## Open Questions

None outstanding. Design approved by user on 2026-06-27; implemented.
