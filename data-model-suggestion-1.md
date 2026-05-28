# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Press Release & PR Management · Created: 2026-05-20

## Philosophy

This model follows classical relational database design with full normalization (3NF+). Every concept in the PR management domain — journalists, outlets, press releases, campaigns, pitches, interactions, analytics events — gets its own dedicated table with strongly typed columns, foreign keys, and referential integrity constraints.

The approach mirrors how enterprise CRM systems (Salesforce, HubSpot) and mature PR platforms (Cision, Muck Rack) structure their data internally. It optimizes for data integrity, complex cross-entity queries (e.g., "find all journalists who covered competitor X in the last 90 days and have an open rate above 40%"), and regulatory compliance through explicit audit columns.

This is the safest choice for a team experienced with relational databases who want a predictable, well-understood schema that can be evolved through standard migration tooling. The trade-off is more tables and more JOIN operations, but PostgreSQL handles this efficiently with proper indexing.

**Best for:** Teams prioritizing data integrity, complex analytics queries, and long-term schema stability in a well-defined PR domain.

**Trade-offs:**
- (+) Strong referential integrity prevents orphaned records
- (+) Complex cross-entity queries are natural with JOINs
- (+) Well-understood migration patterns (Flyway, Alembic, Prisma)
- (+) Easy to reason about data lineage and compliance
- (-) More tables means more JOIN complexity for simple operations
- (-) Schema changes require migrations; adding jurisdiction-specific fields is heavyweight
- (-) Journalist metadata varies by region/outlet type; normalized columns may not capture all variants

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IPTC ninjs 3.2 | Press release content structure mirrors ninjs properties (uri, headline, byline, body, language) |
| Schema.org NewsArticle | Newsroom published releases store structured data fields matching Schema.org properties |
| IPTC NewsCodes | `topic_codes` and `subject_codes` reference IPTC controlled vocabularies for categorization |
| ISO 3166-1/2 | `country_code` and `region_code` columns use ISO 3166 for journalist and outlet geography |
| ISO 639-1 | `language_code` columns use ISO 639-1 two-letter codes for multi-language support |
| GDPR Art. 6(1)(f) | `consent_status`, `opt_out_date`, `data_source` columns support legitimate interest tracking |
| CAN-SPAM / CASL | `suppression_list` table and `unsubscribe_token` support compliance |
| RFC 6376 (DKIM) | `email_send` records track DKIM signing status for deliverability |
| AP Stylebook | `style_guide` reference table supports outlet-specific formatting rules |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, pro, enterprise
    billing_email   VARCHAR(255),
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    avatar_url      VARCHAR(500),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'email',  -- email, google, saml
    auth_subject    VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(100) NOT NULL,  -- admin, editor, viewer, pitch_sender
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    granted_by      UUID REFERENCES app_user(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);

