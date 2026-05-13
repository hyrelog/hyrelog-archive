# HyreLog — production database full reset (operator runbook)

> **DESTRUCTIVE — READ BEFORE RUNNING**
>
> This runbook **wipes all tenant / customer data** from the production HyreLog databases:
>
> - **API:** all four regional databases (`US`, `EU`, `UK`, `AU`) via `services/api/prisma/seed-reset.ts`.
> - **Dashboard:** the single dashboard database via `prisma/seed/resetToDefault.ts`.
>
> After running, customers, companies, users, API keys, events, auth state, and tenant‑owned rows are gone. Reference catalogs (plans, regions, etc.) are re‑seeded by the scripts. **There is no undo other than restoring from an RDS snapshot.**
>
> **Before you do anything else:**
>
> 1. Take a fresh **manual RDS snapshot** of every database in scope (Dashboard + US + EU + UK + AU). Wait for each snapshot to reach `available` before continuing.
> 2. Confirm you are pointed at the correct AWS account:
>
>    ```bash
>    aws sts get-caller-identity
>    ```
>
> 3. Get explicit sign‑off from whoever owns production data.

**Shell:** Every command is written for **Git Bash** on Windows (also valid in macOS/Linux bash). Do **not** translate these to PowerShell — the same caveats as [`MIGRATIONS_AND_FULL_RESET_RUNBOOK.md`](./MIGRATIONS_AND_FULL_RESET_RUNBOOK.md) apply (npm scripts in `hyrelog-api/package.json` shell out to `powershell.exe`; this runbook deliberately uses the pure‑bash equivalents).

---

## 1. Prerequisites

| Tool | Why |
|------|-----|
| **AWS CLI v2** | `aws ecs run-task`, `describe-services`, `describe-tasks`, `secretsmanager get-secret-value`, `rds`. Must be configured with credentials that can read/write the production account. |
| **Git Bash on Windows** | All command samples assume bash semantics (Unix paths, `source`, `${!var}` indirect expansion). |
| **Node.js (LTS) on PATH** | Required by the ECS helper scripts — they shell out to `node -e` to build the run‑task overrides JSON. Run `node -v` to confirm. |
| **Docker** | **Not required** for the ECS path. Only needed if you also intend to validate locally per `MIGRATIONS_AND_FULL_RESET_RUNBOOK.md` §6. |
| **Repo clones** | `hyrelog-api` (holds the ECS helper scripts), and `hyrelog-dashboard` only if you take Path A. |

Sibling documents you will likely need open:

- [`MIGRATIONS_AND_FULL_RESET_RUNBOOK.md`](./MIGRATIONS_AND_FULL_RESET_RUNBOOK.md) — full migrations + reset surface, including local validation.
- [`database-migrations.md`](./database-migrations.md) — strategy overview, links to per‑service flows.
- [`dashboard-ecs-deploy-and-migrate.md`](./dashboard-ecs-deploy-and-migrate.md) — dashboard ECS / task‑definition context.
- [`smoke-tests.md`](./smoke-tests.md) — post‑reset smoke checks.

---

## 2. Order of operations (recommended, conservative)

Reset scripts assume the target schema is **already current**. Apply migrations first; reset second.

1. **Snapshot** every RDS instance in scope and wait until each is `available`.
2. **Confirm migrations are already applied** to all five DBs (Dashboard + 4× API). If not, run migrations first via the procedures in [`MIGRATIONS_AND_FULL_RESET_RUNBOOK.md`](./MIGRATIONS_AND_FULL_RESET_RUNBOOK.md) §3 (Path A) or §4 (Path B).
3. **Reset the four API regional DBs** (Path A §4 or Path B §5 below).
4. **Reset the Dashboard DB** (Path A §4 or Path B §5 below).
5. **Restart the ECS worker service** (§6) — it shares the API DBs and may hold stale assumptions / queue references.
6. **Verify** (§7).

> The API → dashboard order is conservative but they are independent databases; you can run them in parallel from two terminals if the operator is comfortable. **Always reset → then restart worker**, never the other way around.

---

## 3. Choose a path

You must pick one of the following based on whether your laptop (or a bastion) can reach the production RDS endpoints directly:

- **Path A** — Laptop / bastion has direct TCP 5432 connectivity to every RDS endpoint (VPN, SSM tunnel, port‑forward, etc.). You source the `DATABASE_URL_*` exports and run the reset scripts from your shell.
- **Path B** — Laptop cannot reach RDS (the typical prod situation: RDS is in private subnets with no public IP). You launch one‑off **ECS Fargate tasks** inside the VPC using the helper scripts shipped in `hyrelog-api`. **This is the recommended production path.**

