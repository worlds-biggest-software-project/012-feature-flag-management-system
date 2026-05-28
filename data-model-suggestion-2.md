# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Feature Flag Management System · Created: 2026-05-11

## Philosophy

In this model, every change to the system is recorded as an immutable event in a central event store. The event store is the single source of truth. Current state (what flags exist, what their targeting rules are, whether they are enabled) is derived by replaying events into materialised read models (projections). This is a CQRS (Command Query Responsibility Segregation) architecture: writes go through the event store; reads come from purpose-built projections optimised for specific query patterns.

This pattern is used by financial ledger systems, healthcare audit trails, and compliance-heavy platforms. The Microsoft CQRS and Event Sourcing architecture patterns document this approach extensively. For a feature flag platform, the event-sourced model directly solves the "73% of flags never removed" problem — because every flag state transition is recorded, the AI cleanup engine can analyse the full lifecycle history (created, enabled, rules changed, evaluation frequency declining, eventually abandoned) to score staleness with high confidence.

Unleash already stores every flag change as an event in its `events` table for audit purposes, but uses traditional CRUD tables as the source of truth. This model inverts that: events ARE the truth; CRUD-like tables are just cached projections.

**Best for:** Platforms where full audit trail is a regulatory requirement (SOC 2, HIPAA), where temporal queries are essential ("what was the flag state at 2am when the incident occurred?"), and where the AI cleanup engine needs rich lifecycle data to detect stale flags.

**Trade-offs:**
- (+) Complete, immutable audit trail by construction — not bolted on
- (+) Temporal queries are natural: replay events to any point in time
- (+) AI staleness detection has rich lifecycle data (creation, modifications, evaluation patterns)
- (+) Schema evolution is easier: add new event types without migrating existing data
- (+) Debugging production incidents by replaying flag state at any timestamp
- (-) Higher write amplification: every change writes to event store + updates projections
- (-) Eventual consistency between event store and projections (milliseconds, but not zero)
- (-) Projection rebuild can be slow for long-lived flags with many events
- (-) More complex to implement and reason about than CRUD
- (-) Storage grows monotonically (events are never deleted)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenFeature Evaluation API | Read-side projections serve evaluation queries in the OpenFeature format |
| OpenFeature OFREP | OFREP endpoints query materialised projections, not the event store |
| Martin Fowler's Toggle Taxonomy | `flag_type` is recorded in the `FlagCreated` event and propagated to projections |
| CQRS Pattern (Microsoft Architecture) | Strict separation between command (event append) and query (projection read) paths |
| Event Sourcing Pattern (Microsoft Architecture) | Append-only event store with projection rebuild capability |
| SOC 2 Audit Requirements | Immutable event log satisfies audit trail requirements without separate audit tables |
| CUPED / Sequential Testing | Experiment lifecycle events capture all state transitions; results stored as `ExperimentResultsComputed` events |

---

## Event Store (Write Side)

```sql
-- =============================================================
-- EVENT STORE — single source of truth
-- =============================================================

CREATE TABLE events (
    -- Event identity
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sequence_num    BIGSERIAL NOT NULL UNIQUE,               -- global ordering
    
    -- Aggregate identity (which entity this event belongs to)
    aggregate_type  VARCHAR(50) NOT NULL,                    -- flag, segment, experiment, project, environment
    aggregate_id    UUID NOT NULL,                           -- the entity's ID
    
    -- Event metadata
    event_type      VARCHAR(100) NOT NULL,                   -- FlagCreated, FlagEnabled, TargetingRuleAdded, etc.
    event_version   INTEGER NOT NULL DEFAULT 1,              -- schema version for this event type
    
    -- Event payload
    data            JSONB NOT NULL,                          -- event-specific structured data
    
    -- Context
    organization_id UUID NOT NULL,
    project_id      UUID,
    environment_id  UUID,
    
    -- Actor
    actor_id        UUID,                                    -- user who triggered the event
    actor_type      VARCHAR(50) NOT NULL DEFAULT 'user',     -- user, api_token, system, ai_agent
    
    -- Correlation
    correlation_id  UUID,                                    -- groups related events (e.g., bulk operations)
    causation_id    UUID,                                    -- the event that caused this event
    
    -- Timestamps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Primary query pattern: replay events for a specific aggregate
CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, sequence_num);

-- Query by organization and time (for audit log views)
CREATE INDEX idx_events_org_time ON events(organization_id, created_at DESC);

-- Query by event type (for projections that subscribe to specific events)
CREATE INDEX idx_events_type ON events(event_type, sequence_num);

-- Query by project (for project-scoped audit views)
CREATE INDEX idx_events_project ON events(project_id, created_at DESC);

-- Partition by month for storage management
-- CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### Event Type Catalogue

```sql
-- =============================================================
-- EVENT TYPES AND THEIR PAYLOADS (documented, not enforced in SQL)
-- =============================================================

