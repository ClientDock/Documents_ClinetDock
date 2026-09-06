# 03 — Feature Inventory

> Complete catalog of every feature in ClientDock with evidence-based status assessment.

---

## How to Read This Document

Each feature is assessed on two dimensions:

| Dimension | Description |
|---|---|
| **Functional Status** | Does it actually work end-to-end? |
| **Experience Status** | Is it good for users even if it works? |

**Functional Status Values:** COMPLETE · MOSTLY COMPLETE · PARTIALLY COMPLETE · FUNCTIONAL BUT WEAK · FRONTEND ONLY · BACKEND ONLY · PLACEHOLDER · BROKEN · NOT IMPLEMENTED

**Experience Status Values:** EXCELLENT · GOOD · ACCEPTABLE · NEEDS IMPROVEMENT · POOR · NOT APPLICABLE

---

## CORE FEATURES

### F01 — Email/Password Authentication

| Property | Detail |
|---|---|
| **Purpose** | Agency owners create accounts and log in |
| **Target User** | Agency owner |
| **Frontend** | `src/app/login/`, `src/app/signup/`, `src/components/auth/` |
| **Backend** | `src/app/actions/auth.ts`, `src/lib/auth.ts` |
| **Database** | `user`, `session`, `account`, `verification` tables |
| **Functional Status** | MOSTLY COMPLETE |
| **Experience Status** | GOOD |
| **Missing** | Email verification disabled; rate limiting on auth not applied |
| **Action** | Enable email verification; add rate limiting to signIn/signUp |

---

### F02 — Google/GitHub OAuth

| Property | Detail |
|---|---|
| **Purpose** | Social login for agency owners |
| **Frontend** | Login page (conditional buttons) |
| **Backend** | `src/lib/auth.ts` social providers config |
| **Functional Status** | MOSTLY COMPLETE (env-var controlled) |
| **Experience Status** | GOOD |
| **Missing** | No UI feedback when OAuth env vars not configured |
| **Action** | Test in production; ensure account linking works |

---

### F03 — Agency Auto-Creation on Signup

| Property | Detail |
|---|---|
| **Purpose** | Every user gets an agency workspace automatically |
| **Backend** | `src/app/actions/auth.ts:signUpAction` → `createAgency()` |
| **Database** | `agency` table |
| **Functional Status** | COMPLETE |
| **Experience Status** | EXCELLENT (zero friction) |
| **Notes** | Rollback on failure (deletes user if agency creation fails) |
| **Action** | None — this is well implemented |

---

### F04 — Agency Dashboard Overview

| Property | Detail |
|---|---|
| **Purpose** | Agency's main workspace overview |
| **Frontend** | `src/app/dashboard/page.tsx`, `src/app/dashboard/components/` |
| **Backend** | `src/lib/data.ts`: getDashboardStats, getDashboardProjects, getRecentActivities |
| **Functional Status** | FUNCTIONAL BUT WEAK |
| **Experience Status** | ACCEPTABLE |
| **Missing** | Revenue chart uses random data; meetings are hardcoded; "Completed This Month" has no month filter |
| **Action** | P1 — Fix meeting data; P1 — Fix revenue data; P2 — Fix "this month" filter |

---

### F05 — Client Invitation

| Property | Detail |
|---|---|
| **Purpose** | Agency invites a client by email, generating their portal |
| **Frontend** | `InviteClientModal.tsx` |
| **Backend** | `src/app/actions/projects.ts:inviteClientAction` |
| **Database** | `client` table (portalTokenHash, portalTokenExpiresAt) |
| **Functional Status** | MOSTLY COMPLETE |
| **Experience Status** | GOOD |
| **Missing** | No email sent to client on invitation (only portal link is shown to agency). No client email notification that they've been invited. |
| **Action** | P1 — Send invitation email to client with portal link |

---

### F06 — Passwordless Client Portal

| Property | Detail |
|---|---|
| **Purpose** | Clients access their project via unique token URL |
| **Frontend** | `src/app/portal/[token]/PortalClient.tsx` |
| **Backend** | `src/lib/verify-portal-token.ts`, portal page.tsx |
| **Database** | `client.portalTokenHash`, `client.portalTokenExpiresAt` |
| **Functional Status** | MOSTLY COMPLETE |
| **Experience Status** | GOOD |
| **Missing** | No auth on UploadThing middleware; token renewal for long-running projects |
| **Action** | P0 — Add portal token to UploadThing middleware; P2 — Token renewal |

