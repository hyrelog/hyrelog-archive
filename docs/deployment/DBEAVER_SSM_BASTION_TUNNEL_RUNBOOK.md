# DBeaver + SSM Bastion Tunnel Runbook (Private RDS)

Use this to connect DBeaver to private RDS databases through a bastion EC2 instance using AWS Systems Manager (SSM), without opening RDS publicly.

This covers your HyreLog production-style setup end to end:

- dashboard DB
- API US DB
- API EU DB
- API UK DB
- API AU DB

---

## 0) What you need

- AWS CLI configured and authenticated
- Session Manager plugin installed
- IAM permissions for:
  - `ssm:StartSession`
  - `ssm:TerminateSession`
  - `ec2:DescribeInstances`
  - `rds:DescribeDBInstances`
- One running bastion EC2 instance in VPC (managed by SSM) — create one per **§0.3** if you do not have it yet
- DBeaver installed
- DB credentials (typically from Secrets Manager)

---

## 0.1) IAM permissions (your laptop identity vs. the bastion)

Two different IAM setups are involved:

| Who | What they need |
|-----|----------------|
| **Your AWS identity** (IAM user, or IAM Identity Center / SSO role you use with `aws sso login`) | Permission to run **`aws ssm start-session`** (port forward), describe EC2/RDS/SSM for discovery, and end sessions. |
| **The bastion EC2 instance** | An **instance profile** with **`AmazonSSMManagedInstanceCore`** (or equivalent) so SSM can reach the host. This is **not** the same as your user policy — it is attached to the **instance**, not to you. |

### Minimum actions for *your* identity

The bullets in §0 map to the JSON in **`docs/deployment/iam-dbeaver-ssm-operator-policy.json`** (account **`163436765242`**, region **`ap-southeast-2`**). That file also includes **`ssm:DescribeInstanceInformation`**, which §3 uses when listing SSM-managed instances — add it if you omit it and get **AccessDenied** on **`describe-instance-information`**.

- **`ssm:StartSession`** — must allow both:
  - The **bastion instance** (`arn:aws:ec2:…:instance/…` or `…:instance/*` if you accept broader scope).
  - The **AWS-managed SSM document** for port forwarding to a remote host:  
    `arn:aws:ssm:ap-southeast-2::document/AWS-StartPortForwardingSessionToRemoteHost`  
    (note the **empty account** in `::document` for AWS-owned documents).
- **`ssm:TerminateSession`** — usually **`Resource": "*"`** so you can stop sessions by ID.
- **`ec2:DescribeInstances`**, **`rds:DescribeDBInstances`** — read-only discovery (`Resource: "*"` is typical).

Tighten **`ssm:StartSession`** later by replacing **`instance/*`** with your bastion’s **`i-…`** ARN only.

### How to attach this policy

**A) Long-lived IAM user (access keys / `aws configure`)**

1. IAM → **Users** → your user → **Permissions** → **Add permissions** → **Create inline policy** → **JSON** → paste the contents of **`iam-dbeaver-ssm-operator-policy.json`**, or attach a **customer managed policy** with the same statements.
2. Or Git Bash (inline JSON, same pattern as `github-deploy-policy`):

```bash
cd /c/Users/kram/Dropbox/hyrelog/hyrelog-api/docs/deployment

aws iam put-user-policy \
  --user-name YOUR_IAM_USER_NAME \
  --policy-name dbeaver-ssm-operator \
  --policy-document "$(jq -c . iam-dbeaver-ssm-operator-policy.json)"
```

**B) IAM Identity Center (SSO)**

Policies are not attached with **`put-user-policy`**. An admin must add equivalent permissions to your **permission set** (IAM Identity Center → **Permission sets** → edit → **Customer managed policy** or inline policy). Use the same JSON as **`iam-dbeaver-ssm-operator-policy.json`**.

**C) Assume-role workflow**

If you use **`aws sts assume-role`**, attach the policy to that **role** instead:

```bash
aws iam put-role-policy \
  --role-name YOUR_ROLE_NAME \
  --policy-name dbeaver-ssm-operator \
  --policy-document "$(jq -c . iam-dbeaver-ssm-operator-policy.json)"
```

Bastion setup (SSM agent, instance role, network) is in **§0.2**.

---

## 0.2) Bastion EC2: SSM agent, instance profile, networking

You (§0.1) and the **instance** need different IAM. The bastion must be able to **register with SSM** and stay **Online**. If **`describe-instance-information`** never shows the instance, or **PingStatus** is not **Online**, fix this section first.

### A) Confirm the instance can register with SSM (quick check)

After the steps below, this should list your bastion with **PingStatus** `Online`:

```bash
aws ssm describe-instance-information \
  --region ap-southeast-2 \
  --query 'InstanceInformationList[].[InstanceId,PingStatus,AgentVersion]' \
  --output table
```

If the instance never appears, SSM Agent is not talking to the service (wrong IAM, no network path, or agent not running).

### B) SSM Agent on the instance

- **Amazon Linux 2 / 2023:** Agent is pre-installed and enabled; usually nothing to do.
- **Ubuntu:** Often pre-installed; if not: [Install SSM Agent on Linux](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-manual-agent-install.html).
- **Windows:** Install/configure per AWS docs if you use a Windows bastion.

You typically **do not** need SSH to fix agent issues if you use **EC2 Instance Connect** or a **serial console** only when the instance is unreachable.

### C) IAM on the **instance** (not your user)

1. **IAM → Roles → Create role**
   - Trusted entity: **AWS service** → **EC2**.
2. **Add permissions** → attach AWS managed policy **`AmazonSSMManagedInstanceCore`** (minimum for Session Manager).
3. Name the role (e.g. **`hyrelog-bastion-ssm-role`**) → **Create role**.
4. **Instance profile:** Creating an EC2 service role usually creates a matching **instance profile** with the same name. If you create the profile manually: **IAM → Instance profiles → Create** → add that role.
5. **Attach to the bastion:** **EC2 → Instances** → select bastion → **Actions → Security → Modify IAM role** → choose the role → **Update IAM role**.

CLI (replace IDs/names):

```bash
# If you already have a role + instance profile named hyrelog-bastion-ssm-role:
aws ec2 associate-iam-instance-profile \
  --region ap-southeast-2 \
  --instance-id i-BASTION_INSTANCE_ID \
  --iam-instance-profile Name=hyrelog-bastion-ssm-role
```

If the instance already has another profile, use **`replace-iam-instance-profile`** (or remove the old association first per AWS docs).

### D) Networking (SSM must be reachable from the instance)

Pick **one** of these patterns:

| Pattern | What you need |
|--------|----------------|
| **Public subnet + Internet access** | Route table has **`0.0.0.0/0` → Internet Gateway**, instance has **public IP** (or uses a **NAT Gateway** in a private subnet that has IGW route). Outbound HTTPS to AWS can reach **public SSM endpoints**. |
| **Private subnet only (no IGW on subnet)** | Create **VPC interface endpoints** so the instance does not need the public internet for SSM. In **`ap-southeast-2`**, typically create endpoints for: **`com.amazonaws.ap-southeast-2.ssm`**, **`com.amazonaws.ap-southeast-2.ssmmessages`**, **`com.amazonaws.ap-southeast-2.ec2messages`**. Attach **security groups** that allow **HTTPS (443)** from the instance subnets to the endpoint ENIs. See [VPC endpoints for Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-create-vpc.html). |

DNS inside the VPC must resolve (VPC **DNS hostnames** / **DNS support** enabled on the VPC).

### E) After changes

- Wait **1–3 minutes**, then re-run **§0.2 A** and **§3.1** in this runbook.
- If still **Connection lost** / instance missing from SSM: check **endpoint SGs**, **route tables**, **NACLs**, and that the role is **`AmazonSSMManagedInstanceCore`** on the **instance**, not only on your IAM user.

---

## 0.3) Create a bastion EC2 from scratch (console-first)

A **bastion** here is just a **small EC2** in the **same VPC as RDS**. You connect to **RDS** through it using **Session Manager port forwarding** (`AWS-StartPortForwardingSessionToRemoteHost`). You do **not** need to open SSH from the internet if you use SSM only.

### Before you start

Collect:

| Item | Why |
|------|-----|
| **VPC ID** where RDS lives | Bastion must be in **this** VPC. |
| **RDS security group ID** (e.g. `sg-…`) | You will allow Postgres **from** the bastion’s SG. |
| Subnet choice | **Same VPC**; see networking note below. |

**Networking (pick one simple pattern):**

- **Easiest:** Put the bastion in a **public subnet** that has a route **`0.0.0.0/0` → Internet Gateway**, enable **Auto-assign public IP** on launch. SSM agent reaches AWS over the internet; the bastion reaches RDS over **private IPs** inside the VPC.
- **Private-only VPC:** Use a **private subnet** with **NAT Gateway** outbound **or** the three **SSM VPC endpoints** from §0.2 D.

### Step 1 — IAM role for the instance (once per account/VPC pattern)

Create role **`hyrelog-bastion-ssm-role`** (or any name) exactly as in **§0.2 C**: trust **EC2**, attach **`AmazonSSMManagedInstanceCore`**. You will select this role when launching the instance.

