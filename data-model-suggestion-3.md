# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Permitting & Licensing Platform · Created: 2026-05-22

## Philosophy

This model uses a hybrid approach: core structural fields that are common across all jurisdictions are stored as typed relational columns with full indexing and constraint enforcement, while jurisdiction-specific, permit-type-specific, and region-varying fields are stored in JSONB columns with PostgreSQL's native JSON operators and GIN indexes. The result is a schema that is both structured enough for cross-jurisdiction analytics and flexible enough to accommodate the enormous variability between municipal permitting requirements without schema migrations.

The design recognises a fundamental reality of government permitting software: while the lifecycle (apply, review, inspect, issue) is universal, the details vary wildly between jurisdictions. A building permit in San Francisco has different required fields, fee formulas, review departments, and inspection checklists than one in rural Alabama. Enterprise incumbents like Tyler EnerGov and Accela handle this through complex configuration engines. This model achieves similar flexibility at the database level by combining relational rigidity where it matters (referential integrity, foreign keys, status tracking) with JSONB flexibility where variability is highest (form data, fee rules, checklist items, jurisdiction config).

This is the pragmatic choice for a startup or open-source project targeting rapid MVP delivery with the ability to onboard diverse jurisdictions without per-tenant schema changes. PostgreSQL's JSONB capabilities — including containment queries (`@>`), path extraction (`->>`, `#>>`), and GIN indexing — make this approach production-viable rather than a compromise.

**Best for:** Multi-jurisdiction SaaS platforms that need to onboard diverse municipalities rapidly without per-tenant schema migrations, and teams prioritising fast MVP delivery with flexible configuration.

**Trade-offs:**
- (+) Rapid onboarding of new jurisdictions — no schema changes needed for jurisdiction-specific fields
- (+) Fewer tables (~25) reduces schema complexity and ORM mapping overhead
- (+) JSONB columns eliminate the need for EAV (entity-attribute-value) anti-patterns
- (+) PostgreSQL GIN indexes on JSONB provide strong query performance
- (+) Form builders and workflow designers store config directly as JSON — no impedance mismatch
- (-) JSONB fields lack database-level type checking — validation must be enforced in application code
- (-) Complex JSONB queries can be harder to optimise than pure relational joins
- (-) Reporting across JSONB fields requires extraction functions, complicating analytics queries
- (-) Schema documentation is split between DDL (relational) and JSON Schema definitions (JSONB)
- (-) JSONB columns can grow large if not managed, impacting row size and TOAST performance

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| BLDS (Building & Land Development Specification) | Core BLDS fields are relational columns on `permits`; BLDS export view extracts both relational and JSONB fields into standard CSV format |
| JSON Schema (Draft 2020-12) | JSONB columns are validated against JSON Schema definitions stored in `schema_registry`; permit type form schemas are JSON Schema documents |
| IBC (International Building Code) | IBC references stored in `inspection_config` JSONB within `inspection_types`; code sections in checklist items |
| Open311 GeoReport v2 | Open311 attributes map to JSONB `attributes` field on service requests |
| NEPA Data Standard v1.2 | Environmental review metadata stored as structured JSONB aligned with NEPA entity schemas |
| ISO 3166-1/2 | Jurisdiction codes as relational columns |
| ISO 8601 | All timestamps as `TIMESTAMPTZ` |
| OpenAPI 3.1 | API exposes JSONB fields with OpenAPI `additionalProperties` documentation |

---

## Schema Registry (JSON Schema Validation)

```sql
-- Stores JSON Schema definitions for validating JSONB columns
CREATE TABLE schema_registry (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID,                  -- NULL = global schema; set = jurisdiction-specific override
    schema_name VARCHAR(200) NOT NULL,     -- e.g., 'permit_form.BLDG_NEW', 'fee_rule.per_sqft', 'inspection_checklist.foundation'
    schema_version INTEGER NOT NULL DEFAULT 1,
    json_schema JSONB NOT NULL,            -- JSON Schema (Draft 2020-12) definition
    -- Example json_schema:
    -- {
    --   "$schema": "https://json-schema.org/draft/2020-12/schema",
    --   "type": "object",
    --   "properties": {
    --     "stories": {"type": "integer", "minimum": 1, "maximum": 200},
    --     "sqft": {"type": "number", "minimum": 1},
    --     "occupancy_group": {"type": "string", "enum": ["A-1","A-2","B","E","F-1","H-1","I-1","M","R-1","R-2","S-1","U"]},
    --     "construction_type": {"type": "string", "enum": ["I-A","I-B","II-A","II-B","III-A","III-B","IV","V-A","V-B"]}
    --   },
    --   "required": ["stories", "sqft"]
    -- }
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, schema_name, schema_version)
);

CREATE INDEX idx_sr_name ON schema_registry(schema_name);
CREATE INDEX idx_sr_jurisdiction ON schema_registry(jurisdiction_id);
```

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE jurisdictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    jurisdiction_type VARCHAR(50) NOT NULL CHECK (jurisdiction_type IN ('city', 'county', 'state', 'federal', 'special_district')),
    iso_3166_code VARCHAR(10),
    parent_jurisdiction_id UUID REFERENCES jurisdictions(id),
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    
    -- Jurisdiction-wide configuration stored as JSONB
    config JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "branding": {"logo_url": "...", "primary_color": "#003366"},
    --   "sla_defaults": {"residential_days": 15, "commercial_days": 30},
    --   "payment_processor": "stripe",
    --   "payment_config": {"publishable_key": "pk_live_..."},
    --   "notification_channels": ["email", "sms"],
    --   "gis_provider": "esri",
    --   "gis_config": {"service_url": "https://...", "api_key": "..."},
    --   "open_data_enabled": true,
    --   "blds_export_schedule": "daily"
    -- }
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jurisdictions_type ON jurisdictions(jurisdiction_type);

