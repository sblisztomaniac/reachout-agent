# Post-MVP Enhancement Backlog

This document captures features and enhancements that are intentionally deferred from the 1-day MVP build sprint. These items represent the natural evolution of the platform beyond the core workflow.

---

## Epic: Automated Business Discovery Agent

### Context

**Why Deferred**: The 1-day MVP sprint plan includes a "Reality move" that acknowledges we cannot build a fully autonomous discovery agent in the timeframe. Instead, the MVP uses manual lead creation (story-008: "Create lead management with manual entry").

**Strategic Decision**: Focus MVP on proving the core value proposition:
- Manual lead entry → AI research → AI drafting → Approval → Send → Track outcomes

Discovery automation can be added post-MVP without disrupting the core workflow.

---

## Feature: Web Discovery Agent for Lead Generation

### Problem Statement

Users should not have to manually research and enter leads. The platform should autonomously discover relevant businesses based on intent profiles and campaign criteria.

### User Story

**As a** user running an outreach campaign
**I want** the system to automatically discover relevant businesses from the web
**So that** I don't have to manually research and enter leads

### High-Level Requirements

1. **Intent-Driven Discovery**
   - Parse user's intent_profiles to understand ideal customer characteristics
   - Extract discovery criteria from campaign.target_description
   - Identify relevant industries, company sizes, locations, and keywords

2. **Multi-Source Web Search**
   - Integrate with web search MCPs (Open-WebSearch, Brave Search, etc.)
   - Crawl business directories (Crunchbase, AngelList, ProductHunt, etc.)
   - Parse LinkedIn company profiles (if API access available)
   - Scrape industry-specific platforms based on intent

3. **Intelligent Filtering**
   - Deduplicate discovered businesses against existing leads table
   - Apply campaign-specific filters (industry, size, location, funding stage)
   - Score relevance using intent_profiles embeddings
   - Rank by qualification potential (HIGH, MEDIUM, LOW)

4. **Autonomous Lead Creation**
   - Auto-populate leads table with discovered businesses
   - Set qualification_label based on relevance score
   - Mark status as "NEW" for research agent pickup
   - Link to originating campaign_id

5. **Rate Limiting and Scheduling**
   - Respect external API rate limits
   - Run discovery in background (scheduled jobs or on-demand)
   - Batch process discoveries to avoid overloading downstream agents

### Technical Architecture

#### Proposed Agent: Discovery Agent

**Type**: Background Agent (runs on schedule or triggered by campaign creation)

**Inputs**:
- `campaign_id` (UUID) - Campaign to generate leads for
- `intent_profile_id` (UUID, optional) - User's ideal customer profile
- `max_leads` (int) - Maximum leads to discover per run
- `filters` (dict) - Industry, location, size, funding stage filters

**Tools Required**:
- Web Search MCP (for general queries)
- Firecrawl (for business website scraping)
- Embeddings API (for relevance scoring using intent_profiles)
- ZeroDB client (for lead deduplication and insertion)

**Process Flow**:
```
1. Load campaign.target_description and intent_profiles
2. Generate search queries based on intent
3. Execute web searches using MCP
4. For each result:
   a. Extract company metadata (name, website, industry, size)
   b. Scrape website using Firecrawl (if available)
   c. Score relevance against intent_profiles embeddings
   d. Check if company already exists in leads table
   e. If new and above threshold: create lead record
5. Log discovery run to agent_runs table
6. Return count of leads discovered
```

**Outputs**:
- List of newly created lead_ids
- Discovery metrics (searched, filtered, created)
- Error log for failed discoveries

#### Data Model Extensions

**New Fields for `leads` Table** (optional):
- `discovery_source` (TEXT) - Source of lead discovery (e.g., "web_search", "linkedin", "manual")
- `discovery_query` (TEXT) - Search query that found this lead
- `relevance_score` (FLOAT) - Semantic similarity to intent_profiles (0.0-1.0)
- `discovered_at` (TIMESTAMP) - When lead was auto-discovered

**New Table: `discovery_runs`** (optional):
```json
{
  "name": "discovery_runs",
  "schema": {
    "id": {"type": "uuid"},
    "campaign_id": {"type": "uuid"},
    "intent_profile_id": {"type": "uuid"},
    "started_at": {"type": "timestamp"},
    "ended_at": {"type": "timestamp"},
    "search_queries": {"type": "jsonb"},
    "leads_searched": {"type": "integer"},
    "leads_filtered": {"type": "integer"},
    "leads_created": {"type": "integer"},
    "error_log": {"type": "text"},
    "status": {"type": "text"}
  }
}
```

**Enums**:
- `leads.discovery_source`: `WEB_SEARCH`, `LINKEDIN`, `CRUNCHBASE`, `MANUAL`
- `discovery_runs.status`: `RUNNING`, `COMPLETED`, `FAILED`

#### Integration Points

**Discovery Trigger Options**:
1. **On-Demand**: User clicks "Discover Leads" button in campaign UI
2. **Scheduled**: Cron job runs discovery for ACTIVE campaigns daily
3. **Threshold-Based**: Auto-trigger when campaign.leads_count < 10

**Discovery Agent Orchestration**:
```python
# POST /v1/public/agent-orchestration/
{
  "agent_definition": {
    "name": "discovery_agent",
    "role": "Discover relevant businesses for outreach campaign",
    "goal": "Find 20 qualified leads matching campaign criteria",
    "tools": ["web_search_mcp", "firecrawl", "zerodb_client"]
  },
  "input": {
    "campaign_id": "uuid",
    "intent_profile_id": "uuid",
    "max_leads": 20,
    "filters": {
      "industries": ["SaaS", "B2B Software"],
      "company_size": "1-50",
      "location": "United States"
    }
  }
}
```