### Step 2 — Security groups

1. **EC2 → Security Groups → Create security group**
   - **VPC:** same as RDS.
   - **Name:** e.g. `hyrelog-bastion-ssm`.
   - **Inbound:** none required for SSM from the internet (you start sessions via AWS API). Optional: **SSH (22)** only from **your IP** if you want emergency SSH; not required for DBeaver + SSM tunnel.
   - **Outbound:** default **allow all** is fine for troubleshooting; tighter: **HTTPS 443** anywhere (SSM) + **TCP 5432** to the **RDS security group** (or keep **allow all** outbound).

2. **Allow RDS to accept Postgres from the bastion**

   Open the **RDS instance / cluster → VPC security groups → inbound rules** on the SG attached to RDS.

   - **Type:** PostgreSQL (or **Custom TCP** `5432`).
   - **Source:** the **bastion security group** (`hyrelog-bastion-ssm`), **not** a public CIDR. That way only this bastion can reach the DB on the VPC network.

### Step 3 — Launch the instance

**EC2 → Launch instance**

1. **Name:** e.g. `hyrelog-ssm-bastion`.
2. **AMI:** **Amazon Linux 2023** (includes SSM Agent).
3. **Instance type:** **`t3.micro`** or **`t3.small`** is enough.
4. **Key pair:** **No key pair** is OK if you rely only on SSM; create one only if you want SSH fallback.
5. **Network:**
   - **VPC:** RDS VPC.
   - **Subnet:** a subnet in that VPC (public subnet for the “easiest” pattern above).
   - **Auto-assign public IP:** **Enable** if you use a public subnet + IGW path for SSM.
6. **Security group:** select **`hyrelog-bastion-ssm`** (from Step 2).
7. **Advanced details → IAM instance profile:** select the role from Step 1 (`hyrelog-bastion-ssm-role`).
8. **Storage:** default **8–30 GiB** gp3 is fine.

**Launch instance.**

### Step 4 — Verify SSM sees it

Wait **1–3 minutes**, then:

```bash
aws ssm describe-instance-information \
  --region ap-southeast-2 \
  --query 'InstanceInformationList[?PingStatus==`Online`].[InstanceId,PingStatus]' \
  --output table
```

You should see **`i-…`** with **`Online`**. Then continue from **§3** (find bastion ID) and the port-forward commands later in this runbook.

### If the instance never shows **Online**

Work through **§0.2 D–E** (routes, NAT or VPC endpoints, instance profile on the **instance**).

### Cost note

You pay for **EC2 hours**, **EBS**, and (if used) **NAT Gateway** / **VPC endpoints**. **Stop** the bastion when not using DBeaver if you want to save money; **start** it again before tunnels.

---

## 1) Verify local tools

Run in Git Bash or PowerShell:

```bash
aws --version
session-manager-plugin --version
aws sts get-caller-identity
```

If `session-manager-plugin` is missing, install it first:

- [AWS Session Manager plugin install docs](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)

---

## 2) Set baseline environment vars

```bash
export PRIMARY_REGION=ap-southeast-2
export AWS_DEFAULT_REGION=ap-southeast-2
```

Optional account check:

```bash
aws sts get-caller-identity --query '[Account,Arn]' --output text
```

---

## 3) Find bastion instance ID

### 3.1 List SSM-managed online instances

```bash
aws ssm describe-instance-information \
  --region "${PRIMARY_REGION}" \
  --query 'InstanceInformationList[?PingStatus==`Online`].[InstanceId,ComputerName,PlatformName,IPAddress]' \
  --output table
```

### 3.2 Find likely bastion in EC2 list

```bash
aws ec2 describe-instances \
  --region "${PRIMARY_REGION}" \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,PrivateIpAddress,Tags[?Key==`Name`]|[0].Value]' \
  --output table
```

Choose your bastion instance ID and set:

```bash
export BASTION_INSTANCE_ID=i-REPLACE_ME
```

---

## 4) Get RDS endpoints

List likely DBs:

```bash
for r in ap-southeast-2 us-east-1 eu-west-1 eu-west-2; do
  echo "== $r =="
  aws rds describe-db-instances \
    --region "$r" \
    --query 'DBInstances[].[DBInstanceIdentifier,Endpoint.Address,Endpoint.Port,DBName,DBInstanceStatus]' \
    --output table
done
```

Set endpoint vars (replace with real values):

```bash
export DASHBOARD_RDS_HOST=dashboard-host.rds.amazonaws.com
export API_US_RDS_HOST=api-us-host.rds.amazonaws.com
export API_EU_RDS_HOST=api-eu-host.rds.amazonaws.com
export API_UK_RDS_HOST=api-uk-host.rds.amazonaws.com
export API_AU_RDS_HOST=api-au-host.rds.amazonaws.com
```

