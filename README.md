# ClientDock Documentation

This folder contains the documentation for ClientDock, a multi-tenant SaaS product for agency client onboarding, asset collection, checklist-based review, approvals, and automated follow-up.

This README is a factual index of the existing `.md` documents. It summarizes the current product, codebase, architecture, status, risks, research, business model, and future direction. Detailed evidence remains in the linked source documents.

## Document Index

### Project and codebase

- [00 — Master Project Overview](./00-MASTER-PROJECT-OVERVIEW.md): product definition, stack, architecture, repository structure, maturity, strengths, and priorities.
- [01 — Current State](./01-CURRENT-STATE.md): evidence-based status of every main product area.
- [02 — Codebase Understanding](./02-CODEBASE-UNDERSTANDING.md): routes, server actions, libraries, database layer, patterns, and implementation notes.
- [03 — Feature Inventory](./03-FEATURE-INVENTORY.md): feature-by-feature functional and experience status.
- [04 — Completion Assessment](./04-COMPLETION-ASSESSMENT.md): launch-readiness assessment and production blockers.
- [ROADMAP](./ROADMAP.md): implementation tasks and planned phases.

### Technical audits

- [Architecture Audit](./architecture/ARCHITECTURE-AUDIT.md): technical decisions to preserve, debt, performance, and data-model recommendations.
- [Database Overview](./database/DATABASE-OVERVIEW.md): SQLite configuration, tables, relationships, migrations, and schema risks.
- [Bug Report](./bugs/BUGS.md): confirmed bugs, impact, fixes, and tests.
- [Security Audit](./security/SECURITY-AUDIT.md): security findings, strengths, priority order, and pre-production checklist.
- [UX Audit](./ux/UX-AUDIT.md): agency/client experience, completion funnel, friction, and UX remediation.
- [Infrastructure Guide](./infrastructure/CURRENT-INFRASTRUCTURE.md): setup, environment, Docker, Litestream, cron, CI/CD, monitoring, and deployment.

### Product and business

- [Product Understanding](./product/PRODUCT-UNDERSTANDING.md): problem, users, workflows, positioning, gaps, and opportunities.
- [Monetization](./product/MONETIZATION.md): plans, billing, conversion strategy, pricing, and retention.

### Research

- [Research Gap Audit](./research/00-RESEARCH-GAP-AUDIT.md): research requirements, existing evidence, gaps, and planned work.
- [ClientDock Baseline](./research/01-CLIENTDOCK-BASELINE.md): product reconstruction, segments, workflow, maturity, and hypotheses.
- [Market Category Analysis](./research/02-MARKET-CATEGORY-ANALYSIS.md): categories, competitors, search intent, and positioning.
- [Pricing and ROI Model](./research/08-PRICING-AND-ROI-MODEL.md): cost of manual intake, willingness to pay, packaging, and upgrade drivers.
- [Opportunity Scoring and Decisions](./research/09-OPPORTUNITY-SCORING-AND-DECISIONS.md): scored opportunities and Build/Improve/Integrate/Experiment/Wait/Don't Build decisions.
- [Final Product Strategy](./research/10-FINAL-PRODUCT-STRATEGY.md): strategic synthesis, phases, metrics, and negative scope.
- [Competitor Analysis](./research/COMPETITOR-ANALYSIS.md): direct competitors, portals, agency suites, DIY alternatives, pricing, and review insights.
- [US Market](./research/US-MARKET.md): US market, ICP, buyer pain, compliance, pricing, integrations, and GTM.
- [India Market](./research/INDIA-MARKET.md): Indian agency segments, WhatsApp, UPI/Razorpay, GST, pricing, and regional strategy.

## Product Summary

ClientDock is a client intake and onboarding automation portal for:

- Digital agencies.
- Freelancers and solo consultants.
- Web design and development agencies.
- SEO and performance marketing agencies.
- Branding and creative studios.
- High-ticket service providers.

The problem is that agencies need clients to provide files, credentials, content, approvals, and project prerequisites. These materials usually arrive through email, Slack, WhatsApp, Google Drive, Dropbox, WeTransfer, and scattered conversations. This creates manual chasing, wrong file formats, poor visibility, project delays, and delayed milestone billing.