CREATE INDEX idx_user_org ON app_user(organization_id);
CREATE INDEX idx_role_org ON role(organization_id);
```

## Journalist Database & CRM

```sql
CREATE TABLE outlet (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    outlet_type     VARCHAR(50) NOT NULL,  -- newspaper, magazine, blog, tv, radio, podcast, newsletter
    website_url     VARCHAR(500),
    country_code    CHAR(2),               -- ISO 3166-1 alpha-2
    region_code     VARCHAR(6),            -- ISO 3166-2
    language_code   CHAR(2) DEFAULT 'en',  -- ISO 639-1
    circulation     INTEGER,
    domain_authority SMALLINT,             -- 0-100
    is_verified     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE journalist (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    full_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(50),
    title           VARCHAR(255),          -- e.g., "Senior Tech Reporter"
    bio             TEXT,
    twitter_handle  VARCHAR(100),
    linkedin_url    VARCHAR(500),
    preferred_contact_method VARCHAR(50),   -- email, twitter_dm, phone
    preferred_contact_time   VARCHAR(100),  -- e.g., "mornings EST"
    country_code    CHAR(2),               -- ISO 3166-1
    city            VARCHAR(255),
    is_verified     BOOLEAN NOT NULL DEFAULT false,
    verified_at     TIMESTAMPTZ,
    consent_status  VARCHAR(20) NOT NULL DEFAULT 'implied',  -- implied, explicit, opted_out
    opt_out_date    TIMESTAMPTZ,
    data_source     VARCHAR(100),          -- manual, api_import, web_scrape
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE journalist_outlet (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journalist_id   UUID NOT NULL REFERENCES journalist(id) ON DELETE CASCADE,
    outlet_id       UUID NOT NULL REFERENCES outlet(id) ON DELETE CASCADE,
    role_at_outlet  VARCHAR(100),          -- staff_writer, contributor, editor, freelance
    start_date      DATE,
    end_date        DATE,                  -- NULL = current
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (journalist_id, outlet_id, start_date)
);

CREATE TABLE journalist_beat (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journalist_id   UUID NOT NULL REFERENCES journalist(id) ON DELETE CASCADE,
    beat_name       VARCHAR(100) NOT NULL, -- technology, healthcare, finance, politics
    confidence      NUMERIC(3,2) DEFAULT 1.0,  -- 0.00-1.00 AI-inferred confidence
    source          VARCHAR(50) NOT NULL,  -- manual, ai_inferred, profile_stated
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (journalist_id, beat_name)
);

CREATE TABLE suppression_list (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    email           VARCHAR(255) NOT NULL,
    reason          VARCHAR(50) NOT NULL,  -- opt_out, bounce, complaint, manual
    suppressed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_journalist_email ON journalist(email);
CREATE INDEX idx_journalist_country ON journalist(country_code);
CREATE INDEX idx_journalist_outlet_active ON journalist_outlet(journalist_id) WHERE end_date IS NULL;
CREATE INDEX idx_journalist_beat ON journalist_beat(journalist_id);
CREATE INDEX idx_outlet_type ON outlet(outlet_type);
CREATE INDEX idx_outlet_country ON outlet(country_code);
```

## Media Lists

```sql
CREATE TABLE media_list (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    is_dynamic      BOOLEAN NOT NULL DEFAULT false,  -- static list vs. saved search
    filter_criteria JSONB,                 -- for dynamic lists: {"beats": ["tech"], "countries": ["US","GB"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE media_list_member (
    media_list_id   UUID NOT NULL REFERENCES media_list(id) ON DELETE CASCADE,
    journalist_id   UUID NOT NULL REFERENCES journalist(id) ON DELETE CASCADE,
    added_by        UUID REFERENCES app_user(id),
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    notes           TEXT,
    PRIMARY KEY (media_list_id, journalist_id)
);

CREATE INDEX idx_media_list_org ON media_list(organization_id);
```

## Press Release Authoring

```sql
CREATE TABLE press_release (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    headline        VARCHAR(500) NOT NULL,
    subheadline     VARCHAR(500),
    body_html       TEXT NOT NULL,
    body_text       TEXT NOT NULL,          -- plain-text version for email
    summary         TEXT,                   -- boilerplate / abstract
    language_code   CHAR(2) NOT NULL DEFAULT 'en',  -- ISO 639-1
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',  -- draft, review, approved, published, archived
    author_id       UUID NOT NULL REFERENCES app_user(id),
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    embargo_until   TIMESTAMPTZ,           -- NULL = no embargo
    published_at    TIMESTAMPTZ,
    newsroom_url    VARCHAR(500),           -- public URL on branded newsroom
    schema_org_type VARCHAR(50) DEFAULT 'NewsArticle',  -- Schema.org type
    seo_keywords    TEXT[],
    iptc_subject_codes TEXT[],             -- IPTC NewsCodes subject references
    word_count      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE press_release_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    press_release_id UUID NOT NULL REFERENCES press_release(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    headline        VARCHAR(500) NOT NULL,
    body_html       TEXT NOT NULL,
    body_text       TEXT NOT NULL,
    changed_by      UUID NOT NULL REFERENCES app_user(id),
    change_summary  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (press_release_id, version_number)
);

CREATE TABLE press_release_attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    press_release_id UUID NOT NULL REFERENCES press_release(id) ON DELETE CASCADE,
    file_name       VARCHAR(255) NOT NULL,
    file_type       VARCHAR(50) NOT NULL,  -- image, video, document, infographic
    mime_type       VARCHAR(100) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    storage_url     VARCHAR(500) NOT NULL,
    alt_text        VARCHAR(500),
    caption         TEXT,
    sort_order      SMALLINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE press_release_tag (
    press_release_id UUID NOT NULL REFERENCES press_release(id) ON DELETE CASCADE,
    tag_name        VARCHAR(100) NOT NULL,
    PRIMARY KEY (press_release_id, tag_name)
);

CREATE INDEX idx_pr_org ON press_release(organization_id);
CREATE INDEX idx_pr_status ON press_release(status);
CREATE INDEX idx_pr_published ON press_release(published_at DESC) WHERE published_at IS NOT NULL;
CREATE INDEX idx_pr_embargo ON press_release(embargo_until) WHERE embargo_until IS NOT NULL;
CREATE INDEX idx_pr_version ON press_release_version(press_release_id, version_number DESC);
```

## Newsroom / Owned Media

```sql
CREATE TABLE newsroom (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    custom_domain   VARCHAR(255),
    logo_url        VARCHAR(500),
    brand_colors    JSONB DEFAULT '{}',    -- {"primary": "#1a73e8", "secondary": "#fff"}
    seo_title       VARCHAR(200),
    seo_description VARCHAR(500),
    social_links    JSONB DEFAULT '{}',    -- {"twitter": "...", "linkedin": "..."}
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE TABLE newsroom_release (
    newsroom_id     UUID NOT NULL REFERENCES newsroom(id) ON DELETE CASCADE,
    press_release_id UUID NOT NULL REFERENCES press_release(id) ON DELETE CASCADE,
    published_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_featured     BOOLEAN NOT NULL DEFAULT false,
    PRIMARY KEY (newsroom_id, press_release_id)
);
```

## Campaign & Pitching

```sql
CREATE TABLE campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    press_release_id UUID REFERENCES press_release(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',  -- draft, active, paused, completed
    scheduled_at    TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pitch (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES campaign(id) ON DELETE CASCADE,
    journalist_id   UUID NOT NULL REFERENCES journalist(id),
    sender_id       UUID NOT NULL REFERENCES app_user(id),
    subject_line    VARCHAR(500) NOT NULL,
    body_html       TEXT NOT NULL,
    body_text       TEXT NOT NULL,
    personalization_notes TEXT,            -- AI-generated personalization context
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',  -- draft, queued, sent, delivered, bounced, failed
    sent_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    bounce_type     VARCHAR(20),           -- hard, soft
    unsubscribe_token VARCHAR(100) UNIQUE,
    message_id      VARCHAR(255),          -- SMTP Message-ID for threading
    dkim_status     VARCHAR(20),           -- pass, fail, none
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pitch_followup (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pitch_id        UUID NOT NULL REFERENCES pitch(id) ON DELETE CASCADE,
    followup_number SMALLINT NOT NULL,
    subject_line    VARCHAR(500),
    body_html       TEXT NOT NULL,
    body_text       TEXT NOT NULL,
    scheduled_at    TIMESTAMPTZ NOT NULL,
    sent_at         TIMESTAMPTZ,
    status          VARCHAR(30) NOT NULL DEFAULT 'scheduled',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_org ON campaign(organization_id);
CREATE INDEX idx_campaign_status ON campaign(status);
CREATE INDEX idx_pitch_campaign ON pitch(campaign_id);
CREATE INDEX idx_pitch_journalist ON pitch(journalist_id);
CREATE INDEX idx_pitch_status ON pitch(status);
CREATE INDEX idx_pitch_sent ON pitch(sent_at DESC) WHERE sent_at IS NOT NULL;
```

## Email Engagement Tracking

```sql
CREATE TABLE email_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pitch_id        UUID NOT NULL REFERENCES pitch(id) ON DELETE CASCADE,
    event_type      VARCHAR(30) NOT NULL,  -- open, click, reply, unsubscribe, complaint
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address      INET,
    user_agent      TEXT,
    link_url        VARCHAR(500),          -- for click events
    is_bot          BOOLEAN DEFAULT false, -- filter out bot opens
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_email_event_pitch ON email_event(pitch_id);
CREATE INDEX idx_email_event_type ON email_event(event_type, occurred_at DESC);
```

## Media Monitoring & Coverage

```sql
CREATE TABLE coverage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    headline        VARCHAR(500) NOT NULL,
    url             VARCHAR(500),
    outlet_id       UUID REFERENCES outlet(id),
    journalist_id   UUID REFERENCES journalist(id),
    published_at    TIMESTAMPTZ,
    discovered_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    content_snippet TEXT,                  -- first 500 chars
    coverage_type   VARCHAR(50) NOT NULL,  -- article, mention, interview, op_ed, broadcast, podcast
    sentiment_score NUMERIC(3,2),          -- -1.00 to 1.00
    sentiment_label VARCHAR(20),           -- positive, neutral, negative, mixed
    reach_estimate  BIGINT,                -- estimated audience reach
    share_count     INTEGER DEFAULT 0,
    is_attributed   BOOLEAN NOT NULL DEFAULT false,  -- linked to a campaign
    attributed_campaign_id UUID REFERENCES campaign(id),
    attributed_pitch_id    UUID REFERENCES pitch(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE coverage_tag (
    coverage_id     UUID NOT NULL REFERENCES coverage(id) ON DELETE CASCADE,
    tag_name        VARCHAR(100) NOT NULL,
    PRIMARY KEY (coverage_id, tag_name)
);

CREATE TABLE monitoring_query (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    query_string    TEXT NOT NULL,          -- boolean search query
    sources         TEXT[],                -- news, social, podcast, broadcast
    is_active       BOOLEAN NOT NULL DEFAULT true,
    alert_enabled   BOOLEAN NOT NULL DEFAULT false,
    alert_channels  TEXT[],                -- email, slack, webhook
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coverage_org ON coverage(organization_id);
CREATE INDEX idx_coverage_outlet ON coverage(outlet_id);
CREATE INDEX idx_coverage_journalist ON coverage(journalist_id);
CREATE INDEX idx_coverage_published ON coverage(published_at DESC);
CREATE INDEX idx_coverage_sentiment ON coverage(sentiment_label);
CREATE INDEX idx_coverage_campaign ON coverage(attributed_campaign_id) WHERE attributed_campaign_id IS NOT NULL;
CREATE INDEX idx_monitoring_org ON monitoring_query(organization_id);
```

## Analytics & Relationship Scoring

```sql
CREATE TABLE journalist_relationship_score (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    journalist_id   UUID NOT NULL REFERENCES journalist(id),
    overall_score   NUMERIC(5,2) NOT NULL, -- 0-100
    open_rate       NUMERIC(5,2),          -- percentage
    reply_rate      NUMERIC(5,2),
    coverage_rate   NUMERIC(5,2),          -- pitches that resulted in coverage
    total_pitches   INTEGER NOT NULL DEFAULT 0,
    total_coverage  INTEGER NOT NULL DEFAULT 0,
    last_pitched_at TIMESTAMPTZ,
    last_coverage_at TIMESTAMPTZ,
    score_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, journalist_id)
);

CREATE TABLE campaign_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES campaign(id) ON DELETE CASCADE,
    total_pitches   INTEGER NOT NULL DEFAULT 0,
    total_delivered  INTEGER NOT NULL DEFAULT 0,
    total_opened    INTEGER NOT NULL DEFAULT 0,
    total_clicked   INTEGER NOT NULL DEFAULT 0,
    total_replied   INTEGER NOT NULL DEFAULT 0,
    total_bounced   INTEGER NOT NULL DEFAULT 0,
    total_coverage  INTEGER NOT NULL DEFAULT 0,
    total_reach     BIGINT DEFAULT 0,
    avg_sentiment   NUMERIC(3,2),
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (campaign_id)
);

CREATE INDEX idx_rel_score_org ON journalist_relationship_score(organization_id);
CREATE INDEX idx_rel_score_journalist ON journalist_relationship_score(journalist_id);
CREATE INDEX idx_rel_score_value ON journalist_relationship_score(overall_score DESC);
```

## Wire Distribution & Regulatory

```sql
CREATE TABLE distribution_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    press_release_id UUID NOT NULL REFERENCES press_release(id),
    wire_service    VARCHAR(50) NOT NULL,  -- pr_newswire, business_wire, globenewswire
    distribution_scope TEXT[],             -- regions/countries targeted
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',  -- pending, submitted, distributed, failed
    submitted_at    TIMESTAMPTZ,
    distributed_at  TIMESTAMPTZ,
    wire_reference  VARCHAR(255),          -- external reference from wire service
    cost_cents      INTEGER,
    regulatory_filing BOOLEAN NOT NULL DEFAULT false,  -- SEC EDGAR, SEDAR
    filing_type     VARCHAR(50),           -- 8-K, 6-K, material_change
    filing_reference VARCHAR(255),         -- EDGAR accession number
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dist_pr ON distribution_order(press_release_id);
CREATE INDEX idx_dist_status ON distribution_order(status);
```

## AI Features

```sql
CREATE TABLE ai_prediction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    prediction_type VARCHAR(50) NOT NULL,  -- send_time, placement_likelihood, journalist_match
    entity_type     VARCHAR(50) NOT NULL,  -- pitch, press_release, journalist
    entity_id       UUID NOT NULL,
    prediction_value NUMERIC(5,2),         -- score 0-100 or probability 0-1
    prediction_detail JSONB,               -- model-specific output
    model_version   VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE style_guide (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    outlet_id       UUID NOT NULL REFERENCES outlet(id),
    style_rules     JSONB NOT NULL,        -- {"tone": "formal", "date_format": "AP", ...}
    sample_headlines TEXT[],
    sample_leads    TEXT[],
    last_analyzed_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (outlet_id)
);

CREATE INDEX idx_ai_pred_entity ON ai_prediction(entity_type, entity_id);
CREATE INDEX idx_ai_pred_type ON ai_prediction(prediction_type, created_at DESC);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    user_id         UUID REFERENCES app_user(id),
    action          VARCHAR(50) NOT NULL,  -- create, update, delete, publish, approve, send
    entity_type     VARCHAR(50) NOT NULL,  -- press_release, pitch, campaign, journalist, media_list
    entity_id       UUID NOT NULL,
    changes         JSONB,                 -- {"field": {"old": "...", "new": "..."}}
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organization_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | organization, app_user, role, user_role |
| Journalist Database & CRM | 5 | journalist, outlet, journalist_outlet, journalist_beat, suppression_list |
| Media Lists | 2 | media_list, media_list_member |
| Press Release Authoring | 4 | press_release, press_release_version, press_release_attachment, press_release_tag |
| Newsroom / Owned Media | 2 | newsroom, newsroom_release |
| Campaign & Pitching | 3 | campaign, pitch, pitch_followup |
| Email Engagement | 1 | email_event |
| Monitoring & Coverage | 3 | coverage, coverage_tag, monitoring_query |
| Analytics & Scoring | 2 | journalist_relationship_score, campaign_analytics |
| Distribution & Regulatory | 1 | distribution_order |
| AI Features | 2 | ai_prediction, style_guide |
| Audit | 1 | audit_log |
| **Total** | **30** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation, safe cross-service references, and prevents sequential ID enumeration attacks on public APIs.

2. **Journalist and outlet as separate first-class entities** — journalists move between outlets frequently; the `journalist_outlet` junction table with `start_date`/`end_date` captures employment history without data loss.

3. **Explicit consent tracking on journalist records** — `consent_status`, `opt_out_date`, and `data_source` columns directly support GDPR Article 6(1)(f) legitimate interest basis and CAN-SPAM/CASL suppression requirements.

4. **Press release versioning as separate table** — every edit creates a version record, providing complete content history without bloating the main table. The `press_release` table always holds the current version.

5. **Email events as append-only log** — open/click/reply tracking stored as individual events rather than aggregate counters, enabling time-series analysis and bot-filtering after the fact.

6. **Campaign-level analytics as materialized summary** — `campaign_analytics` is a denormalized summary table computed periodically from pitch and email_event data, avoiding expensive real-time aggregation queries on dashboards.

7. **Journalist relationship scoring as organization-scoped** — each organization has its own relationship scores with journalists, reflecting their specific interaction history rather than a global score.

8. **IPTC subject codes on press releases** — `iptc_subject_codes` array column enables categorization using the industry-standard IPTC NewsCodes vocabulary, facilitating wire distribution and media monitoring integration.

9. **Distribution orders decoupled from press releases** — a single press release can be distributed through multiple wire services with independent status tracking, supporting the multi-wire distribution model used by enterprise comms teams.

10. **Row-level security ready** — all major tables include `organization_id` foreign keys, enabling PostgreSQL RLS policies for multi-tenant data isolation without application-level filtering.
