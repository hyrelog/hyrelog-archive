# Stripe Setup Guide for HyreLog

This guide walks through setting up Stripe for HyreLog Dashboard billing: products, prices (monthly and optional yearly), Customer Portal, webhooks, and linking prices to your database. Use **test mode** for local development.

---

## Do I need both monthly and yearly prices?

**No.** For MVP you can use **monthly only**:

- Create one **recurring monthly** price per paid plan (Starter, Growth, Scale, Enterprise).
- Store each price ID in the Dashboard `Plan` table as `stripePriceMonthlyId`.
- The billing UI uses `stripePriceMonthlyId` for the "Upgrade" button.

**Optional: add yearly billing**

- If you want annual plans (often with a discount), create a **recurring yearly** price for each product and set `stripePriceYearlyId` on the Plan.
- The current UI only uses monthly for Checkout; to offer a monthly/yearly choice you would add a toggle or separate buttons that pass either `stripePriceMonthlyId` or `stripePriceYearlyId` to Checkout.
- The webhook already supports both: it matches the subscription’s price ID to either `stripePriceMonthlyId` or `stripePriceYearlyId` to resolve the plan.

**Summary:** Start with monthly prices only. Add yearly later if you want an annual option.

---

## Prerequisites

- Stripe account: [dashboard.stripe.com](https://dashboard.stripe.com)
- Switch to **Test mode** (toggle in the sidebar) for local/dev
- Dashboard running with Stripe env vars (see below)

---

## 1. Create products and prices in Stripe

### 1.1 Open Products

1. In Stripe Dashboard (test mode), go to **Product catalog** (or **Products**).
2. Click **Add product**.

### 1.2 Create one product per paid plan

Create a product for each plan you want to sell (e.g. Starter, Growth, Scale, Enterprise). **Free** is not a Stripe product; it is the default when there is no subscription.

**Example: Starter plan**

1. **Name:** `Starter` (or "HyreLog Starter").
2. **Description:** Optional (e.g. "Up to 50k events/month").
3. **Image:** Optional.
4. Leave **Pricing** for the next step.

### 1.3 Add a monthly price

1. In the same product page, under **Pricing**, click **Add a price**.
2. **Price type:** Recurring.
3. **Amount:** e.g. `29` (USD) or your currency.
4. **Billing period:** Monthly.
5. **Price description:** Optional (e.g. "Starter monthly").
6. Save. Stripe shows the **Price ID** (e.g. `price_1ABC...`). **Copy it**; you will need it for the Dashboard database.

### 1.4 (Optional) Add a yearly price

1. On the same product, click **Add another price**.
2. **Price type:** Recurring.
3. **Amount:** e.g. `290` (USD) for ~2 months free, or your chosen yearly amount.
4. **Billing period:** Yearly.
5. Save and copy the **Price ID** (e.g. `price_1DEF...`) for `stripePriceYearlyId`.

### 1.5 Repeat for other plans

Create a product and at least one monthly price for each of: Starter, Growth, Scale, Enterprise (or whichever plans you use). Note every **monthly** Price ID; optionally note **yearly** Price IDs.

**Example list:**

| Plan     | Product name | Monthly price ID  | Yearly price ID (optional) |
|----------|--------------|-------------------|----------------------------|
| STARTER  | Starter      | price_xxx_starter_m | price_xxx_starter_y     |
| GROWTH   | Growth       | price_xxx_growth_m  | price_xxx_growth_y      |
| SCALE    | Scale        | price_xxx_scale_m   | price_xxx_scale_y       |
| ENTERPRISE | Enterprise | price_xxx_ent_m   | price_xxx_ent_y         |

---

## 2. Customer Portal

1. In Stripe Dashboard go to **Settings** > **Billing** > **Customer portal** (or **Developers** > **Webhooks** area, depending on Stripe layout).
2. Enable the **Customer portal**.
3. Configure as needed:
   - **Allow customers to:** switch plans, cancel, update payment method.
   - **Business information:** company name, support link (e.g. your Help page).
4. Save. The "Manage billing" button in the Dashboard will create a Portal session so customers can manage subscriptions and payment methods.

---

## 3. Webhooks

### 3.1 Add webhook endpoint (production)

1. **Developers** > **Webhooks** > **Add endpoint**.
2. **Endpoint URL:** `https://<your-dashboard-host>/api/stripe/webhook` (e.g. `https://app.hyrelog.com/api/stripe/webhook`).
3. **Events to send:** Select:
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
4. Add endpoint. Copy the **Signing secret** (starts with `whsec_`). Set it as `STRIPE_WEBHOOK_SECRET` in your production Dashboard env.

### 3.2 Local development: Stripe CLI

For local testing, Stripe cannot reach localhost. Use the CLI to forward events:

1. Install [Stripe CLI](https://stripe.com/docs/stripe-cli).
2. Log in: `stripe login`.
3. Forward:

   ```bash
   stripe listen --forward-to localhost:4000/api/stripe/webhook
   ```

4. The CLI prints a **webhook signing secret** (e.g. `whsec_...`). Use this in your **local** `.env` as `STRIPE_WEBHOOK_SECRET`.
5. Restart the Dashboard so it picks up the new secret.
6. Trigger a test event:

   ```bash
   stripe trigger customer.subscription.updated
   ```

   Or complete a test Checkout in the browser; Stripe will send `customer.subscription.created` or `updated` to your endpoint.

---

## 4. Environment variables (Dashboard)

Add to your Dashboard `.env` (or host env):

```bash
# Stripe (server-side only; never expose secret key to client)
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

- **Test mode:** Use keys from Stripe Dashboard with "Test mode" on (`sk_test_...`, `whsec_...` from CLI or test webhook).
- **Production:** Use live keys and the webhook signing secret from the production endpoint.

Optional for redirects and links:

```bash
NEXT_PUBLIC_APP_URL=https://app.hyrelog.com
NEXT_PUBLIC_DOCS_URL=https://docs.hyrelog.com
NEXT_PUBLIC_SUPPORT_EMAIL=support@hyrelog.com
NEXT_PUBLIC_STATUS_URL=https://status.hyrelog.com
```

---

## 5. Link Stripe prices to Dashboard database

The Dashboard `Plan` model has `stripePriceMonthlyId` and `stripePriceYearlyId`. After creating prices in Stripe, set these in your database so the webhook can map subscriptions to plans and the UI can start Checkout with the correct price.

### Option A: SQL

Connect to the Dashboard database and run (replace `price_xxx` with the real IDs from step 1):

```sql
UPDATE plans SET "stripePriceMonthlyId" = 'price_xxx_starter_m' WHERE code = 'STARTER';
UPDATE plans SET "stripePriceMonthlyId" = 'price_xxx_growth_m'  WHERE code = 'GROWTH';
UPDATE plans SET "stripePriceMonthlyId" = 'price_xxx_scale_m'   WHERE code = 'SCALE';
UPDATE plans SET "stripePriceMonthlyId" = 'price_xxx_ent_m'     WHERE code = 'ENTERPRISE';

-- Optional: yearly
UPDATE plans SET "stripePriceYearlyId" = 'price_xxx_starter_y' WHERE code = 'STARTER';
-- ... repeat for GROWTH, SCALE, ENTERPRISE
```

Table name may be `"Plan"` depending on your Prisma schema; adjust if needed.

### Option B: Seed script

If you use a Prisma seed, update the plan records with the Stripe price IDs after creating or finding plans. Example (conceptual):

```ts
await prisma.plan.updateMany({
  where: { code: 'STARTER' },
  data: { stripePriceMonthlyId: process.env.STRIPE_PRICE_STARTER_MONTHLY },
});
```

Run the seed after creating prices in Stripe and setting the env vars.

### Verify

- In the Dashboard, open **Billing > Subscription**. Plans that have `stripePriceMonthlyId` set should show an **Upgrade** button.
- Clicking Upgrade creates a Checkout session with that price and redirects to Stripe.

---

## 6. Checkout metadata (company link)

When creating the Checkout session, the Dashboard passes `metadata.dashboardCompanyId` so that when Stripe sends the webhook, the handler can find the correct company and update its `Subscription` row. Ensure your server action (e.g. `createCheckoutSession`) includes:

```ts
metadata: { dashboardCompanyId: companyId }
```

in the Stripe Checkout session (or equivalent). The webhook reads `event.data.object.metadata.dashboardCompanyId` to look up the company.

---

## 7. Test cards (Stripe test mode)

Use these in Checkout when testing:

| Scenario        | Card number         |
|----------------|---------------------|
| Success        | 4242 4242 4242 4242 |
| Decline        | 4000 0000 0000 0002 |
| Auth required  | 4000 0025 0000 3155 |

Use any future expiry, any CVC, any postal code. More: [Stripe test cards](https://stripe.com/docs/testing#cards).

---

## 8. Flow summary

| User action       | What happens |
|-------------------|--------------|
| Clicks Upgrade    | Dashboard creates Stripe Checkout with chosen price and `metadata.dashboardCompanyId`; user is redirected to Stripe. |
| Completes payment| Stripe redirects back to your success URL; Stripe sends `customer.subscription.created` or `updated` to your webhook. |
| Webhook received | Dashboard verifies signature, finds company by `metadata.dashboardCompanyId`, finds plan by price ID, upserts `Subscription` (status, period, trialEndsAt, etc.). |
| Manage billing   | Dashboard creates a Stripe Customer Portal session; user manages subscription or payment method on Stripe and returns to your app. |

---

## 9. Production checklist

- [ ] Use **live** Stripe keys (`sk_live_...`) and **live** webhook signing secret.
- [ ] Create **live** products and prices (or clone from test); update `stripePriceMonthlyId` / `stripePriceYearlyId` in production DB if price IDs differ.
- [ ] Add **production** webhook endpoint in Stripe (e.g. `https://app.hyrelog.com/api/stripe/webhook`) with events `customer.subscription.created`, `updated`, `deleted`.
- [ ] Set `NEXT_PUBLIC_APP_URL` (or ensure redirect origin) so Checkout and Portal return URLs are correct.
- [ ] Never expose `STRIPE_SECRET_KEY` or `STRIPE_WEBHOOK_SECRET` to the client; use them only in server actions and API routes.

---

## 10. Troubleshooting

| Issue | Check |
|-------|--------|
| Webhook returns 400 | Signature verification failed. Ensure `STRIPE_WEBHOOK_SECRET` matches the endpoint (CLI secret for local, endpoint secret for production). |
| Upgrade does nothing | Stripe not configured or `stripePriceMonthlyId` missing for that plan. Check env and DB. |
| Subscription not updating | Webhook must receive events. For local, run `stripe listen` and use the CLI secret. Check `metadata.dashboardCompanyId` is set on Checkout session. |
| Wrong plan after payment | Price ID in webhook must match a Plan’s `stripePriceMonthlyId` or `stripePriceYearlyId`; otherwise the handler falls back to FREE. |

For more on running locally and production deployment, see [MVP_DELIVERABLES.md](../../hyrelog-api/docs/MVP_DELIVERABLES.md) in the API repo.
