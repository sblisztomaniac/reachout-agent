# PRD: Agentic AI Outreach Copilot  
**API-Aligned with AINative Studio (v1)**

---

## 0. Objective (No Abstractions)

Build an **Agentic AI system** that helps freelancers and solo founders **discover, qualify, research, and reach out to businesses** with minimal manual effort, using **only existing AINative Studio APIs**.

This PRD is **fully realigned** to what is *actually possible* with the current API surface.  
No imaginary endpoints. No hand-wavy agents.

---

## 1. Product Definition (System-Truth)

**Outreach Copilot** is a **multi-agent workflow system**, composed using:

- Agent Orchestration
- Agent Coordination
- Agent Framework (Memory)
- Agent Learning
- Agent State
- Agent Bridge
- Auth + RBAC

The product does **not** invent new primitives.  
It **composes workflows** using existing APIs.

---

## 2. Target Users

### Primary
- Freelancers
- Solo founders
- Indie consultants

### Secondary
- Small agencies (≤10 people)

---

## 3. Authentication & Identity (FOUNDATION)

### APIs Used
- `POST /api/v1/auth/`
- `POST /api/v1/auth/register`
- `GET /api/v1/users/me`

### Role Usage
| Role | Purpose |
|---|---|
| USER | Run outreach workflows |
| ADMIN | Manage agents, debug, monitor |

All agents, memory, tasks, learning are **scoped to authenticated user**.

---

## 4. Core Product Objects → API Mapping

| Product Concept | API Primitive |
|---|---|
| User | Auth User |
| Outreach Campaign | Agent Sequence |
| Agent | Orchestrated Agent |
| Lead | Memory Entry |
| Outreach Draft | Agent Message |
| Follow-up | Task + State |
| Learning | Agent Learning Insight |

---

## 5. High-Level Workflow (End-to-End)

1. User logs in
2. User defines **Outreach Intent**
3. System creates agents
4. Agent sequence executes:
   - Discover → Qualify → Research → Draft
5. User reviews drafts
6. Messages sent via Agent Bridge
7. Replies + outcomes logged
8. Learning fed back into system

---

## 6. Outreach Intent (User Input → Memory)

### Inputs
- Services offered
- Target industries
- Geography
- Company size
- Outreach channel
- Tone preferences
- Daily limits

### API Usage
- `POST /agent-framework/memories`

