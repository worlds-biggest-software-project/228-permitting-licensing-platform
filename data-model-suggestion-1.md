# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Permitting & Licensing Platform · Created: 2026-05-22

## Philosophy

This model follows a fully normalized relational design where every domain concept receives its own table with well-defined foreign key relationships. The schema is structured around the core lifecycle of permits and licences: application intake, review workflow, inspection scheduling, fee assessment, and compliance tracking. Each entity is modelled explicitly with typed columns, enforced constraints, and referential integrity.

The approach mirrors how mature government platforms like Accela structure their data model: root objects (Records, Inspections, Parcels, Owners, Conditions, Fees, Workflows) with supporting sub-objects for contacts, documents, and history. By fully normalizing, we avoid data duplication, ensure consistency through foreign keys, and make cross-entity queries straightforward using standard SQL joins.

This design is best suited for teams that value data integrity above all else, expect complex cross-entity reporting (e.g., "all permits for parcels owned by entity X with outstanding inspection failures"), and operate in regulatory environments where referential constraints prevent orphaned or inconsistent records.

**Best for:** Government agencies needing strong data integrity, complex cross-entity queries, and regulatory audit compliance in a traditional relational database environment.

**Trade-offs:**
- (+) Maximum data integrity through foreign key constraints and CHECK constraints
- (+) Straightforward complex queries across entities using standard SQL joins
- (+) Well-understood by government IT teams experienced with relational databases
- (+) Clean alignment with BLDS open data export requirements
- (-) Higher table count (~45-50 tables) increases schema complexity
- (-) Schema migrations required when new permit types or jurisdiction-specific fields are needed
- (-) Many-to-many junction tables add query complexity for relationship traversal
- (-) Less flexible for jurisdictions with widely varying data requirements

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| BLDS (Building & Land Development Specification) | `permits`, `inspections`, and `contractors` tables map directly to BLDS CSV schemas; BLDS field names used as column names where applicable (e.g., `permit_num`, `applied_date`, `issued_date`) |
| IBC (International Building Code) | `permit_types` and `inspection_checklists` reference IBC occupancy classifications and code sections |
| Open311 GeoReport v2 | `service_requests` table models Open311 service request entities for CRM integration |
| ISO 3166-1/2 | `jurisdictions` table uses ISO 3166 codes for country and subdivision identification |
| ISO 8601 | All timestamps stored as `TIMESTAMPTZ` in ISO 8601 format |
| NEPA Data Standard v1.2 | `environmental_reviews` table aligns with NEPA project/process/document entity structure |
| IFC (ISO 16739) | `bim_submissions` table stores IFC file references for digital building permit workflows |
| OAuth 2.0 / OIDC | `users` and `auth_sessions` tables support federated identity (Login.gov compatible) |
| WCAG 2.1 AA | Data model supports multi-language content via `translations` table |
| PCI DSS | No cardholder data stored; `payments` table references external payment processor transaction IDs |

---

## Core Identity & Multi-Tenancy

```sql
-- Each jurisdiction (city, county, state agency) is a tenant
CREATE TABLE jurisdictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    jurisdiction_type VARCHAR(50) NOT NULL CHECK (jurisdiction_type IN ('city', 'county', 'state', 'federal', 'special_district')),
    iso_3166_code VARCHAR(10),           -- ISO 3166-1/2 code (e.g., 'US-CA', 'US-CA-037')
    parent_jurisdiction_id UUID REFERENCES jurisdictions(id),
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    config JSONB DEFAULT '{}',           -- Jurisdiction-specific configuration
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jurisdictions_type ON jurisdictions(jurisdiction_type);
CREATE INDEX idx_jurisdictions_parent ON jurisdictions(parent_jurisdiction_id);

-- Departments within a jurisdiction
CREATE TABLE departments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) NOT NULL,            -- e.g., 'BLDG', 'FIRE', 'PLAN', 'PW'
    department_type VARCHAR(50) NOT NULL CHECK (department_type IN ('building', 'fire', 'planning', 'public_works', 'environmental', 'licensing', 'code_enforcement', 'other')),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, code)
);

CREATE INDEX idx_departments_jurisdiction ON departments(jurisdiction_id);
```

---

## User & Access Management

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    identity_provider VARCHAR(50) DEFAULT 'local',  -- 'local', 'login_gov', 'azure_ad', 'okta'
    identity_provider_id VARCHAR(255),                -- External IdP subject identifier
    user_type VARCHAR(50) NOT NULL CHECK (user_type IN ('staff', 'applicant', 'contractor', 'inspector', 'admin', 'system')),
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_type ON users(user_type);
CREATE INDEX idx_users_idp ON users(identity_provider, identity_provider_id);