---

### F07 — Project Creation

| Property | Detail |
|---|---|
| **Purpose** | Agency creates a project for a client |
| **Frontend** | `CreateProjectModal.tsx` |
| **Backend** | `src/app/actions/projects.ts:createProjectAction` |
| **Database** | `project` table |
| **Functional Status** | COMPLETE |
| **Experience Status** | GOOD |
| **Missing** | Plan limit check has a bug (counts all projects not non-complete) |
| **Action** | P1 — Fix checkProjectLimit bug |

---

### F08 — Checklist Management

| Property | Detail |
|---|---|
| **Purpose** | Agency defines what the client needs to submit |
| **Frontend** | `ChecklistManager.tsx` in project detail |
| **Backend** | `addChecklistItemAction`, `deleteChecklistItemAction` |
| **Database** | `checklist_item` table |
| **Functional Status** | MOSTLY COMPLETE |
| **Experience Status** | GOOD |
| **Missing** | `acceptedFormats` field not enforced server-side; no bulk add |
| **Action** | P2 — Enforce format validation on submission |

---

### F09 — Client File Submission

| Property | Detail |
|---|---|
| **Purpose** | Client uploads files for checklist items via portal |
| **Frontend** | Portal client page with UploadThing component |
| **Backend** | `src/lib/uploadthing.ts:clientSubmission`, `createSubmission` DB call |
| **Database** | `submission` table |
| **Functional Status** | PARTIALLY COMPLETE |
| **Experience Status** | ACCEPTABLE |
| **Missing** | No auth on UploadThing middleware; no notification sent to agency; accepted formats not enforced |
| **Action** | P0 — Fix UploadThing auth; P1 — Send file_uploaded notification |

---

### F10 — Approval/Rejection Workflow

| Property | Detail |
|---|---|
| **Purpose** | Agency approves or rejects client submissions |
| **Frontend** | `ChecklistManager.tsx` approval/rejection UI |
| **Backend** | `approveChecklistItemAction`, `rejectChecklistItemAction` |
| **Database** | `checklist_item` (status, rejectionReason, completed, completedAt) |
| **Functional Status** | MOSTLY COMPLETE |
| **Experience Status** | GOOD |
| **Missing** | Project progress not updated after approval; no notification email to client on rejection |
| **Action** | P1 — Update project.progress after checklist status change; P2 — Email client on rejection |

---

### F11 — Internal Notes (Agency Scratchpad)

| Property | Detail |
|---|---|
| **Purpose** | Private agency notes per project, hidden from client |
| **Frontend** | `InternalNotes.tsx` |
| **Backend** | `updateInternalNotesAction` |
| **Database** | `project.internalNotes` |
| **Functional Status** | COMPLETE |
| **Experience Status** | GOOD |
| **Action** | None |

---

### F12 — Automated Email Reminders

| Property | Detail |
|---|---|
| **Purpose** | Auto-chase clients who have pending checklist items |
| **Frontend** | `ReminderSettings.tsx` per project |
| **Backend** | `updateReminderSettingsAction`, `/api/cron/reminders/route.ts` |
| **Database** | `project.autoReminder`, `project.reminderFrequencyDays` |
| **Functional Status** | MOSTLY COMPLETE |
| **Experience Status** | GOOD |
| **Missing** | No external cron scheduler connected; relies on `CRON_SECRET` bearer auth |
| **Action** | P2 — Document cron scheduling setup; P2 — Add last-reminder timestamp field to project |

---

### F13 — Manual Reminder Send

| Property | Detail |
|---|---|
| **Purpose** | Agency manually triggers a reminder email to client |
| **Frontend** | `ReminderSettings.tsx` send button |
| **Backend** | `sendManualReminderAction` |
| **Functional Status** | MOSTLY COMPLETE |
| **Experience Status** | GOOD |
| **Missing** | Rotates portal token on every manual reminder (this is intentional but not communicated to user) |
| **Action** | P3 — Add UI note explaining token rotation |

---

### F14 — Timeline Events (Milestones)

