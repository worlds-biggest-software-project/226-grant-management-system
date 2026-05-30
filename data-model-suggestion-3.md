# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Grant Management System · Created: 2026-05-21

## Philosophy

This model uses a reduced set of relational tables for core structural entities (organisations, programmes, applications, awards) while pushing variable, jurisdiction-specific, and programme-specific data into JSONB columns. The insight is that grant management spans an enormous range of contexts -- a federal STEM research grant and a community foundation arts grant share the same lifecycle skeleton (apply, review, award, report) but differ dramatically in their detail fields, compliance requirements, budget structures, and reporting formats. A fully normalised schema forces all these variants into a single rigid structure; a hybrid approach puts the shared skeleton in relational columns and the variable detail in JSONB.

This is the pattern used by platforms like WizeHive Zengineer and Submittable, which let each programme define its own form fields and data structures. In the API layer, the Submittable v4 API reflects this: submissions have a fixed envelope (id, status, created_at, submitter) but variable `formEntries` containing programme-specific field data. The same pattern applies to budget structures (federal OMB categories vs. foundation-specific categories), compliance checklists (federal 2 CFR 200 vs. state-specific rules), and reporting templates.

The trade-off is query complexity for the variable fields. You can still filter and index JSONB (PostgreSQL GIN indexes, containment operators), but queries like "find all applications where the custom field 'population_served' exceeds 10,000" require JSONB path expressions rather than simple column comparisons. For a platform serving multiple programme types across multiple jurisdictions, this flexibility is worth the query overhead.

**Best for:** Multi-programme, multi-jurisdiction platforms where each funder defines different application fields, budget categories, compliance requirements, and reporting templates. Ideal for rapid MVP development and platforms that serve both federal and foundation grantmakers.

**Trade-offs:**
- (+) Highly flexible: new programme-specific fields require no schema migration
- (+) Fewer tables (~20-25) than normalised approach, faster initial development
- (+) Naturally supports multi-programme platforms where each programme has different forms
- (+) JSONB fields align well with JSON Schema validation (GREAT Act, OpenAPI 3.1)
- (+) API layer can expose dynamic schemas per programme
- (-) JSONB queries are more verbose and harder to optimise than column queries
- (-) Data integrity for JSONB fields must be enforced at application level, not database level
- (-) Reporting across heterogeneous JSONB structures requires careful query design
- (-) GIN indexes on JSONB are larger than B-tree indexes on columns
- (-) Risk of "schema-in-code" where the real schema is scattered across application validators

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OMB 2 CFR Part 200 (Uniform Guidance) | Federal compliance fields stored as a JSON Schema-validated JSONB structure within awards; programmes can opt in to federal compliance mode which enforces 2 CFR 200 field requirements |
| GREAT Act / OMB M-24-11 | Core GREAT Act data elements map to relational columns; additional elements stored in `federal_data` JSONB with schema validation against the 540 standardised fields |
| GSDM | Federal reporting view assembles GSDM-aligned output from relational columns + JSONB fields |
| JSON Schema (Draft 2020-12) | Each programme defines a JSON Schema for its application form, budget template, and reporting template; JSONB payloads are validated against these schemas at the application layer |
| SAM.gov / UEI | UEI is a relational column (frequently queried); additional SAM.gov registration data in JSONB |
| Assistance Listing Numbers | Relational column on programmes and awards for indexed lookup |
| FFATA / USASpending.gov | FFATA-required fields are relational columns; supplementary reporting fields in JSONB |
| ISO 3166 | Jurisdiction codes in relational columns; jurisdiction-specific compliance rules in JSONB |
| OpenAPI 3.1 | API dynamically generates OpenAPI schemas from stored JSON Schemas, so each programme's API documentation reflects its custom fields |

---

## Core Tables

