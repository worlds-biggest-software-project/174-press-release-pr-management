# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Press Release & PR Management · Created: 2026-05-20

## Philosophy

This model uses PostgreSQL relational tables for core structural data (identities, relationships, foreign keys) while leveraging JSONB columns for variable, domain-specific, and rapidly evolving data. The "hybrid" approach recognizes that PR management has both well-defined entities (press releases, campaigns, pitches) and highly variable metadata (journalist preferences by region, outlet-specific style rules, jurisdiction-dependent compliance fields, AI model outputs).

This pattern is widely used by modern SaaS platforms — Stripe stores payment metadata as JSONB alongside relational core fields, Shopify uses JSON columns for variant product attributes, and HubSpot's CRM stores custom properties as structured JSON. The key insight is that some data (who, what, when, relationships) benefits from relational constraints, while other data (preferences, settings, AI outputs, regional variations) benefits from schema flexibility.

For PR management specifically, this matters because journalist metadata varies wildly: a US journalist may have a Twitter/X handle and AP Style preferences, while a European journalist has GDPR consent documentation and multilingual bylines. Rather than adding nullable columns for every possible variant, JSONB columns absorb this variation cleanly.

**Best for:** Rapid MVP development, multi-region deployments with jurisdiction-specific fields, teams that need schema flexibility without sacrificing relational integrity for core entities.

