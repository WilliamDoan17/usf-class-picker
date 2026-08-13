# Build Plan

Phases build on the decisions in `ARCHITECTURE.md`. Each phase should be
independently runnable/testable before moving to the next — no phase depends
on later ones existing.

## Phase 0 — Project setup
- [x] Repo structure per `ARCHITECTURE.md` (`scraper/`, `tracker/`, `app/`, `docs/`)
- [x] `pyproject.toml` (deps: `httpx`, `PySide6`, `apscheduler`; dev extras: `pytest`, `ruff`)
- [ ] SQLite schema drafted: tracked classes, poll history
- [ ] Types: ClassSection Shape

## Phase 1 — Scraper (client + parser + service)
- [ ] Capture the real POST requests the Staff Search site sends (form fields,
      headers, session/cookie behavior) — needed before writing the client
- [ ] Capture the semester `<select>` options (value/label pairs) from the
      Staff Search form — decides whether `semester` is stored as a raw term
      code with a computed label, or something else. Held open per
      `ARCHITECTURE.md` §4a until this real data is in hand.
- [ ] `client.py`: session handling, retries/backoff
- [ ] `parser.py`: raw response → `ClassSection` objects
- [ ] `service.py`: `search_classes(criteria) -> list[ClassSection]`
      (in-memory only, no persistence — see `ARCHITECTURE.md` §4a)
- [ ] Validate against real searches by criteria from the README (semester,
      CRN, subject/number, title, professor, credits)
- **Exit criteria**: can run a search from a throwaway script and get correct
  structured results — no UI or storage needed yet.

## Phase 2 — Storage
- [ ] SQLite tables per `ARCHITECTURE.md` §4a / `SCHEMA.md`:
      `tracked_classes` (static class info + `untracked_at` for soft
      delete, `UNIQUE(crn, semester)`) and `poll_history` (seats_available,
      status, polled_at, FK to tracked class)
- [ ] `store.py`: add tracked class, soft-delete (untrack) a tracked class,
      record poll result, read latest known status per tracked class
      (latest `poll_history` row — not a duplicated column on
      `tracked_classes`)
- **Exit criteria**: a tracked class and its poll history can be persisted
  and read back without the tracker or UI existing yet.

## Phase 3 — Tracker / poller
- [ ] Background poll loop (own thread), adaptive interval logic from
      `ARCHITECTURE.md` §2 (hourly default, ramp during high-churn windows)
- [ ] Jitter + rate-limit backoff
- [ ] Diff logic: detect open-seat transitions per tracked class
- **Exit criteria**: can track a class via script/CLI and see poll history
  accumulate in SQLite with correct interval behavior.

## Phase 4 — Notifications
- [ ] Native tray notification (`QSystemTrayIcon.showMessage`)
- [ ] Fallback channel (email or ntfy.sh/Pushover) for when the app isn't
      running or tray notifications are unavailable (see Linux caveat)
- **Exit criteria**: a detected seat opening reliably reaches the user
  through at least one channel.

## Phase 5 — Desktop UI (PySide6)
- [ ] Search form (criteria from README) + results list
- [ ] Track/untrack action from results
- [ ] Tracked-classes view with live status
- [ ] System tray icon wired to the poller from Phase 3
- **Exit criteria**: full loop works end-to-end from the UI — search, pick,
  track, get notified — on the dev machine.

## Phase 6 — Cross-platform + packaging
- [ ] Verify tray/notifications on target Linux setup (Hyprland: confirm
      waybar tray + mako/dunst present, or document the gap)
- [ ] Auto-start on login: Windows (Startup folder/registry), Linux
      (`.desktop` autostart entry)
- [ ] Packaging (e.g. PyInstaller) for distributing without a Python install
- **Exit criteria**: app installs and auto-starts cleanly on both a Windows
  machine and the target Linux/Hyprland setup.

## Phase 7 — Auto-registration (future, gated)
Not started until Phases 0–6 are solid. Distinct risk profile: requires
storing USF credentials and taking real registration actions.
- [ ] Design credential storage (OS keychain, not plaintext/SQLite)
- [ ] Confirm the registration POST flow (likely needs an authenticated
      session against OASIS/MyUSF, separate from the public Staff Search
      scraper)
- [ ] Explicit confirm-before-register step, not silent auto-submit, at
      least for the first version
- [ ] Revisit `ARCHITECTURE.md` §1 threat model before building this phase —
      credential handling changes the risk profile even for single-tenant use
