# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Permitting & Licensing Platform · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable event stored in a central event store. Rather than mutating rows in place, the system appends domain events (e.g., `PermitApplicationSubmitted`, `InspectionScheduled`, `FeePaymentReceived`, `ReviewDecisionMade`) to an append-only log. Current state is derived by replaying events or maintained in materialised read models (projections) that are rebuilt from the event stream.

This pattern is drawn from financial systems where immutable transaction logs are regulatory requirements, and from government audit frameworks (NIST SP 800-53 AU controls) that require demonstrable, tamper-proof records of all state transitions. The CQRS (Command Query Responsibility Segregation) companion pattern separates the write path (command handlers that validate and emit events) from the read path (projections optimised for specific query patterns like permit status dashboards, inspector schedules, and analytics).

For a permitting platform, this approach is particularly compelling because: (a) government agencies face strict audit requirements — every approval, denial, fee change, and inspection result must be traceable to a user, timestamp, and reason; (b) temporal queries ("what was the status of permit X on date Y?") are natural to answer from an event stream; (c) AI analytics models can consume the event stream directly for pattern detection, anomaly scoring, and predictive modelling.

**Best for:** Agencies with strict audit and compliance requirements, platforms needing temporal queries ("as-of" date lookups), and AI-powered analytics consuming event streams for pattern detection and prediction.

**Trade-offs:**
- (+) Complete, immutable audit trail — every state change is permanently recorded with full context
- (+) Temporal queries are trivial: replay events to any point in time to see historical state
- (+) AI/ML pipelines can consume the event stream directly for training and inference
- (+) Schema evolution is simpler: add new event types without modifying existing tables
- (+) Read models can be independently optimised for different query patterns
- (-) Higher storage requirements — events accumulate indefinitely; snapshots needed for performance
- (-) Increased complexity — developers must understand event sourcing, projections, and eventual consistency
- (-) Read model rebuild time can be significant for long-lived aggregates with many events
- (-) Debugging requires understanding the event sequence, not just current state
- (-) Eventual consistency between event store and read models requires careful handling

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| BLDS (Building & Land Development Specification) | Read model projections (views) produce BLDS-compliant permit, inspection, and contractor data for open data export |
| NIST SP 800-53 (AU Controls) | Immutable event store satisfies AU-2 (Audit Events), AU-3 (Content of Audit Records), AU-10 (Non-repudiation), and AU-11 (Audit Record Retention) |
| IBC (International Building Code) | Inspection events reference IBC code sections; code compliance check events capture rule evaluations |
| Open311 GeoReport v2 | Open311 service request events map to domain events; read model projects Open311-compliant API responses |
| NEPA Data Standard v1.2 | Environmental review events align with NEPA process/milestone entity structure |
| ISO 8601 | All event timestamps in ISO 8601 with timezone |
| ISO 3166-1/2 | Jurisdiction identification in aggregate root metadata |
| OAuth 2.0 / OIDC | Authentication events in the event stream; user identity metadata on every event |

---

## Event Store (Core Infrastructure)

