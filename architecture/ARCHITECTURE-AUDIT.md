# Architecture Audit — ClientDock

> Assessment of the technical architecture, its strengths, weaknesses, and where changes are needed.

---

## What Should Remain (Do Not Change)

| Decision | Rationale |
|---|---|
| **Next.js App Router** | Excellent choice: SSR, Server Components, and Server Actions provide a clean architecture. No reason to change. |
| **Server Actions pattern** | Clean, type-safe, and avoids a separate API layer. The `ActionResult<T>` pattern is good. Keep it. |
| **SQLite + Drizzle ORM** | Appropriate for the current scale. WAL mode, mmap, and synchronous=NORMAL are correctly configured. |
| **Better Auth** | Well-integrated, supports all necessary auth flows. Good choice. |
| **UploadThing** | Good choice for file hosting without managing S3 directly. Keep it. |
| **Resend for email** | Clean, modern email API. Good choice. |
| **Single-tenant-to-multi-tenant model (agency per user)** | Well-designed. The 1:1 user-agency relationship simplifies auth while enabling multi-tenancy. |
| **Drizzle migrations (auto-applied)** | `instrumentation.ts` running migrations on startup is elegant for this stack. |
| **plan-config.ts as single source of truth** | Excellent pattern. All plan limits in one place, never duplicated. |
| **Docker + Litestream** | Production-grade SQLite deployment strategy. |

---

## What Should Be Improved (Not Replaced)

### ARCH-001 — Date Type Standardization

**Problem:** Three different date representations across the codebase:
- `TEXT` ISO strings: agency, project, client, checklist_item, etc.
- `INTEGER timestamp_ms`: user, session, account, verification (Better Auth managed)
- `INTEGER timestamp (seconds)`: leads, lead_activity_log

**Impact:** Cannot sort leads by date alongside project dates. UI date formatting requires different code paths. Cross-table date comparisons are error-prone.

**Recommendation:** Standardize application-owned tables to use `TEXT` ISO 8601 strings. Better Auth tables (user, session, account, verification) use INTEGER timestamps — leave these as-is (managed by Better Auth). Migrate `leads` and `lead_activity_log` to TEXT.

**Migration approach:**
```sql
-- Example: migrate leads.created_at from INTEGER to TEXT ISO string
ALTER TABLE leads RENAME COLUMN created_at TO created_at_unix;
ALTER TABLE leads ADD COLUMN created_at TEXT;
UPDATE leads SET created_at = datetime(created_at_unix, 'unixepoch');
-- After verifying data: DROP COLUMN created_at_unix
```

---

### ARCH-002 — Progress Tracking Inconsistency

**Problem:** `project.progress` is stored as an integer AND computed from checklist items. They can diverge because approval actions don't update the stored value.

**Recommendation:** Establish a single strategy:

**Option A — Always Computed (Recommended):**
Remove `progress` from DB. Compute it everywhere from checklist items. Add it to the response shape as a virtual field. Pro: always accurate. Con: extra query.

**Option B — Stored, Always Updated:**
Keep `progress` column. Update it in `approveChecklistItemAction` and `rejectChecklistItemAction`:
```typescript
const newProgress = calculateProjectProgress(item.projectId);
await updateProject(item.projectId, { progress: newProgress });
```
This is the faster fix.

---

### ARCH-003 — Rate Limiting Requires Redis

**Problem:** The rate limiter fails closed — if Redis is unavailable, all requests are blocked. This makes local development difficult and creates a reliability dependency.

**Current code:**
```typescript
} catch {
  // If Redis is unavailable, fail-closed to prevent abuse
  console.warn(`⚠️ Rate limit check failed for ${key}, blocking request`);
  return false; // BLOCKS the request!
}
```

**Recommendation:**
- In development (`NODE_ENV !== 'production'`): fail-open (allow the request)
- In production: fail-closed is correct but log an alert

```typescript
} catch {
  if (process.env.NODE_ENV !== 'production') {
    return true; // Allow in dev — Redis not required
  }
  console.error(`CRITICAL: Rate limit check failed for ${key}`);
  return false; // Block in production
}
```

---

### ARCH-004 — Activity Log Used as Cron State Machine

