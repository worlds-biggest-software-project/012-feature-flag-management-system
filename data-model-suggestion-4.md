# Data Model Suggestion 4: Document-Oriented with Relational Anchors

> Project: Feature Flag Management System · Created: 2026-05-11

## Philosophy

This model treats each flag as a self-contained document — a single rich JSON/JSONB record that contains everything needed to evaluate the flag: variants, targeting rules (with inline segment conditions), overrides, scheduled changes, and metadata. There are no separate tables for variants, targeting rules, or segment conditions. The flag document IS the configuration.

Relational "anchor" tables provide multi-tenancy structure (organizations, projects, environments), authentication (users, API tokens), and analytics/experimentation (evaluation stats, experiments), but the core flag domain is purely document-oriented. This mirrors GrowthBook's MongoDB architecture where each feature is a single document containing all evaluation logic, and Flipt's file-based approach where each flag is a complete YAML document.

The critical difference from Suggestion 3 (Hybrid JSONB) is the degree of document nesting. Suggestion 3 stores variants on the `flags` table and targeting rules on a separate `flag_configs` table. This model collapses everything — including per-environment state — into a single `flag_documents` row per flag per environment. One SELECT, one document, complete evaluation context.

**Best for:** Teams optimising for evaluation latency (single-read flag evaluation), systems that sync flag configs to edge/CDN storage (the document IS the cache entry), and architectures where the SDK wire format should match the storage format exactly.

**Trade-offs:**
- (+) Single-read flag evaluation — no JOINs, no assembly, one document returned
- (+) Document format matches SDK wire protocol and CDN cache format exactly
- (+) Easy to sync to edge stores (Cloudflare KV, Redis, DynamoDB) — just write the document
- (+) Simple backup and restore — each document is self-contained
- (+) Natural fit for Git-backed storage (one file per flag, like Flipt)
- (-) Reusable segments must be denormalised into every flag document that references them
- (-) Updating a segment definition requires updating ALL flag documents that embed it
- (-) Cross-flag queries (e.g., "which flags target segment X?") require scanning all documents
- (-) Document size can grow very large for flags with many complex targeting rules
- (-) No database-level referential integrity within documents
- (-) Harder to run aggregate reports across flags without extracting data from JSONB

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenFeature Evaluation API | Document structure maps 1:1 to OpenFeature evaluation response; `variants`, `targeting`, `default_variant` fields directly serialisable |
| OpenFeature OFREP | OFREP `/evaluate/flags/{key}` endpoint returns the document's evaluation result with zero transformation |
| Flipt YAML Flag Format | Document structure inspired by Flipt's declarative flag definitions; can export as YAML for GitOps |
| ConfigCat config.json | Similar to ConfigCat's approach where SDKs download a single JSON config and evaluate locally |
| Martin Fowler's Toggle Taxonomy | `flag_type` field in document drives lifecycle expectations |
| JSON Schema (IETF) | Full JSON Schema validation for document structure enforced at API layer |

---

## Anchor Tables (Relational)

```sql
-- =============================================================
-- RELATIONAL ANCHORS — structure, auth, multi-tenancy
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

## Flag Documents (The Core)

```sql
-- =============================================================
-- FLAG DOCUMENTS — one self-contained document per flag per environment
-- =============================================================

CREATE TABLE flag_documents (
    -- Relational identity
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    environment_id  UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
    flag_key        VARCHAR(255) NOT NULL,
    
    -- The complete flag document
    document        JSONB NOT NULL,
    
    -- Metadata for querying without parsing the document
    flag_type       VARCHAR(20) NOT NULL DEFAULT 'release',  -- denormalised from document
    value_type      VARCHAR(20) NOT NULL DEFAULT 'boolean',  -- denormalised from document
    is_enabled      BOOLEAN NOT NULL DEFAULT false,          -- denormalised from document
    is_archived     BOOLEAN NOT NULL DEFAULT false,          -- denormalised from document
    tags            TEXT[] DEFAULT '{}',                      -- denormalised from document
    
    -- Versioning
    version         INTEGER NOT NULL DEFAULT 1,
    etag            VARCHAR(64) NOT NULL,                    -- SHA-256 of document content
    
    -- Timestamps
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    UNIQUE (project_id, environment_id, flag_key)
);