```sql
-- The single source of truth: an append-only event log
CREATE TABLE events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,  -- 'permit', 'licence', 'inspection', 'enforcement_case', 'contractor', 'parcel'
    aggregate_id UUID NOT NULL,            -- ID of the domain entity this event belongs to
    sequence_num BIGINT NOT NULL,          -- Monotonically increasing within an aggregate
    event_type VARCHAR(200) NOT NULL,      -- e.g., 'PermitApplicationSubmitted', 'InspectionCompleted', 'FeeAssessed'
    event_version INTEGER NOT NULL DEFAULT 1,  -- Schema version for this event type
    
    -- Context
    jurisdiction_id UUID NOT NULL,
    caused_by_user_id UUID,               -- User who triggered the event (NULL for system events)
    caused_by_ip INET,
    correlation_id UUID,                  -- Groups related events from a single command
    causation_id UUID,                    -- The event that caused this event (for event chains)
    
    -- Payload
    event_data JSONB NOT NULL,            -- The event-specific payload (immutable after write)
    metadata JSONB DEFAULT '{}',          -- Non-domain metadata (client version, request ID, etc.)
    
    -- Timestamps
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),  -- When the event happened in the real world
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now(),  -- When the event was written to the store
    
    UNIQUE(aggregate_type, aggregate_id, sequence_num)
);

-- Primary query pattern: replay events for an aggregate
CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, sequence_num);

-- Secondary: query events by type across all aggregates (for projections)
CREATE INDEX idx_events_type ON events(event_type, recorded_at);

-- Temporal queries: find all events in a time range
CREATE INDEX idx_events_occurred ON events(occurred_at);
CREATE INDEX idx_events_recorded ON events(recorded_at);

-- Multi-tenant filtering
CREATE INDEX idx_events_jurisdiction ON events(jurisdiction_id, aggregate_type, recorded_at);

-- Correlation: find all events from a single user action
CREATE INDEX idx_events_correlation ON events(correlation_id);

-- User audit: find all events by a specific user
CREATE INDEX idx_events_user ON events(caused_by_user_id, occurred_at);

-- Partition by month for performance at scale
-- CREATE TABLE events_2026_05 PARTITION OF events
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

---

## Aggregate Snapshots (Performance Optimisation)

```sql
-- Periodic snapshots to avoid replaying long event histories
CREATE TABLE aggregate_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    sequence_num BIGINT NOT NULL,          -- Event sequence number at time of snapshot
    snapshot_data JSONB NOT NULL,          -- Serialised aggregate state
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(aggregate_type, aggregate_id, sequence_num)
);

CREATE INDEX idx_snapshots_aggregate ON aggregate_snapshots(aggregate_type, aggregate_id, sequence_num DESC);
```

---

## Event Type Catalogue

The following event types define the domain language. Each event type has a defined schema for its `event_data` JSONB payload.

### Permit Lifecycle Events

```sql
-- Example event_data payloads (stored in the JSONB event_data column):

-- PermitApplicationCreated
-- {
--   "permit_num": "BLD-2026-00142",
--   "permit_type_code": "BLDG_NEW",
--   "permit_class": "residential",
--   "work_type": "new",
--   "description": "New single-family residence",
--   "site_address": "123 Main St, Springfield, IL 62701",
--   "parcel_number": "15-23-400-001",
--   "estimated_cost": 450000.00,
--   "applicant": {"name": "Jane Smith", "email": "jane@example.com", "phone": "555-0100"},
--   "form_data": {"stories": 2, "sqft": 2400, "bedrooms": 4, "garage": true}
-- }

-- PermitApplicationSubmitted
-- {
--   "submitted_at": "2026-05-22T14:30:00Z",
--   "ai_completeness_score": 94.5,
--   "ai_missing_documents": ["structural_calculations"],
--   "document_ids": ["uuid1", "uuid2", "uuid3"]
-- }

-- PermitAssignedToReviewer
-- {
--   "reviewer_id": "uuid",
--   "reviewer_name": "Bob Johnson",
--   "department": "building",
--   "sla_target_date": "2026-06-15",
--   "assignment_reason": "auto_routed_by_permit_type"
-- }

-- PermitReviewDecisionMade
-- {
--   "decision": "approved",
--   "reviewer_id": "uuid",
--   "conditions": ["Install fire sprinklers per IFC 903.2", "Submit revised structural calcs"],
--   "code_references": ["IBC 903.2.1.1", "IBC 1612.3"],
--   "review_cycle": 2,
--   "notes": "Approved with conditions after second review cycle"
-- }

-- PermitIssued
-- {
--   "issued_date": "2026-06-10",
--   "expires_date": "2027-06-10",
--   "conditions_of_approval": ["condition1", "condition2"],
--   "total_fees_assessed": 4250.00,
--   "total_fees_paid": 4250.00
-- }

-- PermitStatusChanged
-- {
--   "previous_status": "under_review",
--   "new_status": "approved",
--   "reason": "All review departments approved"
-- }

-- PermitExpired, PermitRevoked, PermitWithdrawn, PermitSuspended, PermitCompleted
```

### Inspection Events

```sql
-- InspectionRequested
-- {
--   "inspection_type_code": "FOUNDATION",
--   "inspection_type_name": "Foundation Inspection",
--   "requested_date": "2026-07-15",
--   "requested_by": "uuid",
--   "ibc_reference": "IBC 110.3.1",
--   "notes": "Foundation pour completed, ready for inspection"
-- }

