# 00 — Master Project Overview: ClientDock

> **Last Audited:** September 2026  
> **Auditor:** Senior Technical/Product Architect  
> **Product Name:** ClientDock  
> **Repository:** `d:\Programming\ClientDock\ClientDock\SAAS`

---

## What Is ClientDock?

ClientDock is a **multi-tenant SaaS client portal** built for agencies, freelancers, and high-ticket service providers. It solves the painful problem of client onboarding: collecting files, credentials, approvals, and assets without the chaos of email chains, Slack threads, and Google Drive links.

### Core Value Proposition

> *"Stop chasing clients for files. Give them a simple, secure portal. Get everything you need, organized."*

### Target Market

| Segment | Description |
|---|---|
| **Primary** | Digital agencies (web, marketing, SEO, creative) |
| **Secondary** | Freelancers (developers, designers, copywriters) |
| **Tertiary** | Consultants and high-ticket service providers |
| **Geographic Focus** | US market (primary), India (secondary) |

---

## Tech Stack (Evidence-Based)

| Layer | Technology | Version |
|---|---|---|
| Framework | Next.js App Router | 16.2.9 |
| Language | TypeScript | ^5 |
| Database | SQLite via better-sqlite3 | ^12.11.1 |
| ORM | Drizzle ORM | ^0.45.2 |
| Auth | Better Auth | ^1.6.20 |
| File Upload | UploadThing | ^7.7.4 |
| Email | Resend | ^6.14.0 |
| Billing | Razorpay | ^2.9.8 |
| Caching/Rate Limit | Redis | ^6.0.0 |
| Styling | Tailwind CSS | v4 |
| Testing | Vitest | ^4.1.9 |
| Drag and Drop | @hello-pangea/dnd | ^18.0.1 |
| Icons | Lucide React | ^1.21.0 |
| Containerization | Docker + Docker Compose | — |
| SQLite Backup | Litestream | 0.3.13 |
| CI/CD | GitHub Actions | — |

---

## Application Architecture

```
User (Better Auth identity)
 └─ Agency (one per user, auto-created on signup)
     ├─ Client (invited by agency, unique portal token)
     │   ├─ Project (one per engagement)
     │   │   ├─ ChecklistItem (requirements with approval workflow)
     │   │   │   └─ Submission (client-uploaded files)
     │   │   ├─ ActivityLog (audit trail)
     │   │   └─ TimelineEvents (milestones)
     │   └─ ... (more projects)
     ├─ Lead (CRM pipeline entry)
     │   └─ LeadActivityLog
     ├─ Meeting (scheduled meetings)
     ├─ Notification (in-app notifications)
     └─ Subscription / Payment / WebhookEvent (billing)
```

### Multi-tenancy Model
Every meaningful database record is scoped to an `agency_id`. The application enforces this via:
1. `getCurrentSession()` → gets authenticated user + agency
2. `authorizeProjectAccess()` / `authorizeClientAccess()` → throws on cross-tenant access
3. All queries use `WHERE agency_id = ?` or join through client → agency

---

## Repository Structure

```
SAAS/
├── src/
│   ├── app/
│   │   ├── actions/         # Server actions (auth, projects, leads, email, notifications, settings)
│   │   ├── api/             # API routes (auth, uploadthing, cron/reminders, razorpay, clients, projects)
│   │   ├── dashboard/       # Agency dashboard (billing, analytics, leads, projects, clients, files, meetings, notifications, settings, account)
│   │   ├── portal/[token]/  # Client portal (passwordless token-based)
│   │   ├── blog/            # Marketing blog (7 articles)
│   │   ├── components/      # Landing page sections (Hero, Pricing, FAQ, Testimonials, Footer, etc.)
│   │   ├── legal/           # Privacy, Terms, GDPR, Cookies
│   │   ├── login/           # Login page
│   │   ├── signup/          # Signup page
│   │   └── [other pages]    # about, careers, changelog, community, contact, docs, help, r/
│   ├── components/          # Shared UI (auth forms, billing badges, sidebar)
│   ├── db/                  # Schema, migrations, seeds
│   ├── lib/                 # Utilities (auth, db helpers, data, billing, rate limit, redis, razorpay, uploadthing)
│   ├── styles/              # Global CSS
│   └── test/                # Test files and mock data
├── drizzle/                 # SQL migrations (6 migrations)
├── scripts/                 # Setup scripts (litestream)
├── data/                    # SQLite database storage
├── Dockerfile               # Multi-stage production Docker build
├── docker-compose.yml       # Docker Compose with data volume
└── .github/workflows/       # CI pipeline
```

---

## Current Completion Assessment

> **Evidence-Based Estimate: ~60–65% Complete for MVP Launch**

