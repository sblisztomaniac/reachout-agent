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
| `intent_profiles` | User intent descriptions, constraints | Retrieve similar past intents (optional) |
| `company_research` | site/product summaries, hooks | Draft personalization (mandatory) |
| `outreach_drafts` | approved drafts + context | Reuse winning patterns (mandatory) |
| `replies` | inbound reply text + tags | Classification + learnings (recommended) |

Default embedding model for all namespaces: **`BAAI/bge-small-en-v1.5` (384-dim)**

---

## 3) Table Schema (ZeroDB Tables API)

> Below are the exact tables you should create via:
`POST /v1/public/{project_id}/database/tables`

### 3.1 Field Value Enums (App-Level Enforcement)

ZeroDB does not support SQL enums or check constraints. These are valid values enforced by application logic.

#### User and Profile Enums
- `users.role`: `USER`, `ADMIN`
- `users.status`: `ACTIVE`, `INACTIVE`, `SUSPENDED`
- `user_profiles.tone_preference`: `friendly`, `professional`, `casual`, `formal`

#### Campaign Enums
- `campaigns.status`: `DRAFT`, `ACTIVE`, `PAUSED`, `COMPLETED`, `ARCHIVED`
- `campaigns.autonomy_level`: `DRAFT_ONLY` (human approval required), `ASSISTED` (auto-send with approval), `AUTONOMOUS` (future)

#### Lead Lifecycle Enums
- `leads.qualification_label`: `UNQUALIFIED`, `LOW`, `MEDIUM`, `HIGH`
- `leads.status`: `NEW`, `RESEARCHING`, `QUALIFIED`, `DRAFTING`, `SENT`, `REPLIED`, `CLOSED`

#### Agent Run Enums
- `agent_runs.run_type`: `DISCOVERY`, `ENRICHMENT`, `QUALIFICATION`, `DRAFTING`, `LEARNING`, `FOLLOW_UP`
- `agent_runs.status`: `QUEUED`, `RUNNING`, `SUCCESS`, `FAILED`, `TIMEOUT`

#### Outreach Enums
- `outreach_drafts.status`: `DRAFT`, `APPROVED`, `REJECTED`, `SENT`
- `outreach_drafts.channel`: `email`, `linkedin`
- `outreach_sends.status`: `SENT`, `DELIVERED`, `BOUNCED`, `FAILED`
- `outreach_outcomes.reply_label`: `POSITIVE`, `INTERESTED`, `NOT_NOW`, `NOT_INTERESTED`, `UNSUBSCRIBE`, `SPAM`
- `followups.status`: `SCHEDULED`, `SENT`, `CANCELLED`

---

### 3.2 `users`
System-level user profile data (minimal duplication if auth exists elsewhere).