| Property | Detail |
|---|---|
| **Purpose** | Agency marks project milestones for client visibility |
| **Frontend** | `WorkHistoryTimeline.tsx` |
| **Backend** | Timeline event CRUD actions (need to verify implementation) |
| **Database** | `timeline_events` table |
| **Functional Status** | PARTIALLY COMPLETE |
| **Experience Status** | ACCEPTABLE |
| **Missing** | Need to verify add/edit/delete milestone actions exist and work |
| **Action** | P2 — Verify timeline CRUD implementation |

---

### F15 — Activity Log

| Property | Detail |
|---|---|
| **Purpose** | Full audit trail of project actions |
| **Frontend** | `ActivityLogCard.tsx` |
| **Backend** | `logActivity()` called throughout project actions |
| **Database** | `activity_log` table |
| **Functional Status** | COMPLETE |
| **Experience Status** | GOOD |
| **Action** | None |

---

### F16 — Project Status Management

| Property | Detail |
|---|---|
| **Purpose** | Agency updates project status (pending/in-progress/stuck/delayed/complete) |
| **Frontend** | Edit project modal, project header |
| **Backend** | `updateProjectDetailsAction` |
| **Database** | `project.status` enum |
| **Functional Status** | COMPLETE |
| **Experience Status** | GOOD |
| **Action** | None |

---

### F17 — Portal Token Rotation

| Property | Detail |
|---|---|
| **Purpose** | Generate new secure portal link (invalidates old) |
| **Frontend** | `PortalLinkCard.tsx` |
| **Backend** | `rotatePortalTokenAction` |
| **Database** | `client.portalTokenHash`, `client.portalTokenExpiresAt` |
| **Functional Status** | COMPLETE |
| **Experience Status** | GOOD |
| **Action** | None |

---

### F18 — Leads CRM Pipeline

| Property | Detail |
|---|---|
| **Purpose** | Track prospective clients through a sales pipeline |
| **Frontend** | `LeadsDashboard` with kanban columns |
| **Backend** | `src/app/actions/leads.ts` — full CRUD |
| **Database** | `leads`, `lead_activity_log` tables |
| **Functional Status** | MOSTLY COMPLETE |
| **Experience Status** | GOOD |
| **Missing** | `softDeleteLeadsAction` has no ownership check (security bug); no lead import; no email outreach |
| **Action** | P0 — Fix ownership check; P3 — Add lead import |

---

### F19 — Lead to Client Conversion

| Property | Detail |
|---|---|
| **Purpose** | Convert a won lead into a client automatically |
| **Frontend** | Convert button in leads UI |
| **Backend** | `convertLeadToClientAction` |
| **Functional Status** | MOSTLY COMPLETE |
| **Experience Status** | GOOD |
| **Missing** | Does not send invitation email to new client |
| **Action** | P2 — Send client invitation email on conversion |

---

### F20 — In-App Notifications

| Property | Detail |
|---|---|
| **Purpose** | Agency receives notifications for important events |
| **Frontend** | `NotificationDropdown.tsx`, `NotificationsPageClient.tsx` |
| **Backend** | `src/app/actions/notifications.ts` |
| **Database** | `notification` table |
| **Functional Status** | PARTIALLY COMPLETE |
| **Experience Status** | ACCEPTABLE |
| **Missing** | `file_uploaded` notifications never created; `checkDeadlineNotifications` never called; no email for agency notifications |
| **Action** | P1 — Create file_uploaded notification in UploadThing callback; P1 — Wire deadline check to cron |

---

### F21 — Analytics Dashboard

| Property | Detail |
|---|---|
| **Purpose** | Agency views performance metrics |
| **Frontend** | `src/app/dashboard/analytics/page.tsx` |
| **Backend** | Inline DB queries in page component |
| **Database** | `project`, `client`, `checklist_item` tables |
| **Functional Status** | PARTIALLY COMPLETE |
| **Experience Status** | ACCEPTABLE |
| **Missing** | Point-in-time only; no trends; no revenue data; no lead analytics; plan-gated behind `fullAnalytics` |
| **Action** | P3 — Add time-series tracking; P4 — Revenue analytics |

---

### F22 — Billing & Subscription Management

