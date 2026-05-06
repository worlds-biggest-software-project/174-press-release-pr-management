# Standards & API Reference

> Project: Press Release & PR Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022** — Information security management; governs access controls, audit logging, and data handling for PR management platforms managing confidential embargo communications, unreleased product announcements, and sensitive media contact databases. URL: https://www.iso.org/standard/82875.html

- **ISO/IEC 27018:2019 — Protection of PII in Public Clouds** — Governs processing of journalist and media contact personal data (email addresses, phone numbers, beat information) in cloud-hosted PR media databases; relevant given GDPR journalist exemption ambiguity. URL: https://www.iso.org/standard/76559.html

### W3C & IETF Standards

- **AP Stylebook (Associated Press Style)** — The de facto standard for press release writing in US markets; governs grammar, punctuation, capitalization, numbers, abbreviations, and media-specific terminology; AP Style is the mandatory format for press releases submitted to PR Newswire, Business Wire, and all major wire services. URL: https://www.apstylebook.com/

- **RFC 4287 — Atom Syndication Format** — IETF standard for web feed syndication; PR Newswire, Business Wire, GlobeNewswire, and AccessWire publish full-text XML/HTML press release feeds via RSS 2.0 and Atom; enables automated ingestion of newswire content into media monitoring platforms and news aggregators. URL: https://datatracker.ietf.org/doc/html/rfc4287

- **RSS 2.0** — PR Newswire offers RSS news feeds for its main wire and topic-specific feeds; the standard machine-readable format for press release distribution monitoring. URL: https://www.rssboard.org/rss-specification

- **RFC 5321 — SMTP** — Foundation for email pitch delivery; PR management platforms send journalist pitches and press releases via SMTP with SPF/DKIM/DMARC authentication required for deliverability. URL: https://datatracker.ietf.org/doc/html/rfc5321

- **RFC 7208 — SPF** — Required DNS authentication for all PR pitch sending domains; enforced by Gmail and Outlook for bulk senders from 2024/2025. URL: https://datatracker.ietf.org/doc/html/rfc7208

- **RFC 6376 — DKIM** — Required cryptographic email signing for PR pitch deliverability to journalist inboxes; mandated by major inbox providers. URL: https://datatracker.ietf.org/doc/html/rfc6376

- **RFC 6749 — OAuth 2.0** — Authorization framework used by PR management platform APIs (Meltwater, Cision) for third-party integrations, CRM connections, and business intelligence tool access. URL: https://datatracker.ietf.org/doc/html/rfc6749

### Data Model & API Specifications

- **OpenAPI 3.1** — Meltwater publishes an OpenAPI specification for the Meltwater Developer Portal API; Cision offers a REST API for BI tool integration (Tableau, PowerBI, Domo); enables programmatic media monitoring data retrieval and PR analytics export. URL: https://spec.openapis.org/oas/latest.html

- **Meltwater API (REST/JSON)** — REST API with full reference for all Meltwater API endpoints, parameters, and response schemas; downloadable OpenAPI specification; used for programmatic media monitoring, coverage reporting, and media contact data retrieval. URL: https://developer.meltwater.com/docs/meltwater-api/reference/endpoints/

- **PR Newswire API / RSS Feeds** — PR Newswire provides RSS news feeds (XML/HTML) and programmatic content distribution APIs for wire service distribution; Business Wire, GlobeNewswire, and AccessWire offer similar REST APIs with sub-500ms press release delivery. URL: https://www.prnewswire.com/rss/

- **RTPR (Real Time Press Release API)** — Third-party API aggregating PR Newswire, Business Wire, GlobeNewswire, and AccessWire for programmatic press release consumption; REST API compatible with Python, Node.js; designed for trading algorithms and media monitoring. URL: https://www.rtpr.io/

- **JSON Schema** — Standard format for press release structured data (headline, body, contacts, categories, embargo date/time, multimedia attachments) used in distribution API request payloads. URL: https://json-schema.org/

- **Schema.org NewsArticle / PressRelease** — Structured data vocabulary for marking up press releases in web pages for Google News indexing and rich snippet display; relevant for owned newsroom/press room pages. URL: https://schema.org/NewsArticle

### Security & Authentication Standards

- **GDPR Article 6(1)(f) — Legitimate Interests** — Journalist contact data in PR media databases is typically processed under "legitimate interests" as the GDPR lawful basis; however, individual journalists retain rights to opt out of contact databases; PR platforms must maintain opt-out registries. URL: https://gdpr-info.eu/art-6-gdpr/

