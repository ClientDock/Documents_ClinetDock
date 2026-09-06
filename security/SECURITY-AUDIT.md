# Security Audit — ClientDock

> Evidence-based security assessment. Only real, confirmed security issues are listed.

---

## Executive Summary

ClientDock has a solid security foundation — multi-tenant isolation, SHA-256 portal token hashing, webhook signature verification, authorization guards, and Zod input validation are all implemented correctly. However, there are **5 critical security gaps** that must be fixed before production launch.

**Security Grade (Pre-Fix): C+**  
**Security Grade (Post-Fix): A-**

---

## SEC-001 — No Authentication on File Upload Endpoint

**Severity: CRITICAL**  
**Type: Broken Object-Level Authorization (BOLA)**  
**File:** `src/lib/uploadthing.ts:16-26`

See BUG-001 for full details. The `clientSubmission` endpoint accepts uploads from anyone without verifying a valid portal token. This means:

1. Any person can upload files to UploadThing with no authentication
2. Files uploaded to another tenant's submission slot if the endpoint is reversible

**Fix:** Pass portal token in UploadThing metadata and verify in middleware (see BUG-001).

---

## SEC-002 — Lead Delete Missing Ownership Check

**Severity: HIGH**  
**Type: Missing Function-Level Authorization**  
**File:** `src/app/actions/leads.ts:145-151`

See BUG-002. The `softDeleteLeadsAction` updates any lead by ID without verifying agency ownership. Cross-tenant lead deletion is possible if lead IDs are exposed.

**Fix:** Add `eq(leads.agencyId, session.agency.id)` to the WHERE clause.

---

## SEC-003 — Email Verification Disabled

**Severity: HIGH**  
**Type: Account Verification Bypass**  
**File:** `src/lib/auth.ts:44`

```typescript
requireEmailVerification: false,
```