CREATE TABLE departments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) NOT NULL,
    department_type VARCHAR(50) NOT NULL,
    config JSONB DEFAULT '{}',            -- Department-specific settings
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, code)
);
```

---

## Users & RBAC

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    user_type VARCHAR(50) NOT NULL CHECK (user_type IN ('staff', 'applicant', 'contractor', 'inspector', 'admin')),
    identity_provider VARCHAR(50) DEFAULT 'local',
    identity_provider_id VARCHAR(255),
    
    -- User profile and preferences as JSONB (varies by user type)
    profile JSONB DEFAULT '{}',
    -- Staff profile example:
    -- {
    --   "employee_id": "EMP-1234",
    --   "certifications": ["ICC Certified Plans Examiner", "ICC Certified Building Official"],
    --   "inspection_specialties": ["structural", "electrical"],
    --   "max_daily_inspections": 8
    -- }
    -- Contractor profile example:
    -- {
    --   "business_name": "Acme Construction",
    --   "licence_number": "CSLB-987654",
    --   "licence_state": "CA",
    --   "licence_expiry": "2027-03-15",
    --   "insurance_policy": "INS-456789",
    --   "insurance_expiry": "2026-12-31",
    --   "bonding_amount": 25000.00,
    --   "trades": ["general", "electrical"]
    -- }
    
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_type ON users(user_type);
CREATE INDEX idx_users_profile_gin ON users USING GIN(profile);

-- Roles and permissions (simplified — role definitions include permission sets as JSONB)
CREATE TABLE user_roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    department_id UUID REFERENCES departments(id),
    role VARCHAR(50) NOT NULL,
    
    permissions JSONB NOT NULL DEFAULT '[]',
    -- Example: ["permits:read", "permits:create", "inspections:read", "inspections:schedule", "fees:read"]
    
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by UUID REFERENCES users(id),
    revoked_at TIMESTAMPTZ,
    UNIQUE(user_id, jurisdiction_id, department_id, role)
);

CREATE INDEX idx_ur_user ON user_roles(user_id);
CREATE INDEX idx_ur_jurisdiction ON user_roles(jurisdiction_id);
```

---

## Parcels

```sql
CREATE TABLE parcels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    parcel_number VARCHAR(100) NOT NULL,
    address_line1 VARCHAR(255),
    city VARCHAR(100),
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    country_code VARCHAR(2) DEFAULT 'US',
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    geom GEOMETRY(Polygon, 4326),
    
    -- Parcel attributes vary significantly by jurisdiction
    attributes JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "zoning_code": "R-1",
    --   "zoning_description": "Single-Family Residential",
    --   "lot_size_sqft": 7500,
    --   "land_use": "residential",
    --   "flood_zone": "X",
    --   "fire_hazard_zone": "non-VHFHSZ",
    --   "historic_district": false,
    --   "owner": {
    --     "name": "Jane Smith",
    --     "type": "individual",
    --     "mailing_address": "123 Main St, Springfield, IL 62701"
    --   },
    --   "assessed_value": 350000,
    --   "year_built": 1985,
    --   "existing_structures": [{"type": "single_family", "sqft": 1800, "stories": 1}]
    -- }
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, parcel_number)
);

CREATE INDEX idx_parcels_jurisdiction ON parcels(jurisdiction_id);
CREATE INDEX idx_parcels_number ON parcels(parcel_number);
CREATE INDEX idx_parcels_geom ON parcels USING GIST(geom);
CREATE INDEX idx_parcels_attributes ON parcels USING GIN(attributes);
```

---

## Permit Types (Configuration-Driven)

