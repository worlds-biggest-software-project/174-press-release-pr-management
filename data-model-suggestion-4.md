# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Press Release & PR Management · Created: 2026-05-20

## Philosophy

This model uses a graph layer to represent the relationship-heavy aspects of PR management — journalist networks, outlet affiliations, beat coverage overlaps, campaign influence chains, and competitor mentions — while retaining relational tables for transactional CRUD operations (creating press releases, sending pitches, tracking email events). The graph is implemented using PostgreSQL's `ltree` extension for hierarchies and dedicated `graph_node`/`graph_edge` tables for the relationship network, avoiding the need for a separate graph database.

PR management is fundamentally a relationship business. The most valuable data is not the press release itself, but the web of connections: which journalists cover which beats at which outlets, who has covered your competitors, which journalists influence other journalists, how coverage propagates through syndication networks, and which pitches lead to coverage chains. Traditional relational models flatten these relationships into junction tables that become expensive to traverse. A graph-native approach makes relationship traversal — "find journalists who covered competitor X at outlets that syndicate to Y" — a natural query pattern rather than a multi-JOIN nightmare.

This pattern is used by social networks (LinkedIn's relationship graph), knowledge management systems (Wikipedia's entity graph), and fraud detection platforms where relationship traversal is the core value proposition.

**Best for:** Teams building AI-powered journalist targeting, relationship intelligence, influence mapping, coverage propagation analysis, and conflict-of-interest detection.

**Trade-offs:**
- (+) Relationship traversal queries (2-3 hops) are dramatically faster than multi-table JOINs
- (+) Natural model for journalist influence networks and coverage syndication chains
- (+) Graph queries enable novel AI features: "journalists most similar to one who covered us"
- (+) Flexible: new relationship types added as edge types, not schema changes
- (-) Graph query patterns (recursive CTEs, edge traversal) are less familiar to most developers
- (-) More complex data ingestion: every entity needs both a relational record and a graph node
- (-) Graph consistency must be maintained alongside relational tables (dual-write concern)
- (-) PostgreSQL-native graph is less performant than dedicated graph databases (Neo4j) at 10M+ edges

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IPTC ninjs 3.2 | Press release content follows ninjs property naming in relational tables |
| Schema.org NewsArticle | Newsroom releases carry Schema.org structured data; graph connects article entities to people/organizations |
| IPTC NewsCodes | Beat nodes in the graph reference IPTC subject codes as canonical identifiers |
| ISO 3166-1/2 | Geography nodes use ISO 3166 codes; location edges connect entities to jurisdictions |
| ISO 639-1 | Language edges connect content to language nodes using ISO 639-1 |
| GDPR Art. 6(1)(f) | Consent tracked on journalist relational records; graph edges carry privacy-scoped metadata |
| AP Stylebook | Style guide nodes represent outlet-specific conventions; edges link outlets to their style preferences |

---

## Graph Layer

