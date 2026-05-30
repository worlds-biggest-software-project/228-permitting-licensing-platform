# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Permitting & Licensing Platform · Created: 2026-05-22

## Philosophy

This model layers a property graph structure on top of relational tables to capture the dense, interconnected relationships that pervade the permitting and licensing domain. Permits connect to parcels, parcels to owners, owners to contractors, contractors to licences, licences to inspections, inspections to violations, violations to enforcement cases — and all of these entities intersect across time and jurisdiction. A pure relational model handles this with junction tables and multi-hop joins; a graph-relational hybrid makes these relationships first-class, traversable entities.

The approach uses PostgreSQL as the single database with two complementary layers: (1) **relational tables** for operational CRUD — creating permits, scheduling inspections, processing payments — where transactional integrity and simple queries dominate; and (2) **a graph layer** implemented via `graph_nodes` and `graph_edges` tables that mirror the relationships between entities, enabling efficient traversal queries. The graph layer can alternatively be powered by Apache AGE (a PostgreSQL extension providing openCypher graph queries) or by a standalone graph database like Neo4j for organisations with graph DB expertise.

This is the right model when the platform's value proposition includes relationship analysis: conflict-of-interest detection (does an inspector have a financial relationship with the contractor?), ownership chain analysis (who ultimately owns the entity that owns the parcel?), contractor network scoring (what is this contractor's history across all jurisdictions?), and cross-jurisdiction compliance tracing (has this contractor had violations in adjacent cities?).

**Best for:** Platforms emphasising relationship analysis, contractor network scoring, cross-jurisdiction compliance, conflict-of-interest detection, and ownership chain traversal across densely connected government entities.

**Trade-offs:**
- (+) Relationship traversal queries (N-hop) are dramatically faster than multi-join SQL
- (+) Natural model for ownership chains, contractor networks, and conflict-of-interest graphs
- (+) Cross-jurisdiction queries (contractor history across cities) are first-class operations
- (+) Graph visualisation of entity relationships provides powerful investigative UX
- (+) AI/ML features (network scoring, anomaly detection) benefit from graph structure
- (-) Dual-layer architecture (relational + graph) increases operational complexity
- (-) Graph data must be kept in sync with relational tables (eventual consistency risk)
- (-) Development team needs graph query expertise (Cypher or recursive SQL)
- (-) Graph indexes and storage add overhead for simple CRUD operations
- (-) Fewer developers are familiar with graph data modelling compared to pure relational

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| BLDS (Building & Land Development Specification) | Relational `permits`, `inspections` tables align with BLDS field names for open data export |
| IBC (International Building Code) | Inspection types and checklist items reference IBC code sections; stored as relational data |
| Open311 GeoReport v2 | Service requests modelled as graph nodes connected to parcels and enforcement cases |
| ISO 3166-1/2 | Jurisdiction nodes in graph layer use ISO 3166 codes; relational `jurisdictions` table uses same |
| ISO 17442 (LEI) | Where applicable, business entities can be tagged with Legal Entity Identifiers for cross-system matching |
| NEPA Data Standard v1.2 | Environmental review entities modelled as nodes connected to permit and project nodes |
| OpenCypher / GQL | Graph queries use openCypher syntax (via Apache AGE) or standard SQL recursive CTEs |
| IFC (ISO 16739) | BIM submission documents linked as nodes to permit and parcel nodes |

---

## Relational Layer (Operational CRUD)

The relational layer handles day-to-day operations. It is structurally similar to Data Model Suggestion 1 (normalized relational) but with fewer junction tables — relationships that would normally require junction tables are instead captured in the graph layer.

### Core Tables

```sql
CREATE TABLE jurisdictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    jurisdiction_type VARCHAR(50) NOT NULL CHECK (jurisdiction_type IN ('city', 'county', 'state', 'federal', 'special_district')),
    iso_3166_code VARCHAR(10),
    parent_jurisdiction_id UUID REFERENCES jurisdictions(id),
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    config JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE departments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) NOT NULL,
    department_type VARCHAR(50) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, code)
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    user_type VARCHAR(50) NOT NULL CHECK (user_type IN ('staff', 'applicant', 'contractor', 'inspector', 'admin')),
    identity_provider VARCHAR(50) DEFAULT 'local',
    identity_provider_id VARCHAR(255),
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_type ON users(user_type);

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
    zoning_code VARCHAR(50),
    zoning_description VARCHAR(255),
    lot_size_sqft NUMERIC(12, 2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, parcel_number)
);

CREATE INDEX idx_parcels_jurisdiction ON parcels(jurisdiction_id);
CREATE INDEX idx_parcels_number ON parcels(parcel_number);
CREATE INDEX idx_parcels_geom ON parcels USING GIST(geom);
```

