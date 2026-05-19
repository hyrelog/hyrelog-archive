# RDS Password Rotation -> ECS Auto Sync (No Guessing)

This runbook sets up automatic ECS redeploys after RDS-managed Secrets Manager rotations, so API tasks always reload fresh DB credentials.

It is written for your current prod setup:

- AWS account: `163436765242`
- ECS region/cluster: `ap-southeast-2` / `hyrelog-prod-ecs`
- API service: `hyrelog-api`
- RDS-managed secrets:
  - `arn:aws:secretsmanager:us-east-1:163436765242:secret:rds!db-f85d5f08-a9c5-4f04-9b1b-5ebe82af004b-YxJ40o`
  - `arn:aws:secretsmanager:eu-west-1:163436765242:secret:rds!db-c2457e98-ee04-43df-88a0-2ba6a51bcd4b-pHNiwD`
  - `arn:aws:secretsmanager:eu-west-2:163436765242:secret:rds!db-84a7c030-16be-4378-99fb-2b6380614c48-N2CU3U`
  - `arn:aws:secretsmanager:ap-southeast-2:163436765242:secret:rds!db-62ea06d6-f3e2-4c18-8131-94b8907d3ebe-5MN4ox`

---

## 0) Immediate fix (do this now)

**Why login/API breaks after rotation**

| Service | How it gets DB password | After RDS rotates |
|---------|-------------------------|-----------------|
| **API / worker** | RDS JSON secret → `DB_PASSWORD_*` → entrypoint builds `DATABASE_URL_*` | Redeploy picks up new password automatically |
| **Dashboard** | Static secret `hyrelog-prod/DATABASE_URL` (full URL) | Secret is **stale** until you sync it; redeploy alone is not enough |

Dashboard RDS (`hyrelog-prod-dashboard`) has its **own** rotation secret (`rds!db-8790c16b-...` in `ap-southeast-2`). It is **not** in the EventBridge rule below (that rule only lists the four **regional API** secrets).

**Recommended one-shot fix** (from `hyrelog-api` repo root):

```bash
bash scripts/deployment/sync-secrets-after-rds-rotation.sh
```

Windows (PowerShell):

```powershell
.\scripts\deployment\sync-secrets-after-rds-rotation.ps1
```

That script rebuilds `hyrelog-prod/DATABASE_URL` from the current dashboard RDS secret, then force-redeploys `hyrelog-api`, `hyrelog-worker`, and `hyrelog-dashboard`.

**API/worker only** (if dashboard auth DB is fine):

```bash
export PRIMARY_REGION='ap-southeast-2'
export ECS_CLUSTER='hyrelog-prod-ecs'
export API_SERVICE='hyrelog-api'

aws ecs update-service \
  --region "$PRIMARY_REGION" \
  --cluster "$ECS_CLUSTER" \
  --service "$API_SERVICE" \
  --force-new-deployment

aws ecs wait services-stable \
  --region "$PRIMARY_REGION" \
  --cluster "$ECS_CLUSTER" \
  --services "$API_SERVICE"
```

Also redeploy **worker** and **dashboard** when regional API passwords or the dashboard DB password rotated:

```bash
for svc in hyrelog-api hyrelog-worker hyrelog-dashboard; do
  aws ecs update-service --region ap-southeast-2 --cluster hyrelog-prod-ecs --service "$svc" --force-new-deployment
done
aws ecs wait services-stable --region ap-southeast-2 --cluster hyrelog-prod-ecs --services hyrelog-api hyrelog-worker hyrelog-dashboard
```

### Lambda runtime (Node.js 20 EOL on AWS)

The redeploy Lambda `hyrelog-prod-rds-rotation-ecs-redeploy` should use a supported managed runtime (use **`nodejs22.x`**). This does not require re-zipping the function unless you change dependencies.

```bash
export PRIMARY_REGION='ap-southeast-2'
export LAMBDA_FUNCTION_NAME='hyrelog-prod-rds-rotation-ecs-redeploy'

aws lambda update-function-configuration \
  --region "$PRIMARY_REGION" \
  --function-name "$LAMBDA_FUNCTION_NAME" \
  --runtime nodejs22.x
```

