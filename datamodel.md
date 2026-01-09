# Data Model (ZeroDB-Aligned) — Agentic AI Outreach Copilot
**Goal:** A buildable data model for the PRD that uses **ZeroDB Tables + Embeddings** correctly, per the ZeroDB Developer Guide (Dec 13, 2025).

---

## 0) ZeroDB Rules We Must Not Violate

### Embeddings / Vectors
- **Default model:** `BAAI/bge-small-en-v1.5`
- **Dimensions:** **384**
- **Store & search must use same model + same namespace**
- Prefer `POST /embeddings/embed-and-store` for text you will retrieve semantically

### Tables
- All table endpoints must use:
  - `/v1/public/{project_id}/database/tables/...`
- Inserts must use:
  - `row_data` (NOT `data`, NOT `rows`)

---

## 1) What Goes in Tables vs What Goes in Embeddings

### Put in **Tables** (structured truth)
- Users / profiles
- Campaign definitions
- Leads (canonical record)
- Outreach drafts + approvals
- Send logs + outcomes
- Follow-up schedules
- Agent runs + statuses
- Analytics events
- Integrations metadata

### Put in **Embeddings** (semantic retrieval)
- Company research notes (website summary, pain hypothesis, hook)
- Lead “why relevant” rationale
- Outreach drafts (to reuse best patterns / retrieve by intent)
- Reply text snippets (to classify & learn)
- User preferences / tone guidelines (optional, but powerful)

**Principle:**  
Tables = *system of record*  
Embeddings = *system of recall*

---

## 2) Namespaces (Embeddings) — Clean Separation

Use namespaces to avoid mixing incompatible retrieval spaces:

| Namespace | What gets stored | Used for |
|---|---|---|
| `intent_profiles` | User intent descriptions, constraints | Retrieve similar past intents |
| `lead_rationales` | “why this lead fits” explanations | Similar-lead discovery |
| `company_research` | site/product summaries, hooks | Draft personalization |
| `outreach_drafts` | approved drafts + context | Reuse winning patterns |
| `replies` | inbound reply text + tags | Classification + learnings |

Default embedding model for all namespaces: **`BAAI/bge-small-en-v1.5` (384-dim)**

---

## 3) Table Schema (ZeroDB Tables API)

> Below are the exact tables you should create via:
`POST /v1/public/{project_id}/database/tables`

### 3.1 `users`
System-level user profile data (minimal duplication if auth exists elsewhere).

