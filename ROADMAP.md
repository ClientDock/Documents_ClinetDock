# ClientDock — Master Roadmap

> This roadmap is generated from the full codebase audit. Every item is evidence-based. Items are ordered by priority within each phase. Estimated effort is in developer-hours for a single senior full-stack developer.

---

## Legend

| Symbol | Meaning |
|---|---|
| 🔴 P0 | Production blocker — must fix before any real users |
| 🟠 P1 | Critical — fix before public launch |
| 🟡 P2 | Important — fix within first 30 days of launch |
| 🟢 P3 | Enhancement — fix within 90 days |
| 🔵 P4 | Future feature — roadmap item |

---

## Phase 0 — Security & Correctness Fixes (BEFORE LAUNCH)

These items prevent active harm to users or the business. Ship nothing until these are done.

### 🔴 P0-1 — Block Dev Routes in Production
**File:** `src/app/api/razorpay/dev-bypass/route.ts`, `src/app/api/test-db/route.ts`, `src/app/dev/page.tsx`  
**Effort:** 1 hour  
**Action:** Add `NODE_ENV === 'production'` guard returning 404 to each dev route.  
**Verify:** Deploy to staging, confirm routes return 404.

---

### 🔴 P0-2 — Fix UploadThing Authorization
**File:** `src/lib/uploadthing.ts`  
**Effort:** 3 hours  
**Action:** Pass portal token in upload metadata; verify in middleware before allowing upload.  
**Details in:** [bugs/BUGS.md BUG-001](./bugs/BUGS.md)  
**Verify:** Upload without token → 401. Upload with expired token → 401. Upload with valid token → success.

---

### 🔴 P0-3 — Fix Lead Delete Ownership Check
**File:** `src/app/actions/leads.ts:145-151`  
**Effort:** 30 minutes  
**Action:** Add `eq(leads.agencyId, session.agency.id)` to softDeleteLeadsAction WHERE clause.  
**Details in:** [bugs/BUGS.md BUG-002](./bugs/BUGS.md)

---

### 🔴 P0-4 — Fix Billing Webhook Email Addresses
**File:** `src/app/api/razorpay/webhook/route.ts:115, 122`  
**Effort:** 1 hour  
**Action:** Look up agency user's actual email from DB and use it in webhook email calls.  
**Details in:** [bugs/BUGS.md BUG-003](./bugs/BUGS.md)

---

### 🔴 P0-5 — Add UPLOADTHING vars to .env.example
**File:** `.env.example`  
**Effort:** 15 minutes  
**Action:** Add `UPLOADTHING_SECRET` and `UPLOADTHING_APP_ID` with documentation comments.

---

## Phase 1 — Critical Business Fixes (Launch Week)

These don't block launch but will immediately damage trust or revenue if not fixed.

### 🟠 P1-1 — Fix Subscription Cancellation Plan Downgrade
**File:** `src/app/api/razorpay/webhook/route.ts:99-118`  
**Effort:** 1 hour  
**Action:** Add `agency.plan = 'free'` update in `subscription.cancelled` webhook handler.  
**Details in:** [bugs/BUGS.md BUG-004](./bugs/BUGS.md)

---

### 🟠 P1-2 — Fix Payment Status Enum in Billing UI
**File:** `src/app/dashboard/billing/page.tsx:167`  
**Effort:** 30 minutes  
**Action:** Replace `status === 'success' || 'paid'` with `status === 'captured'`.  
**Details in:** [bugs/BUGS.md BUG-005](./bugs/BUGS.md)

---

### 🟠 P1-3 — Fix checkProjectLimit Bug
**File:** `src/lib/check-plan.ts:55-65`  
**Effort:** 30 minutes  
**Action:** Add `sql\`${project.status} != 'complete'\`` to the project count WHERE clause.  
**Details in:** [bugs/BUGS.md BUG-006](./bugs/BUGS.md)

---

### 🟠 P1-4 — Remove Random Revenue Chart / Show Empty State
**File:** `src/app/dashboard/page.tsx:18-25`  
**Effort:** 1 hour  
**Action:** Replace random data with empty state or real data from payment table.  
**Details in:** [bugs/BUGS.md BUG-007](./bugs/BUGS.md)

---

### 🟠 P1-5 — Fix Meeting Widget (Remove Hardcoded Demo Data)
**File:** `src/app/dashboard/page.tsx:28-48`  
**Effort:** 1 hour  
**Action:** Show "No upcoming meetings" empty state. Remove fake hardcoded meetings.  
**Details in:** [bugs/BUGS.md BUG-008](./bugs/BUGS.md)

---

### 🟠 P1-6 — Enable Email Verification for Production
**File:** `src/lib/auth.ts:44`  
**Effort:** 30 minutes + email template verification  
**Action:** Set `requireEmailVerification: true`. Verify Resend API is configured.  
**Details in:** [security/SECURITY-AUDIT.md SEC-003](./security/SECURITY-AUDIT.md)

---

