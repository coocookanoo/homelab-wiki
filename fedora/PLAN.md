> **Wiki Notice:** Project documentation should be copied to `/home/adminbill/projects/docs/<project-name>/` and pushed to [homelab-wiki](https://github.com/coocookanoo/homelab-wiki) so it appears at https://wiki.billsserver1.duckdns.org

---

# Homepage Desktop Widget — Build Docs

## Status: Working ✓

---

## Context
PyQt6 desktop widget for Fedora (KDE Plasma 6 / Wayland) that replicates the self-hosted Homepage dashboard at `http://10.0.0.1:3000/`. Always-on-bottom, frameless, draggable, with live system stats, service status dots, and an embedded browser that slides in when clicking a service card. Services, bookmarks, disk mounts, and API keys are editable at runtime via a built-in settings panel (no code changes needed).

Config source: `/home/adminbill/projects/homeserver/configs/homepage/`

---

## File Structure

```
/home/adminbill/projects/Fedoradesktop/
├── main.py                      # Entry point
├── config.py                    # Hardcoded defaults (fallback if settings.json absent)
├── settings_store.py            # Runtime config: load/save settings.json, fallback to config.py
├── settings.json                # User-editable config (services, bookmarks, disks, API keys)
├── requirements.txt
├── run.sh                       # Manual launch script
├── desktop-widget.service       # systemd user service for autostart
├── ui/
│   ├── __init__.py
│   ├── main_window.py           # Root window, workers, signal routing
│   ├── sliding_stack.py         # Animated slide between dashboard / browser / settings / btop
│   ├── browser_page.py          # Embedded QWebEngineView + nav bar
│   ├── btop_page.py             # Live server btop via ttyd over SSH (page 3)
│   ├── settings_page.py         # In-app settings editor (4-tab QTabWidget)
│   ├── stats_bar.py             # CPU / RAM / disk bar
│   ├── service_card.py          # Clickable card with status dot
│   ├── section_panel.py         # Section group (Media, Automation, etc.)
│   ├── bookmarks_bar.py         # Bottom bookmark links
│   └── styles.py                # QSS stylesheet + colour constants
└── workers/
    ├── __init__.py
    ├── system_stats_worker.py   # psutil thread — 5s interval
    ├── service_status_worker.py # HTTP ping thread — 60s interval
    └── api_data_worker.py       # Sonarr/Radarr/Prowlarr API — 60s interval
```

---

## Running

```bash
# Development (VSCode)
F5   # uses .vscode/launch.json — console: internalConsole (no terminal attached)
     # Debug output goes to the VS Code Debug Console panel, not the integrated terminal
     # Required: running with integratedTerminal causes zellij to hijack the pty

# Manual
./run.sh

# Autostart with desktop session
cp desktop-widget.service ~/.config/systemd/user/
systemctl --user enable --now desktop-widget.service
```

---

## Key Implementation Notes

### Wayland / XWayland
`WindowStaysOnBottomHint` only works under XWayland (X11 `_NET_WM_STATE_BELOW` hint). Force it in `main.py` before `QApplication`:
```python
if "WAYLAND_DISPLAY" in os.environ and "QT_QPA_PLATFORM" not in os.environ:
    os.environ["QT_QPA_PLATFORM"] = "xcb"
```

### WebEngine under XWayland
Running `QWebEngineView` inside an XWayland app causes GPU context failures. Required env vars (set in `main.py` and `.vscode/launch.json`):
```python
os.environ["QTWEBENGINE_CHROMIUM_FLAGS"] = "--disable-gpu --no-sandbox --in-process-gpu --log-level=3"
os.environ["QTWEBENGINE_DISABLE_SANDBOX"] = "1"
os.environ["LIBGL_ALWAYS_SOFTWARE"] = "1"   # forces Mesa software rendering in all subprocesses
```

### Sliding navigation
`SlidingStack` (`ui/sliding_stack.py`) animates between three pages using `QPropertyAnimation` (280ms, `InOutCubic`):

| Page index | Content | How to reach |
|---|---|---|
| 0 | Dashboard | `slide_to(0, direction=-1)` — ← Dashboard / Cancel in settings |
| 1 | Browser | `slide_to(1, direction=1)` — click service card or bookmark |
| 2 | Settings | `slide_to(2, direction=1)` — click ⚙ |
| 3 | btop (server) | `slide_to(3, direction=1)` — click 📊 btop |

`replace_page(index, widget)` swaps a page in-place (no animation) — used by `_rebuild_dashboard()` after settings are saved.

**Do NOT change window flags at runtime** — calling `setWindowFlag()` on a visible frameless window forces Qt to recreate the native window handle, causing it to vanish.

### Dashboard header buttons
Three buttons live at the top-right of the dashboard title bar:

- **📊 btop** — slides to an embedded live view of the homeserver's `btop` (page 3). See [btop page](#btop-page-uibtop_pagepy) below.

- **⌨ nvim** — launches the nvim-cockpit dev environment in an Alacritty window:
  ```python
  subprocess.Popen(["alacritty", "-e", "zellij", "--layout", "...cockpit.kdl"],
      start_new_session=True,
      stdin=subprocess.DEVNULL, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL,
      env=env)  # env strips QT_QPA_PLATFORM / LIBGL_ALWAYS_SOFTWARE
  ```
  `stdin/stdout/stderr=DEVNULL` is required — without it, zellij inherits the parent pty and hijacks the terminal instead of opening a new window. `start_new_session=True` fully detaches the process.

- **⚙** — slides to the settings page (page 2).

### btop page (`ui/btop_page.py`)
Shows a live `btop` session for the **homeserver** (`adminbill@10.0.0.1`) inside a `QWebEngineView` via [ttyd](https://github.com/tsl0922/ttyd).

**How it works:**
1. `BtopPage.load()` is called when the 📊 button is clicked.
2. `_ensure_ttyd()` kills any orphaned ttyd on port 7681 (`fuser -k 7681/tcp`), then spawns:
   ```
   ttyd --writable -p 7681 -t fontSize=14 -t disableLeaveAlert=true -- ssh -t adminbill@10.0.0.1 btop
   ```
3. A 700ms `QTimer` fires before loading `http://localhost:7681` — gives ttyd time to bind the port.
4. `BtopPage.stop()` terminates ttyd on widget close.

**Requirements:**
- `sudo dnf install ttyd` (one-time)
- Passwordless SSH key for `adminbill@10.0.0.1` (already configured)
- `10.0.0.1` in `~/.ssh/known_hosts` (already added)

**Gotcha — orphaned ttyd:** If the widget is force-killed, ttyd keeps running on port 7681. On next open, `_ensure_ttyd()` calls `fuser -k 7681/tcp` to clear it before starting fresh.

**Debug console note:** Must run with `"console": "internalConsole"` in launch.json — `integratedTerminal` gives the process a pty that zellij/ttyd will hijack.

### Settings panel (`ui/settings_page.py`)
Four-tab `QTabWidget`:

| Tab | Widget | What you can edit |
|---|---|---|
| Services | `QTreeWidget` (sections → service rows) | name, URL, description; add/delete sections and services |
| Bookmarks | `QTableWidget` | label, URL; add/delete rows |
| Disk Mounts | `QTableWidget` | label, path; add/delete rows |
| API Keys | `QTableWidget` (fixed 3 rows) | URL and API key for Sonarr, Radarr, Prowlarr |

All cells are double-click editable. **Save** writes `~/.config/homewidget/settings.json`, emits `settings_saved` → `MainWindow._on_settings_saved()` rebuilds the dashboard and restarts workers with the new config. **Cancel / ← Dashboard** slides back without saving.

### Runtime config (`settings_store.py` + `settings.json`)
`settings_store` is a module-level singleton:
- Loads `~/.config/homewidget/settings.json` on first access; caches the result.
- Falls back to `config.py` defaults if the file is absent or unreadable.
- `save(data)` writes the file and updates the cache.
- `invalidate()` clears the cache (forces re-read on next access).

`config.py` is now a **fallback only** — it is not read at runtime unless `settings.json` is missing. Do not rely on editing `config.py` for live changes.

Workers and `BookmarksBar` receive their config as constructor parameters (injected by `MainWindow._init_workers()` / `_build_dashboard()`), so rebuilding the dashboard or restarting workers always picks up the latest `settings_store` values.

### Browser nav bar
Three buttons: **← Dashboard** (always returns to widget), **→** (forward in page history), **⌂ Reload** (refresh current page).

`← Dashboard` always goes home regardless of browser history — SPAs like Jellyfin push many `pushState()` entries that would otherwise trap the user. Browser history is cleared after each fresh load (via `loadFinished`) so the blank-page `setHtml()` call from `clear()` doesn't add an extra back step.

**Init order gotcha:** `_build_view()` must be called before `_build_nav()` in `BrowserPage.__init__` — the nav bar's reload button references `self._view` via a lambda, which must exist at connection time.

### Worker architecture
```
SystemStatsWorker(disk_mounts)   → stats_ready(dict)        → StatsBarWidget.update_stats()
ServiceStatusWorker(services)    → status_ready(str, bool)  → card.set_status()
ApiDataWorker(api_config)        → api_data_ready(str, dict) → card.update_api_data()
                                 → api_error(str, str)       → card.update_api_data("–")
```
All workers are `QThread` subclasses with internal `msleep()` loops. Cross-thread signals are auto-queued by PyQt6 — no locking needed. Workers are stopped and recreated (with fresh config) whenever settings are saved.

### SSL / self-signed certs
- `requests`: `verify=False` + `urllib3.disable_warnings()` for service status pings
- `QWebEngineView`: `page().certificateError.connect(lambda e: e.acceptCertificate())`

### Position persistence
Saved to `~/.config/homewidget/HomepageWidget.conf` via `QSettings` on close.

---

## Configuration

### Runtime editing (preferred)
Click ⚙ in the widget header. Changes are saved to `~/.config/homewidget/settings.json` and take effect immediately — no restart needed.

### Manual file edit
Edit `~/.config/homewidget/settings.json` directly, then restart the widget.

### config.py fallback values

| Variable | Purpose |
|----------|---------|
| `WINDOW_X/Y` | Default position (overridden by saved QSettings) |
| `WINDOW_WIDTH/HEIGHT` | Fixed window size (1200×760) |
| `STATS_REFRESH_MS` | System stats interval (5000ms) |
| `SERVICE_REFRESH_MS` | HTTP ping interval (60000ms) |
| `API_REFRESH_MS` | Sonarr/Radarr/Prowlarr interval (60000ms) |
| `DISK_MOUNTS` | Disk paths to monitor |
| `SECTIONS` | Service groups, names, URLs |
| `SONARR/RADARR/PROWLARR_API_KEY` | API keys |
| `BOOKMARKS` | Bottom bar links |

---

## Dependencies

| Package | Version | Source |
|---------|---------|--------|
| PyQt6 | 7.0.0 | system (`python3-pyqt6`) |
| PyQt6-WebEngine | 6.10.1 | system (`python3-pyqt6-webengine`) |
| psutil | 7.0.0 | pip |
| requests | 2.32.5 | pip |
| ttyd | system | `sudo dnf install ttyd` |

Install system packages (pip versions have Qt version conflicts):
```bash
sudo dnf install python3-pyqt6-webengine ttyd
```

---

## Known Harmless Console Output

| Message | Source | Why harmless |
|---------|--------|-------------|
| `Sandboxing disabled by user` | Chromium | Expected — `QTWEBENGINE_DISABLE_SANDBOX=1` |
| `Feature-Policy header: payment` | Homepage app | Next.js sending unsupported header |
| `SharedImageManager::ProduceMemory` | Chromium | GPU texture cleanup noise |
| `Cross-Origin-Opener-Policy` | Homepage app | Served over HTTP, not HTTPS |
