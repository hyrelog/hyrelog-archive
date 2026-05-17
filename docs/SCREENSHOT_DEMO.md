# Screenshot demo data (Northwind Systems)

Populates **Northwind Systems** with ~6,000 audit events (configurable), four workspaces, five team members, saved views, and a sample export — for local UI screenshots.

## Prerequisites

- PostgreSQL for **dashboard** and **API** (default region) with migrations applied
- API and dashboard env files configured (`DATABASE_URL`, `HYRELOG_API_URL`, `DASHBOARD_SERVICE_TOKEN` for normal use)

## One-time setup

From **hyrelog-dashboard** (reference plans/regions):

```bash
npm run db:seed
```

From **hyrelog-api/services/api** (events + API company):

```bash
npm run prisma:seed:screenshot
```

Optional: `DEMO_EVENT_COUNT=12000 npm run prisma:seed:screenshot` for heavier charts.

From **hyrelog-dashboard** again (users + company link):

```bash
npm run db:seed:screenshot
```

## Login

| Field | Value |
|-------|--------|
| URL | http://localhost:4000 |
| Email | `demo@northwind.io` |
| Password | `ScreenshotDemo2026!` |

Company admins see **company overview** on home; use **View as** to switch workspaces.

## Re-run

Scripts remove the previous Northwind demo by fixed IDs, then recreate. To append without clearing API data: `DEMO_SKIP_CLEAR=1`.

## Member-only screenshots

Log in as `jordan.lee@northwind.io` (same password) — workspace-scoped home and Explorer.
