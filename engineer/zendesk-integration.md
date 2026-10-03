# Engineer SOP — Zendesk Integration + MDM/TPRM/AI Operations

## Zendesk Portal Setup
1. Create Zendesk account at `techfalconai.zendesk.com` (or actual subdomain)
2. Configure SSO integration (`SSO & Directory` in Sheru console) for support staff
3. Set webhook endpoint (`/api/v1/siem-destinations`) to push `audit_logs` events to Zendesk
4. Create Zendesk groups: `MDM Enrollment`, `TPRM Contracts`, `AI Governance`, `Agent Health`, `Security`

## Operations Workflow
- **Agent unreachable** (`/devices/{id}` status `offline` + `last_heartbeat_at` > 5 min): Zendesk ticket auto-created via Celery `device_unreachable_check`
- **MDM non-compliant** (`mdm_service` sync returns `compliance_status: non_compliant`): Zendesk notification via webhook
- **TPRM contract expiry** (`contracts.renewal_date` within 30 days): Zendesk alert via Celery `tprm-contract-expiry-check`
- **AI governance violation** (`AiGovernancePolicy.mode == default_deny` + device fails): Zendesk notification via webhook
- **Audit chain break** (`audit_logs.verify` shows breaks): Engineer escalates to super admin (SSM parameter access required)

## Deployment Notes
- `tprm-service` (`:8002`) and `mdm-service` (`:8003`) run as separate ECS Fargate tasks
- `ai_gateway_service` (`:8004` — planned if ALB listener rules added for `/ai-gateway/*`)
- Backend API (`:8000`) serves all REST endpoints; WebSocket gateway (`:8001`) handles agent connections
- CI pipeline (`backend-ci.yml`) verifies TPRM + MDM + AI Gateway module imports before deploy