**Problem:** The cron reminder uses `activityLog` with a `LIKE '%reminder%'` query to determine when the last reminder was sent. This is fragile — any action containing "reminder" in the text will match.

**Recommendation:** Add a dedicated `lastReminderSentAt TEXT` field to the `project` table:

```typescript
// In schema.ts:
lastReminderSentAt: text("last_reminder_sent_at"),
```

Update this field when a reminder is sent. The cron query becomes:
```typescript
const daysSinceLast = project.lastReminderSentAt 
  ? (Date.now() - new Date(project.lastReminderSentAt).getTime()) / 86400000
  : Infinity;
if (daysSinceLast < project.reminderFrequencyDays) continue;
```

This is cleaner, more reliable, and queryable.

---

### ARCH-005 — File Size Tracking Not Persisted Correctly

**Problem:** `submission.fileSize` is nullable. UploadThing provides the file size in the upload callback, but the current `onUploadComplete` only logs to console. The size may not be reliably stored.

**Recommendation:** Ensure the `onUploadComplete` callback stores file metadata to the database:
```typescript
.onUploadComplete(async ({ file, metadata }) => {
  await createSubmission(metadata.checklistItemId, {
    fileUrl: file.url,
    fileName: file.name,
    fileSize: file.size,
  });
})
```

---

### ARCH-006 — `any` Type Casts in Server Actions

**Problem:** Multiple files use `eslint-disable @typescript-eslint/no-explicit-any` and `as any` casts, particularly in `projects.ts` and `project-details.ts`. This defeats TypeScript's safety guarantees.

**Specific instances:**
- `src/app/actions/projects.ts:1-2` — eslint-disable at file top
- `src/app/actions/project-details.ts:1-2` — eslint-disable at file top
- `updateChecklistItem(item.id, { acceptedFormats } as any)` — missing field in type
- `{ status: "rejected" as any, rejectionReason: ... }` — enum casting issues

**Recommendation:** Fix the Drizzle type definitions to match actual usage. The `any` casts indicate a mismatch between the schema types and what the code needs to update.

---

### ARCH-007 — N+1 Query Patterns in Data Layer

**Problem:** `getAllProjectsWithFiles()` in `src/lib/data.ts:335-399` has a classic N+1 query pattern:
1. Fetch all projects (1 query)
2. For each project, fetch checklist items (N queries)
3. For each checklist item, fetch submissions (N*M queries)

For an agency with 20 projects, each with 10 items = 201+ queries for one page load.

**Recommendation:** Use JOIN-based queries:
```typescript
// One query to get all submissions for all checklist items across all projects
db.select()
  .from(submission)
  .innerJoin(checklistItem, eq(submission.checklistItemId, checklistItem.id))
  .innerJoin(project, eq(checklistItem.projectId, project.id))
  .innerJoin(client, eq(project.clientId, client.id))
  .where(eq(client.agencyId, agencyId))
  .all();
```

---

## What Does NOT Need to Change

| Concern | Verdict |
|---|---|
| SQLite → PostgreSQL | **NOT needed** for current scale. Keep SQLite. |
| Monorepo → Microservices | **NOT needed**. This is appropriate as a monolith. |
| Server Actions → REST API | **NOT needed**. Server Actions work well with Next.js App Router. |
| Better Auth → Custom Auth | **NOT needed**. Better Auth is a solid library. |
| Drizzle → Prisma | **NOT needed**. Drizzle is lighter and correct for SQLite. |
| TypeScript → JavaScript | **NOT needed**. TypeScript is the right choice. |
| Redis → In-memory cache | **NOT needed**. Redis is appropriate; just handle unavailability gracefully. |

---

## Technical Debt Summary

| Item | Severity | Effort | Priority |
|---|---|---|---|
| Date type inconsistency | Medium | Medium | P2 |
| Progress tracking inconsistency | Medium | Low | P1 |
| Rate limit fail-closed in dev | Low | Low | P2 |
| Activity log as cron state | Medium | Low | P2 |
| File size not persisted | Low | Low | P2 |
| `any` type casts | Low | Medium | P3 |
| N+1 queries in files page | Medium | Medium | P3 |
| `project.checklist_data` unused field | Low | Low | P4 |
| `checklist_item.required_files` unused | Low | Low | P4 |
| Duplicate `ActionResult<T>` type definitions | Low | Low | P4 |
