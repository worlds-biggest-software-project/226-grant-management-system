# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Grant Management System · Created: 2026-05-21

## Philosophy

This model combines conventional relational tables for operational CRUD with a property graph layer for relationship-heavy queries. The core insight is that grant management involves rich, interconnected relationships that are awkward to express in pure relational joins: an organisation is both a grantmaker for one programme and a subrecipient on another; a reviewer on Panel A is a PI on an application to Programme B (potential conflict of interest); a federal grant flows through a pass-through entity to multiple subrecipients who themselves have subrecipients; two foundations fund overlapping geographic regions serving the same population. These relationship networks are natural graph queries but painful multi-table JOINs.

The architecture uses PostgreSQL's `ltree` extension for hierarchical data (organisation hierarchies, budget category trees, jurisdiction hierarchies) and a pair of generic `graph_node` / `graph_edge` tables that create a property graph overlay on top of the relational entities. Every major entity (organisation, programme, application, award, user) has a corresponding graph node. Relationships between them (funds, applies_to, reviews, subawards, employs, conflicts_with) are graph edges with typed properties. This enables queries like "find all paths from Federal Agency X to end recipients within 3 hops" or "identify all reviewers who have organisational ties to any applicant" using recursive CTEs or, for larger deployments, a dedicated graph database (Neo4j, Amazon Neptune) fed from the same data.

The trade-off is dual-write complexity: when a new award is created, both the relational `awards` table and the graph layer must be updated. However, for a domain where relationship analysis, conflict-of-interest detection, and funding flow tracing are high-value features, the graph layer unlocks capabilities that would be prohibitively expensive in pure relational SQL.

**Best for:** Organisations that need to trace funding flows across multiple tiers of subrecipients, detect conflicts of interest among reviewers, analyse funder networks, or build relationship-based recommendation engines (e.g., "foundations that fund similar programmes").

**Trade-offs:**
- (+) Natural representation of funding flows, organisational hierarchies, and reviewer networks
- (+) Conflict-of-interest detection is a simple graph traversal, not a complex multi-JOIN
- (+) Subrecipient chain analysis ("follow the money") scales to arbitrary depth
- (+) Enables network-based grant prospecting ("funders connected to your existing funders")
- (+) ltree hierarchies are fast for jurisdiction and budget category lookups
- (-) Dual-write to relational + graph tables adds complexity and consistency risk
- (-) Graph query patterns (recursive CTEs) are less familiar to most developers
- (-) Graph layer must be kept in sync with relational tables
- (-) More infrastructure if a dedicated graph database is used alongside PostgreSQL
- (-) Higher storage overhead due to redundant relationship data

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OMB 2 CFR Part 200 (Uniform Guidance) | 200.332 subrecipient monitoring modelled as graph edges between pass-through entities and subrecipients; risk assessment properties on edges |
| GREAT Act / OMB M-24-11 | Core data elements in relational columns; graph layer provides relationship context for reporting |
| GSDM | Award and recipient relational tables mirror GSDM categories; graph edges connect awards to agencies, recipients, and programmes |
| FFATA / USASpending.gov | Funding flow graph enables traceability from federal agency through pass-through entities to end recipients, supporting FFATA reporting chain |
| SAM.gov / UEI | UEI on organisation nodes enables cross-referencing with SAM.gov entity data |
| ISO 3166 | Jurisdiction hierarchy modelled with ltree for efficient geographic queries |
| Assistance Listing Numbers | ALN on programme and award nodes; graph edges connect programmes to awards to recipients |

---

## Graph Layer Tables

