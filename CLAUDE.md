# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

**Hermeneus** (ἑρμηνεύς — the interpreter) — ORGANVM intelligence layer. A Next.js 16 system with full contextual awareness of every repo across the eight-organ system. Features natural language chat (Groq/OSS LLM with provider cascade), hybrid vector+lexical retrieval, a Postgres-backed operational control plane, and admin dashboard with GitHub SSO.

## Build & Dev Commands

```bash
npm run dev              # next dev (localhost:3000)
npm run lint             # eslint (flat config), --max-warnings=0
npm run test             # vitest run (all tests in tests/)
npm run test -- tests/query-planner.test.ts   # single test file
npm run test:watch       # vitest watch mode
npm run build            # prebuild (generate + validate manifest) then next build

# Data pipeline — regenerate manifest from live workspace
npm run generate         # tsx ingest-worker.ts (requires DATABASE_URL + EMBEDDING_API_KEY)
npm run generate -- --allow-stale-manifest --skip-vector  # offline/CI mode

# Database (Drizzle + Postgres with pgvector)
npm run db:generate      # drizzle-kit generate (after schema changes)
npm run db:migrate       # drizzle-kit migrate
npm run db:studio        # drizzle-kit studio (GUI)

# Operational scripts
npm run maintenance:run  # run maintenance cycle (connectors + eval + alerts)
npm run eval:offline     # run eval suite offline
npm run ci:quality-gate  # CI threshold checks (citation coverage, hallucination, etc.)
npm run validate-manifest
```

## Architecture

```
[111 repos] → ingest-worker.ts → manifest.json + pgvector embeddings
                                         ↓
                              Next.js 15 app (Vercel)
                             /          |          \
                    static pages    /api/chat    /api/admin + /api/cron
                   (repo browser,  (hybrid       (maintenance, alerts,
                    organs, dash)   retrieval     connectors, auth)
                                   + LLM)
```

### Five layers

1. **Ingestion** (`src/lib/ingestion/ingest-worker.ts`) — Reads registry-v2.json, seed.yaml, CLAUDE.md/README.md, git logs → produces `src/data/manifest.json` AND chunks + embeds content into Postgres/pgvector via LangChain text splitters
2. **Frontend** (Next.js 15 + Tailwind 4 + React 19) — Static pages for repos, organs, dashboard, about
3. **AI chat** (`/api/chat`) — Query planner → hybrid retrieval (lexical + vector + knowledge graph + federated) → Groq/OSS LLM streaming with citations and PII masking
4. **Control plane** — Maintenance cycles, connector orchestration, alert evaluation/dispatch, job queue (Postgres SKIP LOCKED), escalation policies
5. **Database** (Drizzle ORM + Postgres + pgvector) — Document chunks with HNSW vector index, job queue, maintenance ledger, alert audit trail, connector cursors, escalation policies

### Chat pipeline detail

```
User query → planQuery() → hybridRetrieve() → buildCitations() → LLM stream
               ↓                ↓
         QueryStrategy     Multi-signal scoring:
         classification     - lexical (TF-IDF)
         + sub-queries      - vector (pgvector cosine)
         + answerability    - knowledge graph traversal
                            - federated knowledge base
```

`QueryStrategy` types: `deterministic`, `single_repo`, `organ_scope`, `cross_organ`, `system_wide`, `graph_traversal`, `live_research`, `analytics`, `exploratory`.

### Key files

