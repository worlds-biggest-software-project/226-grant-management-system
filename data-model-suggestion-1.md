# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Grant Management System · Created: 2026-05-21

## Philosophy

This model follows classical third-normal-form (3NF) relational design, with a dedicated table for every distinct concept in the grant management domain. Every entity -- organisations, grant programmes, applications, reviews, awards, budgets, compliance obligations, subrecipients -- gets its own table with strong foreign key constraints. Junction tables handle many-to-many relationships (e.g., reviewers assigned to applications, documents attached to awards). Reference data (jurisdictions, compliance requirement types, budget categories) is stored in dedicated lookup tables aligned with federal standards (OMB 2 CFR Part 200 cost categories, Assistance Listing Numbers, UEI identifiers).

This approach prioritises data integrity and query flexibility. Complex cross-entity queries -- "show all awards to organisations in jurisdiction X with budget variances exceeding 10%" -- are natural JOINs across well-indexed tables. The schema is self-documenting: table and column names map directly to domain terminology, making it accessible to compliance auditors and report builders. Federal reporting outputs (FFATA, DATA Act, SEFA schedules) can be generated with straightforward SQL views.

The trade-off is schema rigidity. Adding a new concept (e.g., a jurisdiction-specific compliance field) requires a migration. The table count is high (~45-55 tables), which increases ORM mapping complexity and can slow early development. However, for a compliance-heavy domain where data integrity matters more than iteration speed, this is the industry-standard approach used by platforms like Euna, eCivis, and Salesforce NPSP.

**Best for:** Organisations that need rigorous data integrity, complex cross-entity reporting, and alignment with federal data standards (GREAT Act, GSDM, OMB Uniform Guidance).

**Trade-offs:**
- (+) Strongest referential integrity; every relationship is enforced by the database
- (+) Natural fit for federal reporting standards (GSDM, FFATA, SEFA)
- (+) Self-documenting schema maps directly to domain language
- (+) Complex cross-entity queries are straightforward JOINs
- (-) High table count (~50 tables) increases ORM and migration complexity
- (-) Adding jurisdiction-specific or programme-specific fields requires schema changes
- (-) Slower initial development compared to flexible-schema approaches
- (-) Many-to-many junction tables add query verbosity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OMB 2 CFR Part 200 (Uniform Guidance) | Cost categories table aligned with Subpart E cost principles; compliance_requirements table models Subpart F audit requirements; de minimis indirect rate (15%) stored as configurable programme parameter |
| GREAT Act / OMB M-24-11 | Core data elements mapped to dedicated columns matching the 540 standardised grant data fields; machine-readable export views built on normalised tables |
| GSDM (Governmentwide Spending Data Model) | Award, recipient, and agency tables mirror GSDM element categories (Award Characteristics, Recipient/Awardee Information, Award Amounts, Awarding Agency) |
| FFATA / USASpending.gov | Dedicated federal_report_submissions table tracks FFATA submissions; award fields include all required FFATA elements (UEI, award amount, project description, performance data) |
| SAM.gov / UEI | organisations.uei column stores GSA-assigned Unique Entity Identifier; sam_registrations table tracks registration status and expiry |
| Assistance Listing Numbers (ALNs) | Lookup table of federal programme identifiers; linked to awards for SEFA schedule generation |
| ISO 3166 | jurisdictions table uses ISO 3166-1 (country) and ISO 3166-2 (subdivision) codes |
| XBRL Grants Taxonomy | Column naming aligned with XBRL taxonomy labels where applicable for structured reporting |
| WCAG 2.2 / ADA Title II | Not a data model concern but informs UI field labelling conventions stored in form_fields |
| OpenAPI 3.1 / JSON Schema | API resource representations map 1:1 to normalised tables |

---

## Core Entity Tables

### Organisations & Identity