```sql
CREATE TABLE permit_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    category VARCHAR(100) NOT NULL,
    
    -- The entire form definition, fee rules, and workflow config as JSONB
    form_schema JSONB NOT NULL DEFAULT '{}',
    -- JSON Schema defining the intake form fields:
    -- {
    --   "$schema": "https://json-schema.org/draft/2020-12/schema",
    --   "type": "object",
    --   "properties": {
    --     "project_description": {"type": "string", "title": "Project Description", "x-display-order": 1},
    --     "stories": {"type": "integer", "minimum": 1, "title": "Number of Stories", "x-display-order": 2},
    --     "sqft": {"type": "number", "minimum": 1, "title": "Square Footage", "x-display-order": 3},
    --     "occupancy_group": {
    --       "type": "string",
    --       "title": "IBC Occupancy Group",
    --       "enum": ["A-1","A-2","A-3","A-4","A-5","B","E","F-1","F-2","H-1","H-2","H-3","H-4","H-5","I-1","I-2","I-3","I-4","M","R-1","R-2","R-3","R-4","S-1","S-2","U"],
    --       "x-display-order": 4
    --     },
    --     "has_sprinklers": {"type": "boolean", "title": "Fire Sprinkler System", "x-display-order": 5}
    --   },
    --   "required": ["project_description", "sqft"],
    --   "x-conditional-fields": [
    --     {"if": {"occupancy_group": {"const": "A-1"}}, "then": {"required": ["has_sprinklers"]}}
    --   ]
    -- }
    
    fee_rules JSONB NOT NULL DEFAULT '[]',
    -- Fee calculation rules:
    -- [
    --   {
    --     "code": "BLDG_PERMIT",
    --     "name": "Building Permit Fee",
    --     "type": "tiered",
    --     "tiers": [
    --       {"up_to": 50000, "flat": 200},
    --       {"up_to": 100000, "rate": 0.005, "base": 200},
    --       {"up_to": null, "rate": 0.003, "base": 450}
    --     ],
    --     "basis": "estimated_cost",
    --     "is_refundable": false
    --   },
    --   {
    --     "code": "PLAN_REVIEW",
    --     "name": "Plan Review Fee",
    --     "type": "percentage",
    --     "percentage": 65,
    --     "of_fee": "BLDG_PERMIT",
    --     "is_refundable": true
    --   },
    --   {
    --     "code": "TECH_FEE",
    --     "name": "Technology Surcharge",
    --     "type": "flat",
    --     "amount": 25.00,
    --     "is_refundable": false
    --   }
    -- ]
    
    workflow_config JSONB NOT NULL DEFAULT '{}',
    -- Workflow definition:
    -- {
    --   "steps": [
    --     {"order": 1, "name": "Zoning Review", "department": "PLAN", "type": "review", "sla_days": 5},
    --     {"order": 2, "name": "Building Plan Review", "department": "BLDG", "type": "review", "sla_days": 10, "parallel_with": [3]},
    --     {"order": 3, "name": "Fire Review", "department": "FIRE", "type": "review", "sla_days": 7, "parallel_with": [2]},
    --     {"order": 4, "name": "Fee Payment", "type": "payment"},
    --     {"order": 5, "name": "Permit Issuance", "department": "BLDG", "type": "approval", "sla_days": 2}
    --   ]
    -- }
    
    required_documents JSONB DEFAULT '[]',
    -- [
    --   {"type": "site_plan", "label": "Site Plan", "required": true, "formats": ["pdf", "dwg"]},
    --   {"type": "floor_plan", "label": "Floor Plans", "required": true, "formats": ["pdf"]},
    --   {"type": "structural_calcs", "label": "Structural Calculations", "required": false, "formats": ["pdf"]},
    --   {"type": "proof_of_ownership", "label": "Proof of Property Ownership", "required": true, "formats": ["pdf", "jpg", "png"]}
    -- ]
    
    inspection_sequence JSONB DEFAULT '[]',
    -- [
    --   {"order": 1, "type_code": "FOUNDATION", "name": "Foundation", "required": true},
    --   {"order": 2, "type_code": "FRAMING", "name": "Framing", "required": true},
    --   {"order": 3, "type_code": "ELECTRICAL_ROUGH", "name": "Rough Electrical", "required": true},
    --   {"order": 4, "type_code": "PLUMBING_ROUGH", "name": "Rough Plumbing", "required": true},
    --   {"order": 5, "type_code": "INSULATION", "name": "Insulation", "required": true},
    --   {"order": 6, "type_code": "DRYWALL", "name": "Drywall", "required": false},
    --   {"order": 7, "type_code": "FINAL", "name": "Final Inspection", "required": true}
    -- ]
    
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, code)
);

CREATE INDEX idx_pt_jurisdiction ON permit_types(jurisdiction_id);
CREATE INDEX idx_pt_category ON permit_types(category);
```

---

## Permits (Core Record)