### 🟠 P1-7 — Auto-Update Project Progress After Checklist Status Change
**Files:** `src/app/actions/project-details.ts` (approveChecklistItemAction, rejectChecklistItemAction)  
**Effort:** 1 hour  
**Action:** After any checklist item status change, compute and store new progress:
```typescript
const newProgress = calculateProjectProgress(item.projectId);
await updateProject(item.projectId, { progress: newProgress });
```

---

### 🟠 P1-8 — Add File_Uploaded Notification
**File:** `src/lib/uploadthing.ts`, `src/app/actions/notifications.ts`  
**Effort:** 2 hours  
**Action:** In `onUploadComplete`, create `file_uploaded` notification for the agency.  
**Requires:** Pass `agencyId` through UploadThing metadata.  
**Details in:** [bugs/BUGS.md BUG-009](./bugs/BUGS.md)

---

### 🟠 P1-9 — Send Client Invitation Email
**File:** `src/app/actions/projects.ts:inviteClientAction`  
**Effort:** 2 hours  
**Action:** After creating the client record, send an invitation email with the portal link.  
**Details in:** [ux/UX-AUDIT.md UX-001](./ux/UX-AUDIT.md)

---

### 🟠 P1-10 — Fix Pricing Display on Billing Page
**File:** `src/app/dashboard/billing/page.tsx`  
**Effort:** 1 hour  
**Action:** Replace hardcoded "2 projects" / "10 projects" with values from `PLANS[planName].limits.*`.  
**Details in:** [bugs/BUGS.md BUG-013](./bugs/BUGS.md)

---

### 🟠 P1-11 — Add Rate Limiting to Auth Endpoints
**File:** `src/app/actions/auth.ts`  
**Effort:** 2 hours  
**Action:** Apply `checkRateLimit` to `signInAction` and `signUpAction`. Handle Redis unavailability gracefully in development.  
**Details in:** [security/SECURITY-AUDIT.md SEC-004](./security/SECURITY-AUDIT.md)

---

### 🟠 P1-12 — Set Up Cron Scheduler + Wire Deadline Check
**Files:** `src/app/api/cron/reminders/route.ts`  
**Effort:** 2 hours  
**Action:**  
1. Add `checkDeadlineNotifications()` call to cron endpoint
2. Set up external cron scheduler (Vercel Cron or GitHub Actions)
3. Add health ping to monitoring service

---

### 🟠 P1-13 — Create Litestream Config File
**File:** New `litestream.yml`  
**Effort:** 2 hours  
**Action:** Create `litestream.yml` with S3/R2 configuration. Add COPY to Dockerfile.  
**Details in:** [infrastructure/CURRENT-INFRASTRUCTURE.md](./infrastructure/CURRENT-INFRASTRUCTURE.md)

---

## Phase 2 — Product Quality (First 30 Days Post-Launch)

### 🟡 P2-1 — Implement Meeting CRUD Server Actions
**Files:** New `src/app/actions/meetings.ts`  
**Effort:** 4 hours  
**Action:** Create `createMeetingAction`, `updateMeetingAction`, `deleteMeetingAction`. Wire `MeetingsCalendar.tsx` to real DB queries.

---

### 🟡 P2-2 — Add Client Rejection Email
**File:** `src/app/actions/project-details.ts:rejectChecklistItemAction`  
**Effort:** 2 hours  
**Action:** After rejecting a checklist item, send email to client with rejection reason and portal link.

---

### 🟡 P2-3 — Fix Portal Token Expiry for Long Projects
**Files:** `src/db/schema.ts`, `src/lib/verify-portal-token.ts`  
**Effort:** 2 hours  
**Action:** Increase token expiry to 1 year for active projects. Or add auto-renewal logic for active projects.

---

### 🟡 P2-4 — Add Clients List Page
**File:** New `src/app/dashboard/clients/page.tsx`  
**Effort:** 3 hours  
**Action:** Create a clients list page showing all clients with their project count and last activity.

---

### 🟡 P2-5 — Standardize Date Types (Leads Table)
**File:** `src/db/schema.ts`, new migration  
**Effort:** 3 hours  
**Action:** Migrate `leads.created_at`, `leads.updated_at`, `leads.follow_up_date` from INTEGER to TEXT ISO strings.  
**Details in:** [architecture/ARCHITECTURE-AUDIT.md ARCH-001](./architecture/ARCHITECTURE-AUDIT.md)

---

### 🟡 P2-6 — Add `lastReminderSentAt` to Project Table
**File:** `src/db/schema.ts`, new migration  
**Effort:** 2 hours  
**Action:** Add `last_reminder_sent_at TEXT` column. Update cron to use this instead of LIKE query on activity log.  
**Details in:** [architecture/ARCHITECTURE-AUDIT.md ARCH-004](./architecture/ARCHITECTURE-AUDIT.md)

---

### 🟡 P2-7 — Fix Rate Limiter Dev Behavior
**File:** `src/lib/rate-limit.ts`  
**Effort:** 1 hour  
**Action:** Fail-open in development when Redis is unavailable.

---

### 🟡 P2-8 — Add Onboarding Empty State / Guided Setup
**File:** `src/app/dashboard/page.tsx`  
**Effort:** 4 hours  
**Action:** When agency has 0 clients, show onboarding checklist with "Invite your first client" CTA.

