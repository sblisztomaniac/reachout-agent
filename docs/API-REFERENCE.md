# API Reference Guide

This document provides comprehensive reference for all external APIs used by Reachout Agent, including authentication methods, request/response formats, and example curl commands.

---

## Table of Contents

1. [Authentication](#authentication)
2. [ZeroDB API](#zerodb-api)
3. [Agent Orchestration API](#agent-orchestration-api)
4. [Agent Bridge API](#agent-bridge-api)
5. [Embeddings API](#embeddings-api)
6. [External Tools](#external-tools)
7. [Error Handling](#error-handling)
8. [Rate Limiting](#rate-limiting)

---

## Authentication

### Bearer Token Authentication

All AI-Native Studio APIs (ZeroDB, Agent Orchestration, Agent Bridge, Embeddings) use Bearer token authentication.

**Header Format**:
```
Authorization: Bearer <your_api_key>
Content-Type: application/json
```

**Example**:
```bash
curl -X GET https://api.zerodb.ai/v1/public/projects \
  -H "Authorization: Bearer sk_live_abc123xyz456" \
  -H "Content-Type: application/json"
```

### API Key Storage

Store API keys securely in `.env` file:
```
AI_NATIVE_API_KEY=sk_live_abc123xyz456
ZERODB_API_KEY=sk_live_abc123xyz456  # Same key for all AI-Native services
AGENT_BRIDGE_API_KEY=sk_live_abc123xyz456
```

---

## ZeroDB API

Base URL: `https://api.zerodb.ai/v1/public`

### 1. Create Project

```bash
POST /projects
```

**Request**:
```bash
curl -X POST https://api.zerodb.ai/v1/public/projects \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "project_name": "reachout-agent-mvp",
    "description": "AI-Native Outreach Copilot",
    "settings": {"environment": "development"}
  }'
```

**Response** (200 OK):
```json
{
  "project_id": "proj_abc123xyz456",
  "project_name": "reachout-agent-mvp",
  "created_at": "2026-01-13T10:00:00Z",
  "status": "active"
}
```

---

### 2. Enable Database Features

```bash
POST /projects/{project_id}/features
```

**Request**:
```bash
curl -X POST https://api.zerodb.ai/v1/public/projects/proj_abc123/features \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "features": ["database", "vectors", "files", "events"]
  }'
```

**Response** (200 OK):
```json
{
  "project_id": "proj_abc123",
  "features_enabled": ["database", "vectors", "files", "events"],
  "status": "success"
}
```

---

### 3. Create Table

```bash
POST /v1/public/{project_id}/database/tables
```

**Request**:
```bash
curl -X POST https://api.zerodb.ai/v1/public/proj_abc123/database/tables \
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

**Response** (200 OK):
```json
{
  "table_name": "users",
  "created_at": "2026-01-13T10:05:00Z",
  "status": "created"
}
```

**Field Types**:
- `uuid` - UUID/GUID
- `text` - String (any length)
- `integer` - Integer number
- `float` - Floating point number
- `boolean` - True/false
- `timestamp` - ISO 8601 timestamp
- `jsonb` - JSON object/array

---

### 4. Insert Row

```bash
POST /v1/public/{project_id}/database/tables/{table_name}/rows
```

**Request**:
```bash
curl -X POST https://api.zerodb.ai/v1/public/proj_abc123/database/tables/users/rows \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "row_data": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "user@example.com",
      "password_hash": "hashed_pw",
      "role": "USER",
      "status": "ACTIVE",
      "created_at": "2026-01-13T10:00:00Z",
      "updated_at": "2026-01-13T10:00:00Z"
    }
  }'
```

**Response** (200 OK):
```json
{
  "row_id": "550e8400-e29b-41d4-a716-446655440000",
  "created_at": "2026-01-13T10:00:00Z"
}
```

**CRITICAL**: Use `row_data` parameter, NOT `data` or `rows`.

---

### 5. Query Rows

```bash
GET /v1/public/{project_id}/database/tables/{table_name}/rows
```

**Request (all rows)**:
```bash
curl -X GET "https://api.zerodb.ai/v1/public/proj_abc123/database/tables/users/rows?limit=100" \
  -H "Authorization: Bearer $API_KEY"
```

**Request (with filter)**:
```bash
curl -X POST https://api.zerodb.ai/v1/public/proj_abc123/database/tables/users/rows/query \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "filter": {"email": "user@example.com"},
    "limit": 10
  }'
```

**Response** (200 OK):
```json
{
  "rows": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "user@example.com",
      "role": "USER",
      "status": "ACTIVE",
      ...
    }
  ],
  "total_count": 1
}
```

---

### 6. Update Row

```bash
PUT /v1/public/{project_id}/database/tables/{table_name}/rows/{row_id}
```

**Request**:
```bash
curl -X PUT https://api.zerodb.ai/v1/public/proj_abc123/database/tables/users/rows/550e8400-e29b-41d4-a716-446655440000 \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "update": {
      "status": "INACTIVE",
      "updated_at": "2026-01-13T11:00:00Z"
    }
  }'
```

**Response** (200 OK):
```json
{
  "row_id": "550e8400-e29b-41d4-a716-446655440000",
  "updated_at": "2026-01-13T11:00:00Z"
}
```

---

### 7. Delete Row

```bash
DELETE /v1/public/{project_id}/database/tables/{table_name}/rows/{row_id}
```

**Request**:
```bash
curl -X DELETE https://api.zerodb.ai/v1/public/proj_abc123/database/tables/users/rows/550e8400-e29b-41d4-a716-446655440000 \
  -H "Authorization: Bearer $API_KEY"
```

**Response** (200 OK):
```json
{
  "deleted_row_id": "550e8400-e29b-41d4-a716-446655440000",
  "deleted_at": "2026-01-13T12:00:00Z"
}
```

---

## Agent Orchestration API

Base URL: `https://api.ai-native.io/v1/public/agent-orchestration`

Used for running autonomous agents (research agent, drafting agent, etc.).

### 1. Run Agent

```bash
POST /v1/public/agent-orchestration/
```

**Request (Research Agent)**:
```bash
curl -X POST https://api.ai-native.io/v1/public/agent-orchestration/ \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_definition": {
      "name": "company_research_agent",
      "role": "Research company background and pain points",
      "goal": "Generate comprehensive research summary for lead qualification",
      "tools": ["web_search_mcp", "firecrawl", "zerodb_client"]
    },
    "input": {
      "company_name": "Acme Corp",
      "company_website": "https://acmecorp.com",
      "industry": "SaaS",
      "intent_profile_id": "uuid-of-intent"
    },
    "timeout": 300
  }'
```

**Response** (202 Accepted):
```json
{
  "agent_run_id": "run_xyz789",
  "status": "QUEUED",
  "created_at": "2026-01-13T10:00:00Z"
}
```

---

### 2. Get Agent Run Status

```bash
GET /v1/public/agent-orchestration/{agent_run_id}
```

**Request**:
```bash
curl -X GET https://api.ai-native.io/v1/public/agent-orchestration/run_xyz789 \
  -H "Authorization: Bearer $API_KEY"
```

**Response (In Progress)** (200 OK):
```json
{
  "agent_run_id": "run_xyz789",
  "status": "RUNNING",
  "started_at": "2026-01-13T10:00:05Z",
  "progress": "Researching company website..."
}
```

**Response (Completed)** (200 OK):
```json
{
  "agent_run_id": "run_xyz789",
  "status": "COMPLETED",
  "started_at": "2026-01-13T10:00:05Z",
  "ended_at": "2026-01-13T10:02:30Z",
  "output": {
    "research_summary": "Acme Corp is a B2B SaaS company...",
    "key_findings": ["Pain point 1", "Pain point 2"],
    "pain_points": ["Struggling with X", "Need solution for Y"],
    "recent_news": ["Recently raised Series A", "Launched new product"]
  }
}
```

**Response (Failed)** (200 OK):
```json
{
  "agent_run_id": "run_xyz789",
  "status": "FAILED",
  "started_at": "2026-01-13T10:00:05Z",
  "ended_at": "2026-01-13T10:05:00Z",
  "error_message": "TIMEOUT: Agent exceeded 300 second timeout"
}
```

---

### 3. Run Drafting Agent

**Request (Drafting Agent)**:
```bash
curl -X POST https://api.ai-native.io/v1/public/agent-orchestration/ \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_definition": {
      "name": "outreach_drafting_agent",
      "role": "Draft personalized outreach message",
      "goal": "Create compelling email based on research and user profile",
      "tools": ["zerodb_client"]
    },
    "input": {
      "lead_id": "uuid-of-lead",
      "research_summary": "Acme Corp is struggling with...",
      "user_profile": {
        "tone_preference": "professional",
        "value_proposition": "We help companies reduce churn by 30%"
      },
      "messaging_guidelines": "Keep it under 150 words, focus on pain points"
    },
    "timeout": 180
  }'
```

**Response (Completed)**:
```json
{
  "agent_run_id": "run_abc456",
  "status": "COMPLETED",
  "output": {
    "subject_line": "Reducing churn at Acme Corp",
    "email_body": "Hi [Name],\n\nI noticed Acme Corp recently...",
    "rationale": "Focused on their Series A funding and product launch pain points"
  }
}
```

---

## Agent Bridge API

Base URL: `https://api.ai-native.io/v1/public/agent-bridge`

Used for sending emails/messages through configured email provider.

### 1. Send Email

```bash
POST /v1/public/agent-bridge/message
```

**Request**:
```bash
curl -X POST https://api.ai-native.io/v1/public/agent-bridge/message \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "channel": "email",
    "provider": "sendgrid",
    "to": {
      "email": "contact@acmecorp.com",
      "name": "John Doe"
    },
    "from": {
      "email": "noreply@yourdomain.com",
      "name": "Your Company"
    },
    "subject": "Reducing churn at Acme Corp",
    "body": "Hi John,\n\nI noticed Acme Corp recently...",
    "metadata": {
      "draft_id": "uuid-of-draft",
      "campaign_id": "uuid-of-campaign"
    }
  }'
```

**Response** (200 OK):
```json
{
  "message_id": "msg_xyz123",
  "external_message_id": "sendgrid_id_abc",
  "status": "SENT",
  "sent_at": "2026-01-13T10:15:00Z",
  "provider": "sendgrid"
}
```

---

### 2. Get Message Status

```bash
GET /v1/public/agent-bridge/message/{message_id}
```

**Request**:
```bash
curl -X GET https://api.ai-native.io/v1/public/agent-bridge/message/msg_xyz123 \
  -H "Authorization: Bearer $API_KEY"
```

**Response** (200 OK):
```json
{
  "message_id": "msg_xyz123",
  "status": "DELIVERED",
  "sent_at": "2026-01-13T10:15:00Z",
  "delivered_at": "2026-01-13T10:15:05Z",
  "opened_at": "2026-01-13T10:30:00Z",
  "clicked_at": null,
  "replied_at": null
}
```

---

### 3. Supported Email Providers

Configure in `.env` file:

**SendGrid**:
```
SENDGRID_API_KEY=your_key
SENDGRID_FROM_EMAIL=noreply@yourdomain.com
```

**Resend**:
```
RESEND_API_KEY=your_key
RESEND_FROM_EMAIL=noreply@yourdomain.com
```

**SMTP** (generic):
```
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your_email
SMTP_PASSWORD=your_password
```

---

## Embeddings API

Base URL: `https://api.ai-native.io/v1/public/{project_id}/embeddings`

Used for storing and searching vector embeddings.

### 1. Embed and Store

```bash
POST /v1/public/{project_id}/embeddings/embed-and-store
```

**Request**:
```bash
curl -X POST https://api.ai-native.io/v1/public/proj_abc123/embeddings/embed-and-store \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "namespace": "company_research",
    "document": "Acme Corp is a B2B SaaS company struggling with customer churn. They recently raised Series A funding...",
    "metadata": {
      "lead_id": "uuid-of-lead",
      "research_id": "uuid-of-research",
      "company_name": "Acme Corp"
    },
    "model": "BAAI/bge-small-en-v1.5"
  }'
```

**Response** (200 OK):
```json
{
  "embedding_doc_id": "emb_xyz789",
  "namespace": "company_research",
  "model": "BAAI/bge-small-en-v1.5",
  "dimensions": 384,
  "created_at": "2026-01-13T10:20:00Z"
}
```

**CRITICAL**: Save `embedding_doc_id` to your table for future reference.

---

### 2. Search Embeddings

```bash
POST /v1/public/{project_id}/embeddings/search
```

**Request**:
```bash
curl -X POST https://api.ai-native.io/v1/public/proj_abc123/embeddings/search \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "namespace": "company_research",
    "query": "SaaS companies with churn problems",
    "limit": 5,
    "threshold": 0.7,
    "filter_metadata": {
      "industry": "SaaS"
    },
    "model": "BAAI/bge-small-en-v1.5"
  }'
```

**Response** (200 OK):
```json
{
  "results": [
    {
      "embedding_doc_id": "emb_xyz789",
      "similarity_score": 0.89,
      "document": "Acme Corp is a B2B SaaS company struggling with...",
      "metadata": {
        "lead_id": "uuid-of-lead",
        "company_name": "Acme Corp"
      }
    },
    {
      "embedding_doc_id": "emb_abc456",
      "similarity_score": 0.82,
      "document": "TechStart is a SaaS startup facing...",
      "metadata": {
        "lead_id": "uuid-of-lead-2",
        "company_name": "TechStart"
      }
    }
  ],
  "total_count": 2
}
```

---

### 3. Embedding Namespaces

The MVP uses 4 namespaces:

1. **intent_profiles** (optional)
   - User's ideal customer profiles
   - Used for lead qualification scoring

2. **company_research** (mandatory)
   - Research summaries for each lead
   - Used for semantic search of similar companies

3. **outreach_drafts** (mandatory)
   - Email/message drafts
   - Used for finding similar past drafts

4. **replies** (recommended)
   - Reply bodies from leads
   - Used for sentiment analysis and classification

---

## External Tools

### 1. Firecrawl API

**Base URL**: `https://api.firecrawl.dev`

**Used For**: Scraping company websites for research

**Request**:
```bash
curl -X POST https://api.firecrawl.dev/v1/scrape \
  -H "Authorization: Bearer $FIRECRAWL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://acmecorp.com",
    "formats": ["markdown", "html"],
    "onlyMainContent": true
  }'
```

**Response**:
```json
{
  "success": true,
  "data": {
    "markdown": "# Acme Corp\n\nWe help...",
    "html": "<html>...</html>",
    "metadata": {
      "title": "Acme Corp - Homepage",
      "description": "We help companies..."
    }
  }
}
```

---

### 2. Web Search MCP

**Type**: Model Context Protocol (MCP) server

**Used For**: Web search during research

**Configuration**: Set `WEB_SEARCH_MCP_URL` in `.env`

**Example Usage** (via agent orchestration tool):
```python
# In agent tool definition
tools = [
    {
        "type": "mcp",
        "name": "web_search",
        "server_url": os.getenv("WEB_SEARCH_MCP_URL"),
        "method": "search",
        "parameters": {
            "query": "Acme Corp recent news",
            "num_results": 5
        }
    }
]
```

---

## Error Handling

### Standard Error Response Format

All APIs return errors in this format:

```json
{
  "error": "Error message description",
  "code": 400,
  "details": {
    "field": "email",
    "reason": "Email already exists"
  }
}
```

### Common HTTP Status Codes

| Code | Meaning | Retry? |
|------|---------|--------|
| 200 | Success | N/A |
| 201 | Created | N/A |
| 202 | Accepted (async) | Poll for status |
| 400 | Bad Request | Fix request, don't retry |
| 401 | Unauthorized | Check API key |
| 403 | Forbidden | Check permissions |
| 404 | Not Found | Check resource ID |
| 409 | Conflict (duplicate) | Don't retry |
| 429 | Rate Limit Exceeded | Retry with backoff |
| 500 | Server Error | Retry with backoff |
| 503 | Service Unavailable | Retry with backoff |
| 504 | Gateway Timeout | Retry with backoff |

### Retry Strategy

**Exponential Backoff**:
```python
import time
import httpx

def retry_with_backoff(func, max_retries=3):
    for attempt in range(max_retries):
        try:
            return func()
        except httpx.HTTPStatusError as e:
            if e.response.status_code in [429, 500, 503, 504]:
                wait_time = 2 ** attempt  # 1s, 2s, 4s
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

---

## Rate Limiting

### ZeroDB API

- **Limit**: 100 requests/minute per project
- **Header**: `X-RateLimit-Remaining` in response
- **Exceeded**: 429 status with retry-after header

### Agent Orchestration API

- **Limit**: 10 concurrent agents per project
- **Queue**: Additional requests are queued (202 status)
- **Timeout**: 300 seconds default (configurable)

### Embeddings API

- **Limit**: 1000 embed operations/hour per project
- **Batch**: Use batch endpoints for >10 embeddings

### Best Practices

1. **Cache responses** where possible
2. **Batch operations** instead of individual calls
3. **Use webhooks** instead of polling for async operations
4. **Implement exponential backoff** for retries
5. **Monitor rate limit headers** to avoid hitting limits

---

## Example: Complete Flow

### Scenario: Create user, run research agent, draft outreach

```bash
# 1. Create user
curl -X POST https://api.zerodb.ai/v1/public/proj_abc123/database/tables/users/rows \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"row_data": {"id": "uuid", "email": "user@example.com", ...}}'

# 2. Create lead
curl -X POST https://api.zerodb.ai/v1/public/proj_abc123/database/tables/leads/rows \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"row_data": {"id": "uuid", "company_name": "Acme Corp", ...}}'

# 3. Run research agent
curl -X POST https://api.ai-native.io/v1/public/agent-orchestration/ \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"agent_definition": {...}, "input": {...}}'
# Returns: {"agent_run_id": "run_xyz"}

# 4. Poll agent status
curl -X GET https://api.ai-native.io/v1/public/agent-orchestration/run_xyz \
  -H "Authorization: Bearer $API_KEY"
# Wait until status == "COMPLETED"

# 5. Store research in ZeroDB
curl -X POST https://api.zerodb.ai/v1/public/proj_abc123/database/tables/company_research/rows \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"row_data": {"id": "uuid", "lead_id": "uuid", "research_summary": "...", ...}}'

# 6. Embed research
curl -X POST https://api.ai-native.io/v1/public/proj_abc123/embeddings/embed-and-store \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"namespace": "company_research", "document": "...", ...}'
# Returns: {"embedding_doc_id": "emb_xyz"}

# 7. Update research with embedding_doc_id
curl -X PUT https://api.zerodb.ai/v1/public/proj_abc123/database/tables/company_research/rows/uuid \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"update": {"embedding_doc_id": "emb_xyz"}}'

# 8. Run drafting agent
curl -X POST https://api.ai-native.io/v1/public/agent-orchestration/ \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"agent_definition": {...}, "input": {...}}'
# Returns draft

# 9. Send email via Agent Bridge
curl -X POST https://api.ai-native.io/v1/public/agent-bridge/message \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"channel": "email", "to": {...}, "subject": "...", "body": "..."}'
```

---

## Additional Resources

- **ZeroDB Initialization**: [ZeroDB-INIT.md](./ZeroDB-INIT.md)
- **Local Setup Guide**: [SETUP.md](./SETUP.md)
- **Data Model**: [datamodel.md](./datamodel.md)
- **Product Requirements**: [prd.md](./prd.md)

---

## Support

For API issues:
- Check status page (if available)
- Review error response details
- Verify authentication headers
- Check rate limit headers
- Test with curl before implementing in code
