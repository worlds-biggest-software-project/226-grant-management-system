# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Grant Management System · Created: 2026-05-21

## Philosophy

This model treats every state change in the grant lifecycle as an immutable event. The event store is the single source of truth: when a grant application is submitted, a review score is recorded, an award is approved, a budget modification is made, or a compliance obligation is met, an event is appended to an immutable log. Current state is derived by replaying events or, more practically, maintained in materialised read models (projections) that are rebuilt from the event stream.

This architecture is a natural fit for grant management because regulatory compliance -- OMB 2 CFR Part 200, the GREAT Act, FFATA -- fundamentally requires answering the question "what happened, when, and by whom?" Event sourcing provides this by construction rather than by bolting on an audit trail after the fact. The append-only event store makes it impossible to silently alter historical records, which is critical for single audit compliance and federal reporting. Temporal queries ("what was the budget as of March 15?", "when was the application status changed to approved?") are first-class operations.

The trade-off is operational complexity. Developers must think in terms of commands and events rather than CRUD. Read models must be explicitly defined and kept in sync via event processors. Schema evolution requires careful versioning of event types. However, for a compliance-heavy domain where auditability is not optional, event sourcing eliminates the gap between "what the system shows now" and "what actually happened" -- a gap that causes real problems during audits.

**Best for:** Organisations where full audit trails are mandatory, temporal queries are frequent, and regulatory compliance (OMB Uniform Guidance, FFATA, Single Audit Act) demands provable, tamper-resistant records of every state change.

**Trade-offs:**
- (+) Complete, immutable audit trail by construction -- no separate audit logging needed
- (+) Temporal queries are trivial: replay events to any point in time
- (+) Natural fit for compliance domains where "what happened when" is critical
- (+) Events can feed AI/ML analytics pipelines for pattern detection
- (+) Independent scaling of writes (event ingestion) and reads (projections)
- (-) Higher development complexity: commands, events, projections, event handlers
- (-) Event schema evolution requires careful versioning and migration
- (-) Read model consistency is eventual, not immediate
- (-) Debugging requires understanding event replay, not just querying tables
- (-) Team must learn event-sourcing patterns; steeper onboarding curve

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OMB 2 CFR Part 200 (Uniform Guidance) | Every compliance-relevant action (expenditure approval, budget modification, subrecipient risk assessment) is an immutable event with actor, timestamp, and context; directly satisfies 200.302 financial management standards |
| Single Audit Act (Subpart F) | Event replay generates SEFA schedules at any point in time; auditors can verify expenditure history without relying on mutable state |
| FFATA / USASpending.gov | Federal reporting projections are built from event streams, ensuring reports are traceable to individual source events |
| GREAT Act / OMB M-24-11 | Events carry GREAT Act data element identifiers, enabling machine-readable export from the event store |
| GSDM | Read models for federal reporting are structured to match GSDM element categories |
| SAM.gov / UEI | Organisation registration events track UEI assignment and SAM.gov status changes over time |
| ISO 3166 | Jurisdiction data is stored in reference tables; jurisdiction assignment events track changes |
| XBRL Grants Taxonomy | Structured reporting projections can emit XBRL-aligned data from event streams |

---

## Event Store (Source of Truth)

