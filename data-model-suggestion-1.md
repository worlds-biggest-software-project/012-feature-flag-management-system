# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Feature Flag Management System · Created: 2026-05-11

## Philosophy

This model follows classical third-normal-form relational design. Every concept — flags, variants, targeting rules, segments, segment conditions, experiments, metrics, environments, projects — gets its own table with explicit foreign keys and junction tables. The schema is wide (many tables) but each table is narrow (few columns), with no denormalization or embedded JSON.

This is the approach used by Unleash (PostgreSQL with 30+ tables including `features`, `feature_strategies`, `feature_environments`, `segments`, `feature_tag`, `client_metrics_env`, `events`, `api_tokens`, `environments`, `project_environments`), and by Flagsmith's Django models (`features`, `feature_segments`, `feature_states`, `identities`, `traits`, `segments`, `segment_rules`, `segment_conditions`). The normalized design makes cross-entity queries straightforward and enforces referential integrity at the database level.

This approach is best when data integrity is paramount, complex cross-entity queries are frequent (e.g., "find all flags in project X that target segment Y in environment Z and have not been evaluated in 30 days"), and the team values explicit schema enforcement over flexibility.

**Best for:** Teams building a production-grade platform with strong governance, compliance (SOC 2 audit requirements), and complex cross-entity reporting needs.

**Trade-offs:**
- (+) Full referential integrity enforced at the database level
- (+) Complex cross-entity queries are natural JOINs, not application-level assembly
- (+) Schema is self-documenting — every relationship is explicit
- (+) Migration tooling (Knex, Flyway, Alembic) works well with normalized schemas
- (-) High table count (~35-40 tables) increases migration complexity
- (-) Adding a new targeting dimension (e.g., geo-location) requires a schema migration
- (-) Many JOINs for flag evaluation queries — must be carefully indexed
- (-) Rigid structure makes jurisdiction-specific or customer-specific extensions difficult

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenFeature Evaluation API | Flag, variant, and evaluation context tables map 1:1 to OpenFeature's `Flag`, `Variant`, and `EvaluationContext` concepts |
| OpenFeature OFREP | API endpoints serve data from normalized tables; `/ofrep/v1/evaluate/flags/{key}` queries `flags` + `targeting_rules` + `variants` |
| Martin Fowler's Toggle Taxonomy | `flag_type` enum column maps directly to Release, Experiment, Ops, Permission toggle categories |
| CUPED / Sequential Testing | Separate `experiment_metrics` and `experiment_results` tables store pre-experiment covariates and sequential test checkpoints |
| OAuth 2.0 / RBAC | Dedicated `roles`, `permissions`, `user_roles`, `project_memberships` tables for access control |
| ISO 3166 | `jurisdictions` reference table for geo-targeting rules |

---

## Organization & Tenant Management

```sql
-- =============================================================
-- ORGANIZATION & MULTI-TENANCY
-- =============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, pro, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),                          -- NULL for SSO-only users
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',  -- local, google, github, saml
    auth_provider_id VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE organization_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- owner, admin, member, viewer
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, user_id)
);

CREATE INDEX idx_org_memberships_org ON organization_memberships(organization_id);
CREATE INDEX idx_org_memberships_user ON organization_memberships(user_id);
```

## Projects & Environments

```sql
-- =============================================================
-- PROJECTS & ENVIRONMENTS
-- =============================================================

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

CREATE INDEX idx_projects_org ON projects(organization_id);

CREATE TABLE environments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,                  -- development, staging, production
    slug            VARCHAR(100) NOT NULL,
    color           VARCHAR(7),                             -- hex color for UI
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_protected    BOOLEAN NOT NULL DEFAULT false,         -- requires approval for changes
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (project_id, slug)
);

CREATE INDEX idx_environments_project ON environments(project_id);

CREATE TABLE api_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    environment_id  UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    token_hash      VARCHAR(255) NOT NULL UNIQUE,           -- SHA-256 of the actual token
    token_prefix    VARCHAR(10) NOT NULL,                   -- first 8 chars for identification
    token_type      VARCHAR(20) NOT NULL DEFAULT 'server',  -- server, client, admin
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_api_tokens_env ON api_tokens(environment_id);
CREATE INDEX idx_api_tokens_hash ON api_tokens(token_hash);
```

## Feature Flags

