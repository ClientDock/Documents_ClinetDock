# 01 — Current State of ClientDock

> **Last Updated:** Post-Migration PR (September 2026)
> **Note:** This document reflects the actual state of the codebase after the September 2026 tech stack migration (SQLite→Postgres, Razorpay→Dodo, Resend→Brevo, UploadThing→R2). All old provider references have been removed.

---

## Authentication System

**Status: 95% COMPLETE**

### What Works
- Email/password signup and login via Better Auth
- Automatic agency creation on signup (`signUpAction` → `createAgency`) — missing `await` bug **fixed**
- Session management (7-day sessions, 5-minute cookie cache)
- Optional Google OAuth and GitHub OAuth (env-var controlled)
- Password reset email flow (Brevo integration)
- Email verification templates exist; enforced in production via `NODE_ENV === "production"`
- Rate limiting on signIn/signUp via Redis (`withRateLimit`)
- Account linking between OAuth providers

### Known Issues
- **Dev bypass incompatible with authorization** — `BYPASS_DASHBOARD_AUTH=true` returns a fake agency ID that won't match real DB records, causing all project/client CRUD to fail with "Forbidden" in dev bypass mode.

---

## Agency Dashboard — Main Page

**Status: 85% COMPLETE**

### What Works
- Real stats cards (Total Clients, In Progress, Completed, Delayed) — queries PostgreSQL
- Project list with status filters and progress bars
- Recent Activity feed from `activity_log` table
- Empty state for new users
- Meetings widget exists in sidebar

### What Is Broken/Incomplete
- **Meeting widget shows hardcoded demo data** — `getMeetingsForAgency` query exists in `src/lib/db.ts` but the main dashboard may still use demo meetings. Verify.

---

## Project Detail Page

**Status: 95% COMPLETE**

### What Works
- Full project information display (name, type, description, priority, industry, budget, due date)
- Checklist manager — create, delete items; approve, reject with reason
- File submissions list (per checklist item)
- Internal notes (auto-saving scratchpad, agency-only)
- Activity log display
- Timeline events (milestone list with completion state)
- Reminder settings (enable/disable auto-reminder, set frequency)
- Portal link card with token display and rotation
- Edit project modal with all fields
- `syncProjectProgress` called after every approval/rejection

### What Is Incomplete
- **Rejection email has no portal link** — `sendRejectionEmailAction` is called without a `portalLink` argument (the plaintext token is not stored, by design). The email tells the client their item was rejected but doesn't link back to the portal.

---

## Client Portal

**Status: 80% COMPLETE**

### What Works
- Token-based passwordless access via `/portal/[token]`
- Token verification with SHA-256 hashing and expiry checking (`src/lib/verify-portal-token.ts`)
- Client can view all checklist items and their status
- Rejection reasons shown to the client
- Activity is logged when client uploads

### What Is Incomplete/Needs Verification
- **R2 upload flow needs end-to-end test** — `src/lib/r2.ts` provides `getUploadUrl` (presigned PUT) and `generateKey`. The portal UI must call the presign API, upload directly to R2, then save the returned URL. This flow needs to be verified working.
- **Portal token expiry** — 30-day expiry. For long projects, the link expires. No auto-renewal mechanism exists.
- **`file_uploaded` notification not created** — When a client submits, the agency gets no in-app notification or email.

---

## Clients Management

**Status: 85% COMPLETE**

### What Works
- Client listing
- Client detail page
- Invite client flow (generates portal token hash, saves to PostgreSQL)
- Client limit enforcement based on plan
- Portal token rotation

### What Is Incomplete
- **Client edit** — No name/email update flow confirmed
- **Client deletion** — No UI evidence for deleting a client

---

## Leads CRM

**Status: 90% COMPLETE**

### What Works
- Full Kanban-style pipeline view
- Create lead with full metadata
- Update lead stage with drag-and-drop
- Soft delete leads (with ownership verification — **fixed**)
- Convert lead to client (creates client record, marks lead as "won")
- Pipeline metrics (open leads, pipeline value, win rate, won this month)
- Lead activity logging

