# /build — SiteOps Full Application Build Command

## What This Command Does

Triggers the complete autonomous multi-agent pipeline to generate the SiteOps application
from scratch. A single execution of this command causes all 11 agents to run in sequence
and in parallel, producing a fully working Spring Boot + React application.

---

## Usage

```
/build
```

No arguments required. The orchestrator reads `CLAUDE.md` for all configuration.

---

## Execution Trigger

When this command is run, immediately activate `@planner` with the following prompt:

> You are the master orchestrator for SiteOps — Contractor OS for India.
> Read `CLAUDE.md` fully. Then produce `PLAN.md` and `TASK_LIST.json`.
> After that, spawn all agents in the priority order defined in `CLAUDE.md`.
> Do not stop until `BUILD_REPORT.md` is written and all agents have produced
> their `AGENT_SUMMARY.md`. The system must be fully runnable via `docker compose up`.

---

## Pipeline Stages

```
Stage 1 — Planning
  @planner reads CLAUDE.md → writes PLAN.md + TASK_LIST.json

Stage 2 — Database
  @db → schema.sql, V1__initial_schema.sql, V2__seed_data.sql

Stage 3 — Core Generation (parallel)
  @backend  → siteops-backend/ (Spring Boot, all modules)
  @frontend → siteops-frontend/ (React 18 + Tailwind)

Stage 4 — Integrations (parallel)
  @payment-agent   → Razorpay wiring, invoice payment engine
  @whatsapp-agent  → Meta WhatsApp Business Cloud API notifications
  @report-agent    → iText 7 PDF builders (attendance, invoice, site summary)
  @i18n-agent      → en.json, hi.json, mr.json translations

Stage 5 — Security & Wiring
  @integrations → JWT filter chain, MinIO config, CORS, GlobalExceptionHandler

Stage 6 — Build Validation (checkpoint)
  @reviewer → compile backend, build frontend, verify integration wiring
  → produces REVIEW_REPORT.md (PASS/FAIL)
  → if FAIL: @planner re-runs the broken agent before proceeding

Stage 7 — Quality
  @tester → JUnit 5 + Testcontainers (backend), Vitest + RTL (frontend)

Stage 8 — Infrastructure
  @devops → docker-compose.yml, Dockerfiles, nginx.conf, CI/CD, .env.example
```

---

## Agent Spawning Rules

- Agents at the same stage run with `run_in_background: true`.
- Each agent reads its own `.claude/agents/<name>.md` for detailed instructions.
- Each agent loads relevant skills from `.claude/skills/` before generating code.
- No agent at stage N may start until ALL agents at stage N-1 have written `AGENT_SUMMARY.md`.
- `@reviewer` at stage 6 is a hard checkpoint — if it reports FAIL, `@planner` re-runs the broken agent before `@tester` starts.
- `@db` must complete before `@backend` — backend reads `schema.sql`.

---

## Skills Used Per Agent

| Agent | Skills Loaded |
|-------|--------------|
| `@db` | `db-schema/SKILL.md` |
| `@backend` | `db-schema/SKILL.md`, `api-generator/SKILL.md`, `claude-ai/SKILL.md` |
| `@frontend` | `frontend-scaffold/SKILL.md` |
| `@payment-agent` | `payment-engine/SKILL.md` |
| `@whatsapp-agent` | `whatsapp-notify/SKILL.md` |
| `@report-agent` | `report-generator/SKILL.md` |
| `@i18n-agent` | `frontend-scaffold/SKILL.md` |
| `@integrations` | `api-generator/SKILL.md` |

---

## MCP Tools Activated

| Tool | Activated By |
|------|-------------|
| `filesystem` | All agents (read/write source files) |
| `github` | `@devops` (create repo, push code, set up Actions) |
| `whatsapp-business` | `@whatsapp-agent` (send templated messages) |
| `razorpay` | `@payment-agent` (create payment links, verify webhooks) |

---

## Expected Outputs

After successful completion:

```
siteops/
├── PLAN.md                        # Architecture decisions by @planner
├── TASK_LIST.json                 # Agent task manifest
├── BUILD_REPORT.md                # Final build status + verification results
├── schema.sql                     # Full MySQL schema
├── migrations/
│   ├── V1__initial_schema.sql
│   ├── V2__seed_data.sql
│   └── V3__report_jobs_table.sql
├── siteops-backend/               # Complete Spring Boot application
├── siteops-frontend/              # Complete React application
├── docker-compose.yml
├── .env.example
├── .gitignore
├── README.md
└── .github/workflows/ci.yml
```

---

## Verification (run after build)

```bash
# Backend compiles
cd siteops-backend && mvn compile -q

# Frontend builds
cd siteops-frontend && npm install && npm run build

# Full stack runs
cp .env.example .env
docker compose up --build -d
docker compose ps

# Health check
curl http://localhost:8080/actuator/health
```

---

## Failure Recovery

If any agent fails:
1. `@planner` logs the failure in `PLAN.md` under `## Failures`.
2. `@planner` re-runs the failed agent with a corrective note appended to its prompt.
3. If re-run also fails, the failure is recorded in `BUILD_REPORT.md` with `Status: PARTIAL`.
4. All other agents continue — a single agent failure does not abort the pipeline.

---

## Seed Credentials (after docker compose up)

| Role | Phone | Password |
|------|-------|----------|
| Contractor Admin | 9876543210 | Test@1234 |
| Site Supervisor | 9876543211 | Test@1234 |

App: http://localhost:3000  
API: http://localhost:8080  
MinIO Console: http://localhost:9001
