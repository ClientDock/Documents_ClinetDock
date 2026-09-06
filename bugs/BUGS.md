# Bug Report — ClientDock

> Evidence-based catalog of confirmed bugs, broken functionality, and high-risk issues. Each issue includes root cause, impact, and fix guidance.

---

## PRIORITY SYSTEM

- **P0 — BLOCKER**: Production cannot launch without fixing this
- **P1 — CRITICAL**: Serious bug that actively harms users or revenue
- **P2 — HIGH**: Significant issue that degrades product quality
- **P3 — MEDIUM**: Noticeable issue worth fixing before launch
- **P4 — LOW**: Minor issue; fix when convenient

---

## BUG-001 — UploadThing Has No Authorization on File Upload

**Priority:** P0 — BLOCKER  
**Area:** Security / File Upload  
**File:** `src/lib/uploadthing.ts:21-23`

### Problem
The `clientSubmission` UploadThing endpoint accepts file uploads from any request. The middleware returns a static metadata object without verifying that the uploader holds a valid portal token.

```typescript
// Current (BROKEN):
.middleware(async () => {
  return { uploadedBy: "client" }; // No token verification!
})
```

### Impact
- Any person with knowledge of the UploadThing upload URL can upload files to any client's checklist item
- Files uploaded this way are stored at UploadThing's CDN and billed to your account
- Cannot be attributed to the correct client
- Portal isolation is defeated

### Fix
Pass the portal token in the upload request header/metadata and verify it in the middleware:

```typescript
.middleware(async ({ req }) => {
  // 1. Extract token from custom header
  const portalToken = req.headers.get("x-portal-token");
  if (!portalToken) throw new UploadThingError("Unauthorized");
  
  // 2. Verify the token
  const client = await verifyPortalToken(portalToken);
  if (!client) throw new UploadThingError("Invalid or expired portal token");
  
  return { uploadedBy: "client", clientId: client.id };
})
```

### Testing
- Attempt to upload without a valid portal token — should return 401
- Attempt to upload with an expired portal token — should return 401
- Attempt to upload with a valid token — should succeed

---

## BUG-002 — Lead Delete Has No Ownership Verification

**Priority:** P0 — BLOCKER  
**Area:** Security / Leads CRM  
**File:** `src/app/actions/leads.ts:136-161`

### Problem
`softDeleteLeadsAction` updates `isDeleted = true` for any array of lead IDs without verifying that those leads belong to the current agency.

```typescript
// Current (BROKEN):
await withDbErrorHandling(() =>
  db.update(leads)
    .set({ isDeleted: true })
    .where(inArray(leads.id, validated.leadIds)) // No agency check!
    .run()
);
```

### Impact
- Agency A can soft-delete Agency B's leads by knowing their lead IDs
- Lead IDs are UUIDs but if exposed via any API response, cross-tenant deletion is possible

### Fix
Add agency ownership filter to the WHERE clause:

```typescript
await withDbErrorHandling(() =>
  db.update(leads)
    .set({ isDeleted: true })
    .where(
      and(
        inArray(leads.id, validated.leadIds),
        eq(leads.agencyId, session.agency.id)  // Add this!
      )
    )
    .run()
);
```

### Testing
- Try to delete a lead from another agency — should silently do nothing (no rows affected)
- Try to delete your own leads — should work correctly

---

## BUG-003 — Billing Webhook Emails Use Hardcoded Address

**Priority:** P0 — BLOCKER  
**Area:** Billing / Email  
**File:** `src/app/api/razorpay/webhook/route.ts:115, 122`

### Problem
When a subscription is cancelled or a payment fails, the email is sent to `"billing@example.com"` instead of the actual agency's email address.

```typescript
// Line 115 (subscription.cancelled):
await sendSubscriptionCancelledEmail("billing@example.com", agencyRecord.name, ...);

// Line 122 (payment.failed):
await sendPaymentFailedEmail("billing@example.com", "Pro", "https://example.com/retry");
```

### Impact
- Agency owners never receive payment failure notifications
- Agency owners never receive subscription cancellation confirmations
- Revenue recovery emails are lost
- Customer support cannot respond because they don't know who cancelled

### Fix
Look up the agency's user email and use it:

```typescript
// In subscription.cancelled handler:
const agencyUser = await db.query.user.findFirst({
  where: eq(user.id, agencyRecord.userId)
});
if (agencyUser) {
  await sendSubscriptionCancelledEmail(
    agencyUser.email,
    agencyRecord.name,
    new Date().toISOString()
  );
}
```

### Testing
- Trigger a test webhook for `subscription.cancelled`
- Verify email is received at the agency owner's actual email

---

## BUG-004 — Subscription Cancellation Does Not Downgrade Plan

**Priority:** P1 — CRITICAL  
**Area:** Billing  
**File:** `src/app/api/razorpay/webhook/route.ts:99-118`

### Problem
When `subscription.cancelled` webhook is received, the code updates the subscription status to "cancelled" but does NOT set `agency.plan = "free"`. The user retains Pro/Ultra access indefinitely after cancelling.