```sql
-- The immutable event log: APPEND ONLY
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    stream_id UUID NOT NULL,                  -- aggregate root ID (application, award, etc.)
    stream_type VARCHAR(50) NOT NULL,         -- 'Application', 'Award', 'Organisation', 'Review', 'Budget'
    event_type VARCHAR(100) NOT NULL,         -- 'ApplicationSubmitted', 'ReviewScoreRecorded', 'AwardApproved', etc.
    event_version INTEGER NOT NULL,           -- schema version of this event type
    sequence_number BIGINT NOT NULL,          -- per-stream ordering
    payload JSONB NOT NULL,                   -- event-specific data
    metadata JSONB NOT NULL DEFAULT '{}',     -- {actor_id, ip_address, user_agent, correlation_id, causation_id}
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, sequence_number)
);

-- Optimised for stream replay (most common read pattern)
CREATE INDEX idx_events_stream ON events(stream_id, sequence_number);

-- Optimised for global event processing (projections, event handlers)
CREATE INDEX idx_events_tenant_time ON events(tenant_id, occurred_at);

-- Optimised for event-type-specific queries (e.g., all 'AwardApproved' events)
CREATE INDEX idx_events_type ON events(tenant_id, event_type, occurred_at);

-- Optimised for correlation tracking across aggregates
CREATE INDEX idx_events_correlation ON events((metadata->>'correlation_id')) WHERE metadata->>'correlation_id' IS NOT NULL;

-- Prevent mutation: use a trigger to block UPDATE and DELETE
CREATE OR REPLACE FUNCTION prevent_event_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'Events are immutable. UPDATE and DELETE are not permitted on the events table.';
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_prevent_event_mutation
    BEFORE UPDATE OR DELETE ON events
    FOR EACH ROW EXECUTE FUNCTION prevent_event_mutation();
```

### Event Type Catalogue

The following event types represent the complete grant lifecycle:

```sql
-- Example events with their payload structures:

-- === Organisation Events ===
-- OrganisationRegistered
-- payload: {name, legal_name, uei, ein, organisation_type, jurisdiction, address}

-- OrganisationUpdated
-- payload: {field, old_value, new_value}

-- SAMRegistrationVerified
-- payload: {uei, sam_status, expiry_date, verification_source}

-- === Grant Programme Events ===
-- ProgrammeCreated
-- payload: {title, funder_org_id, programme_type, aln, total_budget, fiscal_year}

-- FundingOpportunityPosted
-- payload: {programme_id, title, opportunity_number, close_date, award_floor, award_ceiling, total_available}

-- FundingOpportunityClosed
-- payload: {opportunity_id, applications_received_count}

-- === Application Events ===
-- ApplicationDrafted
-- payload: {opportunity_id, applicant_org_id, title, requested_amount}

-- ApplicationFormResponseSaved
-- payload: {field_key, field_label, value, section}

-- ApplicationSubmitted
-- payload: {submitted_by, submitted_at, requested_amount, project_dates}

-- ApplicationWithdrawn
-- payload: {withdrawn_by, reason}

-- === Review Events ===
-- ReviewPanelCreated
-- payload: {opportunity_id, name, review_type}

-- ReviewerAssigned
-- payload: {panel_id, application_id, reviewer_id}

-- ReviewerRecused
-- payload: {panel_id, application_id, reviewer_id, reason}

-- ReviewScoreRecorded
-- payload: {assignment_id, criterion, score, max_score, weight, comments}

-- ReviewCompleted
-- payload: {assignment_id, overall_score, recommendation}

-- === Award Events ===
-- AwardApproved
-- payload: {application_id, programme_id, recipient_org_id, award_number, amount, start_date, end_date, indirect_rate, indirect_rate_type, aln}

-- AwardAmended
-- payload: {amendment_type, description, previous_amount, new_amount, previous_end_date, new_end_date, approved_by}

-- AwardSuspended
-- payload: {reason, suspended_by, effective_date}

-- AwardClosed
-- payload: {close_type, final_amount, closeout_date}

-- === Budget Events ===
-- BudgetLineItemCreated
-- payload: {award_id, category_code, category_name, description, amount, fiscal_year}

-- BudgetLineItemModified
-- payload: {line_item_id, previous_amount, new_amount, reason}

-- ExpenditureRecorded
-- payload: {award_id, category_code, amount, transaction_date, description, reference_number}

-- ExpenditureFlaggedDisallowed
-- payload: {transaction_id, reason, regulation_reference, flagged_by}

-- DrawdownRequested
-- payload: {award_id, amount, period_start, period_end}

-- DrawdownDisbursed
-- payload: {award_id, amount, disbursement_date, reference}

-- === Compliance Events ===
-- ComplianceObligationCreated
-- payload: {award_id, requirement_type, regulation_reference, due_date}

-- ComplianceObligationSubmitted
-- payload: {obligation_id, submitted_by, submission_date}

-- ComplianceObligationAccepted
-- payload: {obligation_id, accepted_by, acceptance_date}

-- ComplianceObligationOverdue
-- payload: {obligation_id, days_overdue}

-- SubrecipientAdded
-- payload: {award_id, org_id, subaward_number, amount, start_date, end_date}

-- SubrecipientRiskAssessed
-- payload: {subrecipient_id, risk_level, assessment_notes, monitoring_frequency}

-- SingleAuditRequired
-- payload: {org_id, fiscal_year_end, total_federal_expenditure}

-- SingleAuditSubmitted
-- payload: {org_id, fiscal_year_end, findings_count, material_weakness, fac_submission_date}

-- === Document Events ===
-- DocumentUploaded
-- payload: {file_name, file_type, file_size, storage_key, document_type}

-- DocumentAttached
-- payload: {document_id, target_type, target_id}

-- === Federal Reporting Events ===
-- FederalReportGenerated
-- payload: {report_type, period_start, period_end, awards_included}

-- FederalReportSubmitted
-- payload: {report_id, submitted_to, submitted_at}

-- FederalReportAccepted
-- payload: {report_id, accepted_at, response_details}
```

