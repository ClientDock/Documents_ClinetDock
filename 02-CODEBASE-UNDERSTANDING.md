# 02 — Codebase Understanding

> Deep technical map of the existing implementation. Use this document to understand how the code is actually structured before making any changes.

---

## Directory Structure Analysis

### `src/app/` — Next.js App Router

#### Pages and Routes

| Route | File | Type | Notes |
|---|---|---|---|
| `/` | `app/page.tsx` → `App.tsx` | Landing | Renders all marketing sections |
| `/login` | `app/login/` | Auth | Better Auth client-side login |
| `/signup` | `app/signup/` | Auth | Server action signup |
| `/dashboard` | `app/dashboard/page.tsx` | Dashboard | Main dashboard with stats, projects, meetings |
| `/dashboard/projects` | `app/dashboard/projects/page.tsx` | Dashboard | Project list view |
| `/dashboard/projects/[id]` | `app/dashboard/projects/[id]/page.tsx` | Dashboard | Project detail |
| `/dashboard/clients/[id]` | `app/dashboard/clients/[id]/ClientDetailClient.tsx` | Dashboard | Client detail |
| `/dashboard/leads` | `app/dashboard/leads/page.tsx` | Dashboard | CRM pipeline |
| `/dashboard/analytics` | `app/dashboard/analytics/page.tsx` | Dashboard | Analytics |
| `/dashboard/meetings` | `app/dashboard/meetings/page.tsx` | Dashboard | Meeting calendar |
| `/dashboard/files` | `app/dashboard/files/page.tsx` | Dashboard | All files across projects |
| `/dashboard/notifications` | `app/dashboard/notifications/page.tsx` | Dashboard | Notification center |
| `/dashboard/billing` | `app/dashboard/billing/page.tsx` | Dashboard | Plan and billing |
| `/dashboard/settings` | `app/dashboard/settings/page.tsx` | Dashboard | Settings |
| `/dashboard/account` | `app/dashboard/account/` | Dashboard | Account management |
| `/portal/[token]` | `app/portal/[token]/page.tsx` | Client | Passwordless client portal |
| `/blog` | `app/blog/page.tsx` | Marketing | Blog index |
| `/blog/[slug]` | `app/blog/[slug]/` | Marketing | Individual articles |
| `/api/auth/[...all]` | `app/api/auth/` | API | Better Auth catch-all |
| `/api/uploadthing` | `app/api/uploadthing/route.ts` | API | File upload handler |
| `/api/cron/reminders` | `app/api/cron/reminders/route.ts` | API | Automated reminder cron |
| `/api/razorpay/create-subscription` | `app/api/razorpay/create-subscription/` | API | Billing |
| `/api/razorpay/verify-payment` | `app/api/razorpay/verify-payment/` | API | Billing |
| `/api/razorpay/webhook` | `app/api/razorpay/webhook/route.ts` | API | Billing webhook |
| `/api/razorpay/dev-bypass` | `app/api/razorpay/dev-bypass/` | API | ⚠️ Dev only — must be blocked in production |
| `/api/clients` | `app/api/clients/` | API | Client API routes |
| `/api/projects` | `app/api/projects/` | API | Project API routes |
| `/api/test-db` | `app/api/test-db/` | API | ⚠️ Dev only — must be removed/blocked |
| `/dev` | `app/dev/` | Dev | ⚠️ Dev preview page — must be blocked in production |

#### Server Actions — `src/app/actions/`

| File | Actions | Description |
|---|---|---|
| `auth.ts` | `signUpAction`, `signInAction`, `signOutAction`, `getSessionAction` | Auth flows |
| `projects.ts` | `createProjectAction`, `inviteClientAction`, `updateProjectDetailsAction`, `rotatePortalTokenAction`, `rotatePortalTokenInternal` | Project/client management |
| `project-details.ts` | `addChecklistItemAction`, `deleteChecklistItemAction`, `approveChecklistItemAction`, `rejectChecklistItemAction`, `updateInternalNotesAction`, `updateReminderSettingsAction`, `sendManualReminderAction` | Project detail operations |
| `leads.ts` | `createLeadAction`, `updateLeadStageAction`, `softDeleteLeadsAction`, `convertLeadToClientAction` | CRM operations |
| `email.ts` | `sendReminderEmailAction`, `sendReminderEmail` | Email sending |
| `notifications.ts` | `getNotifications`, `getUnreadCount`, `markNotificationRead`, `markAllNotificationsRead`, `createNotification`, `checkDeadlineNotifications` | Notification management |
| `settings.ts` | `updateNotificationPreferences` | Settings |

