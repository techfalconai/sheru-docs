# Third Party Risk Management (TPRM) Module — Revised Plan

## 1. Executive Summary

Sheru (`https://github.com/techfalconai/sheru`) is a Python-based RMM platform with Device Management, Device LifeCycle, and AI Governance modules. This document defines a **separate-database, federated-service** TPRM module addressing vendor onboarding, risk assessment, continuous monitoring, and contract tracking — with explicit integration points to existing modules and testable compliance mapping.

**Critical assessment of original goal**: The initial request ("add TPRM, separate DB, all compliance, integrate with existing tools") contained 8 structural shortcomings: architectural misalignment (separate DB without federation model), scope overload (monolithic milestone), compliance gap (no framework-to-feature mapping), integration vagueness, security omission, missing risk model, missing agent/console contract, and unaddressed retention policy. This rewritten plan addresses each.

---

## 2. Shortcomings Analysis (Addressed)

| # | Shortcoming | Impact | Resolution in this plan |
|---|-------------|--------|-------------------------|
| 1 | Separate DB breaks multi-tenant RLS and audit hash chain | Data isolation vs. security tradeoff unclear | Separate PostgreSQL instance; audit logs preserved in main DB (`audit_logs`) via API federation; internal JWT auth for service-to-service |
| 2 | "All in 1" scope overload | Unshippable milestone risk | 3-phase delivery: Registry/Contracts → Risk Engine → Monitoring/Lifecycle |
| 3 | "All compliance frameworks" untestable | Requirements cannot be validated | Framework → feature → test criteria matrix (ISO 27001 A.15.1/2, SOC 2 CC9.2, NIST 800-30, GDPR Art. 28) |
| 4 | Integration undefined (Device/AI/Lifecycle) | Module operates in isolation | Vendor registry feeds AI Governance inventory (`vendor_id` FK); lifecycle events for vendor termination; vendors are not devices |
| 5 | Security model omitted | Separate DB has no auth/encryption/backup policy | Fernet encryption (same as `SsoProvider.client_secret`); service tokens; retention sync; object storage archive |
| 6 | No risk scoring algorithm | "Risk assessment" is unmeasurable | NIST 800-30 qualitative with weighted numeric model (Impact × Likelihood + vendor modifiers) |
| 7 | No console/UI contract | Module invisible to users | `TPRMPanel.tsx` in AppLayout; REST routes at `/api/v1/tprm`; Pydantic schemas in `shared/` |
| 8 | No data retention policy | Contract data lives forever | Inherits `tenant.retention_days`; contracts archive to object storage after retention period |

---

## 3. Revised Architecture

### 3.1 Component Placement

```
Sheru Stack (existing)
├── backend/ (FastAPI :8000, SQLAlchemy, Alembic)
│   ├── db/models.py (main DB)
│   ├── services/compliance.py
│   ├── api/routes_*.py
│   └── workers/
├── web/ (React SPA)
│   ├── AppLayout.tsx
│   └── AIGovernancePanel.tsx
├── agent/ (Python asyncio)
└── shared/ (Pydantic schemas)

TPRM Module (new)
├── tprm_service/ (FastAPI :8002 — like gateway :8001)
│   ├── main.py
│   ├── routes_tprm.py
│   ├── services/
│   │   ├── vendors.py
│   │   ├── contracts.py
│   │   ├── assessments.py
│   │   └── monitoring.py
│   ├── db/ (separate PostgreSQL instance)
│   │   ├── models_tprm.py
│   │   └── session_tprm.py
│   └── auth_service_token.py
├── docs/ (this document)
└── web/src/TPRMPanel.tsx (new React panel)
```

### 3.2 Database Strategy: Separate Instance, Federated Auth

- **Justification**: Vendor contracts, DPAs, and assessment notes contain sensitive legal data. Physical isolation supports different encryption keys, backup schedules, and jurisdiction-specific retention.
- **Tradeoff**: Integration is API-level federation (not SQL JOIN across instances). The TPRM service exposes endpoints that the main backend queries using internal service tokens.
- **Auth model**: TPRM service accepts service tokens signed by the main backend auth module (`rmm_backend.auth`). These are not public JWTs; they are scoped to the `tprm_service` role and carry `tenant_id` claims.
- **Audit preservation**: TPRM writes audit events to the main DB `audit_logs` table to preserve the hash chain (`prev_hash` → `entry_hash`). It does not maintain a separate audit chain.

### 3.3 Multi-Tenancy

TPRM inherits tenant isolation from the main backend through the service token. Every query is scoped by `tenant_id` (same pattern as `scoped_query` in `db/tenant_scope.py`). The separate DB uses application-level filtering; RLS is optional but recommended for defense in depth.

