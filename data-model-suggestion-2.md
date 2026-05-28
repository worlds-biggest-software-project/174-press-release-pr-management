# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Press Release & PR Management · Created: 2026-05-20

## Philosophy

This model treats every state change as an immutable event in a central event store. The current state of any entity — a press release, a journalist relationship, a campaign — is derived by replaying its event history. Read-optimized projections (materialized views) serve the UI and API, while the event store serves as the authoritative source of truth.

This pattern is used by financial trading platforms, healthcare systems, and compliance-heavy SaaS products where "what happened and when" is as important as "what is true now." For PR management specifically, it enables powerful capabilities: reconstructing the exact state of a press release at embargo time, proving the full chain of approval for regulatory filings, replaying journalist interaction timelines, and providing AI models with rich temporal training data.

The CQRS (Command Query Responsibility Segregation) layer separates write operations (commands that produce events) from read operations (queries against projections). This allows the read side to be optimized independently — denormalized, cached, or served from a search index — while the write side maintains strict append-only semantics.

**Best for:** Teams building for regulatory compliance (SEC filings, GDPR audit trails), temporal analytics ("what did we know on date X?"), and AI training on interaction history patterns.

**Trade-offs:**
- (+) Complete, immutable audit trail of every action — ideal for embargo compliance and regulatory filings
- (+) Temporal queries are natural: "show the press release as it was at embargo time"
- (+) Rich event history feeds AI models for relationship scoring and prediction
- (+) Read models can be rebuilt from events if requirements change
- (-) Higher storage cost (events accumulate indefinitely)
- (-) Eventual consistency between event store and read projections
- (-) More complex development: developers must think in events, not CRUD
- (-) Debugging requires understanding event replay, not just inspecting current state

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IPTC ninjs 3.2 | Press release content events carry ninjs-aligned payloads (headline, byline, body, language) |
| Schema.org NewsArticle | Read projection for newsroom publishes Schema.org-compliant structured data |
| IPTC NewsCodes | Event payloads reference IPTC subject codes; projections index by NewsCodes taxonomy |
| ISO 3166-1/2 | Geographic identifiers in journalist and outlet events use ISO 3166 |
| GDPR Art. 6(1)(f) | Consent events (`journalist.consent_granted`, `journalist.opted_out`) form an immutable compliance log |
| SEC EDGAR | Distribution events for regulatory filings carry EDGAR accession numbers and filing metadata |
| AP Stylebook | Style guide events record outlet-specific formatting rules applied during content generation |
| OCSF Event Schema | Event structure loosely follows Open Cybersecurity Schema Framework patterns for structured logging |

---

## Event Store (Source of Truth)

```sql
-- The single source of truth: every state change is an event
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,             -- aggregate root ID (e.g., press_release_id)
    stream_type     VARCHAR(50) NOT NULL,       -- press_release, journalist, campaign, pitch, coverage
    event_type      VARCHAR(100) NOT NULL,      -- e.g., press_release.created, pitch.sent, coverage.discovered
    event_version   INTEGER NOT NULL,           -- sequence number within stream
    payload         JSONB NOT NULL,             -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}', -- actor_id, organization_id, ip_address, correlation_id
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
);

-- Critical indexes for event replay and projection building
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version ASC);
CREATE INDEX idx_event_type ON event_store(event_type, occurred_at DESC);
CREATE INDEX idx_event_org ON event_store((metadata->>'organization_id'), occurred_at DESC);
CREATE INDEX idx_event_occurred ON event_store(occurred_at DESC);

-- Example events and their payloads:
--
-- event_type: "press_release.created"
-- payload: {
--   "headline": "Acme Corp Launches New Product",
--   "body_html": "<p>...</p>",
--   "body_text": "...",
--   "language_code": "en",
--   "author_id": "uuid-...",
--   "iptc_subject_codes": ["04000000"],
--   "embargo_until": "2026-06-01T09:00:00Z"
-- }
--
-- event_type: "press_release.approved"
-- payload: {
--   "approved_by": "uuid-...",
--   "approval_notes": "Approved for distribution"
-- }
--
-- event_type: "pitch.sent"
-- payload: {
--   "journalist_id": "uuid-...",
--   "subject_line": "Exclusive: Acme Corp Launch",
--   "body_html": "...",
--   "dkim_status": "pass",
--   "message_id": "<abc@mail.example.com>"
-- }
--
-- event_type: "pitch.opened"
-- payload: {
--   "ip_address": "203.0.113.1",
--   "user_agent": "Mozilla/5.0...",
--   "is_bot": false
-- }
--
-- event_type: "journalist.beat_changed"
-- payload: {
--   "old_beats": ["technology"],
--   "new_beats": ["technology", "ai"],
--   "source": "ai_inferred"
-- }
--
-- event_type: "coverage.attributed"
-- payload: {
--   "coverage_url": "https://example.com/article",
--   "campaign_id": "uuid-...",
--   "pitch_id": "uuid-...",
--   "sentiment_score": 0.75
-- }
```

