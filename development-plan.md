# Feature Flag Management System — Phased Development Plan

> Project: 012-feature-flag-management-system · Created: 2026-05-11
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | OpenFeature SDK ecosystem is TypeScript-first; shared types between server and SDK packages; largest pool of flag-platform contributors |
| Runtime | Node.js 22 with native ESM | Stable LTS; native fetch, structured clone, test runner; ES module alignment with OpenFeature SDKs |
| API Framework | Fastify 5 | 2-3x throughput vs Express; native JSON Schema validation (aligns with JSONB validation strategy); plugin architecture for OpenFeature OFREP endpoints |
| Database | PostgreSQL 16 with JSONB | Hybrid relational + JSONB model (Data Model Suggestion 3); GIN indexes for targeting rule queries; partitioning for audit/evaluation tables; SQLite via better-sqlite3 for single-binary dev mode |
| ORM / Query Builder | Drizzle ORM | Type-safe schema, native JSONB support, zero-overhead SQL generation, first-class PostgreSQL support, migration tooling built in |
| Cache / Pub-Sub | Redis 7 (Valkey compatible) | Flag config caching, SSE fan-out for real-time SDK updates, evaluation stats buffering; optional — system works without it at lower throughput |
| Authentication | Lucia Auth + Arctic (OAuth) | Lightweight, TypeScript-native, supports local + OAuth (GitHub, Google) + SAML via passport-saml; no heavy framework dependency |
| API Specification | OpenAPI 3.1.0 | Machine-readable endpoints; auto-generated client SDKs; Fastify generates OpenAPI from route schemas natively |
| Frontend | React 19 + Vite 6 + Tailwind CSS 4 | Dashboard SPA; React for ecosystem breadth; Vite for fast dev; Tailwind for rapid UI iteration |
| Charting | Recharts 3 | Experiment results visualisation, evaluation trend graphs, rollout progress charts |
| Statistical Engine | Simple-statistics + custom CUPED module | simple-statistics for t-tests, z-tests, confidence intervals; custom TypeScript module for CUPED variance reduction and sequential testing (SPRT) |
| AI / LLM Integration | Anthropic Claude API (claude-sonnet-4-20250514) | Stale flag analysis, natural-language targeting rules, experiment result summaries; called server-side via @anthropic-ai/sdk |
| Hashing | MurmurHash3 (murmurhash3js) | Consistent hashing for percentage rollouts — industry standard used by LaunchDarkly, Unleash, GrowthBook |
| Testing | Vitest + Playwright + Supertest | Vitest for unit/integration; Playwright for E2E dashboard tests; Supertest for API tests |
| Containerisation | Docker (multi-stage) + Docker Compose | Single-container deployment; Compose for local dev with PostgreSQL + Redis |
| CI/CD | GitHub Actions | Lint, test, build, Docker image push on every PR and merge to main |
| Monorepo | pnpm workspaces + Turborepo | Shared types between server, dashboard, SDKs; Turborepo for parallel builds and caching |

### Project Structure

```
feature-flag-system/
├── package.json                    # pnpm workspace root
├── turbo.json                      # Turborepo pipeline config
├── docker-compose.yml              # PostgreSQL + Redis + app
├── Dockerfile                      # Multi-stage production image
│
├── packages/
│   ├── types/                      # Shared TypeScript types
│   │   ├── src/
│   │   │   ├── flag.ts             # Flag, Variant, FlagConfig types
│   │   │   ├── targeting.ts        # TargetingRule, Condition, Distribution
│   │   │   ├── segment.ts          # Segment, SegmentDefinition
│   │   │   ├── experiment.ts       # Experiment, ExperimentConfig, Results
│   │   │   ├── evaluation.ts       # EvaluationContext, EvaluationResult
│   │   │   ├── lifecycle.ts        # FlagLifecycle, StalenessScore
│   │   │   ├── audit.ts            # AuditEvent types
│   │   │   ├── api.ts              # API request/response types
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   ├── evaluation-engine/          # Flag evaluation logic (shared by server + SDKs)
│   │   ├── src/
│   │   │   ├── evaluator.ts        # Core evaluation: overrides → rules → default
│   │   │   ├── condition-matcher.ts # Attribute condition evaluation
│   │   │   ├── consistent-hash.ts  # MurmurHash3-based bucketing
│   │   │   ├── context.ts          # EvaluationContext builder
│   │   │   └── index.ts
│   │   ├── __tests__/
│   │   └── package.json
│   │
│   └── sdk-node/                   # Node.js SDK (OpenFeature provider)
│       ├── src/
│       │   ├── client.ts           # FFMClient
│       │   ├── provider.ts         # OpenFeature provider implementation
│       │   ├── polling.ts          # Config polling with ETag support
│       │   ├── streaming.ts        # SSE real-time updates
│       │   └── index.ts
│       ├── __tests__/
│       └── package.json
│
├── apps/
│   ├── server/                     # API server (Fastify)
│   │   ├── src/
│   │   │   ├── app.ts              # Fastify app factory
│   │   │   ├── server.ts           # Entry point
│   │   │   ├── config.ts           # Environment configuration
│   │   │   ├── db/
│   │   │   │   ├── schema.ts       # Drizzle schema definitions
│   │   │   │   ├── migrate.ts      # Migration runner
│   │   │   │   └── seed.ts         # Development seed data
│   │   │   ├── routes/
│   │   │   │   ├── flags.ts        # Flag CRUD endpoints
│   │   │   │   ├── configs.ts      # Per-environment flag config
│   │   │   │   ├── segments.ts     # Segment CRUD
│   │   │   │   ├── evaluate.ts     # SDK evaluation endpoint
│   │   │   │   ├── experiments.ts  # Experiment management
│   │   │   │   ├── lifecycle.ts    # Stale flag analysis
│   │   │   │   ├── audit.ts        # Audit log queries
│   │   │   │   ├── auth.ts         # Authentication routes
│   │   │   │   ├── webhooks.ts     # Webhook management
│   │   │   │   └── ofrep.ts        # OpenFeature OFREP endpoints
│   │   │   ├── services/
│   │   │   │   ├── flag-service.ts
│   │   │   │   ├── evaluation-service.ts
│   │   │   │   ├── segment-service.ts
│   │   │   │   ├── experiment-service.ts
│   │   │   │   ├── lifecycle-service.ts
│   │   │   │   ├── audit-service.ts
│   │   │   │   ├── webhook-service.ts
│   │   │   │   ├── stats-service.ts
│   │   │   │   └── ai-service.ts
│   │   │   ├── middleware/
│   │   │   │   ├── auth.ts         # Token + session auth
│   │   │   │   ├── rbac.ts         # Role-based access control
│   │   │   │   └── audit.ts        # Auto-audit middleware
│   │   │   ├── workers/
│   │   │   │   ├── stats-aggregator.ts
│   │   │   │   ├── stale-detector.ts
│   │   │   │   ├── scheduled-changes.ts
│   │   │   │   └── webhook-dispatcher.ts
│   │   │   └── lib/
│   │   │       ├── errors.ts
│   │   │       └── validation-schemas.ts
│   │   ├── drizzle/                # Generated migrations
│   │   ├── __tests__/
│   │   └── package.json
│   │
│   └── dashboard/                  # React SPA
│       ├── src/
│       │   ├── main.tsx
│       │   ├── App.tsx
│       │   ├── components/
│       │   ├── pages/
│       │   ├── hooks/
│       │   ├── api/                # Generated API client
│       │   └── stores/
│       ├── __tests__/
│       └── package.json
│
└── tools/
    ├── code-references/            # CLI for scanning flag keys in repositories
    │   ├── src/
    │   │   └── scanner.ts
    │   └── package.json
    └── cli/                        # `ffm` CLI tool
        ├── src/
        │   └── index.ts
        └── package.json
```

---

## Phase 1: Foundation — Database, Auth, Project Structure

### Purpose

Establish the monorepo, database schema, authentication, and the organisation/project/environment hierarchy. After this phase, a developer can sign up, create a project with environments, and generate API tokens. No flag functionality yet.

### Tasks

#### 1.1 — Monorepo Initialisation

**What**: Set up pnpm workspace with Turborepo, shared TypeScript configuration, and CI pipeline.

**Design**:

```jsonc
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "test": { "dependsOn": ["build"] },
    "lint": {},
    "typecheck": { "dependsOn": ["^build"] }
  }
}
```

```jsonc
// pnpm-workspace.yaml
packages:
  - "packages/*"
  - "apps/*"
  - "tools/*"
```

Shared `tsconfig.base.json`:
```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "dist",
    "rootDir": "src"
  }
}
```

GitHub Actions CI:
```yaml
# .github/workflows/ci.yml
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_DB: ffm_test, POSTGRES_PASSWORD: test }
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo lint typecheck test
```

**Testing**:
- `monorepo-setup`: `pnpm install` completes without errors
- `turbo-pipeline`: `pnpm turbo build` builds all packages in dependency order
- `ci-lint`: ESLint runs across all packages with zero violations on initial codebase

#### 1.2 — Database Schema and Migrations

**What**: Implement the Hybrid Relational + JSONB schema (Data Model Suggestion 3) using Drizzle ORM with PostgreSQL.

**Design**:

```typescript
// packages/types/src/flag.ts
export type FlagType = 'release' | 'experiment' | 'ops' | 'permission';
export type ValueType = 'boolean' | 'string' | 'number' | 'json';

export interface Flag {
  id: string;           // UUID
  projectId: string;
  key: string;          // unique within project
  name: string;
  description: string | null;
  flagType: FlagType;
  valueType: ValueType;
  tags: string[];
  variants: Variant[];  // JSONB
  isArchived: boolean;
  createdBy: string | null;
  createdAt: Date;
  updatedAt: Date;
}

export interface Variant {
  key: string;          // e.g. 'control', 'treatment_a'
  name: string;
  value: string;        // serialised value
  sortOrder: number;
}
```

```typescript
// packages/types/src/targeting.ts
export type ConditionOperator =
  | 'equals' | 'not_equals'
  | 'contains' | 'not_contains'
  | 'starts_with' | 'ends_with'
  | 'greater_than' | 'less_than'
  | 'greater_than_or_equal' | 'less_than_or_equal'
  | 'in_list' | 'not_in_list'
  | 'matches_regex'
  | 'is_true' | 'is_false'
  | 'exists' | 'not_exists'
  | 'semver_gt' | 'semver_lt' | 'semver_eq';

export interface Condition {
  attribute: string;
  operator: ConditionOperator;
  value: string | string[];
  contextKind?: 'user' | 'account' | 'device';  // defaults to 'user'
}

export interface ConditionGroup {
  matchType: 'all' | 'any';
  items: Condition[];
}

export interface VariantDistribution {
  variant: string;     // variant key
  weight: number;      // basis points, 0-100000
}

export interface TargetingRule {
  id: string;
  name: string;
  priority: number;
  conditions: ConditionGroup | null;  // null = catch-all
  distribution: VariantDistribution[];
  bucketBy: string;    // context field to hash on
  segmentId?: string;  // optional reference to reusable segment
}

export interface Override {
  contextKind: 'user' | 'account' | 'device';
  contextKey: string;
  variant: string;
}

export interface TargetingConfig {
  rules: TargetingRule[];
  overrides: Override[];
}

export interface ScheduledChange {
  id: string;
  applyAt: string;     // ISO 8601
  changes: Partial<{ isEnabled: boolean; targeting: TargetingConfig }>;
  createdBy: string;
}
```

