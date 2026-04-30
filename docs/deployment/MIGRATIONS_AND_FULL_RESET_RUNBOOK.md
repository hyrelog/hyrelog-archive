# HyreLog — migrations & full database reset (local test → AWS)

**Shell:** Every command below is written for **[Git Bash](https://git-scm.com/downloads)** on Windows (also valid in macOS/Linux **bash** unless noted). Avoid **PowerShell** blocks for consistency; do not rely on **`npm run prisma:migrate:all`** / **`npm run seed:reset:all`** from this runbook—they call **`powershell.exe`** internally in `hyrelog-api/package.json`.

This document is **the operator runbook**: apply Prisma migrations and (optionally) wipe tenant data across **Dashboard** (1 DB), **API** (4 regional DBs), and understand **worker** behavior.

**Start here for API → RDS only:** [APPLY_API_MIGRATIONS_AWS_FROM_SCRATCH.md](./APPLY_API_MIGRATIONS_AWS_FROM_SCRATCH.md) — numbered steps including building the image with `prisma/migrations`, pushing ECR, and the ECS migration task.

---

## Git Bash specifics (read once)

| Topic | Guidance |
|--------|-----------|
| **Repo paths** | Use Unix-style roots, e.g. `cd /c/Users/you/Dropbox/hyrelog/hyrelog-api`. Quote paths if they contain spaces. |
| **`source`** | RDS URL export script **must** be `source scripts/...sh` or `. scripts/...sh` — not `bash script.sh` — so vars stay in your session. |
| **Line endings** | If a cloned `.sh` has Windows CRLF and fails with `$'\r'`, convert to LF or run `sed -i 's/\r$//' script.sh`. |
| **AWS CLI / Node / Python** | Install **AWS CLI v2**, **Node 20+**, and a real **Python 3** on PATH (`python` or `python3`). If **`python`** opens the Microsoft Store stub, invoke the full path to **python.exe** or fix “App execution aliases” in Windows Settings. |
| **`npx prisma`** | Always run API Prisma commands with **`cwd`** = **`hyrelog-api/services/api`** so `prisma.config.ts` resolves. |

---

**Repos and paths**

| Concern | Repository | Notes |
|--------|-------------|------|
| This runbook (docs only) | `hyrelog-archive` | No scripts here |
| API + orchestration scripts | **`hyrelog-api`** | Migrations live under `services/api/prisma/migrations/` |
| Dashboard app + Prisma | **`hyrelog-dashboard`** | Migrations under `prisma/migrations/` |
| Worker | **`hyrelog-api`** (worker service) | **No separate schema**. Worker uses the same API Prisma client and **the same four regional databases**. |

**Worker rule of thumb:** run **`prisma migrate deploy`** only from **`hyrelog-api/services/api`**. After destructive API DB resets, **redeploy or restart** the worker so it is not holding stale assumptions; it does not need its own migrate step.

---

## 0. Safety checklist (do this first)

1. **Snapshot or backup** every RDS instance you will touch (Dashboard + US + EU + UK + AU API DBs).
2. Confirm you are in the **intended AWS account** (`aws sts get-caller-identity`).
3. **Full reset** (`seed-reset` / `resetToDefault`) **deletes all customers, keys, events, auth, etc.** in that database. Not reversible without restore from backup.
4. Order of operations:
   - **Always:** `migrate deploy` **before** relying on new schema.
   - **Reset:** only when you explicitly want empty tenant data; run **after** migrations if the reset script expects new tables/columns.
5. Prefer **testing the same commands locally** (Docker Postgres) before AWS.

---

## 1. Collect values from ECS task definitions (source of truth)

Work from your **rendered** or checked-in task definitions in **`hyrelog-api`**:

- **API:** `hyrelog-api/infra/ecs/task-definition-api.json`  
  - **Static env:** `DB_HOST_US`, `DB_PORT_US`, `DB_NAME_US` (and `_EU`, `_UK`, `_AU`).
  - **Secrets:** `DB_USER_*` / `DB_PASSWORD_*` point at **RDS-managed Secrets Manager** secrets. For the URL builder you need the **secret id/ARN only** — strip any ECS suffix such as `:username::` or `:password::` from `valueFrom` if present; the script uses `get-secret-value` on the **base** secret and reads JSON `username` + `password`.

- **Dashboard:** `hyrelog-api/infra/ecs/task-definition-dashboard.json`  
  - Usually a single secret **`hyrelog-prod/DATABASE_URL`** (or equivalent) containing the **full** `postgresql://...` URL for the dashboard DB.

Copy these into a private notes file; do not paste live passwords into tickets.

---

## 2. Build `DATABASE_URL_US` … `DATABASE_URL_AU` (from RDS secret ARNs + hosts)

**Script (lives in `hyrelog-api`):**

`hyrelog-api/scripts/deployment/export-api-phase11-database-urls-from-rds-secrets.sh`

It **must be sourced** (not executed), so your current shell receives the exports.

### 2.1 Prerequisites

- **AWS CLI** configured with `secretsmanager:GetSecretValue` in **each** region where the RDS secrets live (typically `us-east-1`, `eu-west-1`, `eu-west-2`, `ap-southeast-2`).
- **Python 3** on PATH (`python` or `python3` — see Git Bash table above).
- Working directory: **`hyrelog-api` repository root** (the path used below).

### 2.2 Set inputs (example pattern)

Replace placeholders with values from your task definition / AWS console.

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api

# Base secret ARNs (no :password:: suffix) — same as DB_PASSWORD_* valueFrom roots
export RDS_SECRET_ID_US='arn:aws:secretsmanager:us-east-1:ACCOUNT_ID:secret:rds!db-...'
export RDS_SECRET_ID_EU='arn:aws:secretsmanager:eu-west-1:ACCOUNT_ID:secret:rds!db-...'
export RDS_SECRET_ID_UK='arn:aws:secretsmanager:eu-west-2:ACCOUNT_ID:secret:rds!db-...'
export RDS_SECRET_ID_AU='arn:aws:secretsmanager:ap-southeast-2:ACCOUNT_ID:secret:rds!db-...'

# Optional: override secret regions (defaults shown)
export RDS_REGION_US=us-east-1
export RDS_REGION_EU=eu-west-1
export RDS_REGION_UK=eu-west-2
export RDS_REGION_AU=ap-southeast-2

# RDS endpoints and DB names from task definition env
export DB_HOST_US='hyrelog-prod-api-us....amazonaws.com'
export DB_HOST_EU='hyrelog-prod-api-eu....amazonaws.com'
export DB_HOST_UK='hyrelog-prod-api-uk....amazonaws.com'
export DB_HOST_AU='hyrelog-prod-api-au....amazonaws.com'

export DB_PORT_US=5432
export DB_PORT_EU=5432
export DB_PORT_UK=5432
export DB_PORT_AU=5432

export DB_NAME_US=hyrelog_us
export DB_NAME_EU=hyrelog_eu
export DB_NAME_UK=hyrelog_uk
export DB_NAME_AU=hyrelog_au
```

### 2.3 Source the script

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
source scripts/deployment/export-api-phase11-database-urls-from-rds-secrets.sh
```

On success it prints confirmation **without echoing URLs**. Verify:

```bash
test -n "$DATABASE_URL_US" && echo "US OK"
```

### 2.4 Dashboard URL (single database)

Typically:

```bash
export DATABASE_URL="$(aws secretsmanager get-secret-value \
  --secret-id 'arn:aws:secretsmanager:ap-southeast-2:ACCOUNT_ID:secret:hyrelog-prod/DATABASE_URL' \
  --region ap-southeast-2 \
  --query SecretString --output text)"
```

Or use the **secret id** form (no ARN) exactly as your task definition lists it, for example **`hyrelog-prod/DATABASE_URL`**.

---

## 3. Path A — from your laptop (VPN / bastion / port-forward to RDS)

Use this when **`DATABASE_URL_*`** reaches RDS from your machine (same as `export` script-built URLs).

Repo roots: **`hyrelog-api`**, **`hyrelog-dashboard`**.

### Step A1 — API: deploy migrations (all four regions)

From **`hyrelog-api`** (after §2 exported URLs):

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
export CONFIRM=YES
bash scripts/deployment/run-api-regional-migrations.sh
```

`run-api-regional-migrations.sh` requires **`CONFIRM=YES`** and **`DATABASE_URL_US`**, **`DATABASE_URL_EU`**, **`DATABASE_URL_UK`**, **`DATABASE_URL_AU`**.

Equivalent manual loop (Git Bash):

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api/services/api
for R in US EU UK AU; do
  var="DATABASE_URL_${R}"
  export DATABASE_URL="${!var}"
  npx prisma migrate deploy
done
```

### Step A2 — API: optional full reset (tenant data wiped; plans/catalog reseeded)

**Destructive.** From `services/api`:

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api/services/api
for R in US EU UK AU; do
  var="DATABASE_URL_${R}"
  export DATABASE_URL="${!var}"
  export SEED_RESET_REGION_LABEL="$R"
  npx tsx prisma/seed-reset.ts
done
```

### Step A3 — Dashboard: deploy migrations

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-dashboard
# DATABASE_URL must be set (§2.4)
npx prisma migrate deploy
```

### Step A4 — Dashboard: optional full reset (reference reseed only)

Uses `resetToDefault.ts` (**wipes users, companies, auth**, re-seeds plans/regions/etc.):

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-dashboard
npx tsx prisma/seed/resetToDefault.ts
```

---

## 4. Path B — one-off ECS Fargate tasks (private RDS; laptop cannot connect)

Orchestration scripts live under **`hyrelog-api/scripts/deployment/`**. You still need ECS networking env:

```bash
export PRIMARY_REGION=ap-southeast-2          # Region where ECS *cluster* runs (same as CLI --region)
export ECS_CLUSTER=hyrelog-prod-ecs
export ECS_SUBNET_IDS='subnet-xxx,subnet-yyy'    # comma-separated
export ECS_SECURITY_GROUP_IDS='sg-zzz'           # comma-separated — may be only one SG
```

### Step 4.0 — Finding `ECS_SUBNET_IDS` and `ECS_SECURITY_GROUP_IDS`

One-off **`run-task`** calls need the **same style of networking as a normal Fargate task** that already talks to RDS: **private subnets** (no public IP), **security group(s)** that allow egress to RDS (and ingress if your SG is stateful/filtered tightly). Easiest rule: **copy the subnets and SGs from the live ECS service** you trust for that workload.

| Script / step | Typical service to copy from |
|---------------|------------------------------|
| `run-api-regional-migrations-ecs-task.sh` | **`hyrelog-api`** (API must reach regional RDS from inside the VPC). |
| `run-dashboard-migrations-ecs-task.sh` / `run-dashboard-reset-ecs-task.sh` | **`hyrelog-dashboard`** (same pattern; may differ from API if your stack uses different SGs). |

If API and dashboard share the same subnets/SGs, you can reuse one copy for both.

#### Option A — AWS Console

1. Open **Amazon ECS** → **Clusters** → select **`hyrelog-prod-ecs`** (or your cluster name).
2. Open **Services** → select **`hyrelog-api`** (or **`hyrelog-dashboard`** for dashboard scripts).
3. Open the **service** detail page (**Configuration and metrics**).
4. Find **VPC and security groups** — look for fields similar to **Subnets** and **Security groups** (exact labels vary slightly by console version).
   - **Subnets:** copy every subnet ID **`subnet-...`** shown (typically **two or more**, one per Availability Zone).
   - **Security groups:** copy every **`sg-...`** assigned to tasks in this service.

5. Paste into **`ECS_SUBNET_IDS`** as **comma-separated, no spaces** (spaces are tolerated by HyreLog scripts; commas are required between IDs):

   ```bash
   export ECS_SUBNET_IDS='subnet-aaaaaaaa,subnet-bbbbbbbb'
   export ECS_SECURITY_GROUP_IDS='sg-xxxxxxxx'
   ```

If you only see subnets at the **Service** definition time and not obviously on the Summary card, pick **Deployments** tab → newest deployment → **Networking** → **VPC and security groups** (wording varies).

Alternative path: select a **recently stopped or running Task** → **Networking** tab → subnets + security groups that **that task used**.

#### Option B — AWS CLI (recommended for repeatable copy‑paste)

Set context (adjust names to prod):

```bash
export ECS_CLUSTER='hyrelog-prod-ecs'
export ECS_SERVICE_LOOKUP='hyrelog-api'
export PRIMARY_REGION='ap-southeast-2'

aws ecs describe-services \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE_LOOKUP" \
  --region "$PRIMARY_REGION" \
  --query 'services[0].networkConfiguration.awsvpcConfiguration' \
  --output json
```

You’ll get JSON roughly:

```json
{
  "subnets": ["subnet-aaa", "subnet-bbb"],
  "securityGroups": ["sg-xxx"],
  "assignPublicIp": "DISABLED"
}
```

`assignPublicIp` should be **`DISABLED`** for private networking (matches the migration scripts).

Build **`ECS_*`** vars from CLI output (**`--output text`** turns subnet/SG arrays into **tab-separated** tokens; **`tr`** makes them comma-separated for the scripts):

```bash
export ECS_SUBNET_IDS="$(aws ecs describe-services \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE_LOOKUP" \
  --region "$PRIMARY_REGION" \
  --query 'services[0].networkConfiguration.awsvpcConfiguration.subnets[*]' \
  --output text | tr '\t' ',')"
