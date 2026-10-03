# Agent deployment

The Sheru agent is a lightweight, always-on RMM agent that runs as a native OS
service on Windows, macOS, and Linux. It reports system health and inventory,
executes remote commands/scripts, applies patches, and supports remote control.

## Requirements

- Python 3.11+ (for source installs) — or use the packaged single-file binary
  (no Python required on the endpoint).
- Network access to the Sheru API (HTTPS/WSS) on the configured server URL.

## Environment variables

| Variable                     | Default                    | Purpose                                        |
| ---------------------------- | -------------------------- | ---------------------------------------------- |
| `AGENT_SERVER_URL`           | `http://localhost:8000`    | API origin (use `https://…` in production)     |
| `AGENT_SITE_TOKEN`           | `dev-site-token`           | Enrollment token for the site                  |
| `AGENT_STATE_FILE`           | `<state-dir>/device.json`  | Non-secret device identity (no token)          |
| `AGENT_STATE_DIR`            | `/var/lib/rmm-agent` (POSIX) / `C:\ProgramData\rmm-agent` (Windows) | State + queue directory |
| `AGENT_QUEUE_PATH`           | `<state-dir>/queue.db`     | Durable offline message queue                  |
| `AGENT_HEARTBEAT_INTERVAL`   | `60`                       | Seconds between heartbeats                     |
| `AGENT_METRICS_INTERVAL`     | `300`                      | Seconds between metric samples                 |
| `AGENT_INVENTORY_INTERVAL`   | `900`                      | Seconds between full inventory scans           |
| `AGENT_LOG_DIR`              | `/var/log/rmm-agent`       | Log directory (POSIX)                          |

## Enrollment

On first run the agent posts its system inventory to `/api/v1/agent/enroll`
using the site token and receives a unique device ID and a long-lived device
credential. The credential is stored in the OS-native secret store (Windows
DPAPI, macOS Keychain, Linux libsecret/encrypted-file fallback) — never in
plaintext. The site token is a shared per-site enrollment token for mass
deployment; each device receives its own credential and never reuses the site
token after enrollment.

## Running

### Interactive (dev/test)

```sh
cd agent
uv sync --all-groups
uv run rmm-agent enroll \
  --token dev-site-token \
  --backend-url http://localhost:8000/api/v1/agent/enroll
uv run rmm-agent run
```

Use `uv run rmm-agent --help` for the rest of the CLI. Do not run
`python -m agent.main` — that is the pre-migration package under `agent/agent/`.

### As a service

The agent ships with native service definitions:

- **Linux (systemd)** — unit `sheru-agent` (`Restart=on-failure`).
- **macOS (launchd)** — label `com.sheruagent.daemon` (RunAtLoad, KeepAlive).
- **Windows (Windows Service)** — service `SheruAgent`, running as LocalSystem
  (required for Windows Update, service control, and registry access).

## Packaging

Build the single-file binary, then the installer, on each operating system.
PyInstaller does not cross-compile.

```sh
cd agent
uv sync --all-groups
# Windows also needs the service extra: uv sync --all-groups --extra windows
uv run python build.py
uv run python installers/build.py
```

`dist/installers/` then contains:

| System | Installer | Silent enroll |
| --- | --- | --- |
| Windows | `SheruAgent-Setup.exe` (Inno Setup 6) | `SheruAgent-Setup.exe /VERYSILENT /SUPPRESSMSGBOXES /SERVER=https://console.example /TOKEN=...` |
| macOS | `SheruAgent.pkg` | Write `SHERU_SERVER` and `SHERU_TOKEN` to `/var/tmp/sheru-agent-install.env` (mode 600), then `sudo installer -pkg SheruAgent.pkg -target /` |
| Linux | `sheru-agent_<version>_<arch>.deb` and a `.tar.gz` with `install.sh` | `sudo SHERU_SERVER=https://console.example SHERU_TOKEN=... dpkg -i sheru-agent_*.deb` |

The server value is the console origin. Enrollment is `{origin}/api/v1/agent/enroll`
and the agent socket is `wss://{host}/agent/v1/connect`. The token is handed to
`rmm-agent enroll` and is not stored in `config.yaml`.

The Windows service is SheruAgent (LocalSystem). macOS uses launchd label com.sheruagent.daemon. Linux uses systemd unit sheru-agent. The tray starts at graphical login so consent prompts have a desktop.

Tradeoffs:

- **PyInstaller** (default): simpler, well-supported, larger binary, easier to
  reverse-engineer.
- **Nuitka**: better performance, harder to reverse-engineer, slower build.

For production, sign the binary (Authenticode on Windows, Apple notarization on
macOS) so OS security tooling does not flag it. These packages are unsigned.

## Self-update

The agent checks `/api/v1/agent/update?version=<current>` for a newer signed
version, downloads it, verifies the SHA-256 and Ed25519 signature against a
pinned public key, then performs an atomic swap (previous binary kept as
`.old`) and restarts the service. A failed update never leaves the endpoint
without a working agent.

## Security notes

- Device credentials are stored in the OS secret store, never plaintext.
- Use HTTPS/WSS in production; the agent does not fall back to plaintext.
- Remote actions (script execution, remote control, patch install, file
  transfer) are logged to the backend audit trail.
- The agent runs with least privilege where possible (see the systemd unit's
  hardening directives); Windows runs as LocalSystem because Windows Update,
  service control, and registry access require it.

## Protocol

See [Agent-to-server protocol](agent-protocol.md) for the full message schema.
