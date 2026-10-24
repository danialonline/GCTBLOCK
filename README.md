# GCTBLOCK
A app focused around Cold Turkeys Blocking but Google Calendar and Tasks API integrated. (This app was vibe coded with AI, so some things may not respond or work properly)


11# GCT BLOCK

GCT BLOCK is a local-first Windows productivity app built with Python, PySide6, and SQLite. The current development foundation includes the reference-inspired interface, local configuration persistence, opt-in Google Calendar/Tasks integration, a testable website-policy evaluator, and an optional Chrome/Edge extension for explicitly imported website rules. No Google account is connected until the user explicitly supplies a Desktop OAuth client and completes Google's browser consent flow.

## Development setup

1. Install 64-bit Python 3.10 or newer (Python 3.12 is recommended). The launcher uses `python` from `PATH` or falls back to the Windows `py -3` launcher.
2. From this folder run:

   ```powershell
   .\run_dev.ps1
   ```

   The script installs `requirements.txt`, then launches the app from `src`.
3. To run tests:

   ```powershell
   py -3 -m pip install -r requirements.txt
   $env:PYTHONPATH = "src"
   py -3 -m pytest
   ```

Alternatively, install the package in editable mode with `py -3 -m pip install -e ".[dev]"` and launch with `gct-block`.

## Implemented in this foundation

- Seven navigable PySide6 surfaces: Welcome, Web/App Blocks, Device Blocks, Tasks & Calendar, Schedule, Statistics, and Settings.
- Shared dark/light/system design tokens, pixel-grid backdrop, navigation, custom title controls, Windows native resize hit testing, and Qt high-DPI support.
- Local profile creation/renaming/removal; domain block rules and allowed-domain exceptions; executable/package/window-title rule records; per-profile weekly periods with overnight times, optional dates and time zone; stored Lock Mode preferences; and weekly schedule entry CRUD.
- Persistent theme, first-weekday, and time-format preferences.
- Versioned SQLite storage (schema version 3, with transactional v1/v2 upgrades) for profiles, rules, schedules, settings, and the Google offline cache. Normal app data is saved in the current user's application-local data directory; `GCT_BLOCK_DATA_DIR` can override the directory for development.
- Optional Google Calendar read access and Google Tasks read/write through the official APIs. Network/OAuth work runs in Qt's thread pool; calendar/task selections and fetched data are cached locally for offline display. Calendar events are read-only. New tasks and completion changes are sent to Google only when the user performs those actions.
- OAuth refresh tokens are encrypted with Windows DPAPI for the current Windows user and written atomically to `google-token.dpapi` beside the database. The selected OAuth client JSON is not copied; only its file path is stored as a local preference. Disconnect removes the protected token, Google selections/catalogs, and cached Google events/tasks.
- File and console diagnostics in `gct-block.log`.
- Empty and unavailable states that distinguish future work from real functionality. No demo or sample records are preloaded.
- Twelve editable website category starter lists are stored as versioned package data, seeded into SQLite, and can be customized or restored per category. Domain matching is case-insensitive, IDNA-normalized, and matches exact hosts or dot-delimited subdomains. A pure policy evaluator gives explicit allowed exceptions precedence over explicit block rules and category blocks; across active profiles, any block wins with profile-ID tie-breaking. The opt-in extension below applies exported policies only after explicit user import.
- Weekly profile periods have a pure injectable-instant evaluator for weekday/time matching, overnight periods, date bounds, multiple simultaneous periods, and named IANA time zones. This reports schedule matches only; it does not activate profiles or enforce rules. Overnight periods belong to their start day/date. Repeated fall-back wall-clock times are included in both occurrences; spring-forward gaps are not synthesized. IANA rules come from the `tzdata` package on Windows.
- Statistics includes explicitly opt-in Windows foreground executable measurement, local application aggregation for the seven reporting periods, and CSV export. It records executable identity and sampled foreground seconds only, without window titles or website content. Measurement runs only while the app is open and tracking is started; it is not idle-adjusted, screen-on time, or proof of user interaction. Website tracking, charts, and background tracking are not implemented.
- An opt-in Chrome/Edge Manifest V3 extension under `browser_extension/` applies a selected profile's website policy through the browser Declarative Net Request API. The profile editor exports JSON; the user explicitly imports it through the extension options page. Allowed exceptions override matching blocks. Policy changes require another explicit export/import; no schedule, profile activation, auto-start, or automatic sync is wired to the extension.
- Application rules can be added by choosing an executable, scanning a selected folder with preview (recursive scans inside Windows/Program Files paths are refused), discovering packaged Windows Start-menu app IDs, or entering a window-title pattern. An explicitly started, foreground-only Windows monitor reports configured matches on a background thread; it does **not** prevent launches or terminate processes. Packaged/multi-process app IDs and changing window titles are not reliable standalone enforcement identities.

## Optional Google setup

