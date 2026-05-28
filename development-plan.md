# Press Release & PR Management — Phased Development Plan

> Project: 174-press-release-pr-management · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | TypeScript (Node.js) | Full-stack language unification with the frontend; strong ecosystem for email handling, REST APIs, and webhook integrations; first-class Prisma ORM support for PostgreSQL; excellent async/await patterns for concurrent email sending and monitoring jobs |
| Language (AI services) | Python 3.12+ | ML/NLP ecosystem (scikit-learn, spaCy, sentence-transformers) for journalist matching, sentiment analysis, and send-time prediction; Anthropic/OpenAI SDKs for LLM-powered release drafting and pitch personalisation |
| API framework | Fastify (TypeScript) | Fastest Node.js HTTP framework; built-in schema validation via JSON Schema (aligns with OpenAPI 3.1 spec generation); first-class plugin architecture for modular feature development |
| Frontend | Next.js 15 (App Router) | React-based SPA with server components for dashboard performance; built-in API routes for BFF pattern; excellent TypeScript integration; Tailwind CSS + shadcn/ui for rapid component development |
| Database | PostgreSQL 16 | JSONB columns for variable journalist metadata and AI predictions (Hybrid Relational + JSONB model from data-model-suggestion-3); GIN indexes for containment queries; `ltree` extension for beat hierarchies; row-level security for multi-tenancy; full-text search for journalist and coverage queries |
| ORM | Prisma 6 | Type-safe database client with auto-generated TypeScript types; migration management; supports JSONB columns and PostgreSQL-specific features |
| Task queue | BullMQ (Redis) | Handles async workloads: email sending, media monitoring polling, AI inference jobs, coverage discovery, analytics recomputation; rate limiting for email deliverability; scheduled/delayed jobs for follow-up sequences |
| Cache | Redis 7 | Shared cache for API responses, session data, rate limiting, and BullMQ job queue backing store |
| Email sending | Resend (primary), Nodemailer (fallback) | Resend provides DKIM/SPF/DMARC compliance (RFC 6376, RFC 7208), webhook delivery events, and React Email template support; Nodemailer fallback for self-hosted SMTP |
| Search | Meilisearch | Lightweight full-text search for journalist database, press release content, and coverage articles; typo tolerance; faceted filtering by beat, outlet, country; simpler to self-host than Elasticsearch |
| AI/LLM provider | Anthropic Claude API (primary), OpenAI (fallback) | Claude for press release drafting, pitch personalisation, and coverage summarisation; function calling for structured extraction; prompt caching for cost efficiency |
| NLP/ML | Python microservice (FastAPI) | Sentiment analysis (transformers), journalist-beat matching (sentence-transformers embeddings), send-time prediction (scikit-learn), style guide extraction (spaCy + LLM) |
| File storage | S3-compatible (MinIO for self-hosted) | Press release attachments (images, videos, documents), newsroom assets, exported reports |
| Authentication | NextAuth.js v5 | Email/password, Google OAuth, SAML 2.0 (enterprise); JWT session tokens; multi-tenant org context |
| Containerisation | Docker + Docker Compose | Multi-service orchestration: API, frontend, worker, AI service, PostgreSQL, Redis, Meilisearch, MinIO |
| Testing | Vitest (TypeScript), pytest (Python) | Vitest for unit/integration tests on API and frontend; Playwright for E2E; pytest for AI/ML service tests |
| Code quality | ESLint + Prettier (TS), Ruff (Python) | TypeScript: ESLint flat config + Prettier. Python: Ruff for linting and formatting. Both enforced via pre-commit hooks |
| API documentation | OpenAPI 3.1 (auto-generated from Fastify schemas) | Fastify's @fastify/swagger generates OpenAPI spec from route schemas; Scalar for interactive API docs |
| Monorepo | Turborepo | Manages `apps/api`, `apps/web`, `apps/worker`, `services/ai`, `packages/shared` with cached builds |

### Data Model Selection

The **Hybrid Relational + JSONB** model (data-model-suggestion-3) is selected as the primary data model. Rationale:

1. **17 tables vs. 30** in fully normalized — faster MVP development
2. **JSONB columns** absorb jurisdiction-specific compliance fields (GDPR, CAN-SPAM, CASL) without per-regulation migrations
3. **Relational integrity** preserved for core entities (journalist, outlet, campaign, pitch, press_release)
4. **GIN indexes** on JSONB enable efficient containment queries for journalist targeting and coverage filtering
5. **Compatible with graph extension later** — graph_node/graph_edge tables from data-model-suggestion-4 can be added in a later phase for influence network analysis without reworking core tables

### Project Structure

```
press-release-pr-management/
├── turbo.json
├── package.json
├── docker-compose.yml
├── docker-compose.dev.yml
├── .env.example
├── apps/
│   ├── api/                          # Fastify REST API
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── src/
│   │   │   ├── index.ts              # Server entry point
│   │   │   ├── config.ts             # Environment config with defaults
│   │   │   ├── plugins/              # Fastify plugins (auth, multitenancy, rate-limit)
│   │   │   ├── routes/               # Route modules by domain
│   │   │   │   ├── auth/
│   │   │   │   ├── journalists/
│   │   │   │   ├── outlets/
│   │   │   │   ├── media-lists/
│   │   │   │   ├── press-releases/
│   │   │   │   ├── newsrooms/
│   │   │   │   ├── campaigns/
│   │   │   │   ├── pitches/
│   │   │   │   ├── coverage/
│   │   │   │   ├── monitoring/
│   │   │   │   ├── distribution/
│   │   │   │   ├── analytics/
│   │   │   │   └── ai/
│   │   │   ├── services/             # Business logic layer
│   │   │   │   ├── journalist.service.ts
│   │   │   │   ├── press-release.service.ts
│   │   │   │   ├── campaign.service.ts
│   │   │   │   ├── pitch.service.ts
│   │   │   │   ├── coverage.service.ts
│   │   │   │   ├── monitoring.service.ts
│   │   │   │   ├── distribution.service.ts
│   │   │   │   ├── analytics.service.ts
│   │   │   │   └── ai.service.ts
│   │   │   ├── hooks/                # Fastify lifecycle hooks
│   │   │   └── utils/
│   │   └── tests/
│   │       ├── unit/
│   │       ├── integration/
│   │       └── fixtures/
│   ├── web/                          # Next.js frontend
│   │   ├── package.json
│   │   ├── next.config.ts
│   │   ├── tailwind.config.ts
│   │   ├── src/
│   │   │   ├── app/                  # App Router pages
│   │   │   │   ├── (auth)/           # Login, register, SSO
│   │   │   │   ├── (dashboard)/      # Authenticated app
│   │   │   │   │   ├── journalists/
│   │   │   │   │   ├── press-releases/
│   │   │   │   │   ├── newsroom/
│   │   │   │   │   ├── campaigns/
│   │   │   │   │   ├── coverage/
│   │   │   │   │   ├── monitoring/
│   │   │   │   │   ├── analytics/
│   │   │   │   │   └── settings/
│   │   │   │   └── layout.tsx
│   │   │   ├── components/
│   │   │   │   ├── ui/               # shadcn/ui primitives
│   │   │   │   ├── journalists/
│   │   │   │   ├── press-releases/
│   │   │   │   ├── campaigns/
│   │   │   │   ├── coverage/
│   │   │   │   └── analytics/
│   │   │   ├── hooks/
│   │   │   ├── lib/                  # API client, utils
│   │   │   └── stores/               # Zustand state
│   │   └── tests/
│   │       ├── components/
│   │       └── e2e/                  # Playwright tests
│   └── worker/                       # BullMQ job processors
│       ├── package.json
│       ├── src/
│       │   ├── index.ts
│       │   ├── queues/               # Queue definitions
│       │   │   ├── email.queue.ts
│       │   │   ├── monitoring.queue.ts
│       │   │   ├── ai.queue.ts
│       │   │   └── analytics.queue.ts
│       │   ├── processors/           # Job handlers
│       │   │   ├── send-pitch.processor.ts
│       │   │   ├── send-followup.processor.ts
│       │   │   ├── monitor-coverage.processor.ts
│       │   │   ├── compute-scores.processor.ts
│       │   │   └── ai-inference.processor.ts
│       │   └── utils/
│       └── tests/
├── services/
│   └── ai/                           # Python AI/ML microservice
│       ├── pyproject.toml
│       ├── Dockerfile
│       ├── src/
│       │   ├── main.py               # FastAPI entry point
│       │   ├── routers/
│       │   │   ├── sentiment.py
│       │   │   ├── journalist_match.py
│       │   │   ├── send_time.py
│       │   │   ├── style_guide.py
│       │   │   └── draft.py
│       │   ├── models/               # ML model wrappers
│       │   ├── prompts/              # LLM prompt templates
│       │   └── utils/
│       └── tests/
├── packages/
│   └── shared/                       # Shared TypeScript types & utils
│       ├── package.json
│       ├── src/
│       │   ├── types/                # Shared API types
│       │   │   ├── journalist.ts
│       │   │   ├── press-release.ts
│       │   │   ├── campaign.ts
│       │   │   ├── pitch.ts
│       │   │   ├── coverage.ts
│       │   │   └── analytics.ts
│       │   ├── constants/
│       │   └── validation/           # Zod schemas
│       └── tests/
├── prisma/
│   ├── schema.prisma
│   └── migrations/
└── scripts/
    ├── seed.ts                       # Development seed data
    ├── migrate.ts
    └── setup-dev.sh
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the monorepo structure, development tooling, database schema, authentication system, and multi-tenant organization model. After this phase, a developer can register an account, create an organization, and authenticate against the API. This is the structural foundation for all subsequent phases.

### Tasks

#### 1.1 — Monorepo & Tooling Setup

**What**: Initialize Turborepo monorepo with all packages, configure TypeScript, ESLint, Prettier, Docker Compose, and CI pipeline.

**Design**:

Root `turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["build"] },
    "lint": {},
    "typecheck": {}
  }
}
```

Root `package.json` scripts:
```json
{
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "test": "turbo test",
    "lint": "turbo lint",
    "typecheck": "turbo typecheck",
    "db:migrate": "prisma migrate deploy",
    "db:seed": "tsx scripts/seed.ts"
  }
}
```

`docker-compose.dev.yml`:
```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: prmanagement
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: devpassword
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  meilisearch:
    image: getmeili/meilisearch:v1.11
    ports: ["7700:7700"]
    environment:
      MEILI_ENV: development
      MEILI_MASTER_KEY: dev-master-key
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports: ["9000:9000", "9001:9001"]
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
volumes:
  pgdata:
```

Environment config (`apps/api/src/config.ts`):
```typescript
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.coerce.number().default(3001),
  DATABASE_URL: z.string(),
  REDIS_URL: z.string().default('redis://localhost:6379'),
  MEILISEARCH_URL: z.string().default('http://localhost:7700'),
  MEILISEARCH_KEY: z.string().default('dev-master-key'),
  S3_ENDPOINT: z.string().default('http://localhost:9000'),
  S3_ACCESS_KEY: z.string(),
  S3_SECRET_KEY: z.string(),
  S3_BUCKET: z.string().default('pr-management'),
  JWT_SECRET: z.string(),
  RESEND_API_KEY: z.string().optional(),
  ANTHROPIC_API_KEY: z.string().optional(),
  AI_SERVICE_URL: z.string().default('http://localhost:8000'),
  APP_URL: z.string().default('http://localhost:3000'),
});

export type Env = z.infer<typeof envSchema>;
export const env = envSchema.parse(process.env);
```

**Testing**:
- `Unit: envSchema.parse with all required vars -> valid Env object`
- `Unit: envSchema.parse with missing DATABASE_URL -> ZodError with field name`
- `Unit: envSchema.parse with defaults -> PORT=3001, REDIS_URL=redis://localhost:6379`
- `Integration: docker-compose up -> all services healthy within 30s`
- `Integration: turbo build -> all packages compile without errors`

#### 1.2 — Database Schema & Migrations

**What**: Implement the Hybrid Relational + JSONB data model (17 tables from data-model-suggestion-3) as Prisma schema with initial migration.

**Design**:

Prisma schema (core entities):
```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

model Organization {
  id        String   @id @default(uuid()) @db.Uuid
  name      String   @db.VarChar(255)
  slug      String   @unique @db.VarChar(100)
  planTier  String   @default("free") @db.VarChar(50) @map("plan_tier")
  settings  Json     @default("{}")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  users          AppUser[]
  mediaLists     MediaList[]
  pressReleases  PressRelease[]
  newsrooms      Newsroom[]
  campaigns      Campaign[]
  coverages      Coverage[]
  monitoringQueries MonitoringQuery[]
  journalistScores  JournalistScore[]
  auditLogs      AuditLog[]

  @@map("organization")
}

model AppUser {
  id             String   @id @default(uuid()) @db.Uuid
  organizationId String   @map("organization_id") @db.Uuid
  email          String   @db.VarChar(255)
  fullName       String   @map("full_name") @db.VarChar(255)
  roles          String[] @default(["viewer"])
  profile        Json     @default("{}")
  isActive       Boolean  @default(true) @map("is_active")
  createdAt      DateTime @default(now()) @map("created_at")
  updatedAt      DateTime @updatedAt @map("updated_at")

  organization   Organization @relation(fields: [organizationId], references: [id])

  @@unique([organizationId, email])
  @@index([organizationId])
  @@map("app_user")
}

model Journalist {
  id             String    @id @default(uuid()) @db.Uuid
  fullName       String    @map("full_name") @db.VarChar(255)
  email          String?   @db.VarChar(255)
  title          String?   @db.VarChar(255)
  countryCode    String?   @map("country_code") @db.Char(2)
  city           String?   @db.VarChar(255)
  beats          String[]  @default([])
  profile        Json      @default("{}")
  compliance     Json      @default("{}")
  isVerified     Boolean   @default(false) @map("is_verified")
  verifiedAt     DateTime? @map("verified_at")
  createdAt      DateTime  @default(now()) @map("created_at")
  updatedAt      DateTime  @updatedAt @map("updated_at")

  outlets          JournalistOutlet[]
  pitches          Pitch[]
  coverages        Coverage[]
  mediaListMembers MediaListMember[]
  scores           JournalistScore[]

  @@index([email])
  @@index([countryCode])
  @@map("journalist")
}

model Outlet {
  id              String  @id @default(uuid()) @db.Uuid
  name            String  @db.VarChar(255)
  outletType      String  @map("outlet_type") @db.VarChar(50)
  websiteUrl      String? @map("website_url") @db.VarChar(500)
  countryCode     String? @map("country_code") @db.Char(2)
  languageCode    String  @default("en") @map("language_code") @db.Char(2)
  properties      Json    @default("{}")
  isVerified      Boolean @default(false) @map("is_verified")
  createdAt       DateTime @default(now()) @map("created_at")
  updatedAt       DateTime @updatedAt @map("updated_at")

  journalists     JournalistOutlet[]
  coverages       Coverage[]

  @@index([outletType])
  @@index([countryCode])
  @@map("outlet")
}

model JournalistOutlet {
  id            String    @id @default(uuid()) @db.Uuid
  journalistId  String    @map("journalist_id") @db.Uuid
  outletId      String    @map("outlet_id") @db.Uuid
  roleAtOutlet  String?   @map("role_at_outlet") @db.VarChar(100)
  isCurrent     Boolean   @default(true) @map("is_current")
  startDate     DateTime? @map("start_date") @db.Date
  endDate       DateTime? @map("end_date") @db.Date
  createdAt     DateTime  @default(now()) @map("created_at")

  journalist    Journalist @relation(fields: [journalistId], references: [id], onDelete: Cascade)
  outlet        Outlet     @relation(fields: [outletId], references: [id], onDelete: Cascade)

  @@unique([journalistId, outletId, startDate])
  @@map("journalist_outlet")
}
```

Additional raw SQL migration for GIN indexes (run after Prisma migration):
```sql
CREATE INDEX idx_journalist_beats ON journalist USING GIN(beats);
CREATE INDEX idx_journalist_compliance ON journalist USING GIN(compliance jsonb_path_ops);
CREATE INDEX idx_outlet_properties ON outlet USING GIN(properties jsonb_path_ops);
CREATE INDEX idx_pr_content_meta ON press_release USING GIN(content_meta jsonb_path_ops);
CREATE INDEX idx_pitch_engagement ON pitch USING GIN(engagement jsonb_path_ops);
CREATE INDEX idx_coverage_analysis ON coverage USING GIN(analysis jsonb_path_ops);
CREATE INDEX idx_coverage_tags ON coverage USING GIN(tags);
```

**Testing**:
- `Unit: Prisma schema validation -> no errors`
- `Integration: prisma migrate deploy -> all 17 tables created`
- `Integration: prisma db seed -> sample data inserted (5 orgs, 20 users, 100 journalists, 50 outlets)`
- `Integration: GIN index on journalist.beats -> query with 'ai' = ANY(beats) uses index scan`
- `Integration: JSONB containment query on journalist.compliance -> @> operator uses GIN index`

#### 1.3 — Authentication & Multi-Tenancy

**What**: Implement user registration, login (email/password + Google OAuth), JWT session management, and organization-scoped request context.

**Design**:

Auth plugin for Fastify:
```typescript
// apps/api/src/plugins/auth.plugin.ts
import fp from 'fastify-plugin';
import jwt from '@fastify/jwt';

interface JwtPayload {
  userId: string;
  organizationId: string;
  email: string;
  roles: string[];
}

declare module 'fastify' {
  interface FastifyRequest {
    currentUser: JwtPayload;
    organizationId: string;
  }
}

export const authPlugin = fp(async (fastify) => {
  fastify.register(jwt, { secret: env.JWT_SECRET });

  fastify.decorate('authenticate', async (request: FastifyRequest, reply: FastifyReply) => {
    const token = await request.jwtVerify<JwtPayload>();
    request.currentUser = token;
    request.organizationId = token.organizationId;
  });
});
```

Auth routes:
```typescript
// POST /api/auth/register
interface RegisterBody {
  email: string;
  password: string;
  fullName: string;
  organizationName: string;
}
// Returns: { token: string; user: AppUser; organization: Organization }

// POST /api/auth/login
interface LoginBody {
  email: string;
  password: string;
}
// Returns: { token: string; user: AppUser }

// POST /api/auth/google
interface GoogleAuthBody {
  idToken: string;
}
// Returns: { token: string; user: AppUser }

// GET /api/auth/me
// Returns: { user: AppUser; organization: Organization }
```

Password hashing:
```typescript
import { hash, verify } from '@node-rs/argon2';

const ARGON2_OPTIONS = {
  memoryCost: 19456,
  timeCost: 2,
  outputLen: 32,
  parallelism: 1,
};
```

Multi-tenancy middleware: every query includes `WHERE organization_id = ?` automatically via Prisma client extension:
```typescript
// apps/api/src/plugins/multitenancy.plugin.ts
export function createTenantPrisma(organizationId: string) {
  return prisma.$extends({
    query: {
      $allOperations({ args, query, model }) {
        const tenantModels = [
          'MediaList', 'PressRelease', 'Newsroom', 'Campaign',
          'Coverage', 'MonitoringQuery', 'JournalistScore', 'AuditLog'
        ];
        if (tenantModels.includes(model)) {
          args.where = { ...args.where, organizationId };
        }
        return query(args);
      },
    },
  });
}
```

**Testing**:
- `Unit: password hash -> verify returns true for correct password`
- `Unit: password hash -> verify returns false for incorrect password`
- `Unit: JWT payload creation -> contains userId, organizationId, email, roles`
- `Integration: POST /api/auth/register -> creates organization + user, returns JWT`
- `Integration: POST /api/auth/register with existing email -> 409 Conflict`
- `Integration: POST /api/auth/login with valid credentials -> 200 with JWT`
- `Integration: POST /api/auth/login with invalid password -> 401 Unauthorized`
- `Integration: GET /api/auth/me with valid JWT -> returns user and organization`
- `Integration: GET /api/auth/me without JWT -> 401 Unauthorized`
- `Integration: tenant isolation -> user in org A cannot access org B data`

#### 1.4 — Audit Log Foundation

**What**: Implement audit logging service that records all create/update/delete actions with entity type, entity ID, changes diff, and actor.

**Design**:

```typescript
// apps/api/src/services/audit.service.ts
interface AuditEntry {
  organizationId: string;
  userId: string;
  action: 'create' | 'update' | 'delete' | 'publish' | 'approve' | 'send';
  entityType: string;
  entityId: string;
  changes?: Record<string, { old: unknown; new: unknown }>;
  ipAddress?: string;
}

export class AuditService {
  async log(entry: AuditEntry): Promise<void>;
  async getByEntity(entityType: string, entityId: string): Promise<AuditLog[]>;
  async getByOrg(organizationId: string, opts: { limit: number; offset: number }): Promise<AuditLog[]>;
}
```

Fastify hook for automatic audit trail:
```typescript
// Attach to onResponse hook — log after successful mutations
fastify.addHook('onResponse', async (request, reply) => {
  if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(request.method) && reply.statusCode < 400) {
    // Extract audit context from request.auditContext (set by route handlers)
    if (request.auditContext) {
      await auditService.log({
        organizationId: request.organizationId,
        userId: request.currentUser.userId,
        ...request.auditContext,
        ipAddress: request.ip,
      });
    }
  }
});
```