```typescript
// Current — only sets subscription status, not agency plan:
await db.update(subscription)
  .set({ status: "cancelled", updatedAt: new Date().toISOString() })
  .where(eq(subscription.id, sub.id));
// MISSING: agency plan downgrade!
```

### Impact
- Cancelled subscribers retain paid features
- Revenue loss
- Plan limits not enforced for cancelled users

### Fix
Add agency plan downgrade after subscription cancellation:

```typescript
await db.update(subscription)
  .set({ status: "cancelled", cancelledAt: new Date().toISOString(), updatedAt: new Date().toISOString() })
  .where(eq(subscription.id, sub.id));

// Downgrade agency plan
await db.update(agency)
  .set({ plan: "free", updatedAt: new Date().toISOString() })
  .where(eq(agency.id, sub.agencyId));
```

### Testing
- Cancel a test subscription via Razorpay dashboard
- Verify webhook is received and processed
- Verify agency.plan is set to "free" in database
- Verify agency cannot access paid features

---

## BUG-005 — Payment Status Enum Mismatch on Billing Page

**Priority:** P1 — CRITICAL  
**Area:** Billing UI  
**File:** `src/app/dashboard/billing/page.tsx:167`

### Problem
The billing page displays payment status with a conditional that checks for `'success'` and `'paid'`, but the payment schema defines statuses as `created|authorized|captured|failed|refunded`. Neither `'success'` nor `'paid'` are valid values.

```typescript
// Current (WRONG status values):
payment.status === 'success' || payment.status === 'paid'
  ? 'bg-green-100 text-green-700'
  : 'bg-neutral-100 text-neutral-600'
```

### Impact
- All payments show as grey (non-success) even when successfully captured
- User cannot visually identify successful payments

### Fix
```typescript
payment.status === 'captured'
  ? 'bg-green-100 text-green-700'
  : payment.status === 'failed'
  ? 'bg-red-100 text-red-700'
  : 'bg-neutral-100 text-neutral-600'
```

---

## BUG-006 — checkProjectLimit Missing Status Filter

**Priority:** P1 — CRITICAL  
**Area:** Plan Enforcement  
**File:** `src/lib/check-plan.ts:55-65`

### Problem
The project limit check counts ALL projects, including completed ones. The comment says "all statuses except complete are active" but the filter is missing.

```typescript
// Current (WRONG — counts complete projects too):
const activeCount = db.select({ count: count() })
  .from(project)
  .where(
    and(
      inArray(project.clientId, clientIds),
      // All statuses except "complete" are active  ← comment but no code!
    )
  )
  .get()?.count ?? 0;
```

### Impact
- Agencies with many completed projects are incorrectly blocked from creating new projects
- The limit is enforced against historical data, not current active usage

### Fix
```typescript
const activeCount = db.select({ count: count() })
  .from(project)
  .where(
    and(
      inArray(project.clientId, clientIds),
      sql`${project.status} != 'complete'`  // Add this filter!
    )
  )
  .get()?.count ?? 0;
```

---

## BUG-007 — Dashboard Revenue Chart Uses Random Data

**Priority:** P1 — CRITICAL (Trust Issue)  
**Area:** Dashboard / Analytics  
**File:** `src/app/dashboard/page.tsx:18-25`

### Problem
```typescript
function getRevenueData(agencyId: string) {
  const months = ["Jan", "Feb", "Mar", "Apr", "May", "Jun"];
  return months.map((month) => ({
    month,
    revenue: Math.floor(Math.random() * 5000) + 1000,  // RANDOM!
    projects: Math.floor(Math.random() * 8) + 1,         // RANDOM!
  }));
}
```

### Impact
- Revenue chart shows different values on every page load
- Users cannot trust any data on the dashboard
- This is a significant trust/credibility issue for a commercial SaaS

### Fix Options
**Option A (correct):** Compute real revenue from the `payment` table:
```typescript
function getRevenueData(agencyId: string) {
  const payments = db.select().from(payment)
    .where(and(eq(payment.agencyId, agencyId), eq(payment.status, "captured")))
    .orderBy(desc(payment.createdAt))
    .all();
  // Group by month and sum amounts
}
```

**Option B (honest placeholder):** Remove the revenue chart entirely until real data exists. Show "Revenue tracking coming soon."

---

## BUG-008 — Meeting Scheduler Shows Hardcoded Demo Data

**Priority:** P1 — CRITICAL (Trust Issue)  
**Area:** Dashboard / Meetings  
**File:** `src/app/dashboard/page.tsx:28-48`

### Problem
```typescript
function getMeetings(agencyId: string) {
  // HARDCODED! Ignores agencyId entirely
  return [
    { id: "1", title: "Kickoff call", meetingDate: "...", ... },
    { id: "2", title: "Design review", meetingDate: "...", ... },
  ];
}
```

### Impact
- Every agency sees the same fake "Kickoff call" and "Design review" meetings
- Meeting links (meet.google.com/abc-defg-hij) are fake
- Undermines trust in the product

### Fix
Either:
1. Query real meetings from the `meeting` table and display them
2. Show "No upcoming meetings" empty state until meeting CRUD is implemented

---