---

## Command Handlers (Write Side)

```sql
-- Commands are validated and produce events. This is the "write model."
-- Command validation uses lightweight aggregate state tables.

-- Aggregate version tracking (optimistic concurrency control)
CREATE TABLE aggregate_versions (
    stream_id UUID PRIMARY KEY,
    stream_type VARCHAR(50) NOT NULL,
    tenant_id UUID NOT NULL,
    current_version BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_agg_versions_tenant ON aggregate_versions(tenant_id, stream_type);

-- Example: SubmitApplication command handler (pseudocode in PL/pgSQL)
CREATE OR REPLACE FUNCTION handle_submit_application(
    p_tenant_id UUID,
    p_application_id UUID,
    p_submitted_by UUID,
    p_actor_id UUID,
    p_ip_address INET
) RETURNS UUID AS $$
DECLARE
    v_current_version BIGINT;
    v_event_id UUID;
    v_app_status TEXT;
BEGIN
    -- Load current aggregate state
    SELECT current_version INTO v_current_version
    FROM aggregate_versions WHERE stream_id = p_application_id
    FOR UPDATE;  -- pessimistic lock for this aggregate

    -- Validate: check current status from latest projection
    SELECT status INTO v_app_status
    FROM applications_read WHERE id = p_application_id;

    IF v_app_status != 'draft' THEN
        RAISE EXCEPTION 'Application % is not in draft status (current: %)', p_application_id, v_app_status;
    END IF;

    -- Append event
    INSERT INTO events (tenant_id, stream_id, stream_type, event_type, event_version, sequence_number, payload, metadata)
    VALUES (
        p_tenant_id,
        p_application_id,
        'Application',
        'ApplicationSubmitted',
        1,
        v_current_version + 1,
        jsonb_build_object(
            'submitted_by', p_submitted_by,
            'submitted_at', now()
        ),
        jsonb_build_object(
            'actor_id', p_actor_id,
            'ip_address', p_ip_address::text,
            'correlation_id', gen_random_uuid()
        )
    )
    RETURNING id INTO v_event_id;

    -- Update aggregate version
    UPDATE aggregate_versions SET current_version = v_current_version + 1, updated_at = now()
    WHERE stream_id = p_application_id;

    RETURN v_event_id;
END;
$$ LANGUAGE plpgsql;
```

---

## Read Models (Projections)

Read models are materialised views updated by event processors. Each projection is optimised for a specific query pattern.

### Application Read Model