**Testing**:
- `Unit: AuditService.log -> inserts record with all fields`
- `Unit: AuditService.getByEntity -> returns records for specific entity`
- `Integration: POST /api/journalists -> audit log entry created with action=create`
- `Integration: PATCH /api/press-releases/:id -> audit log records changed fields`
- `Integration: audit log entries include IP address from request`

---

## Phase 2: Journalist Database & Media Lists

### Purpose
Build the journalist and outlet database with search, filtering, beat tracking, and GDPR-compliant contact management. Implement media list creation (static and dynamic). After this phase, PR teams can manage their journalist contacts, search by beat/outlet/geography, build targeted media lists, and maintain regulatory compliance with consent tracking.

### Tasks

#### 2.1 — Journalist & Outlet CRUD

**What**: REST API endpoints for creating, reading, updating, and deleting journalists and outlets with JSONB profile and compliance data.

**Design**:

```typescript
// packages/shared/src/types/journalist.ts
interface JournalistProfile {
  phone?: string;
  bio?: string;
  social?: {
    twitter?: string;
    linkedin?: string;
    mastodon?: string;
  };
  preferences?: {
    contactMethod?: 'email' | 'twitter_dm' | 'phone';
    contactTime?: string;
    pitchFormat?: 'brief_summary' | 'full_release' | 'exclusive_angle';
    noAttachments?: boolean;
  };
  languages?: string[];
  specializations?: string[];
}

interface JournalistCompliance {
  consentStatus: 'implied' | 'explicit' | 'opted_out';
  consentBasis?: 'legitimate_interest' | 'explicit_consent';
  dataSource: 'manual' | 'api_import' | 'public_profile' | 'web_scrape';
  dataSourceUrl?: string;
  optOutDate?: string | null;
  gdprJurisdiction?: boolean;
  caslJurisdiction?: boolean;
  suppressedOrgs?: string[];
  consentHistory?: Array<{
    action: string;
    date: string;
    source: string;
    basis?: string;
  }>;
}
```

API routes:
```typescript
// GET    /api/journalists              -> list with pagination, filtering, search
// GET    /api/journalists/:id          -> single journalist with outlets
// POST   /api/journalists              -> create journalist
// PATCH  /api/journalists/:id          -> update journalist
// DELETE /api/journalists/:id          -> soft-delete (mark opted_out)
// POST   /api/journalists/:id/outlets  -> link journalist to outlet
// DELETE /api/journalists/:id/outlets/:outletId -> unlink

// GET    /api/outlets                  -> list with filtering
// GET    /api/outlets/:id              -> single outlet with journalists
// POST   /api/outlets                  -> create outlet
// PATCH  /api/outlets/:id              -> update outlet

// Query parameters for GET /api/journalists:
interface JournalistQuery {
  q?: string;              // full-text search on name, email, bio
  beats?: string[];        // filter by beats (array overlap)
  countryCode?: string;    // ISO 3166-1
  outletType?: string;     // newspaper, blog, podcast, etc.
  consentStatus?: string;  // implied, explicit, opted_out
  isVerified?: boolean;
  page?: number;           // default 1
  limit?: number;          // default 25, max 100
  sort?: 'name' | 'updated' | 'created';
  order?: 'asc' | 'desc';
}
```

**Testing**:
- `Unit: JournalistProfile Zod schema -> validates correct profile`
- `Unit: JournalistProfile Zod schema -> rejects invalid contactMethod`
- `Unit: JournalistCompliance Zod schema -> validates consent history array`
- `Integration: POST /api/journalists -> creates journalist with beats array and compliance JSONB`
- `Integration: GET /api/journalists?beats=technology&countryCode=US -> returns filtered results`
- `Integration: PATCH /api/journalists/:id compliance.consentStatus=opted_out -> adds consent history entry`
- `Integration: DELETE /api/journalists/:id -> marks as opted_out, not hard delete`
- `Integration: POST /api/journalists/:id/outlets -> creates journalist_outlet record`
- `Integration: GET /api/journalists/:id -> includes current and past outlet relationships`

#### 2.2 — Journalist Search Integration

**What**: Index journalists and outlets in Meilisearch for typo-tolerant full-text search with faceted filtering by beat, country, outlet type, and verification status.

**Design**:

Meilisearch index configuration:
```typescript
// apps/api/src/services/search.service.ts
interface JournalistSearchDocument {
  id: string;
  fullName: string;
  email: string | null;
  title: string | null;
  beats: string[];
  countryCode: string | null;
  city: string | null;
  bio: string | null;
  outlets: Array<{ name: string; type: string; role: string }>;
  isVerified: boolean;
  consentStatus: string;
  updatedAt: string;
}

const JOURNALIST_INDEX_SETTINGS = {
  searchableAttributes: ['fullName', 'email', 'title', 'bio', 'outlets.name', 'beats'],
  filterableAttributes: ['beats', 'countryCode', 'outlets.type', 'isVerified', 'consentStatus'],
  sortableAttributes: ['fullName', 'updatedAt'],
  distinctAttribute: 'id',
};
```

Sync service for keeping Meilisearch in sync:
```typescript
export class SearchSyncService {
  async indexJournalist(journalist: Journalist): Promise<void>;
  async removeJournalist(id: string): Promise<void>;
  async reindexAll(): Promise<void>;
  async search(query: string, filters: JournalistSearchFilters): Promise<SearchResult<JournalistSearchDocument>>;
}
```

**Testing**:
- `Unit: JournalistSearchDocument mapper -> transforms Prisma journalist to search document`
- `Integration: indexJournalist -> document appears in Meilisearch within 500ms`
- `Integration: search("tech reporter") -> returns journalist with "Technology Reporter" title`
- `Integration: search with beat filter -> returns only journalists with matching beats`
- `Integration: search with typo "journlist" -> returns journalist results (typo tolerance)`
- `Integration: reindexAll with 1000 journalists -> completes within 10s`

#### 2.3 — Media List Management

**What**: CRUD for static and dynamic media lists. Static lists have manually added journalists. Dynamic lists use saved filter criteria that re-evaluate at query time.

**Design**:

```typescript
// packages/shared/src/types/media-list.ts
interface MediaListFilterCriteria {
  beats?: string[];
  countries?: string[];
  outletTypes?: string[];
  minDomainAuthority?: number;
  excludeOptedOut: boolean;
  isVerified?: boolean;
}

// GET    /api/media-lists              -> list all for org
// GET    /api/media-lists/:id          -> list with members (evaluated for dynamic)
// POST   /api/media-lists              -> create (static or dynamic)
// PATCH  /api/media-lists/:id          -> update name, description, or filter criteria
// DELETE /api/media-lists/:id          -> delete list
// POST   /api/media-lists/:id/members  -> add journalist(s) to static list
// DELETE /api/media-lists/:id/members/:journalistId -> remove journalist from static list
```

Dynamic list evaluation:
```typescript
export class MediaListService {
  async evaluateDynamicList(list: MediaList): Promise<Journalist[]> {
    const criteria = list.filterCriteria as MediaListFilterCriteria;
    const where: Prisma.JournalistWhereInput = {};
    if (criteria.beats?.length) where.beats = { hasSome: criteria.beats };
    if (criteria.countries?.length) where.countryCode = { in: criteria.countries };
    if (criteria.excludeOptedOut) {
      where.compliance = { path: ['consentStatus'], not: 'opted_out' };
    }
    // ... additional criteria
    return prisma.journalist.findMany({ where });
  }
}
```

**Testing**:
- `Unit: evaluateDynamicList with beats=["technology"] -> returns only tech journalists`
- `Unit: evaluateDynamicList with excludeOptedOut=true -> excludes opted-out journalists`
- `Integration: POST /api/media-lists (static) -> creates list with 0 members`
- `Integration: POST /api/media-lists/:id/members -> adds journalist, increments journalist_count`
- `Integration: GET /api/media-lists/:id (dynamic) -> evaluates filter and returns matching journalists`
- `Integration: PATCH dynamic list filter criteria -> GET returns updated results`
- `Integration: journalist opts out -> excluded from dynamic list results`

#### 2.4 — GDPR Compliance & Suppression

**What**: Implement GDPR Article 6(1)(f) legitimate interest tracking, opt-out handling, data subject access requests (DSAR), and CAN-SPAM/CASL suppression lists.

**Design**:

```typescript
// apps/api/src/services/compliance.service.ts
export class ComplianceService {
  // Record consent event in journalist's compliance JSONB
  async recordConsentEvent(journalistId: string, event: {
    action: 'imported' | 'pitched' | 'opted_out' | 'consent_granted' | 'data_exported' | 'data_deleted';
    basis?: 'legitimate_interest' | 'explicit_consent';
    source: string;
  }): Promise<void>;

  // Handle opt-out: update journalist, add to org suppression
  async processOptOut(organizationId: string, email: string): Promise<void>;

  // DSAR: export all data held for a journalist
  async exportJournalistData(journalistId: string): Promise<JournalistDataExport>;

  // Right to erasure: anonymize journalist data
  async processErasureRequest(journalistId: string): Promise<void>;

  // Check if journalist can be contacted by this org
  async canContact(organizationId: string, journalistId: string): Promise<boolean>;
}
```

Opt-out endpoint (unsubscribe link in emails):
```typescript
// GET /api/unsubscribe/:token -> renders confirmation page
// POST /api/unsubscribe/:token -> processes opt-out
```

**Testing**:
- `Unit: canContact -> returns false if journalist consent_status is opted_out`
- `Unit: canContact -> returns false if org is in journalist's suppressedOrgs`
- `Unit: processOptOut -> updates consent_status, adds consent_history entry`
- `Integration: POST /api/unsubscribe/:token -> journalist marked as opted_out`
- `Integration: exportJournalistData -> returns all pitches, coverage, scores for journalist`
- `Integration: processErasureRequest -> journalist PII replaced with "[redacted]"`
- `Integration: after opt-out, GET /api/media-lists/:id excludes journalist`

---

## Phase 3: Press Release Authoring & Newsroom

### Purpose
Build the press release content creation system with rich text editing, version history, multimedia attachments, embargo management, and branded newsroom publishing. After this phase, teams can create, edit, approve, and publish press releases to a branded public newsroom with SEO-optimised Schema.org markup.

### Tasks

#### 3.1 — Press Release CRUD & Versioning

**What**: API for creating, editing, and managing press releases with full version history and IPTC/Schema.org metadata.

**Design**:

```typescript
// packages/shared/src/types/press-release.ts
interface PressReleaseContentMeta {
  summary?: string;
  seoKeywords?: string[];
  iptcSubjectCodes?: string[];  // IPTC NewsCodes references
  schemaOrg?: {
    '@type': 'NewsArticle';
    datePublished?: string;
    publisher?: { '@type': 'Organization'; name: string };
  };
  contacts?: Array<{
    name: string;
    title: string;
    email: string;
    phone?: string;
  }>;
  quotes?: Array<{
    speaker: string;
    title: string;
    text: string;
  }>;
  multimedia?: Array<{
    type: 'image' | 'video' | 'document' | 'infographic';
    url: string;
    altText?: string;
    caption?: string;
    mimeType: string;
    sizeBytes: number;
  }>;
  translations?: Record<string, {
    headline: string;
    bodyHtml: string;
  }>;
  styleGuideApplied?: string;
  aiQualityScore?: number;
  readabilityScore?: number;
}

interface PressReleaseApproval {
  approvedBy?: string;
  approvedAt?: string;
  approvalNotes?: string;
  legalReview?: {
    reviewedBy: string;
    reviewedAt: string;
    status: 'pending' | 'cleared' | 'requires_changes';
  };
}

type PressReleaseStatus = 'draft' | 'review' | 'approved' | 'published' | 'archived';
```

State machine:
```
draft -> review -> approved -> published -> archived
  ^        |                      |
  +--------+  (rejected)          +-> archived
```

API routes:
```typescript
// GET    /api/press-releases               -> list for org
// GET    /api/press-releases/:id           -> single with versions
// POST   /api/press-releases               -> create draft
// PATCH  /api/press-releases/:id           -> update (creates version)
// POST   /api/press-releases/:id/submit    -> draft -> review
// POST   /api/press-releases/:id/approve   -> review -> approved
// POST   /api/press-releases/:id/reject    -> review -> draft
// POST   /api/press-releases/:id/publish   -> approved -> published
// POST   /api/press-releases/:id/archive   -> any -> archived
// GET    /api/press-releases/:id/versions  -> version history
// GET    /api/press-releases/:id/versions/:version -> specific version
```

**Testing**:
- `Unit: state machine draft->review->approved->published -> valid transitions`
- `Unit: state machine draft->published -> invalid, throws StateTransitionError`
- `Unit: version creation -> increments version_number, stores previous content`
- `Integration: POST /api/press-releases -> creates draft with version_count=1`
- `Integration: PATCH /api/press-releases/:id -> creates version record, updates current`
- `Integration: POST /api/press-releases/:id/approve -> sets approval JSONB, changes status`
- `Integration: embargo_until in future -> published_at not set until embargo lifts`
- `Integration: GET /api/press-releases/:id/versions -> returns all versions in order`
- `Fixture: press release with IPTC subject codes -> stored and queryable via content_meta`

#### 3.2 — File Attachment Management

**What**: Upload and manage press release attachments (images, videos, documents) stored in S3-compatible storage.

**Design**:

```typescript
// apps/api/src/services/storage.service.ts
export class StorageService {
  async uploadFile(params: {
    file: Buffer;
    fileName: string;
    mimeType: string;
    pressReleaseId: string;
    organizationId: string;
  }): Promise<{
    url: string;
    storageKey: string;
    sizeBytes: number;
  }>;

  async deleteFile(storageKey: string): Promise<void>;
  async getSignedUrl(storageKey: string, expiresInSec?: number): Promise<string>;
}

// POST /api/press-releases/:id/attachments -> multipart upload
// DELETE /api/press-releases/:id/attachments/:attachmentId
```

File validation:
```typescript
const ALLOWED_MIME_TYPES = [
  'image/jpeg', 'image/png', 'image/webp', 'image/gif', 'image/svg+xml',
  'video/mp4', 'video/webm',
  'application/pdf',
  'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
];
const MAX_FILE_SIZE = 50 * 1024 * 1024; // 50MB
```

**Testing**:
- `Unit: validateFile with 10MB PNG -> passes`
- `Unit: validateFile with 100MB file -> throws FileTooLargeError`
- `Unit: validateFile with .exe -> throws InvalidMimeTypeError`
- `Integration: POST multipart upload -> file stored in S3, metadata in content_meta.multimedia`
- `Integration: DELETE attachment -> file removed from S3, removed from content_meta`
- `Integration: getSignedUrl -> returns valid presigned URL`

#### 3.3 — Branded Newsroom

**What**: Public-facing newsroom pages that display published press releases with SEO-optimised Schema.org NewsArticle markup, custom branding, and social feed integration.

**Design**:

Newsroom configuration:
```typescript
interface NewsroomConfig {
  branding: {
    logoUrl?: string;
    colors: { primary: string; secondary?: string; background?: string };
    favicon?: string;
  };
  seo: {
    title: string;
    description: string;
    ogImage?: string;
  };
  socialFeeds?: Array<{
    platform: 'twitter' | 'linkedin' | 'mastodon';
    handle: string;
  }>;
  layout: 'grid' | 'list' | 'magazine';
  categories?: string[];
  contactInfo?: {
    email: string;
    phone?: string;
  };
}
```

Public API (no auth required):
```typescript
// GET /newsroom/:slug                   -> newsroom landing page data
// GET /newsroom/:slug/releases          -> paginated published releases
// GET /newsroom/:slug/releases/:releaseId -> single release with Schema.org JSON-LD
// GET /newsroom/:slug/feed.xml          -> RSS 2.0 feed (RFC 4287 / RSS 2.0 compliant)
// GET /newsroom/:slug/feed.json         -> JSON Feed
```

Schema.org JSON-LD output (per RFC/Schema.org NewsArticle standard):
```json
{
  "@context": "https://schema.org",
  "@type": "NewsArticle",
  "headline": "Acme Corp Launches New Product",
  "datePublished": "2026-06-01T09:00:00Z",
  "dateModified": "2026-06-01T09:00:00Z",
  "author": { "@type": "Organization", "name": "Acme Corp" },
  "publisher": {
    "@type": "Organization",
    "name": "Acme Corp",
    "logo": { "@type": "ImageObject", "url": "https://..." }
  },
  "description": "...",
  "image": "https://...",
  "articleBody": "..."
}
```

RSS 2.0 feed generation:
```typescript
export class FeedService {
  async generateRssFeed(newsroom: Newsroom, releases: PressRelease[]): Promise<string>;
  async generateJsonFeed(newsroom: Newsroom, releases: PressRelease[]): Promise<object>;
}
```

**Testing**:
- `Unit: generateRssFeed -> valid RSS 2.0 XML with channel and item elements`
- `Unit: Schema.org JSON-LD output -> valid against Schema.org NewsArticle spec`
- `Integration: GET /newsroom/:slug -> returns newsroom data with branding config`
- `Integration: GET /newsroom/:slug/releases -> returns only published releases`
- `Integration: GET /newsroom/:slug/releases -> excludes embargoed releases (embargo_until > now)`
- `Integration: GET /newsroom/:slug/feed.xml -> valid RSS 2.0 parseable by feed reader`
- `Integration: newsroom with is_published=false -> 404`
- `E2E: published press release appears on newsroom page within 5s`

---

## Phase 4: Campaign & Pitch Management

### Purpose
Build the campaign creation and email pitching system with personalisation, follow-up sequences, email delivery tracking (opens, clicks, replies), and DKIM/SPF compliance. After this phase, PR teams can create campaigns linked to press releases, send personalised pitches to media lists, schedule follow-ups, and track engagement.

### Tasks

#### 4.1 — Campaign CRUD & Lifecycle

**What**: Create and manage campaigns with configurable send schedules, follow-up rules, and media list targeting.

**Design**:

```typescript
// packages/shared/src/types/campaign.ts
interface CampaignConfig {
  sendSchedule: {
    type: 'immediate' | 'scheduled' | 'ai_optimized';
    scheduledAt?: string;  // ISO 8601
  };
  followupRules: Array<{
    delayHours: number;
    subjectTemplate: string;
    bodyTemplate: string;
    onlyIf: 'no_open' | 'no_reply' | 'no_click';
  }>;
  personalization: {
    enabled: boolean;
    model?: 'template' | 'outlet_style_match';
  };
  mediaListIds: string[];
  abTest?: {
    enabled: boolean;
    variants: Array<{
      subjectLine: string;
      percentage: number;
    }>;
  };
}

interface CampaignStats {
  totalPitches: number;
  delivered: number;
  bounced: number;
  opened: number;
  uniqueOpens: number;
  clicked: number;
  replied: number;
  coverageCount: number;
  totalReach: number;
  avgSentiment: number | null;
}

type CampaignStatus = 'draft' | 'active' | 'paused' | 'completed';
```

API routes:
```typescript
// GET    /api/campaigns               -> list for org
// GET    /api/campaigns/:id           -> campaign with stats
// POST   /api/campaigns               -> create campaign
// PATCH  /api/campaigns/:id           -> update config
// POST   /api/campaigns/:id/launch    -> draft -> active, enqueue pitches
// POST   /api/campaigns/:id/pause     -> active -> paused
// POST   /api/campaigns/:id/resume    -> paused -> active
// POST   /api/campaigns/:id/complete  -> active -> completed
```

**Testing**:
- `Unit: campaign state machine draft->active->paused->active->completed -> valid`
- `Unit: campaign launch -> enqueues pitch jobs for all journalists in media lists`
- `Unit: campaign launch with excludeOptedOut -> skips opted-out journalists`
- `Integration: POST /api/campaigns with mediaListIds -> creates campaign`
- `Integration: POST /api/campaigns/:id/launch -> creates pitch records for all list members`
- `Integration: campaign stats updated after pitch events -> accurate counts`
- `Integration: campaign with scheduled send -> pitches not sent before scheduledAt`

#### 4.2 — Pitch Sending & Email Delivery

**What**: Send personalised email pitches via Resend API with DKIM/SPF compliance, delivery tracking, and rate limiting.

**Design**:

```typescript
// apps/worker/src/processors/send-pitch.processor.ts
interface SendPitchJob {
  pitchId: string;
  campaignId: string;
  journalistId: string;
  senderId: string;
}

export class SendPitchProcessor {
  async process(job: Job<SendPitchJob>): Promise<void> {
    // 1. Load pitch, journalist, sender
    // 2. Check compliance (canContact)
    // 3. Check suppression list
    // 4. Render email template with personalization
    // 5. Send via Resend API
    // 6. Update pitch.delivery JSONB with message_id, dkim_status, sent_at
    // 7. Schedule follow-up jobs based on campaign.config.followupRules
  }
}
```

Email template rendering:
```typescript
interface PitchEmail {
  from: { name: string; email: string };
  to: string;
  subject: string;
  html: string;
  text: string;
  headers: {
    'List-Unsubscribe': string;       // CAN-SPAM / CASL compliance
    'List-Unsubscribe-Post': string;  // RFC 8058 one-click unsubscribe
    'X-Campaign-Id': string;
  };
  tags: Array<{ name: string; value: string }>;
}
```

Rate limiting configuration:
```typescript
const EMAIL_RATE_LIMITS = {
  perSecond: 10,       // Resend API limit
  perMinute: 100,
  perHour: 1000,
  perDay: 5000,        // adjust per org plan tier
};
```