```sql
-- Enable ltree extension for hierarchical paths
CREATE EXTENSION IF NOT EXISTS ltree;

-- Graph nodes represent all entities that participate in relationships
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       VARCHAR(50) NOT NULL,  -- journalist, outlet, beat, organization, press_release,
                                           -- campaign, geography, topic, competitor, influencer
    external_id     UUID,                  -- references the relational table record
    label           VARCHAR(255) NOT NULL,  -- display name
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties vary by node_type:
    -- journalist: {"email": "...", "title": "Senior Reporter", "verified": true}
    -- outlet: {"type": "newspaper", "country": "US", "domain_authority": 85}
    -- beat: {"iptc_code": "04000000", "name": "Economy", "parent_path": "news.economy"}
    -- topic: {"keywords": ["ai", "machine learning"], "trending": true}
    -- competitor: {"name": "CompetitorCo", "website": "..."}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Graph edges represent typed, weighted, directional relationships
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,  -- See edge type catalog below
    weight          NUMERIC(5,2) DEFAULT 1.0,  -- relationship strength 0-100
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties vary by edge_type:
    -- writes_for: {"role": "staff_writer", "since": "2024-01", "is_current": true}
    -- covers_beat: {"confidence": 0.95, "source": "ai_inferred", "article_count": 47}
    -- pitched_to: {"campaign_id": "...", "sent_at": "...", "result": "coverage"}
    -- resulted_in_coverage: {"coverage_id": "...", "sentiment": 0.8, "reach": 1500000}
    -- syndicates_to: {"frequency": "daily", "delay_hours": 2}
    -- influences: {"type": "citation", "frequency": "monthly"}
    -- located_in: {"type": "primary"}
    -- competes_with: {"overlap_beats": ["technology", "ai"]}
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_until     TIMESTAMPTZ,           -- NULL = currently valid
    organization_id UUID,                  -- NULL for global edges (journalist-outlet); set for org-specific (pitched_to)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Critical graph indexes
CREATE INDEX idx_gn_type ON graph_node(node_type);
CREATE INDEX idx_gn_external ON graph_node(external_id) WHERE external_id IS NOT NULL;
CREATE INDEX idx_gn_label ON graph_node(label);
CREATE INDEX idx_gn_properties ON graph_node USING GIN(properties jsonb_path_ops);

CREATE INDEX idx_ge_source ON graph_edge(source_id, edge_type);
CREATE INDEX idx_ge_target ON graph_edge(target_id, edge_type);
CREATE INDEX idx_ge_type ON graph_edge(edge_type);
CREATE INDEX idx_ge_org ON graph_edge(organization_id) WHERE organization_id IS NOT NULL;
CREATE INDEX idx_ge_active ON graph_edge(source_id, target_id) WHERE valid_until IS NULL;
CREATE INDEX idx_ge_weight ON graph_edge(edge_type, weight DESC);

-- Beat hierarchy using ltree
CREATE TABLE beat_hierarchy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID NOT NULL REFERENCES graph_node(id),
    path            LTREE NOT NULL,        -- e.g., 'news.technology.ai.generative_ai'
    iptc_code       VARCHAR(20),           -- IPTC NewsCodes reference
    label           VARCHAR(100) NOT NULL,
    depth           SMALLINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_beat_path ON beat_hierarchy USING GIST(path);
CREATE INDEX idx_beat_iptc ON beat_hierarchy(iptc_code) WHERE iptc_code IS NOT NULL;
```

### Edge Type Catalog

| Edge Type | Source → Target | Description |
|-----------|----------------|-------------|
| `writes_for` | journalist → outlet | Employment/contributor relationship |
| `covers_beat` | journalist → beat | Topics the journalist reports on |
| `pitched_to` | campaign → journalist | A pitch was sent to this journalist |
| `resulted_in_coverage` | pitch → coverage | A pitch produced this coverage article |
| `mentions_competitor` | coverage → competitor | Coverage article mentions a competitor |
| `syndicates_to` | outlet → outlet | Content syndication relationship |
| `influences` | journalist → journalist | Citation/reference relationship between journalists |
| `located_in` | journalist/outlet → geography | Geographic location |
| `covers_company` | journalist → organization | Journalist regularly covers this company |
| `similar_to` | journalist → journalist | AI-computed similarity based on beat overlap |
| `cited_by` | coverage → coverage | One article cites/references another |
| `competes_with` | organization → competitor | Competitive relationship |
| `speaks_language` | journalist → language | Language capabilities |

---

## Relational Layer (Transactional CRUD)

### Identity & Multi-Tenancy

```sql
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    graph_node_id   UUID REFERENCES graph_node(id),  -- link to graph
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    roles           TEXT[] NOT NULL DEFAULT '{viewer}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_user_org ON app_user(organization_id);
```

### Journalist & Outlet (Relational Records)