| Path | Purpose |
|------|---------|
| `src/lib/ingestion/ingest-worker.ts` | Manifest generation + vector embedding pipeline |
| `src/data/manifest.json` | Generated data snapshot (committed, refreshed intentionally) |
| `src/lib/types.ts` | TypeScript interfaces for manifest schema |
| `src/lib/manifest.ts` | Manifest loader + query helpers |
| `src/lib/query-planner.ts` | Cost-based strategy selection + query decomposition |
| `src/lib/hybrid-retrieval.ts` | Multi-signal retrieval (lexical + vector + graph + federated) |
| `src/lib/retrieval.ts` | Tier-1/Tier-2 context assembly (original lexical retrieval) |
| `src/lib/citations.ts` | Citation extraction and inline formatting |
| `src/lib/graph.ts` | Knowledge graph (produces/consumes edges from seeds) |
| `src/lib/entity-registry.ts` | Named entity index for disambiguation |
| `src/app/api/chat/route.ts` | Chat endpoint: rate limiting, security, LLM streaming |
| `src/lib/db/schema.ts` | Drizzle schema (all tables + vector index) |
| `src/lib/maintenance.ts` | Maintenance cycle orchestration + scorecard |
| `src/lib/alerts.ts` | System alert evaluation from scorecard data |
| `src/lib/alert-sinks.ts` | Alert dispatch (Slack, webhook, email) with retry |
| `src/lib/connectors/` | Pluggable data connectors (docs, workspace, GitHub, Slack) |
| `src/lib/platform-config.ts` | Runtime config registry (connectors, retention, compliance, SLOs) |
| `src/lib/security.ts` | PII masking, access context, audit logging |
| `src/lib/observability.ts` | Counters, timing, metrics snapshot |
| `src/auth.ts` | NextAuth config (GitHub OAuth provider) |
| `src/middleware.ts` | Edge middleware — session enforcement for /admin, /dashboard |
| `drizzle.config.ts` | Drizzle Kit config (migrations at `src/lib/db/migrations/`) |

### Pages

- `/` — Landing: metrics, organ cards, deployments
- `/repos` — Filterable repo browser
- `/repos/[slug]` — Repo detail with sections, git stats, links
- `/organs` — Organ grid overview
- `/organs/[key]` — Organ detail with repos
- `/dashboard` — Metrics, promotion pipeline, CI health (requires auth)
- `/ask` — AI chat interface
- `/about` — Methodology and organ descriptions
- `/admin/login` — Admin login
- `/admin/intel` — Admin intelligence panel

### API routes

- `POST /api/chat` — Streaming chat with rate limiting (10 req/min per IP)
- `POST /api/feedback` — User feedback on chat responses
- `GET /api/analytics/word-frequency` — Corpus text analytics
- `POST /api/cron/maintenance` — Cron-triggered maintenance cycle (auth: `CRON_SECRET`)
- `GET|POST /api/admin/intel` — Admin scorecard/alerts (auth: session or `ADMIN_API_TOKEN`)
- `POST /api/admin/session` — Admin session management
- `GET|POST /api/auth/[...nextauth]` — NextAuth handlers

## Database

Postgres with pgvector extension. Schema in `src/lib/db/schema.ts`:

- `document_chunks` — Chunked content with 384-dim embeddings (HNSW index, HuggingFace all-MiniLM-L6-v2) + tsvector full-text search (GIN index)
- `jobs` — Postgres-based job queue with SKIP LOCKED dequeue
- `maintenance_runs` — Maintenance cycle ledger with scorecards
- `singleton_locks` — Distributed locking for maintenance/cron
- `alert_deliveries` — Alert dispatch audit trail
- `escalation_policies` — Configurable alert escalation rules
- `connector_cursors` — Incremental sync state per connector

Connection: `DATABASE_URL` env var. Default: `postgresql://postgres:postgres@localhost:5432/stakeholder_portal`

## Environment

Copy `.env.example` → `.env`. Key variable groups:

- **LLM**: `GROQ_API_KEY`, `GROQ_MODEL`, `GROQ_API_URL` (primary); `OSS_LLM_API_URL`, `OSS_LLM_MODEL` (fallback)
- **Database**: `DATABASE_URL` (required for ingestion, maintenance, vector search)
- **Embeddings**: `EMBEDDING_API_KEY`, `EMBEDDING_MODEL`, `EMBEDDING_API_URL`
- **Auth**: `AUTH_SECRET`, `AUTH_GITHUB_ID`, `AUTH_GITHUB_SECRET` (NextAuth GitHub SSO)
- **Admin**: `ADMIN_API_TOKEN`, `ADMIN_LOGIN_PASSWORD`, `ADMIN_SESSION_SECRET`
- **Alerts**: `SLACK_WEBHOOK_URL`, `ALERT_WEBHOOK_URL`, `RESEND_API_KEY`, `ALERT_EMAIL_TO`
- **Cron**: `CRON_SECRET`, `CRON_CONNECTOR_IDS`
- **Federation**: `MY_KNOWLEDGE_BASE_API_URL`, `MY_KNOWLEDGE_BASE_ENABLED`
- **CI thresholds**: `ALERT_MIN_CITATION_COVERAGE`, `ALERT_MAX_HALLUCINATION_RATE`, etc.

## CI/CD Configuration