```sql
-- Organisations: grantseekers, grantmakers, subrecipients, government agencies
CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(500) NOT NULL,
    legal_name VARCHAR(500),
    uei VARCHAR(12),                          -- GSA Unique Entity Identifier (replaced DUNS)
    ein VARCHAR(10),                          -- Employer Identification Number
    organisation_type VARCHAR(50) NOT NULL,   -- 'nonprofit', 'foundation', 'government_agency', 'university', 'corporate'
    profit_structure VARCHAR(30),             -- 'nonprofit', 'for_profit', 'government'
    jurisdiction_id UUID REFERENCES jurisdictions(id),
    physical_address JSONB,                   -- {street, city, state, zip, country_code}
    mailing_address JSONB,
    website VARCHAR(500),
    phone VARCHAR(30),
    sam_registration_status VARCHAR(30),       -- 'active', 'expired', 'pending'
    sam_registration_expiry DATE,
    fiscal_year_end_month SMALLINT DEFAULT 12,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisations_tenant ON organisations(tenant_id);
CREATE INDEX idx_organisations_uei ON organisations(uei) WHERE uei IS NOT NULL;
CREATE INDEX idx_organisations_type ON organisations(tenant_id, organisation_type);
CREATE INDEX idx_organisations_name ON organisations(tenant_id, name);

-- Jurisdictions: ISO 3166 aligned
CREATE TABLE jurisdictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code CHAR(2) NOT NULL,            -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),              -- ISO 3166-2
    name VARCHAR(200) NOT NULL,
    level VARCHAR(20) NOT NULL,               -- 'country', 'state', 'county', 'city'
    parent_id UUID REFERENCES jurisdictions(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jurisdictions_country ON jurisdictions(country_code);
CREATE INDEX idx_jurisdictions_parent ON jurisdictions(parent_id);

-- Multi-tenant isolation
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    tenant_type VARCHAR(30) NOT NULL,         -- 'grantmaker', 'grantseeker', 'government'
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Users & Access Control

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email VARCHAR(320) NOT NULL,
    display_name VARCHAR(200) NOT NULL,
    password_hash VARCHAR(255),
    auth_provider VARCHAR(30) DEFAULT 'local', -- 'local', 'oidc', 'saml'
    auth_provider_id VARCHAR(500),
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);

CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,               -- 'admin', 'programme_officer', 'reviewer', 'applicant', 'finance_officer'
    description TEXT,
    permissions JSONB NOT NULL DEFAULT '[]',   -- ['applications.read', 'applications.review', 'awards.approve']
    is_system_role BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE user_roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    role_id UUID NOT NULL REFERENCES roles(id),
    organisation_id UUID REFERENCES organisations(id), -- scoped to an organisation (optional)
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by UUID REFERENCES users(id),
    UNIQUE(user_id, role_id, organisation_id)
);

CREATE TABLE user_organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    relationship VARCHAR(50) NOT NULL,        -- 'member', 'admin', 'contact', 'authorised_representative'
    is_primary BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, organisation_id, relationship)
);
```

### Grant Programmes & Funding Opportunities

```sql
-- Grant programmes: a funder's programme that issues grants
CREATE TABLE grant_programmes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    funder_organisation_id UUID NOT NULL REFERENCES organisations(id),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    programme_type VARCHAR(50) NOT NULL,      -- 'federal', 'state', 'foundation', 'corporate'
    aln VARCHAR(10),                          -- Assistance Listing Number (formerly CFDA)
    funding_source VARCHAR(50),               -- 'federal_appropriation', 'endowment', 'corporate_budget'
    total_budget NUMERIC(15,2),
    currency CHAR(3) DEFAULT 'USD',           -- ISO 4217
    fiscal_year INTEGER,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_programmes_tenant ON grant_programmes(tenant_id);
CREATE INDEX idx_programmes_funder ON grant_programmes(funder_organisation_id);
CREATE INDEX idx_programmes_aln ON grant_programmes(aln) WHERE aln IS NOT NULL;

-- Funding opportunities: a specific open call within a programme
CREATE TABLE funding_opportunities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    grant_programme_id UUID NOT NULL REFERENCES grant_programmes(id),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    eligibility_criteria TEXT,
    opportunity_number VARCHAR(100),          -- e.g., Grants.gov opportunity number
    grants_gov_id VARCHAR(50),                -- Grants.gov opportunity ID for integration
    posted_date DATE,
    close_date TIMESTAMPTZ,
    expected_awards_count INTEGER,
    award_floor NUMERIC(15,2),
    award_ceiling NUMERIC(15,2),
    total_funding_available NUMERIC(15,2),
    status VARCHAR(30) NOT NULL DEFAULT 'draft', -- 'draft', 'posted', 'closed', 'archived'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_opportunities_tenant ON funding_opportunities(tenant_id);
CREATE INDEX idx_opportunities_programme ON funding_opportunities(grant_programme_id);
CREATE INDEX idx_opportunities_status ON funding_opportunities(tenant_id, status);
CREATE INDEX idx_opportunities_close_date ON funding_opportunities(close_date);
```