---

## Core Library Files — `src/lib/`

### `auth.ts` — Better Auth Configuration
- Configures Better Auth with Drizzle adapter
- Email/password enabled with verification DISABLED
- Google + GitHub OAuth conditional on env vars
- 7-day sessions with 1-day sliding renewal
- 5-minute cookie cache

### `auth-client.ts` — Client-side Auth
- Exports `authClient` for use in Client Components
- Minimal (270 bytes)

### `get-agency.ts` — Session Resolution
- `getCurrentSession()` — primary function for getting user + agency on server
- Has a `BYPASS_DASHBOARD_AUTH` shortcut for local dev
- Used everywhere in server components and actions
- Returns `null` if no valid session (callers must redirect)

### `authorize.ts` — Authorization Guards
- `authorizeProjectAccess(projectId)` — verifies project belongs to current agency
- `authorizeClientAccess(clientId)` — verifies client belongs to current agency
- Both throw on unauthorized access (callers catch and return error)

### `db.ts` — Database Helper Functions
- CRUD helpers for: projects, checklist items, submissions, clients, activities, agencies, leads, lead activities
- All use Drizzle ORM with synchronous sqlite (`.get()`, `.all()`, `.run()`)
- `inviteClient` checks for existing email duplicate before inserting

### `data.ts` — Dashboard Data Functions
- `getDashboardStats()` — computes 4 stat cards from DB
- `getDashboardProjects()` — maps projects + clients into `DashboardProject` format
- `getRecentActivities()` — last 6 activity log entries
- `getProjectDetail()` — full project detail with checklist + submissions + activity + timeline
- `getAllProjectsWithFiles()` — all projects with their file submissions (for Files page)
- `calculateProjectProgress()` — computes progress from checklist completion

### `plan-config.ts` — Plan Definitions (Single Source of Truth)
- Defines Free, Pro, Ultra, Custom plans with all limits
- `PLANS` record — pricing, limits, CTA labels
- Helper functions: `canUseFeature`, `getLimit`, `isUpgrade`, `getMinimumPlanFor`, `formatPrice`
- Pro: $29/mo, Ultra: $79/mo (with launch offer pricing)

### `check-plan.ts` — Runtime Plan Enforcement
- `getAgencyPlan(agencyId)` — reads plan from DB
- `checkProjectLimit(agencyId)` — enforces maxActiveProjects limit ⚠️ **BUG: missing status filter**
- `checkClientLimit(agencyId)` — enforces maxClients limit
- `requireFeature(agencyId, feature)` — boolean feature gate
- `getAgencyUsage(agencyId)` — returns used/limit for projects, clients, team members
- `getActiveSubscription(agencyId)` — finds active Razorpay subscription

### `razorpay.ts` — Billing Integration
- Singleton Razorpay SDK instance
- Customer creation, subscription creation, subscription cancellation
- Payment signature verification (HMAC-SHA256)
- Webhook signature verification

### `redis.ts` — Redis Client
- Creates Redis client with connection management
- Fail-closed: if Redis unavailable, `incrementCounter` throws
- Used only by `rate-limit.ts`

### `rate-limit.ts` — Rate Limiting
- `checkRateLimit(key, limit, window)` — Redis-based counter
- `withRateLimit(key, limit, window, fn)` — wrapper
- **Currently unused in auth actions** — a significant omission

### `uploadthing.ts` — File Upload
- Defines two upload routes: `clientSubmission` (32MB max), `agencyUpload` (128MB max)
- No auth middleware on either route — **security gap**

### `verify-portal-token.ts` — Portal Token Verification
- Takes raw token string, hashes with SHA-256
- Looks up by hash in client table
- Checks expiry date

### `email.ts` — Email Sender
- Wraps Resend API
- Falls back to console.log in development (when `RESEND_API_KEY` not set)

### `email-templates.ts` — Email HTML Templates
- `verificationEmailHtml` — email verification
- `passwordResetEmailHtml` — password reset
- Plain HTML string templates

### `billing-emails.ts` — Billing Email Templates
- `sendPaymentFailedEmail`, `sendSubscriptionCancelledEmail` — billing-specific emails
- Uses Resend

