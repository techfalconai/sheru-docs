# Engineer SOP — Sheru Operations

## Module Architecture
- `backend/src/rmm_backend/`: Active REST API (`FastAPI`) + WebSocket gateway (`:8001`)
- `agent/src/rmm_agent/`: Cross-platform Python agent (inventory, patches, remote, MDM, AI, backup)
- `mdm_service/`: Separate PostgreSQL instance (`mdm`) — Intune (`Graph REST`), Kandji (`subdomain REST`), Jamf (`cloud REST`)
- `tprm_service/`: Separate PostgreSQL instance (`tprm`) — vendor registry (`vendors`), contracts (`contracts`), assessments (`assessments`), monitoring (`monitoring_events`)
- `ai_gateway_service/`: Separate PostgreSQL instance (`ai_gateway`) — multi-provider routing (`openai_client`, `anthropic_client`), prompt versioning (`prompt_version_manager`), rate limits (`rate_limiter`), cost tracking (`cost_tracker`), observability (`observability`), audit trail (`audit_trail`), analytics (`analytics_dashboard`)

## Deployment (Cloud — AWS ECS)
- `make up` (local Docker Compose with `tprm-service` on `:8002`, `mdm-service` on `:8003`)
- `make up-prod` (`docker-compose.prod.yml` -> `ecs.tf` -> `alb.tf`)
- `infra/deploy.sh`: build ECR images (`backend`, `web`), push, `terraform apply`
- ALB rules: `/api/*` → API (`8000`), `/tprm/*` → TPRM (`8002`), `/mdm/*` → MDM (`8003`), `/agent/*` + `/console/*` → Gateway (`8001`)

## Database
- Main: `rmm_backend` (`postgres` service, TimescaleDB, RLS enabled via `rmm_app` role)
- TPRM: `tprm` (`tprm_service` connects with `DATABASE_URL_TPRM`)
- MDM: `mdm` (`mdm_service` connects with `DATABASE_URL_MDM`)
- AI Gateway: `ai_gateway` (planned — `DATABASE_URL_GATEWAY`)

## CI Pipeline
- `backend-ci.yml`: lint (`ruff`), type check (`mypy`), migrations (`alembic upgrade head`), pytest (tests/unit + integration), security (`bandit`, `semgrep`)
- `agent-ci.yml`: lint (`ruff`), mypy, pytest (`tests/unit`, `tests/integration`), build binary (`build.py`)
- `ci.yml`: web lint (`pnpm lint`), type check (`pnpm typecheck`), build (`pnpm build`)

## Monitoring & Alerting
- `alerts`: device-level metrics, compliance mismatches, MDM events
- `siem_destinations`: webhook/syslog push for audit logs
- `audit_logs`: hash-chained audit trail (`prev_hash` → `entry_hash`)
- `telemetry_metrics`: TimescaleDB hypertable (CPU, memory, disk, network samples)