```sql
CREATE TABLE permits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    permit_type_id UUID NOT NULL REFERENCES permit_types(id),
    
    -- BLDS-aligned relational fields (common across all jurisdictions)
    permit_num VARCHAR(100) NOT NULL,
    description TEXT,
    applied_date DATE,
    issued_date DATE,
    completed_date DATE,
    expires_date DATE,
    status VARCHAR(50) NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'submitted', 'under_review', 'revisions_requested',
        'approved', 'issued', 'inspection_scheduled', 'inspection_complete',
        'completed', 'expired', 'denied', 'withdrawn', 'suspended', 'revoked'
    )),
    status_changed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    permit_class VARCHAR(50),
    work_type VARCHAR(50),
    parcel_id UUID REFERENCES parcels(id),
    site_address TEXT,
    estimated_cost NUMERIC(14, 2),
    applicant_id UUID REFERENCES users(id),
    
    -- Jurisdiction-specific form data (validated against permit_type.form_schema)
    form_data JSONB NOT NULL DEFAULT '{}',
    -- Example for a residential building permit in Springfield, IL:
    -- {
    --   "project_description": "New 2-story single-family residence with attached garage",
    --   "stories": 2,
    --   "sqft": 2400,
    --   "occupancy_group": "R-3",
    --   "construction_type": "V-B",
    --   "has_sprinklers": false,
    --   "bedrooms": 4,
    --   "bathrooms": 2.5,
    --   "garage_sqft": 480,
    --   "foundation_type": "slab",
    --   "heating_type": "forced_air_gas",
    --   "sewer_connection": "public"
    -- }
    
    -- Workflow state (current position in the review pipeline)
    workflow_state JSONB DEFAULT '{}',
    -- {
    --   "current_step": 2,
    --   "steps": [
    --     {"order": 1, "name": "Zoning Review", "status": "completed", "reviewer": "uuid", "completed_at": "2026-05-20T16:00:00Z", "decision": "approved"},
    --     {"order": 2, "name": "Building Plan Review", "status": "in_progress", "reviewer": "uuid", "started_at": "2026-05-20T16:05:00Z", "sla_date": "2026-05-30"},
    --     {"order": 3, "name": "Fire Review", "status": "in_progress", "reviewer": "uuid", "started_at": "2026-05-20T16:05:00Z", "sla_date": "2026-05-27"},
    --     {"order": 4, "name": "Fee Payment", "status": "pending"},
    --     {"order": 5, "name": "Permit Issuance", "status": "pending"}
    --   ]
    -- }
    
    -- Assessed fees (calculated from permit_type.fee_rules)
    fees JSONB DEFAULT '[]',
    -- [
    --   {"code": "BLDG_PERMIT", "description": "Building Permit Fee", "amount": 3500.00, "status": "paid", "paid_date": "2026-05-18", "receipt": "RCP-2026-00891"},
    --   {"code": "PLAN_REVIEW", "description": "Plan Review Fee", "amount": 2275.00, "status": "paid", "paid_date": "2026-05-18", "receipt": "RCP-2026-00891"},
    --   {"code": "TECH_FEE", "description": "Technology Surcharge", "amount": 25.00, "status": "paid", "paid_date": "2026-05-18", "receipt": "RCP-2026-00891"}
    -- ]
    
    -- Conditions of approval
    conditions JSONB DEFAULT '[]',
    -- [
    --   {"code": "IFC-903.2", "description": "Install fire sprinklers per IFC 903.2", "status": "pending"},
    --   {"code": "IBC-1612", "description": "Submit revised flood elevation certificate", "status": "satisfied", "satisfied_at": "2026-06-01"}
    -- ]
    
    -- AI analysis results
    ai_analysis JSONB DEFAULT '{}',
    -- {
    --   "completeness_score": 94.5,
    --   "completeness_checked_at": "2026-05-15T10:30:00Z",
    --   "missing_documents": ["structural_calculations"],
    --   "risk_score": 12.3,
    --   "risk_factors": ["high_value_project", "first_time_contractor"],
    --   "predicted_days_to_issue": 18,
    --   "similar_permits": ["BLD-2025-00891", "BLD-2025-01023"]
    -- }
    
    -- Contractor info (denormalised for query convenience)
    contractor JSONB,
    -- {
    --   "user_id": "uuid",
    --   "business_name": "Acme Construction LLC",
    --   "licence_number": "CSLB-987654",
    --   "licence_state": "IL",
    --   "phone": "555-0200",
    --   "email": "permits@acme.com"
    -- }
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, permit_num)
);

CREATE INDEX idx_permits_jurisdiction ON permits(jurisdiction_id);
CREATE INDEX idx_permits_type ON permits(permit_type_id);
CREATE INDEX idx_permits_status ON permits(status);
CREATE INDEX idx_permits_parcel ON permits(parcel_id);
CREATE INDEX idx_permits_applicant ON permits(applicant_id);
CREATE INDEX idx_permits_applied ON permits(applied_date);
CREATE INDEX idx_permits_issued ON permits(issued_date);
CREATE INDEX idx_permits_num ON permits(permit_num);
CREATE INDEX idx_permits_form_data ON permits USING GIN(form_data);
CREATE INDEX idx_permits_workflow ON permits USING GIN(workflow_state);
CREATE INDEX idx_permits_fees ON permits USING GIN(fees);
CREATE INDEX idx_permits_ai ON permits USING GIN(ai_analysis);
```