```sql
CREATE TABLE journalist (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    graph_node_id   UUID REFERENCES graph_node(id),  -- link to graph
    full_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    title           VARCHAR(255),
    country_code    CHAR(2),
    city            VARCHAR(255),
    consent_status  VARCHAR(20) NOT NULL DEFAULT 'implied',
    opt_out_date    TIMESTAMPTZ,
    data_source     VARCHAR(100),
    is_verified     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE outlet (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    graph_node_id   UUID REFERENCES graph_node(id),  -- link to graph
    name            VARCHAR(255) NOT NULL,
    outlet_type     VARCHAR(50) NOT NULL,
    website_url     VARCHAR(500),
    country_code    CHAR(2),
    language_code   CHAR(2) DEFAULT 'en',
    domain_authority SMALLINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_journalist_email ON journalist(email);
CREATE INDEX idx_journalist_graph ON journalist(graph_node_id);
CREATE INDEX idx_outlet_graph ON outlet(graph_node_id);
```

### Press Release & Newsroom

```sql
CREATE TABLE press_release (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    graph_node_id   UUID REFERENCES graph_node(id),  -- link to graph for relationship tracking
    headline        VARCHAR(500) NOT NULL,
    subheadline     VARCHAR(500),
    body_html       TEXT NOT NULL,
    body_text       TEXT NOT NULL,
    summary         TEXT,
    language_code   CHAR(2) NOT NULL DEFAULT 'en',
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    author_id       UUID NOT NULL REFERENCES app_user(id),
    iptc_subject_codes TEXT[],
    embargo_until   TIMESTAMPTZ,
    published_at    TIMESTAMPTZ,
    word_count      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE newsroom (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    custom_domain   VARCHAR(255),
    config          JSONB DEFAULT '{}',
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_pr_org ON press_release(organization_id);
CREATE INDEX idx_pr_status ON press_release(status);
CREATE INDEX idx_pr_graph ON press_release(graph_node_id);
```

### Campaign & Pitching

```sql
CREATE TABLE campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    name            VARCHAR(255) NOT NULL,
    press_release_id UUID REFERENCES press_release(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
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
    sent_at         TIMESTAMPTZ,
    message_id      VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE email_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pitch_id        UUID NOT NULL REFERENCES pitch(id) ON DELETE CASCADE,
    event_type      VARCHAR(30) NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    properties      JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_org ON campaign(organization_id);
CREATE INDEX idx_pitch_campaign ON pitch(campaign_id);
CREATE INDEX idx_pitch_journalist ON pitch(journalist_id);
CREATE INDEX idx_email_event_pitch ON email_event(pitch_id);
```

### Coverage & Monitoring

```sql
CREATE TABLE coverage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    headline        VARCHAR(500) NOT NULL,
    url             VARCHAR(500),
    outlet_id       UUID REFERENCES outlet(id),
    journalist_id   UUID REFERENCES journalist(id),
    published_at    TIMESTAMPTZ,
    discovered_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    coverage_type   VARCHAR(50) NOT NULL,
    sentiment_score NUMERIC(3,2),
    sentiment_label VARCHAR(20),
    reach_estimate  BIGINT,
    content_snippet TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE monitoring_query (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    name            VARCHAR(255) NOT NULL,
    query_string    TEXT NOT NULL,
    sources         TEXT[],
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coverage_org ON coverage(organization_id);
CREATE INDEX idx_coverage_graph ON coverage(graph_node_id);
CREATE INDEX idx_coverage_published ON coverage(published_at DESC);
```

### Distribution

```sql
CREATE TABLE distribution_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    press_release_id UUID NOT NULL REFERENCES press_release(id),
    wire_service    VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    details         JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dist_pr ON distribution_order(press_release_id);
```

### Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    user_id         UUID REFERENCES app_user(id),
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organization_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Graph Query Examples

### Find journalists 2 hops from a target journalist (influence network)

