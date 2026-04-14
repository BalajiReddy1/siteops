# SiteOps — Contractor OS for India
## Orchestrator Master Configuration

---

## Project Overview

SiteOps is a vertical SaaS platform for small-to-mid construction contractors in India.
It solves three deeply painful, undigitised problems:

1. **Worker attendance and daily wage ledger** — contractors manage 10–50 daily-wage
   workers across multiple active sites, paying ₹600–700/day, with partial withdrawals
   and monthly settlements — all currently on handwritten cards or WhatsApp.

2. **Invoice and payment tracking** — 70% of contractors face payment delays beyond
   30 days from builders. No invoice trail = no leverage = cash flow crises.

3. **Site progress reporting** — running 3–5 sites across different cities with no
   unified view, no shared system between site, office, and builder.

The platform starts as a daily ops tool (worker ledger + invoicing + site tracker),
and is architected to evolve into a verified contractor marketplace.

Target user: Small-to-mid construction contractor in India (civil, tiles, waterproofing,
finishing), managing 1–10 active sites, 10–100 daily-wage workers.

---

## Tech Stack

| Layer            | Technology                                      |
|------------------|-------------------------------------------------|
| Frontend         | React 18 + Tailwind CSS + Vite                  |
| Backend          | Spring Boot 3.x (Java 17)                       |
| Database         | MySQL 8                                         |
| Auth             | JWT (access + refresh tokens)                   |
| File Storage     | MinIO (S3-compatible)                           |
| Payments         | Razorpay API                                    |
| Notifications    | WhatsApp Business Cloud API (Meta)              |
| PDF Reports      | iText 7                                         |
| AI Features      | Claude API (Anthropic) — smart reminders, cost estimator, invoice auto-fill |
| Localisation     | i18next (frontend), MessageSource (backend)     |
| Containerisation | Docker Compose                                  |
| Build Tools      | Maven (backend), Vite (frontend)                |
| Migrations       | Flyway                                          |

---

## Agent Roster

When given the single build prompt, activate agents in this exact order:

| Agent              | Role                                                         | Priority |
|--------------------|--------------------------------------------------------------|----------|
| `@planner`         | Reads CLAUDE.md, produces PLAN.md + TASK_LIST.json           | 1        |
| `@db`              | MySQL schema, Flyway migrations, seed data                   | 2        |
| `@backend`         | Spring Boot entities, repos, services, REST controllers      | 3 (parallel) |
| `@frontend`        | React pages, components, hooks, Zustand store                | 3 (parallel) |
| `@payment-agent`   | Razorpay integration, invoice engine, payment tracking       | 4        |
| `@whatsapp-agent`  | WhatsApp Business API notifications to workers/builders      | 4        |
| `@report-agent`    | iText PDF: attendance reports, invoice PDFs, site summaries  | 4        |
| `@i18n-agent`      | Marathi + Hindi translations for all UI strings              | 4        |
| `@integrations`    | JWT security config, MinIO wiring, external API plumbing     | 5        |
| `@reviewer`        | Compile backend, build frontend, verify integration wiring   | 6        |
| `@tester`          | JUnit 5 + Testcontainers (backend), Vitest + RTL (frontend)  | 7        |
| `@devops`          | Docker Compose, GitHub Actions CI/CD, .env.example           | 8        |

Agents at the same priority level run **in parallel** using `run_in_background: true`.

---

## Execution Order

```
STEP 1 ──► @planner          (reads this file, produces PLAN.md + TASK_LIST.json)
STEP 2 ──► @db               (schema, migrations, seed data)
STEP 3 ──► @backend          ─┐
           @frontend          ─┤  parallel
STEP 4 ──► @payment-agent    ─┐
           @whatsapp-agent    ─┤  parallel
           @report-agent      ─┤
           @i18n-agent        ─┘
STEP 5 ──► @integrations     (wires all external APIs + security)
STEP 6 ──► @reviewer         (compile, build, verify integration — checkpoint)
STEP 7 ──► @tester           (writes all tests)
STEP 8 ──► @devops           (docker, CI/CD, env)
```

---

## Domain Model

### Core Entities

