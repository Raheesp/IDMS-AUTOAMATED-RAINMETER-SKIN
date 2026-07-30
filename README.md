# AutomationReminder

A [Rainmeter](https://www.rainmeter.net) desktop widget for AM Motors staff. It puts three things
on your desktop that would otherwise mean opening three different tools:

- **A live view of your UiPath automation schedule** — next run, countdown, gaps between tasks, and run history — read straight from Windows Task Scheduler.
- **Live Booking / Docket / Enquiry counts** from the IDMS portal, refreshed automatically.
- **An IDMS report downloader** for [sos.ammotors.in](https://sos.ammotors.in) — 32 report types, on-demand or scheduled, with one-click email delivery and flexible date ranges.

It's built as two Rainmeter skins that sit side by side on your desktop, backed by PowerShell,
Python (via [Playwright](https://playwright.dev)), and Lua.

<p align="center">
  <em>AUTOMATION bar → click to expand into a full schedule + history dashboard.<br/>
  PORTAL DATA bar → click IDMS to expand the report picker.</em>
</p>

---

## Contents

- [Quick start](#quick-start)
- [What each piece does](#what-each-piece-does)
  - [AUTOMATION bar](#automation-bar)
  - [PORTAL DATA bar](#portal-data-bar)
  - [IDMS report panel](#idms-report-panel)
  - [Scheduling downloads](#scheduling-downloads)
  - [Emailing reports](#emailing-reports)
  - [Theming](#theming)
- [How it's built](#how-its-built)
- [Where your data lives](#where-your-data-lives)
- [Sharing this with your team](#sharing-this-with-your-team)
- [Troubleshooting](#troubleshooting)
- [Known limitations](#known-limitations)
- [Project structure](#project-structure)
- [License](#license)

---

## Quick start

**You need:**
- [Rainmeter](https://www.rainmeter.net) installed (free).
- This whole `AutomationReminder` folder, copied to your PC.
- Your IDMS login for `sos.ammotors.in`.

**Install:**

1. Open the `Downloads` folder inside `AutomationReminder`.
2. Double-click **`Setup.bat`**.
3. It installs a real Python (skips the Microsoft Store stub), the `playwright` /
   `pandas` / `openpyxl` packages, and a Chromium browser for Playwright to drive.
4. It asks **"Should this PC show its own live Task Scheduler?"** — press Enter
   (default = yes) if this PC runs the real automation and should show its own
   live schedule. Only answer `N` if you're deliberately setting up a view-only
   copy of *someone else's* schedule (see [Sharing this with your team](#sharing-this-with-your-team)).
5. It registers two background scheduled tasks (data refresh + IDMS availability
   check) and opens Notepad so you can enter your IDMS password.
6. In Rainmeter: right-click the tray icon → **Manage** → load
   `AutomationReminder.ini`, then load `AutomationReminder\Downloads\Downloads.ini`
   the same way. Drag the two bars wherever you like (PORTAL DATA is meant to sit
   just under the main AUTOMATION bar).

Full walkthrough, every field explained, and the day-to-day usage guide: **[SETUP.md](SETUP.md)**.
There's also a standalone HTML user guide at
[`Downloads/IDMS_User_Guide.html`](Downloads/IDMS_User_Guide.html) — open it in any
browser, no server needed.

---

## What each piece does

### AUTOMATION bar

A slim bar showing the status dot, next task name, a live countdown, and the next
three tasks in the queue. Click the brand icon to jump straight to Task Scheduler.
Click **ALL TASKS** to expand the full dashboard:

- Every task under a configurable Task Scheduler folder (`\Uipath Automation\` by
  default — change `TaskFolder=` in `AutomationReminder.ini`), with its next run
  time, the **gap** before it, and a countdown.
- A **run history** feed (last 48 hours) with start time, duration, and pass/fail,
  pulled from the Task Scheduler operational log — needs *Task Scheduler → Enable
  All Tasks History* turned on to populate.
- Scroll with the mouse wheel when the list is longer than the visible area.

This is powered by `RefreshTasks.ps1`, which queries Task Scheduler once a minute
and writes two small feed files that the widget reads every second.

### PORTAL DATA bar

Shows live **Booking / Docket / Enquiry** counts pulled from the IDMS portal,
refreshed automatically every 15 minutes by a background scheduled task
(`AM DataDownloader`). Click **RUN NOW** to force an immediate refresh. Click
**UPDATE PASSWORD** any time your IDMS login stops working — it opens the
credentials file in Notepad.

### IDMS report panel

Click **IDMS ⌄** on the PORTAL DATA bar to open the report picker: 32 report
types across two columns, each with its own checkbox. A few worth knowing about:

- **Follow Ups** is split into four picks — Current, Overdue, Upcoming, and
  "All 3" (grabs all three as separate files in one click) — so you're not stuck
  downloading all of them every time you only need one.
- **CRM Enquiry** is split into **Active** and **Lost** — these are two
  genuinely different, independently-downloadable views on the CRM Enquiry List
  page, not one entry that happened to always grab whichever tab was open.
- Hover any checkbox to see a tooltip with that report's real data availability
  (earliest/latest date with actual data), refreshed automatically once a day so
  you know before you download whether your chosen range will return anything.

**Date range**, applied to whichever reports are checked:

| Option | What it does |
|---|---|
| **AS-IS** *(default)* | Downloads immediately with whatever range the report page already shows — no filtering, fastest option. |
| **TODAY** | Every report's range set to today. |
| **THIS MONTH** | 1st of this month through today. |
| **LAST MONTH** | The full previous calendar month. |
| **LAST 2 MONTHS** | 1st of two months ago through today. |
| **CUSTOM…** | Opens `idms_range.txt` in Notepad — edit `FROM=`/`TO=` (`YYYY-MM-DD`), save, close. |

If a report comes back empty on your chosen range, it's **automatically retried**
with This Month, then Last Month, then Last 2 Months, then the report's own
as-is view, before being marked failed — a report only ends up in
`0_FAILED_REPORTS.txt` after failing across a couple of months of data, not just
your original window. If a wider range was what actually worked, that's called
out explicitly in the status line and in `0_USED_WIDER_DATE_RANGE.txt`.

Click **DOWNLOAD** — the status line shows live per-report progress and a final
summary. Files land in `%LOCALAPPDATA%\AutomationReminder\idms_downloads\`
(**OPEN FOLDER** jumps straight there).

**BULK DUMP** mode (toggle **CREATE DUMP**) instead builds a month-by-month
archive of a report from a chosen starting month through today, into one
combined file or one file per month.

### Scheduling downloads

Click **SCHEDULE TASK** instead of DOWNLOAD to run the same selection unattended,
on a real Windows Scheduled Task (so it fires even if Rainmeter isn't running).
Opens a draft in Notepad:

```
TIME=09:00
PATH=
REPEAT=once
```

- `TIME=` — 24-hour `HH:MM`.
- `PATH=` — optional output folder (defaults to the normal downloads folder).
  Add `DATEFOLDER=today` or `DATEFOLDER=yesterday` to save into a dated
  `DD-MM-YYYY` subfolder automatically, every time it runs.
- `REPEAT=` — `once`, `daily`, `weekly` (add `DAYS=Mon,Wed,Fri`), `monthly` (add
  `MONTHS=`, day-of-month from `DATE=` or today's), or `interval` (add
  `EVERY=<minutes>`).
- `NAME=` — optional friendly name shown in Task Scheduler and the SCHEDULES list.

Save, close Notepad, click **CONFIRM SCHEDULE**. See everything you've scheduled
via the **SCHEDULES** button. **DELETE TASK** removes one — paste its exact name.

### Emailing reports

Click **EMAIL** to send the current selection via Outlook (reuses a running
Outlook instance, or opens and later closes one of its own). A Notepad draft
lets you set `TO=` / `CC=` / `SUBJECT=` / `BODY=` and either `TIME=NOW` (send
immediately) or the same `TIME=`/`REPEAT=` vocabulary as scheduling, for
recurring email delivery.

Every send **always freshly downloads** the selected reports first — it never
silently reuses old files. If that fresh download can't complete (portal hiccup,
network issue), it automatically falls back to the most recently downloaded copy
of each report already on disk, and says so clearly in a banner at the top of
the email itself and in the status line — so you either get fresh data, clearly
labelled not-quite-fresh data, or an honest failure, never a silent lie about
what you're looking at.

### Theming

Both bars share one theme system. Open the AUTOMATION dashboard and look at the
**THEME** row, top-right: click any dot to instantly recolor both skins.

| Theme | Look |
|---|---|
| Sunset *(default)* | Warm gold / coral — golden-hour wallpapers |
| Midnight | Cool blue — dark wallpapers |
| Glass | Frosted, highly translucent — blends with anything |
| Aurora | Violet & teal — vibrant wallpapers |
| Mono | Grayscale — busy/high-contrast wallpapers |
| Ivory | The only **light** card — bright/white wallpapers |
| Ember | Deep crimson & gold — red/racing-livery wallpapers |
| Emerald | Deep jade green — nature/forest wallpapers |
| **Custom** | Click the dashed dot — opens `Custom.inc` in Notepad, type in any RGB colors you want, save, done |

Your choice is stored per-PC in `@Resources\Settings.inc` and never synced —
each machine can run its own theme without conflicting with anyone else's copy.

---

## How it's built

Two independent Rainmeter `.ini` skins, sharing a theme system, backed by three
languages doing three different jobs:

- **Lua** (`AutomationTasks.lua`, `Downloads\IdmsEngine.lua`,
  `Downloads\DownloadEngine.lua`) — the UI logic: reading feed files, formatting
  countdowns, driving buttons, polling background jobs for a result.
- **PowerShell** — everything that talks to Windows: Task Scheduler queries and
  registration, Outlook COM automation, credential/file management. Every
  PowerShell entry point writes a status file the Lua side polls, so long-running
  work (a 30-report download, sending an email) never blocks the UI.
- **Python + Playwright** (`Downloads\IdmsDownload.py`) — drives a real (headless)
  Chromium browser against the IDMS portal to log in and pull each report's
  export. Handles three different download mechanisms the site uses across its
  report pages (direct browser download, popup-to-signed-link, and client-side
  blob export), auto-retries on empty results with progressively wider date
  ranges, and holds a stale-safe lock so two downloads (a schedule firing during
  an ad-hoc click, say) never run against the portal at the same time.

Long-running PowerShell/Python work is launched via Rainmeter's `RunCommand`
plugin with a generous timeout (report downloads can legitimately take several
minutes for a large selection) and reports progress through a status file the
UI polls — with a client-side give-up timer on every poll, so even a killed or
hung backend process surfaces a clear error instead of leaving the panel frozen.

---

## Where your data lives

| What | Where | Synced? |
|---|---|---|
| The skin itself (`.ini`/`.lua`/theme files) | This folder | Yes — safe to share/sync freely |
| Your IDMS password (`creds.txt`) | `%LOCALAPPDATA%\AutomationReminder\` | **No** — machine-local, never touches this folder |
| Downloaded reports | `%LOCALAPPDATA%\AutomationReminder\idms_downloads\` | No |
| Live task schedule feed (`_taskdata.txt`, `_taskhist.txt`) | `%LOCALAPPDATA%\AutomationReminder\` | No, unless you deliberately set up sharing (below) |
| Theme choice, live-vs-viewer mode (`Settings.inc`) | `@Resources\Settings.inc`, in this folder | Yes, but each PC's value applies locally |

Copying or sharing this whole folder never hands over your password or your
downloaded reports — those only ever live under your own `%LOCALAPPDATA%`.

---

## Sharing this with your team

**Giving someone the whole tool:** just hand them the folder and point them at
[SETUP.md](SETUP.md) — `Setup.bat` is identical for everyone; it detects their
own Python install and asks for their own IDMS password, so no personal
credentials travel with the folder.

**Letting teammates *view* your live schedule** without running their own copy
of the real UiPath tasks (e.g. dashboard-only viewers) needs one extra manual
step — syncing only two small feed files one-way via Syncthing, and setting
`IsSource=0` on their end so their own (empty) Task Scheduler never overwrites
what you send them. Full walkthrough: **[SETUP.md, section 5](SETUP.md#5-sharing-the-live-schedule-with-your-team-view-only)**.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| AUTOMATION bar shows old tasks/history that never update, especially after copying this folder to another PC | `Setup.bat` was answered `N` (or an old copy carried over `IsSource=0`) on a PC that should be live. Re-run `Setup.bat` and answer **Y** ("Should this PC show its own live Task Scheduler?"). |
| Login/download suddenly fails everywhere | Click **UPDATE PASSWORD**, fix the password in Notepad, save, retry. If it keeps failing with the same password, it's very likely a temporary portal slowdown, not a bad password — the panel now retries automatically (up to ~2 minutes) before giving up, and says so honestly rather than blaming your credentials. |
| A report is skipped / in `0_FAILED_REPORTS.txt` | It failed across AS-IS **and** a couple of months of automatic wider retries — check the reason listed next to it, and see [Known limitations](#known-limitations) for reports with confirmed site-side issues. |
| Email says it used "previously-downloaded copies" | The fresh download couldn't complete before sending (portal hiccup) — the email still sent, using the most recent copy already on disk instead of failing outright. The banner in the email itself says exactly how old that data is. |
| Panel stuck saying "Sending…" / "Downloading…" forever | Every status poll now has a built-in give-up timer, so this shouldn't happen — if it does, check Task Manager for a stuck `powershell.exe`/`pythonw.exe`, close it, and retry. |
| Theme row / a button looks clipped at the window edge | Rainmeter's window doesn't always grow to match a widened background shape — if you're editing layout, keep new elements within the already-proven-safe bounds rather than assuming `DynamicWindowSize` will expand for you. |

---

## Known limitations

Four reports on `sos.ammotors.in` are confirmed broken on the site's own
backend and are hidden from the panel entirely rather than offered as buttons
that would always fail — **Price View List** and **Delivery Completed** return
server errors, **Event Request Report** returns a 404 on export, and
**Cancelled Booking** returns zero rows in every window tested. The download
logic for all four is still intact in `IdmsDownload.py`, ready to re-enable if
the site side ever gets fixed.

**Follow Ups - Upcoming** is the largest of the 32 reports and can occasionally
time out inside a very large batch (e.g. selecting everything at once); it
downloads reliably on its own or with a handful of others.

See [SETUP.md](SETUP.md#known-limitations-as-of-2026-07-24) for the full,
per-report detail.

---

## Project structure

```
AutomationReminder/
├── AutomationReminder.ini        Main skin: AUTOMATION bar + dashboard
├── AutomationTasks.lua           Its logic: countdown, gaps, history, theme ring
├── RefreshTasks.ps1              Queries Task Scheduler once a minute
├── SETUP.md                      Full install + day-to-day usage guide
├── @Resources/
│   ├── Settings.inc              Per-PC settings (Theme, IsSource) — never synced
│   └── Themes/                   One .inc per theme, incl. Custom.inc (yours to edit)
└── Downloads/                    Companion skin: PORTAL DATA bar + IDMS panel
    ├── Downloads.ini             The skin itself
    ├── IdmsEngine.lua            IDMS panel logic
    ├── DownloadEngine.lua        PORTAL DATA bar logic
    ├── IdmsDownload.py           Playwright automation — the actual downloads
    ├── AutoDownload.py           Lightweight Booking/Docket/Enquiry count refresh
    ├── IdmsScheduleTask.ps1 / IdmsEmailScheduleTask.ps1
    │                             Register a scheduled download / email
    ├── IdmsRunSchedule.ps1       What a scheduled download actually runs
    ├── IdmsSendEmail.ps1         Outlook COM automation for EMAIL
    ├── IdmsListSchedules.ps1 / IdmsListEmails.ps1 / IdmsDeleteTask.ps1
    │                             SCHEDULES / EMAILS / DELETE TASK buttons
    ├── IdmsEmailTemplate.html    Email HTML template
    ├── IDMS_User_Guide.html      Standalone user guide (open in any browser)
    └── Setup.ps1 / Setup.bat     One-time install script
```

---

## License

Personal / internal use.