The recommended category is **client intake and onboarding automation**, also called a **client asset intake portal** or **agency kickoff gateway**.

ClientDock is not intended to become a general project-management tool, Slack replacement, full CRM, generic file store, or full accounting platform.

## Core Workflows

### Agency

1. Sign up or log in.
2. Create an agency workspace, client, and project.
3. Define checklist items, instructions, and accepted formats.
4. Share the portal link.
5. Monitor submissions and progress.
6. Approve or reject each item with feedback.
7. Send automated reminders for missing items.
8. Start production when intake is complete.

### Client

1. Receive a portal link.
2. Open the portal without creating an account.
3. View project context and checklist items.
4. Upload files or provide requested information.
5. Return later through the token link.
6. See approvals, rejection reasons, and revisions.
7. Resubmit rejected items.

### Lead conversion

1. Add a lead to the CRM pipeline.
2. Move it through stages.
3. Convert a won lead into a client.
4. Create a project and portal for that client.

## Technology and Architecture

| Layer | Technology |
|---|---|
| Framework | Next.js App Router 16.2.9 |
| Language | TypeScript 5 |
| Database | SQLite with better-sqlite3 12.11.1 |
| ORM | Drizzle ORM 0.45.2 |
| Auth | Better Auth 1.6.20 |
| Uploads | UploadThing 7.7.4 |
| Email | Resend 6.14.0 |
| Billing | Razorpay 2.9.8 |
| Cache/rate limiting | Redis 6.0.0 |
| Styling | Tailwind CSS v4 |
| Testing | Vitest 4.1.9 |
| Deployment | Docker and Docker Compose |
| SQLite backup | Litestream 0.3.13 |
| CI/CD | GitHub Actions |

```text
Better Auth user
  └─ Agency tenant
      ├─ Clients
      │   └─ Projects
      │       ├─ Checklist items
      │       │   └─ File submissions
      │       ├─ Activity logs
      │       └─ Timeline events
      ├─ Leads and lead activity
      ├─ Meetings
      ├─ Notifications
      └─ Subscriptions, payments, and webhook events
```

The application is a Next.js monolith. The dashboard uses authenticated sessions and Server Actions. The client portal uses a hashed, expiring token. SQLite is accessed through Drizzle. Agency data is intended to be isolated with `agency_id` scoping and authorization helpers.

Important code patterns:

- `getCurrentSession()` resolves the current agency.
- `authorizeProjectAccess()` and `authorizeClientAccess()` protect tenant-owned resources.
- Server actions validate with Zod, authenticate, authorize, enforce plan limits, write to the database, revalidate paths, and return `ActionResult<T>`.
- `plan-config.ts` is the intended single source of truth for plan limits and features.
- `instrumentation.ts` runs database migrations at startup.

## Current Status

The audits estimate ClientDock at approximately **60–65% complete** or **about 62% launch-ready**. The core workflow exists, but production readiness is blocked by security, billing, notification, infrastructure, and data-trust problems.

### Working or mostly working

- Signup, login, sessions, and optional OAuth.
- Automatic agency creation on signup.
- Multi-tenant authorization patterns.
- Client and project creation and editing.
- Checklist creation and deletion.
- Hashed, expiring passwordless portal tokens.
- Client file submission.
- Agency approval and rejection.
- Internal notes, activity logs, and project status.
- Portal token rotation.
- Leads pipeline and lead conversion.
- Notification list and read states.
- Basic dashboard and analytics metrics.
- Razorpay subscription creation and webhook verification.
- Docker build and migration foundation.

### Incomplete or untrustworthy

- Email verification is disabled.
- Auth rate limiting exists but is not applied.
- UploadThing upload authorization is missing.
- Lead deletion lacks agency ownership verification.
- Dashboard revenue uses random data.
- Dashboard meetings use hardcoded demo data.
- Stored project progress can diverge from checklist progress.
- Client invitation emails are missing.
- Client rejection emails are missing.
- Upload and deadline notifications are incomplete.
- Portal tokens expire after 30 days without complete recovery.
- Accepted file formats are not enforced server-side.
- Meetings lack complete server actions, queries, and reminders.
- Analytics has no historical trends, real revenue, or lead analytics.
- Billing webhook emails use a hardcoded address.
- Cancellation does not correctly update plan access.
- Billing UI checks invalid payment status values.
- Project limits count completed projects incorrectly.
- Stripe is missing for US/international customers.
- Team members, templates, storage sync, white labeling, and review collection are future work.
- Litestream configuration is missing from the Docker image.
- Monitoring, error tracking, and staging are missing.