```typescript
// apps/server/src/db/schema.ts (Drizzle)
import { pgTable, uuid, varchar, text, boolean, integer, timestamp, jsonb, uniqueIndex, index } from 'drizzle-orm/pg-core';

export const organizations = pgTable('organizations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  planTier: varchar('plan_tier', { length: 50 }).notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  name: varchar('name', { length: 255 }).notNull(),
  passwordHash: varchar('password_hash', { length: 255 }),
  authProvider: varchar('auth_provider', { length: 50 }).notNull().default('local'),
  authProviderExternalId: varchar('auth_provider_id', { length: 255 }),
  isActive: boolean('is_active').notNull().default(true),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const organizationMemberships = pgTable('organization_memberships', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  role: varchar('role', { length: 50 }).notNull().default('member'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_org_memberships_unique').on(table.organizationId, table.userId),
]);

export const projects = pgTable('projects', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull(),
  description: text('description'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_projects_org_slug').on(table.organizationId, table.slug),
]);

export const environments = pgTable('environments', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 100 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull(),
  color: varchar('color', { length: 7 }),
  sortOrder: integer('sort_order').notNull().default(0),
  isProtected: boolean('is_protected').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_environments_project_slug').on(table.projectId, table.slug),
]);

export const apiTokens = pgTable('api_tokens', {
  id: uuid('id').primaryKey().defaultRandom(),
  environmentId: uuid('environment_id').notNull().references(() => environments.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  tokenHash: varchar('token_hash', { length: 255 }).notNull().unique(),
  tokenPrefix: varchar('token_prefix', { length: 10 }).notNull(),
  tokenType: varchar('token_type', { length: 20 }).notNull().default('server'),
  expiresAt: timestamp('expires_at', { withTimezone: true }),
  createdBy: uuid('created_by').references(() => users.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

Error handling for migrations:
```typescript
// apps/server/src/db/migrate.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { migrate } from 'drizzle-orm/node-postgres/migrator';

export async function runMigrations(connectionString: string): Promise<void> {
  const db = drizzle(connectionString);
  try {
    await migrate(db, { migrationsFolder: './drizzle' });
  } catch (error) {
    console.error('Migration failed:', error);
    process.exit(1);
  }
}
```

**Testing**:
- `schema-migration-up`: Migrations apply cleanly to an empty PostgreSQL 16 database
- `schema-migration-idempotent`: Running migrations twice produces no errors
- `schema-org-create`: Insert organization, verify slug uniqueness constraint
- `schema-project-cascade`: Delete organization cascades to projects and environments
- `schema-env-defaults`: New project auto-creates dev/staging/production environments

#### 1.3 — Authentication and User Management

**What**: Implement local email/password auth, GitHub OAuth, session management, and API token generation.

**Design**:

```typescript
// apps/server/src/routes/auth.ts
interface RegisterRequest {
  email: string;
  password: string;       // min 8 chars, bcrypt hashed
  name: string;
  organizationName: string;
}

interface LoginRequest {
  email: string;
  password: string;
}

interface AuthResponse {
  user: { id: string; email: string; name: string };
  session: { token: string; expiresAt: string };
}

// Routes:
// POST /api/v1/auth/register       → AuthResponse
// POST /api/v1/auth/login          → AuthResponse
// POST /api/v1/auth/logout         → 204
// GET  /api/v1/auth/github         → redirect to GitHub OAuth
// GET  /api/v1/auth/github/callback → AuthResponse
// POST /api/v1/auth/tokens         → { token: string, prefix: string }
// DELETE /api/v1/auth/tokens/:id   → 204
```

API token format: `ffm_<type>_<32-random-hex>` where type is `srv` (server), `cli` (client), or `adm` (admin). Token stored as SHA-256 hash; first 8 characters stored as `token_prefix` for identification. Token scoped to a single environment.

Session management: HTTP-only secure cookies with 30-day expiry; sliding window renewal on each request.

```typescript
// apps/server/src/middleware/auth.ts
interface AuthContext {
  type: 'session' | 'api_token';
  userId: string | null;        // null for API tokens
  organizationId: string;
  environmentId: string | null; // set for API token auth
  role: 'owner' | 'admin' | 'member' | 'viewer';
}

// Middleware extracts auth from:
// 1. Authorization: Bearer <api_token>
// 2. Cookie: ffm_session=<session_token>
```

**Testing**:
- `auth-register`: Register user, verify organization and membership created
- `auth-register-duplicate-email`: Returns 409 Conflict
- `auth-login-valid`: Returns session token, sets cookie
- `auth-login-invalid-password`: Returns 401
- `auth-token-create`: Generate API token, verify hash stored, prefix matches
- `auth-token-auth`: API request with valid token returns 200
- `auth-token-expired`: Expired token returns 401
- `auth-session-renewal`: Session cookie updated on each authenticated request
- `auth-github-oauth-flow`: Mock GitHub OAuth returns user and creates membership

#### 1.4 — Organisation, Project, and Environment CRUD

**What**: REST API endpoints for managing organisations, projects, and environments with role-based access control.

**Design**:

```typescript
// Route definitions
// Organizations
// GET    /api/v1/organizations                → Organization[]
// GET    /api/v1/organizations/:orgSlug       → Organization
// PATCH  /api/v1/organizations/:orgSlug       → Organization
// POST   /api/v1/organizations/:orgSlug/members → Membership (invite)
// DELETE /api/v1/organizations/:orgSlug/members/:userId → 204

// Projects
// GET    /api/v1/projects                     → Project[] (scoped to org)
// POST   /api/v1/projects                     → Project
// GET    /api/v1/projects/:projectSlug        → Project
// PATCH  /api/v1/projects/:projectSlug        → Project
// DELETE /api/v1/projects/:projectSlug        → 204

// Environments
// GET    /api/v1/projects/:projectSlug/environments → Environment[]
// POST   /api/v1/projects/:projectSlug/environments → Environment
// PATCH  /api/v1/projects/:projectSlug/environments/:envSlug → Environment
// DELETE /api/v1/projects/:projectSlug/environments/:envSlug → 204
```

RBAC matrix:

| Action | Owner | Admin | Member | Viewer |
|--------|-------|-------|--------|--------|
| Create project | Y | Y | N | N |
| Delete project | Y | Y | N | N |
| Create environment | Y | Y | Y | N |
| Delete environment | Y | Y | N | N |
| Manage members | Y | Y | N | N |
| View all | Y | Y | Y | Y |

Default environments created on project creation: `development` (sort 0), `staging` (sort 1), `production` (sort 2, protected).

**Testing**:
- `project-create`: POST creates project with 3 default environments
- `project-slug-unique`: Duplicate slug within org returns 409
- `project-delete-cascade`: Deleting project removes environments and API tokens
- `env-protected`: Protected environment requires approval for flag changes (enforced in Phase 6)
- `rbac-viewer-blocked`: Viewer role cannot create projects (returns 403)
- `rbac-member-create-env`: Member can create environments but not delete them
- `org-member-invite`: Adding a member sends them an email-based invitation

---

## Phase 2: Core Flag Management — CRUD, Variants, Per-Environment Config

### Purpose

Implement feature flag creation, variant management, and per-environment configuration. After this phase, users can create flags with variants and configure them independently per environment. No targeting or evaluation yet.

### Tasks

#### 2.1 — Flag CRUD with Variants

**What**: REST API for creating, reading, updating, archiving, and listing flags with embedded JSONB variants.

**Design**:

```typescript
// Drizzle schema addition
export const flags = pgTable('flags', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  key: varchar('key', { length: 255 }).notNull(),
  name: varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  flagType: varchar('flag_type', { length: 20 }).notNull().default('release'),
  valueType: varchar('value_type', { length: 20 }).notNull().default('boolean'),
  tags: text('tags').array().default([]),
  variants: jsonb('variants').notNull().default([]),
  isArchived: boolean('is_archived').notNull().default(false),
  createdBy: uuid('created_by').references(() => users.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_flags_project_key').on(table.projectId, table.key),
  index('idx_flags_type').on(table.flagType),
]);

// Routes:
// POST   /api/v1/projects/:slug/flags          → Flag
// GET    /api/v1/projects/:slug/flags          → { flags: Flag[], total: number }
// GET    /api/v1/projects/:slug/flags/:key     → Flag
// PATCH  /api/v1/projects/:slug/flags/:key     → Flag
// POST   /api/v1/projects/:slug/flags/:key/archive → Flag
// DELETE /api/v1/projects/:slug/flags/:key     → 204

// Flag creation auto-generates default variants based on valueType:
// boolean → [{ key: "on", value: "true" }, { key: "off", value: "false" }]
// string  → [{ key: "default", value: "" }]
// number  → [{ key: "default", value: "0" }]
// json    → [{ key: "default", value: "{}" }]
```

```typescript
// CreateFlagRequest JSON Schema (Fastify validation)
const createFlagSchema = {
  type: 'object',
  required: ['key', 'name'],
  properties: {
    key: { type: 'string', pattern: '^[a-z0-9][a-z0-9._-]{0,253}[a-z0-9]$' },
    name: { type: 'string', minLength: 1, maxLength: 255 },
    description: { type: 'string' },
    flagType: { type: 'string', enum: ['release', 'experiment', 'ops', 'permission'] },
    valueType: { type: 'string', enum: ['boolean', 'string', 'number', 'json'] },
    tags: { type: 'array', items: { type: 'string' } },
    variants: {
      type: 'array',
      items: {
        type: 'object',
        required: ['key', 'value'],
        properties: {
          key: { type: 'string', pattern: '^[a-z0-9_-]+$' },
          name: { type: 'string' },
          value: { type: 'string' },
          sortOrder: { type: 'integer' }
        }
      }
    }
  }
} as const;
```

Flag key validation: lowercase alphanumeric with dots, hyphens, and underscores; 2-255 characters; must start and end with alphanumeric. This matches the OpenFeature flag key specification.

**Testing**:
- `flag-create-boolean`: Creates flag with auto-generated on/off variants
- `flag-create-custom-variants`: Creates flag with user-specified variants
- `flag-key-validation`: Rejects keys with spaces, uppercase, or special characters
- `flag-key-unique`: Duplicate key within project returns 409
- `flag-list-pagination`: GET with `?limit=20&offset=0` returns paginated results
- `flag-list-filter-type`: GET with `?flagType=experiment` returns only experiment flags
- `flag-list-filter-tag`: GET with `?tag=checkout` returns tagged flags
- `flag-archive`: Archived flags excluded from list by default, included with `?includeArchived=true`
- `flag-variants-jsonb`: Variant JSONB stored and retrieved with correct types

#### 2.2 — Per-Environment Flag Configuration

**What**: Manage flag state (enabled/disabled) and targeting configuration independently per environment.

**Design**:

```typescript
// Drizzle schema
export const flagConfigs = pgTable('flag_configs', {
  id: uuid('id').primaryKey().defaultRandom(),
  flagId: uuid('flag_id').notNull().references(() => flags.id, { onDelete: 'cascade' }),
  environmentId: uuid('environment_id').notNull().references(() => environments.id, { onDelete: 'cascade' }),
  isEnabled: boolean('is_enabled').notNull().default(false),
  defaultVariant: varchar('default_variant', { length: 100 }),
  targeting: jsonb('targeting').notNull().default({ rules: [], overrides: [] }),
  scheduledChanges: jsonb('scheduled_changes').default([]),
  version: integer('version').notNull().default(1),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_flag_configs_unique').on(table.flagId, table.environmentId),
]);