## Command Log

```sql
-- Commands that trigger events (optional, useful for debugging)
CREATE TABLE command_log (
    command_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    command_type    VARCHAR(100) NOT NULL,   -- create_press_release, send_pitch, approve_release
    payload         JSONB NOT NULL,
    actor_id        UUID NOT NULL,
    organization_id UUID NOT NULL,
    correlation_id  UUID NOT NULL,          -- links command to resulting events
    status          VARCHAR(20) NOT NULL DEFAULT 'accepted',  -- accepted, rejected, failed
    rejection_reason TEXT,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_command_correlation ON command_log(correlation_id);
CREATE INDEX idx_command_actor ON command_log(actor_id, received_at DESC);
```

## Snapshot Store (Performance Optimization)

```sql
-- Periodic snapshots to avoid replaying full event history
CREATE TABLE snapshot_store (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,      -- event_version at snapshot time
    state           JSONB NOT NULL,         -- full aggregate state at this point
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- Keep only latest snapshot per stream for fast lookup
CREATE INDEX idx_snapshot_latest ON snapshot_store(stream_id, snapshot_version DESC);
```

---

## Read Projections (Query Side)

These tables are materialized from events and can be rebuilt at any time by replaying the event store.

### Identity & Multi-Tenancy Projection

```sql
CREATE TABLE proj_organization (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE proj_user (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES proj_organization(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    roles           TEXT[] NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_user_org ON proj_user(organization_id);
```

### Journalist Database Projection

```sql
CREATE TABLE proj_journalist (
    id              UUID PRIMARY KEY,
    full_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    title           VARCHAR(255),
    beats           TEXT[] NOT NULL DEFAULT '{}',
    country_code    CHAR(2),
    city            VARCHAR(255),
    preferred_contact_method VARCHAR(50),
    consent_status  VARCHAR(20) NOT NULL DEFAULT 'implied',
    current_outlets JSONB DEFAULT '[]',
    -- Denormalized: [{outlet_id, outlet_name, role, since}]
    social_profiles JSONB DEFAULT '{}',
    -- {"twitter": "@handle", "linkedin": "url"}
    is_verified     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE proj_outlet (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    outlet_type     VARCHAR(50) NOT NULL,
    website_url     VARCHAR(500),
    country_code    CHAR(2),
    language_code   CHAR(2) DEFAULT 'en',
    domain_authority SMALLINT,
    journalist_count INTEGER NOT NULL DEFAULT 0,  -- denormalized count
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_journalist_email ON proj_journalist(email);
CREATE INDEX idx_proj_journalist_beats ON proj_journalist USING GIN(beats);
CREATE INDEX idx_proj_journalist_country ON proj_journalist(country_code);
CREATE INDEX idx_proj_outlet_type ON proj_outlet(outlet_type);
```

### Press Release Projection

```sql
CREATE TABLE proj_press_release (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    headline        VARCHAR(500) NOT NULL,
    subheadline     VARCHAR(500),
    body_html       TEXT NOT NULL,
    body_text       TEXT NOT NULL,
    summary         TEXT,
    language_code   CHAR(2) NOT NULL DEFAULT 'en',
    status          VARCHAR(30) NOT NULL,
    author_name     VARCHAR(255),
    author_id       UUID,
    approved_by_name VARCHAR(255),
    approved_at     TIMESTAMPTZ,
    embargo_until   TIMESTAMPTZ,
    published_at    TIMESTAMPTZ,
    newsroom_url    VARCHAR(500),
    iptc_subject_codes TEXT[],
    tags            TEXT[] DEFAULT '{}',
    attachments     JSONB DEFAULT '[]',
    -- [{file_name, file_type, storage_url, alt_text}]
    version_count   INTEGER NOT NULL DEFAULT 1,
    word_count      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_pr_org ON proj_press_release(organization_id);
CREATE INDEX idx_proj_pr_status ON proj_press_release(status);
CREATE INDEX idx_proj_pr_published ON proj_press_release(published_at DESC);
```