### `db-errors.ts` — Error Handling Wrapper
- `withDbErrorHandling(fn)` — wraps DB operations, converts common errors to friendly messages
- Handles SQLITE_CONSTRAINT_UNIQUE, SQLITE_READONLY, SQLITE_BUSY, etc.

---

## Database Layer — `src/db/`

### `index.ts` — DB Connection
- Creates better-sqlite3 connection
- Enables WAL mode, sets `synchronous = NORMAL`, `temp_store = MEMORY`, 256MB mmap
- Wraps in Drizzle ORM

### `schema.ts` — Table Definitions
All tables defined with Drizzle ORM SQLite dialect. See [database/DATABASE-SCHEMA.md](./database/DATABASE-SCHEMA.md) for full detail.

### `migrate.ts` — Migration Runner
- Auto-applies all pending migrations from `/drizzle/` directory
- Called from `instrumentation.ts` on server startup

### `setup.ts` — Development Seed Script
- Called by `npm run setup-db`
- Seeds the database with demo data for development

---

## Key Patterns to Follow

### 1. Server Action Pattern
```typescript
"use server";

export async function myAction(data: InputType): Promise<ActionResult<OutputType>> {
  try {
    // 1. Validate input with Zod
    const validated = MySchema.parse(data);
    
    // 2. Get authenticated session
    const session = await getCurrentSession();
    if (!session) return { success: false, error: "Not authenticated" };
    
    // 3. Authorize (check ownership)
    await authorizeProjectAccess(validated.projectId);
    
    // 4. Check plan limits if needed
    const gate = checkProjectLimit(session.agency.id);
    if (!gate.allowed) return { success: false, error: "..." };
    
    // 5. Execute DB operation with error handling
    const result = await withDbErrorHandling(() => someDbOperation(validated));
    
    // 6. Revalidate affected paths
    revalidatePath("/dashboard/...");
    
    return { success: true, data: result };
  } catch (error) {
    if (error instanceof z.ZodError) {
      return { success: false, error: error.issues.map(e => e.message).join(", ") };
    }
    return { success: false, error: error instanceof Error ? error.message : "Unknown error" };
  }
}
```

### 2. Authorization Pattern
```typescript
// Always call authorizeProjectAccess AFTER getting session
// It throws on unauthorized, so wrap in try/catch
await authorizeProjectAccess(projectId);
```

### 3. Database Query Pattern (Drizzle + SQLite)
```typescript
// Synchronous queries use .get() for single row, .all() for multiple
const row = db.select().from(table).where(eq(table.id, id)).get();
const rows = db.select().from(table).where(eq(table.agencyId, agencyId)).all();

// Async operations (when using Drizzle query builder)
const rows = await db.query.table.findMany({ where: eq(table.agencyId, agencyId) });
```

### 4. Plan Gating Pattern
```typescript
// Feature check
const gate = requireFeature(session.agency.id, "automatedReminders");
if (!gate.allowed) {
  return { success: false, error: "Automated reminders require Pro plan." };
}

// Limit check  
const gate = checkClientLimit(session.agency.id);
if (!gate.allowed) {
  return { success: false, error: `Limit reached.` };
}
```

---

## Important Implementation Notes

### Synchronous vs Async DB Access
- `better-sqlite3` is synchronous by design
- Most DB queries in `src/lib/db.ts` and `src/lib/data.ts` use sync APIs
- `src/app/dashboard/billing/actions.ts` uses `await db.query.*` (Drizzle async API)
- **Do not mix async and sync approaches inconsistently** — pick one per context

### Progress Tracking Inconsistency
- `project.progress` stores progress as 0–100 integer
- `calculateProjectProgress()` computes it from `completed` checklist items
- `approveChecklistItemAction` does NOT update `project.progress`
- This means `project.progress` shown in dashboard may be stale
- **Fix: call `calculateProjectProgress` and update `project.progress` after any checklist status change**

### Portal Token Security Model
```
Plaintext token (UUID, never stored) → SHA-256 hash → Stored in DB
When verifying: hash incoming token → compare with stored hash
Token expires in 30 days
Token is rotated on every reminder email sent
```

### TypeScript Patterns
- `ActionResult<T>` discriminated union is the standard return type for server actions
- Zod schemas are defined locally in each action file
- Several files use `any` casts (particularly around DB updates) — a tech debt item
- `eslint-disable` comments in projects.ts indicate type-fighting with Drizzle