---

### 🟡 P2-9 — Add Plan Limit Upgrade CTAs
**Files:** Server actions that return plan limit errors  
**Effort:** 3 hours  
**Action:** When a plan limit error is returned, include an upgrade link. Show upgrade modal instead of generic error.

---

### 🟡 P2-10 — Add "Save Confirmed" Indicator for Internal Notes
**File:** `src/app/dashboard/projects/[id]/components/InternalNotes.tsx`  
**Effort:** 1 hour  
**Action:** Show "Saved ✓" indicator briefly after auto-save fires.

---

### 🟡 P2-11 — Set Up Error Tracking (Sentry)
**Effort:** 2 hours  
**Action:** `npm install @sentry/nextjs`. Configure Sentry DSN. Wire to Next.js error handlers.

---

## Phase 3 — Growth Features (30–90 Days)

### 🟢 P3-1 — Implement Team Members
**Files:** New schema tables, UI, server actions  
**Effort:** 2–3 weeks  
**Action:** Allow agency to invite team members who can access the dashboard. Enforce `maxTeamMembers` limit.

---

### 🟢 P3-2 — Add Stripe Payment Integration
**Files:** New `src/lib/stripe.ts`, new API routes  
**Effort:** 1 week  
**Action:** Add Stripe as payment processor for US/international users.  
**Details in:** [product/MONETIZATION.md](./product/MONETIZATION.md)

---

### 🟢 P3-3 — Project Templates
**Effort:** 1 week  
**Action:** Allow agencies to save checklist configurations as named templates. Apply template when creating a project.

---

### 🟢 P3-4 — Implement Weekly Digest Email
**File:** `src/app/api/cron/` (new digest endpoint)  
**Effort:** 3 hours  
**Action:** Weekly summary of pending items, overdue projects, won leads. Respect `agency.weeklyDigest` preference.

---

### 🟢 P3-5 — Review Collection System
**Files:** Existing `review` schema, new UI and actions  
**Effort:** 1 week  
**Action:** When project is marked complete, auto-email client asking for a review. Display reviews on agency profile.

---

### 🟢 P3-6 — Optimize N+1 Queries in Files Page
**File:** `src/lib/data.ts:getAllProjectsWithFiles`  
**Effort:** 2 hours  
**Action:** Replace nested for loops with JOIN-based query.  
**Details in:** [architecture/ARCHITECTURE-AUDIT.md ARCH-007](./architecture/ARCHITECTURE-AUDIT.md)

---

### 🟢 P3-7 — Real Revenue Analytics
**File:** `src/app/dashboard/analytics/page.tsx`  
**Effort:** 4 hours  
**Action:** Build revenue chart from actual `payment` table data. Add monthly breakdown.

---

### 🟢 P3-8 — Client Profile Editing
**Files:** `src/app/dashboard/clients/[id]/`  
**Effort:** 2 hours  
**Action:** Allow agency to edit client name and email. Re-send portal invitation.

---

### 🟢 P3-9 — Lead Import from CSV
**Files:** `src/app/dashboard/leads/`  
**Effort:** 1 day  
**Action:** Allow agencies to import leads from a CSV file. Auto-detect columns.

---

## Phase 4 — Competitive Moat (90+ Days)

### 🔵 P4-1 — White Labeling
Custom domain, custom logo, custom colors for the client portal. Pro/Ultra plan feature.

### 🔵 P4-2 — Invoice Generation
Auto-generate invoice PDF when project is complete. Link to payment. Store in client record.

### 🔵 P4-3 — Client Approval Workflow
Agency shares deliverable (design, PDF) → client approves or leaves revision comments → complete when approved.

### 🔵 P4-4 — Zapier/Make Integration
Expose webhooks for key events: client_invited, file_uploaded, item_approved, project_complete.

### 🔵 P4-5 — Mobile App (PWA)
Progressive Web App for clients on mobile. Portal is already mobile-friendly; make it installable.

### 🔵 P4-6 — Lead Email Outreach
Send outreach emails to leads from within ClientDock. Track opens/clicks.

---

## Summary Timeline

| Phase | Duration | Key Goal |
|---|---|---|
| Phase 0 | 1–2 days | Block security holes |
| Phase 1 | 1–2 weeks | Fix critical bugs; make product trustworthy |
| Phase 2 | 2–4 weeks | Fill product gaps; ready for public launch |
| Phase 3 | 1–3 months | Growth features; upgrade triggers |
| Phase 4 | 3–12 months | Competitive differentiation |

---

## Effort Summary

| Phase | Total Estimated Effort |
|---|---|
| Phase 0 (P0 blockers) | ~8 hours |
| Phase 1 (P1 critical) | ~20 hours |
| Phase 2 (P2 quality) | ~30 hours |
| Phase 3 (P3 growth) | ~8 weeks |
| Phase 4 (P4 moat) | ongoing |

**To reach "safe to launch" state: ~28 hours of focused work.**  
**To reach "production-grade SaaS": ~2–3 months of focused work.**