-- FlagCreated:
-- { "key": "checkout-v2", "name": "Checkout V2", "flag_type": "release",
--   "value_type": "boolean", "description": "..." }

-- FlagUpdated:
-- { "changes": { "name": { "from": "old", "to": "new" } } }

-- FlagArchived:
-- { "reason": "Cleanup: fully rolled out for 30 days" }

-- FlagDeleted:
-- { "final_state": { ... full snapshot ... } }

-- FlagEnabled:
-- { "environment_id": "...", "enabled": true }

-- FlagDisabled:
-- { "environment_id": "...", "enabled": false, "reason": "kill switch" }

-- VariantAdded:
-- { "variant_key": "dark_mode", "value": "true", "sort_order": 1 }

-- VariantUpdated:
-- { "variant_key": "dark_mode", "changes": { "value": { "from": "true", "to": "false" } } }

-- VariantRemoved:
-- { "variant_key": "dark_mode" }

-- TargetingRuleAdded:
-- { "environment_id": "...", "priority": 1,
--   "segments": ["segment-uuid"], "variant_weights": { "control": 50000, "treatment": 50000 } }

-- TargetingRuleUpdated:
-- { "rule_id": "...", "changes": { "variant_weights": { ... } } }

-- TargetingRuleRemoved:
-- { "rule_id": "..." }

-- SegmentCreated:
-- { "name": "Beta Users", "match_type": "all",
--   "conditions": [ { "attribute": "plan", "operator": "equals", "value": "beta" } ] }

-- SegmentUpdated:
-- { "changes": { "conditions": { "added": [...], "removed": [...] } } }

-- ExperimentStarted:
-- { "flag_id": "...", "environment_id": "...", "hypothesis": "...",
--   "method": "frequentist", "alpha": 0.05, "use_cuped": true }

-- ExperimentResultsComputed:
-- { "experiment_id": "...", "results": [ { "variant": "control", "sample_size": 5000,
--   "mean": 0.042, "p_value": 0.031, "is_significant": true } ] }

-- ExperimentCompleted:
-- { "experiment_id": "...", "winning_variant": "treatment_a", "summary": "..." }

-- OverrideSet:
-- { "environment_id": "...", "context_key": "user-123", "context_kind": "user",
--   "variant_key": "treatment_a" }

-- OverrideRemoved:
-- { "environment_id": "...", "context_key": "user-123", "context_kind": "user" }

-- FlagEvaluationBatchRecorded:
-- { "environment_id": "...", "date": "2026-05-11",
--   "evaluations": 45230, "unique_contexts": 12500,
--   "variant_counts": { "control": 22615, "treatment": 22615 } }
```

## Read-Side Projections

```sql
-- =============================================================
-- PROJECTIONS (materialised read models, rebuilt from events)
-- =============================================================

-- Projection metadata: tracks which events each projection has consumed
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_sequence_num BIGINT NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ----- Flag State Projection (current state of all flags) -----

CREATE TABLE proj_flags (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    organization_id UUID NOT NULL,
    key             VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    flag_type       VARCHAR(20) NOT NULL,
    value_type      VARCHAR(20) NOT NULL,
    tags            TEXT[] DEFAULT '{}',
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    event_sequence  BIGINT NOT NULL                         -- last event that updated this row
);

CREATE UNIQUE INDEX idx_proj_flags_key ON proj_flags(project_id, key);
CREATE INDEX idx_proj_flags_project ON proj_flags(project_id);

-- ----- Flag Environment Projection -----

CREATE TABLE proj_flag_environments (
    id              UUID PRIMARY KEY,
    flag_id         UUID NOT NULL REFERENCES proj_flags(id),
    environment_id  UUID NOT NULL,
    is_enabled      BOOLEAN NOT NULL DEFAULT false,
    default_variant_key VARCHAR(100),
    updated_at      TIMESTAMPTZ NOT NULL,
    event_sequence  BIGINT NOT NULL
);

CREATE UNIQUE INDEX idx_proj_flag_env ON proj_flag_environments(flag_id, environment_id);

-- ----- Variants Projection -----

CREATE TABLE proj_variants (
    id              UUID PRIMARY KEY,
    flag_id         UUID NOT NULL REFERENCES proj_flags(id),
    key             VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    value           TEXT NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    event_sequence  BIGINT NOT NULL
);

CREATE UNIQUE INDEX idx_proj_variants_key ON proj_variants(flag_id, key);

-- ----- Targeting Rules Projection -----