---

## 4. Data Model

### 4.1 Schema: `tprm`

New PostgreSQL instance; Alembic migrations live in `tprm_service/alembic/`.

```python
# tprm_service/db/models_tprm.py (SQLAlchemy 2.0 async)

class Vendor(Base):
    __tablename__ = "vendors"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("tprm.tenants_ref.id"), index=True)
    # Note: tenant_ref is a sync mirror of tenant IDs (not full tenant table)
    name: Mapped[str] = mapped_column(String(255), index=True)
    vendor_type: Mapped[str] = mapped_column(String(32))  # software | cloud | consulting | ai_tool | infrastructure
    status: Mapped[str] = mapped_column(String(16), default="active", index=True)  # active | inactive | decommissioned
    jurisdiction: Mapped[str | None] = mapped_column(String(64))  # EU / US / CA / etc.
    data_access_level: Mapped[str] = mapped_column(String(16), default="read")  # read | modify | admin
    critical_path: Mapped[bool] = mapped_column(Boolean, default=False)
    external_reference_id: Mapped[str | None] = mapped_column(String(255))  # link to AI Governance inventory item
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

class Contract(Base):
    __tablename__ = "contracts"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    vendor_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("vendors.id", ondelete="CASCADE"), index=True)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), index=True)
    contract_type: Mapped[str] = mapped_column(String(32))  # service | dpa | subprocessor | license
    start_date: Mapped[date] = mapped_column(Date)
    end_date: Mapped[date] = mapped_column(Date)
    renewal_date: Mapped[date | None] = mapped_column(Date)
    status: Mapped[str] = mapped_column(String(16), default="active", index=True)  # active | expired | terminated | pending_renewal
    file_path: Mapped[str | None] = mapped_column(String(500))  # encrypted file reference (object storage or DB)
    retention_policy: Mapped[int] = mapped_column(Integer, default=365)  # days; inherits tenant.retention_days
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

class Assessment(Base):
    __tablename__ = "assessments"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    vendor_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("vendors.id", ondelete="CASCADE"), index=True)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), index=True)
    assessor_user_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("users.id"), index=True)
    # Note: assessor_user_id references main DB users.id; assessment records include user reference by UUID only (not FK constraint across DBs)
    assessment_date: Mapped[date] = mapped_column(Date, server_default=func.now())
    impact_confidentiality: Mapped[int] = mapped_column(Integer)  # 1-10
    impact_integrity: Mapped[int] = mapped_column(Integer)
    impact_availability: Mapped[int] = mapped_column(Integer)
    likelihood_score: Mapped[int] = mapped_column(Integer)  # 1-10
    vendor_modifiers_json: Mapped[dict | None] = mapped_column(JSONB)  # {"data_access_level": 2, "critical_path": 3, "jurisdiction_eu_bonus": 2}
    weighted_score: Mapped[float] = mapped_column(Float)
    risk_level: Mapped[str] = mapped_column(String(16))  # Low | Medium | High | Critical
    framework_tags_json: Mapped[list] = mapped_column(JSONB, default=list)  # ["ISO_27001", "SOC_2", "NIST_800_30", "GDPR"]
    notes: Mapped[str | None] = mapped_column(Text)
    status: Mapped[str] = mapped_column(String(16), default="draft")  # draft | final | reviewed | superseded
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

class MonitoringEvent(Base):
    __tablename__ = "monitoring_events"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    vendor_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("vendors.id", ondelete="CASCADE"), index=True)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), index=True)
    event_type: Mapped[str] = mapped_column(String(32), index=True)  # reassessment_due | contract_expiry | risk_change | incident | lifecycle_link
    triggered_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    resolved_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    severity: Mapped[str] = mapped_column(String(16), default="info")  # info | warning | critical
    description: Mapped[str | None] = mapped_column(Text)
    linked_device_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), index=True)  # optional link to device (for AI vendor links)
```

### 4.2 Integration Data Model Changes (Existing DB)

```sql
-- Migration to main DB (rmm_backend): add vendor reference to AI Governance inventory
ALTER TABLE inventory_items ADD COLUMN vendor_id UUID REFERENCES vendors(id) NULL;

-- Note: vendors.id is a UUID reference only; no FK constraint across DB instances.
-- Integration is handled at the API/service layer.
```

---

## 5. API Specification

### 5.1 Service Endpoints (`tprm_service/main.py`, port `:8002`)

Base path: `/` (service-level); main backend proxies `/api/v1/tprm` → `:8002` (optional; can also expose directly if nginx routes added).