### Permits

```sql
CREATE TABLE permit_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    category VARCHAR(100) NOT NULL,
    ibc_occupancy_group VARCHAR(10),
    requires_plan_review BOOLEAN DEFAULT false,
    requires_inspection BOOLEAN DEFAULT true,
    form_config JSONB DEFAULT '{}',
    fee_config JSONB DEFAULT '[]',
    workflow_config JSONB DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, code)
);

CREATE TABLE permits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    permit_type_id UUID NOT NULL REFERENCES permit_types(id),
    permit_num VARCHAR(100) NOT NULL,
    description TEXT,
    applied_date DATE,
    issued_date DATE,
    completed_date DATE,
    expires_date DATE,
    status VARCHAR(50) NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'submitted', 'under_review', 'revisions_requested',
        'approved', 'issued', 'inspection_complete',
        'completed', 'expired', 'denied', 'withdrawn', 'suspended', 'revoked'
    )),
    status_changed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    permit_class VARCHAR(50),
    work_type VARCHAR(50),
    parcel_id UUID REFERENCES parcels(id),
    site_address TEXT,
    estimated_cost NUMERIC(14, 2),
    applicant_id UUID REFERENCES users(id),
    form_data JSONB DEFAULT '{}',
    ai_completeness_score NUMERIC(5, 2),
    ai_risk_score NUMERIC(5, 2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, permit_num)
);

CREATE INDEX idx_permits_jurisdiction ON permits(jurisdiction_id);
CREATE INDEX idx_permits_status ON permits(status);
CREATE INDEX idx_permits_parcel ON permits(parcel_id);
CREATE INDEX idx_permits_applicant ON permits(applicant_id);
CREATE INDEX idx_permits_applied ON permits(applied_date);
CREATE INDEX idx_permits_num ON permits(permit_num);
```

### Contractors

```sql
CREATE TABLE contractors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    business_name VARCHAR(255) NOT NULL,
    contact_name VARCHAR(255),
    licence_number VARCHAR(100),
    licence_type VARCHAR(100),
    licence_state VARCHAR(50),
    licence_expiry_date DATE,
    phone VARCHAR(50),
    email VARCHAR(255),
    address TEXT,
    insurance_policy_number VARCHAR(100),
    insurance_expiry_date DATE,
    bonding_amount NUMERIC(14, 2),
    is_active BOOLEAN NOT NULL DEFAULT true,
    
    -- Graph-derived scoring (updated by graph analytics jobs)
    network_risk_score NUMERIC(5, 2),     -- Computed from graph: contractor's network violation rate
    jurisdiction_count INTEGER DEFAULT 0,  -- Number of jurisdictions this contractor operates in
    total_permits INTEGER DEFAULT 0,       -- Total permits across all jurisdictions
    violation_count INTEGER DEFAULT 0,     -- Total violations across all jurisdictions
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contractors_licence ON contractors(licence_number);
CREATE INDEX idx_contractors_user ON contractors(user_id);
CREATE INDEX idx_contractors_risk ON contractors(network_risk_score);
```

### Inspections

```sql
CREATE TABLE inspection_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    code VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    ibc_code_section VARCHAR(50),
    typical_duration_minutes INTEGER DEFAULT 30,
    checklist_config JSONB DEFAULT '[]',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, code)
);

CREATE TABLE inspections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permit_id UUID NOT NULL REFERENCES permits(id),
    inspection_type_id UUID NOT NULL REFERENCES inspection_types(id),
    parent_inspection_id UUID REFERENCES inspections(id),
    inspector_id UUID REFERENCES users(id),
    scheduled_date DATE,
    scheduled_time_start TIME,
    scheduled_time_end TIME,
    status VARCHAR(50) NOT NULL DEFAULT 'requested' CHECK (status IN (
        'requested', 'scheduled', 'in_progress', 'passed', 'failed',
        'partial', 'cancelled', 'no_access', 'rescheduled'
    )),
    result VARCHAR(50),
    result_notes TEXT,
    completed_at TIMESTAMPTZ,
    details JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_insp_permit ON inspections(permit_id);
CREATE INDEX idx_insp_inspector ON inspections(inspector_id);
CREATE INDEX idx_insp_status ON inspections(status);
CREATE INDEX idx_insp_scheduled ON inspections(scheduled_date);
```

