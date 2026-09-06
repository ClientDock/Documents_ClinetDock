# Database Overview

> Complete technical audit of the ClientDock database design, implementation, and usage.

---

## Engine & Configuration

| Property | Value |
|---|---|
| Engine | SQLite via `better-sqlite3` ^12.11.1 |
| ORM | Drizzle ORM ^0.45.2 (SQLite dialect) |
| Mode | WAL (Write-Ahead Logging) — enabled at connection time |
| synchronous | NORMAL (not FULL — acceptable for this use case) |
| temp_store | MEMORY |
| mmap_size | 256MB |
| Path | Configurable via `DATABASE_PATH` env var; default `./data/clientdock.db` |
| Migrations | Auto-applied on server start via `src/db/migrate.ts` + `src/instrumentation.ts` |
| Migration count | 6 migrations (0000–0005) |

### Why This Matters

SQLite in WAL mode with `synchronous = NORMAL` provides:
- Good concurrent read performance (multiple readers don't block each other)
- Single-writer limitation (SQLite is not horizontally scalable)
- Excellent performance for small-to-medium datasets
- Not suitable for high-concurrency writes (> 100 concurrent writers)

For a SaaS with hundreds to low thousands of agencies, SQLite is entirely appropriate. At ~10,000+ active concurrent agencies, you would consider migrating to PostgreSQL.

---

## Tables Overview

| Table | Row Count Expectation | Purpose | Multi-tenant |
|---|---|---|---|
| `user` | = agency count | Better Auth identity | No (global) |
| `session` | = active sessions | Auth sessions | No (via user) |
| `account` | = user count | Auth credentials + OAuth | No (via user) |
| `verification` | low (temporary) | Email/password verification tokens | No (via user) |
| `agency` | = user count (1:1) | Tenant workspace | ✅ Root tenant |
| `client` | ~5–50 per agency | Agency's clients | ✅ via agencyId |
| `project` | ~1–10 per client | Client engagements | ✅ via clientId→agencyId |
| `checklist_item` | ~5–20 per project | Project requirements | ✅ via projectId |
| `submission` | ~1–5 per item | Client-uploaded files | ✅ via checklistItemId |
| `activity_log` | ~10–50 per project | Audit trail | ✅ via projectId |
| `timeline_events` | ~3–10 per project | Project milestones | ✅ via projectId |
| `notification` | ~10–100 per agency | In-app notifications | ✅ via agencyId |
| `leads` | ~10–200 per agency | CRM pipeline | ✅ via agencyId |
| `lead_activity_log` | ~5–20 per lead | Lead history | ✅ via leadId |
| `subscription` | ~1 per agency | Billing subscription | ✅ via agencyId |
| `payment` | ~12 per year per agency | Payment records | ✅ via agencyId |
| `webhookEvent` | = Razorpay events | Webhook idempotency log | No (global) |
| `review` | ~1–5 per project | Client reviews | ✅ via agencyId |
| `meeting` | ~5–50 per agency | Scheduled meetings | ✅ via agencyId |

---

## Schema Analysis

### `user` Table
```sql
id TEXT PRIMARY KEY
name TEXT NOT NULL
email TEXT NOT NULL UNIQUE
emailVerified INTEGER (boolean) DEFAULT false
image TEXT
createdAt INTEGER (timestamp_ms)
updatedAt INTEGER (timestamp_ms)
```
- Managed entirely by Better Auth
- `emailVerified` is always `false` in current config (verification disabled)

### `agency` Table (Tenant Root)
```sql
id TEXT PRIMARY KEY
user_id TEXT NOT NULL REFERENCES user.id ON DELETE CASCADE
name TEXT NOT NULL
slug TEXT NOT NULL UNIQUE
plan TEXT DEFAULT 'free'  -- enum: free, pro, ultra, custom
razorpay_customer_id TEXT
email_notifications INTEGER (boolean) DEFAULT true
auto_reminders INTEGER (boolean) DEFAULT false
weekly_digest INTEGER (boolean) DEFAULT false
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```
- **Critical**: This is the root of all multi-tenancy
- `slug` is unique — used for agency URLs (not yet surfaced in product)
- `plan` field is the source of truth for billing plan
- ⚠️ `created_at`/`updated_at` use TEXT (ISO strings) while `user.createdAt`/`updatedAt` use INTEGER timestamps — **inconsistency**

### `client` Table
```sql
id TEXT PRIMARY KEY
agency_id TEXT NOT NULL REFERENCES agency.id ON DELETE CASCADE
email TEXT NOT NULL
name TEXT NOT NULL
portal_token_hash TEXT NOT NULL UNIQUE  -- SHA-256 hash of the raw token
portal_token_expires_at TEXT NOT NULL   -- ISO date string
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```
- **Migration note**: Original migration (0000) had `portal_token` column (plaintext). Migration 0004 dropped it and added `portal_token_hash`. This is a **security improvement**.
- No composite unique index on (agency_id, email) — a client with the same email could theoretically be in two agencies, which is fine. But within one agency, `inviteClient()` checks for duplicate email manually.
- ⚠️ No index on `email` — if queries filter by email frequently, add one.

### `project` Table
```sql
id TEXT PRIMARY KEY
client_id TEXT NOT NULL REFERENCES client.id ON DELETE CASCADE
name TEXT NOT NULL
type TEXT
description TEXT
status TEXT DEFAULT 'pending'  -- enum: pending, in-progress, stuck, delayed, complete
priority TEXT DEFAULT 'medium'  -- enum: top, high, medium, low
industry TEXT
budget TEXT  -- stored as text (not validated as number)
progress INTEGER DEFAULT 0  -- 0-100
checklist_data TEXT  -- JSON string (not used by current app?)
due_date TEXT  -- ISO date string
internal_notes TEXT
auto_reminder INTEGER (boolean) DEFAULT false
reminder_frequency_days INTEGER DEFAULT 3
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```
- ⚠️ `budget` is stored as TEXT — not typed. Could be "$5000" or "5000" or "Five thousand." Consider migrating to REAL/INTEGER.
- ⚠️ `progress` column vs computed progress inconsistency (see F10)
- ⚠️ `checklist_data` TEXT (JSON) is defined but not used by current application — may be legacy

### `checklist_item` Table
```sql
id TEXT PRIMARY KEY
project_id TEXT NOT NULL REFERENCES project.id ON DELETE CASCADE
title TEXT NOT NULL
description TEXT
required_files TEXT  -- JSON string array (stored but usage unclear)
completed INTEGER (boolean) DEFAULT false
completed_at TEXT
status TEXT DEFAULT 'pending'  -- enum: pending, uploaded, approved, rejected
rejection_reason TEXT
accepted_formats TEXT  -- comma-separated: "png,svg,pdf" — not enforced server-side
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```
- `required_files` TEXT (JSON) — defined in schema but appears unused in current app
- `accepted_formats` — stored but NOT validated on submission (security gap)

### `submission` Table
```sql
id TEXT PRIMARY KEY
checklist_item_id TEXT NOT NULL REFERENCES checklist_item.id ON DELETE CASCADE
file_url TEXT NOT NULL
file_name TEXT NOT NULL
file_size INTEGER  -- nullable
submitted_at TEXT NOT NULL
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```
- `file_url` is the UploadThing CDN URL — if UploadThing changes or the file is deleted, the URL is dead
- No `submitted_by` field — no way to know if submission was from client or agency

### `activity_log` Table
```sql
id TEXT PRIMARY KEY
project_id TEXT NOT NULL REFERENCES project.id ON DELETE CASCADE
actor TEXT NOT NULL  -- "Agency", "Client", or "System"
action TEXT NOT NULL  -- free-form description
created_at TEXT NOT NULL
```
- Actor is a string, not a FK — no referential integrity on who performed the action
- Used as pseudo-cron state for reminders (LIKE '%reminder%' query) — fragile

### `notification` Table
```sql
id TEXT PRIMARY KEY
agency_id TEXT NOT NULL REFERENCES agency.id ON DELETE CASCADE
type TEXT NOT NULL  -- enum: file_uploaded, deadline_approaching, status_changed, reminder_sent
title TEXT NOT NULL
message TEXT NOT NULL
project_id TEXT  -- nullable
client_name TEXT  -- nullable
read INTEGER (boolean) DEFAULT false
created_at TEXT NOT NULL
```
- ⚠️ No composite index on (agency_id, read) — the code has it in schema but verify in migration
- `project_id` not FK — just a reference string

### `leads` Table
```sql
id TEXT PRIMARY KEY
agency_id TEXT NOT NULL REFERENCES agency.id ON DELETE CASCADE
contact_name TEXT NOT NULL
company_name TEXT
email TEXT
phone TEXT
location TEXT
source TEXT
estimated_value REAL DEFAULT 0
stage TEXT DEFAULT 'new'  -- enum: new, contacted, qualified, proposal_sent, won, lost
follow_up_date INTEGER (timestamp)  -- uses Unix timestamp (different from other date fields!)
notes TEXT
loss_reason TEXT
is_deleted INTEGER (boolean) DEFAULT false  -- soft delete
created_at INTEGER (timestamp) NOT NULL  -- Unix timestamp (different format!)
updated_at INTEGER (timestamp) NOT NULL
```
- ⚠️ `follow_up_date`, `created_at`, `updated_at` use INTEGER (Unix timestamp) while other tables use TEXT (ISO string) — **major inconsistency** that could cause bugs in sorting, comparison, and display
- `is_deleted` soft delete — no recover UI

### `subscription` Table
```sql
id TEXT PRIMARY KEY
agency_id TEXT NOT NULL REFERENCES agency.id ON DELETE CASCADE
razorpay_subscription_id TEXT NOT NULL UNIQUE
razorpay_plan_id TEXT NOT NULL
plan_name TEXT  -- enum: free, pro, ultra, custom
status TEXT  -- enum: created, authenticated, active, halted, cancelled, expired, completed, pending
current_period_start TEXT
current_period_end TEXT
cancelled_at TEXT
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```

### `payment` Table
```sql
id TEXT PRIMARY KEY
agency_id TEXT NOT NULL REFERENCES agency.id ON DELETE CASCADE
subscription_id TEXT REFERENCES subscription.id ON DELETE CASCADE
razorpay_payment_id TEXT UNIQUE
razorpay_order_id TEXT
amount INTEGER NOT NULL  -- in paise/cents
currency TEXT DEFAULT 'USD'  -- ⚠️ Razorpay uses INR, not USD
status TEXT  -- enum: created, authorized, captured, failed, refunded
method TEXT
invoice_url TEXT
failure_reason TEXT
created_at TEXT NOT NULL
```
- ⚠️ `currency` defaults to 'USD' but Razorpay is an Indian payment processor that primarily deals in INR
- ⚠️ Billing page checks `payment.status === 'success' || 'paid'` — neither are valid values

---

## Relationship Map

```
user (1) ──── (1) agency
                    │
        ┌───────────┼───────────────┐
        │           │               │
     client       lead           meeting
     (many)      (many)          (many)
        │           │
   project       lead_activity_log
   (many)
        │
   ┌────┴────────────────┐
   │                     │
checklist_item    activity_log
   (many)         (many)
        │
   submission
   (many)
```

Also per agency:
- notification (many)
- subscription (1 active at a time)
- payment (many)
- review (many, via project)

---

## Migration History

| Migration | File | Changes |
|---|---|---|
| 0000 | `0000_fine_prima.sql` | Initial schema: user, session, account, verification, agency, client (with plaintext portal_token), project, checklist_item, submission, activity_log, timeline_events |
| 0001 | `0001_majestic_ironclad.sql` | Unknown — needs inspection |
| 0002 | `0002_powerful_raza.sql` | Unknown — needs inspection |
| 0003 | `0003_steep_chimera.sql` | Unknown — needs inspection |
| 0004 | `0004_drop_client_portal_token.sql` | Replaced `portal_token` (plaintext) with `portal_token_hash` + `portal_token_expires_at` — **important security migration** |
| 0005 | `0005_clumsy_gateway.sql` | Unknown — adds leads, notifications, billing, or meetings tables |

---

## Critical Issues

### Issue 1: Date Type Inconsistency
**Tables using TEXT (ISO string):** agency, client, project, checklist_item, submission, activity_log, subscription, payment, webhook_event, review, meeting  
**Tables using INTEGER (Unix timestamp):** user.createdAt (ms), session.createdAt (ms), leads.createdAt (s), leads.followUpDate (s)

This inconsistency causes:
- Sorting bugs (text vs number comparison)
- Display bugs (date formatting requires different code paths)
- Integration bugs (comparing dates across tables)

**Fix:** Standardize all dates. Recommendation: Use TEXT (ISO 8601) throughout. Migrate the leads table.

### Issue 2: `checklist_item.accepted_formats` Not Enforced
The `accepted_formats` field (comma-separated: "png,svg,pdf") is stored per item but never validated on file upload. Clients can upload any file type regardless of what the agency specified.

### Issue 3: `project.progress` vs Computed Progress
`project.progress` (stored INTEGER) and `calculateProjectProgress()` (computed from completed items) can diverge. `approveChecklistItemAction` does NOT update `project.progress`.

### Issue 4: `activity_log` as Cron State
The cron reminder checks `WHERE action LIKE '%reminder%'` to find the last reminder. This is fragile — any action containing the word "reminder" will be picked up. Add a dedicated `last_reminder_sent_at` field to the `project` table.

### Issue 5: Orphaned Webhook Events
`webhookEvent` table stores all Razorpay webhook payloads for idempotency but has no cleanup mechanism. Over time, this table will grow unboundedly.