-- Many-to-many: users can belong to multiple departments across jurisdictions
CREATE TABLE user_department_roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    department_id UUID NOT NULL REFERENCES departments(id),
    role VARCHAR(50) NOT NULL CHECK (role IN ('viewer', 'clerk', 'reviewer', 'inspector', 'supervisor', 'admin')),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by UUID REFERENCES users(id),
    revoked_at TIMESTAMPTZ,
    UNIQUE(user_id, department_id, role)
);

CREATE INDEX idx_udr_user ON user_department_roles(user_id);
CREATE INDEX idx_udr_department ON user_department_roles(department_id);

-- Permissions per role (RBAC)
CREATE TABLE role_permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    role VARCHAR(50) NOT NULL,
    resource VARCHAR(100) NOT NULL,       -- e.g., 'permits', 'inspections', 'fees'
    action VARCHAR(50) NOT NULL,          -- e.g., 'read', 'create', 'update', 'delete', 'approve'
    UNIQUE(jurisdiction_id, role, resource, action)
);

CREATE INDEX idx_role_permissions_role ON role_permissions(jurisdiction_id, role);
```

---

## Parcel & Address Management

```sql
-- Parcels (land records) — aligned with BLDS parcel references
CREATE TABLE parcels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    parcel_number VARCHAR(100) NOT NULL,  -- APN/AIN — Assessor Parcel Number
    legal_description TEXT,
    address_line1 VARCHAR(255),
    address_line2 VARCHAR(100),
    city VARCHAR(100),
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    country_code VARCHAR(2) DEFAULT 'US',
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    geom GEOMETRY(Polygon, 4326),         -- PostGIS geometry for parcel boundary
    zoning_code VARCHAR(50),
    zoning_description VARCHAR(255),
    lot_size_sqft NUMERIC(12, 2),
    land_use VARCHAR(100),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, parcel_number)
);

CREATE INDEX idx_parcels_jurisdiction ON parcels(jurisdiction_id);
CREATE INDEX idx_parcels_number ON parcels(parcel_number);
CREATE INDEX idx_parcels_geom ON parcels USING GIST(geom);
CREATE INDEX idx_parcels_zoning ON parcels(jurisdiction_id, zoning_code);

-- Parcel owners
CREATE TABLE parcel_owners (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parcel_id UUID NOT NULL REFERENCES parcels(id),
    owner_name VARCHAR(255) NOT NULL,
    owner_type VARCHAR(50) CHECK (owner_type IN ('individual', 'business', 'trust', 'government', 'other')),
    mailing_address TEXT,
    phone VARCHAR(50),
    email VARCHAR(255),
    ownership_percentage NUMERIC(5, 2) DEFAULT 100.00,
    effective_date DATE,
    end_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_parcel_owners_parcel ON parcel_owners(parcel_id);
CREATE INDEX idx_parcel_owners_name ON parcel_owners(owner_name);
```

---

## Permit Type Configuration

```sql
-- Configurable permit types per jurisdiction
CREATE TABLE permit_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    code VARCHAR(50) NOT NULL,            -- e.g., 'BLDG_NEW', 'BLDG_REMODEL', 'ELEC', 'PLUMB'
    name VARCHAR(255) NOT NULL,
    category VARCHAR(100) NOT NULL,       -- Aligns with BLDS PermitClass: 'residential', 'commercial', 'industrial', 'other'
    subcategory VARCHAR(100),
    description TEXT,
    ibc_occupancy_group VARCHAR(10),      -- IBC occupancy classification (A-1, B, R-1, etc.)
    requires_plan_review BOOLEAN DEFAULT false,
    requires_inspection BOOLEAN DEFAULT true,
    default_fee_schedule_id UUID,         -- FK added after fee_schedules table
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, code)
);

CREATE INDEX idx_permit_types_jurisdiction ON permit_types(jurisdiction_id);
CREATE INDEX idx_permit_types_category ON permit_types(category);

-- Form field definitions per permit type (configurable intake forms)
CREATE TABLE permit_type_fields (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_type_id UUID NOT NULL REFERENCES permit_types(id),
    field_name VARCHAR(100) NOT NULL,
    field_label VARCHAR(255) NOT NULL,
    field_type VARCHAR(50) NOT NULL CHECK (field_type IN ('text', 'textarea', 'number', 'date', 'select', 'multiselect', 'checkbox', 'file_upload', 'address', 'parcel_lookup')),
    is_required BOOLEAN DEFAULT false,
    display_order INTEGER NOT NULL DEFAULT 0,
    validation_rules JSONB DEFAULT '{}',  -- e.g., {"min": 0, "max": 999999, "pattern": "^[A-Z]{3}"}
    options JSONB,                        -- For select/multiselect: [{"value": "new", "label": "New Construction"}, ...]
    conditional_on JSONB,                 -- Show/hide based on other field values
    help_text TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(permit_type_id, field_name)
);