---

## Inspections

```sql
CREATE TABLE inspection_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    
    -- Inspection configuration as JSONB (checklists, code references, duration)
    config JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "ibc_code_section": "IBC 110.3.1",
    --   "typical_duration_minutes": 45,
    --   "requires_re_inspection_on_fail": true,
    --   "checklist": [
    --     {"item": "Footing depth per approved plans", "ibc_ref": "IBC 1809.4", "is_critical": true},
    --     {"item": "Rebar placement and spacing", "ibc_ref": "IBC 1810.3.9.1", "is_critical": true},
    --     {"item": "Form alignment and bracing", "is_critical": false},
    --     {"item": "Anchor bolt placement", "ibc_ref": "IBC 1911.1", "is_critical": true},
    --     {"item": "Soil condition adequate", "is_critical": false},
    --     {"item": "Drainage provisions", "ibc_ref": "IBC 1805.4", "is_critical": false}
    --   ]
    -- }
    
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, code)
);

CREATE INDEX idx_it_jurisdiction ON inspection_types(jurisdiction_id);

CREATE TABLE inspections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_id UUID NOT NULL REFERENCES permits(id),
    inspection_type_id UUID NOT NULL REFERENCES inspection_types(id),
    parent_inspection_id UUID REFERENCES inspections(id),
    inspector_id UUID REFERENCES users(id),
    
    -- Core relational fields
    scheduled_date DATE,
    scheduled_time_start TIME,
    scheduled_time_end TIME,
    status VARCHAR(50) NOT NULL DEFAULT 'requested' CHECK (status IN (
        'requested', 'scheduled', 'in_progress', 'passed', 'failed',
        'partial', 'cancelled', 'no_access', 'rescheduled'
    )),
    result VARCHAR(50),
    completed_at TIMESTAMPTZ,
    
    -- Inspection details as JSONB (results, photos, location, checklist outcomes)
    details JSONB DEFAULT '{}',
    -- {
    --   "checklist_results": [
    --     {"item": "Footing depth per approved plans", "result": "pass", "notes": "36 inches confirmed"},
    --     {"item": "Rebar placement and spacing", "result": "pass"},
    --     {"item": "Form alignment and bracing", "result": "pass"},
    --     {"item": "Anchor bolt placement", "result": "fail", "notes": "Spacing exceeds 6 feet in NW corner. IBC 1911.1 requires max 6ft spacing."},
    --     {"item": "Soil condition adequate", "result": "pass"},
    --     {"item": "Drainage provisions", "result": "na", "notes": "Not applicable — slab on grade"}
    --   ],
    --   "photos": [
    --     {"file_id": "uuid", "caption": "Foundation NE corner", "latitude": 39.7817, "longitude": -89.6501},
    --     {"file_id": "uuid", "caption": "Anchor bolt issue NW corner"}
    --   ],
    --   "notes": "Anchor bolt spacing issue must be corrected before framing inspection",
    --   "location": {"latitude": 39.7817, "longitude": -89.6501, "accuracy_meters": 5},
    --   "weather": {"condition": "clear", "temp_f": 72},
    --   "duration_minutes": 40,
    --   "ai_scheduling_score": 87.3
    -- }
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_insp_permit ON inspections(permit_id);
CREATE INDEX idx_insp_inspector ON inspections(inspector_id);
CREATE INDEX idx_insp_status ON inspections(status);
CREATE INDEX idx_insp_scheduled ON inspections(scheduled_date);
CREATE INDEX idx_insp_type ON inspections(inspection_type_id);
CREATE INDEX idx_insp_details ON inspections USING GIN(details);
```

---

## Documents