**Trade-offs:**
- (+) Fewer tables than fully normalized (15-20 vs. 30+); simpler to develop against
- (+) JSONB columns absorb jurisdiction-specific, outlet-specific, and AI-generated metadata without migrations
- (+) PostgreSQL JSONB supports indexing (GIN), containment queries, and partial updates
- (+) New fields can be added instantly without schema migration or downtime
- (-) JSONB fields lack column-level constraints; validation must happen in application code
- (-) Complex JSONB queries can be slower than indexed relational columns for high-cardinality filtering
- (-) Reporting tools (Metabase, Tableau) handle JSONB less natively than flat columns
- (-) Schema documentation must be maintained separately since JSONB structure is not self-documenting in DDL

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IPTC ninjs 3.2 | Press release `content` JSONB column follows ninjs property naming (headline, byline, body, language) |
| Schema.org NewsArticle | `structured_data` JSONB column on press releases stores Schema.org-compliant markup |
| IPTC NewsCodes | `metadata.iptc_subjects` array within press release JSONB references IPTC controlled vocabularies |
| ISO 3166-1/2 | `location.country_code` and `location.region_code` in journalist/outlet JSONB use ISO 3166 |
| ISO 639-1 | `language_code` fields use ISO 639-1; multi-language content stored as JSONB with language keys |
| GDPR Art. 6(1)(f) | `compliance` JSONB on journalist records stores consent history, opt-out records, data source |
| CAN-SPAM / CASL | `compliance.suppression` within contact JSONB tracks jurisdiction-specific opt-out state |
| AP Stylebook | `style_rules` JSONB on outlet records captures AP Style deviations and outlet-specific conventions |
| RFC 6376 (DKIM) | `delivery_metadata` JSONB on pitch records stores DKIM status, SPF results, deliverability signals |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "branding": {"logo_url": "...", "primary_color": "#1a73e8"},
    --   "defaults": {"language": "en", "timezone": "America/New_York"},
    --   "integrations": {"slack_webhook": "...", "salesforce_org_id": "..."},
    --   "compliance": {"gdpr_enabled": true, "default_consent_basis": "legitimate_interest"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    roles           TEXT[] NOT NULL DEFAULT '{viewer}',  -- admin, editor, viewer, pitch_sender
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "avatar_url": "...",
    --   "timezone": "America/New_York",
    --   "notification_prefs": {"email": true, "slack": true, "in_app": true},
    --   "auth": {"provider": "google", "subject": "...", "last_login": "..."}
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_user_org ON app_user(organization_id);
```

## Journalist Database & CRM

```sql
CREATE TABLE outlet (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    outlet_type     VARCHAR(50) NOT NULL,     -- newspaper, magazine, blog, tv, radio, podcast, newsletter
    website_url     VARCHAR(500),
    country_code    CHAR(2),                  -- ISO 3166-1
    language_code   CHAR(2) DEFAULT 'en',     -- ISO 639-1
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "circulation": 250000,
    --   "domain_authority": 85,
    --   "social_profiles": {"twitter": "@nytimes", "linkedin": "..."},
    --   "style_rules": {
    --     "base": "ap_style",
    --     "date_format": "Month DD, YYYY",
    --     "title_case": true,
    --     "oxford_comma": false,
    --     "tone": "formal",
    --     "sample_headlines": ["...", "..."],
    --     "sample_leads": ["...", "..."]
    --   },
    --   "editorial_calendar": {"deadlines": {"daily": "14:00 ET", "sunday": "Wed 12:00 ET"}},
    --   "podcast": {"rss_feed": "...", "episode_count": 450, "avg_listeners": 50000}
    -- }
    is_verified     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE journalist (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    full_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    title           VARCHAR(255),
    country_code    CHAR(2),                  -- ISO 3166-1
    city            VARCHAR(255),
    beats           TEXT[] NOT NULL DEFAULT '{}',  -- ["technology", "ai", "startups"]
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "phone": "+1-555-0123",
    --   "bio": "Senior tech reporter covering...",
    --   "social": {"twitter": "@handle", "linkedin": "...", "mastodon": "..."},
    --   "preferences": {
    --     "contact_method": "email",
    --     "contact_time": "mornings EST",
    --     "pitch_format": "brief_summary",
    --     "no_attachments": true
    --   },
    --   "languages": ["en", "fr"],
    --   "specializations": ["enterprise_saas", "ai_ethics", "open_source"]
    -- }
    compliance      JSONB NOT NULL DEFAULT '{}',
    -- compliance example:
    -- {
    --   "consent_status": "implied",
    --   "consent_basis": "legitimate_interest",
    --   "data_source": "public_profile",
    --   "data_source_url": "https://...",
    --   "opt_out_date": null,
    --   "gdpr_jurisdiction": true,
    --   "casl_jurisdiction": false,
    --   "suppressed_orgs": ["uuid-..."],
    --   "consent_history": [
    --     {"action": "imported", "date": "2026-01-15", "source": "manual"},
    --     {"action": "pitched", "date": "2026-03-01", "basis": "legitimate_interest"}
    --   ]
    -- }
    is_verified     BOOLEAN NOT NULL DEFAULT false,
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE journalist_outlet (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journalist_id   UUID NOT NULL REFERENCES journalist(id) ON DELETE CASCADE,
    outlet_id       UUID NOT NULL REFERENCES outlet(id) ON DELETE CASCADE,
    role_at_outlet  VARCHAR(100),
    is_current      BOOLEAN NOT NULL DEFAULT true,
    start_date      DATE,
    end_date        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (journalist_id, outlet_id, start_date)
);

CREATE INDEX idx_journalist_email ON journalist(email);
CREATE INDEX idx_journalist_country ON journalist(country_code);
CREATE INDEX idx_journalist_beats ON journalist USING GIN(beats);
CREATE INDEX idx_journalist_compliance ON journalist USING GIN(compliance jsonb_path_ops);
CREATE INDEX idx_outlet_type ON outlet(outlet_type);
CREATE INDEX idx_outlet_country ON outlet(country_code);
CREATE INDEX idx_outlet_properties ON outlet USING GIN(properties jsonb_path_ops);
CREATE INDEX idx_jo_current ON journalist_outlet(journalist_id) WHERE is_current = true;
```

## Media Lists

```sql
CREATE TABLE media_list (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    list_type       VARCHAR(20) NOT NULL DEFAULT 'static',  -- static, dynamic
    filter_criteria JSONB,
    -- dynamic list filter example:
    -- {
    --   "beats": ["technology", "ai"],
    --   "countries": ["US", "GB", "CA"],
    --   "outlet_types": ["newspaper", "blog"],
    --   "min_domain_authority": 50,
    --   "exclude_opted_out": true
    -- }
    created_by      UUID NOT NULL REFERENCES app_user(id),
    journalist_count INTEGER NOT NULL DEFAULT 0,  -- denormalized
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE media_list_member (
    media_list_id   UUID NOT NULL REFERENCES media_list(id) ON DELETE CASCADE,
    journalist_id   UUID NOT NULL REFERENCES journalist(id) ON DELETE CASCADE,
    added_by        UUID REFERENCES app_user(id),
    notes           TEXT,
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (media_list_id, journalist_id)
);

CREATE INDEX idx_media_list_org ON media_list(organization_id);
```

## Press Release Authoring & Newsroom

```sql
CREATE TABLE press_release (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    headline        VARCHAR(500) NOT NULL,
    subheadline     VARCHAR(500),
    body_html       TEXT NOT NULL,
    body_text       TEXT NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    language_code   CHAR(2) NOT NULL DEFAULT 'en',
    author_id       UUID NOT NULL REFERENCES app_user(id),
    content_meta    JSONB NOT NULL DEFAULT '{}',
    -- content_meta example:
    -- {
    --   "summary": "Boilerplate paragraph...",
    --   "seo_keywords": ["product launch", "ai"],
    --   "iptc_subject_codes": ["04000000", "04003000"],
    --   "schema_org": {
    --     "@type": "NewsArticle",
    --     "datePublished": "2026-06-01T09:00:00Z",
    --     "publisher": {"@type": "Organization", "name": "Acme Corp"}
    --   },
    --   "contacts": [
    --     {"name": "Jane Doe", "title": "VP Comms", "email": "jane@acme.com", "phone": "+1-555-0100"}
    --   ],
    --   "quotes": [
    --     {"speaker": "CEO Name", "title": "CEO", "text": "We are excited..."}
    --   ],
    --   "multimedia": [
    --     {"type": "image", "url": "...", "alt": "Product screenshot", "caption": "..."},
    --     {"type": "video", "url": "...", "embed_code": "..."}
    --   ],
    --   "translations": {
    --     "fr": {"headline": "...", "body_html": "..."},
    --     "de": {"headline": "...", "body_html": "..."}
    --   },
    --   "style_guide_applied": "ap_style",
    --   "ai_quality_score": 87.5,
    --   "readability_score": 65.2
    -- }
    approval        JSONB NOT NULL DEFAULT '{}',
    -- approval example:
    -- {
    --   "approved_by": "uuid-...",
    --   "approved_at": "2026-05-15T14:30:00Z",
    --   "approval_notes": "Approved for distribution",
    --   "legal_review": {"reviewed_by": "uuid-...", "reviewed_at": "...", "status": "cleared"}
    -- }
    embargo_until   TIMESTAMPTZ,
    published_at    TIMESTAMPTZ,
    word_count      INTEGER,
    version_count   INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE press_release_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    press_release_id UUID NOT NULL REFERENCES press_release(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    headline        VARCHAR(500) NOT NULL,
    body_html       TEXT NOT NULL,
    content_meta    JSONB NOT NULL DEFAULT '{}',
    changed_by      UUID NOT NULL REFERENCES app_user(id),
    change_summary  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (press_release_id, version_number)
);

CREATE TABLE newsroom (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    custom_domain   VARCHAR(255),
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "branding": {"logo_url": "...", "colors": {"primary": "#1a73e8"}},
    --   "seo": {"title": "...", "description": "...", "og_image": "..."},
    --   "social_feeds": [{"platform": "twitter", "handle": "@acme"}],
    --   "layout": "grid",
    --   "categories": ["product", "leadership", "financial"],
    --   "contact_info": {"email": "press@acme.com", "phone": "..."}
    -- }
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_pr_org ON press_release(organization_id);
CREATE INDEX idx_pr_status ON press_release(status);
CREATE INDEX idx_pr_published ON press_release(published_at DESC) WHERE published_at IS NOT NULL;
CREATE INDEX idx_pr_embargo ON press_release(embargo_until) WHERE embargo_until IS NOT NULL;
CREATE INDEX idx_pr_content_meta ON press_release USING GIN(content_meta jsonb_path_ops);
```

## Campaigns, Pitching & Engagement

```sql
CREATE TABLE campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    press_release_id UUID REFERENCES press_release(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "send_schedule": {"type": "immediate" | "scheduled" | "ai_optimized"},
    --   "scheduled_at": "2026-06-01T09:00:00Z",
    --   "followup_rules": [
    --     {"delay_hours": 72, "subject": "Following up on...", "only_if": "no_open"},
    --     {"delay_hours": 168, "subject": "Quick check-in...", "only_if": "no_reply"}
    --   ],
    --   "personalization": {"enabled": true, "model": "outlet_style_match"},
    --   "media_lists": ["uuid-...", "uuid-..."],
    --   "ab_test": {"enabled": false, "variants": []}
    -- }
    stats           JSONB NOT NULL DEFAULT '{}',
    -- stats (updated incrementally):
    -- {
    --   "total_pitches": 150,
    --   "delivered": 145, "bounced": 5,
    --   "opened": 89, "unique_opens": 72,
    --   "clicked": 23, "replied": 12,
    --   "coverage_count": 8, "total_reach": 4500000,
    --   "avg_sentiment": 0.65
    -- }
    created_by      UUID NOT NULL REFERENCES app_user(id),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
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
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    delivery        JSONB NOT NULL DEFAULT '{}',
    -- delivery example:
    -- {
    --   "sent_at": "2026-06-01T09:05:00Z",
    --   "delivered_at": "2026-06-01T09:05:02Z",
    --   "message_id": "<abc@mail.example.com>",
    --   "dkim_status": "pass",
    --   "spf_status": "pass",
    --   "bounce_type": null,
    --   "unsubscribe_token": "abc123"
    -- }
    engagement      JSONB NOT NULL DEFAULT '{}',
    -- engagement (updated as events arrive):
    -- {
    --   "opens": [
    --     {"at": "2026-06-01T10:15:00Z", "ip": "203.0.113.1", "ua": "...", "is_bot": false}
    --   ],
    --   "clicks": [
    --     {"at": "2026-06-01T10:16:00Z", "url": "https://newsroom.acme.com/launch"}
    --   ],
    --   "reply": {"at": "2026-06-01T14:30:00Z", "snippet": "Thanks, I'd like to schedule..."},
    --   "open_count": 3,
    --   "click_count": 1,
    --   "has_replied": true,
    --   "last_event_at": "2026-06-01T14:30:00Z"
    -- }
    personalization JSONB DEFAULT '{}',
    -- personalization example:
    -- {
    --   "ai_notes": "Journalist recently covered competitor X; angle on differentiation",
    --   "outlet_style_match": true,
    --   "predicted_open_probability": 0.72,
    --   "predicted_coverage_probability": 0.35,
    --   "optimal_send_time": "2026-06-01T09:00:00Z"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_org ON campaign(organization_id);
CREATE INDEX idx_campaign_status ON campaign(status);
CREATE INDEX idx_pitch_campaign ON pitch(campaign_id);
CREATE INDEX idx_pitch_journalist ON pitch(journalist_id);
CREATE INDEX idx_pitch_status ON pitch(status);
CREATE INDEX idx_pitch_engagement ON pitch USING GIN(engagement jsonb_path_ops);
```

## Coverage & Monitoring

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
    coverage_type   VARCHAR(50) NOT NULL,
    tags            TEXT[] DEFAULT '{}',
    analysis        JSONB NOT NULL DEFAULT '{}',
    -- analysis example:
    -- {
    --   "sentiment_score": 0.75,
    --   "sentiment_label": "positive",
    --   "reach_estimate": 1500000,
    --   "share_count": 234,
    --   "content_snippet": "Acme Corp today announced...",
    --   "key_messages_hit": ["product_launch", "ai_integration"],
    --   "competitor_mentions": ["CompetitorA"],
    --   "spokesperson_quoted": true,
    --   "attribution": {
    --     "campaign_id": "uuid-...",
    --     "pitch_id": "uuid-...",
    --     "confidence": 0.92
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE monitoring_query (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
    -- config example:
    -- {
    --   "query_string": "\"Acme Corp\" OR acmecorp",
    --   "sources": ["news", "social", "podcast"],
    --   "languages": ["en", "fr"],
    --   "exclude_domains": ["reddit.com"],
    --   "alerts": {
    --     "enabled": true,
    --     "channels": ["email", "slack"],
    --     "frequency": "real_time",
    --     "sentiment_threshold": -0.5
    --   }
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coverage_org ON coverage(organization_id);
CREATE INDEX idx_coverage_published ON coverage(published_at DESC);
CREATE INDEX idx_coverage_analysis ON coverage USING GIN(analysis jsonb_path_ops);
CREATE INDEX idx_coverage_tags ON coverage USING GIN(tags);
CREATE INDEX idx_monitoring_org ON monitoring_query(organization_id);
```

## Distribution & Regulatory

```sql
CREATE TABLE distribution_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    press_release_id UUID NOT NULL REFERENCES press_release(id),
    wire_service    VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "distribution_scope": ["US", "GB", "CA"],
    --   "submitted_at": "2026-06-01T08:55:00Z",
    --   "distributed_at": "2026-06-01T09:00:00Z",
    --   "wire_reference": "PRN-20260601-ABC",
    --   "cost_cents": 75000,
    --   "regulatory": {
    --     "is_filing": true,
    --     "filing_type": "8-K",
    --     "edgar_accession": "0001234567-26-000123",
    --     "sedar_reference": null
    --   },
    --   "syndication_urls": ["https://finance.yahoo.com/...", "https://..."]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dist_pr ON distribution_order(press_release_id);
CREATE INDEX idx_dist_status ON distribution_order(status);
```

## Relationship Scoring & AI

```sql
CREATE TABLE journalist_score (
    organization_id UUID NOT NULL REFERENCES organization(id),
    journalist_id   UUID NOT NULL REFERENCES journalist(id),
    scores          JSONB NOT NULL,
    -- scores example:
    -- {
    --   "overall": 78.5,
    --   "open_rate": 0.68,
    --   "reply_rate": 0.22,
    --   "coverage_rate": 0.15,
    --   "total_pitches": 25,
    --   "total_coverage": 4,
    --   "trend": "improving",
    --   "last_pitched_at": "2026-05-01T...",
    --   "last_coverage_at": "2026-04-15T...",
    --   "best_send_time": "Tuesday 09:00 ET",
    --   "preferred_topics": ["ai", "enterprise_saas"],
    --   "response_time_avg_hours": 6.5
    -- }
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organization_id, journalist_id)
);

CREATE INDEX idx_jscore_overall ON journalist_score((scores->>'overall') DESC);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    user_id         UUID REFERENCES app_user(id),
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "changes": {"status": {"old": "draft", "new": "approved"}},
    --   "ip_address": "203.0.113.1",
    --   "user_agent": "..."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organization_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Example JSONB Queries

### Find journalists who prefer email contact and cover AI in the US

```sql
SELECT id, full_name, email, beats
FROM journalist
WHERE country_code = 'US'
  AND 'ai' = ANY(beats)
  AND profile @> '{"preferences": {"contact_method": "email"}}';
```

### Find outlets with domain authority above 70 that follow AP Style

```sql
SELECT id, name, outlet_type, properties->>'domain_authority' AS da
FROM outlet
WHERE (properties->>'domain_authority')::int > 70
  AND properties @> '{"style_rules": {"base": "ap_style"}}';
```

### Find pitches with high AI-predicted coverage probability

```sql
SELECT p.id, p.subject_line, j.full_name,
       p.personalization->>'predicted_coverage_probability' AS coverage_prob
FROM pitch p
JOIN journalist j ON j.id = p.journalist_id
WHERE p.campaign_id = '{{campaign_id}}'
  AND (p.personalization->>'predicted_coverage_probability')::numeric > 0.5
ORDER BY (p.personalization->>'predicted_coverage_probability')::numeric DESC;
```

### Find coverage with positive sentiment attributed to a campaign

```sql
SELECT headline, url, analysis->>'sentiment_label' AS sentiment,
       (analysis->>'reach_estimate')::bigint AS reach
FROM coverage
WHERE organization_id = '{{org_id}}'
  AND analysis @> '{"attribution": {"campaign_id": "{{campaign_id}}"}}'
  AND (analysis->>'sentiment_score')::numeric > 0.3
ORDER BY (analysis->>'reach_estimate')::bigint DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | organization, app_user (roles as array, not junction table) |
| Journalist Database & CRM | 3 | journalist, outlet, journalist_outlet |
| Media Lists | 2 | media_list, media_list_member |
| Press Release & Newsroom | 3 | press_release, press_release_version, newsroom |
| Campaigns & Pitching | 2 | campaign, pitch (engagement as JSONB, not separate events table) |
| Coverage & Monitoring | 2 | coverage, monitoring_query |
| Distribution | 1 | distribution_order |
| Scoring & AI | 1 | journalist_score (all scores in JSONB) |
| Audit | 1 | audit_log |
| **Total** | **17** | ~45% fewer tables than normalized model |

---

## Key Design Decisions

1. **JSONB for variable metadata, relational for identity and relationships** — core identifiers (`id`, `email`, `organization_id`), foreign keys, and high-cardinality filter columns (`status`, `country_code`, `beats`) remain relational. Variable, evolving, or domain-specific data goes into typed JSONB columns.

2. **Engagement tracking in pitch JSONB instead of separate events table** — individual opens/clicks are stored as arrays within `pitch.engagement` JSONB. This trades event-level queryability for simpler schema and fewer JOINs. For high-volume analytics, a separate time-series store can be added later.

3. **Roles as PostgreSQL array instead of junction table** — `app_user.roles` is a `TEXT[]` column rather than a separate `user_role` table. For typical PR team sizes (2-50 users per org with 3-5 roles), this eliminates a table and simplifies queries. Fine-grained permissions can live in the `profile` JSONB.

4. **JSONB GIN indexes for containment queries** — `jsonb_path_ops` GIN indexes on compliance, properties, engagement, and analysis columns enable efficient `@>` containment queries without extracting every field into separate columns.

5. **Multi-language press releases in JSONB** — translations stored as `content_meta.translations` JSONB map (keyed by ISO 639-1 code) rather than a separate `press_release_translation` table. Simpler for teams supporting 2-5 languages; a separate table would be better for 20+ languages.

6. **Campaign stats as denormalized JSONB** — `campaign.stats` stores running totals that are updated incrementally as pitch events occur. This avoids expensive aggregation queries on dashboards while the JSONB structure allows adding new metrics without migrations.

7. **Compliance JSONB on journalist records** — GDPR consent history, CAN-SPAM suppression, CASL jurisdiction flags, and opt-out tracking are stored as structured JSONB. This accommodates jurisdiction-specific compliance fields that vary by country without adding nullable columns for each regulation.

8. **AI predictions embedded in pitch records** — predicted open probability, coverage probability, and optimal send time live in `pitch.personalization` JSONB rather than a separate predictions table. This keeps AI outputs co-located with the entities they describe.

9. **Outlet style rules as JSONB** — outlet-specific writing conventions (AP Style deviations, tone, headline patterns, editorial deadlines) are captured in `outlet.properties` JSONB. This is inherently variable data that differs per outlet and evolves as editorial standards change.

10. **17 tables vs. 30 in normalized model** — the JSONB approach reduces table count by ~45% while maintaining relational integrity for core entities. This accelerates MVP development and reduces migration burden, at the cost of application-level JSONB validation.