## Feature Status

| Feature | Status | Main missing work |
|---|---|---|
| Authentication | Mostly complete | Verification and rate limits |
| OAuth | Mostly complete, environment controlled | Production testing and feedback |
| Agency creation | Complete | No major gap identified |
| Dashboard | Functional but weak | Real revenue and meeting data |
| Client invitation | Mostly complete | Automated email |
| Passwordless portal | Mostly complete | Upload auth and token recovery |
| Project creation | Complete with limit bug | Correct active-project query |
| Checklist | Mostly complete | Format enforcement and templates |
| File submission | Partial | Authorization, metadata, notifications |
| Approval/rejection | Mostly complete | Progress update and client email |
| Internal notes | Complete | Save indicator is optional |
| Reminders | Mostly complete | Scheduler and durable reminder state |
| Timeline | Partial | Verify CRUD |
| Leads CRM | Mostly complete | Ownership fix, CSV import later |
| Notifications | Partial | Upload/deadline triggers and email |
| Analytics | Partial | Historical and revenue data |
| Meetings | Frontend-only or placeholder | Server actions, queries, reminders |
| Billing | Partial | Webhooks, gateway, status enum |
| Settings | Partial | Profile, agency, password, account controls |
| Reviews | Schema exists | Request, collect, and display workflow |
| Team members | Planned/schema concepts | Invitations, roles, permissions |
| Templates | Planned | Reusable service checklists |
| White labeling | Future | Domain, logo, and colors |
| Storage sync | Future | Google Drive and Dropbox |
| Client chat | Stub or intentionally out of scope | Do not build as a Slack replacement |

## Confirmed Bugs and Security Findings

1. UploadThing accepts client uploads without a valid portal-token check.
2. Lead deletion updates IDs without checking the current agency.
3. Billing webhook email recipients are hardcoded to `billing@example.com`.
4. Subscription cancellation updates status but not agency plan access.
5. Billing UI checks `success` and `paid`, while the database uses `created`, `authorized`, `captured`, `failed`, and `refunded`.
6. `checkProjectLimit` does not exclude completed projects.
7. Revenue chart values are generated with `Math.random()`.
8. Dev bypass and test database routes may be exposed in production.
9. Email verification is disabled.
10. Auth rate limiting is not applied.
11. Webhook errors return HTTP 200, which can suppress retries.
12. Portal token verification does not decide access based on project status.
13. Rejection reasons must remain plain text when rendered.
14. CORS is not explicitly restricted.

Security strengths recorded by the audits:

- Agency scoping is used throughout the server-side data layer.
- Portal tokens are hashed with SHA-256 and have expiry dates.
- Payment and webhook signatures are verified.
- Webhook idempotency is implemented.
- Server actions validate input.
- Authorization helpers exist for projects and clients.
- Cron uses `CRON_SECRET` bearer authentication.
- Agency deletion cascades tenant data.
- Authentication errors are intended not to leak account existence.

## Database Facts

The database uses SQLite in WAL mode through better-sqlite3 and Drizzle.

- `synchronous = NORMAL`.
- Temporary storage is in memory.
- Memory mapping is configured to 256 MB.
- `DATABASE_PATH` is configurable and defaults to `./data/clientdock.db`.
- Six migrations are identified as `0000` through `0005`.
- Migrations run automatically at startup.

Main table groups:

- Auth: `user`, `session`, `account`, `verification`.
- Tenant and project: `agency`, `client`, `project`, `checklist_item`, `submission`.
- History: `activity_log`, `timeline_events`.
- CRM: `leads`, `lead_activity_log`.
- Communication: `notification`, `meeting`.
- Billing: `subscription`, `payment`, `webhookEvent`.
- Social proof: `review`.