CREATE INDEX idx_ptf_permit_type ON permit_type_fields(permit_type_id);

-- Required document types per permit type
CREATE TABLE permit_type_required_documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_type_id UUID NOT NULL REFERENCES permit_types(id),
    document_type VARCHAR(100) NOT NULL,  -- e.g., 'site_plan', 'floor_plan', 'structural_calcs', 'proof_of_ownership'
    description TEXT,
    is_required BOOLEAN DEFAULT true,
    display_order INTEGER DEFAULT 0,
    accepted_formats TEXT[] DEFAULT ARRAY['pdf'],
    max_file_size_mb INTEGER DEFAULT 50,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ptrd_permit_type ON permit_type_required_documents(permit_type_id);
```

---

## Permits (Core Record)

```sql
-- The central permit/licence record — aligns with BLDS permits.csv and Accela "Record"
CREATE TABLE permits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    permit_type_id UUID NOT NULL REFERENCES permit_types(id),
    
    -- BLDS-aligned fields
    permit_num VARCHAR(100) NOT NULL,     -- BLDS: PermitNum (unique within jurisdiction)
    description TEXT,                      -- BLDS: Description
    applied_date DATE,                    -- BLDS: AppliedDate
    issued_date DATE,                     -- BLDS: IssuedDate
    completed_date DATE,                  -- BLDS: CompletedDate (final inspection/CO)
    expires_date DATE,                    -- BLDS: ExpiresDate
    status VARCHAR(50) NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'submitted', 'under_review', 'revisions_requested',
        'approved', 'issued', 'inspection_scheduled', 'inspection_complete',
        'completed', 'expired', 'denied', 'withdrawn', 'suspended', 'revoked'
    )),
    status_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- BLDS: PermitClass and PermitType
    permit_class VARCHAR(50),             -- 'residential', 'commercial', 'industrial'
    work_type VARCHAR(50),                -- 'new', 'addition', 'alteration', 'demolition', 'repair'
    
    -- Location
    parcel_id UUID REFERENCES parcels(id),
    site_address TEXT,                    -- BLDS: OriginalAddress1
    
    -- Valuation
    estimated_cost NUMERIC(14, 2),        -- BLDS: EstProjectCost
    declared_valuation NUMERIC(14, 2),
    
    -- Applicant
    applicant_id UUID REFERENCES users(id),
    applicant_name VARCHAR(255),
    applicant_email VARCHAR(255),
    applicant_phone VARCHAR(50),
    
    -- Contractor
    contractor_id UUID REFERENCES contractors(id),
    
    -- Processing
    assigned_reviewer_id UUID REFERENCES users(id),
    current_workflow_step_id UUID,        -- FK added after workflow tables
    sla_target_date DATE,
    
    -- Custom form data (answers to permit_type_fields)
    form_data JSONB DEFAULT '{}',
    
    -- AI features
    ai_completeness_score NUMERIC(5, 2),  -- 0-100 score from AI completeness checker
    ai_completeness_checked_at TIMESTAMPTZ,
    ai_risk_score NUMERIC(5, 2),          -- Predictive compliance risk
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, permit_num)
);

CREATE INDEX idx_permits_jurisdiction ON permits(jurisdiction_id);
CREATE INDEX idx_permits_type ON permits(permit_type_id);
CREATE INDEX idx_permits_status ON permits(status);
CREATE INDEX idx_permits_parcel ON permits(parcel_id);
CREATE INDEX idx_permits_applicant ON permits(applicant_id);
CREATE INDEX idx_permits_contractor ON permits(contractor_id);
CREATE INDEX idx_permits_reviewer ON permits(assigned_reviewer_id);
CREATE INDEX idx_permits_applied_date ON permits(applied_date);
CREATE INDEX idx_permits_issued_date ON permits(issued_date);
CREATE INDEX idx_permits_num ON permits(permit_num);

-- Permit status history — aligns with BLDS permits_history.csv
CREATE TABLE permit_status_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_id UUID NOT NULL REFERENCES permits(id),
    previous_status VARCHAR(50),
    new_status VARCHAR(50) NOT NULL,
    changed_by UUID REFERENCES users(id),
    change_reason TEXT,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_psh_permit ON permit_status_history(permit_id);
