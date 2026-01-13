# ZeroDB Initialization Guide

This guide explains how to create a ZeroDB project and initialize all database tables for the Reachout Agent MVP.

---

## Overview

ZeroDB is the primary NoSQL database for Reachout Agent. Before you can run the application, you must:

1. Create a ZeroDB project
2. Enable the database feature for your project
3. Create all 13 MVP tables in the correct order
4. Verify tables were created successfully

**IMPORTANT**: Complete this guide BEFORE running `/ralph-story` or starting the FastAPI backend.

---

## Prerequisites

- ZeroDB account (via AI-Native Studio or https://zerodb.ai)
- curl installed (for API calls)
- Your AI-Native API key (see [SETUP.md](./SETUP.md))

---

## Step 1: Create ZeroDB Project

### Option A: Via API (Recommended)

```bash
# Set your API key as environment variable
export API_KEY="your_ai_native_api_key_here"

# Create project
curl -X POST https://api.zerodb.ai/v1/public/projects \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "project_name": "reachout-agent-mvp",
    "description": "AI-Native Outreach Copilot MVP Database",
    "settings": {
      "environment": "development"
    }
  }'
```

Expected response:
```json
{
  "project_id": "proj_abc123xyz456",
  "project_name": "reachout-agent-mvp",
  "created_at": "2026-01-13T10:00:00Z",
  "status": "active"
}
```

**Save the `project_id`** - you'll need it for all subsequent operations.

### Option B: Via Dashboard

1. Log in to AI-Native Studio or ZeroDB dashboard
2. Click "Create New Project"
3. Enter project name: "reachout-agent-mvp"
4. Copy the generated `project_id`

---

## Step 2: Enable Database Features

```bash
# Replace <project_id> with your actual project ID
PROJECT_ID="proj_abc123xyz456"

# Enable database, vector, and file features
curl -X POST https://api.zerodb.ai/v1/public/projects/$PROJECT_ID/features \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "features": ["database", "vectors", "files", "events"]
  }'
```

Expected response:
```json
{
  "project_id": "proj_abc123xyz456",
  "features_enabled": ["database", "vectors", "files", "events"],
  "status": "success"
}
```

---

## Step 3: Update .env File

Add the project credentials to your `.env` file:

```bash
# Open .env file
nano .env  # or your preferred editor

# Update these lines:
ZERODB_PROJECT_ID=proj_abc123xyz456
ZERODB_API_KEY=your_api_key_here
```

Save and close the file.

---

## Step 4: Create All Database Tables

You must create tables in the correct order to respect foreign key relationships.

### 4.1 Create Users Table (No dependencies)

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "users",
    "schema": {
      "id": {"type": "uuid"},
      "email": {"type": "text"},
      "password_hash": {"type": "text"},
      "role": {"type": "text"},
      "status": {"type": "text"},
      "created_at": {"type": "timestamp"},
      "updated_at": {"type": "timestamp"}
    }
  }'
```

**CRITICAL**: Notice the schema format:
- ✅ CORRECT: `{"field": {"type": "uuid"}}`
- ❌ WRONG: `{"field": "UUID PRIMARY KEY"}`

ZeroDB does NOT support SQL DDL syntax. All constraints must be enforced at the application level.

### 4.2 Create User Profiles Table (Depends on: users)

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "user_profiles",
    "schema": {
      "id": {"type": "uuid"},
      "user_id": {"type": "uuid"},
      "company_name": {"type": "text"},
      "industry": {"type": "text"},
      "job_title": {"type": "text"},
      "tone_preference": {"type": "text"},
      "outreach_goals": {"type": "text"},
      "linkedin_profile_url": {"type": "text"},
      "twitter_handle": {"type": "text"},
      "created_at": {"type": "timestamp"},
      "updated_at": {"type": "timestamp"}
    }
  }'
```

### 4.3 Create Intent Profiles Table (Depends on: users)

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "intent_profiles",
    "schema": {
      "id": {"type": "uuid"},
      "user_id": {"type": "uuid"},
      "intent_description": {"type": "text"},
      "target_persona": {"type": "text"},
      "value_proposition": {"type": "text"},
      "example_companies": {"type": "jsonb"},
      "embedding_doc_id": {"type": "text"},
      "created_at": {"type": "timestamp"},
      "updated_at": {"type": "timestamp"}
    }
  }'
