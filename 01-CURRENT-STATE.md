# 01 — Current State of ClientDock

> Evidence-based assessment of what actually exists and works as of September 2026 audit.

---

## Authentication System

**Status: MOSTLY COMPLETE (90%)**

### What Works
- Email/password signup and login via Better Auth
- Automatic agency creation on signup (`signUpAction` → `createAgency`)
- Session management (7-day sessions, 5-minute cookie cache)
- Optional Google OAuth and GitHub OAuth (env-var controlled)
- Password reset email flow (templates exist, Resend integration)
- Account linking between OAuth providers
- User deletion support
- Session-based server-side auth via `getCurrentSession()`

### What Is Incomplete or Problematic
- **Email verification is DISABLED in production config** (`requireEmailVerification: false` in `src/lib/auth.ts:44`). This means any email can be used without verification. This is a security and trust issue for a production SaaS.
- **No rate limiting on login attempts at the action level** — `withRateLimit` exists in `src/lib/rate-limit.ts` but is NOT applied in `signInAction` or `signUpAction`. The rate limit module requires Redis, which may not be available.
- **Redirect after login** — Login page redirects to `/dashboard` (needs verification this works correctly for OAuth flows too).

---

## Agency Dashboard — Main Page

**Status: FUNCTIONAL BUT NEEDS IMPROVEMENT (75%)**

### What Works
- Real stats cards (Total Clients, In Progress, Completed, Delayed) — queries DB
- Project list with status filters and progress bars
- Recent Activity feed from `activity_log` table
- Empty state for new users

### What Is Broken/Incomplete
- **Revenue chart uses `Math.random()`** — `getRevenueData()` in `src/app/dashboard/page.tsx:18-25` generates random data. No real revenue data is displayed. This is labeled as "mock" in comments but visible to real users.
- **Meeting Scheduler shows hardcoded demo data** — `getMeetings()` in `src/app/dashboard/page.tsx:28-48` returns hardcoded meetings. The meeting database table is never queried on the main dashboard.
- **Stats "Completed This Month" counts ALL completed projects** — The query in `src/lib/data.ts:83-89` has no date filter for "this month." It shows total completed, not monthly.

---

## Project Detail Page

**Status: MOSTLY COMPLETE (80%)**

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

### What Is Incomplete
- **Progress field inconsistency** — `project.progress` stores an integer (0–100) but `calculateProjectProgress()` in `src/lib/data.ts:7-28` computes it from checklist items. These two values can diverge. When a checklist item is approved, the `progress` column is NOT automatically updated.
- **Timeline events** — UI exists to display timeline, but the component for _editing_ timeline events (adding/removing milestones) needs verification.
- **"Download All" button** — `DownloadAllButton.tsx` exists as a 763-byte file — likely a stub. Verify actual download functionality.
- **ChatCard** — `ChatCard.tsx` exists as 1,531 bytes, appears to be a placeholder messaging UI.
- **TodoListCard** — `TodoListCard.tsx` exists; verify if wired to actual data or demo only.

---

## Client Portal

**Status: MOSTLY COMPLETE (85%)**

### What Works
- Token-based passwordless access via `/portal/[token]`
- Token verification with SHA-256 hashing and expiry checking (`src/lib/verify-portal-token.ts`)
- Client can view all checklist items and their status
- Client can upload files to checklist items via UploadThing
- Rejection reasons are shown to the client
- Activity is logged when client uploads

### What Is Incomplete/Problematic
- **UploadThing middleware has NO authentication** — The `clientSubmission` uploader in `src/lib/uploadthing.ts:21-23` returns `{ uploadedBy: "client" }` but does NOT verify the portal token. Any person with the upload URL could upload files without a valid portal session.
- **Portal tokens expire in 30 days** — For long-running projects (common in agencies), the portal link sent to the client will expire. No mechanism exists to automatically renew tokens for active projects.
- **No file type validation beyond extension** — UploadThing accepts defined file types, but the `acceptedFormats` field on checklist items is stored as a comma-separated string (e.g., `"png,svg,pdf"`) and is NOT enforced server-side.