```sql
-- =============================================================
-- FEATURE FLAGS
-- =============================================================

CREATE TYPE flag_type AS ENUM ('release', 'experiment', 'ops', 'permission');
CREATE TYPE value_type AS ENUM ('boolean', 'string', 'number', 'json');

CREATE TABLE flags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    key             VARCHAR(255) NOT NULL,                   -- unique flag key within project
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    flag_type       flag_type NOT NULL DEFAULT 'release',    -- Fowler's taxonomy
    value_type      value_type NOT NULL DEFAULT 'boolean',
    tags            TEXT[] DEFAULT '{}',
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (project_id, key)
);

CREATE INDEX idx_flags_project ON flags(project_id);
CREATE INDEX idx_flags_key ON flags(project_id, key);
CREATE INDEX idx_flags_type ON flags(flag_type);
CREATE INDEX idx_flags_archived ON flags(is_archived) WHERE is_archived = false;

CREATE TABLE flag_variants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_id         UUID NOT NULL REFERENCES flags(id) ON DELETE CASCADE,
    key             VARCHAR(100) NOT NULL,                   -- variant identifier (e.g., 'control', 'treatment_a')
    name            VARCHAR(255) NOT NULL,
    value           TEXT NOT NULL,                           -- the actual value returned
    description     TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (flag_id, key)
);

CREATE INDEX idx_flag_variants_flag ON flag_variants(flag_id);

-- Per-environment flag state (enabled/disabled + default variant)
CREATE TABLE flag_environments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_id         UUID NOT NULL REFERENCES flags(id) ON DELETE CASCADE,
    environment_id  UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
    is_enabled      BOOLEAN NOT NULL DEFAULT false,
    default_variant_id UUID REFERENCES flag_variants(id),   -- fallback when no rules match
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (flag_id, environment_id)
);

CREATE INDEX idx_flag_env_flag ON flag_environments(flag_id);
CREATE INDEX idx_flag_env_env ON flag_environments(environment_id);
```

## Segments & Targeting Rules

```sql
-- =============================================================
-- SEGMENTS & TARGETING
-- =============================================================

CREATE TABLE segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    match_type      VARCHAR(10) NOT NULL DEFAULT 'all',     -- all (AND), any (OR)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_segments_project ON segments(project_id);

CREATE TYPE condition_operator AS ENUM (
    'equals', 'not_equals',
    'contains', 'not_contains',
    'starts_with', 'ends_with',
    'greater_than', 'less_than',
    'greater_than_or_equal', 'less_than_or_equal',
    'in_list', 'not_in_list',
    'matches_regex',
    'is_true', 'is_false',
    'exists', 'not_exists',
    'semver_gt', 'semver_lt', 'semver_eq'
);

CREATE TABLE segment_conditions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    segment_id      UUID NOT NULL REFERENCES segments(id) ON DELETE CASCADE,
    attribute       VARCHAR(255) NOT NULL,                  -- e.g., 'email', 'plan', 'country'
    operator        condition_operator NOT NULL,
    value           TEXT NOT NULL,                           -- comparison value (or JSON array for in_list)
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_segment_conditions_segment ON segment_conditions(segment_id);

-- Targeting rules link flags (in an environment) to segments or percentage rollouts
CREATE TABLE targeting_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_environment_id UUID NOT NULL REFERENCES flag_environments(id) ON DELETE CASCADE,
    name            VARCHAR(255),
    priority        INTEGER NOT NULL DEFAULT 0,             -- lower = higher priority
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_targeting_rules_flag_env ON targeting_rules(flag_environment_id);

-- Each rule can match one or more segments (AND logic between them)
CREATE TABLE targeting_rule_segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    targeting_rule_id UUID NOT NULL REFERENCES targeting_rules(id) ON DELETE CASCADE,
    segment_id      UUID NOT NULL REFERENCES segments(id) ON DELETE RESTRICT,
    UNIQUE (targeting_rule_id, segment_id)
);

-- Each rule distributes matched users across variants (percentage rollout)
CREATE TABLE targeting_rule_variants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    targeting_rule_id UUID NOT NULL REFERENCES targeting_rules(id) ON DELETE CASCADE,
    variant_id      UUID NOT NULL REFERENCES flag_variants(id) ON DELETE CASCADE,
    weight          INTEGER NOT NULL CHECK (weight >= 0 AND weight <= 100000),
    -- Weight is in basis points (100000 = 100%). Allows 0.001% precision.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rule_variants_rule ON targeting_rule_variants(targeting_rule_id);

-- Individual user overrides (bypass all targeting rules)
CREATE TABLE flag_overrides (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_environment_id UUID NOT NULL REFERENCES flag_environments(id) ON DELETE CASCADE,
    context_key     VARCHAR(255) NOT NULL,                  -- user ID or account ID
    context_kind    VARCHAR(50) NOT NULL DEFAULT 'user',    -- user, account, device
    variant_id      UUID NOT NULL REFERENCES flag_variants(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (flag_environment_id, context_key, context_kind)
);

CREATE INDEX idx_flag_overrides_lookup ON flag_overrides(flag_environment_id, context_kind, context_key);
```