---

## 4. Path A — laptop / bastion can reach RDS

Use this only when `DATABASE_URL_*` resolves to a reachable endpoint from your shell.

### Step A1 — Export `DATABASE_URL_US` … `DATABASE_URL_AU`

Follow [`MIGRATIONS_AND_FULL_RESET_RUNBOOK.md`](./MIGRATIONS_AND_FULL_RESET_RUNBOOK.md) §2 exactly:

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api

# Set RDS_SECRET_ID_*, DB_HOST_*, DB_PORT_*, DB_NAME_* per the source runbook §2.2,
# then source (do not execute) the URL builder so vars stay in your shell:
source scripts/deployment/export-api-phase11-database-urls-from-rds-secrets.sh

# Sanity-check that all four are set:
for R in US EU UK AU; do
  var="DATABASE_URL_${R}"
  if [[ -n "${!var:-}" ]]; then echo "$R OK"; else echo "$R MISSING"; fi
done
```

### Step A2 — Export the dashboard `DATABASE_URL`

Same as `MIGRATIONS_AND_FULL_RESET_RUNBOOK.md` §2.4:

```bash
export DATABASE_URL="$(aws secretsmanager get-secret-value \
  --secret-id 'arn:aws:secretsmanager:ap-southeast-2:ACCOUNT_ID:secret:hyrelog-prod/DATABASE_URL' \
  --region ap-southeast-2 \
  --query SecretString --output text)"
```

> Replace the secret ID/region to match your task definition. Do **not** echo `$DATABASE_URL` into shared logs.

### Step A3 — Reset all four API regions (destructive)

Copy of `MIGRATIONS_AND_FULL_RESET_RUNBOOK.md` §3 Step A2, run from **`services/api`**:

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api/services/api

for R in US EU UK AU; do
  var="DATABASE_URL_${R}"
  export DATABASE_URL="${!var}"
  export SEED_RESET_REGION_LABEL="$R"
  echo "==> Resetting region $R"
  npx tsx prisma/seed-reset.ts
done
```

`SEED_RESET_REGION_LABEL` is what `seed-reset.ts` prints in its banner; setting it per region makes the logs auditable.

### Step A4 — Reset the dashboard database (destructive)

Copy of `MIGRATIONS_AND_FULL_RESET_RUNBOOK.md` §3 Step A4:

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-dashboard

# DATABASE_URL must already be the production dashboard URL (Step A2)
npx tsx prisma/seed/resetToDefault.ts
```

Then jump to **§6 Restart the worker**, then **§7 Verification**.

---

## 5. Path B — ECS Fargate one‑off tasks (private RDS, typical prod)

All commands run from the **`hyrelog-api`** repo root unless noted. The two helper scripts launch one‑off Fargate tasks that reuse the production task definitions (and therefore the production `DATABASE_URL` / `DATABASE_URL_US..AU` secrets).

### Step B1 — Set ECS networking env

These four exports are required by both scripts:

```bash
export PRIMARY_REGION=ap-southeast-2     # Region where the ECS cluster runs
export ECS_CLUSTER=hyrelog-prod-ecs       # Your prod cluster name
export ECS_SUBNET_IDS='subnet-aaa,subnet-bbb'   # comma-separated, no spaces
export ECS_SECURITY_GROUP_IDS='sg-ccc'           # comma-separated
```

To copy `ECS_SUBNET_IDS` and `ECS_SECURITY_GROUP_IDS` from the live services (recommended), follow [`MIGRATIONS_AND_FULL_RESET_RUNBOOK.md`](./MIGRATIONS_AND_FULL_RESET_RUNBOOK.md) §4.0. The short CLI form:

```bash
export ECS_SERVICE_LOOKUP='hyrelog-api'    # use 'hyrelog-dashboard' for the dashboard reset

export ECS_SUBNET_IDS="$(aws ecs describe-services \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE_LOOKUP" \
  --region "$PRIMARY_REGION" \
  --query 'services[0].networkConfiguration.awsvpcConfiguration.subnets[*]' \
  --output text | tr '\t' ',')"

export ECS_SECURITY_GROUP_IDS="$(aws ecs describe-services \
  --cluster "$ECS_CLUSTER" \
  --services "$ECS_SERVICE_LOOKUP" \
  --region "$PRIMARY_REGION" \
  --query 'services[0].networkConfiguration.awsvpcConfiguration.securityGroups[*]' \
  --output text | tr '\t' ',')"