```
Contractor
├── has many Sites
├── has many Workers
├── has many Invoices
└── belongs to Organisation (future: marketplace)

Site
├── belongs to Contractor
├── has many Workers (via SiteAssignment)
├── has many AttendanceRecords
├── has many ProgressUpdates
└── has many Invoices

Worker
├── belongs to Contractor
├── has many AttendanceRecords
├── has many WageWithdrawals
└── has a WageLedger (computed)

AttendanceRecord
├── belongs to Site
├── belongs to Worker
├── date, status (PRESENT/ABSENT/HALF_DAY), daily_rate, notes

WageWithdrawal
├── belongs to Worker
├── amount, withdrawal_date, recorded_by, notes

Invoice
├── belongs to Site
├── belongs to Contractor
├── builder_name, builder_contact, amount_due, amount_paid
├── status (DRAFT/SENT/PARTIALLY_PAID/PAID/OVERDUE)
├── due_date, pdf_file_id

ProgressUpdate
├── belongs to Site
├── photos (via Attachments), description, milestone_label
├── reported_by, reported_at

Notification
├── belongs to Worker or Builder (polymorphic)
├── channel (WHATSAPP/SMS), message, status, sent_at

File
├── owner_type, owner_id (polymorphic)
├── storage_key (MinIO), filename, content_type, size_bytes

AuditLog
├── actor_id, action, entity_type, entity_id, change_set (JSON)
```

---

## Module Boundaries

```
siteops-backend/
└── src/main/java/com/siteops/
    ├── auth/           # JWT, refresh token, Spring Security
    ├── contractor/     # Contractor profile, settings
    ├── site/           # Site CRUD, assignment, status
    ├── worker/         # Worker profile, wage rate
    ├── attendance/     # Daily attendance marking, half-day logic
    ├── wages/          # Wage ledger computation, withdrawals
    ├── invoice/        # Invoice lifecycle, Razorpay payment links
    ├── progress/       # Site progress updates, photo attachments
    ├── notification/   # WhatsApp Business API dispatcher
    ├── reports/        # iText PDF generation
    ├── files/          # MinIO upload/download/presigned URLs
    ├── ai/             # Claude API: cost estimator, smart reminders, invoice auto-fill
    └── audit/          # AOP-based audit logging

siteops-frontend/
└── src/
    ├── api/            # Axios client + React Query hooks
    ├── store/          # Zustand: auth, contractor context
    ├── components/     # ui/, layout/, shared/
    ├── features/
    │   ├── auth/
    │   ├── dashboard/
    │   ├── sites/
    │   ├── workers/
    │   ├── attendance/
    │   ├── wages/
    │   ├── invoices/
    │   ├── progress/
    │   └── reports/
    ├── pages/          # Route-level components
    └── i18n/           # en.json, hi.json, mr.json
```

---

## Key Business Rules

### Attendance & Wages
- Daily rate is set per worker per site (not global) — same worker can earn ₹650 at Site A and ₹700 at Site B.
- Half-day = 50% of daily rate.
- Wage ledger = SUM(attendance earnings) - SUM(withdrawals). Never negative displayed — show alert if withdrawal would overdraw.
- Bulk attendance marking: contractor marks all workers for a site in one screen — default PRESENT, tap to toggle ABSENT or HALF_DAY.
- Attendance can be marked offline and synced when connectivity returns (PWA with service worker).

### Invoices
- Invoice can be created in DRAFT, reviewed, then sent to builder (changes status to SENT).
- When builder pays partial: record partial payment, status → PARTIALLY_PAID, remaining balance computed automatically.
- Invoices overdue > 30 days: system flags them automatically via scheduled job.
- Razorpay payment link generated per invoice — contractor can share WhatsApp link to builder.

### WhatsApp Notifications
- Worker receives WhatsApp message on payday (monthly settlement) with breakdown: days worked, withdrawals, net payable.
- Builder receives WhatsApp message when invoice is sent, with payment link.
- Contractor receives WhatsApp alert when invoice becomes overdue.
- All messages use Meta WhatsApp Business Cloud API with approved templates.

### Site Progress
- Progress update = text description + milestone label (e.g. "Foundation complete", "First floor slab") + up to 5 photos.
- Photos stored in MinIO under `siteops-progress` bucket.
- Builder can be given a read-only share link to view site progress (no login required for viewer link).

### Reports (PDF)
- **Attendance Report**: per site per month — worker name, days present, days absent, half-days, earnings, withdrawals, net payable. Generated by `@report-agent` using iText 7.
- **Invoice PDF**: professional invoice with contractor details, site address, line items, totals, payment status, Razorpay QR code.
- **Site Summary Report**: per site — all progress milestones, photo thumbnails, invoice history, timeline.

