# Production DB password recovery (rotation off → one manual rotate → deploy)

Use this after RDS password rotation broke the API (Event Explorer: **Failed to load events**) or login issues on the dashboard.

**Shell:** **Git Bash** on Windows (same as [dashboard-ecs-deploy-and-migrate.md](./dashboard-ecs-deploy-and-migrate.md)).

**Copy/paste (Git Bash):**

| Shell | Line continuation | Wrong in Git Bash |
|-------|-------------------|-------------------|
| **Git Bash** | none — use **one line**, or paste a whole `<<'EOF'` block | trailing `\` at end of line |
| PowerShell | backtick `` ` `` at end of line | **never use `` ` `` in Git Bash** — bash treats it as command substitution |

If you see `bash: --region: command not found`, you used PowerShell-style `` ` `` breaks. Use the **single-line** command in each section instead.

**Root causes addressed:**

1. **API/worker** — entrypoint must URL-encode passwords that start with `-` (node `--` fix in docker-entrypoint-database-url.sh).
2. **Dashboard** — hyrelog-prod/DATABASE_URL must be rebuilt from the dashboard RDS secret after each dashboard DB rotation.
3. **ECS** — API tasks that fail startup roll back and leave an old task with a stale password.

**Account / region defaults:** 163436765242, primary ECS region ap-southeast-2.

---

## Inventory (5 rotating databases)

| DB instance | AWS region | RDS managed secret ARN |
|-------------|------------|-------------------------|
| hyrelog-prod-api-us | us-east-1 | arn:aws:secretsmanager:us-east-1:163436765242:secret:rds!db-f85d5f08-a9c5-4f04-9b1b-5ebe82af004b-YxJ40o |
| hyrelog-prod-api-eu | eu-west-1 | arn:aws:secretsmanager:eu-west-1:163436765242:secret:rds!db-c2457e98-ee04-43df-88a0-2ba6a51bcd4b-pHNiwD |
| hyrelog-prod-api-uk | eu-west-2 | arn:aws:secretsmanager:eu-west-2:163436765242:secret:rds!db-84a7c030-16be-4378-99fb-2b6380614c48-N2CU3U |
| hyrelog-prod-api-au | ap-southeast-2 | arn:aws:secretsmanager:ap-southeast-2:163436765242:secret:rds!db-62ea06d6-f3e2-4c18-8131-94b8907d3ebe-5MN4ox |
| hyrelog-prod-dashboard | ap-southeast-2 | arn:aws:secretsmanager:ap-southeast-2:163436765242:secret:rds!db-8790c16b-e0de-4e58-b2d6-200c1221aa86-6PHmH7 |

---

## Phase 0 — Session bootstrap (run first every time)

Paste this whole block into Git Bash:

```bash
export PRIMARY_REGION=ap-southeast-2
export AWS_DEFAULT_REGION=ap-southeast-2
export AWS_ACCOUNT_ID=163436765242
export ECS_CLUSTER=hyrelog-prod-ecs
export PROJECT_PREFIX=hyrelog-prod
export MSYS_NO_PATHCONV=1
export API_REPO_DIR=/c/Users/kram/Dropbox/hyrelog/hyrelog-api
export ECR_API="${AWS_ACCOUNT_ID}.dkr.ecr.${PRIMARY_REGION}.amazonaws.com/hyrelog-api"
export ECR_WORKER="${AWS_ACCOUNT_ID}.dkr.ecr.${PRIMARY_REGION}.amazonaws.com/hyrelog-worker"
aws sts get-caller-identity
docker --version
git --version
```

Expect account 163436765242. Docker Desktop must be running.

Confirm the entrypoint fix is in your build branch:

```bash
grep -n 'node -e.*--' "${API_REPO_DIR}/scripts/docker/docker-entrypoint-database-url.sh"
```

You should see `--` before the password argument in urlencode().

---

## Phase 1 — Stop automatic rotation (do this first)

### 1A) Disable the EventBridge auto-redeploy rule (optional but recommended)

```bash
aws events disable-rule --region "${PRIMARY_REGION}" --name hyrelog-prod-rds-rotation-success
```

### 1B) Disable automatic rotation on each RDS managed secret