1. In Google Cloud Console, create/select a project, enable **Google Calendar API** and **Google Tasks API**, and configure the OAuth consent screen for the intended account/audience.
2. Create an OAuth client ID with application type **Desktop app** and download its JSON file. Keep this file private and do not add it to source control.
3. In GCT BLOCK, open **Settings**, choose that Desktop OAuth JSON, then choose **Connect Google account**. The app opens Google's sign-in/consent page in the default browser. Enter credentials only on Google's site.
4. Select the calendars and task lists GCT BLOCK should use. On **Tasks & Calendar**, use the explicit refresh buttons to sync selected data. Cached events/tasks are shown offline; failures are surfaced and do not imply that remote writes succeeded.
5. Use **Disconnect** to remove the local token and Google cache. This does not delete remote Google data.

Requested OAuth scopes are `https://www.googleapis.com/auth/calendar.readonly` and `https://www.googleapis.com/auth/tasks`. Calendar is read-only; task creation/completion uses the Google Tasks API. Google Task due dates are dates (not timed reminders). No Google account, client secret, or authorization token is included in this project.

## Architecture and subsystem dependencies

The UI is split into navigation/page modules and shared widgets; `config/theme.py` owns the palette. Pages call a small `Database` persistence API rather than issuing SQL themselves. Google OAuth and API calls are separated into `integrations/google_auth.py`, `integrations/google_sync.py`, and a Qt background coordinator. The database layer is independent of Qt UI behavior and opens with foreign keys, busy timeout, WAL journaling for file databases, parameterized statements, and transactional schema migrations (`PRAGMA user_version`).

Schema responsibilities:

| Tables | Purpose |
| --- | --- |
| `app_settings` | Persistent local preferences |
| `block_profiles`, `profile_lock_settings` | Profile identity and saved lock preferences (not enforced) |
| `website_categories`, `category_domains`, `profile_categories`, `profile_domains`, `profile_app_rules` | Editable website category data, per-profile category selections, domain block/allow rules, and application identity rules |
| `weekly_schedule_periods`, `user_schedule_entries` | Per-profile weekly periods and user-authored schedule records |
| `cached_calendar_events`, `cached_tasks` | Selected Google data cached for offline display |
| `usage_sessions` | Local usage-measurement foundation |
| `enforcement_events` | Diagnostic/audit event foundation |

## Explicit phase boundary

Application launch prevention, Windows-wide website/network enforcement, scheduled device actions, startup registration, tray/background service, password-protected focus-lock enforcement, website tracking, and charts are not implemented and are not represented as active controls. Device Blocks provides manual **Lock Windows session**, **Sign out**, and **Shut down** actions using documented Windows session APIs. Sign-out and shutdown require a separate confirmation, do not force-close applications, and may be unavailable if Windows privileges or policy do not permit the request. These actions do not enforce a GCT BLOCK focus session. Schedule entries, rule records, saved focus-lock preferences, category selections, and block profiles are configuration records only. The optional browser extension applies a manually imported policy in browser profiles where the user installed and enabled it; it does not inspect other browsers or network traffic outside its host browser. Website policies are not connected to schedules or automatic activation. The opt-in foreground monitor only detects and reports matching apps; no process is terminated or prevented from starting. The separate opt-in Statistics monitor locally measures foreground executable identity while GCT BLOCK is running; it does not record window titles or website content. No Windows policies are changed. Google OAuth is optional and user-initiated; the app does not collect a Google password.

## Optional Chrome / Edge website enforcement

1. In Chrome or Edge, open `chrome://extensions` or `edge://extensions`, enable **Developer mode**, and choose **Load unpacked**. Select this project's `browser_extension` folder. This is an optional, user-installed extension; GCT BLOCK does not install it or change browser settings.
2. In GCT BLOCK, edit a web/app profile and choose **Export browser policy**.
3. Open the extension's **Details → Extension options** page, choose the exported JSON, and select **Import policy**. The browser stores dynamic domain rules and applies them in that browser profile.
4. Repeat export/import after changing categories or rules. To stop enforcement, choose **Clear GCT BLOCK rules** in extension options or disable/remove the extension.

The extension blocks exact domains and subdomains in browser requests. This is browser-scoped deterrence, not a security boundary: another browser/profile, private browsing when the extension is not enabled there, another device, alternate domains, or non-browser traffic may not be covered. The bundled lists are GCT BLOCK's small curated starter examples, are not sourced from a live maintained third-party feed, and are not exhaustive. No network or DNS settings are changed.

Domain input is stored as a DNS name, normalized to lower-case ASCII with Python's standard-library IDNA codec. Exact domain/subdomain matching is used by the policy evaluator and optional browser extension, not by a Windows-wide network blocker.

Profile activation, schedule/auto-start activation, GCT BLOCK focus-lock enforcement, Windows-wide enforcement, website measurement, charting, CSV export, and persistent background tracking remain later work. Installer, PyInstaller release packaging, and MSI/uninstaller work are also later deliverables.

## Validation

The focused test suite covers fresh and forward SQLite migrations, category data and profile persistence, domain-matching boundaries and precedence, application selection/matching helpers, local cache behavior, profile/rule/schedule validation, OAuth client/token-store boundaries, mocked Google API pagination and task writes, stable enums, and reporting-period choices. Run it with the command above. GUI rendering and native Windows resize behavior must still be exercised on a Windows desktop with PySide6 installed.