```sql
-- Enable PostgreSQL extensions
CREATE EXTENSION IF NOT EXISTS ltree;

-- Graph nodes: one row per entity that participates in the relationship graph
CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    node_type VARCHAR(50) NOT NULL,           -- 'organisation', 'user', 'programme', 'opportunity', 'application', 'award'
    entity_id UUID NOT NULL,                  -- FK to the corresponding relational table (not enforced to stay generic)
    label VARCHAR(500) NOT NULL,              -- human-readable label for display
    properties JSONB DEFAULT '{}',            -- node-specific summary properties for graph queries
    -- properties example for organisation:
    -- {"uei": "ABC123DEF456", "type": "nonprofit", "jurisdiction": "US-VA", "total_awards": 5}
    -- properties example for award:
    -- {"amount": 250000, "status": "active", "aln": "47.076", "start_date": "2026-01-01"}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, node_type, entity_id)
);

CREATE INDEX idx_graph_nodes_tenant ON graph_nodes(tenant_id, node_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes(entity_id);
CREATE INDEX idx_graph_nodes_props ON graph_nodes USING GIN (properties);

-- Graph edges: typed relationships between nodes
CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    source_node_id UUID NOT NULL REFERENCES graph_nodes(id),
    target_node_id UUID NOT NULL REFERENCES graph_nodes(id),
    edge_type VARCHAR(50) NOT NULL,           -- relationship type (see catalogue below)
    properties JSONB DEFAULT '{}',            -- edge-specific data
    weight NUMERIC(10,4) DEFAULT 1.0,         -- for weighted graph algorithms
    valid_from TIMESTAMPTZ DEFAULT now(),     -- temporal validity
    valid_to TIMESTAMPTZ,                     -- NULL = currently valid
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(source_node_id, target_node_id, edge_type, valid_from)
);

CREATE INDEX idx_graph_edges_source ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_graph_edges_tenant ON graph_edges(tenant_id, edge_type);
CREATE INDEX idx_graph_edges_valid ON graph_edges(source_node_id, edge_type) WHERE valid_to IS NULL;
```

### Edge Type Catalogue

```sql
-- Edge types and their semantics:
--
-- FUNDING RELATIONSHIPS
-- 'funds'              org -> programme          Foundation X funds Programme Y
-- 'awards_to'          programme -> award         Programme Y awards Award Z
-- 'receives'           award -> org              Organisation Q receives Award Z
-- 'subawards_to'       award -> award            Pass-through Award Z subawards to Sub-Award W
--                      properties: {subaward_number, amount, risk_level}
-- 'pass_through'       org -> org                Org A passes federal funds through to Org B
--                      properties: {award_id, amount}
--
-- APPLICATION RELATIONSHIPS
-- 'applies_to'         org -> opportunity        Org applies to Opportunity
-- 'submits'            user -> application       User submits Application
-- 'collaborates_on'    org -> application        Co-PI organisation on application
--
-- REVIEW RELATIONSHIPS
-- 'reviews'            user -> application       Reviewer assigned to Application
--                      properties: {score, recommendation, panel_id}
-- 'panelist_on'        user -> opportunity       User is panelist for Opportunity review
--
-- ORGANISATIONAL RELATIONSHIPS
-- 'employs'            org -> user               Organisation employs User
-- 'affiliated_with'    user -> org               User has affiliation (board, consultant, etc.)
--                      properties: {role, start_date, end_date}
-- 'subsidiary_of'      org -> org                Org A is subsidiary of Org B
-- 'partners_with'      org -> org                Strategic partnership
--
-- CONFLICT RELATIONSHIPS (derived / computed)
-- 'potential_conflict'  user -> application      Reviewer has organisational tie to applicant
--                      properties: {conflict_type, org_id, detected_at, resolved}
--
-- GEOGRAPHIC RELATIONSHIPS
-- 'located_in'         org -> jurisdiction       Organisation's primary jurisdiction
-- 'serves'             programme -> jurisdiction Programme serves this geographic area
```

---

## Hierarchical Data with ltree

```sql
-- Jurisdiction hierarchy using ltree
CREATE TABLE jurisdictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    name VARCHAR(200) NOT NULL,
    level VARCHAR(20) NOT NULL,               -- 'country', 'state', 'county', 'city'
    path ltree NOT NULL,                      -- e.g., 'US', 'US.VA', 'US.VA.Fairfax', 'US.VA.Fairfax.Reston'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jurisdictions_path ON jurisdictions USING GIST (path);
CREATE INDEX idx_jurisdictions_country ON jurisdictions(country_code);

-- Example queries:
-- All subdivisions of Virginia:
--   SELECT * FROM jurisdictions WHERE path <@ 'US.VA';
-- All ancestors of a city:
--   SELECT * FROM jurisdictions WHERE path @> 'US.VA.Fairfax.Reston';

-- Budget category hierarchy using ltree
CREATE TABLE budget_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    is_direct_cost BOOLEAN NOT NULL,
    path ltree NOT NULL,                      -- e.g., 'direct', 'direct.personnel', 'direct.personnel.salary'
    display_order INTEGER NOT NULL
);

CREATE INDEX idx_budget_categories_path ON budget_categories USING GIST (path);
```