```

If API and dashboard share networking, you only need to do this once. Otherwise rerun with `ECS_SERVICE_LOOKUP='hyrelog-dashboard'` before the dashboard step.

### Step B2 — Reset all four API regions via ECS

Script: `scripts/deployment/run-api-regional-reset-ecs-task.sh`

| Env var | Required? | Notes |
|---------|-----------|-------|
| `PRIMARY_REGION` | yes | ECS cluster region. |
| `ECS_CLUSTER` | yes | ECS cluster name. |
| `ECS_SUBNET_IDS` | yes | Comma‑separated private subnet IDs. |
| `ECS_SECURITY_GROUP_IDS` | yes | Comma‑separated SG IDs that can egress to all four RDS instances. |
| `TASK_DEFINITION` | no | Default `hyrelog-api` (family). Use `hyrelog-api:REV` to pin. |
| `CONTAINER_NAME` | no | Default `hyrelog-api`. Must match the container in the task definition. |

The task uses the `hyrelog-api` task definition’s **existing** secrets for `DATABASE_URL_US`, `DATABASE_URL_EU`, `DATABASE_URL_UK`, `DATABASE_URL_AU` — you do **not** need to source `export-api-phase11-database-urls-from-rds-secrets.sh` for this path.

Run:

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
bash scripts/deployment/run-api-regional-reset-ecs-task.sh
```

The container override runs (inside `/app/services/api`):

```text
for region in US EU UK AU:
  DATABASE_URL=$DATABASE_URL_<region>
  SEED_RESET_REGION_LABEL=<region>
  npx --yes tsx prisma/seed-reset.ts
```

The script waits for `tasks-stopped`, checks `containers[0].exitCode`, and fails loudly if it is not `0`.

**CloudWatch:** stream is under log group **`/ecs/hyrelog-api`**. Find the stream whose name contains the task ID from the script output (`task <ARN>`). Expect four `==> Seed reset region …` banners followed by `All four API regions reset successfully.`

### Step B3 — Reset the dashboard DB via ECS

Script: `scripts/deployment/run-dashboard-reset-ecs-task.sh`

| Env var | Required? | Notes |
|---------|-----------|-------|
| `PRIMARY_REGION` | yes | ECS cluster region. |
| `ECS_CLUSTER` | yes | ECS cluster name. |
| `ECS_SUBNET_IDS` | yes | Use the dashboard service’s subnets if they differ from API. |
| `ECS_SECURITY_GROUP_IDS` | yes | Use the dashboard service’s SGs if they differ from API. |
| `ECS_SERVICE_DASHBOARD` | no | Default `hyrelog-dashboard`. Used to resolve `TASK_DEFINITION` from the live service so the one‑off task runs the same image revision as prod. |
| `TASK_DEFINITION` | no | If unset or family‑only, the script calls `describe-services` against `ECS_SERVICE_DASHBOARD` and pins to whatever revision is live. |
| `CONTAINER_NAME` | no | Default `hyrelog-dashboard`. Must match the container in the task definition. |

Run:

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
bash scripts/deployment/run-dashboard-reset-ecs-task.sh
```

The container override runs (inside `/app`):

```text
export NODE_PATH=/opt/prisma-cli/node_modules
export NODE_EXTRA_CA_CERTS=/etc/ssl/certs/aws-rds-global-bundle.pem
/opt/prisma-cli/node_modules/.bin/tsx prisma/seed/resetToDefault.ts
```

The script waits for `tasks-stopped`, checks the container exit code, and fails loudly if it is not `0`.

**CloudWatch:** stream is under log group **`/ecs/hyrelog-dashboard`**. Expect `==> Dashboard database: resetToDefault (tsx)` followed by `Dashboard reset complete.` and a successful script summary.

---

## 6. After API reset — restart the ECS worker service

The worker shares the four regional API databases and may hold queued jobs that reference now‑deleted IDs. Force a new deployment so it picks up a clean state.

The worker cluster/service names live in your ECS / GitHub Actions secrets (e.g. `ECS_CLUSTER`, `ECS_SERVICE_WORKER`). Use the same names your CI/CD already uses.

### AWS Console

1. Open **Amazon ECS → Clusters → `<your-prod-cluster>`** (e.g. `hyrelog-prod-ecs`).
2. Open **Services** → select the worker service (e.g. `hyrelog-worker`).
3. Click **Update service**.
4. Tick **Force new deployment**.
5. Leave the task definition / desired count unchanged.
6. **Update** and wait for the new deployment to reach **`PRIMARY`** + **steady state** (rolling deployment circle goes back to green).

### AWS CLI

```bash
aws ecs update-service \
  --region "$PRIMARY_REGION" \
  --cluster "$ECS_CLUSTER" \
  --service hyrelog-worker \
  --force-new-deployment \
  --no-cli-pager
