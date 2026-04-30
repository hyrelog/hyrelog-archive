# Postman Collection for HyreLog API

This directory contains Postman collections and environments for testing the HyreLog API (V1 customer API and Dashboard contract API).

## Files

- `HyreLog API.postman_collection.json` - Full collection (Root, Internal, **Dashboard**, V1 API)
- `HyreLog Local.postman_environment.json` - Local development (`base_url` http://localhost:3000)
- `HyreLog Production.postman_environment.json` - Production (`base_url` https://api.hyrelog.com) — **same variables as Local; fill secrets/IDs yourself**
- `ENVIRONMENT_SETUP.md` - Detailed guide on where to get each environment variable value

## Setup Instructions

### 1. Import into Postman

1. Open Postman
2. Click **Import** and select:
   - `HyreLog API.postman_collection.json`
   - `HyreLog Local.postman_environment.json` and/or **`HyreLog Production.postman_environment.json`** (same variable names)
3. Select **HyreLog Local** or **HyreLog Production** from the environment dropdown (top right). Use **Production** to hit [`https://api.hyrelog.com`](https://api.hyrelog.com/openapi.json); use Local for `npm run dev` on port 3000.

### 2. Seed Test Data (for V1 API)

From the API repo root:

```bash
npm run seed
```

Copy from output: company key, workspace key, company ID, workspace ID, project ID, plan tier.

### 3. Dashboard Token (for Dashboard folder)

For **Local:** set `dashboard_token` from **DASHBOARD_SERVICE_TOKEN** in the API `.env` (repo root).  
For **Production:** set `dashboard_token` from the deployed **`DASHBOARD_SERVICE_TOKEN`** (e.g. ECS task secrets). All `/dashboard/*` requests require this header.

Set `internal_token` for **`/internal/*`** the same way: local `.env` vs production **`INTERNAL_TOKEN`** (ECS mirrors `INTERNAL_TOKEN`; see Secrets Manager hyrelog-prod/INTERNAL_TOKEN).

### 4. Configure Environment Variables

**V1 API (from seed):** `company_key`, `workspace_key`, `company_id`, `workspace_id`, `project_id`, `plan_tier`

**Dashboard:** `dashboard_token` (required). For provisioning and company-scoped routes: `dashboard_company_id`, `dashboard_workspace_id`, `api_company_id` (from Provision Company response), `dashboard_key_id`, `archive_id`, `restore_request_id`, `plan_id` as needed.

See **ENVIRONMENT_SETUP.md** for details.

## Available Endpoints

### Root
- `GET /` - No auth

### Internal (`x-internal-token`)
- `GET /internal/health`
- `GET /internal/metrics`

### Dashboard (`x-dashboard-token`)

All Dashboard requests require `x-dashboard-token`. Company-scoped routes also need `x-company-id` (use `api_company_id` or `dashboard_company_id`).

- **Companies:** POST /dashboard/companies (provision), GET /dashboard/companies/:dashboardCompanyId (reconcile)
- **Workspaces:** POST /dashboard/workspaces, GET /dashboard/workspaces/:id, POST .../archive, POST .../restore
- **API Keys:** POST /dashboard/api-keys (sync), POST /dashboard/api-keys/:dashboardKeyId/revoke
- **Company-scoped:** GET /dashboard/company, GET /dashboard/events, POST/GET /dashboard/exports, GET /dashboard/webhooks
- **Restore:** POST/GET/GET :id/DELETE /dashboard/restore-requests
- **Admin (HYRELOG_ADMIN):** GET /dashboard/admin/companies, GET /dashboard/admin/plans, POST .../plan, GET/POST restore approve/reject, GET /dashboard/admin/audit-logs

### V1 API (Bearer API key)

- **Events:** POST /v1/events (ingest), GET /v1/events (query)
- **Keys:** GET /v1/keys/status. Create/Rotate return 403 (dashboard-only).
- **Webhooks:** POST/GET /v1/workspaces/:id/webhooks, POST /v1/webhooks/:id/disable|enable, GET .../deliveries
- **Exports:** POST /v1/exports, GET /v1/exports/:jobId, GET /v1/exports/:jobId/download

## Error Format

```json
{ "error": "Message", "code": "ERROR_CODE" }
```

Common codes: `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `VALIDATION_ERROR`, `REGION_MISMATCH`, `SCOPE_MISMATCH`, `WORKSPACE_ARCHIVED`, `RATE_LIMITED`

## Updating the Collection

- ✅ Root, Internal, V1 API (Events, Keys, Webhooks, Exports)
- ✅ Dashboard (Companies, Workspaces, API Keys, Company-scoped, Restore, Admin)
- Key creation/rotation: 403 dashboard-only; use Dashboard folder for sync/revoke
