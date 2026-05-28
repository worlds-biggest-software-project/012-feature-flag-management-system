# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Feature Flag Management System · Created: 2026-05-11

## Philosophy

This model uses a small number of relational tables for structural integrity (flags, environments, projects, experiments) while storing variable, extensible data in JSONB columns. Targeting rules, segment conditions, variant distributions, evaluation contexts, and experiment configurations are all stored as structured JSONB within their parent rows rather than as separate normalized tables.

This is the approach closest to how Flipt stores flag definitions (YAML/JSON documents with nested variants, rules, and segments) and how GrowthBook uses MongoDB (nested documents for feature definitions). The relational wrapper provides foreign key integrity, indexing, and transactional guarantees that pure document stores lack, while the JSONB columns provide the schema flexibility of a document database.

The key insight for a feature flag platform is that targeting rules and segment conditions vary widely across customers and evolve rapidly. A normalized schema requires a migration every time you add a new operator type or targeting dimension. With JSONB, the application schema evolves independently of the database schema — a new operator or attribute type is just a new value in the JSON.

**Best for:** Rapid MVP development with a small team (2-5 developers), platforms that need to iterate quickly on targeting rule syntax, and systems where the flag evaluation engine already deserialises JSON config (matching the SDK wire format directly).

**Trade-offs:**
- (+) Far fewer tables (~15 vs. 27-35 in normalized models)
- (+) Flag evaluation queries are simpler — one SELECT returns the full flag config
- (+) Adding new targeting operators/attributes requires no schema migration
- (+) JSONB columns map directly to the SDK wire format (SDK receives nearly the same JSON structure as stored)
- (+) PostgreSQL GIN indexes on JSONB columns support efficient containment queries
- (-) No database-level referential integrity for data inside JSONB columns
- (-) Complex cross-JSONB queries are possible but syntactically awkward
- (-) JSONB column sizes can grow large for flags with many targeting rules
- (-) Application must validate JSONB structure (JSON Schema validation in the API layer)
- (-) Harder to enforce constraints like "variant weights must sum to 100000" at the database level

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenFeature Evaluation API | `config` JSONB column on `flags` closely mirrors the OpenFeature flag evaluation response structure |
| OpenFeature OFREP | OFREP endpoints can serve `config` JSONB nearly as-is, reducing serialisation overhead |
| Flipt Flag Definition Format | JSONB flag configuration structure inspired by Flipt's YAML flag definitions (variants, rules, rollouts) |
| Martin Fowler's Toggle Taxonomy | `flag_type` relational column; targeting rules in JSONB reference taxonomy for lifecycle expectations |
| JSON Schema (IETF) | All JSONB columns validated against JSON Schema definitions in the application layer |
| ISO 3166 | Country codes used in geo-targeting conditions within JSONB targeting rules |
| CUPED / Sequential Testing | Experiment configuration stored as JSONB with fields for statistical method, CUPED enablement, sequential testing parameters |

---

## Core Tables

```sql
-- =============================================================
-- ORGANIZATION, USERS & MULTI-TENANCY
-- =============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- { "default_stale_days": 90, "require_approval": true,
    --   "allowed_auth_providers": ["local", "google", "github"] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
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
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, user_id)
);
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
    name            VARCHAR(100) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    color           VARCHAR(7),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_protected    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
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
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Feature Flags (with JSONB Configuration)

```sql
-- =============================================================
-- FEATURE FLAGS — core relational + JSONB config
-- =============================================================

CREATE TABLE flags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    key             VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    flag_type       VARCHAR(20) NOT NULL DEFAULT 'release',  -- release, experiment, ops, permission
    value_type      VARCHAR(20) NOT NULL DEFAULT 'boolean',  -- boolean, string, number, json
    tags            TEXT[] DEFAULT '{}',
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    
    -- JSONB: Variants definition (shared across all environments)
    variants        JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   { "key": "control", "name": "Control", "value": "false", "sort_order": 0 },
    --   { "key": "treatment_a", "name": "Treatment A", "value": "true", "sort_order": 1 },
    --   { "key": "treatment_b", "name": "Treatment B",
    --     "value": "{\"checkout_layout\": \"single_page\", \"show_upsell\": true}",
    --     "sort_order": 2 }
    -- ]
    
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (project_id, key)
);

