# AccrediAI — API Design (v1.0)

REST API, all routes under `/api/*` (Next.js API Routes). All responses are JSON. All authenticated routes require a valid Supabase JWT in the `Authorization: Bearer <token>` header, and are additionally scoped to the caller's `institution_id`. Request lifecycle for every route below: **authenticate → authorize (role) → authorize (tenant) → validate (Zod) → execute → respond** (see ARCHITECTURE.md §4).

Standard error shape:
```json
{ "error": { "code": "STRING_CODE", "message": "Human readable message" } }
```

---

## Auth & Onboarding

### `POST /api/auth/register`
- **Purpose:** Create a new institution + its first user (IQAC Coordinator).
- **Request:** `{ institutionName, email, password, fullName }`
- **Response:** `201 { institutionId, userId, token }`
- **Validation:** email format, password min length 8, institutionName non-empty.
- **Auth:** None (public).
- **Errors:** `400` invalid input · `409` email already used for an institution.

### `POST /api/auth/login`
- **Purpose:** Authenticate existing user.
- **Request:** `{ email, password }`
- **Response:** `200 { token, user: { id, role, institutionId } }`
- **Auth:** None (public).
- **Errors:** `401` invalid credentials.

### `POST /api/users/invite`
- **Purpose:** IQAC Coordinator invites a new user with an assigned role.
- **Request:** `{ email, fullName, roleId, criterionIds? }`
- **Response:** `201 { userId, inviteStatus }`
- **Auth:** Role = IQAC Coordinator only.
- **Errors:** `403` insufficient role · `409` user already exists in institution.

---

## Institutions & Users

### `GET /api/institution`
- **Purpose:** Fetch current institution profile + summary.
- **Response:** `200 { id, name, naacCycle, userCount, overallReadinessScore }`
- **Auth:** Any authenticated role.

### `GET /api/users`
- **Purpose:** List users in the institution (admin/management view).
- **Response:** `200 { users: [{ id, fullName, email, role, criteria? }] }`
- **Auth:** IQAC Coordinator, Institution Admin.

---

## Framework Reference (Criteria/Key Indicators/Metrics)

### `GET /api/framework/criteria`
- **Purpose:** List all 7 Criteria with structural + AI-support metadata.
- **Response:** `200 { criteria: [{ id, code, name, weightage, aiFullySupported }] }`
- **Auth:** Any authenticated role.

### `GET /api/framework/criteria/:id`
- **Purpose:** Get Key Indicators + Metrics for one Criterion.
- **Response:** `200 { criterion, keyIndicators: [{ id, code, name, metrics: [...] }] }`
- **Auth:** Any authenticated role. **Errors:** `404` unknown criterion id.

---

## Documents & Evidence Workflow

### `POST /api/documents/upload`
- **Purpose:** Upload one or more evidence files; stores file and queues processing.
- **Request:** `multipart/form-data` — `files[]`, optional `criterionHint`.
- **Response:** `202 { documents: [{ id, fileName, status: "queued" }] }`
- **Validation:** file type ∈ {pdf, docx, image, excel}; max size (e.g. 20MB); at least 1 file.
- **Auth:** IQAC Coordinator, Criterion Coordinator, Faculty (own uploads only).
- **Errors:** `400` unsupported file type/too large · `413` payload too large.

### `GET /api/documents/:id`
- **Purpose:** Get processing status and, once ready, AI classification/validation/recommendations for review.
- **Response:** `200 { id, status, extractedMetadata, classification: {criterion, keyIndicator, metric, confidence, reasoning}, validation: {flags}, recommendations: [...] }`
- **Auth:** Uploader, or any role scoped to that document's Criterion/institution.
- **Errors:** `404` not found · `403` out of tenant/role scope.

### `GET /api/documents`
- **Purpose:** List documents, filterable by criterion, status, uploader.
- **Query params:** `?criterionId=&status=&page=`
- **Response:** `200 { documents: [...], total, page }`
- **Auth:** Any authenticated role (results scoped by role — Criterion Coordinator sees only assigned Criteria).