## B2B Account Targeting

```sql
-- =============================================================
-- B2B ACCOUNT / COMPANY TARGETING
-- =============================================================

CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,                  -- customer's own account ID
    name            VARCHAR(255) NOT NULL,
    plan_tier       VARCHAR(100),
    mrr_cents       BIGINT,                                 -- monthly recurring revenue in cents
    employee_count  INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, external_id)
);

CREATE INDEX idx_accounts_org ON accounts(organization_id);

CREATE TABLE account_traits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    trait_key       VARCHAR(255) NOT NULL,
    trait_value     TEXT NOT NULL,
    trait_type      VARCHAR(20) NOT NULL DEFAULT 'string',  -- string, number, boolean
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (account_id, trait_key)
);

CREATE INDEX idx_account_traits_account ON account_traits(account_id);
CREATE INDEX idx_account_traits_lookup ON account_traits(trait_key, trait_value);
```

## Experimentation

```sql
-- =============================================================
-- EXPERIMENTATION
-- =============================================================

CREATE TYPE experiment_status AS ENUM ('draft', 'running', 'paused', 'completed', 'abandoned');
CREATE TYPE statistical_method AS ENUM ('frequentist', 'bayesian');

CREATE TABLE experiments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_environment_id UUID NOT NULL REFERENCES flag_environments(id) ON DELETE RESTRICT,
    name            VARCHAR(255) NOT NULL,
    hypothesis      TEXT,
    status          experiment_status NOT NULL DEFAULT 'draft',
    statistical_method statistical_method NOT NULL DEFAULT 'frequentist',
    significance_level DECIMAL(4,3) NOT NULL DEFAULT 0.050, -- alpha, typically 0.05
    minimum_sample_size INTEGER,
    use_cuped       BOOLEAN NOT NULL DEFAULT false,         -- CUPED variance reduction
    use_sequential  BOOLEAN NOT NULL DEFAULT false,         -- sequential testing / early stopping
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_experiments_flag_env ON experiments(flag_environment_id);
CREATE INDEX idx_experiments_status ON experiments(status);

CREATE TABLE experiment_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    event_name      VARCHAR(255) NOT NULL,                  -- the event to track
    metric_type     VARCHAR(50) NOT NULL,                   -- conversion, revenue, count, duration
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    minimum_detectable_effect DECIMAL(6,4),                 -- MDE as a proportion
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_experiment_metrics_exp ON experiment_metrics(experiment_id);

CREATE TABLE experiment_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    metric_id       UUID NOT NULL REFERENCES experiment_metrics(id) ON DELETE CASCADE,
    variant_id      UUID NOT NULL REFERENCES flag_variants(id) ON DELETE CASCADE,
    sample_size     BIGINT NOT NULL DEFAULT 0,
    conversions     BIGINT,                                 -- for conversion metrics
    sum_value       DECIMAL(20,6),                          -- for revenue/count metrics
    sum_squared     DECIMAL(30,6),                          -- for variance calculation
    mean            DECIMAL(20,6),
    variance        DECIMAL(20,6),
    confidence_lower DECIMAL(20,6),                         -- CI lower bound
    confidence_upper DECIMAL(20,6),                         -- CI upper bound
    p_value         DECIMAL(10,8),
    is_significant  BOOLEAN,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_experiment_results_exp ON experiment_results(experiment_id);
CREATE INDEX idx_experiment_results_metric ON experiment_results(metric_id);
```

## Flag Lifecycle & Stale Detection