-- InspectionScheduled
-- {
--   "inspector_id": "uuid",
--   "inspector_name": "Mike Davis",
--   "scheduled_date": "2026-07-16",
--   "scheduled_time_start": "09:00",
--   "scheduled_time_end": "10:00",
--   "ai_scheduling_score": 87.3,
--   "route_optimised": true
-- }

-- InspectionCompleted
-- {
--   "result": "pass",
--   "inspector_id": "uuid",
--   "completed_at": "2026-07-16T09:45:00Z",
--   "checklist_results": [
--     {"item": "Footing depth", "result": "pass", "ibc_ref": "IBC 1809.4"},
--     {"item": "Rebar placement", "result": "pass", "ibc_ref": "IBC 1810.3.9.1"},
--     {"item": "Form alignment", "result": "pass"}
--   ],
--   "photos": ["photo_uuid_1", "photo_uuid_2"],
--   "notes": "All items satisfactory",
--   "location": {"latitude": 39.7817, "longitude": -89.6501}
-- }

-- InspectionFailed
-- {
--   "result": "fail",
--   "inspector_id": "uuid",
--   "completed_at": "2026-07-16T10:15:00Z",
--   "failures": [
--     {"item": "Rebar spacing", "ibc_ref": "IBC 1810.3.9.1", "notes": "Spacing exceeds 12 inches in SE corner"}
--   ],
--   "correction_required": true,
--   "re_inspection_required": true
-- }
```

### Fee & Payment Events

```sql
-- FeeAssessed
-- {
--   "fee_code": "BLDG_PERMIT_FEE",
--   "description": "Building permit fee",
--   "amount": 3500.00,
--   "calculation": {"type": "per_sqft", "rate": 1.46, "sqft": 2400, "result": 3504.00, "rounded": 3500.00},
--   "due_date": "2026-06-01"
-- }

-- FeePaymentReceived
-- {
--   "payment_method": "credit_card",
--   "processor": "stripe",
--   "processor_transaction_id": "pi_3PxQr2ABC123",
--   "amount": 3500.00,
--   "currency": "USD",
--   "fees_covered": [{"fee_code": "BLDG_PERMIT_FEE", "amount": 3500.00}],
--   "receipt_number": "RCP-2026-00891"
-- }

-- FeeWaived
-- {
--   "fee_code": "PLAN_REVIEW_FEE",
--   "original_amount": 750.00,
--   "waiver_reason": "Government agency project — fee waiver per council resolution 2025-142",
--   "approved_by": "uuid"
-- }

-- FeeRefunded
-- {
--   "original_payment_transaction_id": "pi_3PxQr2ABC123",
--   "refund_amount": 1750.00,
--   "refund_reason": "Permit withdrawn before plan review",
--   "processor_refund_id": "re_3QyRs3DEF456"
-- }
```

### Licence Events

```sql
-- LicenceApplicationSubmitted
-- {
--   "licence_type_code": "BUSINESS_GENERAL",
--   "business_name": "Acme Construction LLC",
--   "holder_name": "John Doe",
--   "business_address": "456 Oak Ave, Springfield, IL 62702",
--   "form_data": {"business_structure": "llc", "employee_count": 25, "naics_code": "236220"}
-- }

-- LicenceIssued, LicenceRenewed, LicenceSuspended, LicenceRevoked, LicenceExpired
```

### Code Enforcement Events

```sql
-- EnforcementCaseOpened
-- {
--   "case_number": "CE-2026-00312",
--   "source": "complaint",
--   "violation_type": "unpermitted_construction",
--   "description": "Unpermitted deck construction observed at rear of property",
--   "parcel_number": "15-23-400-001",
--   "site_address": "123 Main St",
--   "reporter": {"type": "anonymous", "method": "online_form"},
--   "priority": "normal"
-- }

-- ViolationNoticeIssued
-- {
--   "notice_number": "VN-2026-00198",
--   "notice_type": "notice_of_violation",
--   "code_sections_violated": ["IBC 105.1", "IBC 105.2"],
--   "compliance_deadline": "2026-08-15",
--   "fine_amount": 500.00,
--   "served_method": "mail"
-- }