### `POST /api/documents/:id/approve`
- **Purpose:** Human-in-the-loop approval; commits (possibly edited) classification to the knowledge base and triggers score recalculation.
- **Request:** `{ criterionId, keyIndicatorId, metricId, edited: boolean }`
- **Response:** `200 { documentId, status: "approved", updatedScores: {...} }`
- **Auth:** IQAC Coordinator, Criterion Coordinator (scoped to their Criterion).
- **Errors:** `400` metric doesn't belong to specified Key Indicator · `409` already approved.

### `POST /api/documents/:id/reject`
- **Purpose:** Reject AI classification without committing it (e.g. wrong document entirely).
- **Request:** `{ reason }`
- **Response:** `200 { documentId, status: "rejected" }`
- **Auth:** Same as approve.

---

## Readiness Scores

### `GET /api/readiness/institution`
- **Purpose:** Institution-level overall readiness score + trend.
- **Response:** `200 { currentScore, trend: [{ date, score }], criteria: [{ id, score, aiFullySupported }] }`
- **Auth:** Any authenticated role.

### `GET /api/readiness/criteria/:id`
- **Purpose:** Criterion-level score with drill-down to Key Indicators.
- **Response:** `200 { criterionId, score, keyIndicators: [{ id, score, metrics: [{ id, score, status, gapSummary }] }] }`
- **Auth:** Any authenticated role. **Errors:** `404` unknown criterion.

### `GET /api/readiness/metrics/:id/explain`
- **Purpose:** Full explainability drill-down for one Metric — the "why this score" view.
- **Response:** `200 { metricId, score, contributingDocuments: [...], gaps: [...], recommendedActions: [...] }`
- **Auth:** Any authenticated role.

---

## Recommendations

### `GET /api/recommendations`
- **Purpose:** List open (unresolved) recommendations, optionally filtered by Criterion.
- **Query params:** `?criterionId=&resolved=false`
- **Response:** `200 { recommendations: [{ id, metricId, text, resolved }] }`
- **Auth:** IQAC Coordinator, Criterion Coordinator.

---

## AI Q&A Assistant

### `POST /api/ask`
- **Purpose:** Conversational Q&A over the institution's approved evidence (supported Criteria only).
- **Request:** `{ question, conversationId? }`
- **Response:** `200 { answer, citedDocuments: [{ id, fileName, excerpt }], conversationId }`
- **Auth:** IQAC Coordinator, Criterion Coordinator, Principal (read-only).
- **Errors:** `422` question out of scope (handled gracefully in answer text, not a hard error, per PRD's "grounded refusal" requirement).

---

## Reports (AQAR/SSR Draft Generation)

### `POST /api/reports/generate`
- **Purpose:** Generate a draft AQAR/SSR narrative section for a given Criterion/Key Indicator.
- **Request:** `{ criterionId, keyIndicatorId? }`
- **Response:** `202 { reportJobId }` (async — generation may take a few seconds)

### `GET /api/reports/:jobId`
- **Purpose:** Poll/fetch generated report draft.
- **Response:** `200 { status, content, exportUrl? }`
- **Auth:** IQAC Coordinator, Criterion Coordinator.
- **Errors:** `404` job not found · content only covers `ai_fully_supported` Criteria — `400` if requested for an out-of-scope Criterion.

---

## Analytics

### `GET /api/analytics/summary`
- **Purpose:** Focused analytics — readiness trend + gap distribution by Criterion.
- **Response:** `200 { readinessTrend: [...], gapsByCriterion: [{ criterionId, gapCount }] }`
- **Auth:** IQAC Coordinator, Principal.

---

## Cross-Cutting Rules (apply to every endpoint above)

- **Authentication:** Missing/invalid JWT → `401 { error: { code: "UNAUTHENTICATED" } }`.
- **Authorization (role):** Role lacks permission → `403 { error: { code: "FORBIDDEN_ROLE" } }`.
- **Authorization (tenant):** Resource belongs to another institution → `403 { error: { code: "FORBIDDEN_TENANT" } }` (never a `404`, to keep behavior consistent, but message stays generic to avoid leaking existence of the resource).
- **Validation:** Zod schema failure → `400 { error: { code: "VALIDATION_ERROR", message, fields } }`.
- **Server errors:** Unhandled failure → `500 { error: { code: "INTERNAL_ERROR" } }`, logged server-side with request id.
