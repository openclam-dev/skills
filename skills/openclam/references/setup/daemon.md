---
name: setup-daemon
description: >
  Install and start the OpenClam daemon as a system service. The daemon
  hosts the HTTP API, runs cron jobs, processes events, executes workflows
  and functions, and watches workspace folders for hot-reload.
metadata:
  author: openclam
  version: 0.1.0
---

# Daemon: first start

The daemon is the long-running process that hosts the HTTP API (default
port 3400), ticks cron, processes events, runs workflows, serves the MCP
streamable-HTTP endpoint, and watches extension folders for hot-reload.
You only need it running when something is calling the API (MCP, webhooks,
scheduled actions) or when work needs to happen in the background — the
CLI itself talks directly to the database for read/write commands.

This page covers the one-time install and first start. For day-to-day
operations (logs, status, restart, enabling workspaces), see
[operate/daemon.md](../operate/daemon.md).

## Start it (this is also the install)

There is no separate `install` step. `clam daemon start` writes a
launchd plist (macOS) or a systemd user unit (Linux), then loads it.

```bash
clam daemon start
```

What happens:

- **macOS** — writes `~/Library/LaunchAgents/com.openclam.daemon.plist`
  and loads it with `launchctl`. The daemon auto-restarts on crash and
  starts at login.
- **Linux** — writes `~/.config/systemd/user/openclam-daemon.service`,
  runs `systemctl --user daemon-reload`, enables and starts it.
- **Unsupported platforms** — falls back to a detached spawn. No
  auto-restart.

On success you'll see:

```
Daemon started (PID 12345, managed by launchd).
  HTTP: http://localhost:3400
  Socket: ~/.openclam/daemon.sock
  Auto-restarts on crash. Starts on login.
  Logs: ~/.openclam/daemon.log
```

### Custom port

```bash
clam daemon start --port 4000
# or
clam --json daemon start --input-json '{"port": 4000}'
```

The port is baked into the service config, so re-running `start` with a
different port unloads and reinstalls the service automatically.

### Foreground (debugging only)

```bash
clam daemon start --foreground
```

Runs in the current terminal without installing a service. Use this when
you want to see logs live or attach a debugger. Ctrl-C stops it. Don't
use this for normal operation — it does not survive your shell closing.

## Verify

```bash
clam daemon status
clam status
```

`daemon status` reports running / not running, PID, whether it's managed
by launchd or systemd, and which workspaces are enabled. `clam status`
includes daemon info alongside the active workspace summary.

## Enable workspaces

The daemon starts empty — you tell it which workspaces to serve:

```bash
clam daemon enable <slug>
clam daemon enable --all
```

Enabling persists to `~/.openclam/openclam.json` so it survives restarts.
If the daemon is already running, the change takes effect immediately via
the control socket; otherwise it picks up on next start.

## Notes and gotchas

- **Never use `clam daemon run`** — it doesn't exist. `start` always goes
  through the service manager (launchd / systemd). Use `--foreground` if
  you want to stay attached.
- **Unix socket path** — `~/.openclam/daemon.sock` is the control plane
  the CLI uses to talk to the daemon. Don't delete it while the daemon
  is running.
- **Logs live at `~/.openclam/daemon.log`** — stream with
  `clam daemon logs --follow`.
- **Env vars baked into the service** — `OPENCLAM_HOME`, `OPENCLAM_VAULT_KEY`,
  `OLLAMA_HOST` are captured at service-install time. Change them and
  re-run `clam daemon start` to refresh.
- **Vault and secrets rely on the daemon's env** — if you set `OPENCLAM_VAULT_KEY`
  in your shell and then `clam daemon start`, the daemon inherits it.
  Future shell sessions won't affect the running daemon.

## Next step

Deep daemon ops (`stop`, `restart`, `reload`, `logs -f`, event processing
config, workspace enable/disable in detail) live in
[../operate/daemon.md](../operate/daemon.md). Continue setup with
[ai-providers.md](./ai-providers.md).

See `../surfaces.md` to translate these operations to MCP, HTTP, or the
typed client.