**Schema**
```json
{
  "name": "users",
  "description": "User accounts for OutreachCopilot",
  "schema": {
    "id": "UUID PRIMARY KEY",
    "email": "TEXT UNIQUE",
    "full_name": "TEXT",
    "role": "TEXT DEFAULT 'USER'", 
    "status": "TEXT DEFAULT 'ACTIVE'",
    "created_at": "TIMESTAMP DEFAULT NOW()",
    "updated_at": "TIMESTAMP DEFAULT NOW()"
  }
}


⸻

3.2 user_profiles

Freelancer/Founder-specific business identity.

Schema

{
  "name": "user_profiles",
  "description": "Freelancer/founder profile for personalization and targeting",
  "schema": {
    "id": "UUID PRIMARY KEY",
    "user_id": "UUID NOT NULL",
    "persona": "TEXT", 
    "services": "JSONB",
    "industries": "JSONB",
    "geographies": "JSONB",
    "pricing_note": "TEXT",
    "portfolio_url": "TEXT",
    "case_studies": "JSONB",
    "tone_preference": "TEXT DEFAULT 'friendly'",
    "daily_outreach_limit": "INT DEFAULT 10",
    "created_at": "TIMESTAMP DEFAULT NOW()",
    "updated_at": "TIMESTAMP DEFAULT NOW()"
  }
}


⸻

3.3 intent_profiles

Each “Outreach Intent” becomes a reusable target definition.

Schema

{
  "name": "intent_profiles",
  "description": "Structured outreach intent definitions",
  "schema": {
    "id": "UUID PRIMARY KEY",
    "user_id": "UUID NOT NULL",
    "name": "TEXT NOT NULL",
    "intent_json": "JSONB NOT NULL",
    "status": "TEXT DEFAULT 'ACTIVE'",
    "created_at": "TIMESTAMP DEFAULT NOW()",
    "updated_at": "TIMESTAMP DEFAULT NOW()"
  }
}

Example intent_json

{
  "industries": ["fintech"],
  "geo": ["India"],
  "company_size": ["1-50", "51-200"],
  "signals": ["hiring", "recent_launch"],
  "channels": ["email", "linkedin"],
  "tone": "friendly",
  "constraints": { "no_spam": true, "needs_approval": true }
}


⸻

3.4 campaigns

A campaign is a run configuration tied to an intent.

Schema

{
  "name": "campaigns",
  "description": "Outreach campaign definition and lifecycle",
  "schema": {
    "id": "UUID PRIMARY KEY",
    "user_id": "UUID NOT NULL",
    "intent_profile_id": "UUID NOT NULL",
    "name": "TEXT NOT NULL",
    "status": "TEXT DEFAULT 'DRAFT'", 
    "channels": "JSONB",
    "autonomy_level": "TEXT DEFAULT 'DRAFT_ONLY'", 
    "start_at": "TIMESTAMP",
    "end_at": "TIMESTAMP",
    "created_at": "TIMESTAMP DEFAULT NOW()",
    "updated_at": "TIMESTAMP DEFAULT NOW()"
  }
}


⸻

3.5 agent_runs

Every orchestrated sequence/task execution should be logged.

Schema

{
  "name": "agent_runs",
  "description": "Logs for agent tasks and sequences executions",
  "schema": {
    "id": "UUID PRIMARY KEY",
    "user_id": "UUID NOT NULL",
    "campaign_id": "UUID",
    "agent_name": "TEXT NOT NULL",
    "run_type": "TEXT NOT NULL", 
    "status": "TEXT DEFAULT 'QUEUED'", 
    "input": "JSONB",
    "output": "JSONB",
    "error": "TEXT",
    "started_at": "TIMESTAMP",
    "ended_at": "TIMESTAMP",
    "created_at": "TIMESTAMP DEFAULT NOW()"
  }
}


⸻

3.6 leads

Canonical lead record (one row per company per user, deduped by domain).

Schema

{
  "name": "leads",
  "description": "Canonical lead/company record",
  "schema": {
    "id": "UUID PRIMARY KEY",
    "user_id": "UUID NOT NULL",
    "company_name": "TEXT NOT NULL",
    "domain": "TEXT",
    "website_url": "TEXT",
    "linkedin_url": "TEXT",
    "country": "TEXT",
    "industry": "TEXT",
    "company_size_estimate": "TEXT",
    "signals": "JSONB",
    "qualification_score": "FLOAT DEFAULT 0.0",
    "qualification_label": "TEXT DEFAULT 'UNQUALIFIED'",
    "status": "TEXT DEFAULT 'NEW'",
    "created_at": "TIMESTAMP DEFAULT NOW()",
    "updated_at": "TIMESTAMP DEFAULT NOW()"
  }
}


⸻

3.7 lead_contacts

Contacts discovered for a lead (people to reach out to).

Schema

{
  "name": "lead_contacts",
  "description": "Contacts within a lead/company",
  "schema": {
    "id": "UUID PRIMARY KEY",
    "lead_id": "UUID NOT NULL",
    "name": "TEXT",
    "title": "TEXT",
    "email": "TEXT",
    "linkedin_url": "TEXT",
    "confidence": "FLOAT DEFAULT 0.0",
    "source": "TEXT",
    "created_at": "TIMESTAMP DEFAULT NOW()"
  }
}


⸻

3.8 company_research

Structured research output (and a pointer to the embedded doc).

Schema

{
  "name": "company_research",
  "description": "Research notes per company (structured + link to vector doc id)",
  "schema": {
    "id": "UUID PRIMARY KEY",
    "lead_id": "UUID NOT NULL",
    "summary": "TEXT",
    "pain_hypothesis": "TEXT",
    "hook": "TEXT",
    "sources": "JSONB",
    "embedding_doc_id": "TEXT", 
    "created_at": "TIMESTAMP DEFAULT NOW()",
    "updated_at": "TIMESTAMP DEFAULT NOW()"
  }
}


⸻

3.9 outreach_drafts

Draft messages (email/LinkedIn) + review/approval state.

Schema

{
  "name": "outreach_drafts",
  "description": "Generated outreach drafts awaiting approval or sent",
  "schema": {
    "id": "UUID PRIMARY KEY",
    "campaign_id": "UUID NOT NULL",
    "lead_id": "UUID NOT NULL",
    "contact_id": "UUID",
    "channel": "TEXT NOT NULL",
    "subject": "TEXT",
    "body": "TEXT NOT NULL",
    "status": "TEXT DEFAULT 'DRAFT'",
    "approved_by_user": "BOOLEAN DEFAULT FALSE",
    "approved_at": "TIMESTAMP",
    "embedding_doc_id": "TEXT",
    "created_at": "TIMESTAMP DEFAULT NOW()",
    "updated_at": "TIMESTAMP DEFAULT NOW()"
  }
}


⸻

3.10 outreach_sends

Send logs (what was actually sent, when, how).

Schema

{
  "name": "outreach_sends",
  "description": "Outreach send logs",
  "schema": {
    "id": "UUID PRIMARY KEY",
    "draft_id": "UUID NOT NULL",
    "channel": "TEXT NOT NULL",
    "provider": "TEXT", 
    "to_address": "TEXT",
    "external_message_id": "TEXT",
    "status": "TEXT DEFAULT 'SENT'",
    "sent_at": "TIMESTAMP DEFAULT NOW()",
    "raw_payload": "JSONB"
  }
}


⸻

3.11 outreach_outcomes

Outcome tracking + response classification.

Schema

{
  "name": "outreach_outcomes",
  "description": "Reply outcomes and classifications",
  "schema": {
    "id": "UUID PRIMARY KEY",
    "send_id": "UUID NOT NULL",
    "lead_id": "UUID NOT NULL",
    "reply_received": "BOOLEAN DEFAULT FALSE",
    "reply_text": "TEXT",
    "reply_label": "TEXT", 
    "reply_confidence": "FLOAT DEFAULT 0.0",
    "next_action": "TEXT",
    "embedding_doc_id": "TEXT",
    "created_at": "TIMESTAMP DEFAULT NOW()",
    "updated_at": "TIMESTAMP DEFAULT NOW()"
  }
}


⸻

3.12 followups

Follow-up scheduling and execution.

Schema

{
  "name": "followups",
  "description": "Follow-up schedule + execution status",
  "schema": {
    "id": "UUID PRIMARY KEY",
    "campaign_id": "UUID NOT NULL",
    "lead_id": "UUID NOT NULL",
    "send_id": "UUID",
    "due_at": "TIMESTAMP NOT NULL",
    "status": "TEXT DEFAULT 'SCHEDULED'",
    "followup_body": "TEXT",
    "created_at": "TIMESTAMP DEFAULT NOW()",
    "updated_at": "TIMESTAMP DEFAULT NOW()"
  }
}


⸻

3.13 events

Analytics / telemetry (matches ZeroDB “event tracking” use case).

Schema

{
  "name": "events",
  "description": "Product analytics events",
  "schema": {
    "id": "UUID PRIMARY KEY",
    "event_type": "TEXT NOT NULL",
    "data": "JSONB NOT NULL",
    "timestamp": "TIMESTAMP DEFAULT NOW()"
  }
}


⸻

4) Embeddings Documents (ZeroDB Embed-and-Store)

Use:
POST /v1/public/{project_id}/embeddings/embed-and-store

4.1 Embed Intent Profile (optional but powerful)

namespace: intent_profiles

documents[] example

{
  "id": "intent_{intent_profile_id}",
  "text": "Freelancer offering AI consulting to fintech startups in India; prefers friendly tone; outreach via email and LinkedIn; avoid spam; wants founders/CTOs.",
  "metadata": {
    "type": "intent_profile",
    "user_id": "{user_id}",
    "intent_profile_id": "{intent_profile_id}"
  }
}

4.2 Embed Company Research (recommended)

namespace: company_research

{
  "id": "research_{company_research_id}",
  "text": "Company: Acme Fintech. Summary: ... Pain hypothesis: ... Hook: ... Notable signals: hiring for backend, recent feature launch.",
  "metadata": {
    "type": "company_research",
    "lead_id": "{lead_id}",
    "campaign_id": "{campaign_id}",
    "domain": "acme.com",
    "industry": "fintech"
  }
}

4.3 Embed Outreach Drafts (recommended)

namespace: outreach_drafts

{
  "id": "draft_{draft_id}",
  "text": "Subject: ... Body: ... Personalization notes: ...",
  "metadata": {
    "type": "outreach_draft",
    "draft_id": "{draft_id}",
    "lead_id": "{lead_id}",
    "channel": "email",
    "approved": false
  }
}

4.4 Embed Replies (learning loop)

namespace: replies

{
  "id": "reply_{outcome_id}",
  "text": "Thanks—reach out next quarter; we just hired a contractor.",
  "metadata": {
    "type": "reply",
    "lead_id": "{lead_id}",
    "send_id": "{send_id}",
    "label": "NOT_NOW"
  }
}


⸻

5) Retrieval Patterns (Embeddings Search)

Use:
POST /v1/public/{project_id}/embeddings/search

5.1 “Find similar companies we had success with”

namespace: company_research
Filter by reply_label stored in metadata if you copy it there (optional).

{
  "query": "B2B SaaS fintech startup hiring backend engineers, early stage, needs AI automation",
  "top_k": 10,
  "namespace": "company_research",
  "similarity_threshold": 0.7
}

5.2 “Reuse best outreach drafts for this scenario”

namespace: outreach_drafts

{
  "query": "Friendly email to founder referencing product launch and offering AI workflow automation",
  "top_k": 5,
  "namespace": "outreach_drafts",
  "similarity_threshold": 0.75
}


⸻

6) How This Maps to the PRD Workflow (Data-Truth)

Step A — Create intent
	•	Insert into intent_profiles table
	•	Embed into intent_profiles namespace (optional)

Step B — Discovery run
	•	Log run in agent_runs
	•	Create leads in leads
	•	Store rationale as embeddings in lead_rationales (optional)

Step C — Qualification
	•	Update leads.qualification_score/label
	•	Log run in agent_runs

Step D — Research
	•	Insert company_research row
	•	Embed research in company_research namespace
	•	Save embedding doc id back into company_research.embedding_doc_id

Step E — Drafting
	•	Insert outreach_drafts
	•	Embed draft in outreach_drafts namespace
	•	Save embedding doc id to outreach_drafts.embedding_doc_id

Step F — Approval + send
	•	Update outreach_drafts.approved_by_user=true
	•	Insert outreach_sends
	•	Later: insert outreach_outcomes when reply arrives

Step G — Learning
	•	Embed replies in replies namespace
	•	Use events table for analytics

⸻

7) Minimal Create-Order (So You Don’t Break Yourself)

Create tables in this order:
	1.	users
	2.	user_profiles
	3.	intent_profiles
	4.	campaigns
	5.	agent_runs
	6.	leads
	7.	lead_contacts
	8.	company_research
	9.	outreach_drafts
	10.	outreach_sends
	11.	outreach_outcomes
	12.	followups
	13.	events

⸻

8) ZeroDB Endpoint Mapping (Exactly)

Tables
	•	Create: POST /v1/public/{project_id}/database/tables
	•	Insert row: POST /v1/public/{project_id}/database/tables/{table}/rows with row_data
	•	Query rows: GET /v1/public/{project_id}/database/tables/{table}/rows?limit=...

Embeddings
	•	Embed+Store: POST /v1/public/{project_id}/embeddings/embed-and-store
	•	Search: POST /v1/public/{project_id}/embeddings/search

⸻

9) Critical Data Integrity Rules (Non-Negotiable)
	•	Deduplicate leads by domain per user_id (enforce in app logic; ZeroDB schema shown does not declare composite unique)
	•	Always store:
	•	lead_id in research/drafts/outcomes for joinability
	•	embedding_doc_id for each record that’s embedded
	•	Never mix embedding models in the same namespace

⸻

10) MVP Data Model (If You Want the shortest critical path)

If you want the leanest build that still works:
	•	Tables: users, user_profiles, intent_profiles, campaigns, leads, company_research, outreach_drafts, outreach_sends, outreach_outcomes, events
	•	Namespaces: company_research, outreach_drafts, replies

Everything else can be added after first traction.

⸻

End: Data Model aligned to ZeroDB Tables + Embeddings, with namespaces + PRD mapping

