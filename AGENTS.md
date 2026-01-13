# Agent Knowledge Base

This file contains important context for AI agents working on this codebase.

## Project Overview

**Reachout Agent** is an AI-Native Outreach Copilot for freelancers and solo founders. It automates business discovery, research, and personalized message generation using AI agents and ZeroDB for data persistence.

**Architecture**:
- Backend: FastAPI + AI-Native Studio APIs
- Data Layer: ZeroDB (tables + embeddings)
- Agents: Discovery, Enrichment, Qualification, Drafting, Learning
- External Tools: Web search MCPs, Firecrawl scraper

## Key Patterns & Conventions

### ZeroDB Schema Rules (CRITICAL)
- **NEVER use SQL DDL syntax** (no PRIMARY KEY, UNIQUE, NOT NULL, DEFAULT)
- Schema format: `{"field_name": {"type": "uuid"}}` NOT `"field_name": "UUID PRIMARY KEY"`
- All constraints must be enforced at application level
- Use `row_data` parameter for inserts (NOT `data` or `rows`)

### Data Model
- 13 tables: users, user_profiles, intent_profiles, campaigns, agent_runs, leads, lead_contacts, company_research, outreach_drafts, outreach_sends, outreach_outcomes, followups, events
- 4 embedding namespaces: intent_profiles (optional), company_research (mandatory), outreach_drafts (mandatory), replies (recommended)
- Default embedding model: `BAAI/bge-small-en-v1.5` (384-dim)
- Always track `embedding_doc_id` in tables for cross-reference

### Naming Conventions
- Files: snake_case (user_profile.py)
- Classes: PascalCase (UserProfile)
- Functions: snake_case (create_campaign)
- Constants: UPPER_SNAKE_CASE (DEFAULT_LIMIT)
- API routes: kebab-case (/api/v1/intent-profiles)

## Gotchas & Common Mistakes

### ZeroDB Pitfalls (MUST AVOID)
1. **SQL syntax in schemas** - Always use JSON format: `{"type": "text"}`
2. **Forgetting default values** - Set all defaults in app code before insert
3. **Not checking uniqueness** - Query before insert for unique fields
4. **Missing embedding_doc_id** - Always store after embedding
5. **Mixing embedding models** - Use same model per namespace

### Application-Level Constraints
- Check `users.email` uniqueness before insert
- Check `leads.(user_id, domain)` composite uniqueness
- Validate required fields: email, full_name, company_name, etc.
- Set defaults: role='USER', status='ACTIVE', created_at=now()
- Verify foreign keys exist before insert

## Testing Approach

### Test Structure
```
backend/
├── tests/
│   ├── test_auth.py
│   ├── test_intent.py
│   ├── test_leads.py
│   ├── test_research_agent.py
│   ├── test_drafting_agent.py
│   └── conftest.py  # Fixtures: test DB, mock clients
```

### Every Story Must Include
- [ ] API endpoint implemented and documented
- [ ] Pydantic models defined with validation
- [ ] Unit tests pass (pytest)
- [ ] Manual curl test succeeds
- [ ] ZeroDB data persists correctly
- [ ] Logged in agent_runs table (if agent involved)

## Dependencies & Setup

### Core Dependencies (to be installed)
- FastAPI - Web framework
- Pydantic - Data validation
- pytest - Testing
- httpx - HTTP client for external APIs
- python-dotenv - Environment variables

### ZeroDB API Endpoints
- Tables: `POST /v1/public/{project_id}/database/tables`
- Insert: `POST /v1/public/{project_id}/database/tables/{table}/rows`
- Query: `GET /v1/public/{project_id}/database/tables/{table}/rows`
- Embed: `POST /v1/public/{project_id}/embeddings/embed-and-store`
- Search: `POST /v1/public/{project_id}/embeddings/search`

### External Tool Integration
- **Discovery**: Open-WebSearch or Brave Search MCP
- **Scraping**: Firecrawl or web scraper MCP
- **Sending**: Agent Bridge API to email/LinkedIn providers

## Field Value Enums (App-Level Enforcement)

### User & Profile
- `users.role`: USER, ADMIN
- `users.status`: ACTIVE, INACTIVE, SUSPENDED
- `user_profiles.tone_preference`: friendly, professional, casual, formal

### Campaign
- `campaigns.status`: DRAFT, ACTIVE, PAUSED, COMPLETED, ARCHIVED
- `campaigns.autonomy_level`: DRAFT_ONLY, ASSISTED, AUTONOMOUS

### Lead Lifecycle
- `leads.qualification_label`: UNQUALIFIED, LOW, MEDIUM, HIGH
- `leads.status`: NEW, RESEARCHING, QUALIFIED, DRAFTING, SENT, REPLIED, CLOSED

