# MDM Module Plan (Intune + Kandji + Jamf)

## Architecture (follows TPRM pattern)
- `mdm_service/` — separate DB (`mdm`), federated auth (`mdm_service` scope), FastAPI `:8003`
- Agent `rmm_agent/collectors/mdm_inventory.py` — reports MDM status
- Backend `routes_mdm.py` — mounts MDM endpoints at `/api/v1/mdm`
- React `MDMPanel.tsx` — mounted in `AppLayout` (`NavKey` includes `mdm`)

## Capabilities
1. Enrollment sync (agent + MDM APIs)
2. Remote actions: lock, wipe, restart
3. Policy sync: Intune config profiles, Kandji Blueprints, Jamf policies
4. Inventory aggregation: device inventory from MDM APIs
5. Compliance reporting: enrollment + compliance status per device

## Integration Points
- `mdm_service` pulls from Intune (`graph.microsoft.com`), Kandji (`subdomain.kandji.io`), Jamf (`jamfcloud.com`)
- `agent/src/rmm_agent/collectors/mdm_inventory.py` reports local MDM agent state
- `devices` table linked via `mdm_device_id` reference (optional FK in future)

## Security
- Service-token auth (`mdm_service` scope)
- Fernet encryption for MDM policy JSON (reuse `secret_box.py`)
- Audit logs written to main DB `audit_logs` (hash chain preserved)

## Database Schema (`mdm` DB)
- `mdm_devices` — device sync records (provider, device_id, enrollment, compliance)
- `mdm_policies` — policy sync (provider, policy_type, scope_target)
- `mdm_actions` — remote actions (lock, wipe, restart) with status tracking