Database risks:

- Application dates use ISO text while Better Auth and leads use integer timestamps.
- `project.budget` is text rather than a numeric field.
- Progress is both stored and computed.
- Accepted formats are a comma-separated string and are not enforced.
- `required_files` and `checklist_data` may be unused legacy fields.
- Submission file size may not be reliably stored.
- Submissions do not record who submitted them.
- Reminder state is inferred from activity-log text.
- Some query paths may need composite indexes.
- The files page has an N+1 query pattern.

The architecture audit recommends keeping SQLite at the current scale. PostgreSQL is a later option if concurrency grows substantially. A monolith-to-microservices migration is not currently recommended.

## UX and Product Experience

ClientDock has two user sides.

### Agency side

The agency needs speed, visibility, and control. The dashboard should show who is blocked on the agency, who is blocked on the client, which submissions need review, and which projects are delayed. The UX audit proposes agency setup in under three minutes and a professional experience suitable for high-value client work.

### Client side

The client needs zero-login access, clear explanations, accepted-format examples, mobile-friendly uploads, save-and-return behavior, visible progress, and recovery when uploads or links fail.

### Completion funnel

1. Invitation sent.
2. Link opened.
3. Onboarding started.
4. First item submitted.
5. 50% progress reached.
6. All items submitted.
7. Agency approval completed.
8. Project ready for production kickoff.

UX friction includes missing invitation email, missing rejection email, no agency upload trigger, expired links without recovery, random revenue data, dummy meetings, stale progress, missing client list, plan-limit discrepancies, and weak first-run guidance.

## Pricing and Monetization

The current product and monetization documents describe:

| Plan | Price | Clients | Active projects | Main features |
|---|---:|---:|---:|---|
| Free | $0 | 5 | 10 | Limited team access |
| Pro | $29/month, launch $19 | 25 | 60 | Reminders, analytics, priority support, invoices, lead follow-up |
| Ultra | $79/month, launch $49 | Unlimited | Unlimited | White label and custom branding |
| Custom | Custom | Unlimited | Unlimited | Custom requirements |

Research documents also propose Starter, Pro, Agency, and Scale packaging with different limits, seats, storage, custom domains, sync, webhooks, and support. Those values are proposals and must not be treated as current code configuration until resolved.

Proposed upgrade drivers are active projects, team seats, custom domains, white labeling, reminders, templates, and cloud storage sync. The documents recommend avoiding charges for inactive contacts, punitive per-seat pricing for small agencies, and paywalls on common file formats.

## Market and Competitive Research

### Recommended positioning

> ClientDock is the automated client intake and onboarding portal for digital agencies.

The strongest category overlap is client onboarding, client portals, asset collection, approval workflows, and project kickoff workflows. The recommended category to own is **client intake and onboarding automation**.

### Target market

Primary beachhead:

- Boutique web design agencies.
- Digital marketing and SEO agencies.
- Branding and creative studios.
- Agencies with 2–15 staff and 5–30 active projects.
- US, UK, Australian, and Indian export agencies.

Secondary users include freelancers, solo consultants, and high-ticket service providers. Current poor-fit segments include large enterprises needing SSO/SOC 2 and video agencies handling very large files.

### Competitors and alternatives

Direct specialists: Content Snare, Fileflow/Content Collect, and Ahsuite.

Client portals: Copilot/Assembly, ManyRequests, SuiteDash, and Clinked.

All-in-one suites: HoneyBook, Dubsado, Clientjoy, and Bonsai.

DIY alternatives: Google Drive or Dropbox plus email, Google Forms/Typeform/Tally, Notion/Airtable, WhatsApp/Slack/WeTransfer, and custom WordPress/Webflow portals.

Documented differentiation: zero-login access, item-level review and rejection, clear formats, dynamic chasing, modern agency UX, reusable templates, and future cloud handoff.

## US Market Notes

The US research describes a large agency market dominated by small and boutique firms. The target firm has roughly 2–20 employees, 8–35 active projects, project values of approximately $3,000–$25,000, and a stack such as Slack, Google Workspace, ClickUp/Asana, Loom, Stripe, and Figma.