```sql
CREATE TABLE applications_read (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    funding_opportunity_id UUID NOT NULL,
    applicant_organisation_id UUID NOT NULL,
    applicant_organisation_name VARCHAR(500),
    application_number VARCHAR(50),
    title VARCHAR(500),
    abstract TEXT,
    requested_amount NUMERIC(15,2),
    project_start_date DATE,
    project_end_date DATE,
    status VARCHAR(30) NOT NULL,
    submitted_by UUID,
    submitted_at TIMESTAMPTZ,
    decision_date DATE,
    average_review_score NUMERIC(5,2),
    review_count INTEGER DEFAULT 0,
    last_event_sequence BIGINT NOT NULL,      -- tracks projection currency
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_app_read_tenant_status ON applications_read(tenant_id, status);
CREATE INDEX idx_app_read_opportunity ON applications_read(funding_opportunity_id);
CREATE INDEX idx_app_read_applicant ON applications_read(applicant_organisation_id);
```

### Award Read Model

```sql
CREATE TABLE awards_read (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    application_id UUID,
    grant_programme_id UUID NOT NULL,
    programme_title VARCHAR(500),
    recipient_organisation_id UUID NOT NULL,
    recipient_name VARCHAR(500),
    recipient_uei VARCHAR(12),
    award_number VARCHAR(100) NOT NULL,
    federal_award_id VARCHAR(50),
    title VARCHAR(500),
    award_amount NUMERIC(15,2) NOT NULL,
    obligated_amount NUMERIC(15,2) DEFAULT 0,
    disbursed_amount NUMERIC(15,2) DEFAULT 0,
    total_expenditures NUMERIC(15,2) DEFAULT 0,
    budget_variance_pct NUMERIC(5,2),         -- computed from events
    currency CHAR(3) DEFAULT 'USD',
    award_date DATE,
    start_date DATE,
    end_date DATE,
    status VARCHAR(30) NOT NULL,
    indirect_cost_rate NUMERIC(5,4),
    aln VARCHAR(10),
    amendment_count INTEGER DEFAULT 0,
    subrecipient_count INTEGER DEFAULT 0,
    compliance_obligations_pending INTEGER DEFAULT 0,
    compliance_obligations_overdue INTEGER DEFAULT 0,
    last_event_sequence BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_award_read_tenant ON awards_read(tenant_id, status);
CREATE INDEX idx_award_read_recipient ON awards_read(recipient_organisation_id);
CREATE INDEX idx_award_read_programme ON awards_read(grant_programme_id);
```

### Budget & Financial Read Model

```sql
CREATE TABLE budget_summary_read (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    award_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    category_code VARCHAR(20) NOT NULL,
    category_name VARCHAR(200) NOT NULL,
    is_direct_cost BOOLEAN NOT NULL,
    budgeted_amount NUMERIC(15,2) NOT NULL,
    modified_amount NUMERIC(15,2),
    expended_amount NUMERIC(15,2) DEFAULT 0,
    disallowed_amount NUMERIC(15,2) DEFAULT 0,
    remaining_amount NUMERIC(15,2),           -- computed: (modified or budgeted) - expended
    variance_pct NUMERIC(5,2),
    last_event_sequence BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_budget_read_award ON budget_summary_read(award_id);
CREATE UNIQUE INDEX idx_budget_read_award_cat ON budget_summary_read(award_id, category_code);
```

### Compliance Dashboard Read Model

```sql
CREATE TABLE compliance_dashboard_read (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    award_id UUID NOT NULL,
    award_number VARCHAR(100),
    recipient_name VARCHAR(500),
    obligation_type VARCHAR(300),
    regulation_reference VARCHAR(200),
    due_date DATE,
    status VARCHAR(30) NOT NULL,
    days_until_due INTEGER,                   -- computed, refreshed periodically
    completed_at TIMESTAMPTZ,
    last_event_sequence BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_compliance_dash_tenant ON compliance_dashboard_read(tenant_id, status);
CREATE INDEX idx_compliance_dash_due ON compliance_dashboard_read(tenant_id, due_date) WHERE status IN ('pending', 'in_progress');
```

### Subrecipient Risk Read Model