// Routes:
// GET  /api/v1/projects/:slug/flags/:key/configs             → FlagConfig[]
// GET  /api/v1/projects/:slug/flags/:key/configs/:envSlug    → FlagConfig
// PATCH /api/v1/projects/:slug/flags/:key/configs/:envSlug   → FlagConfig

// Key behaviors:
// - Creating a flag auto-creates a FlagConfig row per environment (disabled, default variant = first variant)
// - PATCH supports partial updates: { isEnabled: true } only changes enabled state
// - Version incremented on every PATCH; optimistic concurrency via If-Match header
```

```typescript
// PATCH request body
interface UpdateFlagConfigRequest {
  isEnabled?: boolean;
  defaultVariant?: string;  // must be a valid variant key on the parent flag
  targeting?: TargetingConfig;
  scheduledChanges?: ScheduledChange[];
}

// Optimistic concurrency:
// Client sends If-Match: "3" (current version)
// Server checks version = 3, updates to version = 4
// If version mismatch, returns 412 Precondition Failed
```

**Testing**:
- `config-auto-created`: Creating a flag creates FlagConfig rows for all environments
- `config-enable-disable`: PATCH isEnabled toggles flag state per environment
- `config-independent`: Enabling in staging does not affect production config
- `config-default-variant-validation`: Setting defaultVariant to nonexistent key returns 400
- `config-optimistic-concurrency`: Concurrent PATCH with stale version returns 412
- `config-version-increment`: Each PATCH increments version by 1

#### 2.3 — Audit Logging

**What**: Automatically record all flag and configuration mutations to the audit log.

**Design**:

```typescript
// Drizzle schema
export const auditEvents = pgTable('audit_events', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  actorId: uuid('actor_id').references(() => users.id),
  actorType: varchar('actor_type', { length: 50 }).notNull().default('user'),
  action: varchar('action', { length: 100 }).notNull(),
  resourceType: varchar('resource_type', { length: 100 }).notNull(),
  resourceId: uuid('resource_id').notNull(),
  projectId: uuid('project_id').references(() => projects.id),
  environmentId: uuid('environment_id').references(() => environments.id),
  changeData: jsonb('change_data').notNull().default({}),
  ipAddress: varchar('ip_address', { length: 45 }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_audit_org_time').on(table.organizationId, table.createdAt),
  index('idx_audit_resource').on(table.resourceType, table.resourceId),
]);

// Audit actions:
// flag.created, flag.updated, flag.archived, flag.deleted
// flag_config.updated, flag_config.enabled, flag_config.disabled
// segment.created, segment.updated, segment.deleted
// experiment.created, experiment.started, experiment.completed
// project.created, project.deleted
// member.invited, member.removed, member.role_changed
```

Implemented as Fastify `onResponse` hook that captures before/after state for mutating requests. The audit middleware wraps service methods to capture state diffs.

```typescript
// Routes:
// GET /api/v1/projects/:slug/audit  → { events: AuditEvent[], total: number }
//   Query params: ?resourceType=flag&action=flag.updated&from=2026-05-01&limit=50
```

**Testing**:
- `audit-flag-create`: Creating a flag generates `flag.created` audit event with after_state
- `audit-config-update`: Updating flag config generates event with before/after change_data
- `audit-query-filter`: Filtering by resourceType, action, and date range returns correct events
- `audit-actor-recorded`: Audit event captures userId for session auth, token ID for API auth
- `audit-pagination`: Large audit logs paginate correctly

---

## Phase 3: Evaluation Engine — Targeting, Consistent Hashing, SDK Endpoint

### Purpose

Build the core flag evaluation engine: condition matching, consistent hashing for percentage rollouts, and the SDK-facing evaluation endpoint. After this phase, an application can evaluate flags against user context at <10ms latency with local SDK evaluation.

### Tasks

#### 3.1 — Evaluation Engine Package

**What**: Implement the deterministic flag evaluation algorithm as a standalone package shared between server and SDKs.

**Design**:

```typescript
// packages/evaluation-engine/src/evaluator.ts
import { murmurhash3_32_gc } from 'murmurhash3js';

export interface EvaluationContext {
  kind: 'user' | 'account' | 'device' | 'multi';
  key: string;                      // primary identifier
  attributes: Record<string, unknown>;
  // For multi-context:
  user?: { key: string; attributes: Record<string, unknown> };
  account?: { key: string; attributes: Record<string, unknown> };
}

export interface EvaluationResult {
  flagKey: string;
  variant: string;            // selected variant key
  value: string;              // variant value
  reason: 'OVERRIDE' | 'TARGETING_MATCH' | 'DEFAULT' | 'DISABLED' | 'ERROR';
  ruleId?: string;            // which targeting rule matched
  metadata?: Record<string, unknown>;
}

export function evaluateFlag(
  flagKey: string,
  config: FlagConfig,        // includes variants, targeting, isEnabled, defaultVariant
  context: EvaluationContext
): EvaluationResult {
  // 1. If flag disabled, return default variant with reason DISABLED
  // 2. Check overrides: exact match on (contextKind, contextKey)
  // 3. Evaluate rules in priority order:
  //    a. Check conditions against context attributes
  //    b. If conditions match (or null = catch-all), hash to select variant
  // 4. If no rules match, return default variant with reason DEFAULT
}
```

```typescript
// packages/evaluation-engine/src/consistent-hash.ts
const TOTAL_BUCKETS = 100_000; // basis points

export function getBucket(hashKey: string, salt: string): number {
  // hashKey = context[bucketBy], salt = flagKey
  const hash = murmurhash3_32_gc(`${salt}:${hashKey}`, 0);
  return hash % TOTAL_BUCKETS;
}

export function selectVariant(
  bucket: number,
  distribution: VariantDistribution[]
): string {
  let cumulative = 0;
  for (const d of distribution) {
    cumulative += d.weight;
    if (bucket < cumulative) {
      return d.variant;
    }
  }
  return distribution[distribution.length - 1].variant;
}
```

```typescript
// packages/evaluation-engine/src/condition-matcher.ts
export function matchesCondition(
  context: EvaluationContext,
  condition: Condition
): boolean {
  const ctxKind = condition.contextKind ?? 'user';
  const attributes = ctxKind === context.kind
    ? context.attributes
    : (context as any)[ctxKind]?.attributes ?? {};
  const actual = attributes[condition.attribute];

  if (actual === undefined) {
    return condition.operator === 'not_exists';
  }

  switch (condition.operator) {
    case 'equals': return String(actual) === String(condition.value);
    case 'not_equals': return String(actual) !== String(condition.value);
    case 'contains': return String(actual).includes(String(condition.value));
    case 'in_list': return (condition.value as string[]).includes(String(actual));
    case 'greater_than': return Number(actual) > Number(condition.value);
    case 'semver_gt': return semverCompare(String(actual), String(condition.value)) > 0;
    // ... all 19 operators
  }
}

export function matchesConditionGroup(
  context: EvaluationContext,
  group: ConditionGroup
): boolean {
  const fn = group.matchType === 'all' ? 'every' : 'some';
  return group.items[fn](c => matchesCondition(context, c));
}
```

**Testing**:
- `eval-disabled-flag`: Disabled flag returns default variant with reason DISABLED
- `eval-override-user`: User override returns specified variant with reason OVERRIDE
- `eval-override-account`: Account-level override works for B2B context
- `eval-rule-match`: Rule with matching conditions returns correct variant
- `eval-rule-priority`: Lower priority number evaluated first
- `eval-rule-catchall`: Rule with null conditions matches all contexts
- `eval-hash-consistency`: Same user+flag always returns same variant across 10000 evaluations
- `eval-hash-distribution`: 100K evaluations with 50/50 split produces 49.5-50.5% distribution (within 1%)
- `eval-condition-equals`: String equality matching
- `eval-condition-in-list`: List membership matching
- `eval-condition-semver-gt`: Semantic version comparison
- `eval-condition-regex`: Regex matching
- `eval-condition-not-exists`: Missing attribute matches not_exists
- `eval-multi-context`: Multi-context evaluation resolves correct context kind per condition
- `eval-no-rules-match`: Falls through to default variant with reason DEFAULT

#### 3.2 — Server-Side Evaluation Endpoint

**What**: REST endpoint for server-side flag evaluation and bulk evaluation for SDK bootstrap.

**Design**:

```typescript
// Routes:
// POST /api/v1/evaluate/flags/:key    → EvaluationResult
// POST /api/v1/evaluate/flags         → { flags: Record<string, EvaluationResult> }

// Single evaluation request:
interface EvaluateRequest {
  context: EvaluationContext;
}

// Bulk evaluation request (SDK bootstrap):
interface BulkEvaluateRequest {
  context: EvaluationContext;
}
// Returns ALL non-archived flags for the environment (determined by API token)

// Server flow:
// 1. Authenticate via API token → resolve environmentId
// 2. Load flag config(s) from DB (or Redis cache if available)
// 3. Run evaluation engine
// 4. Record evaluation event asynchronously (fire-and-forget to stats buffer)
// 5. Return result

// Cache strategy:
// - Flag configs cached in Redis with key: `flag_config:{projectId}:{envId}:{flagKey}`
// - TTL: 60 seconds (configurable)
// - Cache invalidated on flag_config PATCH via Redis pub/sub
// - Without Redis: in-memory LRU cache with 30-second TTL
```

Performance target: <10ms p99 for single flag evaluation with cached config; <50ms for bulk evaluation of 100 flags.

**Testing**:
- `evaluate-single`: POST with context returns correct variant and reason
- `evaluate-bulk`: POST returns all flags for environment
- `evaluate-auth-required`: Missing API token returns 401
- `evaluate-token-scoped`: Server token evaluates only flags in its environment
- `evaluate-cache-hit`: Second evaluation served from cache (verify via metrics)
- `evaluate-cache-invalidation`: Config update invalidates cache, next eval returns new config
- `evaluate-perf-single`: Single evaluation completes in <10ms (benchmark test)
- `evaluate-perf-bulk-100`: 100-flag bulk evaluation completes in <50ms (benchmark test)

#### 3.3 — Segment Management

**What**: Reusable targeting segments that can be referenced by multiple flag targeting rules.

**Design**:

```typescript
// Drizzle schema
export const segments = pgTable('segments', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  definition: jsonb('definition').notNull().default({ matchType: 'all', conditions: [] }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_segments_project').on(table.projectId),
]);

// Routes:
// POST   /api/v1/projects/:slug/segments      → Segment
// GET    /api/v1/projects/:slug/segments      → Segment[]
// GET    /api/v1/projects/:slug/segments/:id  → Segment
// PATCH  /api/v1/projects/:slug/segments/:id  → Segment
// DELETE /api/v1/projects/:slug/segments/:id  → 204