```

Security groups similarly:

```bash
export ECS_SECURITY_GROUP_IDS="$(aws ecs describe-services \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE_LOOKUP" \
  --region "$PRIMARY_REGION" \
  --query 'services[0].networkConfiguration.awsvpcConfiguration.securityGroups[*]' \
  --output text | tr '\t' ',')"
```

For dashboard tasks, rerun with **`ECS_SERVICE_LOOKUP='hyrelog-dashboard'`**.

#### Verify before `run-task`

- **Subnet IDs** belong to **private** subnets with a route path to **RDS / NAT** (same as running API tasks — if prod API works, these subnets work).
- **Security group outbound** rules allow **TCP 5432** (or RDS port in use) to the **RDS SG** — again, cloning from the working **`hyrelog-api`** SG is safest.
- **Region mismatch:** **`run-task`** is called with **`PRIMARY_REGION`**; the subnets/SGs **must exist in that VPC in that Region**. ECS cluster **`hyrelog-prod-ecs`** is wherever you deployed it (**often `ap-southeast-2`** in this repo’s task defs); RDS across US/EU/UK/AU uses **VPC peering/TGW** separately — the migration **task stays in/cluster region** while **DATABASE_URL_*** endpoints point cross-region RDS. If your architecture differs (one cluster per Region), align **`PRIMARY_REGION`** and **`ECS_CLUSTER`** with that cluster instead.

---

### Step B1 — API: migrations in VPC (all four regions in one task)

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
bash scripts/deployment/run-api-regional-migrations-ecs-task.sh
```

