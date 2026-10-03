# Customer User Guide — Sheru (TPRM + MDM + AI Governance)

## Quick Start
1. Log in to the Sheru console (`https://console.sheru.cloud` or `http://alb-dns/`)
2. View device inventory (`/devices`)
3. Check AI governance policies (`/ai-governance`)
4. Monitor MDM enrollment (`/mdm`)

## MDM Enrollment (Intune / Kandji / Jamf)
- **Intune**: Enroll via `mdm_service` sync (`POST /v1/mdm/sync` with `provider: intune`)
- **Kandji**: Enroll via Kandji REST API (`mdm_client.py`)
- **Jamf**: Enroll via Jamf Pro/Cloud (`mdm_client.py`)

## TPRM (Third Party Risk)
- Vendor registry (`/tprm`): onboard vendors, track contracts, run risk assessments
- Contract tracking: service contracts, DPAs (`contract_type: dpa`), subprocessor agreements
- Risk assessment: NIST 800-30 weighted scoring (Impact × Likelihood + modifiers)
- Monitoring: reassessment due, contract expiry alerts (`Celery beat`: hourly + 2-hour intervals)

## AI Governance
- Policies: allow / forbid / default-deny for AI plugins, MCP servers, browsers, CLI tools
- Inventory: discovered AI items per device (`/api/v1/devices/{id}/inventory-items`)
- DNS tracking: AI-related domain queries (`/api/v1/devices/{id}/dns-activity`)
- Prompt findings: PII/PHI classification (`pii`: ssn/email/phone; `phi`: mrn/dob/npi)
- Token usage: daily input/output tokens per AI tool

## Support Portal (Zendesk)
Link: `https://sheru.zendesk.com` (placeholder — update with actual subdomain)
Access: `admin@sheru.zendesk.com` or via SSO (`SSO & Directory` settings)