-- EnforcementCaseResolved, EnforcementCaseEscalated
```

### Workflow Events

```sql
-- WorkflowStarted
-- {
--   "workflow_template_id": "uuid",
--   "workflow_name": "Residential Building Permit Review",
--   "total_steps": 5,
--   "steps": [
--     {"order": 1, "name": "Zoning Review", "department": "planning"},
--     {"order": 2, "name": "Building Plan Review", "department": "building"},
--     {"order": 3, "name": "Fire Review", "department": "fire"},
--     {"order": 4, "name": "Fee Assessment", "department": "finance"},
--     {"order": 5, "name": "Final Approval", "department": "building"}
--   ]
-- }

-- WorkflowStepStarted, WorkflowStepCompleted, WorkflowStepReturned
-- WorkflowCompleted, WorkflowCancelled
```

### Document Events

```sql
-- DocumentUploaded
-- {
--   "document_type": "site_plan",
--   "file_name": "site_plan_v2.pdf",
--   "file_size_bytes": 4521890,
--   "file_type": "application/pdf",
--   "version": 2,
--   "storage_path": "s3://permits-docs/2026/05/uuid.pdf",
--   "uploaded_by": "uuid",
--   "ai_classification": "site_plan",
--   "ai_confidence": 0.97
-- }

-- PlanReviewCommentAdded
-- {
--   "document_id": "uuid",
--   "reviewer_id": "uuid",
--   "review_cycle": 1,
--   "comment_type": "correction_required",
--   "code_reference": "IBC 1005.1",
--   "comment_text": "Egress width insufficient at corridor C2. Minimum 44 inches required.",
--   "page_number": 3
-- }
```

---

## Read Model Projections (Materialised Views)

These tables are rebuilt from events and serve specific query patterns. They are NOT the source of truth — the event store is.

```sql
-- ==========================================
-- PROJECTION: Current permit state
-- ==========================================
CREATE TABLE rm_permits (
    id UUID PRIMARY KEY,
    jurisdiction_id UUID NOT NULL,
    permit_num VARCHAR(100) NOT NULL,
    permit_type_code VARCHAR(50),
    permit_type_name VARCHAR(255),
    permit_class VARCHAR(50),
    work_type VARCHAR(50),
    description TEXT,
    site_address TEXT,
    parcel_number VARCHAR(100),
    
    -- Current status
    status VARCHAR(50) NOT NULL,
    status_changed_at TIMESTAMPTZ,
    
    -- Key dates
    applied_date DATE,
    issued_date DATE,
    completed_date DATE,
    expires_date DATE,
    
    -- Applicant
    applicant_name VARCHAR(255),
    applicant_email VARCHAR(255),
    
    -- Contractor
    contractor_name VARCHAR(255),
    contractor_licence VARCHAR(100),
    
    -- Valuation & fees
    estimated_cost NUMERIC(14, 2),
    total_fees_assessed NUMERIC(12, 2) DEFAULT 0,
    total_fees_paid NUMERIC(12, 2) DEFAULT 0,
    fees_outstanding NUMERIC(12, 2) DEFAULT 0,
    
    -- Workflow
    current_workflow_step VARCHAR(255),
    assigned_reviewer VARCHAR(255),
    sla_target_date DATE,
    
    -- Counts
    inspection_count INTEGER DEFAULT 0,
    inspection_pass_count INTEGER DEFAULT 0,
    inspection_fail_count INTEGER DEFAULT 0,
    document_count INTEGER DEFAULT 0,
    comment_count INTEGER DEFAULT 0,
    
    -- AI
    ai_completeness_score NUMERIC(5, 2),
    ai_risk_score NUMERIC(5, 2),
    
    -- Projection metadata
    last_event_sequence BIGINT NOT NULL,
    last_projected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(jurisdiction_id, permit_num)
);

CREATE INDEX idx_rm_permits_jurisdiction ON rm_permits(jurisdiction_id);
CREATE INDEX idx_rm_permits_status ON rm_permits(status);
CREATE INDEX idx_rm_permits_type ON rm_permits(permit_type_code);
CREATE INDEX idx_rm_permits_dates ON rm_permits(applied_date, issued_date);
CREATE INDEX idx_rm_permits_reviewer ON rm_permits(assigned_reviewer);
CREATE INDEX idx_rm_permits_sla ON rm_permits(sla_target_date) WHERE status IN ('submitted', 'under_review');

