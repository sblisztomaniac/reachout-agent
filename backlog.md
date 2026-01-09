# backlog.md — Agentic AI Outreach Copilot  
**Epics & User Stories mapped explicitly to the Data Model (ZeroDB Tables + Embeddings)**

This backlog is **execution-grade**:
- Every story maps to **tables**, **embedding namespaces**, and **agent APIs**
- Ordered by **critical path**
- No speculative features

---

## EPIC 0 — Platform Foundations (Auth, Project, Guardrails)

### US-0.1 User Authentication & Session
**As a** user  
**I want** to sign up and log in  
**So that** all outreach data is scoped to me  

**Tables**
- `users`

**APIs**
- `/api/v1/auth/`
- `/api/v1/auth/register`

**Acceptance Criteria**
- JWT token issued
- User row exists in `users`
- Role defaults to `USER`

---

### US-0.2 User Profile Setup
**As a** freelancer/founder  
**I want** to define my services, industries, tone  
**So that** agents personalize outreach correctly  

**Tables**
- `user_profiles`

**Acceptance Criteria**
- Profile saved & editable
- Used downstream by agents

---

## EPIC 1 — Outreach Intent System (Targeting Brain)

### US-1.1 Create Outreach Intent
**As a** user  
**I want** to define who I want to reach and why  
**So that** discovery is accurate  

**Tables**
- `intent_profiles`

**Embeddings**
- Namespace: `intent_profiles` (optional)

**Acceptance Criteria**
- Intent stored as structured JSON
- Intent selectable for campaigns

---

### US-1.2 Edit / Archive Intent
**As a** user  
**I want** to reuse or pause intents  
**So that** I can run focused campaigns  

**Tables**
- `intent_profiles.status`

---

## EPIC 2 — Campaign Orchestration

### US-2.1 Create Campaign
**As a** user  
**I want** to launch a campaign from an intent  
**So that** agents execute in sequence  

**Tables**
- `campaigns`

**Acceptance Criteria**
- Campaign links to intent
- Status = `DRAFT` initially

---

### US-2.2 Configure Autonomy Level
**As a** user  
**I want** to choose draft-only vs assisted execution  
**So that** I retain control  

**Tables**
- `campaigns.autonomy_level`

---

## EPIC 3 — Agent Execution Logging

### US-3.1 Log Agent Runs
**As a** system  
**I want** every agent task logged  
**So that** failures are debuggable  

**Tables**
- `agent_runs`

**Acceptance Criteria**
- Every agent task creates a run record
- Status transitions tracked

---

## EPIC 4 — Lead Discovery & Storage

### US-4.1 Discover Raw Leads
**As a** user  
**I want** agents to find companies matching my intent  
**So that** I don’t research manually  

**Tables**
- `leads` (initial insert)

**Embeddings**
- Namespace: `lead_rationales` (why lead fits)

**Acceptance Criteria**
- Leads deduped by domain per user
- Source + signals captured

---

### US-4.2 Attach Contacts to Leads
**As a** user  
**I want** relevant decision-makers identified  
**So that** outreach is directed  

**Tables**
- `lead_contacts`

---

## EPIC 5 — Lead Qualification

### US-5.1 Score Leads
**As a** user  
**I want** low-quality leads filtered out  
**So that** I don’t waste outreach  

**Tables**
- `leads.qualification_score`
- `leads.qualification_label`

**Acceptance Criteria**
- Score between 0–1
- Labels: UNQUALIFIED / MEDIUM / HIGH

---

## EPIC 6 — Company Research Engine

### US-6.1 Generate Research Summary
**As a** user  
**I want** AI to understand each company  
**So that** outreach is contextual  

**Tables**
- `company_research`

**Embeddings**
- Namespace: `company_research`

**Acceptance Criteria**
- Summary, pain hypothesis, hook generated
- Embedded + retrievable

---

## EPIC 7 — Outreach Drafting

### US-7.1 Generate Outreach Drafts
**As a** user  
**I want** personalized drafts per lead  
**So that** messages don’t feel generic  

**Tables**
- `outreach_drafts`

**Embeddings**
- Namespace: `outreach_drafts`

**Acceptance Criteria**
- Channel-specific drafts
- Draft status = `DRAFT`

---

### US-7.2 Edit & Approve Draft
**As a** user  
**I want** to edit and approve drafts  
**So that** nothing sends without consent  

**Tables**
- `outreach_drafts.approved_by_user`
- `outreach_drafts.approved_at`

---

## EPIC 8 — Outreach Execution (Human-in-the-Loop)

### US-8.1 Send Approved Outreach
**As a** user  
**I want** approved drafts sent via agents  
**So that** execution is fast  

**Tables**
- `outreach_sends`

**APIs**
- `/agent-bridge/message`

**Acceptance Criteria**
- Only approved drafts can be sent
- Provider + external_message_id stored

---

## EPIC 9 — Follow-ups & State

### US-9.1 Schedule Follow-ups
**As a** user  
**I want** follow-ups auto-suggested  
**So that** leads aren’t dropped  

**Tables**
- `followups`

---

### US-9.2 Track Lead State
**As a** system  
**I want** lead lifecycle tracked  
**So that** dashboards are accurate  

**Tables**
- `leads.status`

---

## EPIC 10 — Replies & Outcomes

### US-10.1 Capture Replies
**As a** user  
**I want** replies stored and classified  
**So that** learning improves  

**Tables**
- `outreach_outcomes`

**Embeddings**
- Namespace: `replies`

---

### US-10.2 Classify Reply Intent
**As a** system  
**I want** replies labeled (positive, no, later)  
**So that** next action is clear  

**Acceptance Criteria**
- Label + confidence stored
- Suggested next action generated

---

## EPIC 11 — Learning Loop

### US-11.1 Store Interaction Learnings
**As a** system  
**I want** outreach performance logged  
**So that** future drafts improve  

**Tables**
- `events`

**APIs**
- `/agent-learning/interactions`
- `/agent-learning/insights`

---

### US-11.2 Retrieve Winning Patterns
**As a** user  
**I want** AI to reuse what worked  
**So that** outreach quality compounds  

**Embeddings**
- Search `outreach_drafts` namespace

---

## EPIC 12 — Analytics & Visibility

### US-12.1 Campaign Dashboard
**As a** user  
**I want** to see outreach metrics  
**So that** I know what’s working  

**Tables**
- `campaigns`
- `outreach_sends`
- `outreach_outcomes`
- `events`

---

## EPIC 13 — Safety & Limits

### US-13.1 Enforce Outreach Limits
**As a** system  
**I want** daily caps enforced  
**So that** users don’t spam  

**Tables**
- `user_profiles.daily_outreach_limit`
- `events`

---

### US-13.2 Hard Human-in-the-Loop Guardrail
**As a** platform  
**I want** approval mandatory by default  
**So that** trust is never broken  

**Tables**
- `outreach_drafts.approved_by_user`

---

## MVP CUT LINE (If You Need to Ship Fast)

**Must-have Epics**
- EPIC 0 → 8
- EPIC 10 (basic replies)
- EPIC 12 (basic dashboard)

**Defer**
- Advanced learning
- Multi-channel automation
- Agency/team features

---

## Final Reality Check

- Every story maps to **real tables**
- Every AI step produces **persisted artifacts**
- No black-box state
- System is debuggable, auditable, and extensible

---

**End of backlog.md — ready for GitHub / Linear / Jira**
