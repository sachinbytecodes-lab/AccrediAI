# AccrediAI — Implementation Blueprint (Days 2–10)

**This document is the single source of truth for building AccrediAI's v1.0.**
Each day below is written so that a *fresh AI conversation*, given only this file (and the PRD), can continue the build without re-deciding architecture, scope, or priorities. Do not redesign or re-scope — only implement, adapt tactically, and hand off.

> Tech stack is intentionally **not chosen yet** — Day 2 begins with a short, scoped stack decision (not a redesign) based on the constraints below, then proceeds straight into setup.

---

## 0. Frozen Context (carry into every day)

- **Product:** AccrediAI — multi-tenant SaaS AI Accreditation Readiness Copilot for NAAC.
- **Core loop:** Upload evidence → extract → AI classify (Criterion/Key Indicator/Metric + confidence) → validate quality/completeness → detect gaps → recompute Readiness Score → recommend actions → human approval → knowledge base.
- **v1.0 AI depth:** Full AI intelligence on **Criterion 2 (Teaching-Learning & Evaluation), Criterion 3 (Research, Innovations & Extension), Criterion 6 (Governance, Leadership & Management)**. All 7 Criteria exist in UI/data model; the other 4 are structural only.
- **Roles fully built:** IQAC Coordinator, Criterion Coordinator. **Simplified:** Principal, Faculty Member. **Foundational only (auth + placeholder dashboard):** NAAC Coordinator, HOD, Data Entry Operator, Institution Admin.
- **Scoring hierarchy:** Metric → Key Indicator → Criterion → Institution, NAAC-weightage-inspired, always explainable (must show contributing evidence + gaps + next action).
- **Explicit boundary:** Never call it "the NAAC score" or imply grade prediction — always "readiness score, inspired by NAAC's framework."
- **File formats:** PDF & DOCX = fully robust. Images/Excel = best-effort. ZIP batch = stretch goal only.
- **Data:** Realistic synthetic evidence documents (built Day 2-3); architecture must not care whether data is synthetic or real.
- **Fallback trigger:** If behind schedule by end of Day 5, simplify Principal & Faculty dashboards further before cutting anything from the AI engine or the 2 primary roles.
- **Day 10 target:** Live public URL, polished, demoable end-to-end for AB Talks judges, IQAC teams, and a portfolio/LinkedIn audience.

**Standing rule for every day:** Prefer *fewer things fully working* over *more things partially working*. When in doubt, cut breadth, not depth, on the 🟢 core loop.

---

## Day 2 — Stack Decision, Architecture & Environment Setup

**🎯 Objective:** Lock the technical stack, design the system architecture and data model, and get a working local dev environment with a "hello world" full-stack skeleton deployed to a public staging URL.

**📖 What you'll learn:** Pragmatic stack selection under time pressure; multi-tenant schema design; early deployment (deploy early, deploy often).

**🛠 Features to build:**
- Repo scaffold (frontend + backend + database), environment config, CI-friendly structure.
- Multi-tenant database schema v1 (institutions, users, roles, criteria, key indicators, metrics, documents, evidence_scores).
- Auth skeleton (register/login, JWT or session-based).
- One deployed "Hello AccrediAI" page reachable via public URL.

