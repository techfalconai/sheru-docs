# Admin guide

This guide covers administrative tasks: managing user accounts and roles, the
tenant/site model, and day-to-day platform administration.

## Access

Administrators sign in with a **superuser** account. The bootstrap account is
provisioned as a superuser. Non-superusers have access only to data within
their tenant (or no tenant data at all if unassigned).

The **Administration** navigation item appears in the sidebar only for
superusers.

## User management

Under **Administration** you can list, create, and manage every user account.

### Roles

| Role      | Capabilities                                                      |
| --------- | ----------------------------------------------------------------- |
| Admin     | Everything, including administration and cross-tenant visibility  |
| Member    | Platform features scoped to their tenant (or nothing if unassigned) |
| Client    | Customer end-user; sees only the **client portal** (dashboard, device status, incidents, support tickets) for their tenant |

Client accounts are non-superusers that must be assigned a tenant. When a
client signs in they are taken to the **[client portal](#client-portal)** — a
simplified, read-only view of their estate — instead of the staff console.
Clients cannot change ticket status/priority/assignment, manage alert rules,
acknowledge/resolve incidents, trigger patch scans, or use remote control.

To create a client account, press **Add user** and choose the **Client
(portal)** role, then select the tenant (customer) the account belongs to.

### Creating a user

1. Press **Add user**.
2. Enter the email, full name, and a **temporary password** (min. 8 chars).
3. Optionally mark **Administrator (full platform access)**.
4. Press **Create user**. Share the temporary password securely and have the
   user change it at first login (password reset, below).

### Managing accounts

Each row offers:

- **Make admin / Remove admin** — promote or demote a user.
- **Deactivate / Activate** — disable or re-enable a login. Deactivated users
  cannot sign in to the console or API, but their records (e.g. ticket
  history) are preserved.
- **Reset password** — set a new password immediately (forces a fresh login
  for that user if they were logged in — sessions are validated per request).

### Self-protection

You cannot deactivate, delete, or demote your **own** account, which makes it
impossible to accidentally lock yourself out. Use a second admin account if you
need to change your own role.

## Tenant / site model

Sheru organizes data by **tenant** (a customer) and **site** (a location within
the tenant):

- Devices, alert rules, incidents, patches, and tickets belong to a tenant.
- A site provides an **enrollment token** that agents use when joining.
- Superusers see everything; member users see only their tenant.
- Tickets can be assigned to any user; comments record the author.

> The bootstrap creates a `Default Tenant` with a single `Main Site` whose
> token is `dev-site-token`. Replace this token with a private value before
> external deployment.

## Device lifecycle

- **Enrollment**: an agent calls `/api/v1/agent/enroll` with the site token
  and appears under Devices. New devices get an asset tag (`SH-…`) and the
  default **Ready to Deploy** inventory status.
- **Online/offline**: heartbeat status is separate from inventory status.
- **Inventory status** (Snipe-IT-style labels): Ready to Deploy, Deployed,
  Pending, Out for Repair, Lost/Stolen, Broken, Archived. Only **deployable**
  devices can be checked out. Custom labels can be added per tenant.
- **Checkout / check-in**: assign a device to a console user or a free-text
  person, with an optional expected return date. History is kept on the
  device. **Lifecycle** in the sidebar shows fleet counts, overdue returns,
  and warranty windows.
- **Maintenance**: log repair/upgrade work against a device.
- **Decommission**: superusers/tenant admins hide the device from the default
  fleet view, mark it Archived, and tell the agent to uninstall. History is
  retained. Recommission restores Ready to Deploy.

## Audit log

Superusers can view the **Audit log** (sidebar → Audit log) to review
security-relevant and administrative actions: logins, failed logins, and user
management (create/update/deactivate), each with actor, action, resource, and
source IP. Login attempts are rate-limited (5 per 5 minutes per account by
default) to slow brute-force attacks.

## Platform settings

Most platform settings are environment variables in the API (`backend/src/rmm_backend/config.py`:

| Setting                     | Default                     | Purpose                                            |
| --------------------------- | --------------------------- | -------------------------------------------------- |
| `APP_NAME`                  | `Sheru`                     | Name surfaced in API responses                     |
| `JWT_SECRET`                | change-me-…                 | **Must be strong in production**                   |
| `JWT_EXPIRE_MINUTES`        | 480                         | Session length                                     |
| `BOOTSTRAP_ADMIN_EMAIL/PASSWORD` | admin@rmm.local / change-me | First administrator                            |
| `AGENT_OFFLINE_AFTER_SECONDS` | 90                        | Offline threshold                                 |
| `ALERT_EVAL_SECONDS`        | 30                          | Alert evaluator cadence                            |
| `LOGIN_RATE_LIMIT_ENABLED`  | true                        | Enable login rate limiting                         |
| `LOGIN_RATE_LIMIT_MAX_ATTEMPTS` | 5                      | Max login attempts per window                      |
| `LOGIN_RATE_LIMIT_WINDOW_SECONDS` | 300                  | Rate-limit window (seconds)                        |
| `SMTP_*`                    | *(empty)*                   | Email notifications                                |
| `TWILIO_ACCOUNT_SID`        | *(empty)*                   | Twilio account SID for SMS notifications           |
| `TWILIO_AUTH_TOKEN`         | *(empty)*                   | Twilio auth token                                  |
| `TWILIO_FROM_NUMBER`        | *(empty)*                   | Twilio sender phone number (E.164)                 |

## API access

- Interactive API reference: `http://<host>:8000/docs` (Swagger UI).
- All endpoints other than health/enroll/login require a Bearer token from
  `/api/v1/auth/login`.
- Agent endpoints (`/api/v1/agent/*`) use agent tokens and are isolated from
  console authentication.

## Client portal

The **client portal** is a separate, branded interface for customer end-users
(role `client`). It shares the same API but exposes only a tenant-scoped,
read-only view plus self-service support:

- **Dashboard** (`/portal`) — headline counts: total/online/offline devices,
  active incidents, and open tickets.
- **Devices** (`/portal/devices`) — device status, platform, IP, and last seen;
  read-only detail shows hardware specs and metric charts. No remote control,
  patches, or other management actions.
- **Alerts** (`/portal/alerts`) — incidents affecting the tenant's devices,
  read-only.
- **Support** (`/portal/tickets`) — submit a support request and track it, and
  reply to existing requests. Clients cannot change status, priority, or
  assignment (that stays with staff).
- **Account** (`/portal/account`) — view profile and change your own password
  (`/api/v1/auth/change-password`, requires the current password).

Access is enforced on the API: staff-only mutations reject `client` accounts
with `403 Not permitted for client accounts` (ticket updates, alert-rule
changes, incident ack/resolve, patch scan triggers, and remote terminal).

## Operational notes

- Background jobs (status sweep, alert evaluation, SLA sweep) run in the Celery
  worker + beat in production (`RUN_BACKGROUND_TASKS=false`); locally they run
  in-process. The scheduled-automation scheduler always runs in-process.
- Logs are written to stdout; wire the API into your log collector when
  deploying externally.
- Redis is required for the gateway dispatch path, Celery, and readiness.
  Confirm `GET /readyz` reports both `database` and `redis` as `ok` (liveness
  is `GET /healthz` and does not check dependencies).
- Production AWS deployment is provisioned with Terraform (see `infra/`).