### Localisation
- UI available in English, Hindi (हिंदी), and Marathi (मराठी).
- Language selected on first login, stored in user profile, switchable from settings.
- All number formatting uses Indian locale (e.g. ₹1,23,456 not ₹123,456).
- `@i18n-agent` generates translation JSON files for all three languages.

### Security
- JWT access token: 15-minute expiry. Refresh token: 7-day expiry, stored httpOnly cookie.
- Role hierarchy: `SUPER_ADMIN > CONTRACTOR_ADMIN > SITE_SUPERVISOR > VIEWER`
- SITE_SUPERVISOR can mark attendance and update progress but cannot create/edit invoices.
- VIEWER can only read — used for builder progress-view links.
- All write operations logged to audit_logs via Spring AOP.

---

## MCP Tool Integrations

| MCP Tool              | Used By              | Purpose                                              |
|-----------------------|----------------------|------------------------------------------------------|
| `filesystem`          | All agents           | Read/write generated source files                    |
| `github`              | `@devops`            | Create repo, commit files, set up Actions workflows  |
| `whatsapp-business`   | `@whatsapp-agent`    | Send templated messages to workers and builders      |
| `razorpay`            | `@payment-agent`     | Create payment links, verify webhook signatures      |

---

## REST API Surface (abbreviated)

### Auth
- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `POST /api/auth/logout`

### Sites
- `GET/POST /api/v1/sites`
- `GET/PUT/DELETE /api/v1/sites/{id}`
- `GET /api/v1/sites/{id}/dashboard` ← aggregated: workers, attendance %, invoice status

### Workers
- `GET/POST /api/v1/workers`
- `GET/PUT/DELETE /api/v1/workers/{id}`
- `GET /api/v1/workers/{id}/ledger` ← computed wage ledger

### Attendance
- `GET/POST /api/v1/attendance`
- `POST /api/v1/attendance/bulk` ← mark entire site in one call
- `GET /api/v1/attendance/site/{siteId}/date/{date}`

### Wages
- `GET /api/v1/wages/worker/{workerId}/summary`
- `POST /api/v1/wages/withdrawals`
- `POST /api/v1/wages/settle/{workerId}` ← monthly settlement, triggers WhatsApp

### Invoices
- `GET/POST /api/v1/invoices`
- `GET/PUT/DELETE /api/v1/invoices/{id}`
- `POST /api/v1/invoices/{id}/send` ← sends to builder via WhatsApp
- `POST /api/v1/invoices/{id}/payment` ← record payment
- `GET /api/v1/invoices/{id}/payment-link` ← Razorpay link

### Progress
- `GET/POST /api/v1/sites/{id}/progress`
- `POST /api/v1/sites/{id}/progress/{updateId}/photos`
- `GET /api/v1/sites/{id}/share/{token}` ← public read-only (no auth)

### Reports
- `POST /api/v1/reports/attendance` ← async, returns job ID
- `POST /api/v1/reports/invoice/{invoiceId}`
- `POST /api/v1/reports/site-summary/{siteId}`
- `GET /api/v1/reports/{jobId}/download`

### AI Features
- `POST /api/v1/ai/estimate` ← Claude-powered cost estimation (work type + area + city)
- `POST /api/v1/ai/invoice-suggest` ← Claude-generated invoice line items
- `GET /api/v1/ai/reminder-preview/{invoiceId}` ← AI-generated payment reminder preview

### Notifications
- `GET /api/v1/notifications` ← notification history for contractor

---

## Verification Commands

After generation, each agent must verify:

```bash
# Backend
cd siteops-backend && mvn compile -q
cd siteops-backend && mvn test -q

# Frontend
cd siteops-frontend && npm install && npm run build

# DB
mysql -u root -p siteops < schema.sql

# Full stack
docker compose up --build -d && docker compose ps
```

---

## Output Contract

Every agent MUST:
1. Generate all source files in the correct directory structure.
2. Produce `AGENT_SUMMARY.md` listing what was generated and any assumptions.
3. Use only environment variables for secrets — never hardcode credentials.
4. Ensure generated code compiles and runs — no placeholder `// TODO` stubs in critical paths.
5. Write production-quality code — not scaffolding. Real validation, real error handling.