RDS-managed secrets (rds!db-…) are disabled in **Secrets Manager per region**, not on the RDS Modify screen.

For each of the 5 rows in the inventory table:

1. AWS Console → Secrets Manager (correct region).
2. Open the secret whose name starts with rds!db-….
3. Rotation configuration → **Disable automatic rotation** → Save.

### 1C) Verify rotation is off

Paste this whole block (one heredoc — safe to copy as one unit):

```bash
while IFS='|' read -r region secret_arn; do
  enabled=$(aws secretsmanager describe-secret --region "${region}" --secret-id "${secret_arn}" --query RotationEnabled --output text)
  echo "${region}: RotationEnabled=${enabled}"
done <<'EOF'
us-east-1|arn:aws:secretsmanager:us-east-1:163436765242:secret:rds!db-f85d5f08-a9c5-4f04-9b1b-5ebe82af004b-YxJ40o
eu-west-1|arn:aws:secretsmanager:eu-west-1:163436765242:secret:rds!db-c2457e98-ee04-43df-88a0-2ba6a51bcd4b-pHNiwD
eu-west-2|arn:aws:secretsmanager:eu-west-2:163436765242:secret:rds!db-84a7c030-16be-4378-99fb-2b6380614c48-N2CU3U
ap-southeast-2|arn:aws:secretsmanager:ap-southeast-2:163436765242:secret:rds!db-62ea06d6-f3e2-4c18-8131-94b8907d3ebe-5MN4ox
ap-southeast-2|arn:aws:secretsmanager:ap-southeast-2:163436765242:secret:rds!db-8790c16b-e0de-4e58-b2d6-200c1221aa86-6PHmH7
EOF
```

Expect RotationEnabled=False for all five.

> Disabling rotation does not change the current password. It only stops future scheduled rotations.

---

## Phase 2 — Build and push fixed API + worker images

Paste this whole block:

```bash
cd "${API_REPO_DIR}"
export IMAGE_TAG="prod-dbfix-$(date +%Y%m%d%H%M)-$(git rev-parse --short HEAD)"
echo "IMAGE_TAG=${IMAGE_TAG}"
bash scripts/deployment/build-all.sh
aws ecr get-login-password --region "${PRIMARY_REGION}" | docker login --username AWS --password-stdin "${AWS_ACCOUNT_ID}.dkr.ecr.${PRIMARY_REGION}.amazonaws.com"
docker tag hyrelog-api:local "${ECR_API}:${IMAGE_TAG}"
docker tag hyrelog-worker:local "${ECR_WORKER}:${IMAGE_TAG}"
docker push "${ECR_API}:${IMAGE_TAG}"
docker push "${ECR_WORKER}:${IMAGE_TAG}"
```

Save IMAGE_TAG — you need it in Phase 5.

---

## Phase 3 — One manual rotation of all five passwords

Rotate one at a time. Paste this whole block (waits 90s between each secret):

```bash
while IFS='|' read -r region secret_arn; do
  echo "==> Rotating in ${region}..."
  aws secretsmanager rotate-secret --region "${region}" --secret-id "${secret_arn}" --rotate-immediately
  echo "    Waiting 90s..."
  sleep 90
done <<'EOF'
us-east-1|arn:aws:secretsmanager:us-east-1:163436765242:secret:rds!db-f85d5f08-a9c5-4f04-9b1b-5ebe82af004b-YxJ40o
eu-west-1|arn:aws:secretsmanager:eu-west-1:163436765242:secret:rds!db-c2457e98-ee04-43df-88a0-2ba6a51bcd4b-pHNiwD
eu-west-2|arn:aws:secretsmanager:eu-west-2:163436765242:secret:rds!db-84a7c030-16be-4378-99fb-2b6380614c48-N2CU3U
ap-southeast-2|arn:aws:secretsmanager:ap-southeast-2:163436765242:secret:rds!db-62ea06d6-f3e2-4c18-8131-94b8907d3ebe-5MN4ox
ap-southeast-2|arn:aws:secretsmanager:ap-southeast-2:163436765242:secret:rds!db-8790c16b-e0de-4e58-b2d6-200c1221aa86-6PHmH7
EOF
```