```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_id UUID REFERENCES permits(id),
    inspection_id UUID REFERENCES inspections(id),
    licence_id UUID REFERENCES licences(id),
    enforcement_case_id UUID REFERENCES code_enforcement_cases(id),
    
    document_type VARCHAR(100) NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    storage_path TEXT NOT NULL,
    file_type VARCHAR(50),
    file_size_bytes BIGINT,
    version INTEGER NOT NULL DEFAULT 1,
    is_current_version BOOLEAN DEFAULT true,
    previous_version_id UUID REFERENCES documents(id),
    uploaded_by UUID REFERENCES users(id),
    
    -- Document metadata and AI analysis as JSONB
    metadata JSONB DEFAULT '{}',
    -- {
    --   "ai_classification": "site_plan",
    --   "ai_confidence": 0.97,
    --   "ai_completeness": "complete",
    --   "ai_analysis_notes": "Site plan includes all required elements: property lines, setbacks, building footprint, driveway, utilities",
    --   "page_count": 3,
    --   "review_comments": [
    --     {"reviewer_id": "uuid", "cycle": 1, "type": "correction_required", "code_ref": "IBC 1005.1", "text": "Egress width insufficient", "page": 3, "status": "resolved"}
    --   ]
    -- }
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_docs_permit ON documents(permit_id);
CREATE INDEX idx_docs_inspection ON documents(inspection_id);
CREATE INDEX idx_docs_type ON documents(document_type);
CREATE INDEX idx_docs_current ON documents(is_current_version) WHERE is_current_version = true;
CREATE INDEX idx_docs_metadata ON documents USING GIN(metadata);
```

---

## Licences

```sql
CREATE TABLE licence_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    category VARCHAR(100) NOT NULL,
    
    -- Licence configuration as JSONB
    config JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "renewal_period_months": 12,
    --   "requires_inspection": true,
    --   "renewal_grace_period_days": 30,
    --   "form_schema": { ... JSON Schema for licence application fields ... },
    --   "fee_rules": [
    --     {"code": "BUS_LIC_FEE", "name": "Business Licence Fee", "type": "flat", "amount": 150.00},
    --     {"code": "BUS_LIC_LATE", "name": "Late Renewal Penalty", "type": "percentage", "percentage": 25, "of_fee": "BUS_LIC_FEE", "applies_when": "renewal_after_expiry"}
    --   ],
    --   "required_documents": ["proof_of_insurance", "tax_id_certificate"]
    -- }
    
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, code)
);

CREATE TABLE licences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    licence_type_id UUID NOT NULL REFERENCES licence_types(id),
    licence_number VARCHAR(100) NOT NULL,
    
    -- Core relational fields
    holder_name VARCHAR(255) NOT NULL,
    holder_user_id UUID REFERENCES users(id),
    status VARCHAR(50) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'active', 'suspended', 'revoked', 'expired', 'denied', 'renewal_pending'
    )),
    issued_date DATE,
    effective_date DATE,
    expiry_date DATE,
    
    -- Licence-specific data as JSONB
    details JSONB DEFAULT '{}',
    -- {
    --   "business_name": "Joe's Pizza",
    --   "business_address": "789 Elm St, Springfield, IL 62702",
    --   "business_structure": "sole_proprietorship",
    --   "naics_code": "722511",
    --   "employee_count": 12,
    --   "operating_hours": "11am-10pm Mon-Sat, 12pm-8pm Sun",
    --   "seating_capacity": 45,
    --   "liquor_licence": false,
    --   "health_permit_number": "HP-2026-00456",
    --   "renewal_history": [
    --     {"date": "2025-07-01", "amount": 150.00, "receipt": "RCP-2025-04521"},
    --     {"date": "2024-07-01", "amount": 150.00, "receipt": "RCP-2024-03892"}
    --   ],
    --   "conditions": ["Maintain current food handler certificates for all staff"],
    --   "parcel_id": "uuid"
    -- }
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, licence_number)
);

CREATE INDEX idx_lic_jurisdiction ON licences(jurisdiction_id);
CREATE INDEX idx_lic_type ON licences(licence_type_id);
CREATE INDEX idx_lic_status ON licences(status);
CREATE INDEX idx_lic_expiry ON licences(expiry_date);
CREATE INDEX idx_lic_holder ON licences(holder_user_id);
CREATE INDEX idx_lic_details ON licences USING GIN(details);
```

---

## Code Enforcement