CREATE INDEX idx_psh_changed_at ON permit_status_history(changed_at);
```

---

## Contractors & Professionals

```sql
-- Licensed contractors — aligns with BLDS contractors.csv
CREATE TABLE contractors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    user_id UUID REFERENCES users(id),    -- Link to user account if they have portal access
    business_name VARCHAR(255) NOT NULL,
    contact_name VARCHAR(255),
    licence_number VARCHAR(100),          -- BLDS: ContractorLicNum
    licence_type VARCHAR(100),
    licence_state VARCHAR(50),
    licence_expiry_date DATE,
    phone VARCHAR(50),                    -- BLDS: ContractorPhone
    email VARCHAR(255),
    address TEXT,                         -- BLDS: ContractorAddress
    insurance_policy_number VARCHAR(100),
    insurance_expiry_date DATE,
    bonding_amount NUMERIC(14, 2),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contractors_jurisdiction ON contractors(jurisdiction_id);
CREATE INDEX idx_contractors_licence ON contractors(licence_number);
CREATE INDEX idx_contractors_user ON contractors(user_id);

-- Junction: permits <-> contractors (BLDS permit_contractor.csv)
CREATE TABLE permit_contractors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_id UUID NOT NULL REFERENCES permits(id),
    contractor_id UUID NOT NULL REFERENCES contractors(id),
    role VARCHAR(100) NOT NULL DEFAULT 'general',  -- 'general', 'electrical', 'plumbing', 'mechanical', 'fire_protection'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(permit_id, contractor_id, role)
);

CREATE INDEX idx_pc_permit ON permit_contractors(permit_id);
CREATE INDEX idx_pc_contractor ON permit_contractors(contractor_id);
```

---

## Workflow Engine

```sql
-- Workflow templates define the review process for each permit type
CREATE TABLE workflow_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    permit_type_id UUID REFERENCES permit_types(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wt_jurisdiction ON workflow_templates(jurisdiction_id);
CREATE INDEX idx_wt_permit_type ON workflow_templates(permit_type_id);

-- Steps within a workflow template
CREATE TABLE workflow_template_steps (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_template_id UUID NOT NULL REFERENCES workflow_templates(id),
    step_order INTEGER NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    assigned_department_id UUID REFERENCES departments(id),
    assigned_role VARCHAR(50),            -- Role within department that handles this step
    step_type VARCHAR(50) NOT NULL CHECK (step_type IN ('review', 'approval', 'inspection', 'payment', 'notification', 'parallel_review', 'conditional')),
    sla_days INTEGER,                     -- Target days to complete this step
    is_required BOOLEAN DEFAULT true,
    condition_expression JSONB,           -- Conditions for conditional steps
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(workflow_template_id, step_order)
);

CREATE INDEX idx_wts_template ON workflow_template_steps(workflow_template_id);

-- Step dependencies (for parallel/branching workflows)
CREATE TABLE workflow_step_dependencies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    step_id UUID NOT NULL REFERENCES workflow_template_steps(id),
    depends_on_step_id UUID NOT NULL REFERENCES workflow_template_steps(id),
    dependency_type VARCHAR(50) DEFAULT 'finish_to_start' CHECK (dependency_type IN ('finish_to_start', 'start_to_start', 'finish_to_finish')),
    UNIQUE(step_id, depends_on_step_id)
);

-- Active workflow instances for permits
CREATE TABLE workflow_instances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_id UUID NOT NULL REFERENCES permits(id),
    workflow_template_id UUID NOT NULL REFERENCES workflow_templates(id),
    status VARCHAR(50) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'completed', 'cancelled', 'suspended')),
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wi_permit ON workflow_instances(permit_id);
CREATE INDEX idx_wi_status ON workflow_instances(status);

-- Individual step instances within a workflow
CREATE TABLE workflow_step_instances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_instance_id UUID NOT NULL REFERENCES workflow_instances(id),
    template_step_id UUID NOT NULL REFERENCES workflow_template_steps(id),
    assigned_user_id UUID REFERENCES users(id),
    status VARCHAR(50) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'in_progress', 'completed', 'approved', 'rejected', 'skipped', 'returned')),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    due_date DATE,
    decision VARCHAR(50),                 -- 'approve', 'deny', 'revise', 'hold'
    decision_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wsi_workflow ON workflow_step_instances(workflow_instance_id);