**Testing**:
- `Unit: renderPitchEmail -> includes List-Unsubscribe header with correct token`
- `Unit: renderPitchEmail -> HTML and text bodies contain journalist name`
- `Unit: compliance check opted_out journalist -> pitch skipped, status set to "skipped"`
- `Integration (mocked Resend): sendPitch -> pitch.delivery updated with message_id`
- `Integration (mocked Resend): Resend 429 -> job retried with exponential backoff`
- `Integration (mocked Resend): Resend hard bounce -> pitch status set to "bounced"`
- `Integration: rate limiter -> jobs queue when rate limit reached, resume after window`
- `Integration: follow-up scheduled -> job created with correct delay`

#### 4.3 — Email Engagement Tracking

**What**: Process webhook events from Resend for opens, clicks, and replies. Update pitch engagement JSONB and campaign stats.

**Design**:

Webhook handler:
```typescript
// POST /api/webhooks/resend -> receives Resend webhook events
interface ResendWebhookPayload {
  type: 'email.sent' | 'email.delivered' | 'email.opened'
    | 'email.clicked' | 'email.bounced' | 'email.complained';
  data: {
    email_id: string;
    to: string;
    created_at: string;
    click?: { url: string };
  };
}
```

Webhook signature verification:
```typescript
export function verifyResendWebhook(
  payload: string,
  signature: string,
  webhookSecret: string
): boolean;
```

Engagement update logic:
```typescript
// apps/api/src/services/engagement.service.ts
export class EngagementService {
  async processEvent(event: ResendWebhookPayload): Promise<void> {
    // 1. Find pitch by message_id
    // 2. Append event to pitch.engagement JSONB
    // 3. Update aggregate counts (open_count, click_count, has_replied)
    // 4. Update campaign.stats incrementally
    // 5. Detect bot opens (filter by user agent patterns)
    // 6. Cancel scheduled follow-ups if reply received
  }
}
```

Bot detection:
```typescript
const BOT_USER_AGENTS = [
  /barracuda/i, /googlebot/i, /bingbot/i,
  /yahoo.*slurp/i, /protection\.outlook/i,
];
```

**Testing**:
- `Unit: verifyResendWebhook with valid signature -> returns true`
- `Unit: verifyResendWebhook with invalid signature -> returns false`
- `Unit: processEvent open -> appends to engagement.opens array, increments open_count`
- `Unit: processEvent with bot user agent -> is_bot=true, not counted in unique_opens`
- `Unit: processEvent reply -> sets has_replied=true, cancels follow-up jobs`
- `Integration: POST /api/webhooks/resend (open event) -> pitch engagement updated`
- `Integration: POST /api/webhooks/resend (invalid signature) -> 401`
- `Integration: multiple opens -> open_count incremented, unique detection correct`
- `Integration: reply received -> scheduled follow-up jobs cancelled in BullMQ`

#### 4.4 — Follow-Up Sequences

**What**: Automated follow-up emails based on campaign rules (send follow-up if no open/reply after N hours).

**Design**:

```typescript
// apps/worker/src/processors/send-followup.processor.ts
interface FollowUpJob {
  pitchId: string;
  followupNumber: number;
  condition: 'no_open' | 'no_reply' | 'no_click';
  subjectTemplate: string;
  bodyTemplate: string;
}

export class SendFollowupProcessor {
  async process(job: Job<FollowUpJob>): Promise<void> {
    // 1. Load pitch and check condition (has it been opened/replied?)
    // 2. If condition still met -> send follow-up
    // 3. If condition no longer met (they opened/replied) -> skip
    // 4. Update pitch record with follow-up sent status
    // 5. Schedule next follow-up if configured
  }
}
```

**Testing**:
- `Unit: follow-up with condition no_open, pitch has 0 opens -> sends follow-up`
- `Unit: follow-up with condition no_open, pitch has 1 open -> skips`
- `Unit: follow-up with condition no_reply, pitch has reply -> skips`
- `Integration: campaign launched -> follow-up jobs scheduled at correct delays`
- `Integration: follow-up sent -> email appears in pitch engagement history`
- `Integration: 3-followup sequence -> all 3 sent if no response`

---

## Phase 5: Coverage Monitoring & Tracking

### Purpose
Build real-time media coverage monitoring that discovers press mentions, attributes coverage to campaigns, and performs sentiment analysis. After this phase, PR teams can set up monitoring queries, discover coverage automatically, attribute it to campaigns, and see sentiment/reach analytics.

### Tasks

#### 5.1 — Monitoring Query Management

**What**: CRUD for monitoring queries (boolean search queries) that poll news sources for coverage mentions.

**Design**:

```typescript
// packages/shared/src/types/monitoring.ts
interface MonitoringQueryConfig {
  queryString: string;        // Boolean search: "Acme Corp" OR acmecorp
  sources: Array<'news' | 'social' | 'podcast' | 'broadcast'>;
  languages?: string[];       // ISO 639-1
  excludeDomains?: string[];
  alerts: {
    enabled: boolean;
    channels: Array<'email' | 'slack' | 'webhook'>;
    frequency: 'real_time' | 'hourly' | 'daily';
    sentimentThreshold?: number;  // alert if sentiment below this
  };
}

// GET    /api/monitoring             -> list active queries
// POST   /api/monitoring             -> create query
// PATCH  /api/monitoring/:id         -> update query
// DELETE /api/monitoring/:id         -> deactivate query
// POST   /api/monitoring/:id/test    -> run query once, return sample results
```

**Testing**:
- `Unit: MonitoringQueryConfig validation -> accepts valid boolean query`
- `Unit: MonitoringQueryConfig validation -> rejects empty queryString`
- `Integration: POST /api/monitoring -> creates query, starts polling job`
- `Integration: DELETE /api/monitoring/:id -> stops polling job`
- `Integration: POST /api/monitoring/:id/test -> returns sample results without persisting`

#### 5.2 — Coverage Discovery & Ingestion

**What**: Background worker that polls news APIs and newswire RSS feeds (PR Newswire, Business Wire RSS per RFC 4287/RSS 2.0) to discover coverage matching monitoring queries.

**Design**:

```typescript
// apps/worker/src/processors/monitor-coverage.processor.ts
interface MonitorCoverageJob {
  monitoringQueryId: string;
  organizationId: string;
}

export class MonitorCoverageProcessor {
  async process(job: Job<MonitorCoverageJob>): Promise<void> {
    // 1. Load monitoring query config
    // 2. Poll configured sources:
    //    - News API: search by query string
    //    - RSS feeds: parse PR Newswire, Business Wire, GlobeNewswire feeds
    //    - Social API: search Twitter/X, LinkedIn
    // 3. Deduplicate against existing coverage (by URL)
    // 4. For each new article:
    //    a. Extract metadata (headline, URL, published_at, outlet)
    //    b. Match to known outlet in database
    //    c. Match to known journalist if byline present
    //    d. Enqueue sentiment analysis job
    //    e. Insert coverage record
    // 5. Send alerts if configured
  }
}
```

RSS feed parser:
```typescript
export class RssFeedParser {
  async parseFeed(feedUrl: string): Promise<Array<{
    title: string;
    link: string;
    publishedAt: Date;
    description: string;
    author?: string;
  }>>;
}
```

Polling schedule:
```typescript
const POLL_INTERVALS = {
  real_time: '5 minutes',
  hourly: '1 hour',
  daily: '24 hours',
};
```

**Testing**:
- `Unit: RssFeedParser with valid RSS 2.0 -> extracts title, link, publishedAt`
- `Unit: RssFeedParser with Atom feed -> extracts equivalent fields`
- `Unit: deduplication by URL -> skips already-ingested articles`
- `Integration (mocked news API): monitor-coverage job -> discovers 5 articles, inserts 5 coverage records`
- `Integration: RSS feed poll -> parses PR Newswire feed, creates coverage records`
- `Integration: alert enabled with real_time frequency -> Slack notification sent on new coverage`
- `Integration: duplicate URL -> not inserted again`

#### 5.3 — Coverage Attribution

**What**: Automatically correlate discovered coverage to campaigns and pitches using journalist matching and timing heuristics.

**Design**:

```typescript
// apps/api/src/services/attribution.service.ts
export class AttributionService {
  async attributeCoverage(coverageId: string): Promise<{
    campaignId: string | null;
    pitchId: string | null;
    confidence: number;
  }> {
    // Attribution heuristic:
    // 1. Match by journalist_id: if journalist was pitched in a campaign within 14 days
    // 2. Match by outlet: if outlet was targeted and article appeared within 14 days
    // 3. Match by content similarity: compare coverage headline/snippet to press release headline
    // 4. Score confidence based on match quality (journalist > outlet > content)
    // 5. Store attribution in coverage.analysis JSONB
  }
}
```

**Testing**:
- `Unit: journalist pitched 3 days ago, coverage appears -> attributed with high confidence`
- `Unit: journalist pitched 30 days ago -> not attributed (outside window)`
- `Unit: outlet match but different journalist -> lower confidence`
- `Integration: coverage discovered from pitched journalist -> auto-attributed to campaign`
- `Integration: attribution updates campaign.stats.coverageCount`

#### 5.4 — Sentiment Analysis Integration

**What**: Analyse coverage sentiment using the Python AI microservice. Store sentiment_score (-1.0 to 1.0) and sentiment_label in coverage.analysis JSONB.

**Design**:

Python sentiment endpoint:
```python
# services/ai/src/routers/sentiment.py
from fastapi import APIRouter
from pydantic import BaseModel

class SentimentRequest(BaseModel):
    text: str
    context: str | None = None  # e.g., company name for entity-level sentiment

class SentimentResponse(BaseModel):
    score: float       # -1.0 to 1.0
    label: str         # positive, neutral, negative, mixed
    confidence: float  # 0.0 to 1.0
    entities: list[dict] | None = None  # entity-level sentiment

router = APIRouter(prefix="/sentiment")

@router.post("/analyze")
async def analyze_sentiment(req: SentimentRequest) -> SentimentResponse:
    # Use transformer model (distilbert-base-uncased-finetuned-sst-2-english)
    # or Claude API for nuanced entity-level sentiment
    ...
```

TypeScript client:
```typescript
// apps/api/src/services/ai.service.ts
export class AiService {
  async analyzeSentiment(text: string, context?: string): Promise<{
    score: number;
    label: string;
    confidence: number;
  }>;
}
```

**Testing**:
- `Unit: positive article text -> sentiment_score > 0.3, label=positive`
- `Unit: negative article text -> sentiment_score < -0.3, label=negative`
- `Unit: mixed sentiment -> label=mixed`
- `Integration: coverage ingested -> sentiment analysis job enqueued`
- `Integration: sentiment result -> stored in coverage.analysis.sentiment_score`
- `Fixture: 10 sample articles with known sentiment -> accuracy > 80%`