- **GDPR Article 85 — Processing for Journalistic Purposes** — GDPR allows member states to provide exemptions for journalistic processing; media contact databases may qualify but the exemption is not universal; PR platforms operating in the EU must address GDPR data subject rights for media contacts. URL: https://gdpr-info.eu/art-85-gdpr/

- **CAN-SPAM Act (US)** — Governs commercial email pitches; journalist pitches are not commercial email in the CAN-SPAM sense, but bulk PR distribution platforms must maintain suppression lists and honour opt-out requests from media contacts who no longer wish to receive pitches. URL: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business

- **CASL — Canada's Anti-Spam Legislation** — Stricter consent requirements for email pitches to Canadian media contacts; implied consent through publicly available professional email addresses typically applies but must be documented. URL: https://crtc.gc.ca/eng/internet/anti.htm

- **SOC 2 Type II** — Required enterprise compliance for SaaS PR management platforms; Cision, Meltwater, and Muck Rack maintain compliance certifications for enterprise customers. URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2

- **SAML 2.0 / OIDC** — Required for enterprise SSO integration in PR management platforms; Cision and Meltwater support SAML 2.0 for corporate identity provider integration in enterprise tier. URL: https://docs.oasis-open.org/security/saml/v2.0/

- **OWASP API Security Top 10 (2023)** — Governs REST API security for PR management platform APIs; API1 (Broken Object Level Authorization) is critical for multi-tenant agencies managing multiple client accounts and media contact databases. URL: https://owasp.org/API-Security/

### MCP Server Specifications

PR management is an emerging area for AI-native workflow integration:

- **AI-Assisted PR Writing (2025-2026)** — PR Newswire's Create+ uses AI to draft press releases, evaluate readability/SEO compliance, and score content quality; represents the leading edge of AI-native press release workflows; MCP server integration for programmatic PR drafting and distribution is emerging.

- **Cision + BI Tool API Pattern** — Cision's REST API integrates with Tableau, PowerBI, and Domo for PR analytics dashboards; this API-to-BI pattern is a precursor to AI agent MCP integration for automated PR performance reporting.

- **Meltwater Developer Portal MCP Pattern** — Meltwater's documented REST API with OpenAPI spec enables AI agent integration for programmatic media monitoring queries; community MCP server wrappers for Meltwater are a natural extension.

---

## Similar Products — Developer Documentation & APIs

### Cision (formerly PR Newswire + Vocus + Gorkana)

- **Description:** Largest global PR management platform; media database (1M+ contacts updated 20,000+ daily), press release distribution via PR Newswire to 270,000+ journalists, 4,000+ websites, and 170+ countries; media monitoring (1M+ web sources, 50,000+ print, 5,000+ broadcast); REST API for BI tool integration.
- **API Documentation:** Cision developer portal (enterprise, requires account)
- **RSS/XML Feeds:** https://www.prnewswire.com/rss/
- **SDKs/Libraries:** REST API (JSON); BI connectors (Tableau, PowerBI, Domo); CSV export
- **Developer Guide:** Enterprise API documentation (account/contract required)
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, SAML 2.0 (Enterprise), RSS 2.0 / Atom feeds, CSV export
- **Authentication:** OAuth 2.0 client credentials; API key for distribution feeds

### Meltwater

- **Description:** AI-powered media intelligence platform; media monitoring (300M+ online sources, 55,000+ print publications, 100+ countries); media database (~400,000 contacts); social listening; REST API with OpenAPI spec; strong in enterprise social and news analytics.
- **API Documentation:** https://developer.meltwater.com/docs/meltwater-api/reference/endpoints/
- **Developer Hub:** https://developer.meltwater.com/
- **SDKs/Libraries:** REST API (JSON); downloadable OpenAPI spec; Zapier integration
- **Developer Guide:** https://developer.meltwater.com/
- **Standards:** REST/JSON, OpenAPI 3.0, OAuth 2.0, CSV/Excel export
- **Authentication:** OAuth 2.0; API key for some endpoints

### Muck Rack

- **Description:** Modern PR management platform with media database (~250,000 journalists), media monitoring, pitching, and PR reporting; praised for modern UX and journalist-verified contact data; REST API for CRM-level PR workflow integration; $10,000–$50,000 annual licences.
- **API Documentation:** Muck Rack API (enterprise, requires account)
- **SDKs/Libraries:** REST API (JSON); CRM integrations (Salesforce, HubSpot); CSV export
- **Developer Guide:** Muck Rack developer portal (account required)
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0
- **Authentication:** API key; OAuth 2.0 for CRM integrations

### PR Newswire (Cision distribution wire)