CREATE TABLE proj_targeting_rules (
    id              UUID PRIMARY KEY,
    flag_id         UUID NOT NULL REFERENCES proj_flags(id),
    environment_id  UUID NOT NULL,
    priority        INTEGER NOT NULL,
    segment_ids     UUID[] NOT NULL DEFAULT '{}',
    variant_weights JSONB NOT NULL DEFAULT '{}',
    -- Example: {"control": 50000, "treatment_a": 30000, "treatment_b": 20000}
    updated_at      TIMESTAMPTZ NOT NULL,
    event_sequence  BIGINT NOT NULL
);

CREATE INDEX idx_proj_rules_flag_env ON proj_targeting_rules(flag_id, environment_id);

-- ----- Segments Projection -----

CREATE TABLE proj_segments (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    match_type      VARCHAR(10) NOT NULL DEFAULT 'all',
    conditions      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"attribute": "plan", "operator": "equals", "value": "enterprise"}]
    updated_at      TIMESTAMPTZ NOT NULL,
    event_sequence  BIGINT NOT NULL
);

CREATE INDEX idx_proj_segments_project ON proj_segments(project_id);

-- ----- Overrides Projection -----

CREATE TABLE proj_overrides (
    id              UUID PRIMARY KEY,
    flag_id         UUID NOT NULL REFERENCES proj_flags(id),
    environment_id  UUID NOT NULL,
    context_key     VARCHAR(255) NOT NULL,
    context_kind    VARCHAR(50) NOT NULL DEFAULT 'user',
    variant_key     VARCHAR(100) NOT NULL,
    event_sequence  BIGINT NOT NULL
);

CREATE UNIQUE INDEX idx_proj_overrides_lookup ON proj_overrides(flag_id, environment_id, context_kind, context_key);

-- ----- Experiment Projection -----

CREATE TABLE proj_experiments (
    id              UUID PRIMARY KEY,
    flag_id         UUID NOT NULL REFERENCES proj_flags(id),
    environment_id  UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    hypothesis      TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    statistical_method VARCHAR(20) NOT NULL DEFAULT 'frequentist',
    significance_level DECIMAL(4,3) NOT NULL DEFAULT 0.050,
    use_cuped       BOOLEAN NOT NULL DEFAULT false,
    use_sequential  BOOLEAN NOT NULL DEFAULT false,
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    created_by      UUID,
    latest_results  JSONB,                                  -- most recent computed results
    event_sequence  BIGINT NOT NULL
);

CREATE INDEX idx_proj_experiments_flag ON proj_experiments(flag_id);
CREATE INDEX idx_proj_experiments_status ON proj_experiments(status);

-- ----- Flag Lifecycle Projection (AI staleness analysis) -----

CREATE TABLE proj_flag_lifecycle (
    flag_id         UUID PRIMARY KEY REFERENCES proj_flags(id),
    total_events    INTEGER NOT NULL DEFAULT 0,              -- how many events this flag has
    first_enabled_at TIMESTAMPTZ,
    last_modified_at TIMESTAMPTZ,
    last_evaluated_at TIMESTAMPTZ,
    days_since_last_change INTEGER,
    evaluation_trend VARCHAR(20),                            -- increasing, stable, declining, zero
    staleness_score  DECIMAL(5,2),
    is_stale         BOOLEAN NOT NULL DEFAULT false,
    lifecycle_phase  VARCHAR(20),                            -- new, active, stable, declining, stale, archived
    event_sequence   BIGINT NOT NULL
);

CREATE INDEX idx_proj_lifecycle_stale ON proj_flag_lifecycle(is_stale) WHERE is_stale = true;
CREATE INDEX idx_proj_lifecycle_phase ON proj_flag_lifecycle(lifecycle_phase);
```

## Supporting Tables (Non-Event-Sourced)

```sql
-- =============================================================
-- REFERENCE / CONFIGURATION TABLES
-- (These are NOT event-sourced — they're configuration, not domain state)
-- =============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE organization_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, user_id)
);

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, slug)
);

CREATE TABLE environments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    is_protected    BOOLEAN NOT NULL DEFAULT false,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (project_id, slug)
);

CREATE TABLE api_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    environment_id  UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    token_hash      VARCHAR(255) NOT NULL UNIQUE,
    token_prefix    VARCHAR(10) NOT NULL,
    token_type      VARCHAR(20) NOT NULL DEFAULT 'server',
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    url             VARCHAR(2048) NOT NULL,
    secret          VARCHAR(255),
    events          TEXT[] NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Snapshot Store (Performance Optimisation)

