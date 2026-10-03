# Getting started

This guide walks through installing and running Sheru on your own machine or
server. It covers the full stack: database, REST API, WebSocket gateway, web
console, and the endpoint agent.

## Architecture at a glance

```
┌─────────────┐   WSS /agent     ┌────────────────────┐
│  Endpoints  │◄────────────────►│  backend-gateway   │
│  (agents)   │                  │  (WebSocket)       │
└─────────────┘                  └─────────┬──────────┘
                                           │ Redis
web console ── HTTPS /api ──► backend-api ─┤
              WSS /console ─► gateway      │
                                           ▼
                            PostgreSQL + TimescaleDB
```

- **backend-api** — FastAPI REST API (`backend/`, `rmm_backend.main`).
- **backend-gateway** — WebSocket process for agent connections and the
  console's remote-shell / desktop relay.
- **Celery worker + beat** — background jobs (same `backend/` package).
- **Web console** — browser-based operator UI (`web/`).
- **Agent** — lightweight cross-platform Python agent (`agent/src/rmm_agent/`).
- **PostgreSQL + TimescaleDB** — persistence. The app connects as `rmm_app` so
  Row-Level Security applies; migrations run as the `rmm` superuser.
- **Redis** — agent connection registry, command dispatch, Celery broker.

## Prerequisites

- Docker Desktop (engine required for the local database stack)
- Node.js ≥ 22 and pnpm 11 (`npm i -g pnpm`)
- Python 3.12 managed by `uv` (`brew install uv`)
- On macOS add Docker Desktop's CLI to your PATH:

```sh
export PATH="/Applications/Docker.app/Contents/Resources/bin:$PATH"
```

## Quick start

```sh
cp .env.example .env        # local defaults are fine
make up                     # postgres + redis + API + gateway + worker + beat
make web-dev                # separate terminal: web console on :5173
```

`make up` waits until the API and gateway pass `/healthz`. The API container
applies Alembic migrations on start.

Once running:

- Web console → http://localhost:5173
- API liveness → http://localhost:8000/healthz
- API readiness (Postgres + Redis) → http://localhost:8000/readyz
- API docs (Swagger UI) → http://localhost:8000/docs

## First login

The platform provisions a bootstrap administrator on first start. Values come
from `.env` (`BOOTSTRAP_ADMIN_EMAIL` / `BOOTSTRAP_ADMIN_PASSWORD`); `.env.example`
ships:

| Field    | `.env.example` default                                      |
| -------- | ----------------------------------------------------------- |
| Email    | `admin@rmm.local`                                           |
| Password | `change-me-please-set-a-real-password`                      |

> **Security note:** change these before sharing the platform. They are read by
> `backend/src/rmm_backend/bootstrap.py`.

After signing in, create user accounts for your team under **Administration**.
A standing local enrollment token `dev-site-token` is seeded for agent
installs; issue real tokens from the console before going beyond local
development.

## Installing the agent

On each endpoint you manage:

```sh
cd agent
uv sync --all-groups
uv run rmm-agent enroll \
  --token dev-site-token \
  --backend-url http://localhost:8000/api/v1/agent/enroll
uv run rmm-agent run
```

The agent enrolls with the platform, stores its credential in the OS-native
secret store, and reconnects automatically. See
[Agent deployment](agent-deployment.md) for full details, including packaging
into a service for production.

## Developer / operational commands

| Command            | Purpose                                      |
| ------------------ | -------------------------------------------- |
| `make up`          | start stack (API migrates on start)          |
| `make down`        | stop the stack                               |
| `make api-dev`     | REST API with hot reload (`backend/`)        |
| `make gateway-dev` | WebSocket gateway with hot reload            |
| `make worker-dev`  | Celery worker on the host                    |
| `make web-dev`     | web console (Vite)                           |
| `make lint`        | ruff/mypy (backend) + eslint (web)           |
| `make test`        | pytest (backend) + vitest (web)             |
| `make migrate`     | apply backend Alembic migrations from host   |
| `make reset-db`    | wipe and restart the stack                   |

## Configuration

Key settings (`.env.example`):

| Variable            | Default                        | Purpose                          |
| ------------------- | ------------------------------ | -------------------------------- |
| `APP_NAME`          | `Sheru`                        | Product name surfaced in the API |
| `API_PREFIX`        | `/api/v1`                      | REST mount path                  |
| `JWT_SECRET`        | change-me-…                    | JWT signing secret (set strong)  |
| `BOOTSTRAP_ADMIN_*` | admin@rmm.local / (see `.env`) | First administrator account      |
| `SMTP_HOST`         | *(empty)*                      | Email notifications (alerts)     |