---

## Relational Tables (Operational CRUD)

### Organisations

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    tenant_type VARCHAR(30) NOT NULL,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(500) NOT NULL,
    legal_name VARCHAR(500),
    uei VARCHAR(12),
    ein VARCHAR(10),
    organisation_type VARCHAR(50) NOT NULL,
    jurisdiction_id UUID REFERENCES jurisdictions(id),
    physical_address JSONB,
    mailing_address JSONB,
    website VARCHAR(500),
    sam_registration_status VARCHAR(30),
    sam_registration_expiry DATE,
    -- Graph node reference for fast lookups
    graph_node_id UUID REFERENCES graph_nodes(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisations_tenant ON organisations(tenant_id);
CREATE INDEX idx_organisations_uei ON organisations(uei) WHERE uei IS NOT NULL;
```

### Users

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email VARCHAR(320) NOT NULL,
    display_name VARCHAR(200) NOT NULL,
    password_hash VARCHAR(255),
    auth_provider VARCHAR(30) DEFAULT 'local',
    is_active BOOLEAN NOT NULL DEFAULT true,
    graph_node_id UUID REFERENCES graph_nodes(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    permissions JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE user_roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    role_id UUID NOT NULL REFERENCES roles(id),
    organisation_id UUID REFERENCES organisations(id),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, role_id, organisation_id)
);
```

### Programmes, Opportunities, Applications

```sql
CREATE TABLE grant_programmes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    funder_organisation_id UUID NOT NULL REFERENCES organisations(id),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    programme_type VARCHAR(50) NOT NULL,
    aln VARCHAR(10),
    total_budget NUMERIC(15,2),
    currency CHAR(3) DEFAULT 'USD',
    fiscal_year INTEGER,
    is_active BOOLEAN DEFAULT true,
    graph_node_id UUID REFERENCES graph_nodes(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_programmes_tenant ON grant_programmes(tenant_id);
CREATE INDEX idx_programmes_aln ON grant_programmes(aln) WHERE aln IS NOT NULL;

CREATE TABLE funding_opportunities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    grant_programme_id UUID NOT NULL REFERENCES grant_programmes(id),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    eligibility_criteria TEXT,
    opportunity_number VARCHAR(100),
    grants_gov_id VARCHAR(50),
    posted_date DATE,
    close_date TIMESTAMPTZ,
    expected_awards_count INTEGER,
    award_floor NUMERIC(15,2),
    award_ceiling NUMERIC(15,2),
    total_funding_available NUMERIC(15,2),
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    graph_node_id UUID REFERENCES graph_nodes(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE applications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    funding_opportunity_id UUID NOT NULL REFERENCES funding_opportunities(id),
    applicant_organisation_id UUID NOT NULL REFERENCES organisations(id),
    submitted_by UUID REFERENCES users(id),
    application_number VARCHAR(50) NOT NULL,
    title VARCHAR(500) NOT NULL,
    abstract TEXT,
    requested_amount NUMERIC(15,2),
    project_start_date DATE,
    project_end_date DATE,
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    submitted_at TIMESTAMPTZ,
    decision_date DATE,
    form_data JSONB DEFAULT '{}',
    graph_node_id UUID REFERENCES graph_nodes(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_applications_tenant ON applications(tenant_id);
CREATE INDEX idx_applications_opportunity ON applications(funding_opportunity_id);
CREATE INDEX idx_applications_status ON applications(tenant_id, status);
```

### Awards & Financial

```sql
CREATE TABLE awards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    application_id UUID REFERENCES applications(id),
    grant_programme_id UUID NOT NULL REFERENCES grant_programmes(id),
    recipient_organisation_id UUID NOT NULL REFERENCES organisations(id),
    award_number VARCHAR(100) NOT NULL,
    federal_award_id VARCHAR(50),
    title VARCHAR(500) NOT NULL,
    award_amount NUMERIC(15,2) NOT NULL,
    obligated_amount NUMERIC(15,2) DEFAULT 0,
    disbursed_amount NUMERIC(15,2) DEFAULT 0,
    currency CHAR(3) DEFAULT 'USD',
    award_date DATE NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'active',
    aln VARCHAR(10),
    indirect_cost_rate NUMERIC(5,4),
    -- Pass-through tracking for subrecipient chains
    parent_award_id UUID REFERENCES awards(id),  -- if this is a subaward
    pass_through_organisation_id UUID REFERENCES organisations(id),
    tier_level INTEGER DEFAULT 1,                 -- 1 = direct, 2 = sub, 3 = sub-sub
    graph_node_id UUID REFERENCES graph_nodes(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_awards_tenant ON awards(tenant_id);
CREATE INDEX idx_awards_recipient ON awards(recipient_organisation_id);
CREATE INDEX idx_awards_parent ON awards(parent_award_id) WHERE parent_award_id IS NOT NULL;
CREATE INDEX idx_awards_status ON awards(tenant_id, status);

CREATE TABLE budget_line_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    award_id UUID NOT NULL REFERENCES awards(id),
    budget_category_id UUID NOT NULL REFERENCES budget_categories(id),
    description VARCHAR(500),
    budgeted_amount NUMERIC(15,2) NOT NULL,
    modified_amount NUMERIC(15,2),
    fiscal_year INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_budget_items_award ON budget_line_items(award_id);

CREATE TABLE financial_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    award_id UUID NOT NULL REFERENCES awards(id),
    budget_line_item_id UUID REFERENCES budget_line_items(id),
    transaction_type VARCHAR(30) NOT NULL,
    amount NUMERIC(15,2) NOT NULL,
    transaction_date DATE NOT NULL,
    description VARCHAR(500),
    reference_number VARCHAR(100),
    is_allowable BOOLEAN,
    disallowance_reason TEXT,
    posted_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fin_tx_award ON financial_transactions(award_id);
CREATE INDEX idx_fin_tx_date ON financial_transactions(award_id, transaction_date);
```

### Reviews

```sql
CREATE TABLE review_assignments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    application_id UUID NOT NULL REFERENCES applications(id),
    reviewer_user_id UUID NOT NULL REFERENCES users(id),
    review_type VARCHAR(30) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'assigned',
    scores JSONB DEFAULT '[]',
    overall_score NUMERIC(5,2),
    recommendation VARCHAR(30),
    comments TEXT,
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    UNIQUE(application_id, reviewer_user_id)
);

-- Note: When a review assignment is created, a 'reviews' graph edge is also created.
-- When a conflict is detected, a 'potential_conflict' edge is created.
```

### Compliance & Documents

```sql
CREATE TABLE compliance_obligations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    award_id UUID NOT NULL REFERENCES awards(id),
    requirement_code VARCHAR(50) NOT NULL,
    title VARCHAR(300) NOT NULL,
    regulation_reference VARCHAR(200),
    due_date DATE,
    status VARCHAR(30) NOT NULL DEFAULT 'pending',
    completed_at TIMESTAMPTZ,
    detail JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_award ON compliance_obligations(award_id);
CREATE INDEX idx_compliance_status ON compliance_obligations(status, due_date);

CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    uploaded_by UUID NOT NULL REFERENCES users(id),
    file_name VARCHAR(500) NOT NULL,
    file_type VARCHAR(100),
    file_size_bytes BIGINT,
    storage_key VARCHAR(1000) NOT NULL,
    document_type VARCHAR(50),
    owner_type VARCHAR(50) NOT NULL,
    owner_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_owner ON documents(owner_type, owner_id);

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    user_id UUID,
    action VARCHAR(30) NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB,
    ip_address INET,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(tenant_id, entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(tenant_id, occurred_at);
```

---

## Graph Query Examples

### Conflict of Interest Detection

```sql
-- Find potential conflicts: reviewers who have organisational ties to applicants
-- on the same funding opportunity
WITH reviewer_orgs AS (
    -- All organisations a reviewer is affiliated with
    SELECT
        e.source_node_id AS user_node_id,
        e.target_node_id AS org_node_id,
        e.edge_type,
        e.properties->>'role' AS role
    FROM graph_edges e
    WHERE e.edge_type IN ('employs', 'affiliated_with')
      AND e.valid_to IS NULL
),
applicant_orgs AS (
    -- All organisations that have applied
    SELECT
        e.source_node_id AS org_node_id,
        e.target_node_id AS application_node_id,
        gn_app.entity_id AS application_id
    FROM graph_edges e
    JOIN graph_nodes gn_app ON gn_app.id = e.target_node_id
    WHERE e.edge_type = 'applies_to'
),
reviewer_assignments AS (
    -- All reviewers assigned to applications
    SELECT
        e.source_node_id AS reviewer_node_id,
        e.target_node_id AS application_node_id,
        gn_rev.entity_id AS reviewer_user_id,
        gn_app.entity_id AS application_id
    FROM graph_edges e
    JOIN graph_nodes gn_rev ON gn_rev.id = e.source_node_id
    JOIN graph_nodes gn_app ON gn_app.id = e.target_node_id
    WHERE e.edge_type = 'reviews'
)
SELECT
    ra.reviewer_user_id,
    ra.application_id,
    ro.role AS conflict_role,
    gn_org.label AS conflicting_organisation
FROM reviewer_assignments ra
JOIN reviewer_orgs ro ON ro.user_node_id = ra.reviewer_node_id
JOIN applicant_orgs ao ON ao.org_node_id = ro.org_node_id
                      AND ao.application_node_id = ra.application_node_id
JOIN graph_nodes gn_org ON gn_org.id = ro.org_node_id;
```

### Subrecipient Funding Flow Trace

```sql
-- Trace the complete funding flow from a federal agency to end recipients
-- using recursive CTE on graph edges
WITH RECURSIVE funding_path AS (
    -- Start from the federal programme
    SELECT
        e.source_node_id,
        e.target_node_id,
        e.edge_type,
        e.properties,
        1 AS depth,
        ARRAY[e.source_node_id] AS path
    FROM graph_edges e
    JOIN graph_nodes gn ON gn.id = e.source_node_id
    WHERE gn.entity_id = '...'  -- federal programme UUID
      AND e.edge_type IN ('awards_to', 'subawards_to', 'pass_through')
      AND e.valid_to IS NULL

    UNION ALL

    -- Follow the chain
    SELECT
        e.source_node_id,
        e.target_node_id,
        e.edge_type,
        e.properties,
        fp.depth + 1,
        fp.path || e.source_node_id
    FROM graph_edges e
    JOIN funding_path fp ON e.source_node_id = fp.target_node_id
    WHERE e.edge_type IN ('awards_to', 'subawards_to', 'pass_through', 'receives')
      AND e.valid_to IS NULL
      AND fp.depth < 5  -- safety limit
      AND NOT (e.source_node_id = ANY(fp.path))  -- prevent cycles
)
SELECT
    fp.depth,
    fp.edge_type,
    source_gn.label AS from_entity,
    target_gn.label AS to_entity,
    fp.properties->>'amount' AS amount
FROM funding_path fp
JOIN graph_nodes source_gn ON source_gn.id = fp.source_node_id
JOIN graph_nodes target_gn ON target_gn.id = fp.target_node_id
ORDER BY fp.depth, source_gn.label;
```

### Funder Network Analysis (Grant Prospecting)

```sql
-- Find foundations that fund similar programmes to those your organisation already receives from
-- ("funders of my funders' other grantees also fund programmes like mine")
WITH my_funders AS (
    -- Funders that have awarded grants to my organisation
    SELECT DISTINCT e_fund.source_node_id AS funder_node_id
    FROM graph_edges e_recv
    JOIN graph_edges e_award ON e_award.target_node_id = e_recv.source_node_id
    JOIN graph_edges e_fund ON e_fund.target_node_id = e_award.source_node_id
    JOIN graph_nodes gn_org ON gn_org.id = e_recv.target_node_id
    WHERE gn_org.entity_id = '...'  -- my organisation UUID
      AND e_recv.edge_type = 'receives'
      AND e_award.edge_type = 'awards_to'
      AND e_fund.edge_type = 'funds'
),
similar_funders AS (
    -- Other funders that fund the same programme areas as my funders
    SELECT
        e2.source_node_id AS similar_funder_node_id,
        COUNT(*) AS shared_programme_count
    FROM my_funders mf
    JOIN graph_edges e1 ON e1.source_node_id = mf.funder_node_id AND e1.edge_type = 'funds'
    JOIN graph_nodes gn_prog ON gn_prog.id = e1.target_node_id
    JOIN graph_edges e2 ON e2.edge_type = 'funds'
        AND e2.target_node_id IN (
            SELECT id FROM graph_nodes WHERE node_type = 'programme'
            AND properties->>'programme_type' = gn_prog.properties->>'programme_type'
        )
    WHERE e2.source_node_id NOT IN (SELECT funder_node_id FROM my_funders)
    GROUP BY e2.source_node_id
)
SELECT
    gn.label AS potential_funder,
    gn.properties->>'type' AS funder_type,
    sf.shared_programme_count
FROM similar_funders sf
JOIN graph_nodes gn ON gn.id = sf.similar_funder_node_id
ORDER BY sf.shared_programme_count DESC
LIMIT 20;
```

### Geographic Coverage Analysis

```sql
-- Which jurisdictions are served by active programmes but have no applicants?
-- (identify underserved regions)
SELECT j.name, j.path::text, j.level
FROM jurisdictions j
WHERE j.path <@ 'US'
  AND j.level = 'state'
  AND EXISTS (
      SELECT 1 FROM graph_edges e
      JOIN graph_nodes gn ON gn.id = e.source_node_id
      WHERE e.edge_type = 'serves'
        AND e.target_node_id IN (SELECT id FROM graph_nodes WHERE node_type = 'jurisdiction' AND entity_id = j.id)
  )
  AND NOT EXISTS (
      SELECT 1 FROM organisations o
      WHERE o.jurisdiction_id = j.id
        AND EXISTS (
            SELECT 1 FROM applications a WHERE a.applicant_organisation_id = o.id
        )
  );
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_nodes, graph_edges |
| Multi-tenancy & Organisations | 3 | tenants, organisations, jurisdictions (ltree) |
| Users & Access Control | 3 | users, roles, user_roles |
| Programmes & Opportunities | 2 | grant_programmes, funding_opportunities |
| Applications & Reviews | 2 | applications, review_assignments |
| Awards & Finance | 4 | awards, budget_line_items, budget_categories (ltree), financial_transactions |
| Compliance & Documents | 3 | compliance_obligations, documents, audit_log |
| **Total** | **19** | Plus graph layer (2 tables); moderate count with relationship richness in graph |

---

## Key Design Decisions

1. **Generic graph overlay, not embedded graph**: The `graph_nodes` / `graph_edges` tables are separate from the relational tables rather than being embedded in them (e.g., as adjacency lists on each table). This keeps the relational schema clean for CRUD operations while enabling arbitrary relationship queries through the graph layer.

2. **Bidirectional entity-to-node reference**: Relational entities have a `graph_node_id` column for fast graph lookups, and graph nodes have an `entity_id` column for resolving back to relational data. This bidirectional link minimises JOINs in common access patterns.

3. **Temporal edges for relationship history**: Graph edges have `valid_from` and `valid_to` columns, enabling historical relationship queries: "Who was affiliated with Organisation X when they reviewed Application Y?" This is critical for audit investigations.

4. **ltree for hierarchical data**: Jurisdictions and budget categories use PostgreSQL's `ltree` extension rather than recursive self-referencing tables. ltree queries (ancestor, descendant, depth) are significantly faster than recursive CTEs for static hierarchies.

5. **Subaward chain in relational tables**: The `awards` table includes `parent_award_id` and `tier_level` for direct relational queries on subaward chains, in addition to the graph representation. This provides both fast operational lookups and rich network analysis.

6. **Conflict-of-interest as computed graph edges**: When a review assignment is created, the system traverses the graph to find organisational ties between the reviewer and the applicant organisation. Detected conflicts are stored as `potential_conflict` edges with metadata for human review.

7. **Edge weight for ranking**: Graph edges include a `weight` column that can be used for weighted graph algorithms (e.g., funding flow analysis, funder similarity scoring, risk propagation in subrecipient networks).

8. **Graph layer is eventually consistent**: The graph layer is updated asynchronously when relational data changes. This means graph queries may lag by a few seconds, but this is acceptable for analytical and detection use cases. Operational CRUD uses relational tables directly.

9. **Properties JSONB on nodes and edges**: Rather than creating separate tables for each node/edge type's attributes, type-specific data is stored in JSONB `properties` columns with GIN indexes. This keeps the graph layer flexible without table proliferation.

10. **Migration path to dedicated graph database**: The `graph_nodes` / `graph_edges` schema is intentionally compatible with property graph export formats. For organisations with large relationship networks, the data can be exported to Neo4j (via Cypher LOAD CSV) or Amazon Neptune (via Gremlin/SPARQL) without restructuring the relational side.