CREATE INDEX idx_flags_project ON flags(project_id);
CREATE INDEX idx_flags_key ON flags(project_id, key);
CREATE INDEX idx_flags_type ON flags(flag_type);
CREATE INDEX idx_flags_archived ON flags(is_archived) WHERE is_archived = false;
-- GIN index for searching within variants JSONB
CREATE INDEX idx_flags_variants ON flags USING GIN (variants jsonb_path_ops);

-- Per-environment flag state with JSONB targeting configuration
CREATE TABLE flag_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_id         UUID NOT NULL REFERENCES flags(id) ON DELETE CASCADE,
    environment_id  UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
    is_enabled      BOOLEAN NOT NULL DEFAULT false,
    default_variant VARCHAR(100),                            -- fallback variant key
    
    -- JSONB: Complete targeting configuration for this environment
    targeting       JSONB NOT NULL DEFAULT '{"rules": [], "overrides": []}',
    -- Example:
    -- {
    --   "rules": [
    --     {
    --       "id": "rule-1",
    --       "name": "Beta users",
    --       "priority": 1,
    --       "conditions": {
    --         "match_type": "all",
    --         "items": [
    --           { "attribute": "plan", "operator": "in_list", "value": ["beta", "enterprise"] },
    --           { "attribute": "country", "operator": "equals", "value": "US" }
    --         ]
    --       },
    --       "distribution": [
    --         { "variant": "treatment_a", "weight": 80000 },
    --         { "variant": "control", "weight": 20000 }
    --       ],
    --       "bucket_by": "user_id"
    --     },
    --     {
    --       "id": "rule-2",
    --       "name": "Gradual rollout",
    --       "priority": 2,
    --       "conditions": null,
    --       "distribution": [
    --         { "variant": "treatment_a", "weight": 10000 },
    --         { "variant": "control", "weight": 90000 }
    --       ],
    --       "bucket_by": "user_id"
    --     }
    --   ],
    --   "overrides": [
    --     { "context_key": "user-123", "context_kind": "user", "variant": "treatment_a" },
    --     { "context_key": "acme-corp", "context_kind": "account", "variant": "control" }
    --   ]
    -- }
    
    -- JSONB: Scheduled changes (future flag state changes)
    scheduled_changes JSONB DEFAULT '[]',
    -- Example:
    -- [
    --   { "id": "sc-1", "apply_at": "2026-05-15T09:00:00Z",
    --     "changes": { "is_enabled": true, "targeting": { ... } },
    --     "created_by": "user-uuid" }
    -- ]
    
    version         INTEGER NOT NULL DEFAULT 1,              -- incremented on every change
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (flag_id, environment_id)
);

CREATE INDEX idx_flag_configs_flag ON flag_configs(flag_id);
CREATE INDEX idx_flag_configs_env ON flag_configs(environment_id);
-- GIN index for querying targeting rules
CREATE INDEX idx_flag_configs_targeting ON flag_configs USING GIN (targeting jsonb_path_ops);
```

## Segments (Reusable, JSONB Conditions)

```sql
-- =============================================================
-- SEGMENTS — reusable targeting segments with JSONB conditions
-- =============================================================

CREATE TABLE segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    
    -- JSONB: Segment definition with conditions
    definition      JSONB NOT NULL DEFAULT '{"match_type": "all", "conditions": []}',
    -- Example:
    -- {
    --   "match_type": "all",
    --   "conditions": [
    --     { "attribute": "email", "operator": "ends_with", "value": "@acme.com" },
    --     { "attribute": "signup_date", "operator": "greater_than", "value": "2026-01-01" },
    --     { "attribute": "feature_usage_count", "operator": "greater_than_or_equal", "value": "10" }
    --   ]
    -- }
    --
    -- Supported operators:
    --   equals, not_equals, contains, not_contains, starts_with, ends_with,
    --   greater_than, less_than, greater_than_or_equal, less_than_or_equal,
    --   in_list, not_in_list, matches_regex, is_true, is_false,
    --   exists, not_exists, semver_gt, semver_lt, semver_eq
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_segments_project ON segments(project_id);
CREATE INDEX idx_segments_definition ON segments USING GIN (definition jsonb_path_ops);
```

## Experimentation

```sql
-- =============================================================
-- EXPERIMENTATION
-- =============================================================