-- ==========================================
-- PROJECTION: Inspector daily schedule
-- ==========================================
CREATE TABLE rm_inspector_schedule (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL,
    inspector_id UUID NOT NULL,
    inspector_name VARCHAR(255),
    scheduled_date DATE NOT NULL,
    
    -- Inspection details
    inspection_id UUID NOT NULL,
    permit_id UUID NOT NULL,
    permit_num VARCHAR(100),
    inspection_type VARCHAR(255),
    site_address TEXT,
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    scheduled_time_start TIME,
    scheduled_time_end TIME,
    status VARCHAR(50),
    result VARCHAR(50),
    
    -- Projection metadata
    last_event_sequence BIGINT NOT NULL,
    last_projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_schedule_inspector_date ON rm_inspector_schedule(inspector_id, scheduled_date);
CREATE INDEX idx_rm_schedule_jurisdiction_date ON rm_inspector_schedule(jurisdiction_id, scheduled_date);

-- ==========================================
-- PROJECTION: BLDS open data export
-- ==========================================
CREATE TABLE rm_blds_permits (
    -- Columns map 1:1 to BLDS permits.csv specification
    permit_num VARCHAR(100) NOT NULL,
    description TEXT,
    applied_date DATE,
    issued_date DATE,
    completed_date DATE,
    status_current VARCHAR(50),
    original_address1 TEXT,
    original_city VARCHAR(100),
    original_state VARCHAR(50),
    original_zip VARCHAR(20),
    permit_type VARCHAR(100),
    permit_type_mapped VARCHAR(100),       -- BLDS standard type mapping
    permit_class VARCHAR(50),
    permit_class_mapped VARCHAR(50),       -- BLDS standard class mapping
    work_class VARCHAR(50),
    work_class_mapped VARCHAR(50),
    est_project_cost NUMERIC(14, 2),
    contractor_company_name VARCHAR(255),
    contractor_trade VARCHAR(100),
    contractor_lic_num VARCHAR(100),
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    jurisdiction_id UUID NOT NULL,
    
    -- Projection metadata
    last_projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_blds_jurisdiction ON rm_blds_permits(jurisdiction_id);

-- ==========================================
-- PROJECTION: Fee ledger
-- ==========================================
CREATE TABLE rm_fee_ledger (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL,
    permit_id UUID NOT NULL,
    permit_num VARCHAR(100),
    fee_code VARCHAR(50),
    description VARCHAR(255),
    amount_assessed NUMERIC(12, 2),
    amount_paid NUMERIC(12, 2) DEFAULT 0,
    amount_waived NUMERIC(12, 2) DEFAULT 0,
    amount_refunded NUMERIC(12, 2) DEFAULT 0,
    balance NUMERIC(12, 2),
    status VARCHAR(50),
    due_date DATE,
    last_payment_date DATE,
    receipt_number VARCHAR(100),
    
    last_event_sequence BIGINT NOT NULL,
    last_projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_fees_permit ON rm_fee_ledger(permit_id);
CREATE INDEX idx_rm_fees_jurisdiction ON rm_fee_ledger(jurisdiction_id);
CREATE INDEX idx_rm_fees_status ON rm_fee_ledger(status);

-- ==========================================
-- PROJECTION: Licence registry
-- ==========================================
CREATE TABLE rm_licences (
    id UUID PRIMARY KEY,
    jurisdiction_id UUID NOT NULL,
    licence_number VARCHAR(100),
    licence_type_code VARCHAR(50),
    licence_type_name VARCHAR(255),
    holder_name VARCHAR(255),
    business_name VARCHAR(255),
    business_address TEXT,
    status VARCHAR(50),
    issued_date DATE,
    expiry_date DATE,
    last_renewal_date DATE,
    
    last_event_sequence BIGINT NOT NULL,
    last_projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_licences_jurisdiction ON rm_licences(jurisdiction_id);
CREATE INDEX idx_rm_licences_status ON rm_licences(status);
CREATE INDEX idx_rm_licences_expiry ON rm_licences(expiry_date);

-- ==========================================
-- PROJECTION: Analytics / KPI dashboard
-- ==========================================
CREATE TABLE rm_analytics_daily (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL,
    report_date DATE NOT NULL,
    
    -- Volume metrics
    permits_submitted INTEGER DEFAULT 0,
    permits_issued INTEGER DEFAULT 0,
    permits_completed INTEGER DEFAULT 0,
    permits_denied INTEGER DEFAULT 0,
    inspections_scheduled INTEGER DEFAULT 0,
    inspections_completed INTEGER DEFAULT 0,
    inspections_passed INTEGER DEFAULT 0,
    inspections_failed INTEGER DEFAULT 0,
    
    -- Processing time metrics (days)
    avg_days_to_issue NUMERIC(8, 2),
    avg_days_in_review NUMERIC(8, 2),
    avg_inspection_turnaround NUMERIC(8, 2),
    
    -- Financial metrics
    total_fees_assessed NUMERIC(14, 2) DEFAULT 0,
    total_fees_collected NUMERIC(14, 2) DEFAULT 0,
    
    -- SLA metrics
    permits_within_sla INTEGER DEFAULT 0,
    permits_past_sla INTEGER DEFAULT 0,
    sla_compliance_rate NUMERIC(5, 2),
    
    last_projected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, report_date)
);

CREATE INDEX idx_rm_analytics_jurisdiction_date ON rm_analytics_daily(jurisdiction_id, report_date);
```

---

## Reference Data (Non-Event-Sourced)

```sql
-- These tables store static/semi-static reference data that does not need event sourcing

CREATE TABLE jurisdictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    jurisdiction_type VARCHAR(50) NOT NULL,
    iso_3166_code VARCHAR(10),
    parent_jurisdiction_id UUID REFERENCES jurisdictions(id),
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    config JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    user_type VARCHAR(50) NOT NULL,
    identity_provider VARCHAR(50) DEFAULT 'local',
    identity_provider_id VARCHAR(255),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permit_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    category VARCHAR(100) NOT NULL,
    config JSONB DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, code)
);

CREATE TABLE inspection_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    ibc_code_section VARCHAR(50),
    typical_duration_minutes INTEGER DEFAULT 30,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, code)
);

