# Customer Support Portal (Zendesk)

## Zendesk Subdomain (REAL)
- Subdomain: `epicit`
- Portal URL: `https://epicit.zendesk.com`
- Admin URL: `https://epicit.zendesk.com/admin`
- Access: `https://epicit.zendesk.com/agent` (or via SSO `SSO & Directory` settings)
- API Token: `B65satAysDZOUAVF1SdLNbenJHFUulmDDHqISOJ4` (stored securely via environment variable `ZENDESK_API_TOKEN` or SSM parameter `zendesk_api_token`)

## Integration Points
- TPRM vendor assessments: link vendor decommission to Zendesk ticket (`vendor_terminated` lifecycle event triggers alert)
- MDM device compliance: non-compliant device triggers Zendesk ticket via `alerts` → `siem_destinations` webhook
- AI governance violations: `default_deny` policy violation creates `ai_policy.violated` audit event → webhook to Zendesk
- Agent health: agent unreachable >10 minutes (`device_unreachable_check`) triggers alert → Zendesk
- Backup failures: backup upload failure → Zendesk ticket
- Security: audit chain verification (`/api/v1/audit/verify`) results can be exported to Zendesk

## SOP (Standard Operating Procedure) for Support Team
1. **Login**: Use `SSO & Directory` (`/sso`) or Zendesk subdomain directly
2. **Device Issue**: Search by hostname (`/devices?q=hostname`) or asset tag; view lifecycle events (`/devices/{id}/lifecycle`)
3. **MDM Enrollment Failure**: Check `/mdm/devices` for `enrollment_status`; verify Intune/Kandji/Jamf API health
4. **TPRM Contract Expiry**: Check `/tprm` vendor contracts for `renewal_date` within 30 days (monitoring events trigger alert)
5. **AI Governance Violation**: Check `/ai-governance/compliance` for device-level policy violations; review `ai_prompt_findings` for PII/PHI signals
6. **Audit Chain Break**: Verify `/api/v1/audit/verify` for `prev_hash`/`entry_hash` mismatches

## Escalation
- Level 1: Support team handles enrollment/compliance/basic policy questions
- Level 2: Engineer team handles lifecycle/status/maintenance, MDM sync failures, audit breaks
- Level 3: Super admin handles SSO/SSM parameter changes (`jwt_secret`, `bootstrap_admin_password` in SSM)