```sql
-- Find journalists who are influenced by or influence journalists
-- that the target journalist also influences (2-hop network)
WITH RECURSIVE influence_network AS (
    -- Start: direct connections of target journalist
    SELECT
        e.target_id AS node_id,
        e.weight,
        1 AS depth,
        ARRAY[e.source_id, e.target_id] AS path
    FROM graph_edge e
    JOIN graph_node src ON src.id = e.source_id
    WHERE src.external_id = '{{journalist_id}}'
      AND e.edge_type = 'influences'
      AND e.valid_until IS NULL

    UNION ALL

    -- Recursive: next hop
    SELECT
        e2.target_id,
        e2.weight * 0.5,  -- decay weight by hop
        n.depth + 1,
        n.path || e2.target_id
    FROM influence_network n
    JOIN graph_edge e2 ON e2.source_id = n.node_id
    WHERE e2.edge_type = 'influences'
      AND e2.valid_until IS NULL
      AND n.depth < 2
      AND NOT (e2.target_id = ANY(n.path))  -- prevent cycles
)
SELECT
    gn.label AS journalist_name,
    gn.properties->>'email' AS email,
    MAX(i.weight) AS influence_weight,
    MIN(i.depth) AS hops
FROM influence_network i
JOIN graph_node gn ON gn.id = i.node_id
GROUP BY gn.id, gn.label, gn.properties
ORDER BY MAX(i.weight) DESC
LIMIT 20;
```

### Find journalists who cover beats similar to a target journalist

```sql
-- Journalists with overlapping beat coverage (shared beat nodes)
SELECT
    j_node.label AS journalist_name,
    j_node.properties->>'email' AS email,
    COUNT(DISTINCT shared_beat.id) AS shared_beats,
    ARRAY_AGG(DISTINCT beat_node.label) AS common_beats,
    AVG(e2.weight) AS avg_beat_strength
FROM graph_edge e1
JOIN graph_node src ON src.id = e1.source_id AND src.external_id = '{{journalist_id}}'
JOIN graph_node beat_node ON beat_node.id = e1.target_id
JOIN graph_edge e2 ON e2.target_id = beat_node.id AND e2.edge_type = 'covers_beat'
JOIN graph_node j_node ON j_node.id = e2.source_id AND j_node.id != src.id
LEFT JOIN graph_node shared_beat ON shared_beat.id = beat_node.id
WHERE e1.edge_type = 'covers_beat'
  AND e1.valid_until IS NULL
  AND e2.valid_until IS NULL
GROUP BY j_node.id, j_node.label, j_node.properties
HAVING COUNT(DISTINCT shared_beat.id) >= 2
ORDER BY COUNT(DISTINCT shared_beat.id) DESC, AVG(e2.weight) DESC;
```

### Coverage propagation: trace how a story spread through syndication

```sql
-- Starting from an original coverage article, follow syndication edges
WITH RECURSIVE syndication_chain AS (
    SELECT
        gn.id AS node_id,
        gn.label AS outlet_name,
        c.published_at,
        0 AS hop,
        ARRAY[gn.id] AS path
    FROM coverage c
    JOIN graph_node gn ON gn.id = c.graph_node_id
    WHERE c.id = '{{original_coverage_id}}'

    UNION ALL

    SELECT
        cited.id,
        cited.label,
        c2.published_at,
        s.hop + 1,
        s.path || cited.id
    FROM syndication_chain s
    JOIN graph_edge e ON e.source_id = s.node_id AND e.edge_type = 'cited_by'
    JOIN graph_node cited ON cited.id = e.target_id
    JOIN coverage c2 ON c2.graph_node_id = cited.id
    WHERE s.hop < 5
      AND NOT (cited.id = ANY(s.path))
)
SELECT outlet_name, published_at, hop
FROM syndication_chain
ORDER BY hop, published_at;
```

### Find best journalists for a topic using beat hierarchy (ltree)

```sql
-- Find journalists covering any sub-beat of "technology"
SELECT
    j.full_name, j.email,
    bh.label AS specific_beat,
    bh.path AS beat_path,
    e.weight AS coverage_strength
FROM beat_hierarchy bh
JOIN graph_edge e ON e.target_id = bh.node_id AND e.edge_type = 'covers_beat'
JOIN graph_node jn ON jn.id = e.source_id AND jn.node_type = 'journalist'
JOIN journalist j ON j.graph_node_id = jn.id
WHERE bh.path <@ 'news.technology'  -- all sub-beats under technology
  AND e.valid_until IS NULL
  AND j.consent_status != 'opted_out'
ORDER BY e.weight DESC, bh.depth ASC
LIMIT 50;
```