Uses the **deployed API image**: runs `cd /app/services/api` then `npx prisma@… migrate deploy` four times using **`DATABASE_URL_US` … `DATABASE_URL_AU`** injected exactly like prod.

Watch exit code / CloudWatch **`/ecs/hyrelog-api`** for the migration task stream.

Optional: **`TASK_DEFINITION=hyrelog-api:REV`** if you pin a revision.

### Step B2 — API: optional reset inside VPC

There is **no** first-party wrapper script yet. Practical options:

- Run **Path A** from a host that has RDS connectivity (bastion / SSM port-forward) after **`source scripts/deployment/export-api-phase11-database-urls-from-rds-secrets.sh`**; or  
- Clone the pattern in **`run-api-regional-migrations-ecs-task.sh`**, but replace the embedded command script with calls to **`npx tsx prisma/seed-reset.ts`** (with `DATABASE_URL` set per region).

### Step B3 — Dashboard: migrations in VPC

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
bash scripts/deployment/run-dashboard-migrations-ecs-task.sh
```

Resolves **`hyrelog-dashboard`** task definition from **`ECS_SERVICE_DASHBOARD`** (default `hyrelog-dashboard`). Optional: **`MIGRATION_ACTION=deploy`** (default).

### Step B4 — Dashboard: optional reset in VPC

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
bash scripts/deployment/run-dashboard-reset-ecs-task.sh
```