```sql
CREATE TABLE subrecipient_risk_read (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    award_id UUID NOT NULL,
    organisation_id UUID NOT NULL,
    organisation_name VARCHAR(500),
    uei VARCHAR(12),
    subaward_number VARCHAR(100),
    subaward_amount NUMERIC(15,2),
    risk_level VARCHAR(20),
    risk_assessment_date DATE,
    monitoring_frequency VARCHAR(30),
    status VARCHAR(30),
    last_audit_findings INTEGER DEFAULT 0,
    last_event_sequence BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_subrisk_tenant ON subrecipient_risk_read(tenant_id, risk_level);
```

### Organisation Read Model

```sql
CREATE TABLE organisations_read (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    name VARCHAR(500) NOT NULL,
    legal_name VARCHAR(500),
    uei VARCHAR(12),
    ein VARCHAR(10),
    organisation_type VARCHAR(50),
    jurisdiction_country CHAR(2),
    jurisdiction_subdivision VARCHAR(6),
    sam_registration_status VARCHAR(30),
    sam_registration_expiry DATE,
    total_awards_count INTEGER DEFAULT 0,
    total_awards_amount NUMERIC(15,2) DEFAULT 0,
    total_active_subawards INTEGER DEFAULT 0,
    last_event_sequence BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_org_read_tenant ON organisations_read(tenant_id);
CREATE INDEX idx_org_read_uei ON organisations_read(uei) WHERE uei IS NOT NULL;
```

---

## Event Processing Infrastructure

```sql
-- Projection checkpoint tracking: which event each projection has processed up to
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_processed_event_id UUID,
    last_processed_at TIMESTAMPTZ,
    events_processed BIGINT DEFAULT 0,
    status VARCHAR(20) DEFAULT 'running',     -- 'running', 'paused', 'rebuilding'
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for failed event processing
CREATE TABLE event_processing_errors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(100) NOT NULL,
    event_id UUID NOT NULL REFERENCES events(id),
    error_message TEXT NOT NULL,
    error_details JSONB,
    retry_count INTEGER DEFAULT 0,
    last_retry_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Snapshots for long-lived aggregates (optional optimisation)
CREATE TABLE aggregate_snapshots (
    stream_id UUID NOT NULL,
    stream_type VARCHAR(50) NOT NULL,
    snapshot_version BIGINT NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY(stream_id, snapshot_version)
);
```

---

## Temporal Query Examples

```sql
-- "What was the award budget on March 15, 2026?"
-- Replay all BudgetLineItemCreated and BudgetLineItemModified events up to that date
SELECT
    e.payload->>'category_code' AS category,
    SUM(
        CASE
            WHEN e.event_type = 'BudgetLineItemCreated' THEN (e.payload->>'amount')::numeric
            WHEN e.event_type = 'BudgetLineItemModified' THEN
                (e.payload->>'new_amount')::numeric - (e.payload->>'previous_amount')::numeric
            ELSE 0
        END
    ) AS budget_as_of_date
FROM events e
WHERE e.stream_type = 'Award'
  AND e.stream_id = '...'  -- award ID
  AND e.event_type IN ('BudgetLineItemCreated', 'BudgetLineItemModified')
  AND e.occurred_at <= '2026-03-15T23:59:59Z'
GROUP BY e.payload->>'category_code';

-- "Show the complete history of application status changes"
SELECT
    e.event_type,
    e.occurred_at,
    e.metadata->>'actor_id' AS changed_by,
    e.payload
FROM events e
WHERE e.stream_id = '...'  -- application ID
  AND e.stream_type = 'Application'
  AND e.event_type IN (
    'ApplicationDrafted', 'ApplicationSubmitted', 'ApplicationWithdrawn',
    'AwardApproved'
  )
ORDER BY e.sequence_number;

-- "Which expenditures were flagged as disallowed in Q1 2026?"
SELECT
    e.stream_id AS award_id,
    e.payload->>'transaction_id' AS transaction_id,
    e.payload->>'reason' AS disallowance_reason,
    e.payload->>'regulation_reference' AS regulation,
    e.occurred_at,
    e.metadata->>'actor_id' AS flagged_by
FROM events e
WHERE e.tenant_id = '...'
  AND e.event_type = 'ExpenditureFlaggedDisallowed'
  AND e.occurred_at BETWEEN '2026-01-01' AND '2026-03-31'
ORDER BY e.occurred_at;
```