```sql
CREATE TABLE code_enforcement_cases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    case_number VARCHAR(100) NOT NULL,
    parcel_id UUID REFERENCES parcels(id),
    site_address TEXT,
    assigned_officer_id UUID REFERENCES users(id),
    
    -- Core relational fields
    source VARCHAR(50),
    violation_type VARCHAR(100),
    status VARCHAR(50) NOT NULL DEFAULT 'open' CHECK (status IN (
        'open', 'investigating', 'notice_issued', 'compliance_pending',
        'resolved', 'closed', 'escalated', 'hearing_scheduled'
    )),
    priority VARCHAR(20) DEFAULT 'normal',
    opened_date DATE NOT NULL DEFAULT CURRENT_DATE,
    resolved_date DATE,
    
    -- Case details, violations, notices, and timeline as JSONB
    details JSONB DEFAULT '{}',
    -- {
    --   "description": "Unpermitted deck construction observed at rear of property",
    --   "reporter": {"type": "anonymous", "method": "online_form"},
    --   "related_permit_id": null,
    --   "related_licence_id": null,
    --   "violations": [
    --     {"code": "IBC 105.1", "description": "Work commenced without permit", "severity": "major"},
    --     {"code": "IBC 105.2", "description": "Building permit required for deck > 200 sqft", "severity": "major"}
    --   ],
    --   "notices": [
    --     {
    --       "number": "VN-2026-00198",
    --       "type": "notice_of_violation",
    --       "issued_date": "2026-06-01",
    --       "served_method": "mail",
    --       "compliance_deadline": "2026-07-01",
    --       "fine_amount": 500.00
    --     }
    --   ],
    --   "timeline": [
    --     {"date": "2026-05-28", "action": "Complaint received", "by": "system"},
    --     {"date": "2026-05-30", "action": "Site visit conducted", "by": "uuid", "notes": "Confirmed unpermitted 14x20 deck"},
    --     {"date": "2026-06-01", "action": "NOV issued", "by": "uuid"}
    --   ]
    -- }
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, case_number)
);

CREATE INDEX idx_ce_jurisdiction ON code_enforcement_cases(jurisdiction_id);
CREATE INDEX idx_ce_status ON code_enforcement_cases(status);
CREATE INDEX idx_ce_parcel ON code_enforcement_cases(parcel_id);
CREATE INDEX idx_ce_officer ON code_enforcement_cases(assigned_officer_id);
CREATE INDEX idx_ce_details ON code_enforcement_cases USING GIN(details);
```

---

## Payments

```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    permit_id UUID REFERENCES permits(id),
    licence_id UUID REFERENCES licences(id),
    enforcement_case_id UUID REFERENCES code_enforcement_cases(id),
    payer_id UUID REFERENCES users(id),
    
    total_amount NUMERIC(12, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    status VARCHAR(50) NOT NULL CHECK (status IN ('pending', 'completed', 'failed', 'refunded', 'partially_refunded')),
    
    -- Payment details as JSONB
    details JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "method": "credit_card",
    --   "processor": "stripe",
    --   "processor_transaction_id": "pi_3PxQr2ABC123",
    --   "receipt_number": "RCP-2026-00891",
    --   "card_last_four": "4242",
    --   "fees_covered": [
    --     {"code": "BLDG_PERMIT", "amount": 3500.00},
    --     {"code": "PLAN_REVIEW", "amount": 2275.00},
    --     {"code": "TECH_FEE", "amount": 25.00}
    --   ],
    --   "refunds": []
    -- }
    
    paid_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pay_jurisdiction ON payments(jurisdiction_id);
CREATE INDEX idx_pay_permit ON payments(permit_id);
CREATE INDEX idx_pay_licence ON payments(licence_id);
CREATE INDEX idx_pay_status ON payments(status);
CREATE INDEX idx_pay_details ON payments USING GIN(details);
```

---

## Notifications & Audit

```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    recipient_id UUID REFERENCES users(id),
    channel VARCHAR(20) NOT NULL CHECK (channel IN ('email', 'sms', 'in_app', 'push')),
    notification_type VARCHAR(100) NOT NULL,
    
    content JSONB NOT NULL,
    -- {
    --   "subject": "Permit BLD-2026-00142 Status Update",
    --   "body": "Your building permit application has been approved...",
    --   "recipient_email": "jane@example.com",
    --   "recipient_phone": "+15550100",
    --   "related_entity": {"type": "permit", "id": "uuid", "num": "BLD-2026-00142"},
    --   "template": "permit_status_change",
    --   "template_vars": {"permit_num": "BLD-2026-00142", "new_status": "approved"}
    -- }
    
    status VARCHAR(50) DEFAULT 'pending',
    sent_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notif_recipient ON notifications(recipient_id);
CREATE INDEX idx_notif_status ON notifications(status);

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    user_id UUID REFERENCES users(id),
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(100) NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB,                         -- {"field": {"old": "value", "new": "value"}}
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Example Queries

### Query permit form data with JSONB operators

```sql
-- Find all residential permits with more than 2 stories in a jurisdiction
SELECT permit_num, description, form_data->>'stories' AS stories, estimated_cost
FROM permits
WHERE jurisdiction_id = 'jurisdiction-uuid'
  AND permit_class = 'residential'
  AND (form_data->>'stories')::int > 2
  AND status IN ('submitted', 'under_review', 'approved', 'issued');
```

### Query permits by fee status using JSONB containment

```sql
-- Find permits with outstanding (unpaid) fees
SELECT p.permit_num, p.status, f.value->>'code' AS fee_code,
       (f.value->>'amount')::numeric AS amount
FROM permits p,
     jsonb_array_elements(p.fees) AS f(value)