Runs **`tsx prisma/seed/resetToDefault.ts`** in the dashboard container with **`DATABASE_URL`** from the task secrets.

---

## 5. Worker service

| Action | Command / note |
|--------|----------------|
| Migrations | **None** — same schema/API DBs only. |
| After API migrate | **Deploy new API + worker images** if codegen or deps changed (`prisma generate` is part of Docker build). |
| After API **reset** | Redeploy/restart worker; queues may reference IDs that no longer exist. |

---

## 6. Local validation (recommended before AWS) — Git Bash only

With **Docker Postgres** listening on **`localhost`** ports **54321–54324** (default Compose in `hyrelog-api`), from **`hyrelog-api`** root:

### 6.1 Migrate all four local regions (pure bash — does not invoke PowerShell)

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
for pair in \
  '54321:hyrelog_us' \
  '54322:hyrelog_eu' \
  '54323:hyrelog_uk' \
  '54324:hyrelog_au'
do
  PORT="${pair%%:*}"
  DB="${pair#*:}"
  export DATABASE_URL="postgresql://hyrelog:hyrelog@127.0.0.1:${PORT}/${DB}"
  (cd services/api && npx prisma migrate deploy)
done

npm run prisma:generate
```

### 6.2 Optional — reset API local catalogs only (plans, no tenants)

Prefer the **bash** entry point (calls the same **`seed-reset.ts`** as the npm script):

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
bash scripts/reset-all-regions.sh
```