---

## Reference Data Tables (Shared Across Read/Write)

```sql
-- These are NOT event-sourced; they are stable reference data.

CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    tenant_type VARCHAR(30) NOT NULL,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE jurisdictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    name VARCHAR(200) NOT NULL,
    level VARCHAR(20) NOT NULL,
    parent_id UUID REFERENCES jurisdictions(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE budget_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    is_direct_cost BOOLEAN NOT NULL,
    parent_id UUID REFERENCES budget_categories(id),
    display_order INTEGER NOT NULL
);

CREATE TABLE compliance_requirement_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(300) NOT NULL,
    regulation_reference VARCHAR(200),
    frequency VARCHAR(30),
    applies_to VARCHAR(30),
    threshold_amount NUMERIC(15,2)
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email VARCHAR(320) NOT NULL,
    display_name VARCHAR(200) NOT NULL,
    password_hash VARCHAR(255),
    auth_provider VARCHAR(30) DEFAULT 'local',
    is_active BOOLEAN NOT NULL DEFAULT true,
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
    organisation_id UUID REFERENCES organisations_read(id),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | events (the single source of truth) |
| Command Infrastructure | 1 | aggregate_versions (optimistic concurrency) |
| Event Processing | 3 | projection_checkpoints, event_processing_errors, aggregate_snapshots |
| Read Models (Projections) | 6 | applications_read, awards_read, budget_summary_read, compliance_dashboard_read, subrecipient_risk_read, organisations_read |
| Reference Data | 7 | tenants, jurisdictions, budget_categories, compliance_requirement_types, users, roles, user_roles |
| **Total** | **18** | Significantly fewer tables than normalised; complexity shifts to event handlers and projections |

---

## Key Design Decisions

1. **Single events table**: All domain events share one table with `stream_type` and `event_type` discriminators. This simplifies event processing infrastructure (one subscription, one checkpoint) while `stream_id` + `sequence_number` guarantees per-aggregate ordering.

2. **Immutability enforced at database level**: A trigger physically prevents UPDATE and DELETE on the events table. This is the strongest guarantee possible that audit history cannot be tampered with -- stronger than application-level enforcement.

3. **JSONB payloads with versioned schemas**: Event payloads use JSONB for flexibility. The `event_version` field enables schema evolution: new event versions can add fields without breaking existing event processors, and old events can be up-cast by version-aware handlers.

4. **Metadata for compliance**: Every event carries `actor_id`, `ip_address`, and `correlation_id` in metadata. This satisfies OMB 2 CFR 200.302's requirement for records that "adequately identify the source and application of funds."

5. **Denormalised read models**: Projections like `awards_read` include fields from multiple aggregates (e.g., `recipient_name`, `programme_title`) to avoid JOINs at query time. This is the CQRS trade-off: write-side is normalised events, read-side is denormalised for performance.

6. **Projection checkpoints enable rebuild**: If a projection becomes corrupted or a new read model is needed, it can be rebuilt from scratch by replaying the event store from the beginning. The `projection_checkpoints` table tracks how far each projection has processed.

7. **Aggregate snapshots for performance**: Long-lived aggregates (e.g., an award with thousands of expenditure events) can be snapshotted periodically. Replay starts from the snapshot rather than from event zero, avoiding performance degradation over time.

8. **Reference data is NOT event-sourced**: Stable domain data (jurisdictions, budget categories, compliance requirement types) lives in regular tables. Event sourcing is reserved for business-critical state transitions where audit history matters.

9. **Compliance dashboard as a dedicated projection**: Rather than querying across multiple tables, the compliance dashboard read model pre-computes obligation status, due dates, and days-until-due for fast dashboard rendering.

10. **Events feed AI/ML pipelines**: The event stream is a natural input for ML models: expenditure pattern analysis for disallowance prediction, application scoring pattern analysis, subrecipient risk scoring, and compliance obligation forecasting can all consume the event stream without impacting the operational system.
