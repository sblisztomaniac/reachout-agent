# PRD: AI-Native Outreach Copilot (Reachout Agent)
**Version**: 2.0 (AN8F-Aligned)
**Status**: Pre-Implementation
**Repo**: reachout-agent

---

## 0. Executive Summary

An AI-powered outreach assistant for freelancers and solo founders that reduces manual research and drafting time by automating:
1. Business discovery coordination (via external search tools)
2. Company research and understanding (via AI agents)
3. Personalized outreach generation (via AI agents)
4. Send orchestration with mandatory human approval

**Not a goal**: Fully autonomous spam system, perfect lead discovery, or ToS-violating scrapers.

---

## 1. Problem Definition

**User Pain**:
Freelancers and solo founders spend 70-80% of outreach time on manual research (finding businesses, understanding them, crafting personalized messages) and only 20% on actual relationship building.

**System Responsibility**:
Coordinate external discovery tools and AI agents to generate contextual, personalized outreach drafts that humans can review and send.

**Explicit Non-Responsibility**:
- Web crawling (delegated to external tools/MCPs)
- Platform-native messaging (delegated to external APIs via Agent Bridge)
- Lead list purchasing or spam generation

---

## 2. AN8F Lifecycle Alignment

This product follows the **AI-Native Studio AN8F Development Lifecycle**:

```
PRD (this doc)
  ↓
ZeroDB Contract (datamodel.md + table schemas)
  ↓
Frontend Stub (separate repo, API contracts only)
  ↓
Backend Implementation (this repo: /backend)
  ↓
Kill Mocks (replace stubs with real agent orchestrations)
  ↓
GAP Analysis (validate assumptions, fix holes)
  ↓
Issues → Ship
```

**Repository Structure**:
- `/backend` - FastAPI backend + agent logic
- `/docs` - PRD, data model, backlog
- `/ralph` - Ralph loops for orchestration/retry
- Frontend lives in **separate repo**

---

## 3. System Architecture (Reality-Based)

### 3.1 External Boundary: Discovery Tools

**Discovery is NOT performed by AI-Native agents directly.**

Discovery must use:
- **External Search MCP Servers** (e.g., Open-WebSearch MCP, Brave Search API)
- **External Scraping Services** (e.g., Firecrawl, Bright Data, Apify)
- **Custom Scrapers** (MCP-wrapped where possible)

**Agent Role in Discovery**:
Agents *decide*:
- What search queries to generate
- How to interpret search results
- Whether a result qualifies as a lead

Agents do NOT:
- Crawl Google
- Scrape Instagram directly
- Violate platform ToS

### 3.2 AI-Native APIs Used

| API | Purpose |
|-----|---------|
| `/v1/public/agent-orchestration/` | Create agents, execute tasks |
| `/v1/public/agent-coordination/sequences` | Multi-step workflows |
| `/v1/public/agent-coordination/messages` | Agent-to-agent communication |
| `/v1/public/agent-framework/memories` | Store/retrieve agent context |
| `/v1/public/agent-learning/interactions` | Capture outcomes for improvement |
| `/v1/public/agent-bridge/message` | Send via external channels |
| `/v1/public/{project_id}/database/tables` | ZeroDB table CRUD |
| `/v1/public/{project_id}/embeddings/embed-and-store` | Semantic storage |
| `/v1/public/{project_id}/embeddings/search` | Semantic retrieval |
| `/v1/public/agent-state/states` | Workflow state tracking |

**No Hallucinated Endpoints**: If an operation is not listed above, it either:
1. Doesn't exist in AI-Native
2. Must be implemented as external tool integration
3. Is out of scope for MVP

### 3.3 Ralph Loops Integration

The `/ralph` directory contains:
- Orchestration loops for retry logic
- Control flow wrappers for multi-step processes
- Progress tracking for long-running operations

**Ralph Usage**:
- Wrap discovery → enrichment → matching flows
- Handle transient failures gracefully
- Log progress for debugging

**Ralph is NOT**:
- A magic auto-coder
- A replacement for explicit agent orchestration
- Capable of inventing new capabilities

---

## 4. Agent Roles (Explicit Responsibilities)

### 4.1 Discovery Coordinator Agent
**Input**: User intent (industries, geographies, signals)
**Actions**:
- Generate search queries for external tools
- Call external MCP search tools
- Parse search results
- Insert raw leads into `leads` table