**Response Handling**:
- Agent returns `discovery_run_id` for tracking
- Poll agent status until COMPLETED
- Retrieve newly created lead_ids
- Update campaign.leads_discovered_count

#### UI/UX Considerations

**Campaign Creation Flow**:
- Add checkbox: "Auto-discover leads" (default: OFF for MVP)
- If enabled: Show discovery settings (max_leads, filters, schedule)
- Preview: "We'll discover up to 20 leads matching your criteria"

**Dashboard**:
- Show discovery metrics: "150 leads discovered this week"
- Link to discovery_runs table for audit trail
- Allow manual trigger: "Discover More Leads" button

**Lead List View**:
- Tag auto-discovered leads with "Auto-discovered" badge
- Show discovery_source and relevance_score
- Allow filtering by discovery method

---

## Implementation Estimate (Post-MVP)

**Phase 1: Basic Discovery** (4-6 hours)
- Integrate web search MCP
- Build discovery agent orchestration
- Implement lead deduplication logic
- Create discovery_runs logging table

**Phase 2: Enhanced Filtering** (3-4 hours)
- Add intent_profiles semantic scoring
- Implement relevance threshold filtering
- Add campaign-specific filters (industry, size, etc.)

**Phase 3: Multi-Source Discovery** (6-8 hours)
- Integrate Firecrawl for website scraping
- Add LinkedIn company search (if API available)
- Parse business directories (Crunchbase, AngelList)

**Phase 4: Scheduling & Automation** (2-3 hours)
- Build background job scheduler
- Implement rate limiting and retry logic
- Add discovery metrics dashboard

**Total Estimate**: 15-21 hours (2-3 days)

---

## Dependencies

**External Services**:
- Web Search MCP (Open-WebSearch or Brave Search API)
- Firecrawl API (for website content extraction)
- LinkedIn API (optional, for company profile scraping)
- Business directory APIs (Crunchbase, AngelList - optional)

**Infrastructure**:
- Background job scheduler (Celery, APScheduler, or native FastAPI background tasks)
- Rate limiter for external API calls
- Caching layer for search results (Redis or in-memory)

**MVP Prerequisites**:
- Story-008 (manual lead creation) must be complete
- Story-009 (research agent) must be working
- Embeddings infrastructure (story-010) must be stable

---

## Success Metrics

**Adoption**:
- % of campaigns using auto-discovery (target: >50% after 1 month)
- Average leads discovered per campaign (target: 20-50)

**Quality**:
- % of discovered leads marked HIGH qualification (target: >30%)
- Discovery relevance score (target: avg >0.7)
- % of discovered leads that result in successful sends (target: >15%)

**Performance**:
- Discovery time per lead (target: <30 seconds)
- False positive rate (leads that are immediately unqualified) (target: <10%)
- Deduplication accuracy (target: 99%+)

---

## Risks & Mitigations

**Risk 1: API Rate Limits**
- External search APIs may throttle requests
- **Mitigation**: Implement exponential backoff, batch processing, caching

**Risk 2: Low-Quality Discoveries**
- Web search may return irrelevant businesses
- **Mitigation**: Strict relevance scoring, user feedback loop, manual review option

**Risk 3: LinkedIn API Restrictions**
- LinkedIn heavily restricts automated scraping
- **Mitigation**: Focus on web search + Firecrawl first, add LinkedIn only if API access secured

**Risk 4: Cost of External APIs**
- Firecrawl, Crunchbase APIs can be expensive at scale
- **Mitigation**: Set per-user discovery limits, offer tiered pricing, cache aggressively

---

## Alternative Approaches Considered

### Option 1: User-Provided CSV Import
**Pros**: Simple, no external APIs, works day 1
**Cons**: Still manual, doesn't solve discovery problem

**Decision**: Keep as fallback option, but push for autonomous discovery

### Option 2: Zapier/Make.com Integration
**Pros**: Leverage existing discovery tools without building from scratch
**Cons**: Less control, less integrated UX, external dependency

**Decision**: Consider as interim solution for power users

### Option 3: Crowdsourced Lead Database
**Pros**: Community-driven, lower API costs
**Cons**: Quality control challenges, cold start problem

**Decision**: Defer until platform has significant user base

---

## Related Features (Future)

1. **LinkedIn Profile Enrichment**: Auto-fetch decision-maker contacts from LinkedIn
2. **Email Finder Integration**: Apollo.io, Hunter.io for contact discovery
3. **CRM Import**: Sync existing leads from Salesforce, HubSpot
4. **Lead Scoring ML**: Train model on user feedback to improve discovery relevance
5. **Industry-Specific Discovery**: Tailored scrapers for niche verticals (legal, healthcare, etc.)

---

## References

- Sprint Plan: docs/sprintplan.md (Hour 3-4: "Reality move" on discovery)
- Story-008: Manual lead creation (MVP approach)
- PRD Section 5.3: Future Features (mentions "AI-powered lead discovery")
- Data Model: docs/datamodel.md (leads table schema)

---

## Notes for Implementation

**When to Build This**:
- After MVP demo (post-sprint)
- After validating core workflow with real users
- After securing necessary API access (web search, Firecrawl, etc.)

**Technical Debt to Address First**:
- Story-009 (research agent) must be robust (discovery will create load)
- Agent orchestration retry/timeout handling must be production-ready
- ZeroDB rate limiting must be understood and respected

**Product Validation**:
- Survey MVP users: "Would you pay $X/month for automated lead discovery?"
- A/B test: Manual vs auto-discovered leads - which converts better?
- Measure time savings: Manual entry (5 min/lead) vs auto-discovery (30 sec/lead)