// Segment reference in targeting rules:
// A targeting rule's conditions can be inline OR reference a segment:
// {
//   "id": "rule-1",
//   "segmentId": "uuid-of-segment",  // if present, conditions resolved from segment
//   "conditions": null,               // null when using segmentId
//   "distribution": [...]
// }
```

When evaluating a rule with `segmentId`, the evaluation engine resolves the segment definition from the segment table (cached) and uses its conditions for matching.

**Testing**:
- `segment-create`: Create segment with conditions
- `segment-reference`: Flag rule referencing segment evaluates using segment conditions
- `segment-update-cascade`: Updating segment conditions affects all flags referencing it
- `segment-delete-blocked`: Cannot delete segment referenced by active targeting rules (returns 409)
- `segment-list-usage`: GET segment returns count of flags referencing it

---

## Phase 4: Real-Time Updates, Webhooks, and Node.js SDK

### Purpose

Enable real-time flag updates via SSE streaming, webhook notifications, and ship the first production SDK (Node.js) with OpenFeature provider. After this phase, applications can integrate using the SDK with live updates.

### Tasks

#### 4.1 — Server-Sent Events for Real-Time Flag Updates

**What**: SSE endpoint that streams flag configuration changes to connected SDKs in real time.

**Design**:

```typescript
// Route:
// GET /api/v1/stream → text/event-stream

// Event format:
// event: flag_updated
// data: {"flagKey":"checkout-v2","environment":"production","version":5}

// event: flags_config
// data: {"flags":{...}} (full flag config payload on initial connect)

// Implementation:
// 1. On connect: authenticate via API token, send current flag configs
// 2. Subscribe to Redis pub/sub channel: `flag_updates:{projectId}:{envId}`
// 3. On flag_config PATCH: publish event to channel
// 4. SSE connection forwards events to SDK
// 5. Heartbeat: send comment (: keepalive) every 30 seconds
// 6. Reconnection: SDK sends Last-Event-ID header; server replays missed events