### What Is Incomplete
- **No invite email sent on lead-to-client conversion** — The portal link is generated but no email is sent
- **No lead import** — CSV import not implemented

---

## Analytics

**Status: 45% COMPLETE**

### What Works
- Total projects, total clients, completion rate, average progress metrics
- Projects by status bar chart
- Plan-gating for `fullAnalytics` feature

### What Is Incomplete
- **No time-series data** — All metrics are point-in-time. Trend arrows are computed from current value, not from historical comparison.
- **No revenue analytics** — Dashboard revenue chart was removed with the old Razorpay stack and not replaced.
- **No lead analytics** — Leads/CRM data not in analytics.

---

## Meetings

**Status: 30% COMPLETE**

### What Exists
- Database schema (`meeting` table: full schema with agencyId, projectId, title, meetingDate, etc.)
- `MeetingsCalendar.tsx` — Calendar UI with scheduling
- `getMeetingsForAgency` DB query in `src/lib/db.ts`

### What Is Missing
- **No server actions** — No `createMeeting`, `updateMeeting`, `deleteMeeting` actions exist
- **No meeting reminder emails** — `reminderMinutes` field exists in schema but is never used

---

## Notifications

**Status: 60% COMPLETE**

### What Works
- Notification bell with unread count badge
- In-app notification list
- Notification creation on checklist approve/reject
- Mark as read / mark all read
- `checkDeadlineNotifications()` is called by the cron endpoint

### What Is Incomplete
- **`file_uploaded` notifications not created** — When a client uploads, no notification is inserted.
- **No email delivery for agency notifications** — Notifications are in-app only. Agency doesn't get an email when a client uploads.

---

## Billing (Dodo Payments)

**Status: 85% COMPLETE**

### What Works
- Plan configuration in `plan-config.ts` (Free, Pro, Ultra, Custom)
- Plan enforcement via `checkProjectLimit`, `checkClientLimit`, `requireFeature`
- Billing page with current plan display and usage meters
- Dodo Payments checkout creation (`POST /api/billing/checkout`)
- Webhook handler with HMAC-SHA256 verification (crash bug **fixed**)
- Idempotency via `webhookEvent` table
- Plan upgrade on `subscription.active` event (premature upgrade bug **fixed**)
- Plan downgrade to `free` on `subscription.cancelled`/`expired`/`paused`
- Subscription cancellation endpoint

### What Is Incomplete
- **No Stripe** — Dodo Payments is the only billing option; limited US payment methods

---

## Settings

**Status: 80% COMPLETE**

### What Works
- Notification preferences (email notifications, auto reminders, weekly digest) — saved to DB
- Account page exists

### What Is Incomplete
- **Profile editing** — No name/email update form connected to server actions
- **Agency name/logo editing** — Not implemented
- **Password change UI** — Backend supports it; no UI

---

## Infrastructure

**Status: 85% COMPLETE**

### What Exists
- Multi-stage Docker build (deps → builder → runner)
- Docker Compose: PostgreSQL 16 + Redis 7 + Next.js app
- PostgreSQL data volume (`postgres_data`)
- Redis data volume with AOF persistence
- GitHub Actions CI pipeline
- Drizzle migrations in `drizzle-pg/`
- `.env.example` with all required variables (**CRON_SECRET added** in this PR)

### What Is Missing/Needs Attention
- **`BETTER_AUTH_URL` in Compose is localhost** — Must be updated to the real domain for production VPS deployment
- **No monitoring/error tracking** — No Sentry, Datadog, or equivalent
- **No staging environment** — Only local dev + production Docker config
- **Redis is required for rate limiting** — `rate-limit.ts` fails closed (blocks requests) if Redis is down, which is correct but may surprise operators

---

## Landing Page

**Status: 90% COMPLETE**

### What Works
- Hero, Problem, Features, Pricing, FAQ, Testimonials, CTA, Footer — all functional
- Pricing reads from `plan-config.ts`
- Blog (7 articles)
- Legal pages (Privacy, Terms, GDPR, Cookies)

### What Is Incomplete
- Testimonials are static/demo content
- Company logos section is demo placeholders
