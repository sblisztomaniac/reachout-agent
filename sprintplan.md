# sprintplan.md — 1-Day Build Sprint  
**Agentic AI Outreach Copilot (API + ZeroDB Aligned)**

**Goal:**  
Ship a **working MVP in 1 day** that proves the full loop:

> Intent → Discover (stubbed) → Research → Draft → Approve → Send → Track

No polish. No scale. No perfection.  
Only **truthful execution on the critical path**.

---

## Sprint Constraints

- **Time:** 8–10 focused hours
- **Team:** 1–2 builders
- **Backend:** AINative APIs + ZeroDB
- **Frontend:** Minimal UI (forms + tables)
- **Autonomy:** Draft-only (human approval mandatory)

---

## Definition of “DONE” (Non-Negotiable)

By end of day:
- A user can create an intent
- Generate ≥1 researched lead
- Generate a personalized outreach draft
- Approve & send it
- See it logged in DB

If this works → sprint is a success.

---

## Sprint Architecture (Frozen)

### Tables (MVP subset)
- `users`
- `user_profiles`
- `intent_profiles`
- `campaigns`
- `leads`
- `company_research`
- `outreach_drafts`
- `outreach_sends`
- `outreach_outcomes`
- `events`

### Embedding Namespaces (MVP)
- `company_research`
- `outreach_drafts`
- `replies` (optional if time permits)

---

## Hour-by-Hour Sprint Plan

---

## ⏱ Hour 0–1 — Project & Schema Setup

### Objectives
- Create ZeroDB project
- Create all required tables
- Lock embedding model & namespaces

### Tasks
- Create ZeroDB project (`database_enabled=true`)
- Create tables via `/database/tables`
- Verify inserts work (`row_data` only)

### Acceptance Check
- `GET /database/tables/*/rows` returns empty arrays
- No schema errors

---

## ⏱ Hour 1–2 — Auth + User Profile

### Objectives
- Login works
- User profile persists

### Tasks
- Implement auth flow (JWT)
- Insert user row on first login
- Create/update `user_profiles`

### Tables Touched
- `users`
- `user_profiles`

### Acceptance Check
- Reload → profile persists
- Profile JSON is retrievable

---

## ⏱ Hour 2–3 — Intent + Campaign Creation

### Objectives
- Define *who to reach*
- Bind intent to campaign

### Tasks
- Create intent form (industry, geo, service, tone)
- Insert into `intent_profiles`
- Create `campaigns` row

### Optional (if time)
- Embed intent text into `intent_profiles` namespace

### Acceptance Check
- Campaign row exists
- Intent JSON is correct

---

## ⏱ Hour 3–4 — Lead Creation (Stubbed Discovery)

> ⚠️ **Reality move:**  
> Don’t build real discovery today.  
> Insert 1–3 leads manually or via static agent input.

### Objectives
- Create canonical lead records

### Tasks
- Insert rows into `leads`
- Optionally add `lead_contacts`

### Tables
- `leads`
- `lead_contacts` (optional)

### Acceptance Check
- Leads appear in campaign view
- Deduplication by domain works

---

## ⏱ Hour 4–5 — Company Research Agent (REAL VALUE)

### Objectives
- Generate real company context
- Store & embed research

### Tasks
- Call agent orchestration for research
- Insert `company_research` row
- Embed summary via `embed-and-store`
- Save `embedding_doc_id`

### Tables / Embeddings
- `company_research`
- namespace: `company_research`

### Acceptance Check
- Research text visible in UI
- Semantic search returns this doc

---

## ⏱ Hour 5–6 — Outreach Drafting Agent

### Objectives
- Generate personalized draft
- Store draft + embedding

### Tasks
- Call drafting agent
- Insert `outreach_drafts`
- Embed draft text
- Mark status = `DRAFT`

### Tables / Embeddings
- `outreach_drafts`
- namespace: `outreach_drafts`

### Acceptance Check
- Draft clearly references company research
- Draft editable in UI

---

## ⏱ Hour 6–7 — Human Approval + Send

### Objectives
- Enforce human-in-the-loop
- Log send event

### Tasks
- Approve draft (toggle)
- Call `/agent-bridge/message`
- Insert `outreach_sends`
- Emit `event: outreach_sent`

### Tables
- `outreach_drafts`
- `outreach_sends`
- `events`

### Acceptance Check
- Cannot send unapproved draft
- Send row created successfully

---

## ⏱ Hour 7–8 — Reply Capture + Outcome

### Objectives
- Close the loop
- Enable learning later

### Tasks
- Manually paste reply (for demo)
- Insert `outreach_outcomes`
- Embed reply text (optional)
- Emit analytics event

### Tables / Embeddings
- `outreach_outcomes`
- namespace: `replies` (if time)

### Acceptance Check
- Reply visible in lead timeline
- Outcome label stored

---

## ⏱ Hour 8–9 — Minimal Dashboard + Cleanup

### Objectives
- Show system working end-to-end

### Tasks
- Campaign summary:
  - Leads
  - Drafts
  - Sends
  - Replies
- Clean logs
- Add error states

### Tables
- All MVP tables

---

## ⏱ Hour 9–10 — Buffer / Demo Polish

### Use only if needed
- Fix broken flows
- Improve prompts
- Improve copy clarity
- Prep demo script

---

## What We Explicitly Do NOT Build Today

- Real lead discovery at scale
- Auto follow-ups
- Multi-channel outreach
- Learning optimization
- Team features
- CRM sync
- Fancy UI

These are **future sprints**, not today.

---

## Sprint Demo Script (End of Day)

1. Login
2. Create intent
3. Launch campaign
4. Show researched company
5. Show draft
6. Approve & send
7. Log outcome

If this works → you have a **real agentic product**, not a toy.

---

## Day-After Backlog (Only If This Ships)

- Replace stubbed discovery
- Add qualification scoring
- Add follow-up automation
- Add learning insights

---

## Final Brutal Truth

If you **cannot** ship this in a day:
- Your scope is wrong
- Not your skill

This sprint proves:
- Architecture
- Data model
- Agent workflow
- Business value

Everything else is iteration.

---

**End of sprintplan.md — Ship first, refine later**