Optional: in Secrets Manager, confirm each secret is not In progress before continuing.

---

## Phase 4 — Sync dashboard DATABASE_URL (do not skip)

API/worker build URLs at container start from RDS JSON secrets. The dashboard still uses static hyrelog-prod/DATABASE_URL.

```bash
cd "${API_REPO_DIR}"
REDEPLOY_SERVICES=false bash scripts/deployment/sync-secrets-after-rds-rotation.sh
```

---

## Phase 5 — Register task definitions and deploy ECS

### 5A) Render and register task definitions

Paste this whole block (IMAGE_TAG must still be set from Phase 2). Run from the **hyrelog-api** repo root:

```bash
cd "${API_REPO_DIR}"
export S3_BUCKET_US=hyrelog-prod-events-us
export S3_BUCKET_EU=hyrelog-prod-events-eu
export S3_BUCKET_UK=hyrelog-prod-events-uk
export S3_BUCKET_AU=hyrelog-prod-events-au
echo "Using IMAGE_TAG=${IMAGE_TAG}"
# Password-only recovery: do NOT set IMAGE_TAG on the dashboard unless you also built and pushed hyrelog-dashboard with that tag.
# render-task-defs.sh leaves the dashboard image from infra/ecs/task-definition-dashboard.json unless you set DASHBOARD_IMAGE_TAG.
bash scripts/deployment/render-task-defs.sh --register
```

Confirm the rendered dashboard image exists in ECR before deploy (Git Bash):

```bash
aws ecr describe-images --repository-name hyrelog-dashboard --region "${PRIMARY_REGION}" --image-ids imageTag="$(python -c "import json;print(json.load(open('infra/ecs/rendered/task-definition-dashboard.rendered.json'))['containerDefinitions'][0]['image'].split(':')[-1])")"
```

If that command returns `ImageNotFoundException`, you registered a task definition ECS cannot start — no container logs will appear. Re-render (pull latest `render-task-defs.sh`) or set `DASHBOARD_IMAGE_TAG` to a tag that exists, register again, then deploy.

If Python reports `FileNotFoundError` with a path like `\\c\\Users\\...`, pull latest `main` (render script converts Git Bash paths for Windows Python) and re-run the command above.

### 5B) Roll services to latest task definition revisions

```bash
cd "${API_REPO_DIR}"
export DEPLOY_API=true
export DEPLOY_WORKER=true
# After API/worker image fix only, skip dashboard deploy unless you registered a valid dashboard image:
export DEPLOY_DASHBOARD=false
bash scripts/deployment/update-ecs-services.sh
```

Waits until the selected services are stable. For dashboard, pick up the new `hyrelog-prod/DATABASE_URL` with a **force-new-deployment** on the current task definition (no new image required):

```bash
aws ecs update-service --region "${PRIMARY_REGION}" --cluster "${ECS_CLUSTER}" --service hyrelog-dashboard --force-new-deployment
aws ecs wait services-stable --region "${PRIMARY_REGION}" --cluster "${ECS_CLUSTER}" --services hyrelog-dashboard
```

### 5C) Alternative — force new deployment only

Only if you did **not** run render-task-defs.sh --register above:

```bash
aws ecs update-service --region "${PRIMARY_REGION}" --cluster "${ECS_CLUSTER}" --service hyrelog-api --force-new-deployment
aws ecs update-service --region "${PRIMARY_REGION}" --cluster "${ECS_CLUSTER}" --service hyrelog-worker --force-new-deployment
aws ecs update-service --region "${PRIMARY_REGION}" --cluster "${ECS_CLUSTER}" --service hyrelog-dashboard --force-new-deployment
aws ecs wait services-stable --region "${PRIMARY_REGION}" --cluster "${ECS_CLUSTER}" --services hyrelog-api hyrelog-worker hyrelog-dashboard
```

### 5D) Console fallback

If render-task-defs.sh fails, create new ECS revisions manually:

1. hyrelog-api → image 163436765242.dkr.ecr.ap-southeast-2.amazonaws.com/hyrelog-api:${IMAGE_TAG}
2. hyrelog-worker → image 163436765242.dkr.ecr.ap-southeast-2.amazonaws.com/hyrelog-worker:${IMAGE_TAG}
3. hyrelog-dashboard → only change image if you built a new dashboard; otherwise redeploy existing revision to pick up the new DATABASE_URL secret.

Then run 5B again.

---

## Phase 6 — Verify

### 6A) API task started successfully (critical)

```bash
TASK_ARN=$(aws ecs list-tasks --region "${PRIMARY_REGION}" --cluster "${ECS_CLUSTER}" --service-name hyrelog-api --desired-status RUNNING --query taskArns[0] --output text)
aws ecs describe-tasks --region "${PRIMARY_REGION}" --cluster "${ECS_CLUSTER}" --tasks "${TASK_ARN}" --query 'tasks[0].{started:startedAt,status:lastStatus}' --output table
```

startedAt should be recent (not an old task from before this recovery).

```bash
aws logs tail /ecs/hyrelog-api --since 30m --region "${PRIMARY_REGION}" --format short
```

Must not contain: node: bad option

If deploy rolled back:

```bash
aws ecs describe-services --region "${PRIMARY_REGION}" --cluster "${ECS_CLUSTER}" --services hyrelog-api --query 'services[0].events[0:8].message' --output text
```

### 6B) Health and app

```bash
curl -sS https://api.hyrelog.com/health
```

In the browser: https://app.hyrelog.com → sign in → Event Explorer loads events.

### 6C) Dashboard secret host (sanity)

Paste this whole block:

```bash
python - <<'PY'
import subprocess
import urllib.parse as u
s = subprocess.check_output(
    ["aws", "secretsmanager", "get-secret-value", "--secret-id", "hyrelog-prod/DATABASE_URL", "--region", "ap-southeast-2", "--query", "SecretString", "--output", "text"],
    text=True,
)
print("host=", u.urlparse(s).hostname)
PY
```

Expect host= hyrelog-prod-dashboard.c9umosqssoce.ap-southeast-2.rds.amazonaws.com

---

## Phase 7 — Re-enable automation (optional, after stable)

Only after everything works for 24h.

```bash
aws events enable-rule --region "${PRIMARY_REGION}" --name hyrelog-prod-rds-rotation-success
```

```bash
aws lambda update-function-configuration --region "${PRIMARY_REGION}" --function-name hyrelog-prod-rds-rotation-ecs-redeploy --environment "Variables={ECS_CLUSTER=${ECS_CLUSTER},ECS_SERVICES=hyrelog-api,hyrelog-worker,hyrelog-dashboard}"
```

Keep automatic rotation **disabled** on the five secrets until dashboard ECS uses RDS JSON secrets + entrypoint (same as API). After any future dashboard DB password change:

```bash
cd "${API_REPO_DIR}"
REDEPLOY_SERVICES=true bash scripts/deployment/sync-secrets-after-rds-rotation.sh
```

---

## Quick reference — if events still fail

| Symptom | Likely cause | Action |
|---------|----------------|--------|
| node: bad option in /ecs/hyrelog-api | Old image without entrypoint fix | Phase 2–5 |
| API task startedAt is old | Deploy rolled back | ECS events + CloudWatch for failed task |
| Login fails | Stale hyrelog-prod/DATABASE_URL | Phase 4 + redeploy dashboard |
| Failed to load events | API 500 / stale API DB password | Phase 3 + 5 with new API image |
| Deploy rolled back | Health check / crash on boot | CloudWatch log stream for failed task |

**Related scripts** (hyrelog-api/scripts/deployment/):

| Script | Purpose |
|--------|---------|
| sync-secrets-after-rds-rotation.sh | Rebuild hyrelog-prod/DATABASE_URL; REDEPLOY_SERVICES=false to skip ECS |
| render-task-defs.sh --register | Register ECS task defs with IMAGE_TAG |
| update-ecs-services.sh | Point services at latest task def revisions |
| build-all.sh | Build hyrelog-api:local and hyrelog-worker:local |

See also: [RDS_ROTATION_ECS_AUTO_SYNC_RUNBOOK.md](./RDS_ROTATION_ECS_AUTO_SYNC_RUNBOOK.md).