For simplicity, routes are mounted directly in `tprm_service/routes_tprm.py`:

```python
# tprm_service/routes_tprm.py
from fastapi import APIRouter, Depends
from pydantic import BaseModel

router = APIRouter()

# --- Vendors ---
@router.get("/vendors")
async def list_vendors(tenant_id: uuid.UUID = Depends(get_tenant_from_service_token)) -> list[VendorSchema]: ...

@router.post("/vendors")
async def create_vendor(payload: VendorCreate, tenant_id: uuid.UUID = Depends(...)) -> VendorSchema: ...

@router.get("/vendors/{vendor_id}")
async def get_vendor(vendor_id: uuid.UUID, tenant_id: uuid.UUID = Depends(...)) -> VendorSchema: ...

@router.put("/vendors/{vendor_id}")
async def update_vendor(vendor_id: uuid.UUID, payload: VendorUpdate, ...): ...

@router.post("/vendors/{vendor_id}/decommission")
async def decommission_vendor(vendor_id: uuid.UUID, ...): ...  # triggers lifecycle event in main DB

# --- Contracts ---
@router.get("/vendors/{vendor_id}/contracts")
async def list_contracts(vendor_id: uuid.UUID, tenant_id: uuid.UUID = Depends(...)) -> list[ContractSchema]: ...

@router.post("/vendors/{vendor_id}/contracts")
async def create_contract(vendor_id: uuid.UUID, payload: ContractCreate, ...): ...

# --- Assessments ---
@router.post("/vendors/{vendor_id}/assessments")
async def create_assessment(vendor_id: uuid.UUID, payload: AssessmentCreate, ...): ...

@router.get("/assessments")
async def list_assessments(tenant_id: uuid.UUID = Depends(...)) -> list[AssessmentSchema]: ...

@router.get("/assessments/{assessment_id}")
async def get_assessment(assessment_id: uuid.UUID, ...): ...

# --- Monitoring ---
@router.get("/monitoring")
async def list_monitoring_events(tenant_id: uuid.UUID = Depends(...), event_type: str | None = None) -> list[MonitoringEventSchema]: ...

@router.post("/monitoring/events/{event_id}/resolve")
async def resolve_event(event_id: uuid.UUID, ...): ...
```

### 5.2 Pydantic Schemas (`shared/rmm_shared/tprm_schemas.py` or `tprm_service/schemas.py`)

```python
from pydantic import BaseModel, Field
from datetime import date, datetime
from typing import Optional, Literal

class VendorSchema(BaseModel):
    id: str
    tenant_id: str
    name: str
    vendor_type: Literal["software", "cloud", "consulting", "ai_tool", "infrastructure"]
    status: Literal["active", "inactive", "decommissioned"]
    jurisdiction: Optional[str]
    data_access_level: Literal["read", "modify", "admin"]
    critical_path: bool
    external_reference_id: Optional[str]
    created_at: datetime
    updated_at: datetime

class AssessmentCreate(BaseModel):
    impact_confidentiality: int = Field(..., ge=1, le=10)
    impact_integrity: int = Field(..., ge=1, le=10)
    impact_availability: int = Field(..., ge=1, le=10)
    likelihood_score: int = Field(..., ge=1, le=10)
    vendor_modifiers_json: Optional[dict] = None
    framework_tags_json: list[str] = ["NIST_800_30"]
    notes: Optional[str] = None
```

---

## 6. Compliance Framework Mapping (Testable)

| Framework | Requirement | Module Feature (Phase) | Test Criteria |
|-----------|-------------|------------------------|---------------|
| ISO 27001 A.15.1.1 | Supplier information security policy | `vendors` registry + `vendor_type` classification (P1) | Every vendor has `status`, `vendor_type`, `jurisdiction` |
| ISO 27001 A.15.2.1 | Monitoring and review of supplier services | `monitoring_events` reassessment schedule + contract expiry triggers (P3) | `reassessment_due` events created 30 days before `renewal_date` |
| SOC 2 CC9.2 | Vendor risk assessment and monitoring | Risk scoring (`weighted_score`, `risk_level`) + continuous monitoring (P2, P3) | `Assessment` table has non-null `weighted_score`; `MonitoringEvent` has `resolved_at` |
| NIST 800-30 | Risk assessment (Identify → Assess → Respond) | `impact_*`, `likelihood_score`, `vendor_modifiers_json`, `weighted_score` (P2) | Score computed by service; mapped to Low/Medium/High/Critical |
| GDPR Art. 28 | Data Processing Agreement tracking | `contracts` with `contract_type="dpa"`, `retention_policy`, `start_date`, `end_date` (P1) | Every EU vendor (`jurisdiction="EU"`) has at least one active DPA contract |