### Licences, Enforcement, Payments, Documents

```sql
CREATE TABLE licence_types (
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

CREATE TABLE licences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    licence_type_id UUID NOT NULL REFERENCES licence_types(id),
    licence_number VARCHAR(100) NOT NULL,
    holder_name VARCHAR(255) NOT NULL,
    holder_user_id UUID REFERENCES users(id),
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    issued_date DATE,
    effective_date DATE,
    expiry_date DATE,
    details JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, licence_number)
);

CREATE INDEX idx_lic_jurisdiction ON licences(jurisdiction_id);
CREATE INDEX idx_lic_status ON licences(status);
CREATE INDEX idx_lic_expiry ON licences(expiry_date);

CREATE TABLE code_enforcement_cases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    case_number VARCHAR(100) NOT NULL,
    parcel_id UUID REFERENCES parcels(id),
    site_address TEXT,
    assigned_officer_id UUID REFERENCES users(id),
    source VARCHAR(50),
    violation_type VARCHAR(100),
    description TEXT NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'open',
    priority VARCHAR(20) DEFAULT 'normal',
    opened_date DATE NOT NULL DEFAULT CURRENT_DATE,
    resolved_date DATE,
    details JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, case_number)
);

CREATE INDEX idx_ce_jurisdiction ON code_enforcement_cases(jurisdiction_id);
CREATE INDEX idx_ce_status ON code_enforcement_cases(status);
CREATE INDEX idx_ce_parcel ON code_enforcement_cases(parcel_id);

CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(50) NOT NULL,     -- 'permit', 'inspection', 'licence', 'enforcement'
    entity_id UUID NOT NULL,
    document_type VARCHAR(100) NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    storage_path TEXT NOT NULL,
    file_type VARCHAR(50),
    file_size_bytes BIGINT,
    version INTEGER NOT NULL DEFAULT 1,
    is_current_version BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_docs_entity ON documents(entity_type, entity_id);
CREATE INDEX idx_docs_type ON documents(document_type);

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    entity_type VARCHAR(50) NOT NULL,     -- 'permit', 'licence', 'enforcement'
    entity_id UUID NOT NULL,
    payer_id UUID REFERENCES users(id),
    total_amount NUMERIC(12, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    status VARCHAR(50) NOT NULL,
    details JSONB NOT NULL DEFAULT '{}',
    paid_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pay_entity ON payments(entity_type, entity_id);
CREATE INDEX idx_pay_jurisdiction ON payments(jurisdiction_id);

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL,
    user_id UUID,
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(100) NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB,
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);

CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL,
    recipient_id UUID,
    channel VARCHAR(20) NOT NULL,
    notification_type VARCHAR(100) NOT NULL,
    content JSONB NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    sent_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notif_recipient ON notifications(recipient_id);
CREATE INDEX idx_notif_status ON notifications(status);
```

---

## Graph Layer

The graph layer captures relationships between all entities in the system. It uses a generic property graph model (nodes + edges) that can be queried with recursive CTEs in pure PostgreSQL, or with openCypher via the Apache AGE extension.

### Graph Tables