CREATE INDEX idx_wsi_assigned ON workflow_step_instances(assigned_user_id);
CREATE INDEX idx_wsi_status ON workflow_step_instances(status);
CREATE INDEX idx_wsi_due_date ON workflow_step_instances(due_date);
```

---

## Inspections

```sql
-- Inspection types configured per jurisdiction
CREATE TABLE inspection_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    ibc_code_section VARCHAR(50),         -- IBC reference (e.g., 'IBC 110.3.1')
    typical_duration_minutes INTEGER DEFAULT 30,
    requires_re_inspection_on_fail BOOLEAN DEFAULT true,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, code)
);

CREATE INDEX idx_it_jurisdiction ON inspection_types(jurisdiction_id);

-- Inspection checklist templates
CREATE TABLE inspection_checklists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_type_id UUID NOT NULL REFERENCES inspection_types(id),
    item_order INTEGER NOT NULL,
    item_description TEXT NOT NULL,
    ibc_reference VARCHAR(100),           -- Specific code reference for this checklist item
    is_critical BOOLEAN DEFAULT false,    -- Failure = automatic inspection failure
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ic_type ON inspection_checklists(inspection_type_id);

-- Required inspections per permit type
CREATE TABLE permit_type_inspections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_type_id UUID NOT NULL REFERENCES permit_types(id),
    inspection_type_id UUID NOT NULL REFERENCES inspection_types(id),
    inspection_order INTEGER NOT NULL,
    is_required BOOLEAN DEFAULT true,
    UNIQUE(permit_type_id, inspection_type_id)
);

-- Scheduled/completed inspections — aligns with BLDS inspections.csv
CREATE TABLE inspections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_id UUID NOT NULL REFERENCES permits(id),
    inspection_type_id UUID NOT NULL REFERENCES inspection_types(id),
    parent_inspection_id UUID REFERENCES inspections(id),  -- For re-inspections
    inspector_id UUID REFERENCES users(id),
    
    -- BLDS-aligned fields
    inspection_num VARCHAR(100),
    scheduled_date DATE,
    scheduled_time_start TIME,
    scheduled_time_end TIME,
    completed_date DATE,
    
    status VARCHAR(50) NOT NULL DEFAULT 'requested' CHECK (status IN (
        'requested', 'scheduled', 'in_progress', 'passed', 'failed',
        'partial', 'cancelled', 'no_access', 'rescheduled'
    )),
    result VARCHAR(50) CHECK (result IN ('pass', 'fail', 'partial', 'not_applicable')),
    result_notes TEXT,
    
    -- Location
    site_address TEXT,
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    
    -- AI features
    ai_scheduling_score NUMERIC(5, 2),    -- ML-optimised scheduling priority
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspections_permit ON inspections(permit_id);
CREATE INDEX idx_inspections_inspector ON inspections(inspector_id);
CREATE INDEX idx_inspections_status ON inspections(status);
CREATE INDEX idx_inspections_scheduled ON inspections(scheduled_date);
CREATE INDEX idx_inspections_type ON inspections(inspection_type_id);
CREATE INDEX idx_inspections_parent ON inspections(parent_inspection_id);

-- Inspection checklist results
CREATE TABLE inspection_checklist_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id UUID NOT NULL REFERENCES inspections(id),
    checklist_item_id UUID NOT NULL REFERENCES inspection_checklists(id),
    result VARCHAR(20) CHECK (result IN ('pass', 'fail', 'na', 'not_inspected')),
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_icr_inspection ON inspection_checklist_results(inspection_id);

-- Inspection photos and field documentation
CREATE TABLE inspection_attachments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id UUID NOT NULL REFERENCES inspections(id),
    file_name VARCHAR(255) NOT NULL,
    file_path TEXT NOT NULL,
    file_type VARCHAR(50),
    file_size_bytes BIGINT,
    caption TEXT,
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    captured_at TIMESTAMPTZ,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ia_inspection ON inspection_attachments(inspection_id);
```

---

## Fees & Payments

```sql
-- Fee schedules per jurisdiction
CREATE TABLE fee_schedules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    name VARCHAR(255) NOT NULL,
    effective_date DATE NOT NULL,
    end_date DATE,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fs_jurisdiction ON fee_schedules(jurisdiction_id);
CREATE INDEX idx_fs_effective ON fee_schedules(effective_date);