WHERE p.jurisdiction_id = 'jurisdiction-uuid'
  AND f.value->>'status' = 'pending';
```

### Query inspection checklist failures

```sql
-- Find all failed checklist items across inspections for a permit
SELECT i.id, it.name AS inspection_type, ci.value->>'item' AS checklist_item,
       ci.value->>'notes' AS failure_notes, ci.value->>'ibc_ref' AS code_reference
FROM inspections i
JOIN inspection_types it ON i.inspection_type_id = it.id,
     jsonb_array_elements(i.details->'checklist_results') AS ci(value)
WHERE i.permit_id = 'permit-uuid'
  AND ci.value->>'result' = 'fail';
```

### BLDS export query

```sql
-- Generate BLDS-compliant permits.csv data
SELECT permit_num AS "PermitNum",
       description AS "Description",
       applied_date AS "AppliedDate",
       issued_date AS "IssuedDate",
       completed_date AS "CompletedDate",
       status AS "StatusCurrent",
       site_address AS "OriginalAddress1",
       permit_class AS "PermitClass",
       work_type AS "WorkClass",
       estimated_cost AS "EstProjectCost",
       contractor->>'business_name' AS "ContractorCompanyName",
       contractor->>'licence_number' AS "ContractorLicNum"
FROM permits
WHERE jurisdiction_id = 'jurisdiction-uuid'
  AND status NOT IN ('draft', 'withdrawn');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Schema Registry | 1 | JSON Schema definitions for JSONB validation |
| Core Identity & Multi-Tenancy | 2 | jurisdictions, departments |
| Users & RBAC | 2 | users, user_roles |
| Parcels | 1 | parcels (owner info in JSONB attributes) |
| Permit Type Configuration | 1 | permit_types (form, fee, workflow, docs, inspections all in JSONB) |
| Permits | 1 | permits (fees, workflow state, conditions, AI analysis, contractor in JSONB) |
| Inspections | 2 | inspection_types, inspections (checklist results in JSONB) |
| Documents | 1 | documents (review comments in JSONB metadata) |
| Licences | 2 | licence_types, licences (details in JSONB) |
| Code Enforcement | 1 | code_enforcement_cases (violations, notices, timeline in JSONB) |
| Payments | 1 | payments (details in JSONB) |
| Notifications & Audit | 2 | notifications, audit_log |
| **Total** | **~17** | Compared to ~38 in the normalized model |

---

## Key Design Decisions

1. **JSONB for all jurisdiction-variable data** — instead of separate tables for form fields, fee items, workflow steps, checklist items, and violation notices, these are stored as JSONB arrays/objects within their parent entity. This reduces table count from ~38 to ~17 and eliminates per-jurisdiction schema changes.

2. **JSON Schema validation via `schema_registry`** — JSONB flexibility does not mean schema-less. Every JSONB column that accepts user input references a JSON Schema definition that the application layer validates against. This provides the same data quality guarantees as CHECK constraints, enforced in application code.

3. **GIN indexes on all JSONB columns** — PostgreSQL GIN indexes support efficient containment (`@>`), existence (`?`), and path queries on JSONB. Every JSONB column with query patterns has a GIN index.

4. **Relational columns for universal fields, JSONB for variable fields** — `permit_num`, `status`, `applied_date`, `issued_date`, and `estimated_cost` are relational columns because they exist on every permit and are heavily queried/indexed. `form_data`, `fees`, `workflow_state` are JSONB because their structure varies by permit type and jurisdiction.

5. **Workflow state embedded in permit record** — instead of separate workflow instance and step tables, the current workflow state is a JSONB document on the permit. This simplifies queries ("show me the current step") but means workflow history requires the audit log for reconstruction.

6. **Fees as JSONB array on permit** — assessed fees are stored directly on the permit record. This avoids multi-table joins for the common query "show me all fees for this permit" but means cross-permit fee analytics requires `jsonb_array_elements()` extraction.

7. **Contractor info denormalised on permit** — contractor details are embedded in the permit's `contractor` JSONB field rather than requiring a join to a separate contractors table. The `users` table with `user_type = 'contractor'` and the `profile` JSONB field stores the canonical contractor record.

8. **Inspection checklist results in JSONB** — checklist results are stored as a JSONB array in `inspections.details` rather than as separate rows in a junction table. This keeps the inspection record self-contained and simplifies the mobile inspector app data model.

9. **Documents table serves all entity types** — a single `documents` table with nullable foreign keys to `permits`, `inspections`, `licences`, and `code_enforcement_cases` avoids duplicate document storage tables. Review comments are stored in the document's `metadata` JSONB field.

10. **Permit type as a configuration document** — `permit_types` stores form schema, fee rules, workflow config, required documents, and inspection sequence as JSONB. This means a jurisdiction admin can configure a new permit type entirely through a JSON editor or admin UI without any database schema change.