```sql
-- ==========================================
-- GRAPH NODES
-- ==========================================
-- Each node represents an entity in the relational layer
CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(50) NOT NULL,     -- 'jurisdiction', 'department', 'user', 'parcel', 'permit', 'contractor', 'inspection', 'licence', 'enforcement_case', 'document'
    entity_id UUID NOT NULL,              -- FK to the relational table (not enforced for flexibility)
    label VARCHAR(100) NOT NULL,          -- Human-readable label for display
    jurisdiction_id UUID,                 -- Tenant scoping (NULL for cross-jurisdiction nodes)
    
    -- Node properties (denormalised from relational tables for graph query performance)
    properties JSONB NOT NULL DEFAULT '{}',
    -- Examples:
    -- User node:       {"email": "...", "user_type": "inspector", "display_name": "..."}
    -- Permit node:     {"permit_num": "BLD-2026-00142", "status": "issued", "estimated_cost": 450000}
    -- Parcel node:     {"parcel_number": "15-23-400-001", "address": "123 Main St", "zoning": "R-1"}
    -- Contractor node: {"business_name": "Acme Construction", "licence_number": "CSLB-987654"}
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(entity_type, entity_id)
);

CREATE INDEX idx_gn_entity ON graph_nodes(entity_type, entity_id);
CREATE INDEX idx_gn_jurisdiction ON graph_nodes(jurisdiction_id);
CREATE INDEX idx_gn_label ON graph_nodes(label);
CREATE INDEX idx_gn_properties ON graph_nodes USING GIN(properties);

-- ==========================================
-- GRAPH EDGES (RELATIONSHIPS)
-- ==========================================
-- Each edge represents a typed, directed relationship between two nodes
CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id UUID NOT NULL REFERENCES graph_nodes(id),
    target_node_id UUID NOT NULL REFERENCES graph_nodes(id),
    relationship_type VARCHAR(100) NOT NULL,
    -- Relationship types form the vocabulary of the graph:
    --
    -- Jurisdiction relationships:
    --   'PARENT_OF'           jurisdiction -> jurisdiction
    --   'CONTAINS_DEPARTMENT' jurisdiction -> department
    --
    -- Parcel relationships:
    --   'LOCATED_IN'          parcel -> jurisdiction
    --   'OWNED_BY'            parcel -> user/contractor
    --   'ADJACENT_TO'         parcel -> parcel
    --   'ZONED_AS'            parcel -> zoning_class (virtual node)
    --
    -- Permit relationships:
    --   'APPLIED_FOR'         user -> permit
    --   'FILED_AT'            permit -> jurisdiction
    --   'LOCATED_ON'          permit -> parcel
    --   'CONTRACTED_BY'       permit -> contractor
    --   'REVIEWED_BY'         permit -> user (reviewer)
    --   'HAS_INSPECTION'      permit -> inspection
    --   'HAS_DOCUMENT'        permit -> document
    --   'RELATED_TO'          permit -> permit (linked permits)
    --
    -- Contractor relationships:
    --   'OPERATES_IN'         contractor -> jurisdiction
    --   'HOLDS_LICENCE'       contractor -> licence
    --   'WORKS_ON'            contractor -> permit
    --   'EMPLOYS'             contractor -> user
    --   'SUBCONTRACTOR_OF'    contractor -> contractor
    --
    -- Inspection relationships:
    --   'INSPECTED_BY'        inspection -> user (inspector)
    --   'FOR_PERMIT'          inspection -> permit
    --   'RE_INSPECTION_OF'    inspection -> inspection
    --   'RESULTED_IN'         inspection -> enforcement_case (when failure leads to enforcement)
    --
    -- Licence relationships:
    --   'HELD_BY'             licence -> user/contractor
    --   'ISSUED_BY'           licence -> jurisdiction
    --
    -- Enforcement relationships:
    --   'VIOLATION_AT'        enforcement_case -> parcel
    --   'ASSIGNED_TO'         enforcement_case -> user (officer)
    --   'TRIGGERED_BY'        enforcement_case -> inspection
    --   'RELATED_PERMIT'      enforcement_case -> permit
    --
    -- User/staff relationships:
    --   'WORKS_FOR'           user -> department
    --   'SUPERVISES'          user -> user
    --   'HAS_CONFLICT'        user -> contractor/parcel (conflict of interest)
    
    -- Edge properties
    properties JSONB DEFAULT '{}',
    -- Examples:
    -- CONTRACTED_BY: {"role": "general_contractor", "contract_date": "2026-05-01"}
    -- REVIEWED_BY:   {"department": "building", "decision": "approved", "review_cycle": 2}
    -- INSPECTED_BY:  {"result": "pass", "date": "2026-07-16"}
    -- OWNED_BY:      {"ownership_percentage": 50, "effective_date": "2020-01-15"}
    -- SUBCONTRACTOR_OF: {"trade": "electrical", "contract_value": 85000}
    -- HAS_CONFLICT:  {"conflict_type": "financial", "disclosed": true, "disclosure_date": "2026-03-01"}
    
    -- Temporal validity (when this relationship was/is active)
    valid_from TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to TIMESTAMPTZ,                 -- NULL = currently active
    
    -- Weight for scoring/ranking algorithms
    weight NUMERIC(8, 4) DEFAULT 1.0,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_source ON graph_edges(source_node_id);
CREATE INDEX idx_ge_target ON graph_edges(target_node_id);
CREATE INDEX idx_ge_type ON graph_edges(relationship_type);
CREATE INDEX idx_ge_source_type ON graph_edges(source_node_id, relationship_type);
CREATE INDEX idx_ge_target_type ON graph_edges(target_node_id, relationship_type);
CREATE INDEX idx_ge_valid ON graph_edges(valid_from, valid_to);
CREATE INDEX idx_ge_properties ON graph_edges USING GIN(properties);

-- Composite index for common traversal pattern: find all edges of a type from/to a node
CREATE INDEX idx_ge_traversal_out ON graph_edges(source_node_id, relationship_type, valid_to) WHERE valid_to IS NULL;
CREATE INDEX idx_ge_traversal_in ON graph_edges(target_node_id, relationship_type, valid_to) WHERE valid_to IS NULL;
```

