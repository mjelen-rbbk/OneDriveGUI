# OneDriveGUI — Architecture & OneDrive CLI Interaction

## Overview

OneDriveGUI is a PySide6-based desktop GUI that acts as a full-lifecycle manager for the
[Linux OneDrive client](https://github.com/abraunegg/onedrive) (`onedrive` binary).  
It does **not** call any Microsoft Graph API directly — all cloud operations are delegated to
the CLI binary, and the GUI communicates with it exclusively through:

- **subprocess pipes** (stdin/stdout/stderr)
- **configuration files** on the local filesystem
- **process signals** (SIGINT with SIGKILL fallback, or `systemctl --user stop` when a systemd service is detected)

```
┌─────────────────────────────────────┐
│            OneDriveGUI              │
│  (PySide6 Qt application)           │
│                                     │
│  ┌──────────────┐  ┌─────────────┐  │
│  │ WorkerThread │  │Maintenance  │  │
│  │ (monitor     │  │Worker       │  │
│  │  mode)       │  │(one-off ops)│  │
│  └──────┬───────┘  └──────┬──────┘  │
│         │ subprocess.Popen│         │
│         ▼                 ▼         │
│  ┌────────────────────────────────┐ │
│  │      onedrive binary           │ │
│  │  (abraunegg/onedrive)          │ │
│  └────────────────────────────────┘ │
│         │                           │
│  config files (~/.config/onedrive/) │
└─────────────────────────────────────┘
```

---

## Source Layout

| File | Responsibility |
|------|---------------|
| `src/OneDriveGUI.py` | Application entry point; initialises `QApplication`, loads config, starts `MainWindow`. |
| `src/main_window.py` | Main window; manages per-profile tabs, system tray, worker lifecycle, file transfer list. |
| `src/workers.py` | **Core integration layer.** Two `QThread` subclasses that own and parse the `onedrive` process. |
| `src/wizard.py` | Multi-step setup wizard: version check, create/import profile, SharePoint setup. |
| `src/profile_settings_window.py` | Profile configuration UI with input validation. |
| `src/gui_settings_window.py` | GUI-level settings (autostart, binary path, logging, etc.). |
| `src/global_config.py` | Reads/writes OneDrive config files and the GUI profile index. |
| `src/options.py` | Module-level singletons: `client_bin_path`, `client_version`, `global_config`, `gui_settings`. |
| `src/utils/utils.py` | Helpers: version detection, binary path resolution, file-size formatting. |
| `src/utils/autostart.py` | Manages the `.desktop` autostart entry. |
| `src/settings/gui_settings.py` | Reads/writes the GUI's own settings file. |
| `src/resources/default_config` | Template with every supported OneDrive config key and its default value. |

---

## Startup Sequence

```
OneDriveGUI.py
  └─ options.py
       ├─ config_client_bin_path()      → resolves 'onedrive' or custom path from GUI settings
       ├─ get_installed_client_version() → subprocess.check_output([bin, "--version"])
       └─ create_global_config()        → reads ~/.config/onedrive-gui/profiles
                                          + each profile's config file
  └─ MainWindow.__init__()
       └─ (if no profiles) show SetupWizard
       └─ (if auto_sync)   start WorkerThread per profile
```

The binary path is resolved in `src/utils/utils.py`:

```python
def config_client_bin_path() -> str:
    client_bin_path = gui_settings.get("client_bin_path")
    if client_bin_path == "":
        return "onedrive"          # rely on $PATH
    return client_bin_path         # absolute path from GUI settings
```

Version detection also runs at startup:

```python
client_version_check = subprocess.check_output(
    [client_bin_path, "--version"], stderr=subprocess.STDOUT
)
# Note: the leading `.` matches any character (not a literal dot) before the version number
installed_client_version = re.search(r".\s(v[0-9.]+)", str(client_version_check)).group(1)
```

The numeric version (e.g. `2` `5` `6` → `256`) is stored in `options.client_version` and used
throughout the app to hide/show config options that are only available in newer client releases.

---

## Configuration Files

### Profile Index

**Path:** `~/.config/onedrive-gui/profiles`  
**Format:** Standard INI (`ConfigParser`)

```ini
[bob@live.com]
config_file = /home/bob/.config/onedrive/accounts/bob@live.com/config
auto_sync = True
account_type = Personal
free_space = 14.3GiB

[work@company.com]
config_file = /home/bob/.config/onedrive/accounts/work@company.com/config
auto_sync = False
account_type = Business
free_space = 1.0TiB
```

### OneDrive Config File

**Path:** `~/.config/onedrive/accounts/<profile_name>/config` (default for new profiles)  
**Format:** key-value pairs — no section headers, no INI standard

Because `ConfigParser` requires section headers, `global_config.py::read_config()` prepends
`[onedrive]` before parsing, then strips it when writing back:

```python
new_config_string = "[onedrive]\n" + raw_file_content
config = ConfigParser()
config.read_string(new_config_string)
```

Multi-line `skip_file` / `skip_dir` options (each on its own line) are consolidated into a
single pipe-separated value so `ConfigParser` can handle them:

```
# on disk (multi-line, native OneDrive format)
skip_file = "~*"
skip_file = ".~*"
skip_file = "*.tmp"

# in memory (single line with pipe separator)
skip_file = "~*|.~*|*.tmp"
```

When saving, the `[onedrive]` section header line is removed so the file remains valid for the
CLI binary. A backup (`config_backup`) is written alongside the config on every save.

### In-memory Representation

`global_config` (built by `create_global_config()`) is a nested dict:

```python
{
    "bob@live.com": {
        "config_file": "/home/bob/.config/onedrive/accounts/bob@live.com/config",
        "auto_sync": "True",
        "account_type": "Personal",
        "free_space": "14.3GiB",
        "onedrive": {          # merged defaults + user overrides
            "sync_dir": '"~/OneDrive"',
            "skip_file": '"~*|.~*|*.tmp"',
            "monitor_interval": '"15"',
            ...
        }
    },
    ...
}
```

Default values come from `src/resources/default_config`; user values overlay them so the GUI
always has a complete config dict even for partially-written files.

---

## Worker Threads

All subprocess management lives in `src/workers.py`.  There are two worker classes, both
extending `QThread`.

### WorkerThread — Continuous Monitor

Runs `onedrive --monitor` for one profile.  Started when the user presses **Play** or when
`auto_sync = True` at startup.

**Command template:**

```python
self._command = (
    f"exec {client_bin_path} "
    f"--confdir='{config_dir}' "
    f"--monitor -v {extra_options}"
)
```

`exec` replaces the shell process with `onedrive`, ensuring `kill()` hits the right PID.
`-v` enables verbose output that the GUI relies on for parsing.

**Launching the process:**

```python
self.onedrive_process = subprocess.Popen(
    self._command + " --resync" if resync else self._command,
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,   # merge stderr into stdout
    shell=True,
    universal_newlines=True,
    encoding="utf-8",
    errors="replace",
)
```

**Read loop:**

```python
while self.onedrive_process.poll() is None:   # process still alive?
    self.read_stdout()                         # blocking readline()

# drain remaining output for ~1 second after exit
timeout = time.time() + 1
while time.time() < timeout:
    self.read_stderr()
```

**Stopping (subprocess mode):**

```python
def stop_worker(self):
    # SIGINT → wait up to 10 s → SIGKILL fallback (allows clean DB flush)
    self.onedrive_process.send_signal(signal.SIGINT)
    ...
    self.quit()
    self.wait()
    self.remove_worker.emit(self.profile_name)
```

**Stopping (systemd mode):**

```python
def stop_worker(self):
    subprocess.run(["systemctl", "--user", "stop", self.systemd_unit])
    self.onedrive_process.terminate()   # terminates journalctl
    self.quit()
    self.wait()
    self.remove_worker.emit(self.profile_name)
```

### MaintenanceWorker — One-off Operations

Used for operations that produce a finite amount of output and then exit:
authentication, SharePoint discovery, and shared-folder listing.

**Command template:**

```python
self._command = (
    f"exec {client_bin_path} "
    f"--confdir='{config_dir}' "
    f"{operation_flags}"
)
```

**Operations dispatched by flag pattern:**

| Flag pattern | Purpose |
|---|---|
| `--auth-response "<url>"` | Complete OAuth2 login flow with the callback URL the user pastes in |
| `--get-sharepoint-drive-id 'non-existent-library'` | Enumerate all SharePoint sites (trick: searching for a non-existent library causes the client to list all sites) |
| `--get-sharepoint-drive-id '<library>'` | Resolve the Drive ID for a specific SharePoint library |
| `--list-shared-items` | List shared Business folders |

---

## Output Parsing (WorkerThread.read_stdout)

`read_stdout()` calls `readline()` on the merged stdout/stderr pipe and applies a priority-
ordered chain of `if/elif` checks.  Each matched pattern emits one or more Qt signals to update
the GUI.

### Signal Map

| Qt Signal | Payload | Triggered by |
|-----------|---------|--------------|
| `update_profile_status` | `dict` (status_message, free_space, account_type, optional error_message) + profile name | Almost every parsed line |
| `update_progress_new` | `dict` (file_operation, file_path, progress %, transfer_complete, timestamp) + profile name | File transfer lines |
| `update_credentials` | profile name | OAuth required / refresh token expired |
| `trigger_resync` | profile name | `--resync is required` in output |
| `trigger_big_delete` | profile name | `To delete a large volume of data use` in output |
| `clear_warning` | profile name | Successful sync complete |
| `remove_worker` | profile name | Worker thread finished |

### Parsed Output Patterns

```
Calling Function: testNetwork()
  → status: "Testing network connection…"

authorise this application by … / --reauth and re-authorise this client
  → kill process, status: "OneDrive login is required.", emit update_credentials

Sync with Microsoft OneDrive is complete
Total number of local file(s) added or changed
No changes or items that can be applied were discovered
  → status: "OneDrive sync is complete.", emit clear_warning

Remaining Free Space: 15,389,384,704 bytes
  → parse bytes with regex, humanize, update profile_status["free_space"]

Account Type: Personal
  → parse last word, update profile_status["account_type"]

Initializing the OneDrive API
  → status: "Initializing the OneDrive API", reset failed-file counters

Starting a sync with Microsoft OneDrive
  → status: "Starting a sync…", emit clear_warning

Processing: 42 OneDrive items
  → status: "OneDrive is processing 42 items…"

--resync is required / before using --resync
  → emit trigger_resync (user must confirm before resync proceeds)

To delete a large volume of data use …
  → emit trigger_big_delete (user must confirm)

Uploading file ./path/file.zip ...   (or Downloading / Deleting / Moving)
  → emit update_progress_new with file_operation, file_path, progress=0

Uploading: path/file.zip ... 75%  |  ETA  00:00:04
  → emit update_progress_new with progress=75

ERROR: <message>          (optionally followed by)
Error Message: <detail>
  → emit update_profile_status with error_message for tooltip

Failed items to upload to/from Microsoft OneDrive: 3
  → start collecting failed-file lines, set failed_files_count

Failed to upload: path/file.zip
Failed to download: path/other.zip
  → append to failed_files list (capped at 25)

Unknown key in config file: <key>
  → emit config error status with the offending key name

Network Connection Issue
  → status: "Cannot connect to Microsoft OneDrive Service."

refresh_token / 'refresh_token'
  → status: "Logon details expired. Please re-authenticate.", emit update_credentials

/dlang/
  → status: "OneDrive client crashed. Please check logs."

command not found
  → status with install instructions URL
```

### File Transfer Progress Tracking

Transfer lines come in two forms:

1. **Start/completion line** — e.g. `Uploading file ./dir/file.zip ... done`  
   Parsed with `re.search(r".*/(.+)\s+\.+", stdout)` for the file name.

2. **Progress line** — e.g. `Uploading: dir/file.zip ... 75%  |  ETA   00:00:04`  
   Parsed with `re.search(r"(\w[Downloading|Uploading]+)\:\s+(.+?)[\.]*\s(\d{1,3})\%", stdout)`.
   Note: `[Downloading|Uploading]` in this pattern is a character class (matching single characters
   from the set), not an alternation.  The pattern works in practice because `\w` already captures
   the first character of "Downloading"/"Uploading" and the remaining characters happen to be in
   the set.  The semantically correct alternation would be `(?:Downloading|Uploading)`.

Both produce a `transfer_progress_new` dict that is emitted via `update_progress_new`.  The main
window maintains a list widget of `TaskList` items that display the file icon, name, progress
bar, and a relative-time stamp (e.g. "2m ago") when the transfer completes.

---

## Authentication Flow

OneDrive uses OAuth2.  The GUI surfaces this as follows:

1. `WorkerThread` detects `"authorise this application by"` in output.
2. It kills the running process and emits `update_credentials`.
3. `MainWindow` opens a login dialog that displays the authorisation URL.
4. The user opens the URL in a browser and pastes the callback URL back into the dialog.
5. `MainWindow` creates a `MaintenanceWorker` with `--auth-response "<pasted_url>"`.
6. `MaintenanceWorker.perform_login()` monitors stdout/stderr for `"error reason"`.
7. On success the login dialog closes; on failure the error string is shown.

Re-authentication (expired refresh token) follows the same flow, triggered by `" refresh_token "` appearing in output.

---

## SharePoint / Shared Folders

### Enumerating SharePoint Sites

```python
# Trick: pass a non-existent library name — the client lists all sites
options = "--get-sharepoint-drive-id 'non-existent-library'"
worker = MaintenanceWorker(profile, options)
```

`read_sharepoint_sites()` collects lines matching ` * <site_name>` and emits the list via
`update_sharepoint_site_list`.

### Resolving a Library Drive ID

```python
options = f"--get-sharepoint-drive-id '{library_name}'"
worker = MaintenanceWorker(profile, options)
```

`read_library_drive_ids()` collects `Library Name: …` / `drive_id: …` pairs into a dict and
emits it via `update_library_list`.

### Business Shared Folders

```python
options = "--list-shared-items"
worker = MaintenanceWorker(profile, options)
```

`read_shared_business_folders()` collects `Shared Folder: <name>` lines and emits them via
`update_business_folder_list`.

---

## Resync and Big-Delete Confirmations

The CLI outputs specific warnings when destructive operations are about to occur.  Rather than
proceeding silently, the GUI intercepts these and asks the user:

| CLI output | GUI response |
|---|---|
| `--resync is required` | `trigger_resync` signal → modal dialog. If approved, `WorkerThread.run(resync=True)` appends `--resync` to the command. |
| `To delete a large volume of data use --force` | `trigger_big_delete` signal → modal dialog. If approved, the next sync run includes `--force`. |

---

## Qt Signals & Threading Model

```
Main thread (Qt event loop)
  └─ MainWindow
       ├─ WorkerThread (one per profile, QThread)
       │    signals → main thread slots (update UI)
       └─ MaintenanceWorker (one-off, QThread)
            signals → wizard / settings dialog slots
```

All Qt signal emissions from worker threads are automatically queued across thread boundaries
by Qt's connection mechanism, keeping UI updates on the main thread.

---

## Data Flow Summary

```
User presses Play
     │
     ▼
WorkerThread.__init__
  builds command: "exec onedrive --confdir='…' --monitor -v"
     │
     ▼
subprocess.Popen(command, stdout=PIPE, stderr=STDOUT, shell=True)
     │
     ▼
read loop: readline() ──► pattern matching ──► Qt signals ──► UI updates
     │
  (file ops)                    (status)             (progress bars)
     │
  update_progress_new       update_profile_status   (error handling)
     │                           │
     ▼                           ▼
TaskList widget             Profile tab label
(file name + progress bar)  (status text + tooltip)
```


---

## Gap Analysis: Recommended Architecture vs. Actual Implementation

The creator of the `abraunegg/onedrive` CLI stated the following recommendation for GUI front-ends:

> **If a systemd service is enabled**, monitor the status of that, looking at the systemd logs
> to understand what is going on. To stop the service on system shutdown/suspend, issue a
> systemd service command to stop that service.
>
> **If a systemd service is not enabled**, start a `onedrive --monitor --verbose` instance and
> use the logging to understand what is going on. To stop the service on system
> shutdown/suspend, issue a Ctrl-C (SIGINT) to allow all shutdown handling, process cleanup,
> and clean DB shutdown.

### Gap 1 — No systemd awareness

OneDriveGUI has **zero systemd detection**. Regardless of whether the user has
`systemctl --user enable onedrive@<profile>.service`, the GUI always forks a new
subprocess with `onedrive --monitor -v`.

**Consequence:** The user must manually stop their systemd-managed service before the
GUI will work, because both the GUI subprocess and the systemd service attempt to own
the same SQLite state database. The `"application is already running"` check in
`read_stdout()` was explicitly commented out, so this conflict is silently swallowed.

### Gap 2 — SIGKILL instead of SIGINT

`WorkerThread.stop_worker()` uses `process.kill()` which sends **SIGKILL** — no graceful shutdown.
Similarly, the auth-detection branch in `read_stdout()` kills the process with SIGKILL.

**Consequence:** The `onedrive` binary has no opportunity to flush its SQLite state
database, write its sync state, or perform other cleanup. This is the likely root cause
of users needing `--resync` after closing the GUI.

The CLI author's recommendation is **SIGINT** (Ctrl-C equivalent), which triggers
the client's built-in shutdown handler.

---

## Implementation Plan (Changes Required)

### Change 1 — Systemd unit detection

Add `_detect_systemd_unit()` to `WorkerThread`. It checks the following candidate unit
names in priority order using `systemctl --user is-enabled`:

1. `onedrive@<profile_name>.service` — standard multi-account pattern
2. `onedrive@<config_dir_basename>.service` — alternative naming
3. `onedrive.service` — default single-account service

If any candidate exits with code 0 (`enabled`), that unit name is stored as
`self.systemd_unit` and `self.systemd_mode = True`. If systemd is unavailable
(FileNotFoundError, timeout), detection returns `None` and subprocess mode is used.

### Change 2 — Systemd monitoring mode

Add `_start_systemd_monitoring()` to `WorkerThread`:

1. Check if unit is active (`systemctl --user is-active`). If not, start it.
2. Launch `journalctl --user -u <unit> -f --output=cat -n 0` as the monitored process.
   `--output=cat` strips journal metadata so the existing `read_stdout()` parser works
   without modification. `-n 0` starts with no history, only future log lines.

The existing read loop `while self.onedrive_process.poll() is None: self.read_stdout()`
then works identically for both subprocess and systemd modes.

### Change 3 — SIGINT shutdown (subprocess mode)

Replace `process.kill()` (SIGKILL) with a graceful SIGINT then SIGKILL fallback in
`stop_worker()` and in `read_stdout()` auth detection.

### Change 4 — Systemd stop on shutdown/suspend

`stop_worker()` branches on `self.systemd_mode`:

- **Systemd mode:** `systemctl --user stop <unit>` then terminate the `journalctl` process
- **Subprocess mode:** SIGINT then SIGKILL (Change 3)

This satisfies the CLI author's requirement to issue a systemd service command to stop
the service on shutdown.

### Affected files

| File | Changes |
|---|---|
| `src/workers.py` | All four changes above |
| `docs/ARCHITECTURE.md` | This gap analysis section (documentation only) |

### Unchanged behaviour

- All UI interactions (start/stop buttons, tray icon, version checks)
- Output parsing (`read_stdout()` is identical for both modes)
- Authentication flow (MaintenanceWorker still handles `--auth-response`)
- SharePoint / shared-folder operations
- Profile configuration and settings dialogs