### Campaign & Pitch Projection

```sql
CREATE TABLE proj_campaign (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    press_release_id UUID,
    press_release_headline VARCHAR(500),   -- denormalized for display
    status          VARCHAR(30) NOT NULL,
    total_pitches   INTEGER NOT NULL DEFAULT 0,
    total_delivered INTEGER NOT NULL DEFAULT 0,
    total_opened    INTEGER NOT NULL DEFAULT 0,
    total_clicked   INTEGER NOT NULL DEFAULT 0,
    total_replied   INTEGER NOT NULL DEFAULT 0,
    total_bounced   INTEGER NOT NULL DEFAULT 0,
    total_coverage  INTEGER NOT NULL DEFAULT 0,
    total_reach     BIGINT DEFAULT 0,
    avg_sentiment   NUMERIC(3,2),
    created_by_name VARCHAR(255),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE proj_pitch (
    id              UUID PRIMARY KEY,
    campaign_id     UUID NOT NULL,
    campaign_name   VARCHAR(255),          -- denormalized
    journalist_id   UUID NOT NULL,
    journalist_name VARCHAR(255),          -- denormalized
    journalist_email VARCHAR(255),
    outlet_name     VARCHAR(255),          -- denormalized
    sender_name     VARCHAR(255),
    subject_line    VARCHAR(500) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    sent_at         TIMESTAMPTZ,
    open_count      INTEGER NOT NULL DEFAULT 0,
    click_count     INTEGER NOT NULL DEFAULT 0,
    has_replied     BOOLEAN NOT NULL DEFAULT false,
    replied_at      TIMESTAMPTZ,
    last_event_at   TIMESTAMPTZ,           -- most recent engagement
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_campaign_org ON proj_campaign(organization_id);
CREATE INDEX idx_proj_pitch_campaign ON proj_pitch(campaign_id);
CREATE INDEX idx_proj_pitch_journalist ON proj_pitch(journalist_id);
```

### Coverage Projection

```sql
CREATE TABLE proj_coverage (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    headline        VARCHAR(500) NOT NULL,
    url             VARCHAR(500),
    outlet_name     VARCHAR(255),
    outlet_id       UUID,
    journalist_name VARCHAR(255),
    journalist_id   UUID,
    published_at    TIMESTAMPTZ,
    coverage_type   VARCHAR(50) NOT NULL,
    sentiment_score NUMERIC(3,2),
    sentiment_label VARCHAR(20),
    reach_estimate  BIGINT,
    campaign_name   VARCHAR(255),          -- denormalized attribution
    campaign_id     UUID,
    tags            TEXT[] DEFAULT '{}',
    discovered_at   TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_coverage_org ON proj_coverage(organization_id);
CREATE INDEX idx_proj_coverage_published ON proj_coverage(published_at DESC);
CREATE INDEX idx_proj_coverage_sentiment ON proj_coverage(sentiment_label);
```

### Relationship Score Projection

```sql
CREATE TABLE proj_relationship_score (
    organization_id UUID NOT NULL,
    journalist_id   UUID NOT NULL,
    journalist_name VARCHAR(255),
    overall_score   NUMERIC(5,2) NOT NULL,
    open_rate       NUMERIC(5,2),
    reply_rate      NUMERIC(5,2),
    coverage_rate   NUMERIC(5,2),
    total_pitches   INTEGER NOT NULL DEFAULT 0,
    total_coverage  INTEGER NOT NULL DEFAULT 0,
    last_pitched_at TIMESTAMPTZ,
    last_coverage_at TIMESTAMPTZ,
    score_trend     VARCHAR(20),           -- improving, stable, declining
    computed_at     TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (organization_id, journalist_id)
);

CREATE INDEX idx_proj_rel_score ON proj_relationship_score(overall_score DESC);
```

---

## Projection Rebuild Infrastructure