// Without Redis:
// In-memory EventEmitter; works for single-server deployments
```

```typescript
// Redis pub/sub message format
interface FlagUpdateMessage {
  flagKey: string;
  environmentId: string;
  version: number;
  timestamp: string;
  changeType: 'config_updated' | 'flag_created' | 'flag_archived' | 'flag_deleted';
}
```

**Testing**:
- `sse-connect`: SSE connection returns initial flag configs
- `sse-flag-update`: Updating flag config sends event to connected SSE clients
- `sse-auth-required`: Unauthenticated SSE connection returns 401
- `sse-heartbeat`: Keepalive comment sent every 30 seconds
- `sse-reconnect`: Reconnection with Last-Event-ID replays missed events
- `sse-multi-client`: Multiple SDK connections receive the same update
- `sse-env-scoped`: Client only receives events for their token's environment

#### 4.2 — Webhook System

**What**: Outbound webhooks that notify external systems of flag changes with HMAC signature verification.

**Design**:

```typescript
// Drizzle schema
export const webhooks = pgTable('webhooks', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  url: varchar('url', { length: 2048 }).notNull(),
  secret: varchar('secret', { length: 255 }),
  events: text('events').array().notNull(),
  isActive: boolean('is_active').notNull().default(true),
  lastTriggeredAt: timestamp('last_triggered_at', { withTimezone: true }),
  failureCount: integer('failure_count').notNull().default(0),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// Routes:
// POST   /api/v1/projects/:slug/webhooks      → Webhook
// GET    /api/v1/projects/:slug/webhooks      → Webhook[]
// PATCH  /api/v1/projects/:slug/webhooks/:id  → Webhook
// DELETE /api/v1/projects/:slug/webhooks/:id  → 204
// POST   /api/v1/projects/:slug/webhooks/:id/test → 200 (sends test payload)

// Webhook delivery:
// POST to webhook URL with JSON body:
interface WebhookPayload {
  event: string;              // e.g. 'flag.updated'
  timestamp: string;          // ISO 8601
  project: { id: string; slug: string };
  data: Record<string, unknown>;
}

// Headers:
// X-FFM-Signature: sha256=<HMAC-SHA256(secret, body)>
// X-FFM-Event: flag.updated
// X-FFM-Delivery: <uuid>
// Content-Type: application/json

// Retry policy: 3 attempts with exponential backoff (1s, 5s, 30s)
// After 10 consecutive failures: webhook auto-disabled, notification sent
```

Webhook dispatcher runs as a background worker (`workers/webhook-dispatcher.ts`) consuming from an in-memory queue (Redis list in production).

**Testing**:
- `webhook-create`: Create webhook subscribing to `flag.updated` events
- `webhook-delivery`: Flag update triggers webhook POST with correct payload
- `webhook-signature`: HMAC signature matches expected value
- `webhook-retry`: Failed delivery retries with exponential backoff
- `webhook-auto-disable`: 10 consecutive failures disables webhook
- `webhook-test-endpoint`: Test endpoint sends sample payload to webhook URL
- `webhook-filter`: Webhook only receives subscribed event types

#### 4.3 — Node.js SDK with OpenFeature Provider

**What**: Production-ready Node.js SDK that supports polling, SSE streaming, and implements the OpenFeature Provider interface.

**Design**:

```typescript
// packages/sdk-node/src/client.ts
export interface FFMClientOptions {
  apiKey: string;           // API token (ffm_srv_...)
  baseUrl: string;          // Server URL
  updateMode: 'polling' | 'streaming';  // default: 'streaming'
  pollingIntervalMs?: number;           // default: 30000
  enableEvaluationCache?: boolean;      // default: true
}

export class FFMClient {
  constructor(options: FFMClientOptions);

  async initialize(): Promise<void>;  // Fetch initial configs
  async close(): Promise<void>;       // Clean up connections

  getBooleanValue(flagKey: string, defaultValue: boolean, context?: EvaluationContext): boolean;
  getStringValue(flagKey: string, defaultValue: string, context?: EvaluationContext): string;
  getNumberValue(flagKey: string, defaultValue: number, context?: EvaluationContext): number;
  getJsonValue<T>(flagKey: string, defaultValue: T, context?: EvaluationContext): T;

  getEvaluationDetails(flagKey: string, context?: EvaluationContext): EvaluationResult;

  on(event: 'ready' | 'error' | 'configUpdated', handler: Function): void;
}
```

```typescript
// packages/sdk-node/src/provider.ts
import { Provider, ResolutionDetails, EvaluationContext as OFContext } from '@openfeature/server-sdk';

export class FFMProvider implements Provider {
  metadata = { name: 'ffm-provider' };

  constructor(private client: FFMClient) {}

  async initialize(): Promise<void> {
    await this.client.initialize();
  }

  resolveBooleanEvaluation(
    flagKey: string,
    defaultValue: boolean,
    context: OFContext
  ): ResolutionDetails<boolean> {
    const result = this.client.getEvaluationDetails(flagKey,
      this.mapContext(context));
    return {
      value: result.value === 'true',
      reason: result.reason,
      variant: result.variant,
    };
  }

  // resolveStringEvaluation, resolveNumberEvaluation, resolveObjectEvaluation...

  private mapContext(ofContext: OFContext): EvaluationContext {
    return {
      kind: 'user',
      key: ofContext.targetingKey ?? '',
      attributes: ofContext as Record<string, unknown>,
    };
  }
}
```

SDK local evaluation: configs downloaded and evaluated entirely in-process. No network call per evaluation. Streaming mode keeps configs up-to-date via SSE; polling mode refreshes every N seconds with ETag-based conditional fetch.

**Testing**:
- `sdk-initialize`: Client connects and downloads flag configs
- `sdk-boolean-eval`: getBooleanValue returns correct variant
- `sdk-streaming-update`: Config change received via SSE, next evaluation returns new value
- `sdk-polling-update`: Config change detected on next poll cycle
- `sdk-offline-eval`: Evaluations continue working after network disconnection (cached configs)
- `sdk-openfeature-provider`: OpenFeature Provider resolves boolean, string, number, object flags
- `sdk-openfeature-context`: OpenFeature context correctly mapped to FFM context
- `sdk-ready-event`: 'ready' event fires after initialization
- `sdk-error-handling`: Network errors do not crash; fallback to cached configs

---

## Phase 5: Experimentation Engine

### Purpose

Build the statistical experimentation framework: experiment lifecycle, metric tracking, Bayesian and frequentist analysis, CUPED variance reduction, and sequential testing for valid early stopping. After this phase, users can run statistically rigorous A/B tests on any flag.

### Tasks

#### 5.1 — Experiment Lifecycle Management

**What**: CRUD for experiments linked to flags, with state machine for experiment lifecycle (draft → running → paused → completed/abandoned).

**Design**:

```typescript
// Drizzle schema
export const experiments = pgTable('experiments', {
  id: uuid('id').primaryKey().defaultRandom(),
  flagId: uuid('flag_id').notNull().references(() => flags.id, { onDelete: 'restrict' }),
  environmentId: uuid('environment_id').notNull().references(() => environments.id, { onDelete: 'restrict' }),
  name: varchar('name', { length: 255 }).notNull(),
  hypothesis: text('hypothesis'),
  status: varchar('status', { length: 20 }).notNull().default('draft'),
  config: jsonb('config').notNull().default({}),
  results: jsonb('results'),
  startedAt: timestamp('started_at', { withTimezone: true }),
  endedAt: timestamp('ended_at', { withTimezone: true }),
  createdBy: uuid('created_by').references(() => users.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_experiments_flag').on(table.flagId),
  index('idx_experiments_status').on(table.status),
]);

// State transitions:
// draft → running    (start: validates config, locks flag targeting)
// running → paused   (pause: stops counting new exposures)
// paused → running   (resume)
// running → completed (end with winner selected)
// running → abandoned (end without winner)
// draft → abandoned  (cancel before starting)

// Routes:
// POST   /api/v1/projects/:slug/experiments           → Experiment
// GET    /api/v1/projects/:slug/experiments           → Experiment[]
// GET    /api/v1/projects/:slug/experiments/:id       → Experiment
// PATCH  /api/v1/projects/:slug/experiments/:id       → Experiment
// POST   /api/v1/projects/:slug/experiments/:id/start → Experiment
// POST   /api/v1/projects/:slug/experiments/:id/pause → Experiment
// POST   /api/v1/projects/:slug/experiments/:id/resume → Experiment
// POST   /api/v1/projects/:slug/experiments/:id/complete → Experiment
// POST   /api/v1/projects/:slug/experiments/:id/abandon → Experiment
```

```typescript
// ExperimentConfig type
interface ExperimentConfig {
  statisticalMethod: 'frequentist' | 'bayesian';
  significanceLevel: number;       // alpha, default 0.05
  minimumSampleSize: number | null;
  useCuped: boolean;
  cupedCovariateDays: number;      // default 14
  useSequential: boolean;
  sequentialCheckIntervalHours: number; // default 24
  controlVariant: string;          // variant key
  treatmentVariants: string[];     // variant keys
  metrics: ExperimentMetric[];
  guardrailMetrics: GuardrailMetric[];
}

interface ExperimentMetric {
  name: string;
  eventName: string;
  metricType: 'conversion' | 'revenue' | 'count' | 'duration';
  isPrimary: boolean;
  minimumDetectableEffect: number | null;
  valueField?: string;             // for revenue/count metrics
}

interface GuardrailMetric {
  name: string;
  eventName: string;
  metricType: 'count' | 'rate';
  threshold: number;
  direction: 'increase_is_bad' | 'decrease_is_bad';
}
```

**Testing**:
- `experiment-create`: Create experiment linked to flag with metrics
- `experiment-start-validates`: Starting validates minimum config (control + treatment + primary metric)
- `experiment-state-machine`: Invalid transitions (e.g., completed → running) return 400
- `experiment-flag-lock`: Starting experiment prevents flag targeting changes (returns 409)
- `experiment-complete`: Completing stores winning variant and unlocks flag

#### 5.2 — Metric Event Ingestion

**What**: Endpoint for client applications to report experiment metric events (conversions, revenue, etc.) and a background worker to aggregate them per variant.

**Design**:

```typescript
// Route:
// POST /api/v1/events → 204
interface EventBatch {
  events: MetricEvent[];
}

interface MetricEvent {
  eventName: string;            // matches ExperimentMetric.eventName
  contextKey: string;           // user/account identifier
  contextKind: 'user' | 'account';
  value?: number;               // for revenue/count metrics
  timestamp: string;            // ISO 8601
  properties?: Record<string, unknown>;
}

// Ingestion flow:
// 1. Validate event batch (max 1000 events per batch)
// 2. Buffer events to Redis list (or in-memory queue without Redis)
// 3. Background worker (stats-aggregator) processes events every 60 seconds:
//    a. Match events to running experiments by eventName
//    b. Determine which variant the context was exposed to (from evaluation log)
//    c. Update aggregate counters per experiment × metric × variant
//    d. Store aggregates in experiment.results JSONB
```

```typescript
// Evaluation log (exposure tracking)
export const experimentExposures = pgTable('experiment_exposures', {
  id: uuid('id').primaryKey().defaultRandom(),
  experimentId: uuid('experiment_id').notNull().references(() => experiments.id, { onDelete: 'cascade' }),
  contextKey: varchar('context_key', { length: 255 }).notNull(),
  contextKind: varchar('context_kind', { length: 50 }).notNull().default('user'),
  variant: varchar('variant', { length: 100 }).notNull(),
  exposedAt: timestamp('exposed_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_exposures_unique').on(table.experimentId, table.contextKey, table.contextKind),
  index('idx_exposures_experiment').on(table.experimentId),
]);
```

**Testing**:
- `event-ingest`: POST batch of events returns 204
- `event-validation`: Events with missing required fields return 400
- `event-batch-limit`: Batch exceeding 1000 events returns 400
- `event-aggregation`: Background worker aggregates events into experiment results
- `exposure-tracking`: Flag evaluation during experiment records exposure
- `exposure-dedup`: Same user exposed multiple times counted once

#### 5.3 — Statistical Analysis Engine

**What**: Compute experiment results using frequentist (z-test, t-test) and Bayesian methods, with CUPED variance reduction and sequential testing.

**Design**:

```typescript
// apps/server/src/services/experiment-service.ts

interface VariantStats {
  sampleSize: number;
  conversions?: number;           // for conversion metrics
  sumValue?: number;              // for revenue/count
  sumSquared?: number;            // for variance calculation
  mean: number;
  variance: number;
}

interface AnalysisResult {
  computedAt: string;
  totalSampleSize: number;
  variants: Record<string, {
    sampleSize: number;
    mean: number;
    variance: number;
    relativeLift: number;          // vs control
    absoluteLift: number;
    confidenceInterval: [number, number]; // 95% CI
    pValue: number | null;         // null for Bayesian
    posteriorProbability: number | null; // null for frequentist
    isSignificant: boolean;
    credibleInterval?: [number, number]; // Bayesian only
  }>;
  recommendation: string;
  guardrailsPassed: boolean;
  sequentialCheckpoint?: {
    checkNumber: number;
    adjustedAlpha: number;          // O'Brien-Fleming spending function
    canStop: boolean;
  };
}

// CUPED implementation:
function applyCuped(
  preExperimentData: Map<string, number>,  // contextKey → pre-experiment metric value
  experimentData: Map<string, number>,     // contextKey → experiment metric value
  covariateDays: number
): { adjustedMean: number; adjustedVariance: number } {
  // 1. Compute covariance between pre-experiment and experiment values
  // 2. Compute theta = cov(Y, X) / var(X)
  // 3. Adjusted Y = Y - theta * (X - mean(X))
  // 4. Return adjusted mean and variance (30-50% variance reduction)
}

// Sequential testing (O'Brien-Fleming):
function getSequentialBoundary(
  checkNumber: number,
  totalChecks: number,
  alpha: number
): number {
  // O'Brien-Fleming spending function:
  // alpha_k = 2 * (1 - Phi(z_alpha/2 / sqrt(k/K)))
  // Returns the adjusted alpha for this interim check
}

// Bayesian analysis:
function computeBayesianProbability(
  controlStats: VariantStats,
  treatmentStats: VariantStats
): { probabilityBeatControl: number; credibleInterval: [number, number] } {
  // Normal-Normal conjugate model with uninformative prior
  // 1. Posterior for control: Normal(mean_c, var_c / n_c)
  // 2. Posterior for treatment: Normal(mean_t, var_t / n_t)
  // 3. P(treatment > control) from difference distribution
  // 4. 95% credible interval from difference posterior
}
```

Results computation runs as a scheduled background job (`workers/stats-aggregator.ts`) at the configured `sequentialCheckIntervalHours` (default: every 24 hours for running experiments).

**Testing**:
- `stats-frequentist-significant`: Known-significant data produces p < 0.05 and isSignificant=true
- `stats-frequentist-not-significant`: Known-insignificant data produces p > 0.05
- `stats-bayesian-high-probability`: High-signal data produces posteriorProbability > 0.95
- `stats-cuped-variance-reduction`: CUPED-adjusted variance is 30-50% lower than unadjusted
- `stats-sequential-early-stop`: Sequential test signals canStop=true when effect is large
- `stats-sequential-no-early-stop`: Marginal effect correctly does not allow early stopping
- `stats-guardrail-violation`: Guardrail metric exceeding threshold marks guardrailsPassed=false
- `stats-conversion-metric`: Conversion rate computed as conversions/sampleSize
- `stats-revenue-metric`: Mean revenue computed with correct variance
- `stats-multiple-treatments`: 3+ variants analysed correctly against control

---

## Phase 6: Dashboard UI

### Purpose

Build the React dashboard for flag management, targeting rule editing, experiment monitoring, and audit trail viewing. After this phase, non-technical team members can manage flags without using the API directly.

### Tasks

#### 6.1 — Dashboard Shell, Navigation, and Auth Pages

**What**: React app with authentication flow, organisation/project navigation, and layout shell.

**Design**:

```typescript
// apps/dashboard/src/App.tsx
// Route structure:
// /login
// /register
// /auth/github/callback
// /:orgSlug/                         → Project list
// /:orgSlug/:projectSlug/flags       → Flag list
// /:orgSlug/:projectSlug/flags/:key  → Flag detail
// /:orgSlug/:projectSlug/segments    → Segment list
// /:orgSlug/:projectSlug/experiments → Experiment list
// /:orgSlug/:projectSlug/audit       → Audit log
// /:orgSlug/settings                 → Org settings, members

// Layout:
// Top bar: org selector, project selector, user menu
// Side nav: Flags, Segments, Experiments, Audit, Settings
// Main content: route-specific view
```

API client auto-generated from OpenAPI spec using `openapi-typescript-codegen`:
```typescript
// apps/dashboard/src/api/client.ts
// Generated from /api/v1/openapi.json
```

**Testing**:
- `dashboard-login`: Login form submits credentials, stores session, redirects to projects
- `dashboard-register`: Registration creates org and redirects to project creation
- `dashboard-nav`: Navigation between projects and sections works without page reload
- `dashboard-auth-redirect`: Unauthenticated access redirects to /login

#### 6.2 — Flag Management UI

**What**: Flag list view with filtering/search, flag detail view with variant editor and per-environment configuration.

**Design**:

Flag list page:
- Table with columns: Key, Name, Type, Tags, Status (per-environment toggle chips), Last Modified
- Search bar: full-text search on key and name
- Filters: flag type, tags, archived status
- Sort: by name, created date, last modified
- Bulk actions: archive, add tag

Flag detail page:
- Header: flag key, name, type badge, tags, edit button
- Variant editor: add/remove/edit variants with live preview of values
- Environment tabs: one tab per environment showing:
  - Enable/disable toggle (big, prominent)
  - Default variant selector
  - Targeting rules builder (Phase 6.3)
  - Kill switch button (disables across ALL environments)

Targeting rules builder (visual):
- Ordered list of rules, drag to reorder priority
- Each rule: name, conditions (attribute + operator + value dropdowns), distribution sliders
- Segment selector: dropdown to pick from project segments instead of inline conditions
- Override list: context kind + key + variant selector
- "Add rule" button with templates (percentage rollout, segment-based, individual override)

**Testing**:
- `ui-flag-list`: Renders flags with correct data from API
- `ui-flag-search`: Search filters list in real time
- `ui-flag-create-modal`: Create flag modal validates inputs and creates flag
- `ui-flag-detail-variants`: Variant editor adds/removes variants
- `ui-flag-env-toggle`: Environment toggle enables/disables flag via API
- `ui-flag-kill-switch`: Kill switch disables flag in all environments
- `ui-targeting-add-rule`: Adding a targeting rule updates the targeting config
- `ui-targeting-drag-reorder`: Dragging rules updates priority order
- `ui-targeting-distribution-slider`: Distribution slider adjusts variant weights (must sum to 100%)

#### 6.3 — Experiment Dashboard

**What**: Experiment creation wizard, running experiment monitoring with live charts, and results summary.

**Design**:

Experiment creation wizard (3 steps):
1. Select flag + environment, name experiment, write hypothesis
2. Configure: statistical method (frequentist/Bayesian), select control/treatment variants, enable CUPED/sequential
3. Define metrics: primary metric (event name, type), guardrail metrics, sample size

Running experiment view:
- Status badge (draft/running/paused/completed)
- Sample size progress bar (current vs. target)
- Per-variant metrics table: conversion rate, mean, lift, p-value/posterior, CI
- Time series chart (Recharts): conversion rate per variant over time
- Sequential testing status: current check number, adjusted alpha, stop recommendation
- Guardrail metrics status: green/red indicators

Results summary page:
- Winner banner with confidence level
- Statistical summary table
- "Roll out winner" button → sets flag to 100% winning variant
- "Archive experiment" button

**Testing**:
- `ui-experiment-wizard`: 3-step wizard creates experiment with correct config
- `ui-experiment-chart`: Recharts renders time series for variant metrics
- `ui-experiment-results`: Results table shows lift, p-value, significance correctly
- `ui-experiment-rollout-winner`: "Roll out winner" updates flag targeting to 100% winner

#### 6.4 — Audit Log and Settings Pages

**What**: Searchable audit log viewer and organisation settings (members, API tokens).

**Design**:

Audit log page:
- Filterable table: date range, actor, action type, resource type
- Each row expandable to show before/after change diff (JSON diff viewer)
- Export as CSV

Settings page:
- Members tab: list members, invite by email, change roles, remove
- API tokens tab: list tokens (showing prefix only), create new, delete
- Environments tab: create/edit/delete environments, toggle protected status
- Webhooks tab: list, create, test, delete webhooks

**Testing**:
- `ui-audit-log`: Renders audit events with correct data
- `ui-audit-filter`: Date range and action type filters work correctly
- `ui-audit-diff`: Expandable row shows JSON diff between before/after states
- `ui-settings-members`: Invite member flow sends invitation
- `ui-settings-tokens`: Token creation shows token value once, then only prefix

---

## Phase 7: Flag Lifecycle and Stale Detection

### Purpose

Implement automated stale flag detection with configurable thresholds, evaluation trend tracking, and dashboard health indicators. After this phase, teams can identify and manage flag technical debt before it accumulates.

### Tasks

#### 7.1 — Evaluation Statistics Collection

**What**: Background worker that aggregates flag evaluation counts per day, per environment, per variant.

**Design**:

```typescript
// Drizzle schema
export const flagEvaluationStats = pgTable('flag_evaluation_stats', {
  id: uuid('id').primaryKey().defaultRandom(),
  flagId: uuid('flag_id').notNull().references(() => flags.id, { onDelete: 'cascade' }),
  environmentId: uuid('environment_id').notNull().references(() => environments.id, { onDelete: 'cascade' }),
  date: varchar('date', { length: 10 }).notNull(),  // YYYY-MM-DD
  totalEvaluations: integer('total_evaluations').notNull().default(0),
  uniqueContexts: integer('unique_contexts').notNull().default(0),
  variantBreakdown: jsonb('variant_breakdown').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_eval_stats_unique').on(table.flagId, table.environmentId, table.date),
]);

// Collection flow:
// 1. Each evaluation increments an in-memory counter (or Redis INCR)
// 2. Every 60 seconds, stats-aggregator worker flushes counters to DB
// 3. Uses UPSERT to atomically increment daily totals
// 4. HyperLogLog (Redis) or approximate Set for unique context counting
```

**Testing**:
- `stats-collection`: 1000 evaluations produce correct daily count
- `stats-variant-breakdown`: Variant-level counts match expected distribution
- `stats-unique-contexts`: Unique context count is approximate but within 2% error
- `stats-date-rollover`: Counts correctly partition at midnight UTC

#### 7.2 — Flag Lifecycle Tracking and Staleness Detection

**What**: Background worker that analyses evaluation trends and computes staleness scores for all flags.

**Design**:

```typescript
// Drizzle schema
export const flagLifecycle = pgTable('flag_lifecycle', {
  flagId: uuid('flag_id').primaryKey().references(() => flags.id, { onDelete: 'cascade' }),
  expectedLifetimeDays: integer('expected_lifetime_days'),
  staleAfter: timestamp('stale_after', { withTimezone: true }),
  isStale: boolean('is_stale').notNull().default(false),
  stalenessScore: varchar('staleness_score', { length: 10 }),  // 0-100
  lastEvaluatedAt: timestamp('last_evaluated_at', { withTimezone: true }),
  evaluationTrend: varchar('evaluation_trend', { length: 20 }),
  lifecyclePhase: varchar('lifecycle_phase', { length: 20 }),
  codeReferences: jsonb('code_references').default([]),
  aiAnalysis: jsonb('ai_analysis'),
  cleanupPrUrl: varchar('cleanup_pr_url', { length: 500 }),
  reviewedAt: timestamp('reviewed_at', { withTimezone: true }),
  reviewedBy: uuid('reviewed_by').references(() => users.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// Staleness detection algorithm (runs daily):
function computeStalenessScore(flag: Flag, lifecycle: FlagLifecycle, stats: EvalStats[]): number {
  let score = 0;

  // Factor 1: Age vs expected lifetime (40% weight)
  const ageDays = daysSince(flag.createdAt);
  const expectedDays = lifecycle.expectedLifetimeDays ?? getDefaultLifetime(flag.flagType);
  // release: 30 days, experiment: 90 days, ops: 365 days, permission: 365 days
  if (ageDays > expectedDays) {
    score += 40 * Math.min(ageDays / expectedDays, 3) / 3;
  }

  // Factor 2: Evaluation trend (30% weight)
  const trend = computeEvaluationTrend(stats);
  // 'increasing': 0, 'stable': 5, 'declining': 20, 'zero': 30
  score += trendScore[trend];

  // Factor 3: Fully rolled out (20% weight)
  // If flag is 100% to one variant in all environments for >14 days
  if (isFullyRolledOut(flag)) score += 20;

  // Factor 4: No recent changes (10% weight)
  const daysSinceChange = daysSince(flag.updatedAt);
  if (daysSinceChange > 30) score += 10;

  return Math.min(score, 100);
}

// Lifecycle phases:
// new       → created < 7 days ago
// active    → evaluation trend increasing or stable, recent changes
// stable    → evaluation trend stable, no changes > 14 days
// declining → evaluation trend declining
// stale     → staleness score > 70
```

```typescript
// Evaluation trend computation:
function computeEvaluationTrend(stats: EvalStats[]): 'increasing' | 'stable' | 'declining' | 'zero' {
  const last7 = stats.filter(s => daysSince(s.date) <= 7);
  const prev7 = stats.filter(s => daysSince(s.date) > 7 && daysSince(s.date) <= 14);

  const recentAvg = average(last7.map(s => s.totalEvaluations));
  const prevAvg = average(prev7.map(s => s.totalEvaluations));

  if (recentAvg === 0 && prevAvg === 0) return 'zero';
  if (prevAvg === 0) return 'increasing';

  const changeRate = (recentAvg - prevAvg) / prevAvg;
  if (changeRate > 0.1) return 'increasing';
  if (changeRate < -0.1) return 'declining';
  return 'stable';
}
```

**Testing**:
- `lifecycle-new-flag`: Flag created < 7 days ago has phase "new", score 0
- `lifecycle-active-flag`: Flag with increasing evaluations has phase "active"
- `lifecycle-stale-release`: Release flag older than 30 days with zero evaluations scores > 70
- `lifecycle-ops-not-stale`: Ops flag at 60 days with stable evaluations scores < 30
- `lifecycle-fully-rolled-out`: Flag at 100% to one variant for 14+ days scores +20
- `lifecycle-trend-declining`: 50% evaluation drop week-over-week classified as "declining"
- `lifecycle-trend-zero`: Zero evaluations for 14 days classified as "zero"

#### 7.3 — Lifecycle Dashboard and Notifications

**What**: Dashboard view showing flag health across the project, with Slack/email notifications for stale flags.

**Design**:

Dashboard additions:
- Flag health summary bar: counts of flags in each lifecycle phase (new, active, stable, declining, stale)
- Flag list adds "Health" column with color-coded badge (green/yellow/orange/red)
- Stale flags page: filtered view of all stale flags sorted by staleness score
- Flag detail adds lifecycle panel: evaluation trend chart (7-day, 30-day), staleness score, expected lifetime, code references

Notifications:
- Slack integration via incoming webhook
- Daily digest: "You have N stale flags in project X"
- Individual alerts when a flag transitions to "stale" phase

**Testing**:
- `ui-lifecycle-summary`: Health bar shows correct counts per phase
- `ui-lifecycle-badge`: Flag list shows correct color badge per staleness score
- `ui-lifecycle-chart`: Evaluation trend chart renders 30-day data
- `notification-slack-digest`: Daily digest message sent to configured Slack webhook
- `notification-stale-transition`: Alert sent when flag becomes stale

---

## Phase 8: AI-Powered Features

### Purpose

Integrate LLM capabilities for stale flag analysis, natural-language targeting rule authoring, and experiment result interpretation. After this phase, the platform's AI differentiators are live.

### Tasks

#### 8.1 — AI Stale Flag Analysis

**What**: LLM-powered analysis of stale flags that provides removal recommendations with confidence scores and reasoning.

**Design**:

```typescript
// apps/server/src/services/ai-service.ts
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

interface StaleAnalysisInput {
  flag: Flag;
  lifecycle: FlagLifecycle;
  evaluationStats: EvalStats[];        // last 90 days
  codeReferences: CodeReference[];
  targetingConfig: TargetingConfig;    // per environment
}

interface StaleAnalysisResult {
  recommendation: 'remove' | 'keep' | 'review';
  confidence: number;                   // 0-1
  reasoning: string;                    // plain language explanation
  estimatedLinesRemoved: number;
  riskAssessment: string;
  suggestedActions: string[];
}

async function analyzeStaleFlag(input: StaleAnalysisInput): Promise<StaleAnalysisResult> {
  const response = await client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    system: `You are a feature flag lifecycle analyst. Analyse the flag data
      and recommend whether it should be removed, kept, or reviewed.
      Consider: evaluation trends, code references, flag type (release flags
      should be short-lived per Martin Fowler's taxonomy), targeting config
      (is it fully rolled out?), and time since creation.`,
    messages: [{
      role: 'user',
      content: JSON.stringify(input, null, 2),
    }],
  });
  // Parse structured response
}
```

Analysis runs:
1. Automatically: daily for all flags with staleness score > 50
2. On demand: user clicks "Analyze" on flag detail page

Results stored in `flag_lifecycle.ai_analysis` JSONB column.

**Testing**:
- `ai-stale-recommend-remove`: Fully rolled out release flag with zero evaluations → "remove"
- `ai-stale-recommend-keep`: Ops flag with stable evaluations → "keep"
- `ai-stale-confidence`: Confidence score between 0 and 1
- `ai-stale-reasoning`: Reasoning text references specific data points
- `ai-rate-limit`: Gracefully handles API rate limits with exponential backoff
- `ai-fallback`: Returns rule-based analysis when API is unavailable

#### 8.2 — Natural-Language Targeting Rule Authoring

**What**: Users describe targeting intent in plain English; LLM generates the corresponding structured targeting rule.

**Design**:

```typescript
// Route:
// POST /api/v1/ai/targeting-rule → TargetingRule

interface NLTargetingRequest {
  description: string;
  // e.g. "Roll this out to enterprise customers in the US and Canada
  //       who have used the feature at least 10 times"
  projectId: string;        // for segment context
  flagKey: string;          // for variant context
}

async function generateTargetingRule(req: NLTargetingRequest): Promise<TargetingRule> {
  // 1. Fetch existing segments and flag variants for context
  // 2. Prompt LLM with:
  //    - Available attributes (from existing segment conditions)
  //    - Available operators (full list)
  //    - Available variants
  //    - User's natural language description
  // 3. LLM returns structured TargetingRule JSON
  // 4. Validate returned JSON against TargetingRule schema
  // 5. Return to frontend for user review before applying
}
```

The generated rule is presented in the dashboard for user review and editing before being applied. The LLM never directly modifies flag configuration.

**Testing**:
- `ai-targeting-simple`: "Roll out to 50% of users" → catch-all rule with 50/50 distribution
- `ai-targeting-segment`: "Enterprise customers in the US" → conditions with plan=enterprise AND country=US
- `ai-targeting-complex`: Multi-condition description → correctly structured rule
- `ai-targeting-validation`: Generated rule passes JSON Schema validation
- `ai-targeting-preview`: Frontend displays generated rule for review before applying

#### 8.3 — AI Experiment Result Summaries

**What**: LLM generates plain-language summaries of experiment results for non-technical stakeholders.

**Design**:

```typescript
// Route:
// POST /api/v1/experiments/:id/summarize → { summary: string }

async function summarizeExperimentResults(experimentId: string): Promise<string> {
  // 1. Fetch experiment with results, config, and flag info
  // 2. Prompt LLM:
  //    - Experiment hypothesis
  //    - Statistical results per variant (lift, significance, CI)
  //    - Guardrail metric status
  //    - Sample size and duration
  // 3. LLM returns 3-5 sentence plain-language summary covering:
  //    - What happened (winner, lift magnitude)
  //    - Statistical confidence
  //    - Guardrail status
  //    - Recommended next action
}

// Example output:
// "Treatment A increased conversion rate by 14.3% (from 4.2% to 4.8%),
//  which is statistically significant (p=0.023, 95% CI: [0.2%, 1.0%]).
//  All guardrail metrics stayed within acceptable bounds. We recommend
//  rolling out Treatment A to all users."
```

**Testing**:
- `ai-summary-significant`: Significant result summary includes lift and confidence
- `ai-summary-not-significant`: Non-significant result correctly stated
- `ai-summary-guardrail-violation`: Summary warns about guardrail breach
- `ai-summary-recommendation`: Summary includes actionable next step

---

## Phase 9: B2B Account-Level Targeting

### Purpose

Add first-class support for company/account-level targeting, account traits, and adoption metrics. After this phase, B2B SaaS teams can roll out features per account with usage-based segmentation.

### Tasks

#### 9.1 — Account Entity and Traits

**What**: Account management API for storing B2B customer accounts with custom traits (plan tier, MRR, employee count, custom attributes).

**Design**:

```typescript
// No separate accounts table — accounts are evaluation contexts.
// Account data comes from the SDK context at evaluation time:
// context: {
//   kind: 'multi',
//   user: { key: 'user-123', attributes: { email: '...' } },
//   account: { key: 'acme-corp', attributes: { plan: 'enterprise', mrr: 50000, country: 'US' } }
// }

// Optional: account trait storage for segment building without requiring
// the SDK to send all traits on every evaluation.

// Route:
// PUT /api/v1/projects/:slug/accounts/:key → Account
// GET /api/v1/projects/:slug/accounts/:key → Account
// GET /api/v1/projects/:slug/accounts      → Account[] (paginated)

interface Account {
  key: string;                // external account ID
  name: string;
  traits: Record<string, unknown>;  // arbitrary key-value pairs
  lastSeenAt: Date;
  createdAt: Date;
  updatedAt: Date;
}

// Drizzle schema (optional — traits can also be purely context-based)
export const accounts = pgTable('accounts', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  key: varchar('key', { length: 255 }).notNull(),
  name: varchar('name', { length: 255 }).notNull(),
  traits: jsonb('traits').notNull().default({}),
  lastSeenAt: timestamp('last_seen_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_accounts_project_key').on(table.projectId, table.key),
]);
```

**Testing**:
- `account-upsert`: PUT creates or updates account with traits
- `account-traits-query`: Segment conditions can target account traits
- `account-list-pagination`: GET returns paginated account list with trait search
- `account-context-eval`: Flag evaluation with account context matches account-level rules

#### 9.2 — Account-Level Targeting in the Evaluation Engine

**What**: Extend the evaluation engine and dashboard to support account-level targeting rules and rollouts.

**Design**:

Evaluation engine changes:
```typescript
// Extended condition matching for multi-context:
// Rule conditions can specify contextKind:
// { "contextKind": "account", "attribute": "plan", "operator": "equals", "value": "enterprise" }

// Consistent hashing by account:
// Rule distribution with bucketBy: "account_id" ensures all users in the same account
// get the same variant.

// Evaluation priority with multi-context:
// 1. Check overrides for user context
// 2. Check overrides for account context
// 3. Evaluate rules (conditions may reference user OR account attributes)
// 4. Hash by the rule's bucketBy field (user_id or account_id)
```

Dashboard additions:
- Targeting rule builder: context kind selector (user/account) for each condition
- Distribution: bucket-by selector (hash by user_id or account_id)
- Account browser: list accounts, view their traits, see which flags target them

**Testing**:
- `b2b-account-targeting`: Rule targeting account.plan=enterprise matches correctly
- `b2b-bucket-by-account`: All users in same account get same variant
- `b2b-mixed-conditions`: Rule with user AND account conditions evaluates correctly
- `b2b-account-override`: Account-level override applies to all users in that account
- `ui-b2b-context-selector`: Dashboard shows context kind dropdown in targeting builder

#### 9.3 — Account Adoption Metrics

**What**: Track feature adoption at the account level — how many users within each account have been exposed to and engaged with a feature.

**Design**:

```typescript
// Route:
// GET /api/v1/projects/:slug/flags/:key/adoption → AccountAdoption[]

interface AccountAdoption {
  accountKey: string;
  accountName: string;
  totalUsers: number;           // users in the account
  exposedUsers: number;         // users who evaluated the flag
  activeUsers: number;          // users who triggered a metric event
  adoptionRate: number;         // activeUsers / totalUsers
  variant: string;              // which variant they received
}

// Computed from experiment_exposures + metric events + account membership
// Background worker aggregates daily
```

**Testing**:
- `adoption-per-account`: Adoption rate correctly computed for 3 accounts
- `adoption-variant-breakdown`: Different accounts may receive different variants
- `adoption-dashboard`: Dashboard renders adoption table with sortable columns

---

## Phase 10: Approval Workflows and Change Requests

### Purpose

Implement approval workflows for protected environments, enabling change requests that require reviewer approval before flag changes are applied to production.

### Tasks

#### 10.1 — Change Request System

**What**: When a flag change targets a protected environment, create a change request instead of applying immediately.

**Design**:

```typescript
// Drizzle schema
export const changeRequests = pgTable('change_requests', {
  id: uuid('id').primaryKey().defaultRandom(),
  flagId: uuid('flag_id').notNull().references(() => flags.id, { onDelete: 'cascade' }),
  environmentId: uuid('environment_id').notNull().references(() => environments.id, { onDelete: 'cascade' }),
  title: varchar('title', { length: 255 }).notNull(),
  description: text('description'),
  status: varchar('status', { length: 20 }).notNull().default('pending'),
  proposedChanges: jsonb('proposed_changes').notNull(),
  requestedBy: uuid('requested_by').notNull().references(() => users.id),
  reviewedBy: uuid('reviewed_by').references(() => users.id),
  reviewedAt: timestamp('reviewed_at', { withTimezone: true }),
  appliedAt: timestamp('applied_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_change_requests_pending').on(table.status),
]);

// Routes:
// POST   /api/v1/projects/:slug/change-requests          → ChangeRequest
// GET    /api/v1/projects/:slug/change-requests          → ChangeRequest[]
// GET    /api/v1/projects/:slug/change-requests/:id      → ChangeRequest
// POST   /api/v1/projects/:slug/change-requests/:id/approve → ChangeRequest
// POST   /api/v1/projects/:slug/change-requests/:id/reject  → ChangeRequest
// POST   /api/v1/projects/:slug/change-requests/:id/apply   → ChangeRequest
// DELETE /api/v1/projects/:slug/change-requests/:id/cancel  → 204

// Flow:
// 1. User modifies flag config for protected environment
// 2. Instead of direct PATCH, system creates ChangeRequest with proposed_changes
// 3. Reviewer approves or rejects (with comment)
// 4. If approved, requester (or admin) applies the change
// 5. Applied change is logged in audit with change_request reference
```

**Testing**:
- `cr-protected-env-redirect`: PATCH to protected env config creates change request instead
- `cr-non-protected-direct`: PATCH to non-protected env applies directly
- `cr-approve-flow`: Approve → apply updates flag config
- `cr-reject-flow`: Reject prevents changes; status set to "rejected"
- `cr-self-approve-blocked`: Requester cannot approve their own change request
- `cr-apply-versioning`: Applied change respects optimistic concurrency (version check)

#### 10.2 — Scheduled Flag Changes

**What**: Apply flag configuration changes at a scheduled future time (e.g., "enable at 9am Monday").

**Design**:

```typescript
// Scheduled changes stored in flag_configs.scheduled_changes JSONB:
// [
//   { "id": "sc-1", "applyAt": "2026-05-15T09:00:00Z",
//     "changes": { "isEnabled": true }, "createdBy": "user-uuid" }
// ]

// Background worker: scheduled-changes.ts
// Runs every 60 seconds:
// 1. Query all flag_configs where scheduled_changes contains items with applyAt <= now
// 2. Apply each change (PATCH the flag_config)
// 3. Remove applied scheduled change from the array
// 4. Log to audit as actor_type: 'system'

// Route:
// POST /api/v1/projects/:slug/flags/:key/configs/:env/schedule → ScheduledChange
// DELETE /api/v1/projects/:slug/flags/:key/configs/:env/schedule/:id → 204
```

**Testing**:
- `schedule-create`: Create scheduled change for future time
- `schedule-apply`: Worker applies change at scheduled time
- `schedule-audit`: Applied change logged with actor_type "system"
- `schedule-cancel`: Deleting scheduled change prevents application
- `schedule-protected-env`: Scheduled change for protected env creates change request at apply time

---

## Phase 11: Code References and Cleanup PR Generation

### Purpose

Implement code reference scanning to find flag keys in source repositories, and AI-powered cleanup PR generation that removes stale flag code. This is the highest-impact AI differentiator — no existing tool does this end-to-end.

### Tasks

#### 11.1 — Code Reference Scanner CLI

**What**: CLI tool that scans a Git repository for flag key references and reports them to the server.

**Design**:

```typescript
// tools/code-references/src/scanner.ts

interface ScanResult {
  flagKey: string;
  repository: string;
  filePath: string;
  lineNumber: number;
  snippet: string;               // surrounding 3 lines
  branch: string;
  commitSha: string;
}

// CLI usage:
// ffm scan --api-key=ffm_srv_... --repo=acme/web-app --dir=./src

// Scanner algorithm:
// 1. Fetch all flag keys for the project from the API
// 2. For each file in the directory tree:
//    a. Skip binary files, node_modules, .git, vendor directories
//    b. Search for each flag key as a string literal
//    c. Record file path, line number, and surrounding context
// 3. POST results to /api/v1/projects/:slug/code-references

// Supported patterns:
// - String literals: 'checkout-v2', "checkout-v2"
// - SDK method calls: getFlag('checkout-v2'), client.getBooleanValue('checkout-v2')
// - Template literals: `${flagPrefix}-v2` (partial match)

// Route:
// POST /api/v1/projects/:slug/code-references → 204
// GET  /api/v1/projects/:slug/flags/:key/code-references → CodeReference[]
```

GitHub Action for automated scanning:
```yaml
# .github/workflows/ffm-scan.yml
on:
  push:
    branches: [main]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npx @ffm/code-references scan
        env:
          FFM_API_KEY: ${{ secrets.FFM_API_KEY }}
          FFM_BASE_URL: ${{ vars.FFM_BASE_URL }}
```

**Testing**:
- `scan-find-flag`: Scanner finds flag key in TypeScript/JavaScript source
- `scan-find-python`: Scanner finds flag key in Python source
- `scan-skip-binary`: Binary files and node_modules skipped
- `scan-report-upload`: Scan results uploaded to server via API
- `scan-display-refs`: Dashboard shows code references for each flag

#### 11.2 — AI-Powered Cleanup PR Generation

**What**: For stale flags, AI generates a code diff that removes the flag and its conditional logic, then opens a pull request via the GitHub API.

**Design**:

```typescript
// apps/server/src/services/cleanup-service.ts

interface CleanupPlan {
  flagKey: string;
  winningVariant: string;          // the variant to keep (typically the one at 100%)
  codeReferences: CodeReference[];
  changes: FileChange[];
}

interface FileChange {
  filePath: string;
  repository: string;
  before: string;                  // original code
  after: string;                   // cleaned code
  explanation: string;             // why this change
}

async function generateCleanupPlan(flagKey: string): Promise<CleanupPlan> {
  // 1. Get flag details and identify winning variant
  // 2. Get code references
  // 3. For each code reference, fetch file content from GitHub API
  // 4. Prompt LLM to generate code transformation:
  //    - Remove flag check conditional
  //    - Keep the winning variant code path
  //    - Remove losing variant code path
  //    - Clean up unused imports
  // 5. Return plan for user review

  const response = await client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 4096,
    system: `You are a code cleanup specialist. Given a feature flag key,
      the winning variant value, and the source code containing the flag check,
      generate the cleaned code that:
      1. Removes the flag evaluation call
      2. Keeps only the winning variant's code path
      3. Removes dead code from losing variants
      4. Cleans up any now-unused imports or variables
      Return a JSON array of {filePath, before, after, explanation}.`,
    messages: [{
      role: 'user',
      content: JSON.stringify({
        flagKey,
        winningVariant: 'treatment_a',
        winningValue: 'true',
        codeReferences: [
          { filePath: 'src/checkout.tsx', content: '...' },
        ],
      }),
    }],
  });
}