- CI job MUST be named `test` — branch protection requires this exact check name
- Release Drafter runs on push to main ONLY — pull_request trigger causes invalid targetCommitish
- Branch protection: requires `test` check (non-strict, doesn't require up-to-date branch)
- Vercel deploys automatically on push via GitHub integration
- Gemini CLI may trigger duplicate Vercel deployments alongside the GitHub integration
- Vercel team `ivviiviivvi` (team_bE3F62pxdvINJLUQbEmEOMKi): 3 active projects (stakeholder-portal, specvla-ergon--avditor-mvndi, peer-audited--behavioral-blockchain), 0 custom domains

## Propagation Discipline

When completing work, the 10-index propagation checklist is MANDATORY (see IRF header). **Every index is always applicable** — if this session didn't advance an index, identify the gap and log it as a new IRF item. Never mark "N/A." The default is check-all-10 and consciously execute or document the vacuum, not check-none.

## Conventions

- Node 22+, npm
- TypeScript strict mode
- Tailwind v4 (PostCSS plugin)
- Dark theme with CSS custom properties
- Conventional commits
- Tests in `tests/` (not co-located), using Vitest
- `@/` path alias maps to `src/`

<!-- ORGANVM:AUTO:START -->
## System Context (auto-generated — do not edit)

**Organ:** META-ORGANVM (Meta) | **Tier:** standard | **Status:** PUBLIC_PROCESS
**Org:** `meta-organvm` | **Repo:** `stakeholder-portal`

### Edges
- **Produces** → `external-stakeholders, internal-operators`: hermeneus-intelligence-ui
- **Produces** → `unspecified`: llm-health-api
- **Produces** → `unspecified`: ingestion-health-api
- **Consumes** ← `organvm-corpvs-testamentvm`: registry-v2.json
- **Consumes** ← `organvm-corpvs-testamentvm`: system-metrics.json
- **Consumes** ← `organvm-engine`: organvm-engine

### Siblings in Meta
`.github`, `organvm-corpvs-testamentvm`, `alchemia-ingestvm`, `schema-definitions`, `organvm-engine`, `system-dashboard`, `organvm-mcp-server`, `praxis-perpetua`, `materia-collider`, `organvm-ontologia`, `vigiles-aeternae--agon-cosmogonicum`, `cvrsvs-honorvm`, `custodia-securitatis`

### Governance
- *Standard ORGANVM governance applies*

*Last synced: 2026-05-23T00:26:31Z*

## Active Handoff Protocol

If `.conductor/active-handoff.md` exists, **READ IT FIRST** before doing any work.
It contains constraints, locked files, conventions, and completed work from the
originating agent. You MUST honor all constraints listed there.

If the handoff says "CROSS-VERIFICATION REQUIRED", your self-assessment will
NOT be trusted. A different agent will verify your output against these constraints.

## Session Review Protocol

At the end of each session that produces or modifies files:
1. Run `organvm session review --latest` to get a session summary
2. Check for unimplemented plans: `organvm session plans --project .`
3. Export significant sessions: `organvm session export <id> --slug <slug>`
4. Run `organvm prompts distill --dry-run` to detect uncovered operational patterns

Transcripts are on-demand (never committed):
- `organvm session transcript <id>` — conversation summary
- `organvm session transcript <id> --unabridged` — full audit trail
- `organvm session prompts <id>` — human prompts only


## System Library

Plans: 269 indexed | Chains: 5 available | SOPs: 8 active
Discover: `organvm plans search <query>` | `organvm chains list` | `organvm sop lifecycle`
Library: `/Users/4jp/Code/organvm/praxis-perpetua/library`


## Active Directives

| Scope | Phase | Name | Description |
|-------|-------|------|-------------|
| system | any | atomic-clock | The Atomic Clock |
| system | any | execution-sequence | Execution Sequence |
| system | any | multi-agent-dispatch | Multi-Agent Dispatch |
| system | any | session-handoff-avalanche | Session Handoff Avalanche |
| system | any | system-loops | System Loops |
| system | any | prompting-standards | Prompting Standards |
| system | any | background-task-resilience | background-task-resilience |
| system | any | context-window-conservation | context-window-conservation |
| system | any | session-self-critique | session-self-critique |
| system | any | the-descent-protocol | the-descent-protocol |
| system | any | the-membrane-protocol | the-membrane-protocol |
| system | any | theory-to-concrete-gate | theory-to-concrete-gate |
| system | any | triangulation-protocol | triangulation-protocol |

Linked skills: SOP-TRIADIC-REVIEW-PROTOCOL, cicd-resilience-and-recovery, continuous-learning-agent, evaluation-to-growth, genesis-dna, multi-agent-workforce-planner, promotion-and-state-transitions, quality-gate-baseline-calibration, repo-onboarding-and-habitat-creation, session-self-critique, structural-integrity-audit, the-membrane-protocol, triple-reference


**Prompting (Anthropic)**: context 200K tokens, format: XML tags, thinking: extended thinking (budget_tokens)


## System Density (auto-generated)

AMMOI: 25% | Edges: 0 | Tensions: 0 | Clusters: 0 | Adv: 27 | Events(24h): 37975
Structure: 8 organs / 148 repos / 1654 components (depth 17) | Inference: 0% | Organs: META-ORGANVM:63%, ORGAN-I:53%, ORGAN-II:48%, ORGAN-III:54% +5 more
Last pulse: 2026-05-23T00:26:28 | Δ24h: n/a | Δ7d: n/a


## Dialect Identity (Trivium)

**Dialect:** SELF_WITNESSING | **Classical Parallel:** The Eighth Art | **Translation Role:** The Witness — proves all translations compose without loss

Strongest translations: I (formal), IV (structural), V (analogical)

Scan: `organvm trivium scan META <OTHER>` | Matrix: `organvm trivium matrix` | Synthesize: `organvm trivium synthesize`


## Logos Documentation Layer

**Status:** ACTIVE | **Symmetry:** 0.5 (DREAM)

Nature demands a documentation counterpart. This formation maintains its narrative record in `docs/logos/`.

### The Tetradic Counterpart
- **[Telos (Idealized Form)](../docs/logos/telos.md)** — The dream and theoretical grounding.
- **[Pragma (Concrete State)](../docs/logos/pragma.md)** — The honest account of what exists.
- **[Praxis (Remediation Plan)](../docs/logos/praxis.md)** — The attack vectors for evolution.
- **[Receptio (Reception)](../docs/logos/receptio.md)** — The account of the constructed polis.

### Alchemical I/O
- **[Source & Transmutation](../docs/logos/alchemical-io.md)** — Narrative of inputs, process, and returns.



*Compliance: Record exists without implementation.*

<!-- ORGANVM:AUTO:END -->