### Applications & Review

```sql
-- Grant applications
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
        -- 'draft', 'submitted', 'under_review', 'approved', 'declined', 'withdrawn'
    submitted_at TIMESTAMPTZ,
    decision_date DATE,
    decision_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_applications_tenant ON applications(tenant_id);
CREATE INDEX idx_applications_opportunity ON applications(funding_opportunity_id);
CREATE INDEX idx_applications_applicant ON applications(applicant_organisation_id);
CREATE INDEX idx_applications_status ON applications(tenant_id, status);

-- Customisable application forms
CREATE TABLE form_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    funding_opportunity_id UUID REFERENCES funding_opportunities(id),
    name VARCHAR(200) NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    form_type VARCHAR(30) NOT NULL,           -- 'application', 'progress_report', 'final_report', 'budget'
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE form_fields (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_template_id UUID NOT NULL REFERENCES form_templates(id),
    field_key VARCHAR(100) NOT NULL,
    label VARCHAR(300) NOT NULL,
    field_type VARCHAR(30) NOT NULL,          -- 'text', 'textarea', 'number', 'currency', 'date', 'select', 'file', 'checkbox'
    is_required BOOLEAN DEFAULT false,
    display_order INTEGER NOT NULL,
    validation_rules JSONB,                   -- {"min_length": 10, "max_length": 5000, "options": ["a","b"]}
    help_text TEXT,
    section VARCHAR(200),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE form_responses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES applications(id),
    form_field_id UUID NOT NULL REFERENCES form_fields(id),
    value_text TEXT,
    value_numeric NUMERIC(15,4),
    value_date DATE,
    value_json JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(application_id, form_field_id)
);

-- Review assignments and scores
CREATE TABLE review_panels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    funding_opportunity_id UUID NOT NULL REFERENCES funding_opportunities(id),
    name VARCHAR(200) NOT NULL,
    review_type VARCHAR(30) NOT NULL,         -- 'blind', 'open', 'panel'
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE review_assignments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_panel_id UUID NOT NULL REFERENCES review_panels(id),
    application_id UUID NOT NULL REFERENCES applications(id),
    reviewer_user_id UUID NOT NULL REFERENCES users(id),
    status VARCHAR(30) NOT NULL DEFAULT 'assigned', -- 'assigned', 'in_progress', 'completed', 'recused'
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    UNIQUE(review_panel_id, application_id, reviewer_user_id)
);

CREATE TABLE scoring_rubrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_panel_id UUID NOT NULL REFERENCES review_panels(id),
    criterion_name VARCHAR(300) NOT NULL,
    description TEXT,
    max_score NUMERIC(5,2) NOT NULL,
    weight NUMERIC(5,4) DEFAULT 1.0,
    display_order INTEGER NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE review_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_assignment_id UUID NOT NULL REFERENCES review_assignments(id),
    scoring_rubric_id UUID NOT NULL REFERENCES scoring_rubrics(id),
    score NUMERIC(5,2),
    comments TEXT,
    scored_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(review_assignment_id, scoring_rubric_id)
);
```

### Awards & Financial Management