// Route:
// POST /api/v1/projects/:slug/flags/:key/cleanup-plan → CleanupPlan
// POST /api/v1/projects/:slug/flags/:key/cleanup-pr   → { prUrl: string }

async function createCleanupPR(plan: CleanupPlan): Promise<string> {
  // 1. User reviews and approves cleanup plan
  // 2. Use GitHub API (via octokit) to:
  //    a. Create branch: `ffm/cleanup-{flagKey}`
  //    b. Apply file changes via Contents API
  //    c. Create PR with title "Remove stale flag: {flagKey}"
  //    d. PR body includes: flag details, staleness score, AI reasoning
  // 3. Store PR URL in flag_lifecycle.cleanup_pr_url
  // 4. Return PR URL
}
```

**Testing**:
- `cleanup-plan-simple`: Boolean flag if/else generates correct cleanup (removes if, keeps winning branch)
- `cleanup-plan-multivariant`: Switch/match statement cleanup removes non-winning cases
- `cleanup-plan-import-cleanup`: Unused SDK import removed
- `cleanup-pr-creation`: PR created on GitHub with correct branch, title, and body
- `cleanup-pr-stored`: PR URL stored in flag_lifecycle
- `cleanup-review-required`: Plan shown to user for review before PR creation

---

## Phase 12: OpenFeature OFREP and Edge Evaluation

### Purpose

Implement the OpenFeature Remote Evaluation Protocol (OFREP) for cross-platform compatibility, and add edge evaluation support for sub-5ms latency. After this phase, the platform is fully OpenFeature-aligned and production-ready at scale.

### Tasks

#### 12.1 — OpenFeature OFREP Server Endpoints

**What**: Implement the OFREP specification endpoints so any OpenFeature-compatible SDK can evaluate flags from this server.

**Design**:

```typescript
// OFREP endpoints (aligned with OpenFeature specification):