**Tools Required**:
- External MCP: Open-WebSearch or equivalent
- ZeroDB: `leads` table insert

### 4.2 Enrichment Agent
**Input**: Lead (company name, domain, initial signals)
**Actions**:
- Call external scraper for company website
- Extract: products, team size, recent news, hiring signals
- Store structured research in `company_research` table
- Embed research summary for semantic retrieval

**Tools Required**:
- External: Firecrawl or web scraper MCP
- ZeroDB: `company_research` table + embeddings

### 4.3 Qualification Agent
**Input**: Lead + research
**Actions**:
- Score lead relevance (0-1)
- Label: HIGH / MEDIUM / LOW / UNQUALIFIED
- Update `leads.qualification_score` and `leads.qualification_label`

**Tools Required**:
- Agent Framework: memory search for similar past leads
- ZeroDB: `leads` table update

### 4.4 Drafting Agent
**Input**: Qualified lead + research + user profile
**Actions**:
- Generate personalized outreach draft (email/LinkedIn)
- Reference specific company context (hook)
- Insert into `outreach_drafts` table
- Embed draft for pattern reuse

**Tools Required**:
- Agent Framework: memory retrieval (user tone, winning patterns)
- ZeroDB: `outreach_drafts` table + embeddings

### 4.5 Learning Agent
**Input**: Outreach outcome (reply/no-reply, sentiment)
**Actions**:
- Log interaction via Agent Learning API
- Identify patterns (which hooks work, which industries respond)
- Surface insights for future drafts

**Tools Required**:
- `/v1/public/agent-learning/interactions`
- `/v1/public/agent-learning/insights`

---

## 5. User Flow (Step-by-Step with Tool Boundaries)

```
1. User Login
   → Auth API (`/v1/auth/login`)

2. User Defines Intent
   → Insert into `intent_profiles` table
   → Optionally embed intent for similarity matching

3. User Launches Campaign
   → Insert into `campaigns` table
   → Create Agent Sequence via `/agent-coordination/sequences`

4. Discovery Phase (External Tool Boundary)
   → Discovery Coordinator Agent decides search queries
   → Calls **External MCP** (Open-WebSearch)
   → Parses results → inserts `leads` table

5. Enrichment Phase (External Tool Boundary)
   → Enrichment Agent per lead
   → Calls **External Scraper** (Firecrawl MCP)
   → Parses company data → inserts `company_research`
   → Embeds research summary

6. Qualification Phase (AI-Native Only)
   → Qualification Agent per lead
   → Semantic search in `company_research` embeddings
   → Updates `leads` table with score/label

7. Drafting Phase (AI-Native Only)
   → Drafting Agent per qualified lead
   → Retrieves user profile + research + past winning drafts
   → Generates draft → inserts `outreach_drafts`
   → Embeds draft

8. Human Review (Frontend)
   → User sees drafts in UI
   → Can edit, approve, or reject

9. Send Phase (External Tool Boundary via Agent Bridge)
   → Approved drafts only
   → `/agent-bridge/message` → external email/LinkedIn API
   → Log in `outreach_sends` table

10. Outcome Tracking
    → User manually pastes replies (MVP) or webhook ingestion (future)
    → Insert into `outreach_outcomes`
    → Learning Agent logs via `/agent-learning/interactions`

11. Learning Loop
    → `/agent-learning/insights` surfaces patterns
    → Future drafts improve automatically
```

**Tool Boundaries Summary**:
- **External**: Steps 4, 5, 9 (search, scrape, send)
- **AI-Native Agents**: Steps 6, 7, 10 (qualify, draft, learn)
- **ZeroDB**: All data persistence
- **Ralph Loops**: Wrap steps 4-7 for retry/orchestration

---

## 6. Data Model (Summary - See datamodel.md)

**Tables** (ZeroDB):
- `users`, `user_profiles` - User identity
- `intent_profiles` - Targeting configuration
- `campaigns` - Outreach runs
- `leads` - Canonical company records
- `company_research` - AI-generated understanding
- `outreach_drafts` - Generated messages
- `outreach_sends` - Send logs
- `outreach_outcomes` - Replies + learning data