```sql
-- =============================================================
-- FLAG LIFECYCLE & STALE DETECTION
-- =============================================================

CREATE TABLE flag_evaluation_stats (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_environment_id UUID NOT NULL REFERENCES flag_environments(id) ON DELETE CASCADE,
    date            DATE NOT NULL,
    total_evaluations BIGINT NOT NULL DEFAULT 0,
    unique_contexts BIGINT NOT NULL DEFAULT 0,
    variant_counts  JSONB NOT NULL DEFAULT '{}',            -- {"control": 4500, "treatment": 4500}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (flag_environment_id, date)
);

CREATE INDEX idx_eval_stats_date ON flag_evaluation_stats(flag_environment_id, date);

CREATE TABLE flag_code_references (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_id         UUID NOT NULL REFERENCES flags(id) ON DELETE CASCADE,
    repository      VARCHAR(500) NOT NULL,
    file_path       VARCHAR(1000) NOT NULL,
    line_number     INTEGER,
    snippet         TEXT,
    branch          VARCHAR(255) NOT NULL DEFAULT 'main',
    commit_sha      VARCHAR(40),
    scanned_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_code_refs_flag ON flag_code_references(flag_id);

CREATE TABLE flag_lifecycle (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_id         UUID NOT NULL REFERENCES flags(id) ON DELETE CASCADE,
    expected_lifetime_days INTEGER,                         -- how long the flag should live
    stale_after     TIMESTAMPTZ,                            -- when it becomes stale
    is_stale        BOOLEAN NOT NULL DEFAULT false,
    staleness_score DECIMAL(5,2),                           -- AI-computed score 0-100
    last_evaluated_at TIMESTAMPTZ,
    cleanup_pr_url  VARCHAR(500),                           -- link to auto-generated cleanup PR
    reviewed_at     TIMESTAMPTZ,
    reviewed_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (flag_id)
);

CREATE INDEX idx_lifecycle_stale ON flag_lifecycle(is_stale) WHERE is_stale = true;
```

## Audit Log

```sql
-- =============================================================
-- AUDIT LOG
-- =============================================================

CREATE TABLE audit_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    actor_id        UUID REFERENCES users(id),
    actor_type      VARCHAR(50) NOT NULL DEFAULT 'user',    -- user, api_token, system
    action          VARCHAR(100) NOT NULL,                  -- flag.created, flag.updated, experiment.started, etc.
    resource_type   VARCHAR(100) NOT NULL,                  -- flag, segment, experiment, environment
    resource_id     UUID NOT NULL,
    project_id      UUID REFERENCES projects(id),
    environment_id  UUID REFERENCES environments(id),
    before_state    JSONB,                                  -- snapshot before change
    after_state     JSONB,                                  -- snapshot after change
    metadata        JSONB DEFAULT '{}',
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_org_time ON audit_events(organization_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_events(resource_type, resource_id);
CREATE INDEX idx_audit_actor ON audit_events(actor_id);
CREATE INDEX idx_audit_action ON audit_events(action);

-- Partition audit_events by month for performance
-- (In production, use declarative partitioning on created_at)
```

## Approval Workflows

```sql
-- =============================================================
-- APPROVAL WORKFLOWS (for protected environments)
-- =============================================================

CREATE TYPE approval_status AS ENUM ('pending', 'approved', 'rejected', 'applied', 'cancelled');

CREATE TABLE change_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_environment_id UUID NOT NULL REFERENCES flag_environments(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    status          approval_status NOT NULL DEFAULT 'pending',
    change_payload  JSONB NOT NULL,                         -- the proposed changes
    requested_by    UUID NOT NULL REFERENCES users(id),
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    applied_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_change_requests_status ON change_requests(status) WHERE status = 'pending';
CREATE INDEX idx_change_requests_flag_env ON change_requests(flag_environment_id);
```

## Webhooks & Integrations

