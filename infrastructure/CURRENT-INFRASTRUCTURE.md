# Infrastructure — Current State and Deployment Guide

---

## Development Environment

### Prerequisites
```bash
Node.js >= 20
npm >= 10
Redis (optional — required for rate limiting)
SQLite (auto-created)
```

### Setup
```bash
# 1. Install dependencies
npm install

# 2. Copy and configure environment
cp .env.example .env
# Edit .env with your values

# 3. Initialize database (auto-applied on first start)
npm run dev
# OR manually:
npm run setup-db  # seeds demo data
```

### Environment Variables

| Variable | Required | Description |
|---|---|---|
| `BETTER_AUTH_SECRET` | ✅ YES | 32+ char random string for session signing |
| `BETTER_AUTH_URL` | ✅ YES | App URL (e.g., http://localhost:3000) |
| `NEXT_PUBLIC_BETTER_AUTH_URL` | ✅ YES | Same as above (for client-side) |
| `DATABASE_PATH` | No | SQLite file path (default: ./data/clientdock.db) |
| `BYPASS_DASHBOARD_AUTH` | No | `"true"` for local dev without login |
| `RESEND_API_KEY` | For email | Falls back to console.log without this |
| `EMAIL_FROM` | For email | From address (e.g., noreply@clientdock.app) |
| `EMAIL_FROM_NAME` | No | From name (e.g., ClientDock) |
| `REDIS_URL` | No | Redis connection URL |
| `REDIS_PASSWORD` | No | Redis password |
| `CRON_SECRET` | For cron | Secures cron endpoint |
| `GOOGLE_CLIENT_ID` | No | Google OAuth |
| `GOOGLE_CLIENT_SECRET` | No | Google OAuth |
| `GITHUB_CLIENT_ID` | No | GitHub OAuth |
| `GITHUB_CLIENT_SECRET` | No | GitHub OAuth |
| `RAZORPAY_KEY_ID` | For billing | Razorpay key |
| `RAZORPAY_KEY_SECRET` | For billing | Razorpay secret |
| `RAZORPAY_WEBHOOK_SECRET` | For billing | Webhook validation |
| `NEXT_PUBLIC_RAZORPAY_KEY_ID` | For billing | Client-side Razorpay key |
| `RAZORPAY_PLAN_PRO` | For billing | Razorpay plan ID for Pro |
| `RAZORPAY_PLAN_ULTRA` | For billing | Razorpay plan ID for Ultra |
| `UPLOADTHING_SECRET` | ⚠️ MISSING from .env.example | Required for file uploads |
| `UPLOADTHING_APP_ID` | ⚠️ MISSING from .env.example | Required for file uploads |

> [!WARNING]
> `UPLOADTHING_SECRET` and `UPLOADTHING_APP_ID` are missing from `.env.example` — file uploads will silently fail without these.

---

## Database

### Location
```
./data/clientdock.db        # Default path
./data/clientdock.db-wal    # WAL journal file
./data/clientdock.db-shm    # Shared memory file
```

### Migrations
- Migration files: `./drizzle/*.sql`
- Auto-applied on startup via `src/db/migrate.ts` called from `src/instrumentation.ts`
- 6 migrations exist (0000–0005)
- To generate new migrations: `npm run db:generate`
- To apply manually: `npm run db:migrate`

### Backup
- **Development:** Copy the `.db` file
- **Production:** Litestream (SQLite replication to S3/GCS/R2)

---

## Production Docker Deployment

### Docker Build
```bash
docker build -t clientdock .
```

### Docker Compose
```bash
docker-compose up -d
```

The `docker-compose.yml` includes:
- App container with bind-mount to `./data` directory
- SQLite data persistence across container restarts

### Dockerfile Analysis (Multi-stage)

| Stage | Purpose | Notes |
|---|---|---|
| `deps` | Install production dependencies | Rebuilds `better-sqlite3` native module |
| `builder` | Build Next.js app | Full install + build |
| `runner` | Production runtime | Minimal image + Litestream |

**Known Issue:** The Dockerfile installs Litestream in the runner stage and starts it in the CMD:
```dockerfile
CMD ["sh", "-c", "litestream replicate --config /etc/litestream.yml & ./node_modules/.bin/next start"]
```

But `/etc/litestream.yml` is never copied into the image. Litestream will fail to start. This needs:
1. A `litestream.yml` config file created
2. Copied into the Docker image:
```dockerfile
COPY litestream.yml /etc/litestream.yml
```

### Sample `litestream.yml`
```yaml
dbs:
  - path: /app/data/clientdock.db
    replicas:
      - url: s3://your-bucket/clientdock.db
        access-key-id: ${AWS_ACCESS_KEY_ID}
        secret-access-key: ${AWS_SECRET_ACCESS_KEY}
        region: us-east-1
```

---

## Cron Jobs

### Automated Reminder Cron

**Endpoint:** `GET /api/cron/reminders`  
**Auth:** `Authorization: Bearer $CRON_SECRET`

**External scheduler options:**

**Option 1: Vercel Cron (if deployed to Vercel)**
```json
// vercel.json
{
  "crons": [
    {
      "path": "/api/cron/reminders",
      "schedule": "0 9 * * *"
    }
  ]
}
```

**Option 2: GitHub Actions cron**
```yaml
# .github/workflows/cron.yml
name: Daily Reminders
on:
  schedule:
    - cron: '0 9 * * *'  # 9am UTC daily
jobs:
  remind:
    runs-on: ubuntu-latest
    steps:
      - run: |
          curl -H "Authorization: Bearer ${{ secrets.CRON_SECRET }}" \
            https://your-app-url.com/api/cron/reminders
```

**Option 3: External cron service (cron-job.org, EasyCron, Uptime Robot)**

### Deadline Check (Not Yet Wired)
`checkDeadlineNotifications()` should be called from the same cron job as reminders:
```typescript
// In /api/cron/reminders/route.ts, add:
await checkDeadlineNotifications();
```

---

## CI/CD Pipeline

The GitHub Actions workflow at `.github/workflows/ci.yml` exists. Verify it includes:
- [ ] `npm ci`
- [ ] TypeScript type checking (`npx tsc --noEmit`)
- [ ] Lint check (`npm run lint`)
- [ ] Test run (`npm test`)
- [ ] Docker build test (optional)

---

## Monitoring (Not Yet Implemented)

For production, add at minimum:
1. **Error tracking:** Sentry (`npm install @sentry/nextjs`)
2. **Uptime monitoring:** Better Uptime, Pagerduty, or UptimeRobot
3. **Database monitoring:** SQLite size alerts (alert if DB grows past 5GB)
4. **Cron monitoring:** Healthchecks.io ping after each cron run

---

## Deployment Checklist

### Before Deploying to Production

- [ ] Set `NODE_ENV=production`
- [ ] Set strong `BETTER_AUTH_SECRET` (32+ hex chars)
- [ ] Set `BETTER_AUTH_URL` to production domain
- [ ] Set `NEXT_PUBLIC_BETTER_AUTH_URL` to production domain
- [ ] Set `BYPASS_DASHBOARD_AUTH=false`
- [ ] Configure `RESEND_API_KEY` and email addresses
- [ ] Configure Razorpay production keys (or Stripe)
- [ ] Set `RAZORPAY_WEBHOOK_SECRET`
- [ ] Set `CRON_SECRET` to a strong random value
- [ ] Add `UPLOADTHING_SECRET` and `UPLOADTHING_APP_ID`
- [ ] Create `litestream.yml` and configure S3 bucket
- [ ] Configure Redis if rate limiting is needed
- [ ] Block dev routes in production (dev-bypass, test-db)
- [ ] Enable email verification in auth.ts
- [ ] Set up external cron scheduler for reminders
- [ ] Set up Sentry or equivalent for error tracking
- [ ] Test full signup → invite → portal → upload → approve flow
- [ ] Test Razorpay subscription flow end-to-end
- [ ] Test reminder email delivery
- [ ] Verify database backup/restore via Litestream