| Property | Detail |
|---|---|
| **Purpose** | Agency subscribes to paid plans via Razorpay |
| **Frontend** | `src/app/dashboard/billing/page.tsx` |
| **Backend** | Razorpay API routes, webhook handler, billing actions |
| **Database** | `subscription`, `payment`, `webhookEvent` tables |
| **Functional Status** | PARTIALLY COMPLETE |
| **Experience Status** | ACCEPTABLE |
| **Missing** | Webhook emails use hardcoded address; subscription cancellation doesn't downgrade plan; payment status enum mismatch; no Stripe for US |
| **Action** | P0 — Fix webhook emails; P1 — Fix cancellation downgrade; P1 — Fix payment status; P2 — Evaluate Stripe |

---

### F23 — Plan Feature Gating

| Property | Detail |
|---|---|
| **Purpose** | Enforce plan limits on features |
| **Frontend** | `FeatureLock` component, plan badges |
| **Backend** | `check-plan.ts`, `plan-config.ts` |
| **Functional Status** | MOSTLY COMPLETE |
| **Experience Status** | GOOD |
| **Missing** | `checkProjectLimit` bug (missing status filter) |
| **Action** | P1 — Fix project limit query |

---

### F24 — Notification Preferences (Settings)

| Property | Detail |
|---|---|
| **Purpose** | Agency configures email and reminder preferences |
| **Frontend** | `NotificationPreferences.tsx` in settings |
| **Backend** | `updateNotificationPreferences` action |
| **Database** | `agency.emailNotifications`, `agency.autoReminders`, `agency.weeklyDigest` |
| **Functional Status** | COMPLETE |
| **Experience Status** | GOOD |
| **Action** | None |

---

### F25 — Files Overview Page

| Property | Detail |
|---|---|
| **Purpose** | Agency sees all submitted files across all projects |
| **Frontend** | `AllFilesPageClient.tsx` |
| **Backend** | `getAllProjectsWithFiles()` in `src/lib/data.ts` |
| **Functional Status** | MOSTLY COMPLETE |
| **Experience Status** | GOOD |
| **Missing** | "Download All" button likely not functional |
| **Action** | P2 — Verify download functionality |

---

### F26 — Meetings

| Property | Detail |
|---|---|
| **Purpose** | Agency schedules and tracks meetings |
| **Frontend** | `MeetingsCalendar.tsx` |
| **Backend** | NO server actions exist |
| **Database** | `meeting` table (schema exists) |
| **Functional Status** | FRONTEND ONLY / PLACEHOLDER |
| **Experience Status** | NOT APPLICABLE |
| **Action** | P1 — Implement meeting server actions; P1 — Wire calendar to DB |

---

### F27 — Landing Page

| Property | Detail |
|---|---|
| **Purpose** | Marketing page to convert visitors to signups |
| **Frontend** | `src/app/components/` sections + `App.tsx` |
| **Functional Status** | MOSTLY COMPLETE |
| **Experience Status** | GOOD |
| **Missing** | Static testimonials; static user logos; pricing inconsistency with plan-config |
| **Action** | P2 — Fix pricing inconsistency; P3 — Real testimonials |

---

### F28 — Blog

| Property | Detail |
|---|---|
| **Purpose** | SEO content to drive organic traffic |
| **Frontend** | `src/app/blog/` |
| **Functional Status** | MOSTLY COMPLETE |
| **Experience Status** | ACCEPTABLE |
| **Missing** | No CMS; hardcoded content; no search |
| **Action** | P3 — SEO metadata; P4 — CMS integration |

---

### F29 — Legal Pages

| Property | Detail |
|---|---|
| **Purpose** | Terms, Privacy, GDPR, Cookie Policy |
| **Frontend** | `src/app/legal/` |
| **Functional Status** | UNKNOWN (dirs exist, content not verified) |
| **Experience Status** | UNKNOWN |
| **Action** | VERIFY — Check content exists and is legally sufficient |

---

## FUTURE FEATURES (Schema Exists But Not Implemented)

| Feature | Database Evidence | Status |
|---|---|---|
| **Reviews/Testimonials** | `review` table in schema | NOT IMPLEMENTED in UI |
| **Team Members** | `agency.maxTeamMembers` in plan-config | NOT IMPLEMENTED |
| **White-labeling** | `customBranding` in plan-config | NOT IMPLEMENTED |
| **Invoice Generation** | `invoiceGeneration` in plan-config | NOT IMPLEMENTED |
| **Lead Follow-Up Automation** | `leadFollowUp` in plan-config | NOT IMPLEMENTED |
| **Weekly Digest** | `agency.weeklyDigest` flag | NOT IMPLEMENTED |
