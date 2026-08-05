# AccrediAI — System Architecture (v1.0)

**Status:** Frozen for v1.0 build. Any change to this document during Days 3-10 requires flagging why the original design was insufficient, per standing rules.

## 1. Finalized Tech Stack

| Layer | Choice |
|---|---|
| Frontend | Next.js 15 (App Router), TypeScript, Tailwind CSS, shadcn/ui |
| Backend | Next.js API Routes, Service Layer Architecture |
| API Style | REST |
| Database | PostgreSQL (Supabase) |
| ORM | Prisma |
| Auth | Supabase Auth + JWT |
| Authorization | Role-Based Access Control (custom, DB-backed) |
| File Storage | Supabase Storage |
| AI Layer | Provider-abstracted; Claude Sonnet as default implementation |
| Document Processing | pdf-parse (PDF), mammoth (DOCX) |
| Prompt Management | Modular prompt template files |
| Validation | Zod |
| State Management | TanStack Query + React Context |
| Search | PostgreSQL Full-Text Search |
| Charts | Recharts |
| Deployment | Vercel |
| CI/CD | GitHub + Vercel Git Integration |

**Explicitly deferred to post-v1.0:** OCR (Tesseract), Sentry, Swagger/OpenAPI, Playwright/Vitest test suites, Husky/lint-staged.

---

## 2. Component Diagram

```mermaid
graph TB
    subgraph Client["Browser (Next.js Frontend)"]
        UI[React UI - shadcn/ui + Tailwind]
        TQ[TanStack Query Cache]
    end

    subgraph Vercel["Vercel — Next.js App"]
        API[API Routes /app/api/*]
        SVC[Service Layer]
        AIAB[AI Provider Abstraction]
        DOC[Document Processing<br/>pdf-parse / mammoth]
        SCORE[Scoring Engine]
    end

    subgraph Supabase["Supabase"]
        AUTH[Supabase Auth]
        DB[(PostgreSQL + RLS)]
        STORE[Supabase Storage]
    end

    subgraph External["External Services"]
        CLAUDE[Anthropic Claude API]
    end

    UI --> TQ --> API
    API --> SVC
    SVC --> AIAB --> CLAUDE
    SVC --> DOC
    SVC --> SCORE
    SVC --> DB
    SVC --> STORE
    API --> AUTH
    AUTH --> DB
```

**Why this shape:** One deployable unit (Vercel) hosts frontend + API + service layer + AI orchestration, avoiding a separate backend service to deploy/monitor. Supabase supplies auth, DB, and storage as one managed platform — minimizing the number of external accounts/configs needed in a 10-day build. Claude is the only true external API dependency.

---

## 3. Data Flow — Core Evidence Workflow

```mermaid
sequenceDiagram
    participant U as Coordinator (Browser)
    participant API as Next.js API Route
    participant SVC as Service Layer
    participant STORE as Supabase Storage
    participant DOC as Document Processor
    participant AI as AI Provider (Claude)
    participant DB as PostgreSQL

    U->>API: POST /api/documents/upload (file)
    API->>STORE: store raw file
    API->>DB: create Document row (status: queued)
    API-->>U: 202 Accepted (document id)

    API->>SVC: process(documentId)
    SVC->>DOC: extract text/tables/metadata
    DOC-->>SVC: extracted content
    SVC->>AI: classify(content, framework context)
    AI-->>SVC: {criterion, keyIndicator, metric, confidence, reasoning}
    SVC->>AI: validate(content, metricRequirements)
    AI-->>SVC: {issues, duplicateFlag, completeness}
    SVC->>DB: store classification + validation (status: pending_approval)

    U->>API: GET /api/documents/:id (poll or view)
    API->>DB: fetch status + AI output
    API-->>U: render Approval Screen

    U->>API: POST /api/documents/:id/approve (edits, if any)
    API->>DB: commit final mapping (status: approved)
    API->>SVC: recalculateScores(institutionId, metricId)
    SVC->>DB: update Metric/KeyIndicator/Criterion/Institution scores
    API-->>U: updated Readiness Dashboard
```

---

## 4. Request Lifecycle (Typical Authenticated Request)

```mermaid
flowchart LR
    A[Client Request] --> B{Supabase JWT valid?}
    B -- No --> C[401 Unauthorized]
    B -- Yes --> D{RBAC: role has permission?}
    D -- No --> E[403 Forbidden]
    D -- Yes --> F{institution_id matches resource?}
    F -- No --> G[403 Forbidden - tenant isolation]
    F -- Yes --> H[Zod validates request body]
    H -- Invalid --> I[400 Bad Request]
    H -- Valid --> J[Service Layer executes]
    J --> K[Response formatted + returned]
```

Every API route follows this exact order: **authenticate → authorize (role) → authorize (tenant) → validate input → execute → respond.** This order is fixed across all endpoints in API.md.

---

## 5. AI Interaction Detail

```mermaid
flowchart TD
    A[Extracted Document Content] --> B[Load Framework Context<br/>Criterion 2/3/6 Key Indicators + Metrics]
    B --> C[Classification Prompt Template]
    C --> D[AI Provider Abstraction: callAI]
    D --> E[Claude Sonnet]
    E --> F{Valid JSON +<br/>known Metric ID?}
    F -- No --> G[Retry once with correction prompt]
    G --> F
    F -- Yes --> H[Validation Prompt Template]
    H --> D
    D --> I[Recommendation Prompt Template]
    I --> D
    D --> J[Structured Result:<br/>classification + validation + recommendations]
```

**AI Provider Abstraction contract:**
```ts
interface AIProvider {
  complete(promptTemplate: string, variables: Record<string, unknown>): Promise<string>;
}
// claudeProvider.ts implements this today.
// A future openAIProvider.ts or geminiProvider.ts can implement the same interface without touching Service Layer code.
```

---

## 6. External Services

| Service | Purpose | Failure Handling |
|---|---|---|
| Anthropic Claude API | Classification, validation, recommendations, Q&A, report drafting | Retry once on malformed JSON; on repeated failure, mark document `needs_review` rather than blocking upload |
| Supabase Auth | Login/session/JWT | Standard Supabase error surfaces as 401 |
| Supabase Storage | Evidence file storage | Upload failure surfaces as 502 with retry option in UI |
| Supabase Postgres | All persistent data | Connection errors surface as 503; no direct client-side DB access — always via API routes |

---

## 7. Multi-Tenancy Approach

Every table with institution-owned data carries `institution_id`. Two enforcement layers:
1. **Application layer:** Service Layer functions always take `institutionId` from the authenticated session (never from client input) and scope every query with it.
2. **Database layer:** Postgres Row-Level Security (RLS) policies on Supabase enforce `institution_id = current_setting('app.institution_id')` as a second, defense-in-depth layer — so even a bug in application code cannot leak cross-tenant data.

This satisfies the PRD's "strict data isolation" requirement with two independent safeguards, appropriate for an "enterprise-grade" positioning.