CREATE TABLE experiments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_id         UUID NOT NULL REFERENCES flags(id) ON DELETE RESTRICT,
    environment_id  UUID NOT NULL REFERENCES environments(id) ON DELETE RESTRICT,
    name            VARCHAR(255) NOT NULL,
    hypothesis      TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',    -- draft, running, paused, completed, abandoned
    
    -- JSONB: Full experiment configuration
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "statistical_method": "frequentist",
    --   "significance_level": 0.05,
    --   "minimum_sample_size": 5000,
    --   "use_cuped": true,
    --   "cuped_covariate_days": 14,
    --   "use_sequential": true,
    --   "sequential_check_interval_hours": 24,
    --   "control_variant": "control",
    --   "treatment_variants": ["treatment_a", "treatment_b"],
    --   "metrics": [
    --     { "name": "Conversion Rate", "event": "purchase_completed",
    --       "type": "conversion", "is_primary": true, "mde": 0.02 },
    --     { "name": "Revenue per User", "event": "purchase_completed",
    --       "type": "revenue", "value_field": "amount", "is_primary": false }
    --   ],
    --   "guardrail_metrics": [
    --     { "name": "Error Rate", "event": "js_error", "type": "count",
    --       "threshold": 0.01, "direction": "increase_is_bad" }
    --   ]
    -- }
    
    -- JSONB: Latest computed results
    results         JSONB,
    -- Example:
    -- {
    --   "computed_at": "2026-05-11T10:00:00Z",
    --   "total_sample_size": 15000,
    --   "variants": {
    --     "control": { "sample_size": 7500, "conversion_rate": 0.042,
    --                  "mean_revenue": 12.50, "variance": 85.3 },
    --     "treatment_a": { "sample_size": 7500, "conversion_rate": 0.048,
    --                      "mean_revenue": 14.20, "variance": 91.7,
    --                      "relative_lift": 0.143, "p_value": 0.023,
    --                      "confidence_interval": [0.002, 0.010],
    --                      "is_significant": true }
    --   },
    --   "recommendation": "Treatment A shows a statistically significant 14.3% lift in conversion rate. Recommend full rollout.",
    --   "guardrails_passed": true
    -- }
    
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_experiments_flag ON experiments(flag_id);
CREATE INDEX idx_experiments_status ON experiments(status);
```

## Flag Lifecycle & Analytics

```sql
-- =============================================================
-- FLAG LIFECYCLE, EVALUATION STATS & AI CLEANUP
-- =============================================================

CREATE TABLE flag_evaluation_stats (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_id         UUID NOT NULL REFERENCES flags(id) ON DELETE CASCADE,
    environment_id  UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
    date            DATE NOT NULL,
    total_evaluations BIGINT NOT NULL DEFAULT 0,
    unique_contexts BIGINT NOT NULL DEFAULT 0,
    variant_breakdown JSONB NOT NULL DEFAULT '{}',
    -- Example: {"control": 4500, "treatment_a": 4500}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (flag_id, environment_id, date)
);

CREATE INDEX idx_eval_stats_date ON flag_evaluation_stats(flag_id, environment_id, date);