```

Wait for steady state:

```bash
aws ecs wait services-stable \
  --region "$PRIMARY_REGION" \
  --cluster "$ECS_CLUSTER" \
  --services hyrelog-worker
```

> Replace `hyrelog-worker` with the actual service name from your ECS / GitHub Actions secrets.

---

## 7. Verification

1. Tail CloudWatch logs for both reset tasks (`/ecs/hyrelog-api`, `/ecs/hyrelog-dashboard`) and confirm the success banners shown in §5.
2. Confirm the worker service is back in **steady state** (`aws ecs describe-services`, `deployments[0].rolloutState == COMPLETED`).
3. Run the smoke‑test suite documented in [`smoke-tests.md`](./smoke-tests.md). At a minimum:
   - Dashboard login / signup against `hyrelog-prod` works and lands on an empty workspace state.
   - API `/healthz` (or equivalent) returns 200 for every regional endpoint.
   - Worker logs show new queue polls without per‑tenant errors.
4. Spot‑check one regional DB via your usual read path (e.g. DBeaver over an SSM tunnel per [`DBEAVER_SSM_BASTION_TUNNEL_RUNBOOK.md`](./DBEAVER_SSM_BASTION_TUNNEL_RUNBOOK.md)): tenant tables empty, reference tables (plans, regions) populated.

---

## 8. Troubleshooting

- **`No migration found in prisma/migrations` then `No pending migrations to apply`** during a related migrate step — the API image was built with an empty `prisma/migrations` directory. See [`MIGRATIONS_AND_FULL_RESET_RUNBOOK.md`](./MIGRATIONS_AND_FULL_RESET_RUNBOOK.md) **§7.1** for the root cause and fix (commit `services/api/prisma/migrations/**`, rebuild + push the `hyrelog-api` image, rerun).
- **ECS task fails / non‑zero exit code** — both helper scripts print the failing task ARN and exit with code 1. Inspect:
  1. `aws ecs describe-tasks --cluster "$ECS_CLUSTER" --tasks <ARN> --region "$PRIMARY_REGION"` and check `containers[0].exitCode` plus `stopCode` / `stoppedReason`.
  2. CloudWatch log group **`/ecs/hyrelog-api`** (API reset) or **`/ecs/hyrelog-dashboard`** (dashboard reset). Find the stream whose name contains the task ID. Common causes: missing/expired RDS Secrets Manager permissions on the task role, schema not migrated (reset expects current tables), or a security group that cannot reach one of the regional RDS endpoints.
- **`Expected tsx at /opt/prisma-cli/node_modules/.bin/tsx`** in the dashboard reset task — the running dashboard image was built without the `/opt/prisma-cli` overlay. Rebuild and push `hyrelog-dashboard`, redeploy the service, then rerun `run-dashboard-reset-ecs-task.sh`.

---

## 9. Script reference (paths relative to a `hyrelog-api` clone)

| Path | Purpose |
|------|---------|
| `scripts/deployment/export-api-phase11-database-urls-from-rds-secrets.sh` | Path A: build `DATABASE_URL_US..AU` from RDS Secrets Manager. **Source**, do not execute. |
| `scripts/deployment/run-api-regional-reset-ecs-task.sh` | Path B: one‑off ECS Fargate task that runs `npx tsx prisma/seed-reset.ts` for US/EU/UK/AU using the `hyrelog-api` task definition secrets. |
| `scripts/deployment/run-dashboard-reset-ecs-task.sh` | Path B: one‑off ECS Fargate task that runs `tsx prisma/seed/resetToDefault.ts` against the dashboard DB using the `hyrelog-dashboard` task definition secrets. |
| `scripts/deployment/run-api-regional-migrations.sh` | (Reference) Path A regional migrations loop. |
| `scripts/deployment/run-api-regional-migrations-ecs-task.sh` | (Reference) Path B regional migrations via ECS. |
| `scripts/deployment/run-dashboard-migrations-ecs-task.sh` | (Reference) Path B dashboard migrations via ECS. |
| `scripts/reset-all-regions.sh` | Local‑only catalog reseed helper (see `MIGRATIONS_AND_FULL_RESET_RUNBOOK.md` §6.2). Not for production. |

Application‑level reset entry points (run by the scripts above):

| Path | Repo |
|------|------|
| `services/api/prisma/seed-reset.ts` | `hyrelog-api` |
| `prisma/seed/resetToDefault.ts` | `hyrelog-dashboard` |
