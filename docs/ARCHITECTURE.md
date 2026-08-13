# Architecture Decisions

## 1. Deployment model: local, single-tenant

Not a public multi-tenant website. Once auto-registration is in scope, the app
must hold USF login credentials and take real registration actions — running
that as a hosted service for multiple users would mean storing other people's
credentials and owning the risk of USF's ToS/security exposure for all of them.

Instead: one instance per user, run on infrastructure that user controls
(their own machine, a Raspberry Pi, or a small personal VPS). Same codebase,
just not multi-tenant.

## 2. Polling strategy: adaptive, not flat hourly

Hourly is a fine baseline but misses fast-fill windows (registration opening,
add/drop period) where seats can go in minutes. Poll adaptively:

- Default: every 60 min.
- Ramp to every 1–5 min during known high-churn windows (semester
  registration opens, add/drop period, or when manually flagged "watch
  closely").
- Add jitter to request timing (avoid a perfectly regular bot pattern).
- Back off / respect basic rate limits so the scraper doesn't get IP-blocked.

## 3. Stack

- **Scraper**: Python + `requests`/`httpx`. The target site is static
  (plain POST/GET, no JS rendering needed) so no Selenium/Playwright required.
- **Storage**: SQLite — tracked classes, poll history, last-known status.
- **Interface**: PySide6 (Qt for Python) native desktop app, not a browser
  UI. Cross-platform by default (Windows, Linux, macOS) — same codebase, no
  per-OS UI work needed. The UI calls scraper/tracker service functions
  directly in-process — no HTTP layer, no server process to keep alive
  separately. Runs with a system tray icon so the poller keeps going in the
  background; clicking the tray icon opens the search/track window.
  Background polling runs on its own thread (feeding results back to the UI
  via Qt signals) so it never blocks the UI thread.
- **Scheduler**: a background thread (or APScheduler) driving the poll loop,
  started alongside the Qt app and running for the app's lifetime.
- **Notifications**: native OS notifications via `QSystemTrayIcon`, which
  routes to each platform's native notification system (Windows toast,
  GNOME/KDE notification centers on Linux, Notification Center on macOS).
  Email/push (ntfy.sh / Pushover) as a fallback for alerts while the app
  isn't running.
- **Auto-start**: registered to launch on login (Windows: registry/Startup
  folder; Linux: `.desktop` autostart entry), so it's just always running —
  no manual "start the server" step.
- **Linux tray/notification caveat**: `QSystemTrayIcon` needs a
  `StatusNotifierItem` host (e.g. waybar's tray module) and a
  notifications needs a daemon implementing `org.freedesktop.Notifications`
  (e.g. mako, dunst) to actually appear. GNOME/KDE bundle both; minimal
  Wayland setups (e.g. Hyprland) don't ship either by default — they depend
  on what the user's compositor config already runs. Both fail silently
  rather than erroring if missing, so verify on the actual target Linux
  setup rather than assuming parity with Windows/GNOME/KDE.

## 4. Scraper module design

Three layers, so the tracker never touches HTTP/parsing details:

- **Client**: raw POST/GET calls to the Staff Search endpoints. Owns
  session/cookies, retries with backoff.
- **Parser**: turns the raw response into structured `ClassSection` objects.
  Isolated on purpose — this is the part most likely to break if USF changes
  markup, and a break here shouldn't touch the client or the tracker.
- **Service**: exposes one clean entry point, e.g.
  `search_classes(criteria) -> list[ClassSection]`. Writes results to
  SQLite; the tracker reads from SQLite rather than calling the scraper
  directly, so tracker logic is decoupled from how the data got there.

## 5. Development workflow

- Run the desktop app locally during development: `python app/main.py` launches
  the Qt app, which starts the poller thread and shows the tray icon.
- For always-on polling without keeping a personal machine running, this
  model trades off — a desktop app needs a logged-in session to run. If
  true 24/7 unattended polling matters more than a native window, consider
  running the poller headless (no Qt) on a Pi/VPS and keeping the desktop
  app just as the local UI that reads the same SQLite state. Revisit if this
  becomes a real constraint.
- Proposed structure:

```
usf-class-picker/
  scraper/      # client.py, parser.py, service.py
  tracker/      # poller.py, notifier.py, store.py
  app/          # main.py (Qt app), ui/ (windows, tray icon)
  docs/
    ARCHITECTURE.md
  requirements.txt (or pyproject.toml)
```