---

## Graph Synchronisation

```sql
-- Tracks sync status between relational tables and graph layer
CREATE TABLE graph_sync_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    sync_action VARCHAR(20) NOT NULL CHECK (sync_action IN ('node_created', 'node_updated', 'node_deleted', 'edge_created', 'edge_updated', 'edge_deleted')),
    details JSONB,
    synced_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gsl_entity ON graph_sync_log(entity_type, entity_id);
CREATE INDEX idx_gsl_synced ON graph_sync_log(synced_at);

-- Pending sync queue (for eventual consistency between relational and graph layers)
CREATE TABLE graph_sync_queue (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    operation VARCHAR(20) NOT NULL CHECK (operation IN ('create', 'update', 'delete')),
    payload JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
    retry_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at TIMESTAMPTZ
);

CREATE INDEX idx_gsq_status ON graph_sync_queue(status, created_at);
```

---

## Graph Analytics Tables

```sql
-- Pre-computed graph analytics results (updated by batch jobs)
CREATE TABLE graph_analytics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_type VARCHAR(100) NOT NULL,
    -- Analysis types:
    -- 'contractor_risk_score'      — network-based risk scoring for contractors
    -- 'conflict_of_interest'       — detected conflicts between staff and applicants/contractors
    -- 'jurisdiction_overlap'       — contractors/entities operating across jurisdictions
    -- 'ownership_chain'            — multi-hop ownership chain analysis
    -- 'permit_cluster'             — geospatial/temporal permit clustering
    -- 'violation_pattern'          — recurring violation patterns by contractor/parcel
    
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    jurisdiction_id UUID,
    
    score NUMERIC(8, 4),                  -- Computed score (interpretation depends on analysis_type)
    results JSONB NOT NULL,               -- Full analysis results
    -- Example for contractor_risk_score:
    -- {
    --   "risk_score": 67.5,
    --   "factors": [
    --     {"factor": "violation_rate", "value": 0.15, "weight": 0.4, "contribution": 6.0},
    --     {"factor": "jurisdiction_count", "value": 8, "weight": 0.1, "contribution": 2.0},
    --     {"factor": "network_violation_rate", "value": 0.22, "weight": 0.3, "contribution": 6.6},
    --     {"factor": "subcontractor_risk_avg", "value": 45.2, "weight": 0.2, "contribution": 9.0}
    --   ],
    --   "network_stats": {
    --     "direct_connections": 42,
    --     "total_permits": 156,
    --     "total_violations": 23,
    --     "jurisdictions": ["Springfield", "Decatur", "Champaign"]
    --   }
    -- }
    
    -- Example for conflict_of_interest:
    -- {
    --   "conflict_type": "financial",
    --   "inspector_id": "uuid",
    --   "inspector_name": "Mike Davis",
    --   "related_entity_id": "uuid",
    --   "related_entity_name": "Acme Construction",
    --   "relationship_path": ["inspector(Mike Davis)", "SUPERVISES->", "user(Sarah Davis)", "EMPLOYS<-", "contractor(Acme Construction)"],
    --   "path_length": 3,
    --   "disclosed": false,
    --   "severity": "high"
    -- }
    
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ                -- When this analysis should be recomputed
);

CREATE INDEX idx_ga_type ON graph_analytics(analysis_type);
CREATE INDEX idx_ga_entity ON graph_analytics(entity_type, entity_id);
CREATE INDEX idx_ga_jurisdiction ON graph_analytics(jurisdiction_id);
CREATE INDEX idx_ga_score ON graph_analytics(analysis_type, score DESC);
CREATE INDEX idx_ga_expires ON graph_analytics(expires_at) WHERE expires_at IS NOT NULL;
```

