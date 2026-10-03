# User guide

This guide covers day-to-day use of the Sheru console for operators: viewing
and managing devices, using remote control, responding to alerts, applying
patches, and working the service desk.

## Signing in

Open the web console and sign in with your user account. The bootstrap
administrator account is `admin@rmm.local` (password `change-me`) — replace
this with individual accounts as soon as practical.

## Devices

**Devices** lists every enrolled endpoint, with status, platform, agent
version, IP address, and last-seen time.

- Status pills show **Online / Pending / Offline** (based on recent agent
  heartbeats). Inventory status (Ready to Deploy, Out for Repair, …) is on
  the **Lifecycle** page and the device detail **Asset lifecycle** panel.
- Use the search box to filter by hostname, owner, asset tag, or serial.
- The stat cards give a quick fleet overview (total, online, offline).

### Device lifecycle (asset inventory)

Treat enrolled endpoints like Snipe-IT assets:

- Set manufacturer, model, serial, asset tag, purchase date/cost, and warranty.
- **Check out** to a person (console user or name/email) and **check in** when
  the machine returns to stock. Only deployable statuses can be checked out.
- Change inventory status (Pending, Out for Repair, Lost/Stolen, Archived).
- Log maintenance (repair, upgrade, PAT test).
- The **Lifecycle** nav item shows counts by status, overdue check-ins, and
  warranties expiring in 90 days.

### Device detail

Open a device to see:

- **Overview** — platform, OS version, CPU cores, memory and disk totals,
  agent version, first/last seen.
- **Live charts** — CPU, memory, and disk utilization trends reported by the
  agent.
- **Pending updates** — current patch inventory from the agent, grouped by
  category (e.g. `system`, `brew`, `apt`). Press **Scan now** to request a
  fresh scan.
- **Remote command & script** — run an arbitrary shell command or a typed
  script (shell/bash/python/powershell/batch) on the endpoint and see its exit
  code, stdout, and stderr, with a full run history.
- **File browser** — browse the endpoint filesystem, view file contents, and
  upload new files.
- **Services** — list services (systemd/launchd/Windows) and start, stop, or
  restart them.
- **Inventory** — hardware (CPU, memory, disks, network) and installed
  software, plus recent system events and one-click software installation.
- **Registry editor** — browse and edit the Windows registry (Windows only).
- **osquery** — query the endpoint as a SQL database (requires osquery
  installed). The **Fleet osquery** page runs a query across all online
  endpoints at once.
- **Automation** — schedules recurring commands using 5-field cron expressions.
- **Remote control** — VNC-over-SSH session: view the endpoint screen and send
  mouse/keyboard input. Requires the agent service manager (systemd/launchd/
  Windows Service) running and local SSH enabled. Sessions are time-limited
  and logged in the audit trail.
- **Remote desktop** — basic screenshot-based desktop view (alternative mode when
  full VNC is unavailable).
- **Remote terminal** — an interactive shell on the endpoint (see below).

## Remote terminal

The **Connect** button opens a live terminal session relayed through the API to
the endpoint. It requires the agent to be **online**.

- The terminal behaves like a normal shell; type commands and use the "Disconnect"
  button to end the session.
- Sessions are tied to your authenticated browser session and are authorized
  against the device's tenant.

## Alerts

**Alerts** combines monitoring rules with the incidents they raise.

### Alert rules

A rule watches one metric (e.g. `cpu.percent`) with an aggregation
(`last/avg/min/max`) over a time window, and fires when **condition** is true
(e.g. `value > 90`). Each rule has a severity (`info/warning/critical`) and can
notify by webhook and/or email.

- **New rule** opens the rule editor; **Edit** / **Delete** act on existing
  rules.
- Notifications: webhook posts to the configured URL; email sends to the
  configured recipients (requires SMTP settings); SMS sends to the configured
  phone numbers (requires Twilio settings).

### Incidents

When a rule condition matches, an incident is created; it retains
`open → ack → resolved` lifecycle.

- **Ack** acknowledges an open incident (stops re-triggering noise).
- **Resolve** closes it once the situation is handled.
- Use the filter to view all/open/acknowledged/resolved incidents.

## Patch management

Patch inventory is reported by the agent per platform source (`system`,
`brew`, `apt`, `dnf`, etc.). On the device page:

- **Pending updates** shows what needs updating, with versions.
- **Scan now** requests an up-to-date scan from the agent.

Patch *application* is intentionally manual/out-of-scope for now so an
operator decides when to apply changes.

## Service desk

**Tickets** is an integrated help desk for the fleet.

### Creating tickets

Click **New ticket**, give it a title, optional description, priority
(`low/medium/high/critical`), and optionally link it to a device (from the
dropdown). Tickets can also be tied to alerts and agent reports (`source`
field: `manual`, `rule`, or `agent`).

### Working a ticket

Open a ticket to:

- Change **status** (`open → in_progress → resolved/closed`); resolving stamps
  the resolution time.
- Change **priority**.
- **Assign** it to a team member (unassign by choosing "Unassigned").
- Add **comments** in the Activity timeline for a full audit trail.

The ticket list filters open vs. all tickets, and shows the title, linked
device, priority/status badges, and current assignee.

## Automation

**Automation** schedules recurring commands on endpoints using a 5-field cron
expression (`minute hour day-of-month month day-of-week`).

- **Create task** — pick a device, give it a name, a command, and a schedule
  (e.g. `0 3 * * *` for daily at 03:00).
- Each task shows its next run time and can be enabled/disabled or deleted.
- Expand a task to see its run history with exit codes and captured output.
- Runs are skipped when the agent is offline at the scheduled time.

## Automated checks

**Checks** run recurring health checks on endpoints:

- **Service** — verify a service is running (e.g. `sshd`).
- **Script** — run a script (shell/bash/python/powershell/batch) and treat a
  non-zero exit as failure.
- **Event log** — scan recent events for a matching pattern.
- **osquery** — run an SQL query; optionally assert a row-count range
  (`min_rows`/`max_rows`).

Each check has a config (JSON), an interval, and a last-run status. Use
**Run now** to trigger a check immediately.

## Reports

**Reports** gives fleet, ticket, and incident metrics:

- **Fleet** — total/online/offline/pending devices and a platform breakdown.
- **Tickets** — volume, resolution, SLA breaches, and priority breakdown over
  a selectable window (7/30/90 days).
- **Incidents** — volume, open/resolved counts, and severity breakdown.

## Audit log

**Audit log** (superusers only) records security-relevant and administrative
actions — logins, failed logins, and user management — with actor, action,
resource, and source IP.

## Client portal

Customer accounts (role `client`) sign in to a separate, simplified **client
portal** instead of the operator console — a fleet dashboard, read-only device
status, incident visibility, and self-service support tickets. See the
[admin guide](admin-guide.md#client-portal) for the full feature list.

## Notifications

Notification destinations are configured on alert rules:

- **Webhook** — the API POSTs an incident JSON payload to the configured URL.
- **Email** — needs SMTP settings in the API environment
  (`SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`, `SMTP_FROM`).
- **SMS** — needs Twilio settings (`TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`,
  `TWILIO_FROM_NUMBER`).