**Embeddings** (384-dim, `BAAI/bge-small-en-v1.5`):
- `company_research` namespace - For semantic company matching
- `outreach_drafts` namespace - For pattern reuse
- `replies` namespace - For sentiment/classification learning

---

## 7. MVP Scope (Brutal Honesty)

### IN SCOPE (Deliverable in 1-Day Sprint)
- ✅ Auth + user profile
- ✅ Intent definition
- ✅ Campaign creation
- ✅ **Stubbed discovery** (1-3 manual leads)
- ✅ Real enrichment agent (Firecrawl integration)
- ✅ Real drafting agent
- ✅ Human approval workflow
- ✅ Manual send (Agent Bridge)
- ✅ Outcome logging
- ✅ Basic dashboard

### OUT OF SCOPE (Post-MVP)
- ❌ Real-time discovery at scale
- ❌ Multi-channel automation (email + LinkedIn)
- ❌ Automatic follow-ups
- ❌ CRM sync
- ❌ Team collaboration
- ❌ Reply webhook ingestion
- ❌ Spam detection / deliverability optimization

---

## 8. Non-Goals (Hard Boundaries)

1. **No Autonomous Spam**: Human approval is MANDATORY for all sends
2. **No Platform Abuse**: No ToS violations (LinkedIn scraping limits, etc.)
3. **No Black Boxes**: All agent decisions must be auditable in ZeroDB
4. **No Purchased Lists**: Only organic discovery via search tools
5. **No Guarantees**: System helps, doesn't promise conversions

---

## 9. Success Metrics

**User Outcome**:
- Reduce research time from 30min/lead → 5min/lead
- Increase drafts created from 5/day → 20/day
- Maintain or improve reply rate vs manual outreach

**System Health**:
- <5s p95 latency for draft generation
- >80% draft approval rate (quality signal)
- <10% external tool failure rate

---

## 10. Open Questions & Risks

### Assumptions to Validate
1. **External Tool Reliability**: Can Firecrawl/MCPs handle 100 requests/day?
2. **Agent Quality**: Do drafts feel personalized enough for approval?
3. **Cost**: OpenAI API costs for enrichment + drafting per lead?

### Known Risks
| Risk | Mitigation |
|------|------------|
| External tool rate limits | Implement retry + backoff via Ralph loops |
| Low draft approval rate | Iterative prompt tuning via Agent Learning |
| User expects full automation | Explicit messaging: "Copilot, not autopilot" |
| Search results are low quality | Refined search query generation via agent feedback |

---

## 11. Implementation Phases

### Phase 0: Foundations (Week 1)
- ZeroDB schema creation
- Auth + user profiles
- Intent definition UI

### Phase 1: MVP Core (Week 2 - Sprint Focus)
- Stubbed discovery
- Real enrichment agent (Firecrawl)
- Real drafting agent
- Human approval flow
- Manual send

### Phase 2: Discovery Automation (Week 3-4)
- Replace stubbed discovery with MCP integration
- Qualification scoring
- Batch processing

### Phase 3: Learning Loop (Week 5-6)
- Outcome tracking
- Learning agent integration
- Pattern insights UI

### Phase 4: Scale & Polish (Week 7+)
- Multi-channel support
- Follow-up automation
- CRM integrations

---

## 12. Dependencies

**External Services**:
- Open-WebSearch MCP or Brave Search API (discovery)
- Firecrawl or equivalent (website scraping)
- Email API (SendGrid, Resend, etc.)
- LinkedIn API (if supporting LinkedIn outreach)

**AI-Native Platform**:
- Agent Orchestration
- Agent Coordination
- Agent Framework (Memory)
- Agent Learning
- ZeroDB (Tables + Embeddings)

**Infrastructure**:
- Frontend hosting (separate repo)
- Backend hosting (Railway, Fly.io, etc.)
- API keys for external tools

---

## 13. Acceptance Criteria (MVP Complete)

By end of Phase 1:
- [ ] User can log in and create profile
- [ ] User can define outreach intent
- [ ] User can launch campaign with 1-3 stubbed leads
- [ ] Enrichment agent fetches real company data
- [ ] Drafting agent generates personalized draft
- [ ] User can approve draft in UI
- [ ] Draft sends via Agent Bridge
- [ ] Send logged in `outreach_sends` table
- [ ] All data persisted in ZeroDB
- [ ] No errors in agent execution logs

---

**End of PRD**