---

## Example Graph Queries

### Recursive CTE: Find all permits connected to a contractor (any depth)

```sql
-- Traverse the graph to find all permits a contractor is connected to,
-- including through subcontractor relationships
WITH RECURSIVE contractor_network AS (
    -- Start with the contractor node
    SELECT gn.id AS node_id, gn.entity_id, gn.entity_type, gn.label,
           0 AS depth, ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.entity_type = 'contractor' AND gn.entity_id = 'contractor-uuid'
    
    UNION ALL
    
    -- Traverse outgoing SUBCONTRACTOR_OF and WORKS_ON edges
    SELECT gn2.id, gn2.entity_id, gn2.entity_type, gn2.label,
           cn.depth + 1, cn.path || gn2.id
    FROM contractor_network cn
    JOIN graph_edges ge ON ge.source_node_id = cn.node_id
    JOIN graph_nodes gn2 ON ge.target_node_id = gn2.id
    WHERE ge.relationship_type IN ('WORKS_ON', 'SUBCONTRACTOR_OF')
      AND ge.valid_to IS NULL              -- Only active relationships
      AND gn2.id != ALL(cn.path)           -- Prevent cycles
      AND cn.depth < 5                     -- Max traversal depth
)
SELECT cn.entity_type, cn.entity_id, cn.label, cn.depth,
       p.permit_num, p.status, p.site_address
FROM contractor_network cn
LEFT JOIN permits p ON cn.entity_type = 'permit' AND cn.entity_id = p.id
WHERE cn.entity_type = 'permit'
ORDER BY cn.depth, p.applied_date DESC;
```

### Conflict of Interest Detection

```sql
-- Find all inspectors who have a relationship (any depth <= 3) with
-- the contractor on a permit they are assigned to inspect
WITH RECURSIVE staff_network AS (
    -- Start with inspector nodes
    SELECT gn.id AS node_id, gn.entity_id AS inspector_id, gn.label AS inspector_name,
           0 AS depth, ARRAY[gn.id] AS path,
           ARRAY[]::text[] AS edge_types
    FROM graph_nodes gn
    WHERE gn.entity_type = 'user'
      AND gn.properties->>'user_type' = 'inspector'
    
    UNION ALL
    
    -- Traverse all relationship types (except common ones like FILED_AT)
    SELECT gn2.id, sn.inspector_id, sn.inspector_name,
           sn.depth + 1, sn.path || gn2.id,
           sn.edge_types || ge.relationship_type
    FROM staff_network sn
    JOIN graph_edges ge ON (ge.source_node_id = sn.node_id OR ge.target_node_id = sn.node_id)
    JOIN graph_nodes gn2 ON (
        CASE WHEN ge.source_node_id = sn.node_id THEN ge.target_node_id
             ELSE ge.source_node_id END = gn2.id
    )
    WHERE ge.relationship_type NOT IN ('FILED_AT', 'LOCATED_IN', 'HAS_DOCUMENT')
      AND ge.valid_to IS NULL
      AND gn2.id != ALL(sn.path)
      AND sn.depth < 3
)
SELECT DISTINCT
    sn.inspector_id, sn.inspector_name,
    c.id AS contractor_id, c.business_name,
    sn.depth AS relationship_distance,
    sn.edge_types AS relationship_path
FROM staff_network sn
JOIN graph_nodes gn ON sn.node_id = gn.id AND gn.entity_type = 'contractor'
JOIN contractors c ON gn.entity_id = c.id
-- Only flag if this inspector is actually assigned to inspect this contractor's permits
WHERE EXISTS (
    SELECT 1 FROM inspections i
    JOIN permits p ON i.permit_id = p.id
    JOIN graph_edges ge_pc ON ge_pc.relationship_type = 'CONTRACTED_BY'
    JOIN graph_nodes gn_c ON ge_pc.target_node_id = gn_c.id AND gn_c.entity_id = c.id
    JOIN graph_nodes gn_p ON ge_pc.source_node_id = gn_p.id AND gn_p.entity_id = p.id
    WHERE i.inspector_id = sn.inspector_id
      AND i.status IN ('requested', 'scheduled')
);
```