**📝 Step-by-step implementation plan:**
1. Choose stack optimized for **speed + AI integration ease + free-tier deployability**: recommended default — Next.js (or React + Node/Express) frontend+API, PostgreSQL, an ORM (Prisma/SQLAlchemy), and a Python or Node service layer for AI/document processing. If you (the builder) are stronger in a different stack you already listed (Python/Java/TS), pick that instead — this is a 30-minute decision, not a redesign.
2. Initialize repo with two top-level folders: `/frontend`, `/backend` (or a single Next.js app with `/app` + `/lib/ai`, `/lib/db`).
3. Design and create core tables: `institutions`, `users`, `roles`, `user_roles`, `criteria`, `key_indicators`, `metrics`, `documents`, `document_evidence_map`, `readiness_scores`, `recommendations`, `audit_log`.
4. Seed `criteria`/`key_indicators`/`metrics` tables with NAAC's public framework (all 7 Criteria — full structural data even though only 3 get AI depth).
5. Build register/login endpoints + JWT/session middleware; enforce `institution_id` scoping on every query from day one (this is the multi-tenancy backbone — get it right now, not later).
6. Build a minimal landing/login page and a placeholder dashboard page.
7. Deploy immediately to a public URL (any platform with a generous free tier you're already comfortable with) — confirm the URL loads before ending the day.

**📂 Files/folders to create:**
- `/backend/src/db/schema.*`, `/backend/src/db/seed_naac_framework.*`
- `/backend/src/auth/*`
- `/frontend/app/(auth)/login`, `/frontend/app/(auth)/register`, `/frontend/app/dashboard`
- `/docs/ARCHITECTURE.md` (record your stack choice + schema diagram here — required for tomorrow's handoff)

**🔗 Integrations:** Database hosting (free tier), deployment platform (free tier), no AI APIs yet.

**🧪 Testing tasks:** Register 2 test institutions, confirm data isolation (institution A cannot see institution B's users), confirm login/logout works.

**🐞 Common issues:** CORS between frontend/backend; forgetting `institution_id` on a query (audit every table access before moving on); free-tier DB connection limits.

**✅ End-of-day checklist:**
- [ ] Stack decision recorded in `/docs/ARCHITECTURE.md`
- [ ] Schema created with all 7 Criteria seeded
- [ ] Register/login working for 2+ institutions with verified data isolation
- [ ] App deployed and reachable at a public URL

**📸 Capture:** Screenshot of deployed URL, register/login flow, DB schema diagram or table list.

**➡️ Handoff notes:** Record your public URL, stack, and DB connection approach at the top of `/docs/ARCHITECTURE.md`. Day 3 assumes auth + schema + deployment pipeline already work — it will not re-verify these unless something is broken.

---

## Day 3 — RBAC, Institution Onboarding & Synthetic Data

**🎯 Objective:** Build full role-based access control for all 8 roles, institution onboarding/user invitation flow, and generate the synthetic evidence document set used for the rest of the build.

**📖 What you'll learn:** RBAC design patterns; realistic test-data generation for document-AI systems.

**🛠 Features to build:**
- Role assignment during onboarding; permission-checking middleware.
- IQAC Coordinator can invite users and assign roles.
- Placeholder dashboards for all 8 roles (routes exist, minimal content) — full dashboards for the 2 primary roles come Day 4-5.
- Synthetic evidence document library (~25-40 documents) spanning Criteria 2, 3, 6.

**📝 Step-by-step implementation plan:**
1. Implement `roles` + `permissions` tables (or a simple role-to-capability map if time-constrained) covering all 8 roles from the PRD.
2. Build middleware that checks role + institution scope on every protected route.
3. Build "Invite user" flow for IQAC Coordinator (email/role assignment; can be a simple internal invite record if real email sending is out of scope).
4. Create one route per role that renders a distinct (even if minimal) dashboard shell, so RBAC is demonstrably real, not just backend-only.
5. Generate synthetic documents (use realistic Indian HEI context): for Criterion 2 — teaching plans, evaluation reports, feedback analysis, exam result summaries; for Criterion 3 — research papers, MoUs, patents, extension activity reports, consultancy records; for Criterion 6 — governance meeting minutes, audit reports, strategic plans, HR/administrative policies. Save as real PDF/DOCX files (not just text stubs) — this matters for Day 5's extraction pipeline.
6. Deliberately include some *weak* evidence (missing dates, unsigned, incomplete) in the synthetic set — you'll need these to demo gap detection convincingly on Day 6-7.

**📂 Files/folders:**
- `/backend/src/rbac/*`
- `/backend/src/onboarding/*`
- `/data/synthetic_evidence/criterion_2/*`, `/criterion_3/*`, `/criterion_6/*`
- `/docs/SYNTHETIC_DATA_MANIFEST.md` (list every file, its intended Criterion/Key Indicator/Metric, and whether it's "strong" or "weak" evidence — critical reference for Day 6 testing)

**🔗 Integrations:** None new; optionally a transactional email free-tier service if you want real invite emails (skip if time-constrained).

**🧪 Testing tasks:** Log in as each of the 8 roles and confirm correct route access/denial; confirm a Faculty user cannot access IQAC Coordinator-only endpoints.

**🐞 Common issues:** Permission checks only on frontend (must also enforce backend-side); synthetic documents too clean (won't demonstrate gap detection) — deliberately weaken some.

**✅ End-of-day checklist:**
- [ ] All 8 roles have working auth + at least a placeholder route
- [ ] Backend permission checks verified (not just UI hiding)
- [ ] 25-40 synthetic documents created and manifested, spanning strong and weak evidence

**📸 Capture:** Screenshots of 2-3 different role dashboards, the invite flow, and the synthetic data folder listing.

**➡️ Handoff notes:** `SYNTHETIC_DATA_MANIFEST.md` is required reading for Day 5 (extraction) and Day 6 (validation/scoring) — it defines ground truth for testing AI accuracy.

---

## Day 4 — IQAC Coordinator & Criterion Coordinator Dashboards (Full Build)

**🎯 Objective:** Build the two fully-functional role dashboards end-to-end (layout, navigation, empty states) so the core workflow has a home before AI logic is wired in.

**📖 What you'll learn:** Building enterprise dashboard UX; designing for a data-sparse "empty state" that still looks intentional.

**🛠 Features to build:**
- IQAC Coordinator dashboard: institution-wide readiness overview (all 7 Criteria cards), navigation to all Criteria, user management, upload entry point.
- Criterion Coordinator dashboard: scoped view of their assigned Criterion(s), upload entry point, evidence list.
- Criterion detail page (works for all 7, but only 2/3/6 will show live AI data starting Day 6).
- Upload UI (file picker, multi-file, progress indicator) — wired to a stub endpoint today (real processing starts Day 5).

**📝 Step-by-step implementation plan:**
1. Build shared layout shell: sidebar nav (7 Criteria + Dashboard + Search + Reports), top bar (institution name, user role, logout).
2. Build IQAC Coordinator home: 7 Criterion readiness cards (score placeholder "—" until Day 6-7 wires real numbers), overall institution score placeholder, recent activity feed placeholder.
3. Build Criterion Coordinator home: same card style but filtered to assigned Criterion(s) only; enforce assignment via a `criterion_coordinator_assignments` table.
4. Build Criterion detail page: Key Indicator list → Metric list → "no evidence yet" empty state with a clear call-to-action to upload.
5. Build Upload UI component: drag-and-drop + file picker, accepts PDF/DOCX (and image/Excel with a "best effort" note), shows per-file upload progress, posts to a stub `/api/documents/upload` endpoint that just stores the file today.
6. Design and implement genuinely good empty states — this is a portfolio-critical detail, not filler (see PRD non-functional requirement on UI polish).

**📂 Files/folders:**
- `/frontend/app/dashboard/iqac/*`
- `/frontend/app/dashboard/criterion/[id]/*`
- `/frontend/components/UploadWidget/*`, `/components/ReadinessCard/*`
- `/backend/src/documents/upload.stub.*`

**🔗 Integrations:** File storage (local disk or free-tier object storage) for uploaded files — set this up properly today since Day 5 builds directly on it.

**🧪 Testing tasks:** Upload a synthetic PDF and DOCX from Day 3's set, confirm they're stored and listed; confirm Criterion Coordinator sees only assigned Criteria; confirm empty states render correctly for Criteria with zero evidence.

**🐞 Common issues:** File size limits on free-tier storage/hosting; forgetting to scope Criterion Coordinator assignment server-side (not just filtering client-side).

**✅ End-of-day checklist:**
- [ ] IQAC Coordinator dashboard shows all 7 Criteria with placeholder scores
- [ ] Criterion Coordinator dashboard correctly scoped
- [ ] Upload UI works and stores real files
- [ ] Empty states designed, not default/broken

**📸 Capture:** Screenshots of both dashboards, a Criterion detail page empty state, upload UI mid-upload.

**➡️ Handoff notes:** File storage location/approach must be documented in `/docs/ARCHITECTURE.md` — Day 5's extraction pipeline reads directly from wherever Day 4 stored files.

---

## Day 5 — Document Processing & Extraction Pipeline

**🎯 Objective:** Replace the upload stub with a real extraction pipeline: text, tables, metadata pulled from PDF/DOCX evidence.

**📖 What you'll learn:** Practical document parsing (PDF/DOCX), metadata inference, building a resilient pipeline with partial-failure handling.

**🛠 Features to build:**
- Text extraction from PDF and DOCX.
- Table extraction where present (e.g., result summaries, attendance sheets).
- Metadata inference: title, date, department, organiser, authors/participants, inferred document type (certificate, MoU, report, etc.) — this can start as a lightweight heuristic/LLM-assisted pass; full AI classification confidence comes Day 6.
- Processing status shown in UI (queued → processing → done/failed).

**📝 Step-by-step implementation plan:**
1. Add PDF text/table extraction (e.g., a PDF parsing library) and DOCX extraction (e.g., a DOCX parsing library); wrap both behind one internal `extractDocument(file)` interface so downstream code doesn't care about format.
2. Add best-effort image (OCR) and Excel extraction behind the same interface, clearly flagged lower-confidence in output.
3. Build a metadata-inference step: use an LLM call with the extracted text to pull structured fields (title, date, department, organiser, authors, participants, likely document type) — return as strict JSON.
4. Persist extraction results to `documents` table (raw text, extracted tables, inferred metadata, processing status).
5. Update Upload UI to poll/show processing status per file.
6. Run the full synthetic set (Day 3) through the pipeline; log failures and fix parsing edge cases (e.g., scanned PDFs, tables spanning pages).

**📂 Files/folders:**
- `/backend/src/documents/extract/{pdf,docx,image,excel}.*`
- `/backend/src/documents/metadata_inference.*`
- `/backend/src/documents/pipeline.*` (orchestrates extract → infer → persist)

**🔗 Integrations:** LLM API for metadata inference (this is your first real AI integration — set up API key handling, error handling, and rate-limit awareness now, since Day 6-7 build heavily on this same integration point).

**🧪 Testing tasks:** Run all ~25-40 synthetic documents through the pipeline; verify extraction accuracy against `SYNTHETIC_DATA_MANIFEST.md`; confirm failures degrade gracefully (document marked "needs review", not a crash).

**🐞 Common issues:** DOCX tables not parsing cleanly; LLM returning malformed JSON (add strict prompt + JSON-repair/retry logic); large PDFs timing out (add async/background processing, not synchronous request blocking).

**✅ End-of-day checklist:**
- [ ] PDF + DOCX extraction working reliably on the full synthetic set
- [ ] Metadata inference producing structured, mostly-accurate output
- [ ] Processing status visible in UI
- [ ] Failures handled gracefully, not silently or via crash

**📸 Capture:** Screenshot of a processed document's extracted metadata, a processing-status view, and console/log output of a full synthetic-set run.

**➡️ Handoff notes:** Document your LLM integration pattern (prompt structure, error handling) in `/docs/ARCHITECTURE.md` — Day 6 reuses this exact pattern for classification, so consistency here saves rework.

---

## Day 6 — AI Classification, Validation & Gap Detection (Core AI Engine)

**🎯 Objective:** Build the heart of the product — AI classification of documents to Criterion/Key Indicator/Metric with confidence scores, evidence quality validation, and gap detection, fully working for Criteria 2, 3, and 6.

**📖 What you'll learn:** Prompt engineering for structured classification tasks; RAG-lite grounding using the NAAC framework; building explainable AI outputs.

**🛠 Features to build:**
- Classification engine: given extracted document content, output best-matching Criterion/Key Indicator/Metric + confidence score, for Criteria 2/3/6's metrics only (v1.0 scope).
- Validation engine: flags duplicates, missing dates/signatures, incomplete attachments, weak evidence.
- Gap detection: for each in-scope Metric, determine "sufficient / weak / missing" evidence status.
- Recommendation generator: specific, actionable text suggestions per document and per gap.

**📝 Step-by-step implementation plan:**
1. Build a structured "NAAC framework context" for Criteria 2, 3, 6 — Key Indicators, Metrics, and their evidence requirements, formatted for LLM prompt injection (this is your RAG grounding source; a full vector DB is optional — for ~30-60 metrics, direct prompt injection of the relevant Criterion's framework is simpler and sufficient for v1.0).
2. Write the classification prompt: input = extracted text + metadata + the relevant framework context; output = strict JSON `{criterion, key_indicator, metric, confidence, reasoning}`. Test against the synthetic set and tune against `SYNTHETIC_DATA_MANIFEST.md` ground truth.
3. Write the validation prompt/logic: checks for dates, signatures, required attachment mentions, and flags duplicates via a text-similarity check against already-approved documents in the same institution.
4. Implement gap detection as a deterministic function (not LLM-based) over classified+validated evidence: for each Metric in scope, compare existing evidence against defined requirements → status.
5. Write the recommendation generator prompt: given a document's classification + validation results + the Metric's gap status, produce 1-3 short, specific, actionable recommendations.
6. Wire all of this into the pipeline from Day 5, so upload → extract → classify → validate → gap-check → recommend runs end-to-end automatically.

**📂 Files/folders:**
- `/backend/src/ai/classify.*`, `/backend/src/ai/validate.*`, `/backend/src/ai/gap_detection.*`, `/backend/src/ai/recommend.*`
- `/backend/src/naac_framework/criteria_2_context.*`, `criteria_3_context.*`, `criteria_6_context.*`
- `/docs/AI_PROMPTS.md` (record every prompt template used — needed for Day 7 tuning and Day 8 Q&A reuse)

**🔗 Integrations:** Same LLM API as Day 5. If accuracy is inconsistent, consider a second-pass "self-check" prompt rather than switching models mid-build.

**🧪 Testing tasks:** Run full synthetic set through classification — measure accuracy against manifest ground truth; deliberately test weak/incomplete documents to confirm validation and gap detection catch them.

**🐞 Common issues:** Low classification confidence on ambiguous documents (surface confidence honestly in UI rather than forcing a guess); LLM hallucinating a Metric number that doesn't exist (validate output against your known Metric list before persisting); gap detection logic double-counting evidence across Metrics.

**✅ End-of-day checklist:**
- [ ] Classification working end-to-end for Criteria 2, 3, 6 with confidence scores
- [ ] Validation flags real issues on deliberately-weak synthetic documents
- [ ] Gap detection produces accurate per-Metric status
- [ ] Recommendations are specific and actionable, not generic

**📸 Capture:** Screenshot of a full classification result (JSON or UI view) for one strong and one weak document, and a gap-detection summary for one Criterion.

**➡️ Handoff notes:** `AI_PROMPTS.md` and the accuracy results against the manifest are essential Day 7 input — Day 7 does not re-derive prompts, only wires scoring and surfaces results in UI.

---

## Day 7 — Readiness Scoring, Human Approval Flow & Explainability UI

**🎯 Objective:** Implement the full Metric → Key Indicator → Criterion → Institution scoring hierarchy, the human-in-the-loop approval screen, and the explainability UI ("why this score") for the core workflow.

**📖 What you'll learn:** Weighted hierarchical scoring implementation; designing trustworthy, explainable AI UX.

**🛠 Features to build:**
- Scoring engine implementing the 4-level hierarchy from the PRD.
- Approval screen: coordinator reviews AI classification/validation/recommendations, can edit Criterion/Key Indicator/Metric mapping, approves or rejects.
- Live Readiness Dashboard: real scores (no more placeholders) for Criteria 2, 3, 6; other 4 Criteria show a clear "structure ready, AI coming soon" state (not broken/empty).
- "Why this score" drill-down: click any score to see contributing evidence, gaps, and recommended actions.

**📝 Step-by-step implementation plan:**
1. Implement Metric Readiness Score calculation using evidence status (from Day 6 gap detection), quality signals, and confidence — as a documented, deterministic formula (write it down in `/docs/SCORING_MODEL.md` so it's defensible in a judge Q&A).
2. Implement Key Indicator → Criterion → Institution aggregation using NAAC's published weightages (source and record these weightages in `SCORING_MODEL.md`).
3. Build the Approval screen: shows extracted content preview, AI's suggested Criterion/Key Indicator/Metric + confidence + reasoning, validation flags, recommendations; coordinator can accept or override any field before it commits to the knowledge base and triggers a score recalculation.
4. On approval, trigger score recalculation and update the dashboard (real-time or on next load — real-time is nicer for a demo if time allows).
5. Build the drill-down UI: clicking a score opens a panel/page showing the formula inputs — which documents contributed, which Metrics are weak/missing, and the top 2-3 recommended actions to improve the score.
6. Update Criterion cards for the 4 out-of-scope Criteria to a clear, intentional "Coming in a future release" state — this should look like a designed product decision, not an unfinished feature.

**📂 Files/folders:**
- `/backend/src/scoring/*`
- `/frontend/app/dashboard/approve/[documentId]/*`
- `/frontend/components/ScoreDrilldown/*`
- `/docs/SCORING_MODEL.md`

**🔗 Integrations:** None new.

**🧪 Testing tasks:** Approve a batch of synthetic documents across Criteria 2/3/6 and confirm scores update correctly and match manual hand-calculation for at least one Metric; test rejecting/editing an AI classification and confirm it doesn't corrupt scores.

**🐞 Common issues:** Score not recalculating on approval (check the trigger, not just the formula); weightages not summing correctly (validate they total 100% at each level); drill-down showing stale data after approval.

**✅ End-of-day checklist:**
- [ ] Full scoring hierarchy implemented and documented
- [ ] Approval screen fully functional, coordinator can edit before commit
- [ ] Live dashboard shows real, correct scores for Criteria 2/3/6
- [ ] Drill-down explainability working for at least Metric and Criterion levels

**📸 Capture:** Screenshots of the approval screen, live dashboard with real scores, and a drill-down panel.

**➡️ Handoff notes:** The **core product loop is now fully complete end-to-end** (upload → ...→ score). Day 8 builds supporting features on top of this working foundation — do not modify the core loop unless fixing a bug.

---

## Day 8 — AI Q&A Assistant & AQAR/SSR Draft Generation

**🎯 Objective:** Build the two 🟡 supporting AI features: conversational Q&A over approved evidence, and AQAR/SSR narrative draft generation, both scoped to Criteria 2/3/6.

**📖 What you'll learn:** Retrieval-augmented Q&A over a document set; long-form structured generation grounded in real data.

**🛠 Features to build:**
- Chat-style Q&A interface: ask questions about institutional evidence, get grounded answers with source citations.
- AQAR/SSR draft generator: produces narrative report sections per Metric/Key Indicator using approved evidence, exportable as a document.
- Basic focused analytics view: readiness trend over time, gap distribution chart.

**📝 Step-by-step implementation plan:**
1. Implement retrieval: for a user question, fetch the most relevant approved documents (simple keyword/embedding similarity search is sufficient at this data scale — a full vector DB is optional; even a straightforward similarity search over stored text works for a demo-scale document set).
2. Build the Q&A prompt: inject retrieved document excerpts + question → LLM answers with inline citation of which document(s) it used; always ground answers only in retrieved evidence, and have it say "I don't have evidence for that" rather than fabricate.
3. Build a simple chat UI (message list + input) on a new `/dashboard/ask` route.
4. Build AQAR/SSR generation: for each in-scope Metric/Key Indicator, generate a narrative paragraph from its approved evidence, in NAAC's typical report tone; assemble into a document export (PDF or DOCX) with clear "AI-generated draft — requires institutional review" labeling.
5. Build the analytics view: a readiness-over-time line chart (use approval timestamps as the time axis) and a gap-count bar chart by Criterion.

**📂 Files/folders:**
- `/backend/src/ai/qa_retrieval.*`, `/backend/src/ai/qa_answer.*`
- `/backend/src/ai/report_generation.*`
- `/frontend/app/dashboard/ask/*`, `/frontend/app/dashboard/reports/*`, `/frontend/app/dashboard/analytics/*`

**🔗 Integrations:** Same LLM API; document export library for AQAR/SSR draft output.

**🧪 Testing tasks:** Ask 8-10 realistic IQAC-style questions and confirm grounded, cited answers (and correct refusal on out-of-scope questions); generate an AQAR draft for one Criterion and sanity-check accuracy against the actual approved evidence.

**🐞 Common issues:** Q&A hallucinating beyond retrieved evidence (tighten prompt to explicitly forbid this); report generation producing generic text instead of using specific evidence details (pass more specific extracted fields into the prompt, not just document titles).

**✅ End-of-day checklist:**
- [ ] Q&A returns grounded, cited answers for supported Criteria
- [ ] AQAR/SSR draft generation produces usable, evidence-specific narrative content
- [ ] Analytics view shows real trend/gap data
- [ ] All labeled clearly as supporting/draft features, consistent with PRD framing

**📸 Capture:** Screenshot of a Q&A exchange with citation, an AQAR draft excerpt, and the analytics view.

**➡️ Handoff notes:** All 🟢 and 🟡 features are now functionally complete. Day 9 is dedicated entirely to testing, polish, and bug-fixing — no new features should be started after today unless something from the PRD is missing.

---

## Day 9 — End-to-End Testing, UI Polish & Hardening

**🎯 Objective:** Systematically test the entire product against the PRD's Day 10 success criteria, fix bugs, and apply visual/UX polish (branding, onboarding, error states).

**📖 What you'll learn:** Structured QA against acceptance criteria; the specific, high-leverage polish that separates a "student project" from a "SaaS product."

**🛠 Features to build (polish, not new features):**
- Consistent branding pass (logo/wordmark, color palette, typography) across every screen.
- Onboarding flow polish for first-time institution registration.
- Loading, empty, and error states audited on every page.
- Bug fixes from full end-to-end test pass.

**📝 Step-by-step implementation plan:**
1. Re-read Section 9 ("Success Metrics — Day 10 Definition of Done") of the PRD and turn it into a literal checklist; go through the entire product as if you were an AB Talks judge seeing it for the first time.
2. Register a brand-new institution from scratch and walk the entire core loop end-to-end (upload → approve → dashboard → Q&A → report) without any pre-existing data — this is the single most important test, since judges will likely do exactly this.
3. Fix every bug found, prioritizing anything in the 🟢 core loop first, then 🟡 supporting features.
4. Apply a consistent visual identity: pick 2-3 brand colors + one accent, one heading font + one body font, consistent spacing/card style — apply everywhere, including the 4 "coming soon" Criteria states.
5. Audit every page for loading states (skeleton/spinner, not blank white), empty states (helpful, not broken-looking), and error states (clear message + recovery action, not a stack trace or blank screen).
6. Test on both desktop and mobile/narrow viewport widths — even basic responsiveness matters for a polished impression.
7. Write a short internal "known limitations" note for yourself (not necessarily shown to users) — anything you knowingly did not fix, so Day 10's demo can navigate around it deliberately rather than stumbling into it live.

**📂 Files/folders:**
- Touch nearly everything, but track fixes in `/docs/DAY9_QA_LOG.md` (bug found → fix applied)
- `/frontend/styles/theme.*` (centralize brand colors/fonts here if not already)

**🔗 Integrations:** None new.

**🧪 Testing tasks:** Full fresh-institution walkthrough (see step 2); test each of the 4 fully-functional roles' permissions again; test with intentionally bad input (empty upload, huge file, corrupted PDF) and confirm graceful failure.

**🐞 Common issues:** Polish "creeping" into scope changes — resist adding new features today; inconsistent spacing/colors discovered late — fix systematically via a shared theme file, not page-by-page.

**✅ End-of-day checklist:**
- [ ] Every item in PRD Section 9 verified working
- [ ] Fresh-institution full walkthrough completed successfully, no blockers
- [ ] Consistent branding applied across all pages, including "coming soon" Criteria
- [ ] All loading/empty/error states reviewed and fixed
- [ ] Known limitations documented for demo planning

**📸 Capture:** Screenshots of the polished dashboard, onboarding flow, and at least one before/after of a fixed page.

**➡️ Handoff notes:** Product is feature-complete and polished. Day 10 is deployment finalization + demo/portfolio packaging only — no functional changes unless a launch-blocking bug appears.

---

## Day 10 — Final Deployment, Demo Packaging & Launch

**🎯 Objective:** Finalize production deployment on a stable public URL, prepare the live demo path, and package the product for judges, LinkedIn, and portfolio presentation.

**📖 What you'll learn:** Production deployment hardening; presenting a technical product persuasively to a non-building audience.

**🛠 Features to build:** None — deployment, environment, and presentation only.

**📝 Step-by-step implementation plan:**
1. Do a final production deployment (not staging) — confirm environment variables, database, and file storage are all production-configured, not pointing at local/dev resources.
2. Re-seed a clean demo institution with a curated subset of synthetic evidence (a mix of strong and weak, across Criteria 2/3/6) so a live demo tells a clear "gap → upload → improvement" story rather than a random walkthrough.
3. Do one final full walkthrough on the production URL itself (not localhost) to catch any environment-specific issues.
4. Write and rehearse a tight demo script: Problem (30s) → Core loop live demo (3-4 min: upload → AI classify/validate → approve → score updates → drill-down) → Q&A/report generation (1 min) → Vision/future scope (30s).
5. Prepare a short written project summary (for AB Talks submission, LinkedIn post, and portfolio) — pull directly from the PRD's Executive Summary and this blueprint's Day 10 objective, don't rewrite the positioning from scratch.
6. Do a final check of the Pitch Deck (generated today alongside the PRD and this blueprint) against the actual live product — make sure no slide claims a feature that isn't real in the deployed app.
7. Capture final polished screenshots/short screen recording of the live product for LinkedIn/portfolio use.

**📂 Files/folders:**
- `/docs/DEMO_SCRIPT.md`
- `/docs/LAUNCH_CHECKLIST.md`

**🔗 Integrations:** Confirm production values for any API keys, database, and storage integrations used since Day 2.

**🧪 Testing tasks:** Full end-to-end walkthrough on the live production URL, from a signed-out state, exactly as a judge would experience it.

**🐞 Common issues:** Environment variables missing in production (silent failures on AI calls); demo institution's data too sparse or too perfect to tell a compelling story — curate deliberately.

**✅ End-of-day checklist:**
- [ ] Production URL stable and fully functional end-to-end
- [ ] Clean, curated demo institution/data seeded
- [ ] Demo script written and rehearsed at least once, timed
- [ ] Pitch deck cross-checked against live product for accuracy
- [ ] Final screenshots/recording captured

**📸 Capture:** Final screenshots of every major screen on the production URL, plus a short screen recording of the full demo script.

**➡️ Handoff notes:** This is the final deliverable state. Any future work (Criteria 1/4/5/7 AI depth, additional role dashboards, NBA/NIRF, ERP integrations) begins from this document's "Out of Scope for v1.0" list in the PRD — it does not require re-architecture, only extension of the existing framework.

---

## Appendix A — Documents to Maintain Throughout the Build

| File | Purpose | First created |
|---|---|---|
| `/docs/ARCHITECTURE.md` | Stack, schema, deployment, integration decisions | Day 2 |
| `/docs/SYNTHETIC_DATA_MANIFEST.md` | Ground truth for all synthetic evidence documents | Day 3 |
| `/docs/AI_PROMPTS.md` | All prompt templates used, for reuse/tuning | Day 6 |
| `/docs/SCORING_MODEL.md` | Exact scoring formulas and NAAC weightages used | Day 7 |
| `/docs/DAY9_QA_LOG.md` | Bugs found and fixed during hardening | Day 9 |
| `/docs/DEMO_SCRIPT.md`, `/docs/LAUNCH_CHECKLIST.md` | Final demo/launch materials | Day 10 |

Keeping these current is what lets each new daily AI conversation pick up exactly where the last one left off, without re-deriving decisions already made.

## Appendix B — Scope Discipline Reminder

If a "great idea" comes up mid-build that isn't in the PRD's 🟢/🟡 tiers: write it down in the PRD's Section 4.3 (Out of Scope) instead of building it. That list is the parking lot for v2 — protecting it protects Day 10.