```sql
-- =============================================================
-- SNAPSHOTS (periodic checkpoints to speed up replay)
-- =============================================================

CREATE TABLE aggregate_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    snapshot_data   JSONB NOT NULL,                          -- full aggregate state
    at_sequence_num BIGINT NOT NULL,                        -- snapshot taken at this event
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_snapshots_aggregate ON aggregate_snapshots(aggregate_type, aggregate_id, at_sequence_num DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | events (partitioned by month) |
| Projection Tracking | 1 | projection_checkpoints |
| Flag Projections | 7 | proj_flags, proj_flag_environments, proj_variants, proj_targeting_rules, proj_segments, proj_overrides, proj_flag_lifecycle |
| Experiment Projections | 1 | proj_experiments |
| Reference / Config | 7 | organizations, users, organization_memberships, projects, environments, api_tokens, webhooks |
| Snapshots | 1 | aggregate_snapshots |
| **Total** | **18** | Plus monthly event partitions |

---

## Key Design Decisions

1. **Single `events` table vs. per-aggregate-type tables** — A single events table with `aggregate_type` discrimination keeps the event store simple and allows cross-aggregate queries (e.g., "show me all events in this project in the last hour"). The trade-off is that per-aggregate replay requires filtering by `aggregate_type + aggregate_id`, which is efficiently handled by the composite index.

2. **Projections are denormalized read models** — Unlike Suggestion 1's normalized tables, projections store pre-joined data. `proj_targeting_rules` includes `segment_ids` as a UUID array and `variant_weights` as JSONB rather than using junction tables. This makes flag evaluation reads faster (fewer JOINs) at the cost of more complex projection update logic.

3. **`event_sequence` on every projection row** — Every projection row tracks the last event that updated it. This enables detecting stale projections and supporting projection rebuilds that skip already-processed events.

4. **Correlation and causation IDs** — `correlation_id` groups events triggered by the same user action (e.g., bulk-enabling 50 flags). `causation_id` links events in a causal chain (e.g., `ExperimentCompleted` causes `FlagEnabled` for the winning variant). These are essential for AI analysis of flag lifecycle patterns.

5. **Snapshots every N events** — For flags with long histories (hundreds of events), replaying from the beginning is expensive. Periodic snapshots in `aggregate_snapshots` allow replay to start from the most recent snapshot rather than event #1.

6. **Flag lifecycle projection is derived from event patterns** — The `proj_flag_lifecycle` table computes `lifecycle_phase`, `evaluation_trend`, and `staleness_score` by analysing the pattern of events and evaluation data. This powers the AI cleanup engine without requiring the AI to replay all events itself.

7. **Reference tables are NOT event-sourced** — Organizations, users, and projects use traditional CRUD. These are configuration entities, not domain entities. Event-sourcing everything would add complexity without audit value — user account changes don't need the same temporal analysis as flag changes.

8. **No separate audit table** — The event store IS the audit log. `audit_events` is replaced by querying `events` filtered by `organization_id` and `created_at`. This eliminates the data duplication present in CRUD + audit-log architectures.

## Example Queries

### Reconstruct flag state at a specific point in time

```sql
-- What was the state of flag 'checkout-v2' at 2026-05-10 02:00 UTC?
SELECT data, event_type, created_at
FROM events
WHERE aggregate_type = 'flag'
  AND aggregate_id = :flag_id
  AND created_at <= '2026-05-10 02:00:00+00'
ORDER BY sequence_num ASC;
-- Application replays these events to reconstruct the state
```

### Find flags with declining evaluation trends (AI staleness input)

```sql
SELECT pf.key, pf.name, pl.lifecycle_phase, pl.staleness_score,
       pl.evaluation_trend, pl.last_evaluated_at, pl.days_since_last_change
FROM proj_flag_lifecycle pl
JOIN proj_flags pf ON pf.id = pl.flag_id
WHERE pl.evaluation_trend = 'declining'
  AND pl.lifecycle_phase IN ('declining', 'stale')
  AND pf.is_archived = false
ORDER BY pl.staleness_score DESC;
```

### Full event timeline for incident investigation

```sql
SELECT e.event_type, e.data, e.created_at,
       u.name AS actor_name, e.actor_type
FROM events e
LEFT JOIN users u ON u.id = e.actor_id
WHERE e.project_id = :project_id
  AND e.created_at BETWEEN '2026-05-10 01:00:00+00' AND '2026-05-10 04:00:00+00'
ORDER BY e.sequence_num ASC;
```

### Projection rebuild (in application code, not SQL)

```
1. Read last_sequence_num from projection_checkpoints for 'flag_state'
2. SELECT * FROM events WHERE sequence_num > last_sequence_num ORDER BY sequence_num ASC
3. For each event:
   a. Apply event to the appropriate projection table (INSERT/UPDATE/DELETE)
   b. Update projection_checkpoints.last_sequence_num
4. Commit transaction
```