### Cross-Jurisdiction Contractor History

```sql
-- Find a contractor's complete history across all jurisdictions
SELECT j.name AS jurisdiction,
       COUNT(DISTINCT p.id) AS permits,
       COUNT(DISTINCT CASE WHEN i.result = 'fail' THEN i.id END) AS failed_inspections,
       COUNT(DISTINCT ce.id) AS enforcement_cases,
       ROUND(AVG(p.estimated_cost), 2) AS avg_project_value
FROM graph_nodes gn_c
JOIN graph_edges ge ON ge.source_node_id = gn_c.id AND ge.relationship_type = 'WORKS_ON'
JOIN graph_nodes gn_p ON ge.target_node_id = gn_p.id AND gn_p.entity_type = 'permit'
JOIN permits p ON gn_p.entity_id = p.id
JOIN jurisdictions j ON p.jurisdiction_id = j.id
LEFT JOIN inspections i ON i.permit_id = p.id
LEFT JOIN graph_edges ge_ce ON ge_ce.relationship_type = 'TRIGGERED_BY'
    AND ge_ce.source_node_id IN (SELECT id FROM graph_nodes WHERE entity_type = 'enforcement_case')
LEFT JOIN graph_nodes gn_ce ON ge_ce.source_node_id = gn_ce.id
LEFT JOIN code_enforcement_cases ce ON gn_ce.entity_id = ce.id
WHERE gn_c.entity_type = 'contractor' AND gn_c.entity_id = 'contractor-uuid'
GROUP BY j.name
ORDER BY permits DESC;
```

### Ownership Chain Traversal

```sql
-- Trace the ownership chain for a parcel (who owns the entity that owns the entity...)
WITH RECURSIVE ownership_chain AS (
    SELECT gn.id AS node_id, gn.entity_type, gn.entity_id, gn.label,
           ge.properties->>'ownership_percentage' AS ownership_pct,
           0 AS depth, ARRAY[gn.id] AS path
    FROM graph_nodes gn
    JOIN graph_edges ge ON ge.source_node_id = gn.id AND ge.relationship_type = 'OWNED_BY'
    JOIN graph_nodes gn_parcel ON ge.target_node_id = gn_parcel.id
    WHERE gn_parcel.entity_type = 'parcel' AND gn_parcel.entity_id = 'parcel-uuid'
      AND ge.valid_to IS NULL
    
    UNION ALL
    
    SELECT gn2.id, gn2.entity_type, gn2.entity_id, gn2.label,
           ge2.properties->>'ownership_percentage',
           oc.depth + 1, oc.path || gn2.id
    FROM ownership_chain oc
    JOIN graph_edges ge2 ON ge2.source_node_id = oc.node_id AND ge2.relationship_type = 'OWNED_BY'
    JOIN graph_nodes gn2 ON ge2.target_node_id = gn2.id
    WHERE ge2.valid_to IS NULL
      AND gn2.id != ALL(oc.path)
      AND oc.depth < 10
)
SELECT entity_type, entity_id, label, ownership_pct, depth
FROM ownership_chain
ORDER BY depth ASC;
```

### Apache AGE (openCypher) Alternative

If the Apache AGE extension is available, the same queries become more concise:

```sql
-- Contractor network using openCypher via Apache AGE
SELECT * FROM cypher('permitting_graph', $$
    MATCH (c:contractor {entity_id: 'contractor-uuid'})-[:WORKS_ON|SUBCONTRACTOR_OF*1..5]->(p:permit)
    RETURN p.permit_num, p.status, p.site_address
$$) AS (permit_num TEXT, status TEXT, site_address TEXT);

-- Conflict of interest detection
SELECT * FROM cypher('permitting_graph', $$
    MATCH (inspector:user {user_type: 'inspector'})-[*1..3]-(contractor:contractor)
    WHERE EXISTS {
        MATCH (inspector)-[:INSPECTED_BY]-(insp:inspection)-[:FOR_PERMIT]->(permit:permit)<-[:WORKS_ON]-(contractor)
        WHERE insp.status IN ['requested', 'scheduled']
    }
    RETURN inspector.display_name, contractor.business_name, 
           length(shortestPath((inspector)-[*]-(contractor))) AS distance
$$) AS (inspector TEXT, contractor TEXT, distance INTEGER);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity | 2 | jurisdictions, departments |
| Users | 1 | users |
| Parcels | 1 | parcels |
| Permit Configuration | 1 | permit_types |
| Permits | 1 | permits |
| Contractors | 1 | contractors (with graph-derived scoring columns) |
| Inspections | 2 | inspection_types, inspections |
| Licences | 2 | licence_types, licences |
| Code Enforcement | 1 | code_enforcement_cases |
| Documents | 1 | documents |
| Payments | 1 | payments |
| Notifications & Audit | 2 | notifications, audit_log |
| **Graph Layer** | **2** | **graph_nodes, graph_edges** |
| Graph Infrastructure | 3 | graph_sync_log, graph_sync_queue, graph_analytics |
| **Total** | **~21** | Relational (16) + Graph (5) |

---

## Key Design Decisions

1. **Dual-layer architecture: relational for CRUD, graph for relationships** — the relational layer handles transactional operations (creating permits, recording inspections, processing payments) where ACID guarantees matter. The graph layer handles relationship queries (contractor networks, ownership chains, conflict detection) where traversal performance matters. Neither layer is sufficient alone.

2. **Graph implemented in PostgreSQL tables (not a separate database)** — using `graph_nodes` and `graph_edges` tables within the same PostgreSQL database avoids the operational complexity of a separate graph database. The Apache AGE extension provides openCypher query capability for teams that want graph-native syntax, while recursive CTEs work for teams that prefer pure SQL.

3. **Temporal edges with `valid_from`/`valid_to`** — relationship edges have validity periods. When a contractor's licence expires, the edge is end-dated rather than deleted. When parcel ownership changes, the old ownership edge is end-dated and a new one created. This enables temporal graph queries ("who owned this parcel on date X?").

4. **Node properties denormalised from relational tables** — graph nodes carry a `properties` JSONB field that duplicates key fields from the relational layer. This avoids joining back to relational tables during graph traversal queries, which would negate the traversal performance benefit.

5. **Asynchronous graph synchronisation** — the `graph_sync_queue` table enables eventual consistency between relational and graph layers. When a permit is created or updated in the relational layer, a sync task is enqueued. A background worker processes the queue to create/update graph nodes and edges. This avoids adding graph write latency to every operational transaction.

6. **Graph-derived scoring on relational entities** — the `contractors` table includes columns like `network_risk_score`, `jurisdiction_count`, `violation_count` that are populated by graph analytics batch jobs. This allows the relational layer to use graph-computed scores in standard queries (e.g., "show high-risk contractor permits") without requiring graph queries at read time.

7. **Pre-computed analytics in `graph_analytics`** — expensive graph computations (contractor risk scoring, conflict-of-interest detection, ownership chain analysis) are run as batch jobs and stored in the `graph_analytics` table with expiry timestamps. Real-time graph queries are reserved for interactive exploration; dashboards and reports use pre-computed results.

8. **Relationship type vocabulary as domain language** — the edge `relationship_type` values (`WORKS_ON`, `CONTRACTED_BY`, `INSPECTED_BY`, `RESULTED_IN`, etc.) form a controlled vocabulary that maps directly to the domain language of permitting. This makes graph queries readable and maintainable.

9. **Cross-jurisdiction graph traversal** — unlike the relational layer where data is scoped by `jurisdiction_id`, the graph layer deliberately spans jurisdictions. A contractor node can have `OPERATES_IN` edges to multiple jurisdiction nodes, enabling cross-boundary analysis. This supports the underserved feature identified in the features research: cross-jurisdiction interoperability.

10. **Conflict-of-interest detection as a first-class feature** — the graph model makes it possible to detect indirect relationships between inspectors and contractors/applicants through intermediate entities (family members, shared business connections). This capability is impractical with pure relational queries but natural in a graph model.
