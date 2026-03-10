# SimplyRaffle

**Multi-tenant SaaS platform for school and nonprofit raffle events.**
Each customer gets an isolated instance with their own database, subdomain, and admin panel — provisioned automatically, zero DevOps required on their end.

Live at **[simplyraffle.com](https://www.simplyraffle.com)** · Demo at **[demo.simplyraffle.com](https://demo.simplyraffle.com)**

> Private repository — this README documents the architecture and engineering decisions. Source available on request.

---

## What It Does

Manages the full raffle lifecycle: participant onboarding → ticket distribution → prize selection → weighted random draw → winner notification → post-event reporting.

**For administrators:** Dashboard with real-time stats, bulk CSV import (AI-powered column detection), email blasts with scheduling/cooldowns, configurable draw modes, live ceremony screen, and PDF/print reports.

**For participants:** Magic-link access (no passwords, no accounts). Pick prizes, allocate tickets, view odds in real time from any device.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Cloudflare DNS                        │
│              *.simplyraffle.com → Railway                │
└────────────────────────┬────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌───────────┐   ┌───────────┐   ┌───────────┐
   │  Tenant A │   │  Tenant B │   │  Tenant C │
   │  Next.js  │   │  Next.js  │   │  Next.js  │
   │  + Prisma │   │  + Prisma │   │  + Prisma │
   │  + PG DB  │   │  + PG DB  │   │  + PG DB  │
   └───────────┘   └───────────┘   └───────────┘
         │               │               │
         └───────────────┼───────────────┘
                         │
              ┌──────────┴──────────┐
              │    Shared Infra     │
              │  Resend (email)     │
              │  Railway (hosting)  │
              │  Anthropic (AI)     │
              └─────────────────────┘
```

**Isolation model:** Each tenant runs its own container + PostgreSQL database on Railway. No shared data, no cross-tenant access. Provisioning is fully automated via a CLI tool that creates Railway projects, sets env vars, runs migrations, and configures DNS — a new customer is live in under 60 seconds.

---

## Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 14 (App Router, Server Components) |
| Language | TypeScript (strict, zero `tsc` errors across 30k LOC) |
| Styling | Tailwind CSS |
| ORM / DB | Prisma 5 · PostgreSQL (8 models, 25 migrations) |
| Email | Resend (batch API, 7 templates, family grouping) |
| Auth | iron-session (admin) · magic-link tokens (participants) |
| AI | Anthropic Claude Haiku (CSV column detection) |
| Infra | Railway (containers + managed Postgres) · Cloudflare (DNS + SSL) |
| Validation | Zod (all 54 API routes) |

---

## Engineering Highlights

### Security
- **Timing-safe secret comparison** (`crypto.timingSafeEqual`) on all service-to-service auth (cron, backups)
- **CSRF protection** via `X-Requested-With` header enforcement on all mutating requests
- **iron-session** encrypted cookies with 30-day TTL; middleware verifies on every request
- **Security headers** on all responses: `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Cache-Control: no-store`
- **Password reset flow** with time-limited tokens and bcrypt hashing
- **Student tokens opt-in** — API never returns magic-link tokens unless explicitly requested

### Data Integrity
- **Atomic transactions** for all multi-step mutations (bulk imports, entry submissions, draw execution, factory reset)
- **Atomic JSON updates** via raw PostgreSQL `jsonb_set` for email blast history (no read-modify-write races)
- **Plan-based participant limits** enforced at API level — unknown plan values fail closed to most restrictive tier
- **Profanity filter** on participant imports

### Performance
- **Dashboard polling** pauses when tab is hidden, uses `AbortController` to cancel in-flight requests
- **Batch email API** — 100 emails per Resend call, 400 recipients in ~3 seconds
- **Conditional timers** — countdown intervals only run when actually needed
- **Client-side redirect guard** returns `null` during redirect to prevent UI flash

### AI-Powered Import
- Raw CSV/Excel grid sent to Claude Haiku for header detection and column mapping
- Confidence scoring with fallback to heuristic matching when no API key
- Below-threshold imports surface an amber banner with "Get help" escalation

### Multi-Tenant Provisioning
- CLI provisioner creates Railway project + Postgres DB + env vars + DNS in one command
- Plan limits (`community` / `starter` / `pro`) synced from env vars to DB on every container boot
- Per-customer overrides via `EXTRA_PARTICIPANTS` env var
- Automated backup/restore across instances

---

## Scale

| Metric | Count |
|---|---|
| TypeScript source files | 168 |
| Lines of code | 30,000+ |
| API routes | 54 |
| React components | 42 |
| Prisma models | 8 |
| Database migrations | 25 |
| Commits | 187 |
| Development time | ~3 weeks (solo) |

---

## Key Screens

| Screen | What it does |
|---|---|
| **Admin Dashboard** | Real-time stats, phase-aware checklist, one-click draw |
| **Setup Wizard** | Guided first-run configuration (enforced for new instances) |
| **Participant Import** | AI-powered CSV parsing with confidence scoring |
| **Prize Management** | Drag-to-reorder, image uploads, per-prize ticket limits |
| **Live Ceremony** | Projector-ready draw animation with round-by-round reveals |
| **Student Portal** | Mobile-first ticket allocation with real-time odds display |
| **Reports** | PDF-ready summary, unused ticket reminders, draw audit log |
| **Settings** | Branding, scheduling, email templates, draw mode configuration |

---

## Draw Modes

| Mode | Behavior |
|---|---|
| **Open** | Anyone can enter any prize regardless of ticket balance |
| **Fair First** | Students who haven't won yet get priority in early rounds |
| **Exclusive** | Only students with allocated tickets are eligible |

All modes use cryptographically weighted random selection proportional to ticket allocation.

---

> Source available on request for code review or technical discussion.