Then smoke-test (same as [§5](#5-validate-setup)) and confirm the function configuration shows `nodejs22.x`.

---

## 1) One-time variables

```bash
export AWS_ACCOUNT_ID='163436765242'
export PRIMARY_REGION='ap-southeast-2'
export ECS_CLUSTER='hyrelog-prod-ecs'
export API_SERVICE='hyrelog-api'

export LAMBDA_ROLE_NAME='hyrelog-prod-rds-rotation-ecs-redeploy-role'
export LAMBDA_FUNCTION_NAME='hyrelog-prod-rds-rotation-ecs-redeploy'
export EVENT_RULE_NAME='hyrelog-prod-rds-rotation-success'
export TMP_DIR='scripts/deployment/tmp'

mkdir -p "$TMP_DIR"
```

---

## 2) Create IAM role for Lambda

Create trust policy:

```bash
cat > "$TMP_DIR/rds-rotation-lambda-trust.json" <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

Create role:

```bash
aws iam create-role \
  --role-name "$LAMBDA_ROLE_NAME" \
  --assume-role-policy-document "file://$TMP_DIR/rds-rotation-lambda-trust.json"
```

Attach CloudWatch logs policy:

```bash
aws iam attach-role-policy \
  --role-name "$LAMBDA_ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

Create scoped ECS policy:

```bash
cat > "$TMP_DIR/rds-rotation-lambda-ecs-policy.json" <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowUpdateSpecificService",
      "Effect": "Allow",
      "Action": [
        "ecs:UpdateService",
        "ecs:DescribeServices"
      ],
      "Resource": [
        "arn:aws:ecs:ap-southeast-2:163436765242:service/hyrelog-prod-ecs/hyrelog-api"
      ]
    },
    {
      "Sid": "AllowDescribeCluster",
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeClusters"
      ],
      "Resource": [
        "arn:aws:ecs:ap-southeast-2:163436765242:cluster/hyrelog-prod-ecs"
      ]
    }
  ]
}
EOF
```

```bash
aws iam put-role-policy \
  --role-name "$LAMBDA_ROLE_NAME" \
  --policy-name "${LAMBDA_ROLE_NAME}-ecs-policy" \
  --policy-document "file://$TMP_DIR/rds-rotation-lambda-ecs-policy.json"
```

Get role ARN:

```bash
export LAMBDA_ROLE_ARN="$(aws iam get-role --role-name "$LAMBDA_ROLE_NAME" --query 'Role.Arn' --output text)"
echo "$LAMBDA_ROLE_ARN"
```

---

## 3) Create Lambda function

Create function code:

```bash
mkdir -p "$TMP_DIR/rds-rotation-redeploy"
cat > "$TMP_DIR/rds-rotation-redeploy/index.mjs" <<'EOF'
import { ECSClient, UpdateServiceCommand } from "@aws-sdk/client-ecs";

const ecs = new ECSClient({ region: process.env.AWS_REGION || "ap-southeast-2" });

export const handler = async (event) => {
  console.log("rotation-event", JSON.stringify(event));

  const cluster = process.env.ECS_CLUSTER;
  const services = (process.env.ECS_SERVICES || "")
    .split(",")
    .map((s) => s.trim())
    .filter(Boolean);

  if (!cluster || services.length === 0) {
    throw new Error("Missing ECS_CLUSTER or ECS_SERVICES");
  }

  for (const service of services) {
    await ecs.send(
      new UpdateServiceCommand({
        cluster,
        service,
        forceNewDeployment: true
      })
    );
    console.log(`Forced new deployment for ${cluster}/${service}`);
  }

  return { ok: true };
};
EOF
```

Zip and create:

```bash
cd "$TMP_DIR/rds-rotation-redeploy"
zip -q function.zip index.mjs

aws lambda create-function \
  --region "$PRIMARY_REGION" \
  --function-name "$LAMBDA_FUNCTION_NAME" \
  --runtime nodejs22.x \
  --handler index.handler \
  --role "$LAMBDA_ROLE_ARN" \
  --zip-file fileb://function.zip \
  --timeout 30 \
  --memory-size 128 \
  --environment "Variables={ECS_CLUSTER=$ECS_CLUSTER,ECS_SERVICES=$API_SERVICE}"
```

If `zip` is not installed in Git Bash on Windows, use PowerShell:

```bash
powershell -NoProfile -Command "Compress-Archive -Path index.mjs -DestinationPath function.zip -Force"
```

If function already exists, update code/config:

```bash
aws lambda update-function-code \
  --region "$PRIMARY_REGION" \
  --function-name "$LAMBDA_FUNCTION_NAME" \
  --zip-file "fileb://$TMP_DIR/rds-rotation-redeploy/function.zip"

aws lambda update-function-configuration \
  --region "$PRIMARY_REGION" \
  --function-name "$LAMBDA_FUNCTION_NAME" \
  --runtime nodejs22.x \
  --environment "Variables={ECS_CLUSTER=$ECS_CLUSTER,ECS_SERVICES=$API_SERVICE}"
```

---

## 4) Create EventBridge rule (rotation success)

Create event pattern file:

```bash
cat > "$TMP_DIR/rds-rotation-event-pattern.json" <<'EOF'
{
  "source": ["aws.secretsmanager"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["secretsmanager.amazonaws.com"],
    "eventName": ["RotateSecret"],
    "requestParameters": {
      "secretId": [
        "arn:aws:secretsmanager:us-east-1:163436765242:secret:rds!db-f85d5f08-a9c5-4f04-9b1b-5ebe82af004b-YxJ40o",
        "arn:aws:secretsmanager:eu-west-1:163436765242:secret:rds!db-c2457e98-ee04-43df-88a0-2ba6a51bcd4b-pHNiwD",
        "arn:aws:secretsmanager:eu-west-2:163436765242:secret:rds!db-84a7c030-16be-4378-99fb-2b6380614c48-N2CU3U",
        "arn:aws:secretsmanager:ap-southeast-2:163436765242:secret:rds!db-62ea06d6-f3e2-4c18-8131-94b8907d3ebe-5MN4ox"
      ]
    }
  }
}
EOF
```

Create rule:

```bash
aws events put-rule \
  --region "$PRIMARY_REGION" \
  --name "$EVENT_RULE_NAME" \
  --event-pattern "file://$TMP_DIR/rds-rotation-event-pattern.json" \
  --state ENABLED
```

Get Lambda ARN:

```bash
export LAMBDA_ARN="$(aws lambda get-function --region "$PRIMARY_REGION" --function-name "$LAMBDA_FUNCTION_NAME" --query 'Configuration.FunctionArn' --output text)"
echo "$LAMBDA_ARN"
```

Attach Lambda target:

```bash
aws events put-targets \
  --region "$PRIMARY_REGION" \
  --rule "$EVENT_RULE_NAME" \
  --targets "Id"="1","Arn"="$LAMBDA_ARN"
```

Allow EventBridge invoke:

```bash
aws lambda add-permission \
  --region "$PRIMARY_REGION" \
  --function-name "$LAMBDA_FUNCTION_NAME" \
  --statement-id "${EVENT_RULE_NAME}-invoke" \
  --action "lambda:InvokeFunction" \
  --principal events.amazonaws.com \
  --source-arn "arn:aws:events:${PRIMARY_REGION}:${AWS_ACCOUNT_ID}:rule/${EVENT_RULE_NAME}"
```

---

## 5) Validate setup

Check rule:

```bash
aws events describe-rule \
  --region "$PRIMARY_REGION" \
  --name "$EVENT_RULE_NAME"
```

Check targets:

```bash
aws events list-targets-by-rule \
  --region "$PRIMARY_REGION" \
  --rule "$EVENT_RULE_NAME"
```

Smoke test Lambda manually:

```bash
aws lambda invoke \
  --region "$PRIMARY_REGION" \
  --function-name "$LAMBDA_FUNCTION_NAME" \
  --cli-binary-format raw-in-base64-out \
  --payload '{"manual":"test"}' \
  "$TMP_DIR/lambda-invoke-out.json" && cat "$TMP_DIR/lambda-invoke-out.json"
```

Verify ECS deployment happened:

```bash
aws ecs describe-services \
  --region "$PRIMARY_REGION" \
  --cluster "$ECS_CLUSTER" \
  --services "$API_SERVICE" \
  --query 'services[0].deployments[*].{status:status,rollout:rolloutState,taskDef:taskDefinition,running:runningCount,pending:pendingCount}' \
  --output table
```

---

## 6) Operational checks after real rotation

After the next secret rotation:

1. EventBridge rule gets matched event.
2. Lambda CloudWatch logs show `Forced new deployment`.
3. ECS service starts new tasks and reaches steady state.
4. API logs should not show Prisma `P1000` password auth failures.

Get Lambda logs quickly:

```bash
aws logs tail "/aws/lambda/${LAMBDA_FUNCTION_NAME}" --since 1h --follow --region "$PRIMARY_REGION"
```

---

## 7) Rollback / disable

Disable only automation (keep function and rule):

```bash
aws events disable-rule \
  --region "$PRIMARY_REGION" \
  --name "$EVENT_RULE_NAME"
```

Re-enable:

```bash
aws events enable-rule \
  --region "$PRIMARY_REGION" \
  --name "$EVENT_RULE_NAME"
```

Emergency manual redeploy anytime:

```bash
aws ecs update-service \
  --region "$PRIMARY_REGION" \
  --cluster "$ECS_CLUSTER" \
  --service "$API_SERVICE" \
  --force-new-deployment
```

---

## Notes

- This automation watches rotation API calls via CloudTrail and redeploys API service.
- Set Lambda env `ECS_SERVICES=hyrelog-api,hyrelog-worker,hyrelog-dashboard` so all three restart after regional API secret rotation.
- **Dashboard auth DB** rotation still requires syncing `hyrelog-prod/DATABASE_URL` (see `sync-secrets-after-rds-rotation.sh`) unless you migrate the dashboard task to RDS JSON secrets + entrypoint like API/worker.
- Add the dashboard RDS secret ARN to EventBridge (or a second rule) if you want automation when **only** the dashboard instance rotates.
- In Git Bash, ARNs containing `!` should be single-quoted if used directly in shell commands.
- On Windows Git Bash, keep temp JSON/zip files in the repo (like `scripts/deployment/tmp`) instead of `/tmp` for consistent `file://` behavior with AWS CLI.
- On AWS CLI v2, `aws lambda invoke` with inline JSON payload needs `--cli-binary-format raw-in-base64-out`.