CREATE TABLE flag_lifecycle (
    flag_id         UUID PRIMARY KEY REFERENCES flags(id) ON DELETE CASCADE,
    expected_lifetime_days INTEGER,
    stale_after     TIMESTAMPTZ,
    is_stale        BOOLEAN NOT NULL DEFAULT false,
    staleness_score DECIMAL(5,2),
    last_evaluated_at TIMESTAMPTZ,
    evaluation_trend VARCHAR(20),                            -- increasing, stable, declining, zero
    lifecycle_phase VARCHAR(20),                              -- new, active, stable, declining, stale
    
    -- JSONB: Code references found by static analysis
    code_references JSONB DEFAULT '[]',
    -- Example:
    -- [
    --   { "repository": "acme/web-app", "file": "src/checkout.tsx",
    --     "line": 42, "snippet": "if (getFlag('checkout-v2'))",
    --     "branch": "main", "commit": "abc123", "scanned_at": "2026-05-11T08:00:00Z" },
    --   { "repository": "acme/api", "file": "routes/checkout.py",
    --     "line": 88, "snippet": "flag_client.get_boolean('checkout-v2')",
    --     "branch": "main", "commit": "def456", "scanned_at": "2026-05-11T08:00:00Z" }
    -- ]
    
    -- JSONB: AI analysis results
    ai_analysis     JSONB,
    -- Example:
    -- {
    --   "analysed_at": "2026-05-11T09:00:00Z",
    --   "recommendation": "remove",
    --   "confidence": 0.92,
    --   "reasoning": "Flag has been 100% rolled out to treatment_a for 45 days. Evaluation rate is stable. 3 code references found across 2 repositories.",
    --   "cleanup_pr_url": "https://github.com/acme/web-app/pull/1234",
    --   "estimated_lines_removed": 28
    -- }
    
    cleanup_pr_url  VARCHAR(500),
    reviewed_at     TIMESTAMPTZ,
    reviewed_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_lifecycle_stale ON flag_lifecycle(is_stale) WHERE is_stale = true;
```

## Audit Log & Change Requests

```sql
-- =============================================================
-- AUDIT LOG
-- =============================================================

CREATE TABLE audit_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    actor_id        UUID REFERENCES users(id),
    actor_type      VARCHAR(50) NOT NULL DEFAULT 'user',
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(100) NOT NULL,
    resource_id     UUID NOT NULL,
    project_id      UUID REFERENCES projects(id),
    environment_id  UUID REFERENCES environments(id),
    
    -- JSONB: change details (diff or full state)
    change_data     JSONB NOT NULL DEFAULT '{}',
    -- Example for a targeting rule change:
    -- {
    --   "before": { "targeting": { "rules": [ ... ] } },
    --   "after": { "targeting": { "rules": [ ... ] } },
    --   "diff": { "rules_added": 1, "rules_modified": 0, "rules_removed": 0 }
    -- }
    
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_org_time ON audit_events(organization_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_events(resource_type, resource_id);

-- =============================================================
-- CHANGE REQUESTS (approval workflows for protected environments)
-- =============================================================

CREATE TABLE change_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_id         UUID NOT NULL REFERENCES flags(id) ON DELETE CASCADE,
    environment_id  UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, approved, rejected, applied, cancelled
    
    -- JSONB: the proposed changes
    proposed_changes JSONB NOT NULL,
    -- Example:
    -- {
    --   "is_enabled": true,
    --   "targeting": { "rules": [ ... ], "overrides": [ ... ] }
    -- }
    
    requested_by    UUID NOT NULL REFERENCES users(id),
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    applied_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_change_requests_pending ON change_requests(status) WHERE status = 'pending';

-- =============================================================
-- WEBHOOKS
-- =============================================================

CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    url             VARCHAR(2048) NOT NULL,
    secret          VARCHAR(255),
    events          TEXT[] NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_triggered_at TIMESTAMPTZ,
    failure_count   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organization & Auth | 3 | organizations, users, organization_memberships |
| Projects & Environments | 3 | projects, environments, api_tokens |
| Feature Flags | 2 | flags (with variants JSONB), flag_configs (with targeting JSONB) |
| Segments | 1 | segments (with definition JSONB) |
| Experimentation | 1 | experiments (with config + results JSONB) |
| Flag Lifecycle | 2 | flag_evaluation_stats, flag_lifecycle (with code_references + ai_analysis JSONB) |
| Audit & Approval | 2 | audit_events, change_requests |
| Webhooks | 1 | webhooks |
| **Total** | **15** | |

---

## Key Design Decisions

1. **Targeting rules are JSONB, not separate tables** — In the normalized model (Suggestion 1), targeting requires 4 tables (`targeting_rules`, `targeting_rule_segments`, `targeting_rule_variants`, `segment_conditions`). Here, the entire targeting configuration is a single JSONB column on `flag_configs`. This trades database-enforced integrity for dramatically simpler queries and faster flag evaluation reads.

2. **Variants on the `flags` table, not a separate table** — Variants are part of the flag definition and are shared across environments. Storing them as JSONB on the flag row keeps the flag + variants together in a single read. The `flag_configs` targeting rules reference variants by key string rather than by UUID foreign key.

3. **`version` column for optimistic concurrency** — The `flag_configs` table includes a version number that's incremented on every update. The API layer uses `WHERE version = :expected_version` to prevent concurrent targeting rule edits from overwriting each other.

4. **Experiment results stored inline** — Rather than a separate `experiment_results` table with rows per variant per metric, results are a single JSONB column on the `experiments` table. This works because results are always read and written as a complete set (you never update one variant's results independently).

5. **Code references in JSONB, not a separate table** — The `flag_lifecycle` table stores code references as a JSONB array. This is appropriate because code references are always read together (you never query "find all flags in file X" — you query "find all code references for flag Y"). If cross-flag code reference queries become important, this decision should be revisited.

6. **Scheduled changes on `flag_configs`** — Future flag state changes (e.g., "enable this flag at 9am Monday") are stored as a JSONB array on the `flag_configs` row. A background job reads `scheduled_changes` and applies changes when their `apply_at` timestamp arrives.

7. **JSON Schema validation in the application layer** — Every JSONB column has a corresponding JSON Schema definition enforced by the API. The database accepts any valid JSONB; the application rejects payloads that don't conform to the schema. This keeps the database simple while maintaining data quality.

8. **GIN indexes on JSONB columns** — PostgreSQL GIN indexes with `jsonb_path_ops` enable efficient containment queries like `targeting @> '{"rules": [{"conditions": {"items": [{"attribute": "country"}]}}]}'` to find all flags targeting by country.

## Example Queries

### Full flag evaluation (single query)

```sql
-- Returns everything needed to evaluate a flag in one query
SELECT f.key, f.value_type, f.variants,
       fc.is_enabled, fc.default_variant, fc.targeting
FROM flags f
JOIN flag_configs fc ON fc.flag_id = f.id
WHERE f.project_id = :project_id
  AND f.key = :flag_key
  AND fc.environment_id = :environment_id
  AND f.is_archived = false;

-- Application code then:
-- 1. Checks overrides in targeting.overrides for matching context
-- 2. Evaluates rules in targeting.rules by priority
-- 3. For matching rule, uses consistent hashing on bucket_by field
--    to select variant from distribution weights
-- 4. Falls back to default_variant if no rules match
```

### Bulk fetch all flags for an environment (SDK bootstrap)

```sql
SELECT f.key, f.value_type, f.variants,
       fc.is_enabled, fc.default_variant, fc.targeting
FROM flags f
JOIN flag_configs fc ON fc.flag_id = f.id
WHERE f.project_id = :project_id
  AND fc.environment_id = :environment_id
  AND f.is_archived = false;
```

### Find flags targeting a specific attribute

```sql
-- Find all flags that use "country" in any targeting rule condition
SELECT f.key, f.name, fc.targeting
FROM flags f
JOIN flag_configs fc ON fc.flag_id = f.id
WHERE fc.targeting @> '{"rules": [{"conditions": {"items": [{"attribute": "country"}]}}]}';
```

### JSONB containment query for segment-based rules

```sql
-- Find all flag_configs that reference a specific segment ID in their rules
SELECT fc.id, f.key
FROM flag_configs fc
JOIN flags f ON f.id = fc.flag_id
WHERE fc.targeting @> ('{"rules": [{"segment_id": "' || :segment_id || '"}]}')::jsonb;
```