---

## 7. Implementation Roadmap

### Phase 1 — Vendor Registry & Contracts (Month 1)
- [ ] Create `tprm_service/` package structure (`main.py`, `routes_tprm.py`, `db/session_tprm.py`)
- [ ] Define `models_tprm.py` (Vendor, Contract) with Alembic migration (`tprm_service/alembic/versions/`)
- [ ] Implement `routes_tprm.py` vendors and contracts endpoints
- [ ] Implement `services/contracts.py` with Fernet encryption for attachments (`auth/secret_box.py` reused)
- [ ] Build `VendorSchema` and `ContractSchema` in shared schemas
- [ ] Add React panel skeleton (`web/src/TPRMPanel.tsx`) mounted in `AppLayout.tsx`
- [ ] Write `tests/unit/test_tprm_vendors.py` and `test_tprm_contracts.py`

**Deliverable milestone**: CRUD for vendors and contracts; DPA tracking functional; console panel renders vendor list.

### Phase 2 — Risk Assessment Engine (Month 2)
- [ ] Define `Assessment` and `AssessmentCreate` models
- [ ] Implement `services/assessments.py`: NIST 800-30 scoring algorithm
  - Impact = average(confidentiality, integrity, availability)
  - Likelihood = user input (1-10)
  - Modifiers: `data_access_level` (+0/1/2), `critical_path` (+3 if true), `jurisdiction` (+2 if EU for GDPR weight)
  - Weighted score = round((Impact × 0.6) + (Likelihood × 0.3) + (Modifiers × 0.1), 2)
  - Mapping: <4 Low, 4-6.9 Medium, 7-8.9 High, >=9 Critical
- [ ] Integration: `inventory_items.vendor_id` column added; `AIGovernancePanel` links vendor names to TPRM registry
- [ ] Compliance matrix validated: every framework mapped to feature + test criteria

**Deliverable milestone**: Risk assessments produce measurable scores; AI vendors link to registry; compliance mapping tested.

### Phase 3 — Continuous Monitoring & Lifecycle Integration (Month 3)
- [ ] Define `MonitoringEvent` model
- [ ] Implement `services/monitoring.py`: event triggers (contract expiry -30 days, reassessment schedule, risk score change threshold >1.0)
- [ ] Implement lifecycle integration: `decommission_vendor()` writes `device_lifecycle_events` with `action="vendor_terminated"` (main DB)
- [ ] Build alert integration: `MonitoringEvent.severity="critical"` creates `alerts` entry in main DB (optional; depends on `routes_alerts.py`)
- [ ] Object storage archive: contracts archived after retention period (not deleted); file references updated
- [ ] Full integration tests (`tests/integration/test_tprm_integration.py`)

**Deliverable milestone**: Continuous monitoring active; vendor termination linked to device lifecycle; retention archive functional.

---

## 8. Security & Risk Controls

### 8.1 Encryption
- Contract attachments (`file_path`): Fernet-encrypted at rest using `secret_box.py` (same scheme as `SsoProvider.client_secret`). Decryption only at download/reveal endpoint.
- Database at rest: separate DB instance uses Postgres native encryption (`pgcrypto` extension) or cloud-managed encryption (optional).

### 8.2 Authentication & Authorization
- Service-to-service: `tprm_service/auth_service_token.py` validates tokens signed by `rmm_backend.auth.api_tokens`. Scope: `tprm_service` only.
- Console users: access to `/api/v1/tprm` requires role `tenant_admin` or above (same RBAC as existing routes).
- Internal endpoints (`:8002`) not exposed to public; nginx routes `/api/v1/tprm` internally.

### 8.3 Audit & Integrity
- Audit events for TPRM actions (`vendor_created`, `assessment_finalized`, `contract_expired`, `vendor_decommissioned`) written to main DB `audit_logs` table.
- Hash chain preserved: TPRM service reads `prev_hash` from main DB for each audit entry it creates; writes `entry_hash` using same `verify_chain()` logic in `services/audit.py`.

### 8.4 Data Retention & Backup
- `retention_policy`: default 365 days; inherited from `tenant.retention_days` via sync job (Celery beat task `sync_tprm_retention` or API call at contract creation).
- Archive: contracts past retention moved to object storage (S3-compatible); DB row retains `file_path` pointing to archive URL; `status` changed to `archived`.
- Backup: separate DB instance included in same backup schedule as main DB (`backup_scheduler` worker extended, or separate cron).

### 8.5 Network
- TPRM service (`:8002`) runs in same container/network as main backend; no public port exposure unless nginx routes added.
- WebSocket gateway (`:8001`) unchanged; TPRM does not use WebSockets.