-- Fee items within a schedule
CREATE TABLE fee_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fee_schedule_id UUID NOT NULL REFERENCES fee_schedules(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    calculation_type VARCHAR(50) NOT NULL CHECK (calculation_type IN ('flat', 'per_sqft', 'per_unit', 'percentage', 'tiered', 'formula')),
    base_amount NUMERIC(12, 2),
    rate NUMERIC(12, 6),                  -- For per_sqft, per_unit, percentage calculations
    formula TEXT,                          -- For complex calculations: e.g., 'CASE WHEN valuation < 50000 THEN 100 ELSE valuation * 0.002 END'
    min_amount NUMERIC(12, 2),
    max_amount NUMERIC(12, 2),
    applies_to_permit_types UUID[],       -- Array of permit_type IDs this fee applies to
    is_refundable BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fi_schedule ON fee_items(fee_schedule_id);

-- Fees assessed on a specific permit
CREATE TABLE permit_fees (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_id UUID NOT NULL REFERENCES permits(id),
    fee_item_id UUID REFERENCES fee_items(id),
    description VARCHAR(255) NOT NULL,
    amount NUMERIC(12, 2) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'invoiced', 'paid', 'waived', 'refunded', 'partially_paid')),
    due_date DATE,
    paid_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pf_permit ON permit_fees(permit_id);
CREATE INDEX idx_pf_status ON permit_fees(status);

-- Payment transactions
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    payer_id UUID REFERENCES users(id),
    payment_method VARCHAR(50) CHECK (payment_method IN ('credit_card', 'debit_card', 'ach', 'check', 'cash', 'wire', 'online')),
    payment_processor VARCHAR(50),        -- 'stripe', 'paypal', 'authorize_net', 'govtech_pay'
    processor_transaction_id VARCHAR(255),-- External payment processor reference
    total_amount NUMERIC(12, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    status VARCHAR(50) NOT NULL CHECK (status IN ('pending', 'completed', 'failed', 'refunded', 'partially_refunded')),
    receipt_number VARCHAR(100),
    paid_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_jurisdiction ON payments(jurisdiction_id);
CREATE INDEX idx_payments_payer ON payments(payer_id);
CREATE INDEX idx_payments_processor_txn ON payments(processor_transaction_id);

-- Junction: which fees are covered by which payment
CREATE TABLE payment_fee_allocations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id UUID NOT NULL REFERENCES payments(id),
    permit_fee_id UUID NOT NULL REFERENCES permit_fees(id),
    amount NUMERIC(12, 2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pfa_payment ON payment_fee_allocations(payment_id);
CREATE INDEX idx_pfa_fee ON payment_fee_allocations(permit_fee_id);
```

---

## Documents & Plan Review

```sql
-- Documents attached to permits
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_id UUID NOT NULL REFERENCES permits(id),
    document_type VARCHAR(100) NOT NULL,  -- 'site_plan', 'floor_plan', 'structural_calcs', 'elevation', 'survey', 'photo', 'correspondence'
    file_name VARCHAR(255) NOT NULL,
    file_path TEXT NOT NULL,              -- S3/blob storage path
    file_type VARCHAR(50),               -- MIME type
    file_size_bytes BIGINT,
    version INTEGER NOT NULL DEFAULT 1,
    is_current_version BOOLEAN DEFAULT true,
    previous_version_id UUID REFERENCES documents(id),
    uploaded_by UUID REFERENCES users(id),
    
    -- AI document analysis
    ai_classification VARCHAR(100),       -- AI-detected document type
    ai_completeness_status VARCHAR(50),   -- 'complete', 'incomplete', 'unreadable'
    ai_analysis_notes TEXT,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_permit ON documents(permit_id);
CREATE INDEX idx_documents_type ON documents(document_type);
CREATE INDEX idx_documents_current ON documents(is_current_version) WHERE is_current_version = true;

-- Plan review comments and markups
CREATE TABLE plan_review_comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(id),
    reviewer_id UUID NOT NULL REFERENCES users(id),
    review_cycle INTEGER NOT NULL DEFAULT 1,
    comment_type VARCHAR(50) CHECK (comment_type IN ('correction_required', 'information_needed', 'comment', 'approval', 'code_violation')),
    code_reference VARCHAR(100),          -- e.g., 'IBC 1005.1', 'IFC 903.2'
    comment_text TEXT NOT NULL,
    page_number INTEGER,
    location_x NUMERIC(8, 2),             -- Markup coordinates
    location_y NUMERIC(8, 2),
    status VARCHAR(50) DEFAULT 'open' CHECK (status IN ('open', 'resolved', 'acknowledged', 'disputed')),
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_prc_document ON plan_review_comments(document_id);
CREATE INDEX idx_prc_reviewer ON plan_review_comments(reviewer_id);
CREATE INDEX idx_prc_status ON plan_review_comments(status);
```

---

## Licences (Business & Professional)

```sql
-- Licence types (business, contractor, professional)
CREATE TABLE licence_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    category VARCHAR(100) NOT NULL CHECK (category IN ('business', 'contractor', 'professional', 'special_event', 'other')),
    renewal_period_months INTEGER DEFAULT 12,
    requires_inspection BOOLEAN DEFAULT false,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, code)
);