// POST /ofrep/v1/evaluate/flags/:key
// Request:
// {
//   "context": {
//     "targetingKey": "user-123",
//     "email": "alice@acme.com",
//     "plan": "enterprise"
//   }
// }
// Response:
// {
//   "key": "checkout-v2",
//   "reason": "TARGETING_MATCH",
//   "variant": "treatment_a",
//   "value": true,
//   "metadata": { "ruleId": "rule-1" }
// }

// POST /ofrep/v1/evaluate/flags (bulk evaluation)
// Request: { "context": { ... } }
// Response: { "flags": [ ... ] }

// GET /ofrep/v1/configuration
// Returns: provider metadata, supported flag types

// Authentication: same API token scheme as /api/v1/evaluate
// Context mapping: OpenFeature context → FFM EvaluationContext
```

**Testing**:
- `ofrep-single-eval`: OFREP single flag evaluation returns correct format
- `ofrep-bulk-eval`: OFREP bulk evaluation returns all flags
- `ofrep-context-mapping`: OpenFeature context correctly mapped to FFM context
- `ofrep-error-flag-not-found`: Returns OFREP-standard error for unknown flag
- `ofrep-auth`: Unauthenticated request returns 401 in OFREP format
- `ofrep-openfeature-sdk`: Third-party OpenFeature SDK (e.g., Go SDK) can evaluate via OFREP

#### 12.2 — Edge Evaluation via Redis Cache

**What**: Push flag configurations to Redis for sub-millisecond evaluation at the edge, with cache invalidation via pub/sub.

**Design**:

```typescript
// Edge sync service:
// On every flag_config change:
// 1. Serialize flag config to the SDK wire format (JSON)
// 2. Write to Redis:
//    Key: `edge:{projectId}:{envId}:{flagKey}`
//    Value: serialized flag config
//    TTL: none (explicitly invalidated)
// 3. Publish invalidation event to Redis pub/sub