```sql
-- Grant awards
CREATE TABLE awards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    application_id UUID REFERENCES applications(id),
    grant_programme_id UUID NOT NULL REFERENCES grant_programmes(id),
    recipient_organisation_id UUID NOT NULL REFERENCES organisations(id),
    award_number VARCHAR(100) NOT NULL,
    federal_award_id VARCHAR(50),             -- FAIN for federal awards
    title VARCHAR(500) NOT NULL,
    award_amount NUMERIC(15,2) NOT NULL,
    obligated_amount NUMERIC(15,2) DEFAULT 0,
    disbursed_amount NUMERIC(15,2) DEFAULT 0,
    currency CHAR(3) DEFAULT 'USD',
    award_date DATE NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'active',
        -- 'active', 'suspended', 'closed', 'terminated'
    indirect_cost_rate NUMERIC(5,4),          -- e.g., 0.15 for 15% de minimis
    indirect_cost_rate_type VARCHAR(30),      -- 'negotiated', 'de_minimis', 'none'
    aln VARCHAR(10),                          -- Assistance Listing Number
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_awards_tenant ON awards(tenant_id);
CREATE INDEX idx_awards_recipient ON awards(recipient_organisation_id);
CREATE INDEX idx_awards_programme ON awards(grant_programme_id);
CREATE INDEX idx_awards_status ON awards(tenant_id, status);
CREATE INDEX idx_awards_fain ON awards(federal_award_id) WHERE federal_award_id IS NOT NULL;

-- Award amendments (modifications, supplements, no-cost extensions)
CREATE TABLE award_amendments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    award_id UUID NOT NULL REFERENCES awards(id),
    amendment_number INTEGER NOT NULL,
    amendment_type VARCHAR(50) NOT NULL,      -- 'budget_modification', 'no_cost_extension', 'supplement', 'scope_change'
    description TEXT NOT NULL,
    previous_amount NUMERIC(15,2),
    new_amount NUMERIC(15,2),
    previous_end_date DATE,
    new_end_date DATE,
    approved_by UUID REFERENCES users(id),
    approved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Budget line items
CREATE TABLE budget_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    is_direct_cost BOOLEAN NOT NULL,          -- true=direct, false=indirect (per 2 CFR 200 Subpart E)
    parent_id UUID REFERENCES budget_categories(id),
    display_order INTEGER NOT NULL
);

-- Pre-populate with OMB standard categories:
-- Personnel, Fringe Benefits, Travel, Equipment, Supplies, Contractual,
-- Construction, Other Direct Costs, Indirect Costs

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

-- Financial transactions (expenditures, drawdowns, reimbursements)
CREATE TABLE financial_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    award_id UUID NOT NULL REFERENCES awards(id),
    budget_line_item_id UUID REFERENCES budget_line_items(id),
    transaction_type VARCHAR(30) NOT NULL,    -- 'expenditure', 'drawdown', 'reimbursement', 'return'
    amount NUMERIC(15,2) NOT NULL,
    transaction_date DATE NOT NULL,
    description VARCHAR(500),
    reference_number VARCHAR(100),
    is_allowable BOOLEAN,                     -- compliance flag per 2 CFR 200
    disallowance_reason TEXT,
    posted_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fin_transactions_award ON financial_transactions(award_id);
CREATE INDEX idx_fin_transactions_date ON financial_transactions(award_id, transaction_date);
```

### Compliance & Subrecipient Management