---

## 9. Integration Points

### 9.1 Device Management
- Vendors are **not** devices. No duplication of device inventory.
- Vendor type `"software"` or `"ai_tool"` can reference a device via optional `external_reference_id`, but primary integration is through AI Governance (see 9.3).

### 9.2 Device LifeCycle
- `decommission_vendor()` triggers:
  1. Update vendor status → `decommissioned`
  2. Update related contracts status → `terminated` (if not already)
  3. Create `MonitoringEvent` (`event_type="lifecycle_link"`, `description="Vendor decommissioned; linked contracts terminated"`)
  4. Write `audit_logs` entry (`action="vendor_decommissioned"`)
  5. **Optional**: Write `device_lifecycle_events` (`action="vendor_terminated"`, `device_id` = linked device if any) — requires new action value in `routes_lifecycle.py`.

### 9.3 AI Governance
- Integration mechanism: new optional column `inventory_items.vendor_id` (main DB).
- Workflow:
  1. AI vendor discovered (`vendor_type="ai_tool"`) → added to TPRM registry.
  2. Vendor assessment (`assessment.risk_level="Critical"`) → triggers governance review.
  3. Vendor marked `decommissioned` → `inventory_items.vendor_id` set to NULL (soft disconnect); governance policy remains but references decommissioned vendor.
  4. `AIGovernancePanel.tsx` displays vendor name as hyperlink to `/console/tprm/vendors/{vendor_id}`.

---

## 10. Open Questions / Next Decisions

1. **Service port exposure**: Should `tprm_service` run as separate ECS task (`:8002`) or as sub-process in backend image? Recommendation: same image, separate process (like gateway), to share container but isolate memory.
2. **Object storage**: Should contract attachments use existing storage backend (if any) or S3 directly? The current codebase uses inline Postgres (`FileTransfer.data_b64`, `VaultCredential.secret_encrypted`) for MVP. TPRM follows same pattern for MVP (encrypted DB), with object storage migration in Phase 3.
3. **Risk score weights**: Should modifiers be configurable per tenant? Recommendation: fixed weights for Phase 2; tenant-level customization in Phase 4 (post-roadmap).
4. **Reassessment frequency**: Default reassessment interval (e.g., 90 days for High/Critical vendors, 180 days for Low/Medium). Should this be configurable per vendor or fixed? Recommendation: fixed defaults with optional `reassessment_interval_days` column added in Phase 2.
5. **Integration with legacy `api/` database**: The legacy `api/` package (`rmm` DB) has `tickets` and `ticket_comments`. Should TPRM vendor incidents link to legacy tickets or new backend alerts? Recommendation: link to new `alerts` (Phase 3) rather than legacy tickets.

---

## Appendix A: File List (To Be Created)

In `/tmp/sheru/` (cloned repo):

```
docs/tprm-plan.md                 # This document
backend/src/rmm_backend/db/models_tprm_migration.py  # Alembic migration stub (main DB: inventory_items.vendor_id)
tprm_service/                     # New package (optional placement; can be backend sub-package)
├── __init__.py
├── main.py
├── routes_tprm.py
├── auth_service_token.py
├── db/
│   ├── __init__.py
│   ├── session_tprm.py
│   └── models_tprm.py
├── services/
│   ├── vendors.py
│   ├── contracts.py
│   ├── assessments.py
│   └── monitoring.py
└── schemas/
    └── tprm_schemas.py
web/src/TPRMPanel.tsx
shared/rmm_shared/tprm_schemas.py
```

---

## Appendix B: Judgment Calls (Why This Plan Differs from Original Request)

| Original Request | Judgment Revision | Rationale |
|------------------|-------------------|-----------|
| "Separate DB" (unspecified) | Separate instance + federated auth + audit in main DB | Preserves audit hash chain; avoids split-brain security model |
| "All in 1" scope | 3-phase delivery (1 month each) | Risk of unshippable milestone; phased delivery allows feedback |
| "Integration needs" (vague) | Defined FK model (`inventory_items.vendor_id`) + lifecycle event (`vendor_terminated`) | Integration is testable and traceable |
| "All compliance frameworks" | Mapped to feature/test criteria matrix | Untestable requirements become validated deliverables |
| "Module design doc, data model, API spec, roadmap" | All included + security controls + open questions | Comprehensive plan for a critical project |
| No security specification | Fernet encryption, service tokens, retention sync, archive | Separate DB requires its own security posture |
| No risk scoring model | NIST 800-30 quantitative with modifiers | "Risk assessment" requires measurement |
