# HyreLog sandbox mode (design)

Goal: let prospects and developers **try the product** without touching production data, billing, or real API keys.

## What exists today (local)

| Piece | Purpose |
|-------|---------|
| `npm run db:seed` + `prisma:seed:screenshot` + `db:seed:screenshot` | **Northwind Systems** demo company, ~6k events, fixed logins |
| `demo@northwind.io` / `ScreenshotDemo2026!` | Company admin demo user |
| `jordan.lee@northwind.io` | Workspace member demo user |

See [SCREENSHOT_DEMO.md](./SCREENSHOT_DEMO.md).

This is ideal for **local screenshots** and dev UX review, not public multi-tenant trials.

## Recommended tiers

### Tier 1 — Hosted read-only demo (fastest product win)

- **URL:** `https://demo.app.hyrelog.com` (or `app.hyrelog.com/demo` with routing).
- **Data:** Dedicated **sandbox** stack or single Northwind tenant in a **sandbox AWS account** (preferred) or isolated DB/schema in prod.
- **Auth:** Shared demo login (rotating password in Secrets Manager) or magic link with rate limit; no self-signup.
- **Guards:**
  - `company.sandbox = true` (or `plan = SANDBOX`) on dashboard + API company rows.
  - Block: billing changes, real webhooks, outbound email to non-allowlisted domains, export download above small cap, API key creation for non-demo keys.
  - Rate-limit ingest and Explorer queries per IP.
- **Ops:** Nightly `seed-screenshot` reset job (ECS task) so the environment always looks fresh.

### Tier 2 — Per-user trial sandbox (signup)

- On register, provision `Company` with `sandbox: true`, single workspace, 7-day TTL.
- Seed **small** event sample (500 events), not 6k.
- Auto-delete company + data after TTL (worker cron).
- Upgrade path: “Convert to production” → new company without sandbox flag, connect real keys.

### Tier 3 — API sandbox keys

- Issue keys prefixed `hl_sandbox_` mapped to sandbox companies only.
- Separate base URL optional: `https://sandbox.api.hyrelog.com` → same API with `SANDBOX_MODE=true` env enforcing:
  - No cross-company access
  - Max events/day
  - No S3 export to customer buckets (pre-signed URLs to ephemeral bucket only)

## Data model sketch

```prisma
// dashboard Company (or metadata JSON)
sandbox Boolean @default(false)
sandboxExpiresAt DateTime?
```

API `Company` mirror field for ingest routing and quotas.

Middleware checks `sandbox` before destructive or costly actions.

## Infrastructure options

| Option | Pros | Cons |
|--------|------|------|
| **Separate AWS account** `hyrelog-sandbox` | Blast radius, free tier experiments | More CI/CD |
| **Same account, separate RDS + ECS services** | Cheaper ops | Must tag/limit IAM carefully |
| **Shared prod RDS, schema per sandbox company** | Cheapest | Risk of query leaks; not recommended |

Recommendation: **separate account** or at minimum **separate RDS** for any public demo.

## Implementation order

1. `sandbox` flag + server-side guards (dashboard actions + API routes).
2. Hosted Northwind + nightly reset (Tier 1).
3. Trial provisioning on signup (Tier 2).
4. Sandbox API keys + docs (Tier 3).

## Local developer “sandbox”

Keep using screenshot seed; document in README:

```bash
# dashboard
npm run db:seed && npm run db:seed:screenshot
# api
npm run prisma:seed:screenshot
```

Optional: `SANDBOX_MODE=true` in `.env` to show a banner (“Demo data — not production”).