```json
{
  "type": "outreach_intent",
  "industries": ["fintech"],
  "geo": ["India"],
  "services": ["AI consulting"],
  "tone": "friendly"
}


⸻

7. Agent Architecture (Concrete)

⸻

7.1 Discovery Agent

Purpose
Find potential businesses matching intent.

APIs
	•	POST /agent-orchestration/agents
	•	POST /agent-orchestration/tasks
	•	POST /agent-orchestration/tasks/{id}/execute

Output
	•	Company candidates → memory

{
  "type": "lead_raw",
  "company": "Acme Fintech",
  "source": "website",
  "reason": "Hiring engineers"
}

Stored via:
	•	POST /agent-framework/memories

⸻

7.2 Qualification Agent

Purpose
Filter and score leads.

APIs
	•	POST /agent-coordination/messages
	•	POST /agent-framework/memories/search

Logic
	•	Reads lead_raw
	•	Scores relevance
	•	Writes lead_qualified

{
  "type": "lead_qualified",
  "company": "Acme Fintech",
  "score": 0.82
}


⸻

7.3 Research & Context Agent

Purpose
Understand each company deeply.

APIs
	•	POST /agent-orchestration/tasks
	•	POST /agent-coordination/messages

Output

{
  "type": "company_research",
  "company": "Acme Fintech",
  "summary": "...",
  "pain_hypothesis": "...",
  "outreach_hook": "..."
}

Stored using:
	•	POST /agent-framework/memories

⸻

7.4 Outreach Drafting Agent

Purpose
Generate personalized outreach.

APIs
	•	POST /agent-coordination/messages

Produces
	•	Email / LinkedIn drafts

{
  "type": "outreach_draft",
  "channel": "email",
  "subject": "...",
  "body": "..."
}


⸻

7.5 Human-in-the-Loop Control (MANDATORY)

Default Mode
	•	AI drafts only
	•	User approves before send

APIs
	•	Read from memory
	•	Edit client-side
	•	Send via Agent Bridge

⸻

7.6 Send & Execution (Agent Bridge)

APIs
	•	POST /agent-bridge/message

Used only after approval.

⸻

7.7 Follow-Up & State Tracking

Purpose
Track lifecycle of outreach.

APIs
	•	POST /agent-state/states
	•	POST /agent-state/states/{id}/checkpoints

States
	•	Drafted
	•	Sent
	•	Replied
	•	Followed-up
	•	Closed

⸻

7.8 Learning Agent

Purpose
Improve future outreach.

APIs
	•	POST /agent-learning/interactions
	•	POST /agent-learning/interactions/{id}/feedback
	•	GET /agent-learning/insights

Signals Captured
	•	Reply/no reply
	•	Positive/negative
	•	Time to response

⸻

8. Agent Sequences (Campaigns)

APIs
	•	POST /agent-coordination/sequences
	•	POST /agent-coordination/sequences/{id}/execute

Example Sequence
	1.	Discovery Agent
	2.	Qualification Agent
	3.	Research Agent
	4.	Draft Agent

This is the core campaign unit.

⸻

9. Memory Model (ZeroDB via Agent Framework)

Memory Type	Purpose
outreach_intent	User goals
lead_raw	Unfiltered leads
lead_qualified	Scored leads
company_research	Context
outreach_draft	Draft messages
outreach_outcome	Results


⸻

10. RBAC Enforcement

Role	Access
USER	Run campaigns
DEVELOPER	Debug agents
ADMIN	Full access
GUEST	Read-only

APIs enforce this centrally.

⸻

11. Error Handling (Product Level)

Handled via:
	•	HTTP status
	•	Agent task failure states
	•	Retry at task level

Frontend must:
	•	Show agent execution status
	•	Never silently fail

⸻

12. MVP Scope (STRICT)

Included
	•	Auth
	•	Intent setup
	•	Discovery → Draft
	•	Manual send
	•	Learning capture

Excluded
	•	Auto-sending
	•	CRM sync
	•	Team collaboration
	•	Mass campaigns

⸻

13. Non-Goals (Hard No)
	•	Spam blasting
	•	Purchased lead lists
	•	Fully autonomous outreach
	•	Black-box behavior

⸻

14. Why This Works (Reality Check)
	•	Uses existing APIs only
	•	Aligns with agent-first architecture
	•	Keeps humans in control
	•	Scales from solo to agency
	•	Zero reinvention of backend primitives

⸻

15. Open Decisions (Product, Not Tech)
	•	Email vs LinkedIn priority
	•	Pricing model
	•	Vertical focus for MVP
	•	UI abstraction depth

⸻
## 📡 External Data Acquisition & Tooling Boundary (Clarification)

### Purpose

This section clarifies **how business discovery and data collection occur** in the system, and **what is handled by AINative agents vs external tools**.

This is a clarification of responsibility boundaries, **not a change to product behavior**.

---

### Business Discovery

The system does **not** rely on proprietary lead databases.

Instead, business discovery is performed via **external search and scraping tools**, coordinated by agent workflows.

**Examples of discovery sources (non-binding):**
- Google Search / Google Maps (business listings)
- Public business directories
- Company websites
- Optional social platforms (e.g. Instagram, LinkedIn) as secondary signals

The choice of tools is an **implementation detail** and may evolve without changing product behavior.

---

### Website & Data Scraping

Once a potential business is identified, relevant public information is collected using **external scraping tools**, such as:
- Homepage content
- About / Services pages
- Public announcements or updates

Agents do **not** directly scrape the web themselves; they:
- Decide *what* to search
- Decide *which URLs* to scrape
- Consume and interpret the collected text

---

### Role of AINative Agents

AINative agents act as the **control plane and reasoning layer**, not the raw data collectors.

Agents are responsible for:
- Orchestrating discovery and scraping tasks
- Reasoning over collected data using LLMs
- Structuring business understanding (what the company does, likely needs, stage)
- Matching user offerings to business context
- Generating personalized outreach drafts
- Persisting memory, state, and learning artifacts

---

### Explicit Non-Goals

- AINative APIs do **not** replace search engines or social platforms
- The system does **not** guarantee exhaustive discovery of all businesses
- Social platforms are treated as **signals**, not authoritative data sources

---

### Design Principle

The product is designed to be:
- **Tool-agnostic** at the PRD level
- **Agent-coordinated**, not scraper-dependent
- Flexible to swap discovery or scraping tools without altering product intent

This separation ensures long-term adaptability while preserving a clean product architecture.