```

### 4.4 Create Campaigns Table (Depends on: users, intent_profiles)

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "campaigns",
    "schema": {
      "id": {"type": "uuid"},
      "user_id": {"type": "uuid"},
      "intent_profile_id": {"type": "uuid"},
      "name": {"type": "text"},
      "target_description": {"type": "text"},
      "status": {"type": "text"},
      "autonomy_level": {"type": "text"},
      "messaging_guidelines": {"type": "text"},
      "started_at": {"type": "timestamp"},
      "ended_at": {"type": "timestamp"},
      "created_at": {"type": "timestamp"},
      "updated_at": {"type": "timestamp"}
    }
  }'
```

### 4.5 Create Agent Runs Table (Depends on: users, campaigns)

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "agent_runs",
    "schema": {
      "id": {"type": "uuid"},
      "campaign_id": {"type": "uuid"},
      "agent_name": {"type": "text"},
      "run_type": {"type": "text"},
      "input_data": {"type": "jsonb"},
      "output_data": {"type": "jsonb"},
      "status": {"type": "text"},
      "started_at": {"type": "timestamp"},
      "ended_at": {"type": "timestamp"},
      "error_message": {"type": "text"}
    }
  }'
```

### 4.6 Create Leads Table (Depends on: campaigns)

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "leads",
    "schema": {
      "id": {"type": "uuid"},
      "campaign_id": {"type": "uuid"},
      "company_name": {"type": "text"},
      "company_website": {"type": "text"},
      "industry": {"type": "text"},
      "company_size": {"type": "text"},
      "location": {"type": "text"},
      "qualification_label": {"type": "text"},
      "status": {"type": "text"},
      "created_at": {"type": "timestamp"},
      "updated_at": {"type": "timestamp"}
    }
  }'
```

### 4.7 Create Lead Contacts Table (Depends on: leads)

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "lead_contacts",
    "schema": {
      "id": {"type": "uuid"},
      "lead_id": {"type": "uuid"},
      "contact_name": {"type": "text"},
      "contact_title": {"type": "text"},
      "email": {"type": "text"},
      "linkedin_url": {"type": "text"},
      "phone": {"type": "text"},
      "is_primary": {"type": "boolean"},
      "created_at": {"type": "timestamp"},
      "updated_at": {"type": "timestamp"}
    }
  }'
```

### 4.8 Create Company Research Table (Depends on: leads)

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "company_research",
    "schema": {
      "id": {"type": "uuid"},
      "lead_id": {"type": "uuid"},
      "research_summary": {"type": "text"},
      "key_findings": {"type": "jsonb"},
      "pain_points": {"type": "jsonb"},
      "recent_news": {"type": "jsonb"},
      "embedding_doc_id": {"type": "text"},
      "researched_at": {"type": "timestamp"},
      "created_at": {"type": "timestamp"}
    }
  }'
```

### 4.9 Create Outreach Drafts Table (Depends on: leads, company_research)

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "outreach_drafts",
    "schema": {
      "id": {"type": "uuid"},
      "lead_id": {"type": "uuid"},
      "research_id": {"type": "uuid"},
      "subject_line": {"type": "text"},
      "email_body": {"type": "text"},
      "channel": {"type": "text"},
      "rationale": {"type": "text"},
      "embedding_doc_id": {"type": "text"},
      "status": {"type": "text"},
      "drafted_at": {"type": "timestamp"},
      "approved_at": {"type": "timestamp"},
      "created_at": {"type": "timestamp"},
      "updated_at": {"type": "timestamp"}
    }
  }'
```

### 4.10 Create Outreach Sends Table (Depends on: outreach_drafts)

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "outreach_sends",
    "schema": {
      "id": {"type": "uuid"},
      "draft_id": {"type": "uuid"},
      "lead_contact_id": {"type": "uuid"},
      "sent_at": {"type": "timestamp"},
      "channel": {"type": "text"},
      "status": {"type": "text"},
      "external_message_id": {"type": "text"},
      "error_message": {"type": "text"}
    }
  }'
```

