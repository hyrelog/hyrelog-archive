# Apply HyreLog API migrations to AWS RDS (from scratch)

**Shell:** [Git Bash](https://git-scm.com/downloads). **Repo:** `hyrelog-api` clone. **Goal:** Ship `prisma/migrations` inside the Docker image, then run `prisma migrate deploy` against **US, EU, UK, AU** RDS from **ECS** (recommended) or from your laptop if RDS is reachable.

---

## Step 0 — Migrations exist on disk and in Git

1. Open `hyrelog-api/services/api/prisma/.gitignore`. It **must not** contain a line **`migrations/`** (that would keep migration SQL out of Git and out of Docker — you would see **`No migration found in prisma/migrations`** in ECS logs).

2. Check for migration folders:

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api/services/api
ls -la prisma/migrations
```

- **If you see** directories named like `20260125120000_some_name/` each containing `migration.sql` → continue at **Step 1**.
- **If the folder is missing or empty** → do **Step 0a** once, then commit.

### Step 0a — First-time: generate migration files locally (never against prod RDS)

Use your **local** Docker Postgres for **US** only (HyreLog default port **54321**):

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
docker compose up -d

cd services/api
export DATABASE_URL='postgresql://hyrelog:hyrelog@127.0.0.1:54321/hyrelog_us?schema=public'
npx prisma migrate dev --name initial
```

That creates `prisma/migrations/.../migration.sql`. Commit **the whole tree**:

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
git add services/api/prisma/migrations
git status   # verify SQL files are staged
git commit -m "chore(api): version Prisma migrations for ECS deploy"
```

> **RDS already has tables but never used Prisma migrate?** You need Prisma **baselining** (mark migrations applied without re-running SQL). See [Prisma: Baseline a database](https://www.prisma.io/docs/guides/migrate/developing-with-prisma-migrate/baselining). Do not run `migrate dev` against prod.

---

## Step 1 — Prove the Docker image contains `prisma/migrations`

From **`hyrelog-api` repo root**:

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
docker build -f services/api/Dockerfile -t hyrelog-api:migrations .

docker run --rm --entrypoint bash hyrelog-api:migrations \
  -c 'ls -la /app/services/api/prisma/migrations'
```

You must see your migration directories. If this listing is empty, **do not push** — go back to Step 0.

---

## Step 2 — Push the image to ECR

Use your real account, region, and repository name (below matches common HyreLog prod naming; replace **TAG** with something unique, e.g. **`20260430-$(git rev-parse --short HEAD)`**).

```bash
export AWS_REGION='ap-southeast-2'
export ECR_REPO='163436765242.dkr.ecr.ap-southeast-2.amazonaws.com/hyrelog-api'
export IMAGE_TAG='20260430-migrations'   # change every deploy

aws ecr get-login-password --region "$AWS_REGION" \
  | docker login --username AWS --password-stdin "${ECR_REPO%/*}"

docker tag hyrelog-api:migrations "${ECR_REPO}:${IMAGE_TAG}"
docker push "${ECR_REPO}:${IMAGE_TAG}"
```

**Register a new ECS task definition revision** that sets the API container **`image`** to **`${ECR_REPO}:${IMAGE_TAG}`** (update `infra/ecs/task-definition-api.json`, run your render/register script, or use the AWS Console). You need either:

- **`hyrelog-api:REV`** revision number printed after `register-task-definition`, or  
- an updated **`hyrelog-api`** **service** that already uses this image revision.

One-off migrate tasks pin **`TASK_DEFINITION=hyrelog-api:REV`** so they run **the image that contains your migrations**.

---

## Step 3 — Build `DATABASE_URL_US` … `DATABASE_URL_AU` for logs / optional laptop path

The ECS migration task normally **does not need** these in your laptop shell — the task definition injects **`DB_*` + RDS secrets**, and **`docker-entrypoint-database-url.sh`** builds **`DATABASE_URL_*`** inside the container.

If you **do** migrate from Git Bash against RDS (**VPN/tunnel/bastion**), build URLs by **sourcing** the helper (**repo root `hyrelog-api`**):

1. Copy **`DB_PASSWORD_*`** `valueFrom` from **`infra/ecs/task-definition-api.json`**.  
2. Strip the **`:username::`** / **`:password::`** suffix — keep only the **secret ARN through the random suffix**, e.g.  
   `arn:aws:secretsmanager:us-east-1:163436765242:secret:rds!db-f85d5f08-...-YxJ40o`

3. Set variables and source (adjust hosts/names/secrets if yours differ):

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api

export RDS_SECRET_ID_US='arn:aws:secretsmanager:us-east-1:163436765242:secret:rds!db-...'
export RDS_SECRET_ID_EU='arn:aws:secretsmanager:eu-west-1:163436765242:secret:rds!db-...'
export RDS_SECRET_ID_UK='arn:aws:secretsmanager:eu-west-2:163436765242:secret:rds!db-...'
export RDS_SECRET_ID_AU='arn:aws:secretsmanager:ap-southeast-2:163436765242:secret:rds!db-...'

export RDS_REGION_US=us-east-1
export RDS_REGION_EU=eu-west-1
export RDS_REGION_UK=eu-west-2
export RDS_REGION_AU=ap-southeast-2

export DB_HOST_US='hyrelog-prod-api-us.cs7ic6mo2af4.us-east-1.rds.amazonaws.com'
export DB_HOST_EU='hyrelog-prod-api-eu.cdgc22yu8v5w.eu-west-1.rds.amazonaws.com'
export DB_HOST_UK='hyrelog-prod-api-uk.ctsyimasaugv.eu-west-2.rds.amazonaws.com'
export DB_HOST_AU='hyrelog-prod-api-au.c9umosqssoce.ap-southeast-2.rds.amazonaws.com'

export DB_PORT_US=5432
export DB_PORT_EU=5432
export DB_PORT_UK=5432
export DB_PORT_AU=5432

export DB_NAME_US=hyrelog_us
export DB_NAME_EU=hyrelog_eu
export DB_NAME_UK=hyrelog_uk
export DB_NAME_AU=hyrelog_au

source scripts/deployment/export-api-phase11-database-urls-from-rds-secrets.sh
test -n "$DATABASE_URL_US" && echo "URLs exported OK"
```

---

## Step 4 — Deploy migrations inside the VPC (**recommended**: ECS one-off task)

Pick ** subnets and security groups** from the **`hyrelog-api`** ECS service (**same RDS access as prod**). Quick CLI copy-paste (**Git Bash**):

```bash
export PRIMARY_REGION='ap-southeast-2'
export ECS_CLUSTER='hyrelog-prod-ecs'
export ECS_SERVICE_LOOKUP='hyrelog-api'

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

Pin the **same task-definition revision** that uses the **`IMAGE_TAG`** you pushed:

```bash
export TASK_DEFINITION='hyrelog-api:NN'   # replace NN — e.g. from aws ecs describe-services taskDefinition field
```

Run the migration runner (**hyrelog-api repo root**):

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
bash scripts/deployment/run-api-regional-migrations-ecs-task.sh
```

Wait until the task exits **0**.

### Confirm in CloudWatch

Open log group **`/ecs/hyrelog-api`**. You want:

- **`Loaded Prisma config`** / **`Applying migration`** for each migration, **or**
- **`No pending migrations to apply`** **only after** **`prisma/migrations`** is clearly non-empty in the image (**Step 1**).

You **do not** want **`No migration found in prisma/migrations`** — that means the image still has no migration files.

---

## Step 5 — Optional: deploy from laptop (RDS reachable only)

Only if **`DATABASE_URL_US` … `AU`** work from your machine (tunnel/VPN/private IP):

```bash
cd /c/Users/you/Dropbox/hyrelog/hyrelog-api
export CONFIRM=YES
bash scripts/deployment/run-api-regional-migrations.sh
```

`run-api-regional-migrations.sh` requires **`DATABASE_URL_*`** exported (§3).

---

## Step 6 — Roll API / worker ECS services forward (routine deploy)

After migrations succeed on **all four** DBs (or at least once per region you added):

- Update **`hyrelog-api`** (**and **`hyrelog-worker`** if needed) ECS services to the **same** image/tag you tested, **or** keep `:latest` if that is where you pushed.

Routine script (if your team uses it): `hyrelog-api/scripts/deployment/update-ecs-services.sh`.

---

## Quick checklist

| Check | Meaning |
|------|--------|
| `services/api/prisma/migrations/**/*.sql` in Git | Ships in Docker **`COPY prisma`** layer |
| `docker run ... ls …/prisma/migrations` | Non-empty **before** ECR push |
| ECS migrate task logs | **`Applying migration`** or **no pending** with migrations present |
| Four regions | **US, EU, UK, AU** each get **`migrate deploy`** once per history |

---

## See also

- [MIGRATIONS_AND_FULL_RESET_RUNBOOK.md](./MIGRATIONS_AND_FULL_RESET_RUNBOOK.md) — full reset / dashboard / worker context  
- [database-migrations.md](./database-migrations.md) — strategy overview  
