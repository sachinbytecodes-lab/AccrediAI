# AccrediAI — Project Structure (v1.0)

```
AccrediAI/
├── docs/
│   ├── ARCHITECTURE.md
│   ├── SCHEMA.md
│   ├── API.md
│   ├── UI-WIREFRAMES.md
│   ├── PROJECT-STRUCTURE.md
│   ├── AI_PROMPTS.md              # created Day 6
│   ├── SCORING_MODEL.md           # created Day 7
│   ├── SYNTHETIC_DATA_MANIFEST.md # created Day 3
│   ├── DAY9_QA_LOG.md             # created Day 9
│   ├── DEMO_SCRIPT.md             # created Day 10
│   └── PROJECT_LOG.md             # running daily log, updated every day
│
├── prisma/
│   ├── schema.prisma               # DB schema (source of truth — see SCHEMA.md)
│   ├── seed.ts                     # seeds NAAC framework (criteria/key_indicators/metrics)
│   └── migrations/
│
├── data/
│   └── synthetic_evidence/
│       ├── criterion_2/
│       ├── criterion_3/
│       └── criterion_6/
│
├── src/
│   ├── app/                        # Next.js App Router
│   │   ├── (auth)/
│   │   │   ├── login/page.tsx
│   │   │   └── register/page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx          # sidebar + top bar shell
│   │   │   ├── page.tsx            # Dashboard Home
│   │   │   ├── criteria/[id]/page.tsx
│   │   │   ├── ask/page.tsx
│   │   │   ├── reports/page.tsx
│   │   │   ├── analytics/page.tsx
│   │   │   └── users/page.tsx
│   │   └── api/                    # API Routes — mirrors API.md 1:1
│   │       ├── auth/{register,login}/route.ts
│   │       ├── users/{invite,route.ts}
│   │       ├── institution/route.ts
│   │       ├── framework/criteria/{route.ts,[id]/route.ts}
│   │       ├── documents/{upload,route.ts,[id]/{route.ts,approve/route.ts,reject/route.ts}}
│   │       ├── readiness/{institution/route.ts,criteria/[id]/route.ts,metrics/[id]/explain/route.ts}
│   │       ├── recommendations/route.ts
│   │       ├── ask/route.ts
│   │       ├── reports/{generate/route.ts,[jobId]/route.ts}
│   │       └── analytics/summary/route.ts
│   │
│   ├── components/
│   │   ├── ui/                     # shadcn/ui primitives
│   │   ├── ReadinessCard/
│   │   ├── UploadWidget/
│   │   ├── ApprovalScreen/
│   │   ├── ScoreDrilldown/
│   │   ├── AskAIChat/
│   │   └── layout/{Sidebar,TopBar}.tsx
│   │
│   ├── lib/
│   │   ├── ai/
│   │   │   ├── provider.ts         # AIProvider interface (ARCHITECTURE.md §5)
│   │   │   ├── claudeProvider.ts   # default implementation
│   │   │   └── prompts/            # modular prompt templates
│   │   │       ├── classify.ts
│   │   │       ├── validate.ts
│   │   │       ├── recommend.ts
│   │   │       ├── qa.ts
│   │   │       └── reportGen.ts
│   │   ├── documents/
│   │   │   ├── extract/{pdf.ts,docx.ts,image.ts,excel.ts}
│   │   │   └── pipeline.ts         # extract → infer → persist orchestration
│   │   ├── scoring/
│   │   │   ├── metricScore.ts
│   │   │   ├── keyIndicatorScore.ts
│   │   │   ├── criterionScore.ts
│   │   │   └── institutionScore.ts
│   │   ├── auth/
│   │   │   ├── session.ts          # Supabase JWT helpers
│   │   │   └── rbac.ts             # role + tenant permission checks
│   │   ├── validation/
│   │   │   └── schemas/            # Zod schemas, one file per endpoint group
│   │   ├── db/
│   │   │   └── prisma.ts           # Prisma client singleton
│   │   └── storage/
│   │       └── supabaseStorage.ts
│   │
│   ├── services/                   # Service Layer — business logic, called by API routes
│   │   ├── documentService.ts
│   │   ├── scoringService.ts
│   │   ├── recommendationService.ts
│   │   ├── qaService.ts
│   │   └── reportService.ts
│   │
│   ├── hooks/                      # TanStack Query hooks
│   │   ├── useDocuments.ts
│   │   ├── useReadiness.ts
│   │   └── useAskAI.ts
│   │
│   └── types/
│       └── index.ts                 # shared TS types (mirrors Prisma models + API contracts)
│
├── public/
│   └── branding/                     # logo, favicon, brand assets (Day 9 polish)
│
├── .env.example
├── .gitignore
├── package.json
├── tailwind.config.ts
├── tsconfig.json
└── README.md
```

## Rationale

- **`src/app/api/*` mirrors `API.md` exactly** — anyone (including a fresh AI conversation on any future day) can find the implementation of any endpoint by its documented path with zero guesswork.
- **`services/` is separated from `app/api/*`** per the "Service Layer Architecture" stack decision — API routes stay thin (auth/validation/response shaping only), business logic lives in one testable, reusable place. This is what lets Day 6-8's AI logic be added without touching route files repeatedly.
- **`lib/ai/provider.ts` + `claudeProvider.ts`** encode the AI Provider Abstraction agreed in the stack discussion — a future second provider is a new file, not a rewrite.
- **`lib/ai/prompts/` as separate files** implements "Modular Prompt Templates" concretely — each maps directly to a step in the core workflow (classify → validate → recommend), and to `AI_PROMPTS.md`'s documentation.
- **`data/synthetic_evidence/`** lives outside `src/` since it's test fixtures, not application code — mirrors Blueprint Day 3.
- **`docs/`** is the living single source of truth — every "Handoff notes" section in the Implementation Blueprint points back here.
- **No `tests/` folder in v1.0** — consistent with today's stack decision to defer formal test infrastructure; manual/scripted smoke testing happens inline per the Blueprint's Day 9 plan.

## What goes where, going forward

- New API endpoint → add to `src/app/api/`, document it in `API.md` first.
- New AI capability → new prompt file in `lib/ai/prompts/`, new function in the relevant `services/*.ts`.
- New Criterion getting full AI support (post-v1.0) → set `ai_fully_supported = true` in `criteria` table + add its framework context — no new folders required, confirming the architecture's extensibility claim.