### 4.11 Create Outreach Outcomes Table (Depends on: outreach_sends)

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "outreach_outcomes",
    "schema": {
      "id": {"type": "uuid"},
      "send_id": {"type": "uuid"},
      "reply_received_at": {"type": "timestamp"},
      "reply_body": {"type": "text"},
      "reply_label": {"type": "text"},
      "embedding_doc_id": {"type": "text"},
      "next_action": {"type": "text"},
      "created_at": {"type": "timestamp"}
    }
  }'
```

### 4.12 Create Followups Table (Depends on: outreach_outcomes)

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "followups",
    "schema": {
      "id": {"type": "uuid"},
      "outcome_id": {"type": "uuid"},
      "lead_id": {"type": "uuid"},
      "followup_type": {"type": "text"},
      "message_body": {"type": "text"},
      "scheduled_for": {"type": "timestamp"},
      "sent_at": {"type": "timestamp"},
      "status": {"type": "text"},
      "created_at": {"type": "timestamp"}
    }
  }'
```

### 4.13 Create Events Table (No dependencies)

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "events",
    "schema": {
      "id": {"type": "uuid"},
      "event_type": {"type": "text"},
      "entity_id": {"type": "uuid"},
      "entity_type": {"type": "text"},
      "event_data": {"type": "jsonb"},
      "created_at": {"type": "timestamp"}
    }
  }'
```

---

## Step 5: Verify Tables Were Created

### List all tables:

```bash
curl -X GET https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables \
  -H "Authorization: Bearer $API_KEY"
```

Expected response (truncated):
```json
{
  "tables": [
    {"name": "users", "created_at": "2026-01-13T10:05:00Z"},
    {"name": "user_profiles", "created_at": "2026-01-13T10:06:00Z"},
    {"name": "intent_profiles", "created_at": "2026-01-13T10:07:00Z"},
    {"name": "campaigns", "created_at": "2026-01-13T10:08:00Z"},
    ... (9 more tables)
  ],
  "total_count": 13
}
```

**Verify**: You should see exactly **13 tables**.

### Verify a specific table schema:

```bash
curl -X GET https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables/users \
  -H "Authorization: Bearer $API_KEY"
```

Expected response:
```json
{
  "name": "users",
  "schema": {
    "id": {"type": "uuid"},
    "email": {"type": "text"},
    "password_hash": {"type": "text"},
    "role": {"type": "text"},
    "status": {"type": "text"},
    "created_at": {"type": "timestamp"},
    "updated_at": {"type": "timestamp"}
  },
  "row_count": 0
}
```

### Test inserting a row:

```bash
curl -X POST https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables/users/rows \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "row_data": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "test@example.com",
      "password_hash": "hashed_password_here",
      "role": "USER",
      "status": "ACTIVE",
      "created_at": "2026-01-13T10:00:00Z",
      "updated_at": "2026-01-13T10:00:00Z"
    }
  }'
```

**CRITICAL**: Notice the request format:
- ✅ CORRECT: `{"row_data": {...}}`
- ❌ WRONG: `{"data": {...}}` or `{"rows": [...]}`

### Retrieve the inserted row:

```bash
curl -X GET "https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables/users/rows?limit=1" \
  -H "Authorization: Bearer $API_KEY"
```

Expected response:
```json
{
  "rows": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "test@example.com",
      "role": "USER",
      "status": "ACTIVE",
      ...
    }
  ],
  "total_count": 1
}
```

If you see the test user, your ZeroDB setup is complete!

### Clean up test data:

```bash
# Delete the test row
curl -X DELETE "https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables/users/rows/550e8400-e29b-41d4-a716-446655440000" \
  -H "Authorization: Bearer $API_KEY"