```sql
-- Compliance requirements (template/instance pattern)
CREATE TABLE compliance_requirement_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(300) NOT NULL,
    description TEXT,
    regulation_reference VARCHAR(200),        -- e.g., '2 CFR 200.302', '2 CFR 200.332'
    frequency VARCHAR(30),                    -- 'one_time', 'quarterly', 'annual', 'as_needed'
    applies_to VARCHAR(30),                   -- 'all_federal', 'above_threshold', 'specific_programme'
    threshold_amount NUMERIC(15,2)            -- e.g., 1000000 for single audit
);

CREATE TABLE compliance_obligations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    award_id UUID NOT NULL REFERENCES awards(id),
    requirement_type_id UUID NOT NULL REFERENCES compliance_requirement_types(id),
    due_date DATE,
    status VARCHAR(30) NOT NULL DEFAULT 'pending',
        -- 'pending', 'in_progress', 'submitted', 'accepted', 'overdue'
    completed_at TIMESTAMPTZ,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_award ON compliance_obligations(award_id);
CREATE INDEX idx_compliance_status ON compliance_obligations(status, due_date);

-- Subrecipient tracking (per 2 CFR 200.332)
CREATE TABLE subrecipients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    award_id UUID NOT NULL REFERENCES awards(id),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    subaward_number VARCHAR(100) NOT NULL,
    subaward_amount NUMERIC(15,2) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    risk_level VARCHAR(20) DEFAULT 'standard', -- 'low', 'standard', 'high'
    risk_assessment_date DATE,
    risk_assessment_notes TEXT,
    monitoring_frequency VARCHAR(30),         -- 'quarterly', 'semi_annual', 'annual'
    status VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subrecipients_award ON subrecipients(award_id);
CREATE INDEX idx_subrecipients_org ON subrecipients(organisation_id);

-- Single audit tracking (2 CFR 200 Subpart F)
CREATE TABLE single_audits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    fiscal_year_end DATE NOT NULL,
    total_federal_expenditure NUMERIC(15,2),
    audit_status VARCHAR(30) NOT NULL,        -- 'required', 'in_progress', 'submitted', 'accepted'
    fac_submission_date DATE,                 -- Federal Audit Clearinghouse
    findings_count INTEGER DEFAULT 0,
    material_weakness BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- SEFA (Schedule of Expenditures of Federal Awards) line items
CREATE TABLE sefa_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    single_audit_id UUID NOT NULL REFERENCES single_audits(id),
    award_id UUID NOT NULL REFERENCES awards(id),
    aln VARCHAR(10) NOT NULL,
    programme_name VARCHAR(500),
    pass_through_entity VARCHAR(300),
    federal_expenditure NUMERIC(15,2) NOT NULL,
    is_major_programme BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Documents & Communications

```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    uploaded_by UUID NOT NULL REFERENCES users(id),
    file_name VARCHAR(500) NOT NULL,
    file_type VARCHAR(100),
    file_size_bytes BIGINT,
    storage_key VARCHAR(1000) NOT NULL,       -- S3 key or file path
    document_type VARCHAR(50),                -- 'proposal', 'budget', 'report', 'audit', 'correspondence', 'attachment'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Polymorphic document attachments
CREATE TABLE document_attachments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(id),
    attachable_type VARCHAR(50) NOT NULL,     -- 'application', 'award', 'compliance_obligation', 'subrecipient'
    attachable_id UUID NOT NULL,
    attached_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_doc_attachments_target ON document_attachments(attachable_type, attachable_id);

-- Notifications and communications
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    recipient_user_id UUID NOT NULL REFERENCES users(id),
    notification_type VARCHAR(50) NOT NULL,   -- 'deadline_reminder', 'status_change', 'review_assigned', 'report_due'
    subject VARCHAR(500) NOT NULL,
    body TEXT,
    related_type VARCHAR(50),                 -- 'application', 'award', 'compliance_obligation'
    related_id UUID,
    is_read BOOLEAN DEFAULT false,
    sent_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_user ON notifications(recipient_user_id, is_read);
```

### Audit Trail

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    user_id UUID REFERENCES users(id),
    action VARCHAR(30) NOT NULL,              -- 'create', 'update', 'delete', 'status_change', 'login'
    entity_type VARCHAR(50) NOT NULL,         -- 'application', 'award', 'financial_transaction', etc.
    entity_id UUID NOT NULL,
    changes JSONB,                            -- {"field": "status", "old": "draft", "new": "submitted"}
    ip_address INET,
    user_agent VARCHAR(500),
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_entity ON audit_log(tenant_id, entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, occurred_at);
CREATE INDEX idx_audit_user ON audit_log(user_id);
```

### Reporting & Federal Integration