**Schema**
```json
{
  "name": "users",
  "description": "User accounts for OutreachCopilot",
  "schema": {
    "id": {"type": "uuid"},
    "email": {"type": "text"},
    "full_name": {"type": "text"},
    "role": {"type": "text"},
    "status": {"type": "text"},
    "created_at": {"type": "timestamp"},
    "updated_at": {"type": "timestamp"}
  }
}


⸻

3.3 user_profiles

Freelancer/Founder-specific business identity.

Schema

{
  "name": "user_profiles",
  "description": "Freelancer/founder profile for personalization and targeting",
  "schema": {
    "id": {"type": "uuid"},
    "user_id": {"type": "uuid"},
    "persona": {"type": "text"},
    "services": {"type": "jsonb"},
    "industries": {"type": "jsonb"},
    "geographies": {"type": "jsonb"},
    "pricing_note": {"type": "text"},
    "portfolio_url": {"type": "text"},
    "case_studies": {"type": "jsonb"},
    "tone_preference": {"type": "text"},
    "daily_outreach_limit": {"type": "integer"},
    "created_at": {"type": "timestamp"},
    "updated_at": {"type": "timestamp"}
  }
}


⸻

3.4 intent_profiles

Each “Outreach Intent” becomes a reusable target definition.

Schema

{
  "name": "intent_profiles",
  "description": "Structured outreach intent definitions",
  "schema": {
    "id": {"type": "uuid"},
    "user_id": {"type": "uuid"},
    "name": {"type": "text"},
    "intent_json": {"type": "jsonb"},
    "status": {"type": "text"},
    "created_at": {"type": "timestamp"},
    "updated_at": {"type": "timestamp"}
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

3.5 campaigns

A campaign is a run configuration tied to an intent.

Schema

{
  "name": "campaigns",
  "description": "Outreach campaign definition and lifecycle",
  "schema": {
    "id": {"type": "uuid"},
    "user_id": {"type": "uuid"},
    "intent_profile_id": {"type": "uuid"},
    "name": {"type": "text"},
    "status": {"type": "text"},
    "channels": {"type": "jsonb"},
    "autonomy_level": {"type": "text"},
    "start_at": {"type": "timestamp"},
    "end_at": {"type": "timestamp"},
    "created_at": {"type": "timestamp"},
    "updated_at": {"type": "timestamp"}
  }
}


⸻

3.6 agent_runs

Every orchestrated sequence/task execution should be logged.

Schema

{
  "name": "agent_runs",
  "description": "Logs for agent tasks and sequences executions",
  "schema": {
    "id": {"type": "uuid"},
    "user_id": {"type": "uuid"},
    "campaign_id": {"type": "uuid"},
    "agent_name": {"type": "text"},
    "run_type": {"type": "text"},
    "status": {"type": "text"},
    "input": {"type": "jsonb"},
    "output": {"type": "jsonb"},
    "error": {"type": "text"},
    "started_at": {"type": "timestamp"},
    "ended_at": {"type": "timestamp"},
    "created_at": {"type": "timestamp"}
  }
}


⸻

3.7 leads

Canonical lead record (one row per company per user, deduped by domain).

Schema

{
  "name": "leads",
  "description": "Canonical lead/company record",
  "schema": {
    "id": {"type": "uuid"},
    "user_id": {"type": "uuid"},
    "company_name": {"type": "text"},
    "domain": {"type": "text"},
    "website_url": {"type": "text"},
    "linkedin_url": {"type": "text"},
    "country": {"type": "text"},
    "industry": {"type": "text"},
    "company_size_estimate": {"type": "text"},
    "signals": {"type": "jsonb"},
    "qualification_score": {"type": "float"},
    "qualification_label": {"type": "text"},
    "status": {"type": "text"},
    "created_at": {"type": "timestamp"},
    "updated_at": {"type": "timestamp"}
  }
}


⸻

3.8 lead_contacts

Contacts discovered for a lead (people to reach out to).

Schema

{
  "name": "lead_contacts",
  "description": "Contacts within a lead/company",
  "schema": {
    "id": {"type": "uuid"},
    "lead_id": {"type": "uuid"},
    "name": {"type": "text"},
    "title": {"type": "text"},
    "email": {"type": "text"},
    "linkedin_url": {"type": "text"},
    "confidence": {"type": "float"},
    "source": {"type": "text"},
    "created_at": {"type": "timestamp"}
  }
}


⸻

3.9 company_research

Structured research output (and a pointer to the embedded doc).

Schema

{
  "name": "company_research",
  "description": "Research notes per company (structured + link to vector doc id)",
  "schema": {
    "id": {"type": "uuid"},
    "lead_id": {"type": "uuid"},
    "summary": {"type": "text"},
    "pain_hypothesis": {"type": "text"},
    "hook": {"type": "text"},
    "sources": {"type": "jsonb"},
    "embedding_doc_id": {"type": "text"},
    "created_at": {"type": "timestamp"},
    "updated_at": {"type": "timestamp"}
  }
}


⸻

3.10 outreach_drafts

Draft messages (email/LinkedIn) + review/approval state.

Schema

{
  "name": "outreach_drafts",
  "description": "Generated outreach drafts awaiting approval or sent",
  "schema": {
    "id": {"type": "uuid"},
    "campaign_id": {"type": "uuid"},
    "lead_id": {"type": "uuid"},
    "contact_id": {"type": "uuid"},
    "channel": {"type": "text"},
    "subject": {"type": "text"},
    "body": {"type": "text"},
    "status": {"type": "text"},
    "approved_by_user": {"type": "boolean"},
    "approved_at": {"type": "timestamp"},
    "embedding_doc_id": {"type": "text"},
    "created_at": {"type": "timestamp"},
    "updated_at": {"type": "timestamp"}
  }
}


⸻

3.11 outreach_sends

Send logs (what was actually sent, when, how).

Schema

{
  "name": "outreach_sends",
  "description": "Outreach send logs",
  "schema": {
    "id": {"type": "uuid"},
    "draft_id": {"type": "uuid"},
    "channel": {"type": "text"},
    "provider": {"type": "text"},
    "to_address": {"type": "text"},
    "external_message_id": {"type": "text"},
    "status": {"type": "text"},
    "sent_at": {"type": "timestamp"},
    "raw_payload": {"type": "jsonb"}
  }
}


⸻

3.12 outreach_outcomes

Outcome tracking + response classification.

Schema

{
  "name": "outreach_outcomes",
  "description": "Reply outcomes and classifications",
  "schema": {
    "id": {"type": "uuid"},
    "send_id": {"type": "uuid"},
    "lead_id": {"type": "uuid"},
    "reply_received": {"type": "boolean"},
    "reply_text": {"type": "text"},
    "reply_label": {"type": "text"},
    "reply_confidence": {"type": "float"},
    "next_action": {"type": "text"},
    "embedding_doc_id": {"type": "text"},
    "created_at": {"type": "timestamp"},
    "updated_at": {"type": "timestamp"}
  }
}


⸻

3.13 followups

Follow-up scheduling and execution.

Schema

{
  "name": "followups",
  "description": "Follow-up schedule + execution status",
  "schema": {
    "id": {"type": "uuid"},
    "campaign_id": {"type": "uuid"},
    "lead_id": {"type": "uuid"},
    "send_id": {"type": "uuid"},
    "due_at": {"type": "timestamp"},
    "status": {"type": "text"},
    "followup_body": {"type": "text"},
    "created_at": {"type": "timestamp"},
    "updated_at": {"type": "timestamp"}
  }
}


⸻

3.14 events

Analytics / telemetry (matches ZeroDB “event tracking” use case).

Schema

{
  "name": "events",
  "description": "Product analytics events",
  "schema": {
    "id": {"type": "uuid"},
    "event_type": {"type": "text"},
    "data": {"type": "jsonb"},
    "timestamp": {"type": "timestamp"}
  }
}


⸻

4) Embeddings Documents (ZeroDB Embed-and-Store)

Use:
POST /v1/public/{project_id}/embeddings/embed-and-store

### 4.1 Embedding Strategy Decision Tree

**When to Embed**:
1. **Company Research** (ALWAYS) - For similarity matching when drafting
2. **Outreach Drafts** (ALWAYS) - For pattern reuse and learning
3. **Replies** (RECOMMENDED) - For classification and sentiment analysis
4. **Intent Profiles** (OPTIONAL) - If using intent similarity for discovery

**When NOT to Embed**:
- Lead records themselves (too sparse, use structured filters)
- Send logs (no retrieval value)
- Agent run metadata (debug data only)

**Embedding Flow Pattern**:
1. Create structured record in table (get row ID)
2. Call `POST /embeddings/embed-and-store` with namespace
3. Extract `document.id` from response
4. Update table row with `embedding_doc_id` field
5. Use `embedding_doc_id` to correlate table records with vector docs

**Example: Company Research Flow**
```python
# 1. Insert structured data
research_row = insert_row(
  table="company_research",
  row_data={
    "id": research_id,
    "lead_id": lead_id,
    "summary": "...",
    "pain_hypothesis": "...",
    "hook": "..."
  }
)