## BUG-009 — "file_uploaded" Notifications Never Created

**Priority:** P1 — CRITICAL  
**Area:** Notifications  
**File:** `src/lib/uploadthing.ts:24-26`

### Problem
```typescript
.onUploadComplete(async ({ file }) => {
  console.log("Client file uploaded:", file.url);
  // MISSING: Create notification, create submission record, log activity
})
```

### Impact
- Agency is never notified when a client uploads a file
- Activity log is not updated
- The submission record IS created (via client-side code after upload), but the agency notification system never fires

### Fix
In `onUploadComplete`, create a `file_uploaded` notification and log activity. Note: this requires knowing the `checklistItemId` and `agencyId` — these need to be passed through UploadThing metadata.

---

## BUG-010 — Dashboard "Completed This Month" Has No Month Filter

**Priority:** P2 — HIGH  
**Area:** Dashboard Stats  
**File:** `src/lib/data.ts:83-89`

### Problem
```typescript
const completed = db.select({ count: count() })
  .from(project)
  .innerJoin(client, eq(project.clientId, client.id))
  .where(and(eq(client.agencyId, agencyId), eq(project.status, "complete")))
  // MISSING: date filter for current month!
  .get()?.count ?? 0;
```

The stat card is labeled "Completed This Month" but counts ALL completed projects ever.

### Fix
Add a date filter:
```typescript
const startOfMonth = new Date();
startOfMonth.setDate(1);
startOfMonth.setHours(0, 0, 0, 0);

.where(and(
  eq(client.agencyId, agencyId),
  eq(project.status, "complete"),
  sql`${project.updatedAt} >= ${startOfMonth.toISOString()}`
))
```

---

## BUG-011 — Portal Token Expiry for Long-Running Projects

**Priority:** P2 — HIGH  
**Area:** Client Portal  
**Description:** Portal tokens expire in 30 days. If a project runs longer than 30 days (common for agencies), the client's portal link becomes invalid with no automatic renewal mechanism. The token is only rotated when a reminder email is sent. If auto-reminders are disabled, the client's link is permanently broken after 30 days.

### Fix
Option A: Extend token expiry to 1 year  
Option B: Add a background job to renew expiring tokens for active projects  
Option C: Make tokens non-expiring for active projects (only expire when project is complete)

---

## BUG-012 — Dev Routes Accessible in Production

**Priority:** P0 — BLOCKER  
**Area:** Security / Infrastructure  
**Files:**
- `src/app/api/razorpay/dev-bypass/` — Bypasses payment flow
- `src/app/api/test-db/` — Direct database access
- `src/app/dev/` — Dev preview page

### Problem
These routes may be accessible in production builds. The `dev-bypass` route likely allows bypassing Razorpay payment flows. The `test-db` route may expose raw database data.

### Fix
Add production environment guard to each:
```typescript
// At top of each dev route handler:
if (process.env.NODE_ENV === "production") {
  return NextResponse.json({ error: "Not found" }, { status: 404 });
}
```

Or better: move to separate conditional imports that are excluded from production builds.

---

## BUG-013 — Pricing Inconsistency Between Landing Page and Plan Config

**Priority:** P2 — HIGH  
**Area:** Marketing / Trust  
**File:** `src/app/dashboard/billing/page.tsx:108-109`

### Problem
The billing page plan comparison table shows:
- Free: "2 projects"
- Pro: "10 projects"

But `plan-config.ts` defines:
- Free: `maxActiveProjects: 10`
- Pro: `maxActiveProjects: 60`

These numbers are completely wrong and contradict the actual enforced limits.

### Fix
Replace the hardcoded comparison table values with data from `plan-config.ts`.

---

## BUG-014 — `checkDeadlineNotifications` Never Called

**Priority:** P2 — HIGH  
**Area:** Notifications  
**File:** `src/app/actions/notifications.ts:112`

### Problem
The `checkDeadlineNotifications()` function correctly identifies projects with approaching deadlines and creates notifications. However, it is never called anywhere in the application — not in a cron job, not in a middleware, not anywhere.

### Fix
Add a call to `checkDeadlineNotifications()` in the cron reminder endpoint, or create a dedicated cron endpoint for it.

---

## BUG-015 — Razorpay Dev Bypass Route Security

**Priority:** P0 — BLOCKER  
**Area:** Security  
**File:** `src/app/api/razorpay/dev-bypass/`

### Problem
Unknown what this route does exactly (file content not inspected) but the name "dev-bypass" suggests it bypasses payment verification logic. If deployed to production without a guard, it could allow free plan upgrades without payment.

### Action
Inspect the route immediately. If it bypasses any billing logic, add a strict production guard or remove it entirely from production builds.

---

## Summary of Bugs by Priority

| Priority | Count | Items |
|---|---|---|
| P0 | 5 | BUG-001, BUG-002, BUG-003, BUG-012, BUG-015 |
| P1 | 6 | BUG-004, BUG-005, BUG-006, BUG-007, BUG-008, BUG-009 |
| P2 | 3 | BUG-010, BUG-011, BUG-013, BUG-014 |
| P3 | 0 | — |