---

## Phase 6: Analytics Dashboard

### Purpose
Build comprehensive analytics dashboards showing campaign performance, coverage metrics, share-of-voice, and reach/sentiment trends. After this phase, PR teams have stakeholder-ready dashboards and exportable reports.

### Tasks

#### 6.1 — Campaign Analytics API

**What**: API endpoints returning aggregated campaign performance metrics: pitch funnel, engagement rates, coverage attribution, and reach.

**Design**:

```typescript
// GET /api/analytics/campaigns/:id
interface CampaignAnalyticsResponse {
  campaign: {
    id: string;
    name: string;
    status: string;
    startedAt: string;
  };
  funnel: {
    totalPitches: number;
    delivered: number;
    opened: number;
    clicked: number;
    replied: number;
    coverage: number;
  };
  rates: {
    deliveryRate: number;   // delivered / totalPitches
    openRate: number;       // opened / delivered
    clickRate: number;      // clicked / delivered
    replyRate: number;      // replied / delivered
    coverageRate: number;   // coverage / delivered
  };
  timeline: Array<{
    date: string;
    opens: number;
    clicks: number;
    replies: number;
    coverage: number;
  }>;
  topJournalists: Array<{
    journalistId: string;
    name: string;
    outlet: string;
    engaged: boolean;
    covered: boolean;
  }>;
  reach: {
    totalEstimated: number;
    byOutlet: Array<{ outletName: string; reach: number }>;
  };
}

// GET /api/analytics/overview -> org-wide dashboard
interface OverviewAnalyticsResponse {
  period: { from: string; to: string };
  totalCampaigns: number;
  totalPitchesSent: number;
  avgOpenRate: number;
  avgReplyRate: number;
  totalCoverage: number;
  totalReach: number;
  sentimentBreakdown: {
    positive: number;
    neutral: number;
    negative: number;
    mixed: number;
  };
  topCampaigns: Array<{ id: string; name: string; coverageCount: number }>;
  trendline: Array<{ month: string; coverage: number; reach: number }>;
}
```

**Testing**:
- `Unit: rate calculation with 100 pitches, 80 delivered, 40 opened -> openRate=0.50`
- `Unit: rate calculation with 0 delivered -> all rates return 0 (no division by zero)`
- `Integration: GET /api/analytics/campaigns/:id -> returns full analytics`
- `Integration: campaign with 5 attributed coverage items -> coverage=5 in funnel`
- `Integration: timeline aggregation -> grouped by date correctly`

#### 6.2 — Share-of-Voice Metrics

**What**: Calculate and display share-of-voice relative to configured competitors based on coverage volume, reach, and sentiment.

**Design**:

```typescript
// GET /api/analytics/share-of-voice?competitors=uuid1,uuid2&period=30d
interface ShareOfVoiceResponse {
  period: { from: string; to: string };
  entities: Array<{
    name: string;
    type: 'self' | 'competitor';
    coverageCount: number;
    totalReach: number;
    avgSentiment: number;
    sharePercent: number;  // percentage of total coverage volume
  }>;
  timeline: Array<{
    date: string;
    shares: Record<string, number>;  // entity_name -> coverage count
  }>;
}
```

Competitor tracking model:
```typescript
// Competitors tracked via monitoring queries with a "competitor" tag
// Coverage tagged with competitor mentions stored in coverage.analysis.competitorMentions
```

**Testing**:
- `Unit: share calculation 10 own / 30 total -> 33.3%`
- `Integration: 3 competitors configured, coverage distributed -> accurate share percentages`
- `Integration: timeline over 30 days -> correct daily share values`

#### 6.3 — Report Export

**What**: Export analytics reports as PDF and CSV for stakeholder distribution.

**Design**:

```typescript
// POST /api/analytics/campaigns/:id/export
interface ExportRequest {
  format: 'pdf' | 'csv' | 'xlsx';
  sections?: Array<'funnel' | 'timeline' | 'topJournalists' | 'coverage' | 'reach'>;
}
// Returns: { downloadUrl: string; expiresAt: string }
```

**Testing**:
- `Integration: export campaign as CSV -> valid CSV with correct columns`
- `Integration: export campaign as PDF -> valid PDF file returned`
- `Integration: download URL expires after 1 hour`

---

## Phase 7: Wire Distribution Integration

### Purpose
Integrate with external wire distribution services (PR Newswire, Business Wire, GlobeNewswire) for press release distribution and optional SEC EDGAR regulatory filing. After this phase, teams can distribute press releases through wire services and track distribution status.

### Tasks

#### 7.1 — Distribution Order Management

**What**: Create and manage wire distribution orders linked to press releases with scope targeting and status tracking.

**Design**:

```typescript
// packages/shared/src/types/distribution.ts
interface DistributionDetails {
  distributionScope: string[];    // ISO 3166-1 country codes
  submittedAt?: string;
  distributedAt?: string;
  wireReference?: string;         // external ref from wire service
  costCents?: number;
  regulatory?: {
    isFiling: boolean;
    filingType?: '8-K' | '6-K' | 'material_change';
    edgarAccession?: string;       // EDGAR accession number
    sedarReference?: string;
  };
  syndicationUrls?: string[];
}

type DistributionStatus = 'pending' | 'submitted' | 'distributed' | 'failed' | 'cancelled';

// POST   /api/distribution/orders           -> create order
// GET    /api/distribution/orders            -> list orders for org
// GET    /api/distribution/orders/:id        -> order status
// POST   /api/distribution/orders/:id/submit -> submit to wire service
// POST   /api/distribution/orders/:id/cancel -> cancel pending order
```

**Testing**:
- `Unit: create order -> status=pending, no wireReference`
- `Unit: submit order -> calls wire service API, status=submitted`
- `Integration: POST /api/distribution/orders -> creates with press_release link`
- `Integration: submit order with regulatory.isFiling=true -> includes EDGAR metadata`
- `Integration: order status polled -> updated to distributed when confirmed`

#### 7.2 — Wire Service Adapters

**What**: Pluggable adapter pattern for different wire services (PR Newswire, Business Wire, GlobeNewswire) with a common interface.

**Design**:

```typescript
// apps/api/src/services/distribution/wire-adapter.interface.ts
interface WireServiceAdapter {
  readonly serviceName: string;
  submit(params: {
    pressRelease: PressRelease;
    scope: string[];
    regulatory?: RegulatoryFiling;
  }): Promise<{ wireReference: string; estimatedDistributionAt: Date }>;
  checkStatus(wireReference: string): Promise<DistributionStatus>;
  cancel(wireReference: string): Promise<void>;
}

// Implementations:
// apps/api/src/services/distribution/pr-newswire.adapter.ts
// apps/api/src/services/distribution/business-wire.adapter.ts
// apps/api/src/services/distribution/globenewswire.adapter.ts
```

**Testing**:
- `Unit (mocked API): PRNewswireAdapter.submit -> returns wireReference`
- `Unit (mocked API): BusinessWireAdapter.submit with EDGAR filing -> includes filing metadata`
- `Unit (mocked API): adapter.checkStatus -> returns current DistributionStatus`
- `Integration (mocked): submit + poll status -> status transitions pending->submitted->distributed`
- `Unit: unknown wire service -> WireServiceNotFoundError`

---

## Phase 8: AI-Powered Features

### Purpose
Implement the AI-native differentiating features: journalist targeting based on beat/coverage analysis, relationship health scoring, predictive send-time optimisation, placement prediction, and AI-assisted press release drafting. This phase builds the features that no incumbent platform offers.

### Tasks

#### 8.1 — AI Journalist Targeting

**What**: ML-based journalist matching that recommends the best journalists for a press release based on beat history, past coverage analysis, and content similarity.

**Design**:

Python endpoint:
```python
# services/ai/src/routers/journalist_match.py
class JournalistMatchRequest(BaseModel):
    press_release_headline: str
    press_release_summary: str
    press_release_beats: list[str]
    candidate_journalist_ids: list[str] | None = None
    top_k: int = 50

class JournalistMatchResult(BaseModel):
    journalist_id: str
    relevance_score: float   # 0.0 to 1.0
    match_reasons: list[str] # ["beat_overlap: technology, ai", "covered_similar: 3 articles"]

class JournalistMatchResponse(BaseModel):
    results: list[JournalistMatchResult]
    model_version: str

@router.post("/match")
async def match_journalists(req: JournalistMatchRequest) -> JournalistMatchResponse:
    # 1. Encode press release with sentence-transformers
    # 2. Retrieve journalist beat embeddings from vector index
    # 3. Compute cosine similarity for beat relevance
    # 4. Boost score for journalists who covered similar topics recently
    # 5. Penalize journalists who are opted out or recently pitched
    # 6. Return ranked results with explanations
    ...
```

TypeScript integration:
```typescript
// GET /api/ai/journalist-match?pressReleaseId=uuid&topK=50
// Returns: Array<{ journalistId, name, relevanceScore, matchReasons }>
```

**Testing**:
- `Unit: press release about "AI product launch" -> technology/AI beat journalists ranked highest`
- `Unit: journalist with recent similar coverage -> boosted score`
- `Unit: opted-out journalist -> excluded from results`
- `Integration: POST /match with 1000 candidates -> returns top 50 in < 2s`
- `Fixture: 5 press releases with known best-match journalists -> accuracy > 70%`

#### 8.2 — Relationship Health Scoring

**What**: Compute and maintain per-organization journalist relationship scores based on pitch open rates, reply rates, coverage rates, and interaction recency.

**Design**:

```typescript
// apps/worker/src/processors/compute-scores.processor.ts
interface ComputeScoresJob {
  organizationId: string;
  journalistId?: string;  // null = recompute all
}

// Scoring formula:
// overall_score = (
//   0.30 * normalize(open_rate) +
//   0.25 * normalize(reply_rate) +
//   0.25 * normalize(coverage_rate) +
//   0.10 * recency_factor +     // higher if recently engaged
//   0.10 * consistency_factor   // higher if engagement is consistent
// ) * 100

interface JournalistScoreData {
  overall: number;           // 0-100
  openRate: number;          // 0-1
  replyRate: number;         // 0-1
  coverageRate: number;      // 0-1
  totalPitches: number;
  totalCoverage: number;
  trend: 'improving' | 'stable' | 'declining';
  lastPitchedAt: string | null;
  lastCoverageAt: string | null;
  bestSendTime: string | null;
  preferredTopics: string[];
  responseTimeAvgHours: number | null;
}
```

API endpoint:
```typescript
// GET /api/analytics/relationship-scores -> ranked list of journalists by score
// GET /api/analytics/relationship-scores/:journalistId -> detailed score breakdown
```