CREATE INDEX idx_flag_docs_project_env ON flag_documents(project_id, environment_id);
CREATE INDEX idx_flag_docs_key ON flag_documents(flag_key);
CREATE INDEX idx_flag_docs_type ON flag_documents(flag_type);
CREATE INDEX idx_flag_docs_active ON flag_documents(is_archived) WHERE is_archived = false;
CREATE INDEX idx_flag_docs_tags ON flag_documents USING GIN (tags);
CREATE INDEX idx_flag_docs_document ON flag_documents USING GIN (document jsonb_path_ops);
```

### Flag Document Schema

```json
{
  "key": "checkout-v2",
  "name": "Checkout V2 Redesign",
  "description": "New single-page checkout flow with upsell module",
  "flag_type": "experiment",
  "value_type": "boolean",
  "tags": ["checkout", "q2-2026", "revenue"],
  "is_enabled": true,
  "is_archived": false,

  "variants": [
    {
      "key": "control",
      "name": "Current Checkout",
      "value": "false",
      "sort_order": 0
    },
    {
      "key": "treatment_a",
      "name": "Single-Page Checkout",
      "value": "true",
      "sort_order": 1
    },
    {
      "key": "treatment_b",
      "name": "Single-Page + Upsell",
      "value": "{\"layout\": \"single_page\", \"show_upsell\": true}",
      "sort_order": 2
    }
  ],

  "default_variant": "control",

  "targeting": {
    "rules": [
      {
        "id": "rule-001",
        "name": "Internal QA team",
        "priority": 1,
        "conditions": {
          "match_type": "all",
          "items": [
            {
              "attribute": "email",
              "operator": "ends_with",
              "value": "@acme.com"
            },
            {
              "attribute": "role",
              "operator": "in_list",
              "value": ["qa", "engineering"]
            }
          ]
        },
        "distribution": [
          { "variant": "treatment_b", "weight": 100000 }
        ],
        "bucket_by": "user_id"
      },
      {
        "id": "rule-002",
        "name": "Enterprise accounts - gradual rollout",
        "priority": 2,
        "conditions": {
          "match_type": "all",
          "items": [
            {
              "context_kind": "account",
              "attribute": "plan",
              "operator": "equals",
              "value": "enterprise"
            },
            {
              "context_kind": "account",
              "attribute": "country",
              "operator": "in_list",
              "value": ["US", "CA", "GB"]
            }
          ]
        },
        "distribution": [
          { "variant": "treatment_a", "weight": 30000 },
          { "variant": "control", "weight": 70000 }
        ],
        "bucket_by": "account_id"
      },
      {
        "id": "rule-003",
        "name": "Global gradual rollout",
        "priority": 3,
        "conditions": null,
        "distribution": [
          { "variant": "treatment_a", "weight": 10000 },
          { "variant": "control", "weight": 90000 }
        ],
        "bucket_by": "user_id"
      }
    ],

    "overrides": [
      {
        "context_kind": "user",
        "context_key": "user-ceo-123",
        "variant": "treatment_b",
        "reason": "CEO demo"
      },
      {
        "context_kind": "account",
        "context_key": "acme-corp",
        "variant": "control",
        "reason": "Requested by account manager"
      }
    ]
  },

  "scheduled_changes": [
    {
      "id": "sc-001",
      "apply_at": "2026-05-20T09:00:00Z",
      "description": "Ramp enterprise rollout to 50%",
      "changes": {
        "targeting.rules[1].distribution": [
          { "variant": "treatment_a", "weight": 50000 },
          { "variant": "control", "weight": 50000 }
        ]
      },
      "created_by": "user-uuid-pm"
    }
  ],

  "lifecycle": {
    "expected_lifetime_days": 60,
    "stale_after": "2026-07-10T00:00:00Z",
    "created_by": "user-uuid-eng",
    "created_at": "2026-04-01T10:00:00Z"
  }
}
```

## Segment Library (Reference, Not Embedded)

```sql
-- =============================================================
-- SEGMENT LIBRARY — reusable segment templates
-- Segments are stored here as a reference, but are DENORMALISED
-- into flag documents when a flag references them.
-- =============================================================