// Edge evaluation endpoint (read from Redis, not PostgreSQL):
// GET /api/v1/edge/evaluate/flags/:key → EvaluationResult
// - Read flag config from Redis
// - Run evaluation engine
// - Return result
// - Latency target: <5ms p99

// Bulk edge evaluation:
// GET /api/v1/edge/evaluate/flags → Record<string, EvaluationResult>
// - Redis MGET for all flag configs in environment
// - Evaluate all
// - Return results
```

Future extension: Cloudflare Workers KV sync for true CDN-edge evaluation (documented but not implemented in this phase).

**Testing**:
- `edge-cache-populated`: Flag config change writes to Redis
- `edge-eval-from-cache`: Edge evaluation reads from Redis (not PostgreSQL)
- `edge-cache-invalidation`: Config update invalidates Redis cache
- `edge-eval-latency`: Edge evaluation completes in <5ms (benchmark test)
- `edge-fallback-to-db`: If Redis unavailable, falls back to PostgreSQL evaluation

#### 12.3 — Docker Production Image and Deployment Documentation

**What**: Multi-stage Docker image, Docker Compose for self-hosted deployment, and deployment documentation.

**Design**:

```dockerfile
# Dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY pnpm-lock.yaml package.json pnpm-workspace.yaml turbo.json ./
COPY packages/ ./packages/
COPY apps/server/ ./apps/server/
COPY apps/dashboard/ ./apps/dashboard/
RUN corepack enable && pnpm install --frozen-lockfile
RUN pnpm turbo build --filter=server --filter=dashboard

FROM node:22-alpine AS runner
WORKDIR /app
COPY --from=builder /app/apps/server/dist ./server/
COPY --from=builder /app/apps/dashboard/dist ./dashboard/
COPY --from=builder /app/node_modules ./node_modules/
ENV NODE_ENV=production
EXPOSE 8080
CMD ["node", "server/server.js"]
```

```yaml
# docker-compose.yml (self-hosted deployment)
services:
  ffm:
    image: ffm/feature-flag-system:latest
    ports: ["8080:8080"]
    environment:
      DATABASE_URL: postgres://ffm:password@postgres:5432/ffm
      REDIS_URL: redis://redis:6379
      FFM_SECRET: <generate-random-secret>
    depends_on: [postgres, redis]

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ffm
      POSTGRES_USER: ffm
      POSTGRES_PASSWORD: password
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    volumes: ["redisdata:/data"]

volumes:
  pgdata:
  redisdata:
```

**Testing**:
- `docker-build`: Docker image builds successfully
- `docker-compose-up`: Full stack starts with `docker compose up`
- `docker-health-check`: Health endpoint returns 200 after startup
- `docker-migration-auto`: Database migrations run automatically on startup
- `docker-static-serving`: Dashboard SPA served from server at /

---

## Phase Summary & Dependencies

```
Phase 1: Foundation ─────────────────────────────────────────┐
  (DB, Auth, Org/Project/Env)                                │
                                                             ▼
Phase 2: Flag CRUD ──────────────────────────────────────────┤
  (Flags, Variants, Per-Env Config, Audit)                   │
                                                             ▼
Phase 3: Evaluation Engine ──────────────────────────────────┤
  (Targeting, Hashing, Segments, SDK Endpoint)               │
                         ┌───────────────────────────────────┤
                         ▼                                   ▼
Phase 4: Real-Time + SDK ─────┐         Phase 5: Experimentation
  (SSE, Webhooks, Node SDK)   │           (Stats, CUPED, Bayesian)
                         │    │                              │
                         ▼    ▼                              ▼
                    Phase 6: Dashboard UI ───────────────────┤
                      (Flags, Experiments, Audit)            │
                                                             ▼
Phase 7: Lifecycle ──────────────────────────────────────────┤
  (Eval Stats, Staleness, Health Dashboard)                  │
                         │                                   │
                         ▼                                   │
Phase 8: AI Features ───────────────────────────────────────┤
  (Stale Analysis, NL Targeting, Summaries)                  │
                                                             │
Phase 9: B2B Targeting ──────────────────────────────────────┤
  (Accounts, Account-Level Rules, Adoption)                  │
                                                             │
Phase 10: Approval Workflows ────────────────────────────────┤
  (Change Requests, Scheduled Changes)                       │
                                                             │
Phase 11: Code References + Cleanup ─────────────────────────┤
  (Scanner CLI, AI Cleanup PRs)                              │
                                                             │
Phase 12: OFREP + Edge + Docker ─────────────────────────────┘
  (OpenFeature OFREP, Redis Edge, Production Image)
```

Phases 4 and 5 can be developed in parallel after Phase 3 completes.
Phases 7-11 can be partially parallelised but each builds on the preceding data.
Phase 12 should be last as it is the production hardening phase.

---

## Definition of Done (per phase)

Each phase is considered complete when ALL of the following are satisfied:

1. **All tasks implemented**: Every task in the phase has working code merged to main.
2. **All tests passing**: Every named test scenario passes in CI (Vitest + Playwright + Supertest).
3. **Type-safe**: `pnpm turbo typecheck` passes with zero errors across all packages.
4. **Linted**: `pnpm turbo lint` passes with zero warnings.
5. **API documented**: All new endpoints have OpenAPI schemas generated by Fastify route definitions.
6. **Database migrations**: Schema changes have forward migrations that apply cleanly to existing databases.
7. **No regressions**: All tests from previous phases continue to pass.
8. **Docker builds**: `docker build` produces a working image (from Phase 1 onward).
9. **Reviewed**: Code has been reviewed for security (no credential leaks, no SQL injection, proper auth checks on all endpoints).
10. **Seed data updated**: Development seed script includes examples exercising the new functionality.