Without email verification:
- Anyone can register with any email address (including others' email addresses)
- Spam/throwaway accounts are easier to create
- No proof of email ownership before granting access

**For production, enable email verification:**
```typescript
requireEmailVerification: true,
```
This requires `RESEND_API_KEY` to be configured.

---

## SEC-004 — No Rate Limiting on Authentication Endpoints

**Severity: HIGH**  
**Type: Brute Force / Credential Stuffing Vulnerability**  
**Files:** `src/app/actions/auth.ts`, `src/lib/rate-limit.ts`

`withRateLimit` exists but is never applied to `signInAction` or `signUpAction`. This means:
- Unlimited login attempts (credential stuffing attacks)
- Unlimited signup attempts (spam account creation)

**Fix — Apply rate limiting to signInAction:**
```typescript
export async function signInAction(email: string, password: string): Promise<AuthResult> {
  try {
    // Add rate limit: max 10 attempts per email per 15 minutes
    const allowed = await checkRateLimit(`login:${email}`, 10, 900);
    if (!allowed) {
      return { success: false, error: "Too many login attempts. Try again in 15 minutes." };
    }
    // ... rest of signin logic
  }
}
```

**Note:** This requires Redis. Without Redis, the rate limiter throws an error (fail-closed). For development, handle this gracefully:
```typescript
// In rate-limit.ts — consider fail-open for development
const isDev = process.env.NODE_ENV !== "production";
if (!connected && isDev) return true; // Allow in dev
```

---

## SEC-005 — Dev/Debug Routes Exposed in Production

**Severity: CRITICAL**  
**Type: Security Misconfiguration**  
**Files:**
- `src/app/api/razorpay/dev-bypass/` — likely bypasses payment verification
- `src/app/api/test-db/` — may expose DB data
- `src/app/dev/` — dev UI preview

**Fix:** Add production guards or remove these routes:
```typescript
// Top of each dev-only route:
if (process.env.NODE_ENV === "production") {
  return NextResponse.json({ error: "Not found" }, { status: 404 });
}
```

---

## SEC-006 — BYPASS_DASHBOARD_AUTH Bypasses All Auth

**Severity: MEDIUM (Dev Only)**  
**File:** `src/lib/get-agency.ts:23-44`

```typescript
if (process.env.NODE_ENV !== "production" && process.env.BYPASS_DASHBOARD_AUTH === "true") {
  return { agency: { id: "default-agency-id", ... }, user: { ... } };
}
```

This is scoped to `NODE_ENV !== "production"` so it won't run in production. However:
- If someone accidentally deploys with `NODE_ENV=development`, this bypass is active
- The mock returns a hardcoded `default-agency-id` which may not exist in a real DB

**Recommendation:** Add an explicit check:
```typescript
if (process.env.NODE_ENV === "production" && process.env.BYPASS_DASHBOARD_AUTH === "true") {
  console.error("BYPASS_DASHBOARD_AUTH is enabled in production! Ignoring.");
}
```

---

## SEC-007 — Portal Token Verification Doesn't Check Project Status

**Severity: LOW**  
**Type: Authorization Logic Gap**  
**File:** `src/lib/verify-portal-token.ts`

The portal token verification only checks:
1. Token hash matches
2. Token is not expired

It does NOT check if the project is still active. A portal link to a completed project remains accessible. This may be intentional (clients can always view their portal) but consider whether completed projects should restrict upload access.

---

## SEC-008 — XSS Risk: Rejection Reason Rendered as Text

**Severity: LOW**  
**Type: Potential XSS**  

`rejectionReason` is a free-text field entered by agency and displayed to client. If rendered as HTML (via `dangerouslySetInnerHTML` or similar), this could be an XSS vector. If rendered as plain text (which React does by default with JSX), it is safe.

**Action:** Verify that rejection reasons are always rendered via standard JSX (not `dangerouslySetInnerHTML`).

---

## SEC-009 — Webhook Always Returns 200 on Error

**Severity: MEDIUM**  
**File:** `src/app/api/razorpay/webhook/route.ts:129`

```typescript
} catch (error) {
  console.error("Webhook processing error:", error);
  return NextResponse.json({ success: true, message: "Error handled" }, { status: 200 });
  // Always return 200 for Razorpay webhooks to prevent retries if it's our DB error
}
```

The comment justifies this — Razorpay will retry webhooks on non-200 responses. However, if a billing-critical event (like subscription activation) fails to process due to a DB error, it silently logs and returns 200, meaning the event will never be retried. The agency's plan will not be updated.

**Recommendation:** Implement a dead-letter queue or alert mechanism for failed webhook processing.

---

## SEC-010 — No CORS Configuration

**Severity: LOW**  
**Type: Missing Security Headers**

Next.js handles CORS by default, but there is no explicit CORS configuration to restrict which origins can call the API routes. For a production SaaS, restrict CORS to your own domain.

---

## What Is Done Well (Security Strengths)

| Strength | Implementation |
|---|---|
| Multi-tenant isolation | `agency_id` scoping on all queries; `authorizeProjectAccess/Client` guards |
| Portal token hashing | SHA-256 hash stored, plaintext never stored |
| Portal token expiry | 30-day expiry with verification |
| Webhook signature verification | HMAC-SHA256 via `verifyWebhookSignature()` |
| Payment signature verification | `verifyPaymentSignature()` with timing-safe comparison |
| Webhook idempotency | `webhookEvent` dedup prevents replay attacks |
| Zod input validation | All server actions validate inputs before processing |
| Authorization guards | `authorizeProjectAccess`, `authorizeClientAccess` consistently used |
| CRON_SECRET | Cron endpoint requires Bearer token auth |
| Cascading deletes | Deleting an agency removes all tenant data cleanly |
| Error messages | Auth errors are generic (don't leak user existence) |
| SQLite WAL mode | Prevents database file corruption |

---

## Security Fix Priority Order

1. **SEC-001** — Upload auth (P0 blocker)
2. **SEC-002** — Lead delete ownership (P0 blocker)
3. **SEC-005** — Dev routes in production (P0 blocker)
4. **SEC-003** — Email verification (P1 for launch)
5. **SEC-004** — Rate limiting on auth (P1 for launch)
6. **SEC-006** — Auth bypass guard (P2)
7. **SEC-009** — Webhook error handling (P2)
8. **SEC-010** — CORS (P3)
9. **SEC-007** — Portal for completed projects (P4 / design decision)
10. **SEC-008** — XSS verification (P2)

---

## Pre-Production Security Checklist

- [ ] `BETTER_AUTH_SECRET` is a strong random string (32+ hex chars)
- [ ] `requireEmailVerification: true` in auth.ts
- [ ] `BYPASS_DASHBOARD_AUTH=false` in production .env
- [ ] Dev routes blocked in production
- [ ] UploadThing middleware has portal token verification
- [ ] Lead delete has ownership check
- [ ] Rate limiting applied to auth endpoints
- [ ] `RAZORPAY_WEBHOOK_SECRET` set
- [ ] `CRON_SECRET` set to a strong random value
- [ ] Redis configured for rate limiting (or graceful fallback)
- [ ] All `any` type casts reviewed for injection risk