CREATE TABLE parcels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    parcel_number VARCHAR(100) NOT NULL,
    address_line1 VARCHAR(255),
    city VARCHAR(100),
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    geom GEOMETRY(Polygon, 4326),
    zoning_code VARCHAR(50),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, parcel_number)
);

-- Document blob metadata (files stored externally in S3/blob storage)
CREATE TABLE document_storage (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_name VARCHAR(255) NOT NULL,
    storage_path TEXT NOT NULL,
    file_type VARCHAR(50),
    file_size_bytes BIGINT,
    checksum VARCHAR(128),
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Projection Tracking

```sql
-- Tracks which events each projection has processed (idempotent rebuild support)
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id UUID REFERENCES events(event_id),
    last_sequence_num BIGINT NOT NULL DEFAULT 0,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status VARCHAR(50) DEFAULT 'active' CHECK (status IN ('active', 'rebuilding', 'paused', 'error')),
    error_message TEXT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that failed projection processing
CREATE TABLE projection_dead_letters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(100) NOT NULL,
    event_id UUID NOT NULL REFERENCES events(event_id),
    error_message TEXT NOT NULL,
    retry_count INTEGER DEFAULT 0,
    last_retry_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pdl_projection ON projection_dead_letters(projection_name);
```

---

## Example Queries

### Replay permit state as of a specific date

```sql
-- "What was the status of permit BLD-2026-00142 on June 1, 2026?"
SELECT event_type, event_data, occurred_at
FROM events
WHERE aggregate_type = 'permit'
  AND aggregate_id = (
      SELECT id FROM rm_permits WHERE permit_num = 'BLD-2026-00142'
  )
  AND occurred_at <= '2026-06-01T23:59:59Z'
ORDER BY sequence_num ASC;
-- Application code replays these events to reconstruct the state at that point in time
```

### Find all actions by a specific user in a time range

```sql
-- Full audit trail for user actions (NIST AU-2 compliance)
SELECT event_type, aggregate_type, aggregate_id, event_data, occurred_at
FROM events
WHERE caused_by_user_id = 'user-uuid-here'
  AND occurred_at BETWEEN '2026-05-01' AND '2026-05-31'
ORDER BY occurred_at ASC;
```

### AI analytics: Permit processing duration patterns

```sql
-- Extract submission-to-issuance durations for ML training
WITH submissions AS (
    SELECT aggregate_id, occurred_at AS submitted_at,
           (event_data->>'permit_type_code') AS permit_type,
           (event_data->>'estimated_cost')::numeric AS cost
    FROM events
    WHERE event_type = 'PermitApplicationSubmitted'
      AND jurisdiction_id = 'jurisdiction-uuid'
),
issuances AS (
    SELECT aggregate_id, occurred_at AS issued_at
    FROM events
    WHERE event_type = 'PermitIssued'
      AND jurisdiction_id = 'jurisdiction-uuid'
)
SELECT s.permit_type, s.cost,
       EXTRACT(EPOCH FROM (i.issued_at - s.submitted_at)) / 86400 AS days_to_issue
FROM submissions s
JOIN issuances i ON s.aggregate_id = i.aggregate_id;
```

### Detect permits with anomalous review patterns

```sql
-- Find permits where fees were waived after initial assessment (anomaly detection input)
SELECT e1.aggregate_id,
       e1.event_data->>'fee_code' AS fee_code,
       (e1.event_data->>'amount')::numeric AS original_amount,
       e2.event_data->>'waiver_reason' AS waiver_reason,
       e2.caused_by_user_id AS waived_by
FROM events e1
JOIN events e2 ON e1.aggregate_id = e2.aggregate_id
WHERE e1.event_type = 'FeeAssessed'
  AND e2.event_type = 'FeeWaived'
  AND e1.event_data->>'fee_code' = e2.event_data->>'fee_code'
ORDER BY e2.occurred_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | events (partitioned by month) |
| Snapshots | 1 | aggregate_snapshots |
| Projection Infrastructure | 2 | projection_checkpoints, projection_dead_letters |
| Reference Data | 6 | jurisdictions, users, permit_types, inspection_types, parcels, document_storage |
| Read Model: Permits | 1 | rm_permits |
| Read Model: Inspections | 1 | rm_inspector_schedule |
| Read Model: BLDS Export | 1 | rm_blds_permits |
| Read Model: Fees | 1 | rm_fee_ledger |
| Read Model: Licences | 1 | rm_licences |
| Read Model: Analytics | 1 | rm_analytics_daily |
| **Total** | **~16** | Plus additional read models as needed |

---

## Key Design Decisions

1. **Single `events` table as sole source of truth** — all domain state changes are captured as immutable events. Current state is always derivable by replaying events from the beginning (or from the last snapshot). This guarantees a complete, tamper-proof audit trail.

2. **Aggregate-scoped event sequences** — `sequence_num` is unique per aggregate, providing optimistic concurrency control. If two concurrent commands try to append events with the same sequence number, one will fail and must retry.

3. **JSONB event payloads with versioned schemas** — `event_version` enables schema evolution. When event structure changes, new versions are emitted while old events remain valid. Projection code handles both old and new versions.

4. **Correlation and causation IDs** — `correlation_id` links all events from a single user command (e.g., submitting a permit may generate `PermitApplicationSubmitted`, `FeeAssessed`, and `WorkflowStarted` events). `causation_id` tracks event chains (e.g., `InspectionFailed` causes `ReInspectionRequired`).

5. **Read models are disposable and rebuildable** — every `rm_*` table can be dropped and rebuilt from the event store. This means read model schema changes (adding columns, changing indexes) require no data migration — just rebuild the projection.

6. **BLDS export as a dedicated projection** — the `rm_blds_permits` table maps directly to BLDS field names, making open data CSV export a simple `SELECT *` operation.

7. **Reference data tables are NOT event-sourced** — jurisdictions, users, permit types, and parcels are semi-static data that changes rarely. Event sourcing them would add complexity without meaningful benefit. They are managed with traditional CRUD and `updated_at` timestamps.

8. **Monthly partitioning on events table** — at scale, the events table will grow rapidly. Partitioning by `recorded_at` month enables efficient pruning of old partitions to cold storage while keeping recent events on fast storage.

9. **Projection checkpoint tracking** — the `projection_checkpoints` table ensures projections are idempotent and resumable. If a projection fails mid-rebuild, it can resume from the last successfully processed event.

10. **AI/ML as first-class event consumers** — the event stream is the ideal input for ML pipelines. Training data for processing time prediction, anomaly detection, and scheduling optimisation is extracted directly from events without additional ETL.