**Testing**:
- `Unit: journalist with 80% open rate, 30% reply rate -> high overall score`
- `Unit: journalist with 0 pitches -> score = 0, trend = "stable"`
- `Unit: journalist recently engaged -> recency_factor high`
- `Unit: declining engagement over 3 months -> trend = "declining"`
- `Integration: compute scores job -> updates journalist_score table`
- `Integration: GET /api/analytics/relationship-scores -> sorted by overall descending`

#### 8.3 — Predictive Send-Time Optimisation

**What**: ML model that predicts the best time to send pitches to individual journalists based on historical open/response patterns.

**Design**:

Python endpoint:
```python
# services/ai/src/routers/send_time.py
class SendTimeRequest(BaseModel):
    journalist_id: str
    organization_id: str
    timezone: str  # e.g., "America/New_York"

class SendTimeResponse(BaseModel):
    optimal_time: str         # ISO 8601 datetime
    optimal_day: str          # "Tuesday"
    optimal_hour: int         # 0-23 in journalist's timezone
    confidence: float         # 0-1
    historical_pattern: dict  # {"Monday": {"09": 0.8, "10": 0.6}, ...}

@router.post("/predict")
async def predict_send_time(req: SendTimeRequest) -> SendTimeResponse:
    # 1. Load historical email events for this journalist+org pair
    # 2. Aggregate open times by day-of-week and hour
    # 3. Apply time-series smoothing
    # 4. If insufficient data (< 5 pitches), fall back to global beat averages
    # 5. Return optimal window
    ...
```

**Testing**:
- `Unit: journalist opens emails mostly Tuesday 9am -> optimal_time = next Tuesday 9am`
- `Unit: insufficient data (< 5 pitches) -> falls back to beat average`
- `Unit: timezone conversion -> optimal_hour in journalist's timezone`
- `Integration: campaign with ai_optimized send -> each pitch scheduled at predicted optimal time`
- `Fixture: 10 journalists with known patterns -> predictions match within 2-hour window`

#### 8.4 — AI Press Release Drafting

**What**: LLM-powered press release draft generation from structured inputs (facts, quotes, key messages) with outlet-specific style matching.

**Design**:

Python endpoint:
```python
# services/ai/src/routers/draft.py
class DraftRequest(BaseModel):
    company_name: str
    headline_idea: str
    key_facts: list[str]
    quotes: list[dict]        # [{"speaker": "CEO", "title": "CEO", "text": "..."}]
    key_messages: list[str]
    target_outlet_id: str | None = None  # for style matching
    style: str = "ap_style"   # ap_style, formal, conversational
    word_count_target: int = 400

class DraftResponse(BaseModel):
    headline: str
    subheadline: str | None
    body_html: str
    body_text: str
    word_count: int
    quality_score: float      # 0-100
    readability_score: float  # Flesch-Kincaid
    suggestions: list[str]    # improvement suggestions

@router.post("/generate")
async def generate_draft(req: DraftRequest) -> DraftResponse:
    # 1. If target_outlet_id provided, load outlet style rules
    # 2. Construct prompt with AP Style guidelines, facts, quotes
    # 3. Call Claude API with structured output
    # 4. Post-process: validate AP Style compliance, compute scores
    # 5. Return draft with quality metrics
    ...
```

Prompt template (structure):
```python
SYSTEM_PROMPT = """You are a professional PR writer. Write press releases following AP Style guidelines.
{style_rules}

Output format:
- Headline: Clear, factual, active voice, max 10 words
- Subheadline: Supporting detail (optional)
- Lead paragraph: Who, what, when, where, why in first paragraph
- Body: Supporting facts, quotes, context
- Boilerplate: Company description paragraph
"""
```

**Testing**:
- `Unit: draft with 3 facts and 1 quote -> all facts and quote appear in body`
- `Unit: draft with ap_style -> uses AP date format (month DD, YYYY)`
- `Unit: draft with target_outlet -> adapts tone to outlet style rules`
- `Unit: word_count_target=400 -> generated text within 350-450 words`
- `Integration: POST /api/ai/draft -> returns complete draft with quality score`
- `Integration: draft with outlet style rules -> tone matches outlet profile`

#### 8.5 — Placement Prediction Scoring

**What**: Predict the likelihood of coverage before a pitch is sent, based on journalist history, content relevance, timing, and relationship score.

**Design**:

```python
# services/ai/src/routers/placement.py
class PlacementPredictionRequest(BaseModel):
    journalist_id: str
    organization_id: str
    press_release_summary: str
    press_release_beats: list[str]
    proposed_send_time: str | None = None

class PlacementPredictionResponse(BaseModel):
    probability: float      # 0.0 to 1.0
    confidence: float       # 0.0 to 1.0
    factors: list[dict]     # [{"factor": "beat_match", "impact": "positive", "detail": "..."}]
    recommendation: str     # "strong_match", "moderate_match", "weak_match", "not_recommended"
```

**Testing**:
- `Unit: strong beat match + high relationship score -> probability > 0.5`
- `Unit: no beat match + never pitched -> probability < 0.1`
- `Unit: recently covered competitor -> factor impact = positive`
- `Integration: prediction stored in pitch.personalization JSONB`
- `Integration: GET /api/ai/placement-prediction -> returns prediction for journalist+release pair`

---

## Phase 9: Frontend Application

### Purpose
Build the Next.js web application with all dashboard views, editors, and interactive components. After this phase, the full application is usable through a browser with complete UI for all backend features built in phases 1-8.

### Tasks

#### 9.1 — Application Shell & Navigation

**What**: Next.js App Router layout with sidebar navigation, org context switcher, and authentication flow.

**Design**:

Route structure:
```
/login                    -> Login page
/register                 -> Registration page
/(dashboard)/             -> Authenticated layout
  /journalists            -> Journalist database
  /journalists/[id]       -> Journalist detail
  /press-releases         -> Press release list
  /press-releases/new     -> Create press release
  /press-releases/[id]    -> Edit press release
  /newsroom               -> Newsroom management
  /campaigns              -> Campaign list
  /campaigns/new          -> Create campaign
  /campaigns/[id]         -> Campaign detail + analytics
  /coverage               -> Coverage feed
  /monitoring             -> Monitoring queries
  /analytics              -> Overview dashboard
  /analytics/sov          -> Share of voice
  /settings               -> Organization settings
  /settings/team          -> Team members
```

API client:
```typescript
// apps/web/src/lib/api-client.ts
export class ApiClient {
  constructor(private baseUrl: string, private token: string);
  async get<T>(path: string, params?: Record<string, string>): Promise<T>;
  async post<T>(path: string, body: unknown): Promise<T>;
  async patch<T>(path: string, body: unknown): Promise<T>;
  async delete(path: string): Promise<void>;
}
```

**Testing**:
- `E2E: unauthenticated user visits /journalists -> redirected to /login`
- `E2E: login with valid credentials -> redirected to dashboard`
- `E2E: sidebar navigation -> all links render correct pages`
- `E2E: org context displays in header`

#### 9.2 — Journalist Database UI

**What**: Searchable, filterable journalist table with detail views, inline editing, and media list management.

**Design**:

Components:
```typescript
// JournalistTable: data table with search, filter sidebar, pagination
// JournalistDetail: profile card, outlets, beats, pitch history, scores
// MediaListBuilder: drag-and-drop or checkbox-select journalists into lists
// JournalistImport: CSV upload for bulk import
```

**Testing**:
- `E2E: search "technology" -> filtered results shown`
- `E2E: filter by country=US + beat=AI -> intersected results`
- `E2E: click journalist row -> detail page with outlet history`
- `E2E: create media list -> add 3 journalists -> list shows 3 members`

#### 9.3 — Press Release Editor

**What**: Rich text editor for press releases with multimedia embedding, version history sidebar, and preview mode.

**Design**:

Editor: TipTap (ProseMirror-based) with custom extensions:
```typescript
// Custom extensions:
// - QuoteBlock: for spokesperson quotes with attribution
// - MultimediaEmbed: for images, videos with captions
// - BoilerplateBlock: for company boilerplate paragraph
// - MetadataPanel: sidebar for SEO keywords, IPTC codes, contacts
```

**Testing**:
- `E2E: create press release -> type headline and body -> save as draft`
- `E2E: add image attachment -> appears in editor preview`
- `E2E: submit for review -> status changes to "review"`
- `E2E: view version history -> shows diff between versions`
- `E2E: preview mode -> renders as it would appear in newsroom`

#### 9.4 — Campaign Builder & Analytics

**What**: Campaign creation wizard, pitch preview, real-time analytics dashboard with funnel chart and engagement timeline.

**Design**:

Campaign wizard steps:
```
1. Select press release
2. Select media list(s)
3. Configure personalization (template or AI)
4. Set send schedule (immediate, scheduled, AI-optimized)
5. Configure follow-up rules
6. Preview sample pitches
7. Launch
```

Analytics dashboard components:
```typescript
// FunnelChart: sent -> delivered -> opened -> clicked -> replied -> coverage
// TimelineChart: engagement events over time (line chart)
// TopJournalistsTable: ranked by engagement
// CoverageList: attributed coverage articles
// ReachGauge: total estimated reach
```

**Testing**:
- `E2E: create campaign wizard -> select release, list, schedule -> launch`
- `E2E: campaign analytics -> funnel shows correct counts`
- `E2E: campaign analytics -> timeline chart renders with data points`
- `E2E: real-time update -> new open event appears without page refresh (via polling)`

#### 9.5 — Coverage & Monitoring Views

**What**: Coverage feed with sentiment indicators, monitoring query management, and coverage detail view with attribution info.

**Design**:

Components:
```typescript
// CoverageFeed: chronological list with sentiment badges, outlet logos, reach
// CoverageDetail: full article snippet, sentiment breakdown, attribution chain
// MonitoringQueryManager: CRUD for monitoring queries with live preview
// ShareOfVoiceChart: bar chart comparing self vs. competitors
```

**Testing**:
- `E2E: coverage feed -> shows articles with sentiment colour coding`
- `E2E: filter coverage by sentiment=positive -> only positive articles shown`
- `E2E: coverage detail -> shows attributed campaign and pitch`
- `E2E: monitoring query created -> test results appear`

---

## Phase 10: Newsroom Public Frontend

### Purpose
Build the public-facing newsroom as a standalone rendered page (or Next.js route) with custom branding, SEO optimisation, and social sharing. After this phase, organisations can share their newsroom URL with journalists and the public.

### Tasks

#### 10.1 — Newsroom Landing Page

**What**: Server-rendered public newsroom page with organisation branding, press release grid/list, and category filtering.

**Design**:

```typescript
// Route: /newsroom/[slug]
// SSR: fetch newsroom config + published releases
// Features:
//   - Custom logo, colours from newsroom.config
//   - Grid/list/magazine layout
//   - Category filter tabs
//   - Search within newsroom
//   - Social media feed sidebar (if configured)
//   - Contact info footer
```

