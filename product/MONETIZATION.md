# Monetization Strategy — ClientDock

---

## Current Plan Structure

As defined in `src/lib/plan-config.ts`:

| | Free | Pro | Ultra | Custom |
|---|---|---|---|---|
| **Price** | $0 | $29/mo | $79/mo | Custom |
| **Launch Offer** | — | $19/mo | $49/mo | — |
| **Max Clients** | 5 | 25 | Unlimited | Unlimited |
| **Max Active Projects** | 10 | 60 | Unlimited | Unlimited |
| **Max Team Members** | 1 | 5 | Unlimited | Unlimited |
| **Automated Reminders** | ❌ | ✅ | ✅ | ✅ |
| **Full Analytics** | ❌ | ✅ | ✅ | ✅ |
| **Priority Support** | ❌ | ✅ | ✅ | ✅ |
| **White Labeling** | ❌ | ❌ | ✅ | ✅ |
| **Custom Branding** | ❌ | ❌ | ✅ | ✅ |
| **Invoice Generation** | ❌ | ✅ | ✅ | ✅ |
| **Lead Follow-Up** | ❌ | ✅ | ✅ | ✅ |

---

## Revenue Processor: Razorpay

**Problem for US Market:**
Razorpay is a leading Indian payment processor but has very limited reach in the US:
- Requires Indian business entity to accept Indian payments
- Cannot process US credit cards directly
- Primary currency is INR, not USD

**For US-focused SaaS, this is a fundamental business problem.**

### Recommendation: Add Stripe

For the US market (and most international markets), Stripe is the de facto standard:
- Trusted by US customers
- Accepts all major cards, Apple Pay, Google Pay
- Strong subscription billing capabilities
- Better developer experience
- Stripe Billing portal included

**Implementation approach:**
- Add `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_PRO`, `STRIPE_PRICE_ULTRA` to env
- Add `stripeCustomerId` to agency table
- Create `src/lib/stripe.ts` (mirrors `src/lib/razorpay.ts` pattern)
- Add Stripe webhook handler at `/api/stripe/webhook`
- Feature-flag: show Stripe for US/international users, Razorpay for Indian users

---

## Conversion Strategy Assessment

### Freemium Model Strengths
- 5 clients free → gives real value to freelancers
- Portfolio can be built before paying
- Automated reminders (Pro only) is a strong upgrade incentive — when clients start missing files, agencies feel the pain

### Freemium Model Weaknesses
- **Team members gate** — currently non-functional, but once built, this is the #1 upgrade driver for agencies
- **Analytics** — plan-gated, but basic analytics should be visible to free users to show the value
- **Invoice generation** — not built yet; when built, this is a strong Pro incentive

### Missing Upgrade Triggers
1. **Usage warnings** — no "You're at 4/5 clients" notification
2. **Soft limits** — hitting the client limit should show an upgrade CTA, not just a generic error
3. **Feature previews** — locked features should be visible but blurred ("what you're missing")
4. **In-app upgrade flow** — the billing page exists but upgrade is via button only; no contextual upgrade prompts throughout the app

---

## Pricing Analysis

### Current Pricing vs Market

| Product | Starter/Pro Price | Clients/Projects |
|---|---|---|
| **ClientDock Pro** | $29/mo | 25 clients, 60 projects |
| **HoneyBook** | $19/mo | Unlimited |
| **Dubsado** | $20/mo | Unlimited |
| **Notion** | $16/mo | Unlimited pages |
| **Copilot** | $29/mo | 15 clients |

**Verdict:** ClientDock's $29/mo Pro is priced at the top end of the market, but offers fewer clients than competitors. The value proposition must be clearly differentiated (automated reminders, approval workflow) to justify this.

**The $19/mo launch offer is smart** — attracts early adopters at a competitive price.

### Upgrade Path Clarity
The Free → Pro → Ultra path is clear in concept, but the upgrade CTAs need to be more contextual. When a user hits a limit, the error message should include the upgrade link.

---

## Billing System Issues (Current)

1. **Razorpay plans not documented** — The `.env.example` shows `RAZORPAY_PLAN_PRO` and `RAZORPAY_PLAN_ULTRA` must be created manually in the Razorpay dashboard. No script automates this.

2. **Subscription cancellation flow is broken** — Agency cancels → plan not downgraded (BUG-004). Revenue recovery fails.

3. **No dunning management** — When a payment fails, the agency is not notified and no retry is attempted from the app side (relies entirely on Razorpay's retry logic).

4. **No grace period** — When a payment fails, immediate feature loss could cause churn. Consider a 7-day grace period.

5. **Invoice download** — Payment records are stored but there's no invoice PDF generation for agency tax records.

---

## LTV and Churn Considerations

### High-Risk Churn Scenarios

1. **Client portal link expires** (30 days) → client can't access → agency frustrated → churns
2. **Agency hits project limit** → sees unhelpful error → doesn't understand they need to upgrade → churns
3. **Meetings feature appears broken** → agency loses trust in product quality → churns
4. **Revenue chart shows random data** → agency doesn't trust the tool → churns

### Retention Features to Prioritize

1. **Weekly digest email** — `agency.weeklyDigest` flag exists but is not implemented. Weekly summary of pending items, overdue projects, win rate. Keeps agency engaged even when not logging in daily.
2. **Project completion celebration** — When 100% of checklist items are approved, show a celebration and prompt for a client review.
3. **Activity timeline** — Show agency a "this week in ClientDock" summary.