- **Description:** Leading press release wire service distributing to 270,000+ journalists, 4,000+ websites across 170+ countries; PR Newswire API for programmatic distribution submission; RSS/XML feeds for newswire content consumption; AI-powered Create+ tool for press release drafting.
- **API Documentation:** PR Newswire partner API (requires contract/subscription)
- **RSS Feeds:** https://www.prnewswire.com/rss/
- **SDKs/Libraries:** REST API (JSON); RSS/Atom feeds; CSV analytics export; RTPR (third-party aggregation API)
- **Developer Guide:** PR Newswire partner portal (contract required)
- **Standards:** REST/JSON, OAuth 2.0, RSS 2.0, Atom (RFC 4287), AP Style
- **Authentication:** OAuth 2.0 client credentials; subscription credentials

### Business Wire (Berkshire Hathaway)

- **Description:** Major press release wire service (subsidiary of Berkshire Hathaway); strong in financial disclosure and SEC-mandated press releases; wire distribution API; EDGAR integration for regulatory filings alongside press releases; global distribution network.
- **API Documentation:** Business Wire API (requires subscription)
- **SDKs/Libraries:** REST API (JSON); RSS feeds; financial disclosure format support
- **Developer Guide:** Business Wire developer support (subscription required)
- **Standards:** REST/JSON, OAuth 2.0, RSS 2.0, Atom, AP Style, SEC EDGAR compatible format
- **Authentication:** OAuth 2.0; subscription API credentials

### Prowly (PR Software)

- **Description:** All-in-one PR software with media database, press release creation, online newsroom, and pitch management; REST API; more accessible pricing than Cision/Meltwater ($258–$698/month); growing adoption among mid-market PR teams and agencies.
- **API Documentation:** https://app.prowly.com/api (requires account)
- **SDKs/Libraries:** REST API (JSON); Zapier integration; CRM integrations
- **Developer Guide:** Prowly developer portal (account required)
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, API token auth, Webhooks
- **Authentication:** API token; OAuth 2.0 for integrations

### Propel PRM (PR CRM)

- **Description:** PR-specific CRM platform built around journalist relationship management; contact database, pitch tracking, coverage tagging, and ROI reporting; REST API for CRM integration; positioned as the PR team's system of record for media relationships.
- **API Documentation:** https://www.propelmypr.com/ (requires account)
- **SDKs/Libraries:** REST API (JSON); Salesforce/HubSpot integration; CSV export
- **Developer Guide:** Propel developer portal (account required)
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0
- **Authentication:** OAuth 2.0; API token

### RTPR — Real Time Press Release API

- **Description:** Third-party aggregation API covering PR Newswire, Business Wire, GlobeNewswire, and AccessWire with sub-500ms delivery; REST API compatible with Python, Node.js; designed for financial trading algorithms and automated media monitoring systems.
- **API Documentation:** https://www.rtpr.io/
- **SDKs/Libraries:** REST API; Python client; Node.js client
- **Developer Guide:** https://www.rtpr.io/docs
- **Standards:** REST/JSON, OpenAPI, API key auth
- **Authentication:** API key (subscription-based)

---

## Notes

- **AP Style as the press release content standard**: The Associated Press Stylebook is the universal content standard for US press releases; PR Newswire, Business Wire, and all major wire services enforce AP style in their editorial guidelines; PR management platforms should validate submitted press releases against AP Style rules (date formats, number usage, title capitalisation, abbreviations).

- **Meltwater's OpenAPI spec**: Meltwater's Developer Portal publishes a downloadable OpenAPI specification for their REST API — making it the most developer-friendly of the major PR management platforms; this enables SDK generation and automated integration testing.

- **GDPR and journalist databases**: The GDPR Article 85 journalistic exemption does not unambiguously apply to commercial media contact databases; PR platforms must maintain opt-out mechanisms for journalists who do not wish to be contacted, implement transparent data sourcing practices, and honour deletion requests from media professionals.

- **SEC EDGAR + press release integration**: For public companies, earnings announcements and material disclosures must be filed with the SEC via EDGAR simultaneously with press release distribution; Business Wire has deep EDGAR integration for regulatory compliance alongside PR distribution.

- **AI-native PR (2025-2026)**: PR Newswire's Create+ (AI drafting + SEO scoring), Meltwater's AI-powered pitch intelligence, and Muck Rack's journalist beat matching represent the AI-native frontier; the next generation of PR management tools will use AI agents to draft, personalise, distribute, and monitor press releases autonomously.

- **Open-source landscape**: There is no dominant open-source PR management platform; self-hosted alternatives using open-source components include custom CRM databases (Odoo, SuiteCRM) for media contact management and headless CMS (Ghost, WordPress) for newsroom/press room hosting; the Media Standards Trust's publicity_machine is an open-source press release analysis tool.