```sql
-- Federal reporting submissions (FFATA, DATA Act)
CREATE TABLE federal_report_submissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    report_type VARCHAR(50) NOT NULL,         -- 'ffata', 'data_act', 'sf425', 'sf270', 'rppr'
    reporting_period_start DATE NOT NULL,
    reporting_period_end DATE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    submitted_at TIMESTAMPTZ,
    submitted_by UUID REFERENCES users(id),
    response_status VARCHAR(30),              -- from federal system
    response_details JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE federal_report_line_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID NOT NULL REFERENCES federal_report_submissions(id),
    award_id UUID NOT NULL REFERENCES awards(id),
    data_elements JSONB NOT NULL,             -- GSDM-aligned key-value pairs
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Grant prospecting / opportunity matching
CREATE TABLE saved_searches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    user_id UUID NOT NULL REFERENCES users(id),
    name VARCHAR(200) NOT NULL,
    search_criteria JSONB NOT NULL,           -- {keywords, categories, funders, amount_range, deadline_range}
    is_alert_active BOOLEAN DEFAULT false,
    last_run_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE opportunity_matches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saved_search_id UUID REFERENCES saved_searches(id),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    external_opportunity_id VARCHAR(100),     -- Grants.gov or Candid ID
    source VARCHAR(30) NOT NULL,              -- 'grants_gov', 'candid', 'manual'
    title VARCHAR(500) NOT NULL,
    funder_name VARCHAR(300),
    amount_low NUMERIC(15,2),
    amount_high NUMERIC(15,2),
    deadline TIMESTAMPTZ,
    relevance_score NUMERIC(5,4),             -- ML-generated match score
    status VARCHAR(30) DEFAULT 'new',         -- 'new', 'interested', 'applied', 'dismissed'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy & Identity | 3 | tenants, organisations, jurisdictions |
| Users & Access Control | 4 | users, roles, user_roles, user_organisations |
| Programmes & Opportunities | 2 | grant_programmes, funding_opportunities |
| Applications & Forms | 4 | applications, form_templates, form_fields, form_responses |
| Review & Scoring | 4 | review_panels, review_assignments, scoring_rubrics, review_scores |
| Awards & Finance | 5 | awards, award_amendments, budget_categories, budget_line_items, financial_transactions |
| Compliance & Subrecipients | 5 | compliance_requirement_types, compliance_obligations, subrecipients, single_audits, sefa_entries |
| Documents & Communications | 3 | documents, document_attachments, notifications |
| Audit Trail | 1 | audit_log |
| Reporting & Prospecting | 4 | federal_report_submissions, federal_report_line_items, saved_searches, opportunity_matches |
| **Total** | **35** | Core tables; additional lookup/reference tables may bring total to ~40-45 |

---

## Key Design Decisions

1. **Tenant-scoped everything**: Every major table includes `tenant_id` with indexes that lead on `tenant_id`, enabling row-level security (RLS) policies for multi-tenant isolation. This supports both shared-infrastructure SaaS and self-hosted single-tenant deployments.

2. **UEI as first-class field**: The GSA Unique Entity Identifier is stored directly on organisations rather than in a separate identifiers table, reflecting its mandatory status for all federal grant recipients since April 2022.

3. **Template/instance pattern for compliance**: Compliance requirement types are defined once (aligned with 2 CFR 200 sections) and instantiated per award as compliance_obligations. This allows the system to auto-generate obligation checklists when awards are created while supporting custom requirements.

4. **OMB-aligned budget categories**: The budget_categories table is pre-populated with standard OMB cost categories (Personnel, Fringe, Travel, Equipment, Supplies, Contractual, Construction, Other, Indirect) and supports hierarchical sub-categories via `parent_id`.

5. **Polymorphic document attachments**: Rather than adding document columns to every entity, a junction table with `attachable_type` and `attachable_id` allows documents to be attached to applications, awards, compliance obligations, or subrecipients without schema changes.

6. **Separation of programmes, opportunities, and awards**: This three-level hierarchy (programme > opportunity > application > award) mirrors how both federal agencies and foundations structure their grant-making, and aligns with the GSDM distinction between programme-level and award-level data.

7. **Explicit financial transaction table**: Rather than tracking only budget totals, individual transactions are recorded with allowability flags, enabling cost-level compliance checking per 2 CFR 200 Subpart E and budget variance reporting.

8. **SEFA as a first-class entity**: The Schedule of Expenditures of Federal Awards is modelled explicitly because it is required for single audit compliance above the $1M threshold and is a primary output of the system.

9. **Audit log with JSONB changes**: The audit trail captures before/after field values in JSONB, providing compliance-grade traceability without the overhead of a full event-sourcing pattern.

10. **Federal reporting as structured submissions**: FFATA and DATA Act reports are modelled as submissions with GSDM-aligned data elements in JSONB, balancing standards compliance with the evolving nature of federal reporting requirements.