# 2. Embed for retrieval
embed_response = embed_and_store(
  namespace="company_research",
  documents=[{
    "id": f"research_{research_id}",
    "text": f"{summary}\n{pain_hypothesis}\n{hook}",
    "metadata": {
      "lead_id": lead_id,
      "industry": industry
    }
  }]
)

# 3. Store embedding doc ID
update_row(
  table="company_research",
  row_id=research_id,
  row_data={"embedding_doc_id": f"research_{research_id}"}
)
```

### 4.2 Embed Intent Profile (Optional)

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

### 4.3 Embed Company Research (Mandatory)

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

### 4.4 Embed Outreach Drafts (Mandatory)

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

### 4.5 Embed Replies (Recommended)

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
	•	embedding_doc_id for each record that's embedded
	•	Never mix embedding models in the same namespace

⸻

### 9.1 Application-Level Constraint Enforcement

Since ZeroDB doesn't support SQL constraints, the backend must enforce:

#### Unique Constraints
- `users.email` - Check before insert/update
- `leads.(user_id, domain)` - Composite unique: Query existing before insert
  ```python
  existing = query_rows(
    table="leads",
    filter={"user_id": user_id, "domain": domain}
  )
  if existing: raise ConflictError("Lead already exists")
  ```

#### Not Null Constraints
Validate required fields before insert:
- `users`: email, full_name
- `campaigns`: user_id, intent_profile_id, name
- `leads`: user_id, company_name
- `outreach_drafts`: campaign_id, lead_id, channel, body

#### Default Values
Set in application code before insert:
- `users.role` → `"USER"`
- `users.status` → `"ACTIVE"`
- `user_profiles.tone_preference` → `"friendly"`
- `user_profiles.daily_outreach_limit` → `10`
- `campaigns.status` → `"DRAFT"`
- `campaigns.autonomy_level` → `"DRAFT_ONLY"`
- `leads.qualification_score` → `0.0`
- `leads.qualification_label` → `"UNQUALIFIED"`
- `leads.status` → `"NEW"`
- `outreach_drafts.approved_by_user` → `false`
- All `created_at` / `updated_at` → `datetime.now(UTC)`

#### Foreign Key Integrity
Verify references exist before insert:
- Before inserting `user_profiles`: Check `users.id` exists
- Before inserting `campaigns`: Check `user_id` and `intent_profile_id` exist
- Before inserting `leads`: Check `user_id` exists
- Before inserting `company_research`: Check `lead_id` exists

#### Cascade Deletes (Implement in Code)
When deleting:
- Delete user → Delete user_profiles, intent_profiles, campaigns (cascade)
- Delete campaign → Delete outreach_drafts, outreach_sends (cascade)
- Delete lead → Delete lead_contacts, company_research, outreach_drafts (cascade)

⸻

10) MVP Data Model (If You Want the shortest critical path)

If you want the leanest build that still works:
	•	Tables: users, user_profiles, intent_profiles, campaigns, leads, company_research, outreach_drafts, outreach_sends, outreach_outcomes, events
	•	Namespaces: company_research, outreach_drafts, replies

Everything else can be added after first traction.

⸻

End: Data Model aligned to ZeroDB Tables + Embeddings, with namespaces + PRD mapping