```

---

## Step 6: Initialize Embedding Namespaces (Optional for MVP)

The MVP uses 4 embedding namespaces. These will be created automatically when you first store embeddings, but you can pre-create them if desired:

1. **intent_profiles** (optional namespace)
2. **company_research** (mandatory namespace)
3. **outreach_drafts** (mandatory namespace)
4. **replies** (recommended namespace)

No explicit API call is needed - ZeroDB creates namespaces on first use.

---

## Automated Table Creation Script (Optional)

Instead of manually running curl commands, you can create a Python script:

```python
# scripts/init_zerodb.py
import os
import httpx
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv("ZERODB_API_KEY")
PROJECT_ID = os.getenv("ZERODB_PROJECT_ID")
BASE_URL = "https://api.zerodb.ai/v1/public"

# Table schemas from docs/datamodel.md
TABLES = [
    {
        "name": "users",
        "schema": {
            "id": {"type": "uuid"},
            "email": {"type": "text"},
            # ... rest of schema
        }
    },
    # ... all 13 tables
]

def create_tables():
    for table in TABLES:
        response = httpx.post(
            f"{BASE_URL}/{PROJECT_ID}/database/tables",
            headers={"Authorization": f"Bearer {API_KEY}"},
            json=table
        )
        if response.status_code == 200:
            print(f"✓ Created table: {table['name']}")
        else:
            print(f"✗ Failed to create {table['name']}: {response.text}")

if __name__ == "__main__":
    create_tables()
```

Run with:
```bash
python scripts/init_zerodb.py
```

---

## Common Issues & Troubleshooting

### Issue 1: "401 Unauthorized"

**Cause**: Invalid or missing API key

**Solution**:
1. Verify `ZERODB_API_KEY` in `.env` is correct
2. Test authentication:
   ```bash
   curl -X GET https://api.zerodb.ai/v1/public/projects \
     -H "Authorization: Bearer $API_KEY"
   ```

### Issue 2: "400 Bad Request: Invalid schema format"

**Cause**: Using SQL DDL syntax instead of ZeroDB JSON format

**Solution**: Ensure your schema uses `{"field": {"type": "uuid"}}` format, NOT `"field": "UUID PRIMARY KEY"`

### Issue 3: "409 Conflict: Table already exists"

**Cause**: Table was already created in a previous run

**Solution**: Either skip this table or delete and recreate:
```bash
# Delete table
curl -X DELETE https://api.zerodb.ai/v1/public/$PROJECT_ID/database/tables/users \
  -H "Authorization: Bearer $API_KEY"

# Recreate table (re-run creation curl command)
```

### Issue 4: "Cannot insert row: missing required field"

**Cause**: ZeroDB doesn't support NOT NULL constraints in schema, so all fields are optional

**Solution**: Enforce required fields at application level in Pydantic models:
```python
# backend/models/user.py
class UserCreate(BaseModel):
    email: str  # Required in Pydantic, not enforced by ZeroDB
    password: str
```

### Issue 5: "Rate limit exceeded"

**Cause**: Too many API calls in a short time

**Solution**: Add delays between table creation calls:
```bash
# In your script, add sleep between calls
sleep 1
```

---

## Next Steps

Once all tables are created and verified:

1. ✅ Update `.env` with `ZERODB_PROJECT_ID` and `ZERODB_API_KEY`
2. ✅ Return to [SETUP.md](./SETUP.md) and continue with Step 6
3. ✅ Run the FastAPI backend
4. ✅ Start implementing stories with `/ralph-story`

---

## References

- **Data Model**: [datamodel.md](./datamodel.md) - Full table schemas and relationships
- **API Reference**: [API-REFERENCE.md](./API-REFERENCE.md) - Authentication and endpoint details
- **Setup Guide**: [SETUP.md](./SETUP.md) - Full local development setup
- **ZeroDB Documentation**: https://docs.zerodb.ai (if available)

---

## Notes

- **Table creation order matters** due to foreign key references (enforced at app level)
- **NO SQL constraints** are supported in ZeroDB schemas (see datamodel.md section 9.1)
- **Embedding namespaces** are created automatically on first embedding store
- **All field types** must use ZeroDB JSON format: `{"type": "uuid"}`, `{"type": "text"}`, etc.
- **Application-level validation** is required for uniqueness, NOT NULL, defaults, and foreign keys
