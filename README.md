# Dashboard Helper — VS Code Extension

A companion VS Code extension for [self-hosted developer dashboards](https://github.com/bensouille/dashboard-helper-vscode). It does two things:

1. **URI Handler** — opens remote SSH folders in a VS Code window via `vscode://devdashboard.dashboard-helper/open` links, focusing an existing window if already open
2. **Workspace Reporter** — automatically syncs your active VS Code sessions to the dashboard backend

---

## Features

### 1. URI Handler — open any remote folder from the dashboard

Handles `vscode://devdashboard.dashboard-helper/open` URIs (extension publisher: `devdashboard`).

**URL format:**

```
# Remote SSH workspace
vscode://devdashboard.dashboard-helper/open?remote=my-server&folder=/home/user/myproject

# Local workspace
vscode://devdashboard.dashboard-helper/open?folder=/absolute/path

# Remote host root (no specific folder)
vscode://devdashboard.dashboard-helper/open?remote=my-server
```

The `remote` parameter is an SSH config alias defined in `~/.ssh/config` on your **local** machine. No additional configuration is required for the URI handler to work.

**Smart window management:**

- If the target workspace is **already open** in a window, that window is **focused** (no new window opened, no popup).
- If not open yet, a **new window** is opened (`forceNewWindow: true`), avoiding the *"save workspace configuration?"* prompt that appears when VS Code tries to replace the current window.

The extension achieves this by:
1. Scanning `workspaceStorage` to resolve the exact VS Code SSH authority for the target host — supporting both the plain format (`ssh-remote+hostname`) and VS Code's newer hex-encoded JSON format (`ssh-remote+7b22...7d`).
2. Reading `storage.json` at runtime to check which windows are currently open.

### 2. Workspace Reporter — keep the dashboard in sync

When `dashboardHelper.backendUrl` and `dashboardHelper.agentToken` are set (typically in remote workspace settings), the extension:

- Reports the current workspace as **active** on startup
- Tracks workspace folder additions and removals within the same window
- Marks all workspaces **inactive** when the window closes
- Sends a heartbeat at a configurable interval (default: 60 s)

The `vscode://` URL embedded in each report includes `sshUser@hostname` if `dashboardHelper.sshUser` is set, so the dashboard generates links that preserve Copilot Chat history (VS Code ties chat history to the workspace URI, which includes the SSH user).

This is a lightweight alternative to running the full `agent.py` — no system metrics, just session tracking.

---

## Data collected (GDPR)

When the workspace reporter is active, the extension transmits the following data to the backend URL **you configured**:

- **Hostname** of the remote machine (`hostAlias` setting, or system hostname)
- **Folder paths** of open workspaces
- Active/inactive status and timestamp

No data is sent to any third party. You are the sole operator of the backend.

---

## Installation

### Prerequisites

- VS Code 1.74+
- A running instance of the [developer dashboard backend](https://github.com/bensouille/dashboard-helper-vscode) (for the reporter feature only)

### Build from source

```bash
git clone https://github.com/bensouille/dashboard-helper-vscode.git
cd dashboard-helper-vscode
npm install
npm run compile
npm run package          # → dashboard-helper-1.0.0.vsix
```

### Install the `.vsix`

**Via VS Code UI:** Extensions panel → `⋯` menu → *Install from VSIX…*

**Via CLI (local machine):**
```bash
code --install-extension dashboard-helper-1.0.0.vsix
```

**Via CLI (SSH remote host):**
```bash
code --install-extension dashboard-helper-1.0.0.vsix
# or, from the remote machine directly:
code-server --install-extension dashboard-helper-1.0.0.vsix
```

Install on **both** your local machine (for the URI handler) and every remote host (for session reporting).

---

## Configuration

All settings live under the `dashboardHelper` namespace.

| Setting | Default | Description |
|---|---|---|
| `dashboardHelper.backendUrl` | `""` | URL of your dashboard backend, e.g. `https://dashboard.example.com` |
| `dashboardHelper.agentToken` | `""` | Secret token matching `AGENT_TOKEN` in the backend `.env` |
| `dashboardHelper.hostAlias` | `""` | Identifier for this host (defaults to system hostname) |
| `dashboardHelper.sshUser` | `""` | SSH user used to connect to this host (e.g. `root`). Included in the generated `vscode://` URL as `user@host` so VS Code workspace identity and Copilot Chat history are preserved. |
| `dashboardHelper.reportInterval` | `60` | Heartbeat interval in seconds (min: 10) |

> The URI handler works **without any configuration**. The workspace reporter only activates when `backendUrl` and `agentToken` are both set.

### Setup on a remote host

Add to the remote workspace `.vscode/settings.json` or remote user settings:

```json
{
  "dashboardHelper.backendUrl": "https://dashboard.example.com",
  "dashboardHelper.agentToken": "your-secret-token",
  "dashboardHelper.hostAlias": "my-server",
  "dashboardHelper.sshUser": "steph"
}
```

`hostAlias` should match the SSH `Host` alias you use in `~/.ssh/config` so that the dashboard can generate correct `vscode://` links back to this host.
`sshUser` must match the user you use when connecting via Remote SSH (e.g. `User steph` in `~/.ssh/config`).

### Multi-host setup

Install the same `.vsix` on every remote host. Each host reports independently using its own `hostAlias`. The dashboard backend deduplicates sessions by `(hostname, repo)`.

```
Local machine      →  URI handler active  (opens remote windows from dashboard links)
Remote host A      →  Workspace reporter active
Remote host B      →  Workspace reporter active
...
```

---

## Backend API

The extension calls one endpoint:

```
POST /api/v1/hosts/session-sync
X-Agent-Token: <your-token>
Content-Type: application/json

{
  "hostname": "my-server",
  "repo": "/home/user/myproject",
  "vscode_url": "vscode://devdashboard.dashboard-helper/open?remote=steph%40my-server&folder=%2Fhome%2Fuser%2Fmyproject",
  "is_active": true
}
```

- `hostname` is `hostAlias` (or system hostname).
- `vscode_url` encodes `sshUser@hostname` in the `remote` parameter when `sshUser` is set. The backend will **not overwrite** an existing URL already stored for this session.
- Setting `is_active: false` marks the session as closed (sent automatically on window/extension deactivation).

---

## Debugging

All activity is logged to the **Dashboard Helper** output channel (View → Output → Dashboard Helper). Useful for diagnosing:

- Authority resolution (plain hostname vs hex-encoded JSON)
- Whether a window was found open in `storage.json`
- POST request results

---

## Development

```bash
npm run watch    # recompile on save
```

Press `F5` in VS Code to launch an Extension Development Host for live debugging.

---

## License

MIT