### Multi-Tenancy & Organisations

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    tenant_type VARCHAR(30) NOT NULL,         -- 'grantmaker', 'grantseeker', 'government'
    settings JSONB DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_currency": "USD",
    --   "fiscal_year_start_month": 10,
    --   "federal_compliance_enabled": true,
    --   "branding": {"logo_url": "...", "primary_color": "#003366"}
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(500) NOT NULL,
    legal_name VARCHAR(500),
    uei VARCHAR(12),                          -- relational: frequently queried
    ein VARCHAR(10),
    organisation_type VARCHAR(50) NOT NULL,
    jurisdiction_country CHAR(2),             -- ISO 3166-1
    jurisdiction_subdivision VARCHAR(6),      -- ISO 3166-2
    -- Variable organisation data: addresses, contacts, business types, SAM details
    profile JSONB DEFAULT '{}',
    -- profile example:
    -- {
    --   "physical_address": {"street": "123 Main St", "city": "Washington", "state": "DC", "zip": "20001"},
    --   "mailing_address": {"street": "PO Box 100", "city": "Washington", "state": "DC", "zip": "20001"},
    --   "phone": "+12025551234",
    --   "website": "https://example.org",
    --   "naics_codes": ["541710", "541720"],
    --   "business_types": ["nonprofit_501c3", "small_disadvantaged"],
    --   "sam_registration": {
    --     "status": "active",
    --     "expiry_date": "2027-03-15",
    --     "cage_code": "1ABC2"
    --   },
    --   "fiscal_year_end_month": 6,
    --   "congressional_district": "DC-01"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisations_tenant ON organisations(tenant_id);
CREATE INDEX idx_organisations_uei ON organisations(uei) WHERE uei IS NOT NULL;
CREATE INDEX idx_organisations_type ON organisations(tenant_id, organisation_type);
CREATE INDEX idx_organisations_profile ON organisations USING GIN (profile);
```

### Users & Access Control

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email VARCHAR(320) NOT NULL,
    display_name VARCHAR(200) NOT NULL,
    password_hash VARCHAR(255),
    auth_provider VARCHAR(30) DEFAULT 'local',
    auth_provider_id VARCHAR(500),
    is_active BOOLEAN NOT NULL DEFAULT true,
    -- Role assignments, organisation memberships, preferences all in one JSONB
    access JSONB DEFAULT '{}',
    -- access example:
    -- {
    --   "roles": ["admin", "programme_officer"],
    --   "organisations": [
    --     {"id": "uuid", "relationship": "admin", "is_primary": true}
    --   ],
    --   "programme_scopes": ["uuid-1", "uuid-2"],
    --   "permissions_override": ["reports.export"]
    -- }
    preferences JSONB DEFAULT '{}',
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_access ON users USING GIN (access);
```

### Grant Programmes & Funding Opportunities

```sql
CREATE TABLE grant_programmes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    funder_organisation_id UUID NOT NULL REFERENCES organisations(id),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    programme_type VARCHAR(50) NOT NULL,      -- 'federal', 'state', 'foundation', 'corporate'
    aln VARCHAR(10),                          -- relational: Assistance Listing Number
    total_budget NUMERIC(15,2),
    currency CHAR(3) DEFAULT 'USD',
    fiscal_year INTEGER,
    is_active BOOLEAN DEFAULT true,
    -- Programme-specific configuration: application form schemas, budget templates, compliance rules
    config JSONB DEFAULT '{}',
    -- config example:
    -- {
    --   "application_schema": { ... JSON Schema for custom application fields ... },
    --   "budget_template": {
    --     "categories": [
    --       {"code": "personnel", "label": "Personnel / Salaries", "is_direct": true},
    --       {"code": "fringe", "label": "Fringe Benefits", "is_direct": true},
    --       {"code": "travel", "label": "Travel", "is_direct": true},
    --       {"code": "equipment", "label": "Equipment (>$5,000)", "is_direct": true},
    --       {"code": "supplies", "label": "Supplies", "is_direct": true},
    --       {"code": "contractual", "label": "Contractual", "is_direct": true},
    --       {"code": "other", "label": "Other Direct Costs", "is_direct": true},
    --       {"code": "indirect", "label": "Indirect Costs", "is_direct": false}
    --     ]
    --   },
    --   "compliance_rules": {
    --     "federal_compliance": true,
    --     "indirect_cost_rate_cap": 0.15,
    --     "single_audit_threshold": 1000000,
    --     "required_reports": ["sf425", "rppr", "ffata"],
    --     "subrecipient_monitoring": true
    --   },
    --   "review_config": {
    --     "review_type": "blind",
    --     "min_reviewers": 3,
    --     "scoring_rubric": [
    --       {"criterion": "Significance", "max_score": 10, "weight": 0.30},
    --       {"criterion": "Approach", "max_score": 10, "weight": 0.25},
    --       {"criterion": "Innovation", "max_score": 10, "weight": 0.20},
    --       {"criterion": "Team", "max_score": 10, "weight": 0.15},
    --       {"criterion": "Budget Justification", "max_score": 10, "weight": 0.10}
    --     ]
    --   },
    --   "reporting_templates": {
    --     "progress_report_schema": { ... JSON Schema ... },
    --     "final_report_schema": { ... JSON Schema ... }
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_programmes_tenant ON grant_programmes(tenant_id);
CREATE INDEX idx_programmes_funder ON grant_programmes(funder_organisation_id);
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
    -- Opportunity-specific overrides to programme config
    config_overrides JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_opportunities_tenant ON funding_opportunities(tenant_id);
CREATE INDEX idx_opportunities_programme ON funding_opportunities(grant_programme_id);
CREATE INDEX idx_opportunities_status ON funding_opportunities(tenant_id, status);
```

### Applications (Core + Dynamic Form Data)

```sql
CREATE TABLE applications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    funding_opportunity_id UUID NOT NULL REFERENCES funding_opportunities(id),
    applicant_organisation_id UUID NOT NULL REFERENCES organisations(id),
    submitted_by UUID REFERENCES users(id),
    application_number VARCHAR(50) NOT NULL,
    title VARCHAR(500) NOT NULL,
    requested_amount NUMERIC(15,2),
    project_start_date DATE,
    project_end_date DATE,
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    submitted_at TIMESTAMPTZ,
    decision_date DATE,
    -- Dynamic form data: validated against programme's application_schema
    form_data JSONB DEFAULT '{}',
    -- form_data example (varies by programme):
    -- {
    --   "abstract": "This project will...",
    --   "target_population": "Underserved communities in rural Appalachia",
    --   "population_served": 15000,
    --   "project_goals": ["Increase access to...", "Train 50 community..."],
    --   "key_personnel": [
    --     {"name": "Dr. Jane Smith", "role": "PI", "effort_pct": 50},
    --     {"name": "John Doe", "role": "Co-PI", "effort_pct": 25}
    --   ],
    --   "prior_funding_history": [
    --     {"funder": "NSF", "award_number": "2345678", "amount": 250000, "year": 2024}
    --   ],
    --   "congressional_district": "VA-09",
    --   "geographic_scope": "regional"
    -- }
    -- Budget submitted with application (structured per programme's budget_template)
    budget_data JSONB DEFAULT '{}',
    -- budget_data example:
    -- {
    --   "line_items": [
    --     {"category": "personnel", "description": "PI salary (50%)", "amount": 75000, "year": 1},
    --     {"category": "personnel", "description": "Research asst", "amount": 45000, "year": 1},
    --     {"category": "fringe", "description": "Benefits at 30%", "amount": 36000, "year": 1},
    --     {"category": "travel", "description": "Field visits", "amount": 8000, "year": 1},
    --     {"category": "supplies", "description": "Lab supplies", "amount": 12000, "year": 1},
    --     {"category": "indirect", "description": "IDC at 15%", "amount": 26400, "year": 1}
    --   ],
    --   "total_direct": 176000,
    --   "total_indirect": 26400,
    --   "total_requested": 202400,
    --   "cost_sharing": 50000,
    --   "budget_justification": "Personnel costs reflect..."
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_applications_tenant ON applications(tenant_id);
CREATE INDEX idx_applications_opportunity ON applications(funding_opportunity_id);
CREATE INDEX idx_applications_applicant ON applications(applicant_organisation_id);
CREATE INDEX idx_applications_status ON applications(tenant_id, status);
CREATE INDEX idx_applications_form_data ON applications USING GIN (form_data);
```

### Reviews

```sql
CREATE TABLE reviews (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    application_id UUID NOT NULL REFERENCES applications(id),
    reviewer_user_id UUID NOT NULL REFERENCES users(id),
    review_type VARCHAR(30) NOT NULL,         -- 'blind', 'open', 'panel'
    status VARCHAR(30) NOT NULL DEFAULT 'assigned',
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    -- Scores stored as JSONB array matching programme's scoring_rubric
    scores JSONB DEFAULT '[]',
    -- scores example:
    -- [
    --   {"criterion": "Significance", "score": 8, "max_score": 10, "weight": 0.30, "comments": "Strong need demonstrated"},
    --   {"criterion": "Approach", "score": 7, "max_score": 10, "weight": 0.25, "comments": "Methodology is sound but..."},
    --   {"criterion": "Innovation", "score": 9, "max_score": 10, "weight": 0.20, "comments": "Novel approach to..."},
    --   {"criterion": "Team", "score": 8, "max_score": 10, "weight": 0.15, "comments": "Experienced PI"},
    --   {"criterion": "Budget Justification", "score": 6, "max_score": 10, "weight": 0.10, "comments": "Travel costs seem high"}
    -- ]
    overall_score NUMERIC(5,2),               -- computed weighted average
    recommendation VARCHAR(30),               -- 'fund', 'fund_with_conditions', 'decline'
    overall_comments TEXT,
    UNIQUE(application_id, reviewer_user_id)
);

CREATE INDEX idx_reviews_tenant ON reviews(tenant_id);
CREATE INDEX idx_reviews_application ON reviews(application_id);
CREATE INDEX idx_reviews_reviewer ON reviews(reviewer_user_id);
```

### Awards & Financial Management

```sql
CREATE TABLE awards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    application_id UUID REFERENCES applications(id),
    grant_programme_id UUID NOT NULL REFERENCES grant_programmes(id),
    recipient_organisation_id UUID NOT NULL REFERENCES organisations(id),
    award_number VARCHAR(100) NOT NULL,
    federal_award_id VARCHAR(50),             -- FAIN
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
    indirect_cost_rate_type VARCHAR(30),
    -- Award-specific operational data: amendments, compliance state, subrecipients
    award_detail JSONB DEFAULT '{}',
    -- award_detail example:
    -- {
    --   "amendments": [
    --     {
    --       "number": 1, "type": "no_cost_extension", "date": "2026-09-01",
    --       "previous_end_date": "2027-03-31", "new_end_date": "2027-09-30",
    --       "approved_by": "uuid", "description": "6-month NCE approved"
    --     }
    --   ],
    --   "special_conditions": [
    --     {"condition": "Quarterly financial reporting required", "added_date": "2026-01-15"}
    --   ],
    --   "closeout": {
    --     "final_report_submitted": false,
    --     "final_financial_report_submitted": false,
    --     "equipment_disposition_complete": false,
    --     "ip_disclosure_complete": true
    --   }
    -- }
    -- Budget tracking: structured per programme's budget_template
    budget JSONB DEFAULT '{}',
    -- budget example:
    -- {
    --   "line_items": [
    --     {"category": "personnel", "budgeted": 75000, "modified": 75000, "expended": 62500, "remaining": 12500},
    --     {"category": "fringe", "budgeted": 36000, "modified": 36000, "expended": 30000, "remaining": 6000},
    --     ...
    --   ],
    --   "total_budgeted": 202400,
    --   "total_expended": 145200,
    --   "total_remaining": 57200,
    --   "burn_rate_monthly": 16133
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_awards_tenant ON awards(tenant_id);
CREATE INDEX idx_awards_recipient ON awards(recipient_organisation_id);
CREATE INDEX idx_awards_programme ON awards(grant_programme_id);
CREATE INDEX idx_awards_status ON awards(tenant_id, status);
CREATE INDEX idx_awards_fain ON awards(federal_award_id) WHERE federal_award_id IS NOT NULL;
CREATE INDEX idx_awards_budget ON awards USING GIN (budget);
```

### Financial Transactions

```sql
-- Transactions remain relational because they need strong integrity and aggregate queries
CREATE TABLE financial_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    award_id UUID NOT NULL REFERENCES awards(id),
    transaction_type VARCHAR(30) NOT NULL,    -- 'expenditure', 'drawdown', 'reimbursement', 'return'
    budget_category VARCHAR(50),              -- matches programme's budget category codes
    amount NUMERIC(15,2) NOT NULL,
    transaction_date DATE NOT NULL,
    description VARCHAR(500),
    reference_number VARCHAR(100),
    is_allowable BOOLEAN,
    disallowance_reason TEXT,
    posted_by UUID REFERENCES users(id),
    -- Transaction-specific metadata
    detail JSONB DEFAULT '{}',
    -- detail example:
    -- {
    --   "vendor": "Acme Lab Supplies Inc.",
    --   "invoice_number": "INV-2026-0451",
    --   "cost_type": "direct",
    --   "accounting_code": "GL-4500-100",
    --   "supporting_docs": ["doc-uuid-1", "doc-uuid-2"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fin_tx_award ON financial_transactions(award_id);
CREATE INDEX idx_fin_tx_date ON financial_transactions(award_id, transaction_date);
CREATE INDEX idx_fin_tx_type ON financial_transactions(tenant_id, transaction_type);
```

### Compliance & Subrecipients

```sql
CREATE TABLE compliance_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    award_id UUID NOT NULL REFERENCES awards(id),
    item_type VARCHAR(50) NOT NULL,           -- 'report_due', 'audit_requirement', 'subrecipient_monitoring', 'closeout_task'
    title VARCHAR(300) NOT NULL,
    regulation_reference VARCHAR(200),
    due_date DATE,
    status VARCHAR(30) NOT NULL DEFAULT 'pending',
    completed_at TIMESTAMPTZ,
    completed_by UUID REFERENCES users(id),
    -- Type-specific compliance data
    detail JSONB DEFAULT '{}',
    -- detail examples by type:
    --
    -- report_due:
    -- {"report_type": "sf425", "period_start": "2026-01-01", "period_end": "2026-03-31",
    --  "submission_url": "https://...", "submitted_at": null}
    --
    -- audit_requirement:
    -- {"audit_type": "single_audit", "fiscal_year_end": "2026-06-30",
    --  "total_federal_expenditure": 2500000, "threshold": 1000000,
    --  "fac_submission_date": null, "findings_count": 0}
    --
    -- subrecipient_monitoring:
    -- {"subrecipient_org_id": "uuid", "subaward_number": "SUB-001",
    --  "subaward_amount": 150000, "risk_level": "high",
    --  "monitoring_plan": "quarterly_site_visits",
    --  "last_monitoring_date": "2026-02-15",
    --  "findings": []}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_award ON compliance_items(award_id);
CREATE INDEX idx_compliance_status ON compliance_items(tenant_id, status, due_date);
CREATE INDEX idx_compliance_type ON compliance_items(tenant_id, item_type);
CREATE INDEX idx_compliance_detail ON compliance_items USING GIN (detail);
```

### Documents

```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    uploaded_by UUID NOT NULL REFERENCES users(id),
    file_name VARCHAR(500) NOT NULL,
    file_type VARCHAR(100),
    file_size_bytes BIGINT,
    storage_key VARCHAR(1000) NOT NULL,
    document_type VARCHAR(50),
    -- What this document is attached to
    owner_type VARCHAR(50) NOT NULL,          -- 'application', 'award', 'compliance_item', 'organisation'
    owner_id UUID NOT NULL,
    -- Document metadata
    metadata JSONB DEFAULT '{}',
    -- metadata example:
    -- {"version": 2, "tags": ["budget", "revised"], "supersedes": "doc-uuid-prev"}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_owner ON documents(owner_type, owner_id);
CREATE INDEX idx_documents_tenant ON documents(tenant_id);
```

### Audit Trail

```sql
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

### Prospecting & External Integration

```sql
CREATE TABLE opportunity_matches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    source VARCHAR(30) NOT NULL,              -- 'grants_gov', 'candid', 'manual'
    external_id VARCHAR(100),
    title VARCHAR(500) NOT NULL,
    funder_name VARCHAR(300),
    amount_range NUMRANGE,                    -- PostgreSQL range type for floor/ceiling
    deadline TIMESTAMPTZ,
    relevance_score NUMERIC(5,4),
    status VARCHAR(30) DEFAULT 'new',
    -- Source-specific data (varies by Grants.gov vs Candid vs manual)
    source_data JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_matches_tenant_org ON opportunity_matches(tenant_id, organisation_id);
CREATE INDEX idx_matches_status ON opportunity_matches(tenant_id, status);

-- Notifications
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    recipient_user_id UUID NOT NULL REFERENCES users(id),
    notification_type VARCHAR(50) NOT NULL,
    subject VARCHAR(500) NOT NULL,
    body TEXT,
    related_type VARCHAR(50),
    related_id UUID,
    is_read BOOLEAN DEFAULT false,
    sent_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_user ON notifications(recipient_user_id, is_read);
```

---

## JSONB Query Examples

```sql
-- Find applications where population served exceeds 10,000
SELECT a.id, a.title, a.form_data->>'target_population' AS target_pop,
       (a.form_data->>'population_served')::integer AS pop_served
FROM applications a
WHERE a.tenant_id = '...'
  AND (a.form_data->>'population_served')::integer > 10000;

-- Find awards with budget variances (overspent categories)
SELECT a.id, a.award_number, a.title,
       item->>'category' AS category,
       (item->>'budgeted')::numeric AS budgeted,
       (item->>'expended')::numeric AS expended
FROM awards a,
     jsonb_array_elements(a.budget->'line_items') AS item
WHERE a.tenant_id = '...'
  AND a.status = 'active'
  AND (item->>'expended')::numeric > (item->>'budgeted')::numeric;

-- Find high-risk subrecipients across all awards
SELECT ci.id, ci.title, ci.detail->>'subaward_number' AS subaward,
       ci.detail->>'risk_level' AS risk_level,
       ci.detail->>'subaward_amount' AS amount,
       a.award_number
FROM compliance_items ci
JOIN awards a ON a.id = ci.award_id
WHERE ci.tenant_id = '...'
  AND ci.item_type = 'subrecipient_monitoring'
  AND ci.detail->>'risk_level' = 'high';

-- Validate application form_data against programme's JSON Schema
-- (done at application layer, but the schema is stored in the database)
SELECT gp.config->'application_schema' AS schema
FROM grant_programmes gp
JOIN funding_opportunities fo ON fo.grant_programme_id = gp.id
WHERE fo.id = '...';  -- opportunity ID from the application
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy & Organisations | 2 | tenants, organisations (profile in JSONB) |
| Users & Access Control | 1 | users (roles/permissions in JSONB `access` field) |
| Programmes & Opportunities | 2 | grant_programmes (config in JSONB), funding_opportunities |
| Applications | 1 | applications (form_data + budget_data in JSONB) |
| Reviews | 1 | reviews (scores in JSONB) |
| Awards & Finance | 2 | awards (budget/amendments in JSONB), financial_transactions |
| Compliance | 1 | compliance_items (type-specific detail in JSONB) |
| Documents | 1 | documents (metadata in JSONB) |
| Audit Trail | 1 | audit_log |
| Prospecting & Notifications | 2 | opportunity_matches, notifications |
| **Total** | **14** | Significantly fewer tables; complexity shifts to JSONB structures and JSON Schema validation |

---

## Key Design Decisions

1. **Programme config as the schema source**: Each `grant_programme` stores its application form schema, budget template, scoring rubric, compliance rules, and reporting templates in the `config` JSONB column. This means adding a new programme with entirely different fields requires zero schema migrations -- just inserting a new row with a different JSON Schema.

2. **Relational columns for queryable, standard fields**: Fields that are queried frequently (UEI, ALN, status, amounts, dates) or required by federal standards remain as typed relational columns with proper indexes. JSONB is reserved for variable, programme-specific data.

3. **Financial transactions stay relational**: Despite the JSONB philosophy, financial transactions use a dedicated relational table because aggregate financial queries (SUM, GROUP BY category, date ranges) perform better on indexed columns, and financial data integrity is critical for compliance.

4. **Unified compliance_items table**: Rather than separate tables for reports, audits, subrecipient monitoring, and closeout tasks, a single `compliance_items` table with `item_type` and type-specific JSONB `detail` handles all compliance-related tracking. This reduces table count while maintaining queryability via GIN indexes.

5. **User access in JSONB**: Roles, organisation memberships, and programme-scoped permissions are stored in the user's `access` JSONB field rather than in junction tables. This simplifies the schema at the cost of requiring application-level enforcement for access control.

6. **Budget as JSONB on awards**: The budget is stored as a structured JSONB document on the award, with line items following the programme's budget template. Budget updates modify this document. Financial transactions are reconciled against the budget JSONB at the application layer.

7. **PostgreSQL NUMRANGE for opportunity amounts**: The `amount_range` column on `opportunity_matches` uses PostgreSQL's native range type for efficient "grants between $50K and $250K" queries.

8. **GIN indexes on all JSONB columns**: Every JSONB column used for querying has a GIN index. While these indexes are larger than B-tree, they enable containment queries (`@>`) and key-existence checks that are essential for filtering on dynamic fields.

9. **JSON Schema validation at application layer**: The database stores JSON Schemas in `grant_programmes.config`; the application layer validates `form_data`, `budget_data`, and `scores` against these schemas before writing. This provides type safety without database-level constraints on JSONB content.

10. **Opportunity config overrides**: Funding opportunities can override programme-level configuration via `config_overrides`, which is deep-merged with the programme's config. This supports seasonal or cycle-specific variations (e.g., different scoring rubric for a special round) without duplicating programme definitions.