---

## Clients Management

**Status: FUNCTIONAL BUT NEEDS IMPROVEMENT (70%)**

### What Works
- Client listing page exists
- Client detail page (`ClientDetailClient.tsx`)
- Invite client flow (generates portal token hash, saves to DB)
- Client limit enforcement based on plan

### What Is Incomplete
- **No dedicated clients list page** — `/dashboard/clients` has a `[id]` subdirectory but no `page.tsx` at the clients root. Navigation likely goes directly to client detail.
- **Client edit** — No evidence of a client name/email update flow.
- **Client deletion** — No UI evidence for deleting a client from dashboard.

---

## Leads CRM

**Status: FUNCTIONAL (70%)**

### What Works
- Full Kanban-style pipeline view (`LeadsDashboard` component)
- Create lead with full metadata (contact name, company, email, phone, location, source, estimated value, stage, follow-up date, notes)
- Update lead stage with drag-and-drop (`@hello-pangea/dnd`)
- Soft delete leads (no data loss)
- Convert lead to client (creates client record, marks lead as "won")
- Pipeline metrics (open leads, pipeline value, win rate, won this month)
- Lead activity logging

### What Is Problematic
- **`softDeleteLeadsAction` has no ownership verification** — Lines 145-151 in `src/app/actions/leads.ts` update `isDeleted` for any `leadId` in the array without checking if `lead.agencyId === session.agency.id`. A malicious user could delete another agency's leads.
- **`getAgencyLeads` filters by `isDeleted = false`** — Soft-deleted leads cannot be recovered from the UI (no trash/restore view).
- **No lead import functionality** — Agencies likely want to import leads from CSV or CRM.

---

## Analytics

**Status: PARTIALLY COMPLETE (60%)**

### What Works
- Total projects, total clients, completion rate, average progress metrics
- Projects by status bar chart
- Insights section
- Plan-gating for `fullAnalytics` feature

### What Is Incomplete
- **All metrics are point-in-time** — No time-series data (no "trend" data). The trend arrows shown are computed from the current value alone (e.g., `trend={data.completionRate >= 50 ? "up" : "down"}`), not from a comparison to a previous period.
- **No revenue analytics** — Revenue chart on dashboard is mocked; analytics page has no revenue data.
- **No lead analytics** — Leads/CRM data not included in analytics.
- **No file analytics** — No storage usage, file upload frequency, etc.

---

## Meetings

**Status: FRONTEND ONLY / PLACEHOLDER (40%)**

### What Exists
- Database schema (`meeting` table: id, agencyId, projectId, title, description, meetingDate, durationMinutes, meetLink, reminderMinutes, status)
- `MeetingsCalendar.tsx` (18KB) — Calendar UI with scheduling
- Dashboard sidebar shows hardcoded demo meetings

### What Is Broken
- **Main dashboard `getMeetings()` never queries the database** — Returns hardcoded demo data
- **No server actions for meetings** — No `createMeeting`, `updateMeeting`, `deleteMeeting` actions exist
- **No meeting page wiring** — The `/dashboard/meetings/page.tsx` (1,414 bytes) exists but unclear if it queries DB or renders static UI
- **No meeting reminder** — The `reminderMinutes` field exists in schema but is never used

---

## Notifications

**Status: MOSTLY COMPLETE (70%)**

### What Works
- Notification bell in header with unread count badge
- In-app notification list (`NotificationsPageClient.tsx`)
- Notification creation when checklist items are approved/rejected
- Mark as read / mark all read
- Notification types: `file_uploaded`, `deadline_approaching`, `status_changed`, `reminder_sent`

### What Is Incomplete
- **`file_uploaded` notifications are NOT created** — The UploadThing `onUploadComplete` callback only logs to console. No notification is created when a client submits a file.
- **`checkDeadlineNotifications()` is never called** — This function exists in `src/app/actions/notifications.ts:112-163` but there is no cron job or trigger that calls it.
- **No email delivery for agency notifications** — Notifications are in-app only. When a client uploads a file, the agency does not get an email.

---