### Conflict of interest detection

```sql
-- Find journalists who cover both the target company AND a competitor
SELECT
    j_node.label AS journalist_name,
    comp_node.label AS competitor_covered,
    e_comp.properties->>'article_count' AS competitor_articles
FROM graph_edge e_org
JOIN graph_node j_node ON j_node.id = e_org.source_id AND j_node.node_type = 'journalist'
JOIN graph_edge e_comp ON e_comp.source_id = j_node.id AND e_comp.edge_type = 'covers_company'
JOIN graph_node comp_node ON comp_node.id = e_comp.target_id AND comp_node.node_type = 'competitor'
WHERE e_org.target_id = (
    SELECT graph_node_id FROM organization WHERE id = '{{org_id}}'
)
AND e_org.edge_type = 'covers_company'
AND e_org.valid_until IS NULL
AND e_comp.valid_until IS NULL;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Infrastructure | 3 | graph_node, graph_edge, beat_hierarchy |
| Identity & Multi-Tenancy | 2 | organization, app_user |
| Journalist & Outlet | 2 | journalist, outlet (with graph_node_id links) |
| Press Release & Newsroom | 2 | press_release, newsroom |
| Campaign & Pitching | 3 | campaign, pitch, email_event |
| Coverage & Monitoring | 2 | coverage, monitoring_query |
| Distribution | 1 | distribution_order |
| Audit | 1 | audit_log |
| **Total** | **16** | 3 graph tables + 13 relational tables |

---

## Key Design Decisions

1. **PostgreSQL-native graph instead of separate graph database** — using `graph_node`/`graph_edge` tables with recursive CTEs keeps the entire system in one database, avoiding dual-write synchronization problems. For PR management scale (thousands to low millions of edges), PostgreSQL handles graph traversal efficiently.

2. **Dual identity: relational records linked to graph nodes via `graph_node_id`** — every major entity (journalist, outlet, campaign, coverage) has both a relational record for CRUD operations and a graph node for relationship queries. The `graph_node_id` foreign key bridges the two worlds.

3. **Typed, weighted, temporal edges** — `edge_type` categorizes relationships, `weight` quantifies strength (enabling "strongest connections" queries), and `valid_from`/`valid_until` captures relationship temporality (journalist moved outlets, beat changed).

4. **Beat hierarchy using ltree** — IPTC NewsCodes form a hierarchy (Economy > Markets > Stock Markets). The `ltree` extension enables efficient "find all sub-beat journalists" queries using `<@` (is descendant of) and `@>` (is ancestor of) operators.

5. **Organization-scoped edges for private data** — global edges (journalist-outlet relationships) have `organization_id = NULL`. Organization-specific edges (pitched_to, resulted_in_coverage) carry the org ID, enabling multi-tenant graph queries without leaking private pitch data.

6. **Influence and similarity edges as AI outputs** — `influences` and `similar_to` edge types are computed by AI models based on citation patterns, beat overlaps, and writing style similarity. These enable the "journalists most like X" queries that power intelligent targeting.

7. **Coverage propagation tracking via cited_by edges** — when coverage articles reference or syndicate from other articles, `cited_by` edges trace the propagation chain. This enables "how did our story spread?" analysis that no current PR platform provides.

8. **Graph properties as JSONB** — node and edge properties use JSONB to accommodate type-specific metadata without per-type tables. A journalist node carries different properties than a beat node, and a `writes_for` edge carries different properties than a `covers_beat` edge.

9. **Conflict-of-interest detection as graph query** — identifying journalists who cover both the target company and its competitors is a natural graph traversal. In a relational-only model, this requires multiple JOINs across junction tables; in the graph model, it is a two-hop query.

10. **Scalability path to dedicated graph database** — if the graph grows beyond PostgreSQL's comfortable range (10M+ edges), the `graph_node`/`graph_edge` tables can be migrated to Neo4j or Amazon Neptune while the relational tables remain in PostgreSQL. The `graph_node_id` foreign keys become cross-database references resolved at the application layer.