### Agent Runs
- `agent_runs.run_type`: DISCOVERY, ENRICHMENT, QUALIFICATION, DRAFTING, LEARNING, FOLLOW_UP
- `agent_runs.status`: QUEUED, RUNNING, SUCCESS, FAILED, TIMEOUT

### Outreach
- `outreach_drafts.status`: DRAFT, APPROVED, REJECTED, SENT
- `outreach_drafts.channel`: email, linkedin
- `outreach_sends.status`: SENT, DELIVERED, BOUNCED, FAILED
- `outreach_outcomes.reply_label`: POSITIVE, INTERESTED, NOT_NOW, NOT_INTERESTED, UNSUBSCRIBE, SPAM
- `followups.status`: SCHEDULED, SENT, CANCELLED

## Documentation References

- **PRD**: docs/prd.md - Full architecture and requirements
- **Data Model**: docs/datamodel.md - All table schemas and embedding strategy
- **Backlog**: docs/backlog.md - Epic-based user stories
- **Sprint Plan**: docs/sprintplan.md - 1-day MVP build order

---

## Frontend Patterns (Next.js 14+ App Router)

### Technology Stack
- **Framework**: Next.js 14+ with App Router (NOT Pages Router)
- **Language**: TypeScript with strict mode
- **Styling**: Tailwind CSS + shadcn/ui components
- **State Management**: TanStack Query (React Query) for server state
- **Form Handling**: react-hook-form + zod validation
- **Authentication**: JWT tokens (from backend story-004)

### Directory Structure
```
frontend/
├── app/                    # Next.js App Router pages
│   ├── layout.tsx         # Root layout with providers
│   ├── page.tsx           # Home page
│   ├── login/page.tsx     # Auth pages
│   ├── profile/page.tsx   # User profile
│   ├── intents/           # Intent management
│   └── campaigns/         # Campaign workflow
│       └── [id]/          # Dynamic routes
│           ├── leads/
│           └── drafts/
├── components/
│   ├── ui/                # shadcn/ui components
│   ├── auth/              # AuthProvider, LoginForm
│   ├── campaigns/         # CampaignCard, Dashboard
│   ├── leads/             # LeadCard, ResearchView
│   └── drafts/            # DraftCard, ApprovalFlow
├── lib/
│   ├── api-client.ts      # Centralized API calls with auth
│   ├── query-client.ts    # React Query config
│   └── utils.ts           # shadcn/ui cn() utility
├── hooks/
│   ├── useAuth.ts         # Authentication hook
│   ├── useCampaigns.ts    # Campaign queries/mutations
│   ├── useLeads.ts        # Lead queries/mutations
│   └── useDrafts.ts       # Draft queries/mutations
└── types/
    └── api.ts             # TypeScript types for API responses
```

### API Client Pattern (CRITICAL)

**All API calls must use centralized client**:

```typescript
// lib/api-client.ts
const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';

function getAuthToken(): string | null {
  return localStorage.getItem('jwt_token'); // or cookie
}

export const apiClient = {
  get: async <T>(endpoint: string): Promise<T> => {
    const token = getAuthToken();
    const response = await fetch(`${API_BASE_URL}${endpoint}`, {
      headers: {
        'Authorization': token ? `Bearer ${token}` : '',
        'Content-Type': 'application/json',
      },
    });
    if (!response.ok) throw new Error(await response.text());
    return response.json();
  },

  post: async <T>(endpoint: string, data: any): Promise<T> => {
    const token = getAuthToken();
    const response = await fetch(`${API_BASE_URL}${endpoint}`, {
      method: 'POST',
      headers: {
        'Authorization': token ? `Bearer ${token}` : '',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(data),
    });
    if (!response.ok) throw new Error(await response.text());
    return response.json();
  },

  // ... patch, delete methods
};
```

### React Query Setup

**Root layout provider**:

```typescript
// app/layout.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      refetchOnWindowFocus: false,
    },
  },
});

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <QueryClientProvider client={queryClient}>
          <AuthProvider>{children}</AuthProvider>
        </QueryClientProvider>
      </body>
    </html>
  );
}
```

### Custom Hooks Pattern

**Example: useCampaigns hook**:

```typescript
// hooks/useCampaigns.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { apiClient } from '@/lib/api-client';

export function useCampaigns() {
  return useQuery({
    queryKey: ['campaigns'],
    queryFn: () => apiClient.get('/api/v1/campaigns'),
  });
}

export function useCreateCampaign() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: CampaignCreate) =>
      apiClient.post('/api/v1/campaigns', data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['campaigns'] });
    },
  });
}
```

### Authentication Pattern

**AuthProvider context**:

```typescript
// components/auth/AuthProvider.tsx
'use client';

import { createContext, useContext, useState, useEffect } from 'react';

interface AuthContext {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  isAuthenticated: boolean;
}

const AuthContext = createContext<AuthContext>(null!);

export function AuthProvider({ children }) {
  const [user, setUser] = useState<User | null>(null);

  const login = async (email: string, password: string) => {
    const { access_token, user } = await apiClient.post('/api/v1/auth/login', { email, password });
    localStorage.setItem('jwt_token', access_token);
    setUser(user);
  };

  const logout = () => {
    localStorage.removeItem('jwt_token');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, isAuthenticated: !!user }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => useContext(AuthContext);
```

### Form Validation Pattern

**Using react-hook-form + zod**:

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';

const campaignSchema = z.object({
  name: z.string().min(3, 'Name must be at least 3 characters'),
  intent_profile_id: z.string().uuid('Invalid intent profile'),
  autonomy_level: z.enum(['DRAFT_ONLY', 'ASSISTED', 'AUTONOMOUS']),
});

type CampaignForm = z.infer<typeof campaignSchema>;

function CampaignForm() {
  const form = useForm<CampaignForm>({
    resolver: zodResolver(campaignSchema),
    defaultValues: { autonomy_level: 'DRAFT_ONLY' },
  });

  const createMutation = useCreateCampaign();

  const onSubmit = async (data: CampaignForm) => {
    await createMutation.mutateAsync(data);
    toast({ title: 'Campaign created' });
  };

  return <form onSubmit={form.handleSubmit(onSubmit)}>...</form>;
}
```

### TypeScript Types (Match Backend Models)

```typescript
// types/api.ts
export interface User {
  id: string;
  email: string;
  role: 'USER' | 'ADMIN';
  status: 'ACTIVE' | 'INACTIVE' | 'SUSPENDED';
  created_at: string;
}

export interface Campaign {
  id: string;
  user_id: string;
  name: string;
  status: 'DRAFT' | 'ACTIVE' | 'PAUSED' | 'COMPLETED' | 'ARCHIVED';
  autonomy_level: 'DRAFT_ONLY' | 'ASSISTED' | 'AUTONOMOUS';
  created_at: string;
  updated_at: string;
}

export interface Lead {
  id: string;
  campaign_id: string;
  company_name: string;
  qualification_label: 'UNQUALIFIED' | 'LOW' | 'MEDIUM' | 'HIGH';
  status: 'NEW' | 'RESEARCHING' | 'QUALIFIED' | 'DRAFTING' | 'SENT' | 'REPLIED' | 'CLOSED';
}
```

### shadcn/ui Component Usage

**Install components as needed**:
```bash
npx shadcn-ui@latest add button
npx shadcn-ui@latest add form
npx shadcn-ui@latest add input
npx shadcn-ui@latest add card
npx shadcn-ui@latest add toast
npx shadcn-ui@latest add dialog
npx shadcn-ui@latest add select
npx shadcn-ui@latest add badge
```

**Use consistently**:
- Button for all actions
- Form + Input for all form fields
- Card for content containers
- Toast for notifications
- Dialog for modals
- Select for dropdowns
- Badge for status indicators

### Error Handling Pattern

**Global error boundary**:

```typescript
// app/error.tsx
'use client';

export default function Error({ error, reset }: {
  error: Error;
  reset: () => void;
}) {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen">
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

**React Query error handling**:

```typescript
const { data, error, isLoading } = useCampaigns();

if (error) {
  toast({
    title: 'Error loading campaigns',
    description: error.message,
    variant: 'destructive'
  });
}
```

### Frontend Gotchas (MUST AVOID)

1. **Not using 'use client' directive** - Components with useState, useEffect, event handlers need it
2. **Mixing Pages Router and App Router** - Use only App Router
3. **Not invalidating queries** - Always call `queryClient.invalidateQueries()` after mutations
4. **Forgetting loading states** - Show skeletons or spinners while fetching
5. **Hardcoding API URLs** - Always use `NEXT_PUBLIC_API_URL` env variable
6. **Not handling 401 errors** - Redirect to login on authentication failure
7. **Missing type safety** - Define TypeScript interfaces for all API responses
8. **Optimistic updates without rollback** - Handle mutation failures gracefully

### Every Frontend Story Must Include

- [ ] Page/component implemented in correct directory
- [ ] TypeScript types defined for all API data
- [ ] React Query hooks created for API calls
- [ ] Form validation using react-hook-form + zod
- [ ] Loading states (skeleton or spinner)
- [ ] Error handling (toast notifications)
- [ ] Accessible forms (ARIA labels, keyboard nav)
- [ ] Mobile responsive (Tailwind responsive classes)
- [ ] shadcn/ui components used consistently

### Frontend Environment Variables

Create `frontend/.env.example`:
```
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_ENV=development
```

---

**Last Updated**: 2026-01-13 (Frontend stories added)