**Testing**:
- `E2E: visit /newsroom/acme -> branded page with logo and colours`
- `E2E: published release appears in grid`
- `E2E: unpublished release does not appear`
- `E2E: category filter -> shows only matching releases`
- `E2E: OG meta tags present for social sharing`

#### 10.2 — Press Release Public Page

**What**: Individual press release page with Schema.org JSON-LD, social sharing buttons, multimedia gallery, and related releases.

**Design**:

SEO requirements:
```typescript
// <head> includes:
//   - <title> with headline
//   - <meta name="description"> with summary
//   - Schema.org JSON-LD (NewsArticle)
//   - Open Graph tags (og:title, og:description, og:image)
//   - Twitter Card tags
//   - Canonical URL
```

**Testing**:
- `E2E: press release page -> Schema.org JSON-LD in page source`
- `E2E: press release page -> OG tags for social preview`
- `E2E: embargoed release before embargo time -> 404`
- `E2E: embargoed release after embargo time -> visible`
- `E2E: multimedia gallery -> images render with captions`

---

## Phase 11: Enterprise Features & Integrations

### Purpose
Add enterprise-grade features: SAML SSO, Slack/Teams integration, CRM sync (Salesforce, HubSpot), and webhook API for third-party integrations. After this phase, the platform is ready for enterprise deployment.

### Tasks

#### 11.1 — SAML SSO Integration

**What**: SAML 2.0 single sign-on for enterprise organisations using corporate identity providers (Okta, Azure AD, OneLogin).

**Design**:

```typescript
// apps/api/src/plugins/saml.plugin.ts
interface SamlConfig {
  entryPoint: string;      // IdP SSO URL
  issuer: string;          // SP entity ID
  cert: string;            // IdP X.509 certificate
  callbackUrl: string;     // ACS URL
}

// Routes:
// GET  /api/auth/saml/login        -> redirect to IdP
// POST /api/auth/saml/callback     -> process SAML assertion, create/update user
// GET  /api/auth/saml/metadata     -> SP metadata XML
```

**Testing**:
- `Integration (mocked IdP): SAML flow -> user created/updated, JWT returned`
- `Integration: SAML metadata endpoint -> valid XML with correct ACS URL`
- `Unit: SAML assertion parsing -> extracts email, name, groups`

#### 11.2 — Slack & Teams Integration

**What**: Real-time notifications to Slack/Teams channels for coverage alerts, campaign milestones, and approval requests.

**Design**:

```typescript
interface SlackNotification {
  channel: string;
  blocks: SlackBlock[];
}

// Notification types:
// - New coverage discovered (with sentiment badge)
// - Campaign milestone (100 opens, first coverage)
// - Press release approval request
// - Monitoring alert (negative sentiment)

// Settings per org:
// organization.settings.integrations.slack.webhookUrl
// organization.settings.integrations.teams.webhookUrl
```

**Testing**:
- `Integration (mocked webhook): coverage alert -> Slack message sent with correct blocks`
- `Integration (mocked webhook): approval request -> message includes approve/reject buttons`
- `Unit: Slack block builder -> valid Block Kit JSON`

#### 11.3 — CRM Sync (Salesforce, HubSpot)

**What**: Bidirectional sync of journalist contacts, pitch activity, and coverage data with Salesforce and HubSpot CRMs.

**Design**:

```typescript
// apps/worker/src/processors/crm-sync.processor.ts
interface CrmSyncAdapter {
  readonly name: 'salesforce' | 'hubspot';
  syncContacts(journalists: Journalist[]): Promise<SyncResult>;
  syncActivity(pitches: Pitch[], coverage: Coverage[]): Promise<SyncResult>;
  importContacts(): Promise<Journalist[]>;
}
```

**Testing**:
- `Integration (mocked API): Salesforce sync -> creates/updates contacts`
- `Integration (mocked API): HubSpot sync -> creates contacts with custom properties`
- `Integration: sync conflict resolution -> most recent update wins`

#### 11.4 — Webhook API

**What**: Outbound webhook system for third-party integrations, sending events for coverage discovered, pitch sent, campaign completed.

**Design**:

```typescript
// POST /api/settings/webhooks -> register webhook endpoint
interface WebhookRegistration {
  url: string;
  events: Array<
    'coverage.discovered' | 'pitch.sent' | 'pitch.opened' |
    'pitch.replied' | 'campaign.completed' | 'release.published'
  >;
  secret: string;  // HMAC signing secret
}

// Webhook payload:
interface WebhookPayload {
  event: string;
  timestamp: string;
  data: Record<string, unknown>;
}
// Signed with HMAC-SHA256 in X-Signature header
```

**Testing**:
- `Unit: HMAC signature generation -> matches expected hex digest`
- `Integration: coverage discovered -> webhook sent to registered URL`
- `Integration: webhook delivery failure -> retried 3 times with exponential backoff`
- `Integration: webhook with invalid URL -> marked as failed, admin notified`

---

## Phase 12: Production Hardening & Deployment

### Purpose
Prepare the platform for production deployment with Docker production builds, health checks, database connection pooling, observability (logging, metrics, tracing), rate limiting, and security hardening. After this phase, the platform is deployable to cloud or self-hosted environments.

### Tasks

#### 12.1 — Production Docker Configuration

**What**: Multi-stage Docker builds for all services, production Docker Compose, and health check endpoints.

**Design**:

```dockerfile
# apps/api/Dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --production=false
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
EXPOSE 3001
HEALTHCHECK --interval=30s CMD wget -q --spider http://localhost:3001/health || exit 1
CMD ["node", "dist/index.js"]
```

Health check endpoint:
```typescript
// GET /health -> { status: 'ok', version: string, uptime: number }
// GET /health/ready -> checks DB, Redis, Meilisearch connectivity
```

**Testing**:
- `Integration: docker build -> image builds under 2 minutes`
- `Integration: docker run -> health check passes`
- `Integration: /health/ready with DB down -> 503 with details`

#### 12.2 — Observability

**What**: Structured logging (Pino), metrics (Prometheus), and distributed tracing (OpenTelemetry).

**Design**:

```typescript
// Logging: Pino with JSON output, request ID correlation
// Metrics: prom-client with custom counters
//   - pr_pitches_sent_total (counter)
//   - pr_coverage_discovered_total (counter)
//   - pr_api_request_duration_seconds (histogram)
//   - pr_email_delivery_status (counter, by status)
//   - pr_ai_inference_duration_seconds (histogram)
// Tracing: @opentelemetry/api with Fastify instrumentation
```

**Testing**:
- `Unit: Pino logger -> JSON output with timestamp, level, requestId`
- `Integration: Prometheus /metrics endpoint -> returns valid metrics`
- `Integration: API request -> creates trace span with correct attributes`

#### 12.3 — Security Hardening

**What**: Rate limiting, CORS, CSP headers, input sanitisation, and SQL injection prevention (via Prisma parameterised queries). Alignment with OWASP API Security Top 10.

**Design**:

```typescript
// Rate limiting (per IP and per org):
const RATE_LIMITS = {
  api: { window: '1 minute', max: 100 },
  auth: { window: '15 minutes', max: 10 },
  webhooks: { window: '1 minute', max: 500 },
};

// CORS: configurable allowed origins per org
// CSP: strict policy for newsroom pages
// Input sanitisation: HTML in press release body via DOMPurify
// API key authentication: for webhook and external API access
```

**Testing**:
- `Integration: 101 requests in 1 minute -> 429 Too Many Requests`
- `Integration: CORS preflight from unknown origin -> blocked`
- `Integration: XSS payload in press release body -> sanitised by DOMPurify`
- `Integration: SQL injection attempt in query param -> Prisma parameterised, no injection`

#### 12.4 — Database Optimisation

**What**: Connection pooling (PgBouncer), query optimisation, and periodic maintenance tasks (VACUUM, index maintenance).

**Design**:

```yaml
# docker-compose.prod.yml
services:
  pgbouncer:
    image: edoburu/pgbouncer:latest
    environment:
      DATABASE_URL: postgres://user:pass@postgres:5432/prmanagement
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 200
      DEFAULT_POOL_SIZE: 25
```

**Testing**:
- `Integration: 100 concurrent API requests -> all succeed (connection pool handles load)`
- `Integration: slow query logging -> queries over 500ms logged with EXPLAIN output`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Scaffolding         ─── required by everything
    │
Phase 2: Journalist Database & Lists      ─── requires Phase 1
    │
Phase 3: Press Release & Newsroom         ─── requires Phase 1
    │
    ├── Phase 4: Campaign & Pitching      ─── requires Phases 2 + 3
    │       │
    │       └── Phase 5: Coverage Monitoring ─── requires Phase 4
    │               │
    │               └── Phase 6: Analytics Dashboard ─── requires Phase 5
    │
    └── Phase 7: Wire Distribution        ─── requires Phase 3 (can parallel with 4-6)
    
Phase 8: AI-Powered Features             ─── requires Phases 2 + 4 + 5
    │
Phase 9: Frontend Application            ─── requires Phases 2-6 (can start after Phase 2, grows with each backend phase)
    │
Phase 10: Newsroom Public Frontend        ─── requires Phase 3 (can parallel with 4-8)

Phase 11: Enterprise Features             ─── requires Phases 1-6 (can parallel with 8, 10)

Phase 12: Production Hardening            ─── requires all above, final phase
```

### Parallelism Opportunities

- **Phases 2 and 3** can be developed concurrently after Phase 1
- **Phase 7** (Wire Distribution) can proceed in parallel with Phases 4-6
- **Phase 9** (Frontend) can start as soon as Phase 2 is complete and grow incrementally
- **Phase 10** (Newsroom Frontend) can proceed in parallel with Phases 4-8
- **Phase 11** (Enterprise) can proceed in parallel with Phases 8 and 10

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented with complete functionality.
2. All unit tests pass (Vitest for TypeScript, pytest for Python).
3. All integration tests pass (with mocked external services where applicable).
4. TypeScript strict mode compiles without errors.
5. Python type checking passes (mypy or pyright).
6. ESLint (TypeScript) and Ruff (Python) pass with zero errors.
7. Prettier formatting applied to all TypeScript/TSX files.
8. Docker build succeeds for all affected services.
9. Docker Compose up brings all services to healthy state.
10. Prisma migrations created for any schema changes.
11. New API endpoints documented in auto-generated OpenAPI spec.
12. New environment variables documented in `.env.example`.
13. Meilisearch indexes updated if journalist/outlet schema changes.
14. BullMQ job types registered and processors tested.
15. Audit log entries created for all state-changing operations.
16. GDPR compliance maintained (consent tracking, suppression lists honoured).
17. Feature works end-to-end (API call from frontend through to database and back).