US buyers are expected to want Stripe, USD billing, tax invoices, Apple Pay, possible ACH, custom domains, cloud sync, Zapier/Make, Slack webhooks, and secure credential handling. The proposed GTM includes Product Hunt, selective AppSumo, SEO comparison pages, free templates, agency communities, case studies, cold outreach, and product-led referrals.

## India Market Notes

The India research separates export agencies from domestic agencies. Export agencies serve Western clients and have higher SaaS budgets. Domestic agencies are more price-sensitive and commonly use WhatsApp, Google Drive, and Zoho.

Regional requirements identified:

- WhatsApp is a primary business channel.
- Razorpay supports UPI, domestic cards, NetBanking, GST invoices, and domestic billing.
- RBI recurring-payment rules can make monthly automated collection unreliable.
- Annual upfront billing may be preferred.
- GSTIN and input-tax-credit invoices matter for B2B buyers.
- INR pricing is proposed.
- One global product with regional billing and notifications is preferred over separate products.
- Start with a 1-click WhatsApp share button before a full WhatsApp Business API integration.

## Research Status and Open Questions

Research work already completed or substantially covered:

- ClientDock baseline and product maturity.
- Market category and positioning.
- Direct and adjacent competitors.
- US market and beachhead ICP.
- Product opportunities and negative scope.
- Initial pricing and ROI model.
- India market assumptions.
- Strategic product direction.

Research still identified as necessary:

- Refresh competitor pricing, review counts, funding, and scale with dated sources.
- Mine customer reviews and agency communities.
- Interview 10–15 target agencies.
- Test willingness to pay by agency size.
- Quantify reminder and kickoff-delay ROI.
- Validate Stripe, compliance, and US checkout expectations.
- Validate Indian WhatsApp, UPI, GST, and annual-billing expectations.
- Validate Google Drive and Dropbox demand.
- Design encrypted credential collection before storing secrets.
- Test the client funnel on mobile.
- Define GDPR/CCPA versus later SOC 2 requirements.
- Establish product metrics before launch.

Open hypotheses include whether agencies will pay for intake alone, whether zero-login links feel secure enough, whether storage sync is a major adoption requirement, and whether Indian agencies require WhatsApp to complete onboarding.

## Strategic Direction

The strategy documents group future work into four broad areas:

1. **Activation and integrity:** invitation/rejection emails, templates, truthful data, empty states, and reliable progress.
2. **Commercial and global expansion:** Stripe plus Razorpay, encrypted credentials, and team roles.
3. **Ecosystem and retention:** Google Drive/Dropbox sync, webhooks, WhatsApp sharing, and stronger reminders.
4. **Premium differentiation:** custom domains, white labeling, reviews, client approvals, and advanced analytics.

The strategic decision categories are:

- Build now: invitation emails, rejection alerts, templates, Stripe, encrypted credentials, WhatsApp sharing, and webhooks.
- Improve existing: fake metrics, reminders, and team management.
- Integrate: cloud storage and custom domains.
- Experiment: full WhatsApp API reminders and review collection.
- Wait: AI checklist generation.
- Do not build: full project management, Slack-like chat, and full accounting.

## Conflicting Information to Verify

The documents were created at different stages and contain inconsistent values. These conflicts should be resolved by checking the code:

- Current configuration is described as Free 5 clients and 10 active projects, while some research tables propose other limits.
- The current codebase audit identifies SQLite, Drizzle, and Better Auth, while one baseline passage mentions PostgreSQL and another historical passage mentions Supabase Auth.
- Current plan configuration is $0/$29/$79/Custom, while some research proposes $29/$79/$149 packaging.
- Roadmap phase names and time windows differ across documents.
- Meetings are described as both partially complete and frontend-only; the common fact is that complete server behavior needs verification.
- Completion percentages differ because the audits measure different dimensions.
- Competitor prices, market sizes, review counts, and customer claims should be refreshed from current sources.

## Documentation Rule

Use this README for orientation and the raw project summary. Use the linked source document for detailed evidence, implementation references, and audit findings. Use [ROADMAP.md](./ROADMAP.md) for execution order and task status. Update source documents when new facts are verified rather than silently changing historical findings.
