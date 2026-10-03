# Architecture

This document describes how Sheru is built so external parties can operate,
extend, or integrate with it.

The stack is `backend/` + `web/` + `agent/src/rmm_agent/` + `shared/`. An
earlier `api/` (FastAPI) + `worker/` (Celery) prototype existed alongside
`backend/` during the Phase 0 migration and has since been removed — see
DECISIONS.md.

## Tech stack

| Layer          | Technology                                          |
| -------------- | --------------------------------------------------- |
| REST API       | FastAPI (Python 3.12), SQLAlchemy 2 async, Alembic  |
| WS gateway     | FastAPI / uvicorn process, Redis pub/sub dispatch   |
| Workers        | Celery (same `rmm-backend` package)                 |
| Database       | PostgreSQL 17 + TimescaleDB                         |
| Cache/dispatch | Redis 7                                             |
| Frontend       | React 19, TypeScript, Vite, Recharts, xterm.js      |
| Agent          | Python asyncio (`websockets`), cross-platform       |
| Contract       | Pydantic schemas in `shared/rmm_shared/`            |

## Components

### REST API (`backend/` — `rmm_backend.main`)

FastAPI process on `:8000`. Routes are mounted at `API_PREFIX` (Compose
default `/api/v1`): auth, devices, alerts, patches, remote sessions, osquery,
scripts, software policies, backups, SIEM, SSO/SAML/OIDC/LDAP/SCIM, audit,
reports, users, tenants, enrollment, device lifecycle (asset status, checkout).

Probes (unprefixed):

- `GET /healthz` — process liveness
- `GET /readyz` — Postgres `SELECT 1` + Redis `PING`; `503` if either fails

Multi-tenancy is dual-layer: application query filters plus Postgres RLS via
`set_config('app.current_tenant_id', …)` when the app connects as `rmm_app`.
See `backend/src/rmm_backend/db/tenant_scope.py` and
`infra/docs/deployment.md`.

Ticketing, knowledge base, client portal, and SLA tooling existed only in
the removed legacy `api/` prototype and are not planned for `backend/`. The
console nav in `web/src/AppLayout.tsx` matches what the API actually serves.

### WebSocket gateway (`backend/` — `rmm_backend.gateway.ws_server`)

Separate process on `:8001`:

- `/agent/v1/connect` — agent long-lived connection
- `/console/v1/terminal/{session_id}` — browser remote-shell relay
- `/console/v1/desktop/{session_id}` — browser remote-desktop relay

Agents are registered in Redis; commands are routed with pub/sub
(`gateway/dispatcher.py`). Vite and nginx proxy `/console/` and `/agent/` to
this process, not to the REST API.

### Workers (`backend/` — `rmm_backend.workers.celery_app`)

Celery worker + beat share the backend image. Beat schedule lives in
`celery_app.py` (for example SIEM delivery).

### Web console (`web/`)

React SPA. Vite proxies `/api` → `:8000` and `/console` → `:8001`. Production
nginx (`web/nginx.conf`) does the same, and maps `/health` and `/healthz` to
the API liveness probe, `/readyz` to readiness. Terraform's ALB uses that
path split and serves the SPA from a web ECS task (`nginx.static.conf`,
no in-container proxy).

### Agent (`agent/src/rmm_agent/`)

Cross-platform Python agent (`rmm-agent` CLI). Enrolls via REST, then keeps a
WebSocket open to the gateway. Inventory, metrics, patches, remote sessions,
and request/response actions (exec, files, services, osquery, desktop, …) use
the shared envelope in `shared/rmm_shared/message_schema.py`. See
[Agent-to-server protocol](agent-protocol.md).

The older `agent/agent/` tree is the pre-migration package; do not run
`python -m agent.main`.

### Shared contract (`shared/`)

Pydantic message schemas and enums imported by backend and agent.

## Data model (current backend)

- `tenants` / `sites` — multi-tenant organization. Sites carry enrollment tokens.
- `users` — accounts with roles (`super_admin`, tenant staff, …).
- `devices` — inventory + `status` (online/pending/offline) + device credentials.
- Time-series and operational tables for metrics, patches, alerts, audit,
  scripts, backups, osquery, SIEM destinations, and software policies.

Service-desk tables (`tickets`, `ticket_comments`) existed only in the
removed legacy `api/` schema (`rmm` database); `backend/` uses `rmm_backend`
and never had them.

## Protocols

Agent and browser frames are JSON envelopes over WebSocket. The canonical
types live in `shared/rmm_shared/`. Auth: console REST uses Bearer JWT; agent
WebSocket uses the per-device JWT; the console relay uses a one-time terminal
ticket.

## Security

- Passwords: hashed in `backend/src/rmm_backend/auth/user_auth.py`.
- Device credentials use JWT access + refresh (mTLS deferred; see
  `DECISIONS.md`). Secrets live in OS-native stores on the agent.
- Every tenant-scoped resource is filtered in the API **and** by RLS.
- Superusers (`rmm` DB role, `BYPASSRLS`) must not be the runtime app role.
- Use HTTPS/WSS outside development. The API is intended to sit behind nginx
  (same origin); there is no CORS middleware on `backend/` today.

## Data retention & operations

- Metric retention is bounded by TimescaleDB policies (configured externally).
- Audit logs are hash-chained (`backend/src/rmm_backend/services/audit.py`).
- External uptime checks should hit `/healthz` (liveness) and `/readyz`
  (dependencies) on the API; `/healthz` on the gateway. See
  `infra/docs/runbook.md`.

## Development

See [Getting started](getting-started.md) for local setup and
[Admin guide](admin-guide.md) for operational settings.

Quality gates: `make lint` / `make test` run ruff + mypy + pytest against
`backend/` and eslint + vitest against `web/`. Backend CI
(`.github/workflows/backend-ci.yml`) applies Alembic against `rmm_backend`
with Postgres and Redis sidecars.