```sql
-- =============================================================
-- WEBHOOKS & INTEGRATIONS
-- =============================================================

CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    url             VARCHAR(2048) NOT NULL,
    secret          VARCHAR(255),                           -- HMAC signing secret
    events          TEXT[] NOT NULL,                        -- {'flag.created', 'flag.updated', ...}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_triggered_at TIMESTAMPTZ,
    failure_count   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    webhook_id      UUID NOT NULL REFERENCES webhooks(id) ON DELETE CASCADE,
    event_type      VARCHAR(100) NOT NULL,
    payload         JSONB NOT NULL,
    response_status INTEGER,
    response_body   TEXT,
    delivered_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_webhook_deliveries_webhook ON webhook_deliveries(webhook_id, delivered_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organization & Auth | 3 | organizations, users, organization_memberships |
| Projects & Environments | 3 | projects, environments, api_tokens |
| Feature Flags | 3 | flags, flag_variants, flag_environments |
| Segments & Targeting | 5 | segments, segment_conditions, targeting_rules, targeting_rule_segments, targeting_rule_variants |
| Overrides | 1 | flag_overrides |
| B2B Accounts | 2 | accounts, account_traits |
| Experimentation | 3 | experiments, experiment_metrics, experiment_results |
| Flag Lifecycle | 3 | flag_evaluation_stats, flag_code_references, flag_lifecycle |
| Audit | 1 | audit_events |
| Approval Workflows | 1 | change_requests |
| Webhooks | 2 | webhooks, webhook_deliveries |
| **Total** | **27** | |

---

## Key Design Decisions

1. **Weight in basis points (0-100000)** — Targeting rule variant weights use basis points instead of percentages. This allows 0.001% precision for fine-grained rollouts without floating-point rounding issues. The total weight across all variants in a rule should sum to 100000.

2. **Separate `flag_environments` junction table** — Instead of storing enabled/disabled state on the flag itself, each flag has a separate state per environment. This matches the Unleash/Flagsmith pattern and allows flags to be enabled in staging but disabled in production independently.

3. **Segments are project-scoped, not flag-scoped** — Segments are reusable across all flags within a project. This avoids duplicating targeting conditions across multiple flags and matches the Flagsmith/Unleash data model.

4. **Audit events use JSONB `before_state`/`after_state`** — The single denormalized field in this otherwise-normalized schema. Storing the full state snapshot as JSONB makes audit reconstruction trivial without requiring complex temporal JOINs across multiple tables.

5. **`context_kind` for multi-entity targeting** — The `flag_overrides` table supports `user`, `account`, and `device` context kinds, enabling B2B account-level targeting alongside individual user targeting, aligned with OpenFeature's multi-context evaluation model.

6. **Experiment results stored per-variant per-metric** — Rather than computing statistics on the fly, pre-computed results (mean, variance, p-value, confidence intervals) are stored. This allows the statistical engine to run as a background job without blocking the UI.

7. **`flag_type` enum follows Fowler's taxonomy** — Release, Experiment, Ops, Permission. This classification drives different lifecycle expectations (release flags should be short-lived; ops flags may be permanent) and is the basis for AI-driven staleness scoring.

8. **No soft deletes** — Archived flags use `is_archived = true` on the `flags` table. Deleted resources are truly deleted with `ON DELETE CASCADE`. The audit log preserves the history of all changes including deletions.

## Example Queries

### Evaluate a flag for a user context

```sql
-- Step 1: Check for individual override
SELECT fv.key, fv.value
FROM flag_overrides fo
JOIN flag_variants fv ON fv.id = fo.variant_id
JOIN flag_environments fe ON fe.id = fo.flag_environment_id
JOIN flags f ON f.id = fe.flag_id
WHERE f.key = 'checkout-v2'
  AND fe.environment_id = :env_id
  AND fo.context_key = :user_id
  AND fo.context_kind = 'user'
LIMIT 1;

-- Step 2: If no override, get targeting rules ordered by priority
SELECT tr.id, tr.priority
FROM targeting_rules tr
JOIN flag_environments fe ON fe.id = tr.flag_environment_id
JOIN flags f ON f.id = fe.flag_id
WHERE f.key = 'checkout-v2'
  AND fe.environment_id = :env_id
  AND fe.is_enabled = true
ORDER BY tr.priority ASC;
```

### Find stale flags across all projects

```sql
SELECT f.key, f.name, p.name AS project_name,
       fl.expected_lifetime_days, fl.staleness_score,
       fl.last_evaluated_at
FROM flag_lifecycle fl
JOIN flags f ON f.id = fl.flag_id
JOIN projects p ON p.id = f.project_id
WHERE fl.is_stale = true
  AND f.is_archived = false
ORDER BY fl.staleness_score DESC;
```