CREATE INDEX idx_lt_jurisdiction ON licence_types(jurisdiction_id);

-- Active licences
CREATE TABLE licences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    licence_type_id UUID NOT NULL REFERENCES licence_types(id),
    licence_number VARCHAR(100) NOT NULL,
    holder_name VARCHAR(255) NOT NULL,
    holder_user_id UUID REFERENCES users(id),
    holder_type VARCHAR(50) CHECK (holder_type IN ('individual', 'business')),
    business_name VARCHAR(255),
    business_address TEXT,
    parcel_id UUID REFERENCES parcels(id),
    status VARCHAR(50) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'active', 'suspended', 'revoked', 'expired', 'denied', 'renewal_pending'
    )),
    issued_date DATE,
    effective_date DATE,
    expiry_date DATE,
    last_renewal_date DATE,
    conditions TEXT,                       -- Conditions of licence
    form_data JSONB DEFAULT '{}',         -- Licence-specific custom fields
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, licence_number)
);

CREATE INDEX idx_licences_jurisdiction ON licences(jurisdiction_id);
CREATE INDEX idx_licences_type ON licences(licence_type_id);
CREATE INDEX idx_licences_holder ON licences(holder_user_id);
CREATE INDEX idx_licences_status ON licences(status);
CREATE INDEX idx_licences_expiry ON licences(expiry_date);
CREATE INDEX idx_licences_number ON licences(licence_number);
```

---

## Code Enforcement

```sql
-- Code enforcement cases
CREATE TABLE code_enforcement_cases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    case_number VARCHAR(100) NOT NULL,
    parcel_id UUID REFERENCES parcels(id),
    site_address TEXT,
    reported_by UUID REFERENCES users(id),
    assigned_officer_id UUID REFERENCES users(id),
    source VARCHAR(50) CHECK (source IN ('complaint', 'inspection', 'proactive', 'referral', 'ai_detected', 'open311')),
    violation_type VARCHAR(100),
    description TEXT NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'open' CHECK (status IN (
        'open', 'investigating', 'notice_issued', 'compliance_pending',
        'resolved', 'closed', 'escalated', 'hearing_scheduled'
    )),
    priority VARCHAR(20) DEFAULT 'normal' CHECK (priority IN ('low', 'normal', 'high', 'urgent')),
    opened_date DATE NOT NULL DEFAULT CURRENT_DATE,
    compliance_deadline DATE,
    resolved_date DATE,
    related_permit_id UUID REFERENCES permits(id),
    related_licence_id UUID REFERENCES licences(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, case_number)
);

CREATE INDEX idx_cec_jurisdiction ON code_enforcement_cases(jurisdiction_id);
CREATE INDEX idx_cec_parcel ON code_enforcement_cases(parcel_id);
CREATE INDEX idx_cec_officer ON code_enforcement_cases(assigned_officer_id);
CREATE INDEX idx_cec_status ON code_enforcement_cases(status);

-- Violation notices
CREATE TABLE violation_notices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_id UUID NOT NULL REFERENCES code_enforcement_cases(id),
    notice_number VARCHAR(100) NOT NULL,
    notice_type VARCHAR(50) CHECK (notice_type IN ('warning', 'notice_of_violation', 'citation', 'stop_work_order', 'abatement_order')),
    code_sections_violated TEXT[],        -- Array of code section references
    description TEXT NOT NULL,
    compliance_deadline DATE,
    fine_amount NUMERIC(12, 2),
    issued_date DATE NOT NULL DEFAULT CURRENT_DATE,
    served_date DATE,
    served_method VARCHAR(50),            -- 'mail', 'hand_delivery', 'posting', 'email'
    issued_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vn_case ON violation_notices(case_id);
```

---

## Notifications

```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    recipient_id UUID REFERENCES users(id),
    recipient_email VARCHAR(255),
    recipient_phone VARCHAR(50),
    channel VARCHAR(20) NOT NULL CHECK (channel IN ('email', 'sms', 'in_app', 'push')),
    notification_type VARCHAR(100) NOT NULL,  -- 'permit_status_change', 'inspection_scheduled', 'fee_due', 'licence_renewal', etc.
    subject VARCHAR(255),
    body TEXT NOT NULL,
    related_permit_id UUID REFERENCES permits(id),
    related_inspection_id UUID REFERENCES inspections(id),
    related_licence_id UUID REFERENCES licences(id),
    status VARCHAR(50) DEFAULT 'pending' CHECK (status IN ('pending', 'sent', 'delivered', 'failed', 'bounced')),
    sent_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_recipient ON notifications(recipient_id);