Optional overrides:

```bash
export DB_HOST=127.0.0.1
export DB_USER=hyrelog
export DB_PASS=hyrelog
bash scripts/reset-all-regions.sh
```

### 6.3 Dashboard local

Still in Git Bash:

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-dashboard
export DATABASE_URL='postgresql://USER:PASSWORD@127.0.0.1:PORT/hyrelog_dashboard'
npx prisma migrate deploy
```

Use **`hyrelog-dashboard`** README for the exact compose port and credentials.

---

## 7. Troubleshooting

### 7.1 ECS / CloudWatch: `No migration found in prisma/migrations` then `No pending migrations to apply`

**Meaning:** The container’s `services/api/prisma/migrations` directory is **empty** (or missing migration folders). `migrate deploy` then has nothing to apply; it does **not** prove your RDS is up to date.

**Common cause (HyreLog API):** `hyrelog-api/services/api/prisma/.gitignore` previously listed **`migrations/`**, so migration SQL was **never committed** and **never copied** into the Docker image (`Dockerfile` does `COPY services/api/prisma …`). **Fix:** keep migrations **versioned in git** (do not gitignore `prisma/migrations`), commit all `prisma/migrations/**` files, rebuild and push the **hyrelog-api** image, then run the ECS migration task again.

**Verify locally before rebuild:**

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api/services/api
ls -la prisma/migrations
```

You should see dated migration directories (e.g. `20260125044238_initial_migration`). If the folder is empty, create migrations from your current schema (**dev DB only**) with `npx prisma migrate dev` per Prisma docs, then commit the generated SQL — do **not** point `migrate dev` at production RDS.

---

## 8. References (same Archive)

| Document | Purpose |
|---------|---------|
| [database-migrations.md](./database-migrations.md) | Strategy overview + links |
| [DBEAVER_SSM_BASTION_TUNNEL_RUNBOOK.md](./DBEAVER_SSM_BASTION_TUNNEL_RUNBOOK.md) | SSM tunnels to RDS |
| [dashboard-ecs-deploy-and-migrate.md](./dashboard-ecs-deploy-and-migrate.md) | Dashboard ECS context |

**Scripts (paths relative to `hyrelog-api` clone)**

```text
scripts/deployment/export-api-phase11-database-urls-from-rds-secrets.sh
scripts/deployment/run-api-regional-migrations.sh
scripts/deployment/run-api-regional-migrations-ecs-task.sh
scripts/deployment/run-dashboard-migrations-ecs-task.sh
scripts/deployment/run-dashboard-reset-ecs-task.sh
scripts/reset-all-regions.sh
```