CREATE TABLE segment_library (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    
    definition      JSONB NOT NULL,
    -- Example:
    -- {
    --   "match_type": "all",
    --   "conditions": [
    --     { "attribute": "plan", "operator": "in_list", "value": ["enterprise", "business"] },
    --     { "attribute": "employee_count", "operator": "greater_than", "value": "50" }
    --   ]
    -- }
    
    -- Track which flags embed this segment (for cascade updates)
    embedded_in_flags TEXT[] DEFAULT '{}',
    -- Example: ["checkout-v2", "pricing-page-v3"]
    
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_segment_lib_project ON segment_library(project_id);
```

## Experimentation

```sql
-- =============================================================
-- EXPERIMENTATION — relational for cross-experiment queries
-- =============================================================

CREATE TABLE experiments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    environment_id  UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
    flag_key        VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    hypothesis      TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    
    -- Full experiment configuration and results as JSONB
    config          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "statistical_method": "frequentist",
    --   "significance_level": 0.05,
    --   "use_cuped": true,
    --   "use_sequential": true,
    --   "control_variant": "control",
    --   "treatment_variants": ["treatment_a", "treatment_b"],
    --   "metrics": [ ... ],
    --   "guardrail_metrics": [ ... ]
    -- }
    
    latest_results  JSONB,
    -- {
    --   "computed_at": "...",
    --   "variants": { "control": { ... }, "treatment_a": { ... } },
    --   "recommendation": "...",
    --   "guardrails_passed": true
    -- }
    
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_experiments_flag ON experiments(project_id, flag_key);
CREATE INDEX idx_experiments_status ON experiments(status);
```

## Evaluation Stats & Lifecycle

```sql
-- =============================================================
-- EVALUATION STATISTICS
-- =============================================================

CREATE TABLE flag_evaluation_stats (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    environment_id  UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
    flag_key        VARCHAR(255) NOT NULL,
    date            DATE NOT NULL,
    total_evaluations BIGINT NOT NULL DEFAULT 0,
    unique_contexts BIGINT NOT NULL DEFAULT 0,
    variant_breakdown JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (project_id, environment_id, flag_key, date)
);

CREATE INDEX idx_eval_stats_lookup ON flag_evaluation_stats(project_id, environment_id, flag_key, date);

-- =============================================================
-- FLAG LIFECYCLE TRACKING
-- =============================================================

CREATE TABLE flag_lifecycle (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    flag_key        VARCHAR(255) NOT NULL,
    is_stale        BOOLEAN NOT NULL DEFAULT false,
    staleness_score DECIMAL(5,2),
    evaluation_trend VARCHAR(20),
    lifecycle_phase VARCHAR(20),
    last_evaluated_at TIMESTAMPTZ,
    last_modified_at TIMESTAMPTZ,
    
    code_references JSONB DEFAULT '[]',
    ai_analysis     JSONB,
    cleanup_pr_url  VARCHAR(500),
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (project_id, flag_key)
);

CREATE INDEX idx_lifecycle_stale ON flag_lifecycle(is_stale) WHERE is_stale = true;
```

## Audit Log & Webhooks

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
    resource_id     UUID,
    resource_key    VARCHAR(255),                            -- flag_key for flag resources
    project_id      UUID REFERENCES projects(id),
    environment_id  UUID REFERENCES environments(id),
    
    -- Store document snapshots for history
    before_document JSONB,
    after_document  JSONB,
    
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_org_time ON audit_events(organization_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_events(resource_type, resource_key);

-- =============================================================
-- CHANGE REQUESTS
-- =============================================================

CREATE TABLE change_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    environment_id  UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
    flag_key        VARCHAR(255) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    
    -- The proposed document (complete replacement)
    proposed_document JSONB NOT NULL,
    current_document  JSONB NOT NULL,                       -- snapshot at time of request
    
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Edge Sync Table

```sql
-- =============================================================
-- EDGE SYNC — tracks which documents have been pushed to edge stores
-- =============================================================