## Billing (Razorpay)

**Status: PARTIALLY COMPLETE (55%)**

### What Works
- Plan configuration in `plan-config.ts` (Free, Pro, Ultra, Custom with pricing)
- Plan enforcement via `checkProjectLimit`, `checkClientLimit`, `requireFeature`
- Billing page with current plan display, usage meters, plan comparison
- Razorpay subscription creation API route (`/api/razorpay/create-subscription`)
- Razorpay webhook handler (`/api/razorpay/webhook`) with idempotency
- Subscription status tracking in DB
- Payment history recording
- Subscription cancellation action

### What Is Broken/Incomplete
- **Webhook emails use hardcoded `billing@example.com`** — Lines 115 and 122 in `src/app/api/razorpay/webhook/route.ts` use hardcoded fallback email addresses instead of the actual agency email.
- **Subscription cancellation does NOT downgrade plan immediately** — The `subscription.cancelled` webhook sets subscription status to "cancelled" but does NOT set `agency.plan = "free"`. The plan downgrade logic is commented as "scheduled or handled here."
- **Payment status values are inconsistent** — The schema uses `["created","authorized","captured","failed","refunded"]` but billing page checks for `payment.status === 'success' || payment.status === 'paid'` (neither of which are valid enum values).
- **`dev-bypass` route exists** — `/api/razorpay/dev-bypass` exists which could be a security issue if deployed to production.
- **No Stripe integration** — Only Razorpay, which is India-centric and has limited US support. For US market focus, this is a significant gap.
- **`checkProjectLimit` has a query bug** — The `where` clause at `src/lib/check-plan.ts:59-64` is incomplete — it counts ALL projects but the comment says "all statuses except complete are active." The filter `sql\`project.status != 'complete'\`` is missing.

---

## Settings

**Status: PARTIALLY COMPLETE (65%)**

### What Works
- Notification preferences (email notifications, auto reminders, weekly digest toggles) — actually saved to DB
- Account page exists

### What Is Incomplete
- **Profile editing** — No evidence of name/email update forms connected to server actions in the settings page
- **Agency name/logo editing** — No agency profile update functionality
- **Password change** — Better Auth supports this but no UI exists
- **Danger zone** — Delete account button may exist but connection to Better Auth `deleteUser` needs verification

---

## Landing Page

**Status: MOSTLY COMPLETE (80%)**

### What Works
- Hero section, Problem section, Features section, Pricing section, FAQ section, Testimonials section, Users section, CTA section, Footer
- Pricing section reads from `plan-config.ts` (single source of truth)
- Blog (7 articles with content)
- Legal pages (Privacy, Terms, GDPR, Cookies)

### What Is Incomplete/Problematic
- **Testimonials are static/demo** — No real customer testimonials yet
- **"Users" section** — Shows logos of companies; all are demo placeholders
- **Pricing section shows different values than plan-config** — The billing page comparison table shows "2 projects" for Free but `PLANS.free.limits.maxActiveProjects = 10`. This inconsistency must be fixed.
- **No actual blog CMS** — Blog content is hardcoded in JSX. No authoring workflow.

---

## Infrastructure State

**Status: GOOD FOUNDATION (70%)**

### What Exists
- Multi-stage Docker build (deps → builder → runner)
- Docker Compose with bind-mount volume for SQLite persistence
- Litestream installed in Docker image (SQLite replication to S3)
- GitHub Actions CI pipeline
- Drizzle migrations (auto-applied on server start via `instrumentation.ts`)
- `.env.example` with all required variables documented

### What Is Missing
- **No `litestream.yml` config in the Docker image** — The Dockerfile CMD runs `litestream replicate --config /etc/litestream.yml` but no `litestream.yml` is copied into the image. This means litestream will fail silently at startup.
- **No staging environment** — Only local dev + production Docker config
- **No monitoring/error tracking** — No Sentry, Datadog, or equivalent
- **Redis is optional** — Rate limiting silently fails without Redis; this should be handled more gracefully
- **`UPLOADTHING_SECRET` env var** — Not in `.env.example`; required for file uploads to work