| Area | Completion | Notes |
|---|---|---|
| Auth (Login/Signup/OAuth) | 90% | Working; email verification disabled in prod config |
| Agency Dashboard | 75% | Working; revenue data mocked |
| Client Portal | 85% | Working; file submission and approval flow functional |
| Checklist/Approval Workflow | 85% | Working; good implementation |
| File Upload (UploadThing) | 80% | Route exists; no auth middleware on uploader |
| Notifications | 70% | In-app notifications work; deadline check not wired to cron |
| Leads CRM | 70% | Full CRUD working; convert-to-client works |
| Analytics | 60% | Static computed metrics; no time-series; plan-gated |
| Meetings | 40% | DB schema exists; calendar UI exists; no real DB queries |
| Billing (Razorpay) | 55% | Webhook handler exists; subscription creation API exists; webhook emails use hardcoded addresses |
| Settings | 65% | Notification preferences work; profile editing limited |
| Landing Page | 80% | Full sections; testimonials/users are static/demo |
| Blog | 70% | 7 articles exist; static content |
| Legal Pages | Unknown | Dirs exist; content not verified |
| Testing | 40% | 61 tests claimed; coverage unclear |
| Docker/Deployment | 70% | Dockerfile good; litestream config missing from image |
| Documentation | 0% → 100% | This audit fills the gap |

---

## What Already Works Well

1. **Multi-tenant isolation** — Agency scoping is enforced consistently at the server action layer
2. **Auth system** — Better Auth with email/password + optional OAuth is solid
3. **Client portal** — Token-based passwordless access is elegant and client-friendly
4. **Checklist/approval workflow** — Core flow (add item → client uploads → agency approves/rejects) is well-implemented
5. **Automated reminders** — Cron endpoint is real and functional with CRON_SECRET auth
6. **Database schema** — Well-designed with proper cascades, indexes, and tenant isolation
7. **Plan enforcement** — `check-plan.ts` and `plan-config.ts` provide a clean feature-gating system
8. **Error handling patterns** — `withDbErrorHandling`, `ActionResult<T>` type, and Zod validation are consistent
9. **Docker setup** — Multi-stage Dockerfile with Litestream is production-grade thinking
10. **Leads CRM** — Full pipeline with stage tracking, convert-to-client, and activity logging

## What Is Broken or Incomplete

1. **Revenue data on dashboard is mocked** — `Math.random()` used for revenue chart
2. **Meeting scheduler is disconnected** — Hardcoded demo meetings, no DB queries
3. **Billing webhook emails use hardcoded addresses** — `billing@example.com` hardcoded
4. **Portal token expiry not regenerated on reminder** — tokens expire in 30 days; no long-running refresh
5. **UploadThing middleware has no auth** — Any user can upload to any submission slot
6. **Rate limiting requires Redis** — If Redis is unavailable, requests are blocked (fail-closed, but this breaks dev)
7. **`softDeleteLeadsAction` has no agency ownership check** — Security gap
8. **Progress field is stored but also computed** — Inconsistency between `progress` column and `calculateProjectProgress()`
9. **Billing cancellation webhook does not immediately downgrade plan** — Only sets subscription status, not agency.plan
10. **`checkProjectLimit` has a bug** — The `where` clause is incomplete (missing status filter)

---

## Priority Action List (High Level)

```
P0 — Fix security gaps (upload auth, lead delete ownership check)
P0 — Fix billing webhook email addresses
P1 — Fix meeting scheduler (wire to DB)
P1 — Fix revenue chart (real data or clear mock label)
P1 — Fix checkProjectLimit bug
P1 — Enable email verification in production
P2 — Add portal token rotation strategy for long-lived projects
P2 — Harden UploadThing with per-submission authorization
P2 — Fix Redis fail-closed behavior for development
P3 — Add time-series data for analytics
P3 — Complete settings page (profile editing)
P4 — White-labeling features
P4 — Team members feature
```

---

## Document Index

| Document | Path |
|---|---|
| Current State | [01-CURRENT-STATE.md](./01-CURRENT-STATE.md) |
| Codebase Understanding | [02-CODEBASE-UNDERSTANDING.md](./02-CODEBASE-UNDERSTANDING.md) |
| Feature Inventory | [03-FEATURE-INVENTORY.md](./03-FEATURE-INVENTORY.md) |
| Completion Assessment | [04-COMPLETION-ASSESSMENT.md](./04-COMPLETION-ASSESSMENT.md) |
| Database Overview | [database/DATABASE-OVERVIEW.md](./database/DATABASE-OVERVIEW.md) |
| Database Schema | [database/DATABASE-SCHEMA.md](./database/DATABASE-SCHEMA.md) |
| Database Security | [database/DATABASE-SECURITY.md](./database/DATABASE-SECURITY.md) |
| Bugs | [bugs/BUGS.md](./bugs/BUGS.md) |
| UX Audit | [ux/UX-AUDIT.md](./ux/UX-AUDIT.md) |
| Security Audit | [security/SECURITY-AUDIT.md](./security/SECURITY-AUDIT.md) |
| Architecture Audit | [architecture/ARCHITECTURE-AUDIT.md](./architecture/ARCHITECTURE-AUDIT.md) |
| Product Understanding | [product/PRODUCT-UNDERSTANDING.md](./product/PRODUCT-UNDERSTANDING.md) |
| Monetization | [product/MONETIZATION.md](./product/MONETIZATION.md) |
| US Market | [research/US-MARKET.md](./research/US-MARKET.md) |
| Competitor Analysis | [research/COMPETITOR-ANALYSIS.md](./research/COMPETITOR-ANALYSIS.md) |
| Infrastructure | [infrastructure/CURRENT-INFRASTRUCTURE.md](./infrastructure/CURRENT-INFRASTRUCTURE.md) |
| Master Roadmap | [ROADMAP.md](./ROADMAP.md) |