```sql
-- Tracks which projections are current and their rebuild status
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID,
    last_event_at   TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',  -- active, rebuilding, failed
    rebuild_started_at TIMESTAMPTZ,
    rebuild_completed_at TIMESTAMPTZ,
    event_count     BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Temporal Query: Press Release State at Embargo Time

```sql
-- Reconstruct the press release as it existed at embargo time
-- by replaying events up to that timestamp
SELECT payload
FROM event_store
WHERE stream_id = '{{press_release_id}}'
  AND stream_type = 'press_release'
  AND occurred_at <= '2026-06-01T09:00:00Z'
ORDER BY event_version ASC;

-- The application replays these events in order to build the state:
-- 1. press_release.created  -> initial state
-- 2. press_release.edited   -> apply body changes
-- 3. press_release.approved -> mark as approved
-- Result: exact content that was approved before embargo lifted
```

## Example: Journalist Interaction Timeline

```sql
-- Full timeline of interactions with a specific journalist
SELECT
    event_type,
    payload,
    occurred_at,
    metadata->>'actor_id' AS actor
FROM event_store
WHERE stream_type IN ('pitch', 'coverage')
  AND payload->>'journalist_id' = '{{journalist_id}}'
  AND metadata->>'organization_id' = '{{org_id}}'
ORDER BY occurred_at ASC;
```

## Example: Campaign Funnel Reconstruction

```sql
-- Reconstruct campaign funnel at any point in time
SELECT
    event_type,
    COUNT(*) AS event_count,
    MIN(occurred_at) AS first_at,
    MAX(occurred_at) AS last_at
FROM event_store
WHERE stream_type = 'pitch'
  AND payload->>'campaign_id' = '{{campaign_id}}'
  AND event_type IN ('pitch.sent', 'pitch.delivered', 'pitch.opened', 'pitch.clicked', 'pitch.replied')
GROUP BY event_type
ORDER BY MIN(occurred_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store, command_log, snapshot_store |
| Projection Infrastructure | 1 | projection_checkpoint |
| Identity Projections | 2 | proj_organization, proj_user |
| Journalist Projections | 2 | proj_journalist, proj_outlet |
| Press Release Projections | 1 | proj_press_release |
| Campaign Projections | 2 | proj_campaign, proj_pitch |
| Coverage Projections | 1 | proj_coverage |
| Analytics Projections | 1 | proj_relationship_score |
| **Total** | **13** | 3 event tables + 1 infra + 9 projections |

---

## Key Design Decisions

1. **Single event store table** — all domain events live in one table, partitioned by `stream_type`. This simplifies event replay and enables cross-aggregate temporal queries. PostgreSQL table partitioning by `occurred_at` month handles volume growth.

2. **Events carry full context in payload** — each event payload contains all the data needed to process it, following the "fat event" pattern. This makes projections self-contained and avoids cross-event lookups during replay.

3. **Metadata separates cross-cutting concerns** — `organization_id`, `actor_id`, `ip_address`, and `correlation_id` live in metadata rather than payload, keeping domain data clean while supporting multi-tenancy and audit.

4. **Snapshots prevent unbounded replay** — for entities with long event histories (e.g., a journalist with thousands of interaction events), periodic snapshots allow state reconstruction from the latest snapshot + subsequent events only.

5. **Projections are denormalized and disposable** — read projections deliberately duplicate data (e.g., `journalist_name` in `proj_pitch`) to avoid JOINs. They can be dropped and rebuilt from the event store if the read model needs to change.

6. **Campaign analytics are event-derived, not stored** — pitch counts, open rates, and coverage attribution are computed by processing events. The `proj_campaign` projection stores denormalized totals that update as new events arrive, but the event store remains authoritative.

7. **GDPR compliance through consent events** — `journalist.consent_granted` and `journalist.opted_out` events create an immutable consent trail. GDPR right-to-erasure is handled by "crypto-shredding" — encrypting journalist PII in events with a per-journalist key, then deleting the key on erasure request.

8. **Embargo compliance is built-in** — because the event store preserves the exact state of a press release at any point in time, embargo compliance audits can reconstruct exactly what was distributed, when, and to whom — without relying on mutable state.

9. **AI model training from event streams** — the event store provides a natural training dataset for AI features: journalist response patterns, optimal send times, coverage prediction signals, all with precise timestamps and sequencing.

10. **Projection checkpoint enables safe rebuilds** — the `projection_checkpoint` table tracks which events each projection has processed, enabling incremental updates and safe full rebuilds without data loss or double-processing.