CREATE TABLE edge_sync_state (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    environment_id  UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
    edge_provider   VARCHAR(50) NOT NULL,                   -- cloudflare_kv, redis, dynamodb
    last_synced_at  TIMESTAMPTZ,
    last_synced_version INTEGER,
    sync_status     VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, synced, error
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (project_id, environment_id, edge_provider)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organization & Auth | 3 | organizations, users, organization_memberships |
| Projects & Environments | 3 | projects, environments, api_tokens |
| Flag Documents | 1 | flag_documents (complete flag definition per environment) |
| Segment Library | 1 | segment_library (reference segments, denormalised into documents) |
| Experimentation | 1 | experiments |
| Evaluation & Lifecycle | 2 | flag_evaluation_stats, flag_lifecycle |
| Audit & Approval | 2 | audit_events, change_requests |
| Webhooks | 1 | webhooks |
| Edge Sync | 1 | edge_sync_state |
| **Total** | **15** | |

---

## Key Design Decisions

1. **One document per flag per environment** — Each `flag_documents` row contains the complete evaluation configuration for one flag in one environment. If a flag exists in 3 environments (dev, staging, prod), there are 3 rows with potentially different targeting configurations. This differs from Suggestion 3 which separates flag metadata (shared) from per-environment config.

2. **Denormalised metadata columns** — `flag_type`, `value_type`, `is_enabled`, `is_archived`, and `tags` are stored as both JSONB fields inside the document AND as denormalised relational columns. The relational columns enable efficient queries and filtering without JSONB parsing; the document remains the source of truth.

3. **ETag for cache invalidation** — The `etag` column stores a SHA-256 hash of the document content. SDKs and edge stores can use ETags for conditional requests (`If-None-Match`), enabling efficient cache invalidation without downloading the full document.

4. **Segment denormalisation with cascade tracking** — When a flag references a reusable segment, the segment conditions are copied into the flag document. The `segment_library.embedded_in_flags` array tracks which flag keys embed each segment, enabling cascade updates when a segment definition changes. This is an explicit trade-off: read performance (no JOIN to evaluate) vs. write complexity (must update multiple documents when a segment changes).

5. **Flag key as the primary identifier** — Because documents are self-contained, the flag key (string) is used as the primary reference throughout the system rather than UUIDs. The experiments table, evaluation stats, and lifecycle tracking all reference flags by `(project_id, flag_key)` rather than by UUID foreign key to the document table. This aligns with how SDKs identify flags (by key, not by UUID).

6. **Change requests store complete documents** — Rather than storing a diff, `change_requests` stores both the current document snapshot and the proposed replacement document. This makes it trivial to display a document-level diff in the UI and to apply the change (replace the entire document row).

7. **Edge sync as a first-class concept** — The `edge_sync_state` table tracks synchronisation of flag documents to edge stores (Cloudflare KV, Redis, DynamoDB). Because the document format matches the cache format, sync is a direct write of the JSONB content to the edge key-value store.

8. **Audit stores full document snapshots** — Rather than recording field-level changes, the audit log stores `before_document` and `after_document` as complete JSONB snapshots. This makes audit reconstruction trivial but increases storage requirements. For flags with large documents, this trade-off is worth the simplicity.

## Example Queries

### Flag evaluation (single read)

```sql
-- One query returns everything needed to evaluate a flag
SELECT document
FROM flag_documents
WHERE project_id = :project_id
  AND environment_id = :environment_id
  AND flag_key = :flag_key
  AND is_archived = false;

-- The SDK receives the document and evaluates locally:
-- 1. Parse document.targeting.overrides for exact context match
-- 2. Evaluate document.targeting.rules in priority order
-- 3. For matching rule, hash(context[bucket_by] + flag_key) to select variant
-- 4. Fall back to document.default_variant
```

### Bulk SDK bootstrap (all flags for an environment)

```sql
-- Returns all active flag documents for SDK initialisation
SELECT flag_key, document, etag
FROM flag_documents
WHERE project_id = :project_id
  AND environment_id = :environment_id
  AND is_archived = false;

-- SDK caches these documents locally and evaluates offline
-- On reconnect, sends ETags; server returns only changed documents
```

### Conditional fetch with ETags

```sql
-- SDK sends previously cached ETags; only return changed documents
SELECT flag_key, document, etag
FROM flag_documents
WHERE project_id = :project_id
  AND environment_id = :environment_id
  AND is_archived = false
  AND flag_key NOT IN (
    -- List of flag keys where the SDK's cached ETag still matches
    SELECT flag_key FROM flag_documents
    WHERE project_id = :project_id
      AND environment_id = :environment_id
      AND (flag_key, etag) IN (VALUES
        ('checkout-v2', 'abc123...'),
        ('dark-mode', 'def456...')
      )
  );
```

### Cascade segment update

```sql
-- When segment "enterprise-accounts" is updated, find and update all flag documents
-- that embed it. This runs in the application layer:

-- 1. Get the list of affected flags
SELECT embedded_in_flags FROM segment_library
WHERE id = :segment_id AND project_id = :project_id;

-- 2. For each affected flag key, update the conditions in the matching rule
-- (Application code iterates and updates each flag document's JSONB)

-- 3. Update ETags and versions for all changed documents
UPDATE flag_documents
SET document = :updated_document,
    version = version + 1,
    etag = encode(sha256(:updated_document::text::bytea), 'hex'),
    updated_at = NOW()
WHERE project_id = :project_id
  AND flag_key = :flag_key;
```

### Export flag as YAML (Flipt-compatible GitOps)

```sql
-- Application reads the document and converts to YAML
SELECT document FROM flag_documents
WHERE project_id = :project_id
  AND environment_id = :environment_id
  AND flag_key = :flag_key;

-- The YAML output would look like:
-- key: checkout-v2
-- name: Checkout V2 Redesign
-- type: experiment
-- enabled: true
-- variants:
--   - key: control
--     value: "false"
--   - key: treatment_a
--     value: "true"
-- rules:
--   - segment: internal-qa
--     distributions:
--       - variant: treatment_b
--         rollout: 100
```
