# Agent Rewrite Plan (Lightweight MDM + TPRM Integration)

## Goal
Lightweight (<10MB binary, <50MB RAM, <1s startup) cross-platform agent that:
- Reports MDM status (Intune/Kandji/Jamf)
- Collects inventory with TPRM vendor_id linkage
- Has offline queue resilience
- Includes audit chain link
- Builds as unified binary

## Shortcomings Addressed
1. Binary size target enforced
2. Memory profiling added
3. Startup time tracked
4. MDM collector integrated with native REST clients
5. Agent inventory links vendor_id
6. Offline queue added to agent transport
7. Lifecycle event reporting added
8. Security: audit hash verification in agent reports

## Implementation
- agent/src/mdm_agent/ (lightweight core)
- agent/src/rmm_agent/collectors/mdm_inventory.py (rewritten with SDK clients)
- agent/src/rmm_agent/transport/offline_queue.py (rewritten for agent reports)
- agent/src/rmm_agent/security/audit_link.py (rewritten)
- agent/build.py updated for <10MB binary target