---

## 5) Choose local tunnel ports

Use fixed local ports so DBeaver configs stay stable:

- dashboard -> `15432`
- api-us -> `15433`
- api-eu -> `15434`
- api-uk -> `15435`
- api-au -> `15436`

---

## 6) Start SSM tunnels (one terminal per DB)

Open **five terminals** (or run one at a time). Keep each tunnel terminal open while using DBeaver.

### 6.1 Dashboard tunnel

```bash
aws ssm start-session \
  --region "${PRIMARY_REGION}" \
  --target "${BASTION_INSTANCE_ID}" \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["'"${DASHBOARD_RDS_HOST}"'"],"portNumber":["5432"],"localPortNumber":["15432"]}'
```

### 6.2 API US tunnel

```bash
aws ssm start-session \
  --region "${PRIMARY_REGION}" \
  --target "${BASTION_INSTANCE_ID}" \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["'"${API_US_RDS_HOST}"'"],"portNumber":["5432"],"localPortNumber":["15433"]}'
```

### 6.3 API EU tunnel

```bash
aws ssm start-session \
  --region "${PRIMARY_REGION}" \
  --target "${BASTION_INSTANCE_ID}" \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["'"${API_EU_RDS_HOST}"'"],"portNumber":["5432"],"localPortNumber":["15434"]}'
```

### 6.4 API UK tunnel

```bash
aws ssm start-session \
  --region "${PRIMARY_REGION}" \
  --target "${BASTION_INSTANCE_ID}" \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["'"${API_UK_RDS_HOST}"'"],"portNumber":["5432"],"localPortNumber":["15435"]}'
```

### 6.5 API AU tunnel

```bash
aws ssm start-session \
  --region "${PRIMARY_REGION}" \
  --target "${BASTION_INSTANCE_ID}" \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["'"${API_AU_RDS_HOST}"'"],"portNumber":["5432"],"localPortNumber":["15436"]}'
```

---

## 7) Configure DBeaver connections

Create one PostgreSQL connection per DB.

For each connection in DBeaver:

1. **Database -> New Database Connection -> PostgreSQL**
2. Main:
   - Host: `127.0.0.1`
   - Port: one of `15432..15436`
   - Database: target DB name
   - Username/password: from secret
3. SSL:
   - Start with `require`
   - If your DB/user policies require strict cert validation, configure CA cert accordingly
4. Click **Test Connection**
5. Save

Recommended connection names:

- `prod-dashboard (15432)`
- `prod-api-us (15433)`
- `prod-api-eu (15434)`
- `prod-api-uk (15435)`
- `prod-api-au (15436)`

---

## 8) Validate DB access with SQL

### 8.1 API DBs (run on each of US/EU/UK/AU)

```sql
SELECT "planTier", COUNT(*) FROM plans GROUP BY "planTier" ORDER BY "planTier";
```

Expected tiers:

- FREE
- STARTER
- GROWTH
- ENTERPRISE

### 8.2 Dashboard DB

```sql
SELECT code, name, status FROM plans ORDER BY code;
SELECT code, name, "isActive" FROM addons ORDER BY code;
```

Expected plan codes:

- FREE
- STARTER
- PRO
- BUSINESS
- ENTERPRISE

Expected addon codes:

- ADDON_EXTRA_EVENTS
- ADDON_RETENTION_EXTENDED
- ADDON_EXTRA_SEATS
- ADDON_SSO
- ADDON_ADDITIONAL_REGION

---

## 9) Troubleshooting

## 9.1 Tunnel starts but DBeaver cannot connect

- Ensure tunnel terminal is still open
- Ensure DBeaver host is `127.0.0.1` (not RDS host)
- Confirm local port matches the tunnel command

## 9.2 `TargetNotConnected` / SSM errors

- Bastion is not SSM-managed/online
- SSM agent not running
- IAM permissions missing

## 9.3 `connection refused` / timeout

- Wrong RDS host
- Bastion SG/NACL route cannot reach DB SG
- RDS SG does not allow inbound from bastion or ECS/bastion SG

## 9.4 TLS/certificate errors

- Use SSL mode `require` first
- If strict validation required, import CA and use `verify-ca` / `verify-full`

---

## 10) Close tunnels

Stop each tunnel by pressing `Ctrl+C` in each tunnel terminal.

---

## 11) Security notes

- Do not store raw DB passwords in plaintext files committed to git.
- Prefer pulling creds from Secrets Manager when opening DBeaver connections.
- Close tunnels when not actively using them.

