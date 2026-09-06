# Product Understanding — ClientDock

> A deep understanding of the product from a user and business perspective.

---

## What ClientDock Is

ClientDock is a **client onboarding and asset collection portal** for agencies and freelancers. It sits in the gap between project management tools (Asana, Monday.com) and file storage (Google Drive, Dropbox) — neither of which are purpose-built for the "agency collecting deliverables from a client" workflow.

### The Core Problem It Solves

When an agency onboards a new client, they need:
- Brand assets (logos, fonts, colors)
- Credentials (website login, hosting login, social accounts)
- Content (copy, images, videos)
- Approvals (sign-offs on milestones)

Without a structured tool, this becomes:
- Email threads with "can you resend the logo?"
- Slack messages buried in history
- Google Drive folders that clients can't navigate
- Chasing clients for weeks via manual messages

ClientDock provides a **dedicated, organized portal** where each requirement has a checklist item, the client uploads directly, and the agency approves or rejects with feedback.

---

## Core User Workflows

### Workflow 1: Agency Sets Up a New Client
1. Agency logs into dashboard
2. Creates client record (name + email)
3. Gets a unique portal link for that client
4. Creates a project for the client
5. Adds checklist items (what the client needs to submit)
6. Sends the portal link to the client

### Workflow 2: Client Submits Files
1. Client receives portal link (via email — currently must be sent manually)
2. Client opens link (no login required)
3. Sees their project checklist with status indicators
4. Uploads files for each pending item
5. Marks items as submitted

### Workflow 3: Agency Reviews and Approves
1. Agency receives notification (currently only in-app)
2. Opens project dashboard
3. Sees submitted files per checklist item
4. Approves or rejects with a reason
5. Client is notified (currently must check portal manually)

### Workflow 4: Auto-Reminders
1. Agency enables auto-reminders for a project
2. Sets frequency (e.g., every 3 days)
3. Cron job checks pending items
4. Sends email to client with portal link
5. Token is rotated on each reminder

### Workflow 5: Lead to Client Conversion (CRM)
1. Agency tracks a prospect in the Leads pipeline
2. Prospect becomes a client
3. Agency clicks "Convert to Client"
4. Client record is created automatically
5. Agency can now create a project

---

## Product Positioning

ClientDock is NOT:
- A general project management tool (not competing with Asana/Monday)
- A client communication platform (not Slack/Teams)
- A file storage service (not Dropbox/Google Drive)
- A CRM system (leads CRM is supplementary, not core)

ClientDock IS:
- A **structured onboarding workflow** that eliminates "file chasing"
- A **client compliance tool** (getting clients to do what you need, on time)
- A **professional face** for agencies (makes them look organized)
- An **automation layer** (reminders run themselves)

---

## Where the Product Is Strong

1. **Core workflow is clear and well-implemented** — The invite → checklist → upload → approve flow is solid
2. **Zero-friction client experience** — No client registration required; portal token is elegant
3. **Multi-tenant isolation** — Each agency's data is properly isolated
4. **Automated reminders** — The auto-chaser is a unique value proposition
5. **Landing page messaging** — The problem-focused copy resonates with the target audience
6. **Pricing approach** — Free tier (5 clients) is a reasonable hook

---

## Where the Product Has Gaps

### Gap 1: No Client Notification Emails (Critical)
The product promises to "chase clients automatically" but clients don't even receive an email when they're first invited. The manual reminder system exists, but the initial onboarding email is missing.

**Business Impact:** High — without the initial invitation email, clients never know to visit the portal. The entire value proposition breaks down.

### Gap 2: Meetings Are a Dead Feature
The `meeting` table exists, the calendar UI exists, but no server actions exist. This feature is completely non-functional but visibly present in the sidebar.

**Business Impact:** Medium — confusing for users; makes the product look incomplete.

### Gap 3: No Invoice/Payment Tracking
The plan-config mentions `invoiceGeneration` as a feature, but no invoice functionality exists. Agencies want to know what they billed per project.

**Business Impact:** Medium — valuable for agency workflows; differentiating feature.

### Gap 4: No Team Members
The plan-config mentions `maxTeamMembers` (1 for Free, 5 for Pro, unlimited for Ultra), but there is no team member functionality. Agencies have multiple employees who need access.

**Business Impact:** High — this is a fundamental upgrade driver. Blocking team access makes the Free plan the only usable plan for solo freelancers; agencies need to see value in upgrading.

### Gap 5: No Client Communication Channel
The `ChatCard.tsx` exists as a stub. There is no messaging between agency and client within the portal. All communication happens outside ClientDock.

**Business Impact:** Low-Medium — not critical for v1, but a natural next step.

### Gap 6: Review System Not Surfaced
The `review` table exists in the schema (with rating, title, content, verifiedFlag) but no UI exists to request or display reviews. This could be a differentiating feature — an automated review collection system after project completion.

**Business Impact:** Medium — review collection at project completion is highly valuable for agencies.

---

## Feature Gap Analysis

| Feature | Exists in Code? | Visible to Users? | Business Value |
|---|---|---|---|
| Client invitation email | ❌ NO | No | HIGH — critical for activation |
| Rejection notification to client | ❌ NO | No | HIGH |
| Team members | Schema only | No | HIGH — upgrade driver |
| Invoice generation | Plan config only | Locked | HIGH |
| Meeting scheduling (functional) | Schema only | Broken | MEDIUM |
| Review collection | Schema only | No | MEDIUM |
| White-labeling | Plan config only | Locked | MEDIUM |
| Lead follow-up automation | Plan config only | Locked | MEDIUM |
| Weekly digest emails | Flag in DB | No | LOW |
| Client messaging/chat | Stub UI | No | LOW (v2) |
| Public client portal branding | Not implemented | No | LOW (v2) |

---

## Product Opportunities

### Opportunity 1: Automated Review Collection
When a project is marked complete, automatically email the client asking for a review (1-5 stars + comment). The `review` table already exists. This creates social proof for agencies and a network effect for ClientDock.

### Opportunity 2: Portal White-labeling
Allow Pro/Ultra agencies to replace ClientDock branding with their own. Custom domain, custom logo, custom colors. This is a strong agency upsell — many agencies don't want to show their tool vendor.

### Opportunity 3: Project Templates
Allow agencies to create reusable checklist templates ("Website Design", "Brand Identity", "SEO Audit") that can be applied to new projects in one click. Huge time-saver and a "sticky" feature.

### Opportunity 4: Client Approval Workflow
Currently clients upload files and agencies approve. Expand to: agency creates a design deliverable, shares it in the portal, client must click "Approve" or leave revision comments. This is the "proposal approval" workflow.

### Opportunity 5: Zapier/Make Integration
Expose webhooks or a simple API that allows agencies to connect ClientDock to their existing workflows (CRM, invoice software, Slack).