CREATE INDEX idx_notifications_status ON notifications(status);
CREATE INDEX idx_notifications_type ON notifications(notification_type);
CREATE INDEX idx_notifications_permit ON notifications(related_permit_id);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    user_id UUID REFERENCES users(id),
    action VARCHAR(50) NOT NULL,          -- 'create', 'update', 'delete', 'approve', 'deny', 'login', 'export'
    entity_type VARCHAR(100) NOT NULL,    -- 'permit', 'inspection', 'licence', 'document', 'user', etc.
    entity_id UUID NOT NULL,
    old_values JSONB,                     -- Previous field values (for updates)
    new_values JSONB,                     -- New field values (for creates/updates)
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_jurisdiction ON audit_log(jurisdiction_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_action ON audit_log(action);
CREATE INDEX idx_audit_created ON audit_log(created_at);

-- Partition audit_log by month for performance
-- CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

---

## Open311 / Service Requests Integration

```sql
-- Open311 service requests — for CRM integration
CREATE TABLE service_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    service_request_id VARCHAR(100) NOT NULL,  -- Open311: service_request_id
    service_code VARCHAR(50) NOT NULL,         -- Open311: service_code
    service_name VARCHAR(255),                 -- Open311: service_name
    description TEXT,
    status VARCHAR(50) NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'closed')),
    address_string TEXT,
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    requested_datetime TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_datetime TIMESTAMPTZ,
    
    -- Link to internal entities
    related_permit_id UUID REFERENCES permits(id),
    related_case_id UUID REFERENCES code_enforcement_cases(id),
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, service_request_id)
);

CREATE INDEX idx_sr_jurisdiction ON service_requests(jurisdiction_id);
CREATE INDEX idx_sr_status ON service_requests(status);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity & Multi-Tenancy | 2 | jurisdictions, departments |
| User & Access Management | 3 | users, user_department_roles, role_permissions |
| Parcel & Address | 2 | parcels, parcel_owners |
| Permit Type Configuration | 3 | permit_types, permit_type_fields, permit_type_required_documents |
| Permits (Core Record) | 2 | permits, permit_status_history |
| Contractors | 2 | contractors, permit_contractors |
| Workflow Engine | 5 | templates, steps, dependencies, instances, step_instances |
| Inspections | 5 | inspection_types, checklists, permit_type_inspections, inspections, checklist_results, attachments |
| Fees & Payments | 5 | fee_schedules, fee_items, permit_fees, payments, payment_fee_allocations |
| Documents & Plan Review | 2 | documents, plan_review_comments |
| Licences | 2 | licence_types, licences |
| Code Enforcement | 2 | code_enforcement_cases, violation_notices |
| Notifications | 1 | notifications |
| Audit | 1 | audit_log |
| Open311 Integration | 1 | service_requests |
| **Total** | **~38** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation across multiple application nodes without coordination, essential for a multi-tenant SaaS platform.

2. **Multi-tenancy via `jurisdiction_id` foreign key** — every tenant-scoped table includes a `jurisdiction_id` column. Combined with PostgreSQL Row Level Security policies, this provides strong data isolation in a shared-schema model.

3. **BLDS field naming alignment** — the `permits`, `inspections`, and `contractors` tables use field names that map directly to BLDS CSV schema requirements, making open data export straightforward.

4. **Configurable form fields via `permit_type_fields` + JSONB `form_data`** — the schema supports jurisdiction-specific intake forms without schema migration. The relational `permit_type_fields` table defines the form structure; the JSONB `form_data` column on `permits` stores actual submissions.

5. **Workflow engine as separate tables** — the template/instance pattern allows jurisdictions to define custom review workflows without code changes. Step dependencies support both sequential and parallel review routing.

6. **Inspection hierarchy with parent/child** — re-inspections link back to the original failed inspection via `parent_inspection_id`, maintaining a clear chain of inspection events.

7. **Fee calculation supports multiple patterns** — from simple flat fees through formula-based calculations referencing project valuation, accommodating the wide variety of municipal fee structures.

8. **Audit log with old/new value capture** — stores the actual field values before and after each change, supporting both compliance auditing and data recovery scenarios.

9. **PostGIS geometry column on parcels** — enables spatial queries (proximity searches, zoning boundary checks) that are essential for parcel-based permitting operations.

10. **AI scoring columns on core entities** — `ai_completeness_score`, `ai_risk_score`, and `ai_scheduling_score` columns are first-class fields rather than external metadata, reflecting the AI-native positioning of the platform.
