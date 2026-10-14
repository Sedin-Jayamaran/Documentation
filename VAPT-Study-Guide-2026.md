# QReport VAPT Environment Build — Complete Study Guide (Oct 2026)

> A hands-on record of building the **QReport VAPT** (Vulnerability Assessment & Penetration Testing) environment in AWS Sydney (`ap-southeast-2`) on **13–14 Oct 2026**: every command used, why it was used, what went wrong, and how it was fixed.
>
> ⚠️ This file contains internal resource IDs (VPC, ALB, instance IDs). Keep it private — don't commit it to Git or share it publicly. No passwords or secret values are included; placeholders like `<PASSWORD>` are used instead.

---

## Table of Contents

1. [Big Picture — What We Built](#1-big-picture--what-we-built)
2. [Terminal & AWS CLI Setup (Tips That Saved Time)](#2-terminal--aws-cli-setup)
3. [Terraform Fundamentals Used](#3-terraform-fundamentals-used)
4. [Pre-flight Checks Before Any Apply](#4-pre-flight-checks-before-any-apply)
5. [EC2 Key Pairs — Concepts & Fixes](#5-ec2-key-pairs--concepts--fixes)
6. [Choosing the AMI (Amazon Linux 2 → 2023)](#6-choosing-the-ami)
7. [Common-Resources — Staged Apply with `-target`](#7-common-resources--staged-apply)
8. [SSH into Private Servers via SSM](#8-ssh-into-private-servers-via-ssm)
9. [ClamAV Server Setup & Testing](#9-clamav-server-setup--testing)
10. [Env-Resources — Staged Apply](#10-env-resources--staged-apply)
11. [ECS User-Data Problems (Deadlock & Blank Lines)](#11-ecs-user-data-problems)
12. [Testing the ALB & WAF](#12-testing-the-alb--waf)
13. [Connectivity Validation](#13-connectivity-validation)
14. [Secrets Manager — The Two Secrets](#14-secrets-manager--the-two-secrets)
15. [Database Work Part 1 — Restoring an Old Snapshot & Sharing a Dump](#15-database-part-1--old-snapshot--sharing-a-dump)
16. [Database Work Part 2 — Masking with AWS Batch](#16-database-part-2--masking-with-aws-batch)
17. [Database Work Part 3 — Loading the Masked Dump](#17-database-part-3--loading-the-masked-dump)
18. [CI/CD — CodePipeline & CodeBuild](#18-cicd--codepipeline--codebuild)
19. [Starting the App on ECS & Troubleshooting](#19-starting-the-app-on-ecs--troubleshooting)
20. [DNS & Accessing the App in a Browser](#20-dns--accessing-the-app)
21. [Audit Timeline & Logging](#21-audit-timeline--logging)
22. [Troubleshooting Cheat Sheet (Error → Cause → Fix)](#22-troubleshooting-cheat-sheet)
23. [Pending Items & Teardown Checklist](#23-pending-items--teardown-checklist)
24. [Glossary](#24-glossary)

---

## 1. Big Picture — What We Built

### 1.1 Repository layout

```
qreport-terraform/
├── APP/
│   ├── modules/vpc/                 # reusable VPC module
│   └── sydney/
│       ├── common-resources/        # VPC, RDS, ALB, WAF, S3, ClamAV, (Redis)
│       └── env-resources/           # ECS cluster, ASG, launch template, IAM, TG, services
└── CICD/
    ├── common-resources/            # SHARED pipeline IAM roles + artifact bucket (don't touch)
    └── env-specific/sydney/         # per-env CodeBuild + CodePipeline
```

Each folder is a separate **Terraform root module** with its own state. Environments (production, nonproduction, vapt, staging, uat) are separated using **Terraform workspaces**.

### 1.2 Architecture

```
                         Internet
                            │
                ┌───────────▼────────────┐
                │  WAF (qreport-vapt-webacl) │  ← blocks bots/scanners, bad IPs
                └───────────┬────────────┘
                ┌───────────▼────────────┐
                │ ALB qreport-vapt-ALB    │  :80 → redirect to :443
                │ (public subnets)        │  :443 → rule *-vapt.qreport.com.au
                └───────────┬────────────┘
                            │ target group qreport-vapt (port 3000)
      ┌─────────────────────▼──────────────────────────────────┐
      │ VPC 10.5.0.0/16  (private subnets 10.5.3-5.0/24)        │
      │                                                         │
      │  ECS EC2 instance (m5.2xlarge, ASG)                     │
      │    ├─ qreport-app container ──────┐                     │
      │    └─ qreport-dj container (worker)│                     │
      │                                    ├──► RDS MySQL 8.4.8 │
      │                                    │    qreport-vapt     │
      │                                    └──► ClamAV EC2       │
      │                                         10.5.3.144:3310  │
      │  NAT Gateway (public subnet) ◄── outbound internet      │
      └─────────────────────────────────────────────────────────┘

  CodePipeline vapt-qreport: GitHub (vapt branch) → CodeBuild → ECR qreport-vapt
  Secrets Manager: vapt-DB-Credentials (Terraform-filled), qreport-vapt-app (you fill)
```

### 1.3 Order of work (and why)

| # | Phase | Why this order |
|---|---|---|
| 1 | Pre-flight checks | Avoid "already exists" failures and wrong config |
| 2 | Common-resources (network, DB, ALB, ClamAV) | Everything else lives inside this network |
| 3 | ClamAV manual setup | Not automated by Terraform (no user data) |
| 4 | Env-resources (ECS, ASG, services) | Needs VPC/ALB outputs from common-resources |
| 5 | DB: mask prod data → load into VAPT | App needs data; never load raw prod data |
| 6 | Secrets | App and build both read `qreport-vapt-app` |
| 7 | CI/CD pipeline → image in ECR | ECS needs an image to run |
| 8 | Scale ECS services 0 → 1 | Start the app |
| 9 | DNS / hosts file | So people can reach it |

---

## 2. Terminal & AWS CLI Setup

### 2.1 AWS profile — case matters!

```bash
export AWS_PROFILE=q-report      # ✅ correct
export AWS_Profile=q-report      # ❌ ignored — env var names are case-sensitive
```

**Lesson:** our first error was `No valid credential sources found … no EC2 IMDS role found`. Cause: the variable was spelled `AWS_Profile`, so the CLI fell back to another profile with no credentials and then tried the EC2 metadata service (which doesn't exist on a Mac).

Verify credentials anytime:
```bash
aws sts get-caller-identity
```

### 2.2 Turn off the AWS pager

The CLI opens output in `less` and shows `(END)`. Press `q` to exit, or disable it:
```bash
export AWS_PAGER=""
```

### 2.3 zsh quirks (macOS default shell)

| Problem | Example | Fix |
|---|---|---|
| `#` comments on the command line are passed as arguments | `aws ... --output text  # expect 1` → error | Remove the comment, or `setopt interactivecomments` (add to `~/.zshrc`) |
| A variable with many words isn't split | `--tasks $T` → "Unexpected number of separators" | Use `${=T}` in zsh |
| `read` prompt syntax | `read "?Apply? (y/N) " a` | zsh-specific; bash uses `read -p` |

### 2.4 Know where you are

| Prompt | Where commands run |
|---|---|
| `~/De/W/Q-report-/… ❯` | **Your Mac** — run `aws`, `terraform`, `dig`, `scp` here |
| `[ec2-user@ip-10-5-4-181 ~]$` | **APP server (ECS EC2)** — run `mysql`, `mysqldump` here |
| `[root@ip-10-5-3-144 …]#` | **ClamAV server** |
| `sh-4.2#` | **Inside a Docker container** |

**Mistake we made:** running `scp … ~/Desktop/` *on the server* — it tried to SSH from the server to itself. `scp` that pulls files to your Mac must run **on your Mac**.

### 2.5 Useful `jq` recipes

```bash
# Only key NAMES of a secret (safe to share)
... --query SecretString --output text | jq 'keys'

# Template with empty values
... | jq 'with_entries(.value = "")'

# Filter keys by pattern, hide passwords
... | jq 'with_entries(select(.key|test("DB|DATABASE";"i"))) | with_entries(if (.key|test("PASS";"i")) then .value="***" else . end)'

# Count keys
... | jq 'keys | length'

# Check a key exists
... | jq 'has("SECRET_KEY_BASE")'
```

### 2.6 Compare two secrets without printing them

```bash
A=$(aws secretsmanager get-secret-value --secret-id qreport-vapt-app --region ap-southeast-2 --query SecretString --output text | jq -r .DB_PASSWORD | shasum)
B=$(aws secretsmanager get-secret-value --secret-id vapt-DB-Credentials --region ap-southeast-2 --query SecretString --output text | jq -r .MYSQL_ROOT_PASSWORD | shasum)
[ "$A" = "$B" ] && echo "✅ match" || echo "❌ different"
```
**Tip:** hashing (`shasum`) lets you compare secrets safely — the real values never appear on screen.

---

## 3. Terraform Fundamentals Used

### 3.1 Workspaces = environments

```bash
terraform workspace list          # * marks the current one
terraform workspace select vapt
terraform workspace show          # ALWAYS check before plan/apply
```

**Lesson:** `terraform workspace list` in `APP/sydney/` showed only `default`, because that folder isn't a Terraform root. Workspaces exist **per root module** — run inside `common-resources/` or `env-resources/`.

Your shell prompt shows the workspace (`tf vapt`, `tf staging`) — glance at it every time.

### 3.2 Where state lives (S3 backend)

```hcl
backend "s3" {
  bucket               = "sydney-qreport-terraform-resources"
  key                  = "terraform.tfstate"
  workspace_key_prefix = "common-resources"   # → common-resources/vapt/terraform.tfstate
  dynamodb_table       = "sydney-qreport-terraform-state-locks"
}
```
- Each workspace has its own state → applying `vapt` **cannot** touch production resources.
- `dynamodb_table` locks state during apply. Never `Ctrl+C` an apply — it can leave a stale lock.
- Warning `"dynamodb_table" is deprecated. Use "use_lockfile"` is harmless.

### 3.3 Plan → review → apply the saved plan

```bash
terraform plan -var-file=config/vapt.tfvars -out=stage.tfplan   # review!
terraform apply stage.tfplan                                    # applies EXACTLY what you reviewed
```
**Tips**
- A saved plan applies **without asking** "yes". So paste the `plan` command first, read it, then paste `apply` separately.
- Always check the summary: `X to add, 0 to change, 0 to destroy`. Any unexpected **change/destroy** → stop.
- Delete `*.tfplan` files afterwards — they can contain secrets (e.g. the generated DB password).

### 3.4 `-target` — build one piece at a time

```bash
terraform plan -var-file=config/vapt.tfvars \
  -target=module.vpc.aws_route_table_association.private_rt_assn \
  -out=01-network.tfplan
```
- Terraform also creates everything the target **depends on** — target the "last" resources in a chain to build the whole chain.
- Warnings "Resource targeting is in effect" / "Applied changes may be incomplete" are expected.
- **Always finish with a full, untargeted plan** — targeted applies may not update **outputs**, and other stacks (env-resources) read those outputs.

### 3.5 Reading remote state from another stack

```hcl
data "terraform_remote_state" "common_resources" {
  backend = "s3"
  config = { bucket = "...", key = "common-resources/${var.env_type}/terraform.tfstate", region = "ap-southeast-2" }
}
# use: data.terraform_remote_state.common_resources.outputs.vpc_id
```
env-resources used these outputs: `vpc_id`, `private_subnet_ids`, `lb_security_group_id`, `alb_listener_80_arn`, `alb_listener_443_arn`.

### 3.6 Code vs console (drift)

If you change something **manually in the console** that Terraform manages (desired count, launch template version), the next `terraform apply` **reverts it** to what the code says. Make permanent changes in `.tfvars`/code.

Examples from this build:
- ECS `desired_count` — change in `vapt.tfvars`, not the console.
- Prod's manual launch-template change (instance type/user data) isn't in code → a future prod apply would undo it.

### 3.7 Count/for_each toggle pattern (Redis on/off idea)

```hcl
variable "enable_redis" { type = bool  default = true }

resource "aws_elasticache_replication_group" "redis" {
  count = var.enable_redis ? 1 : 0
  ...
  subnet_group_name = aws_elasticache_subnet_group.redis_subnet_group[0].name
}

output "redis_primary_endpoint" {
  value = one(aws_elasticache_replication_group.redis[*].primary_endpoint_address)
}
```
Then `enable_redis = false` in `vapt.tfvars` only. (Not applied yet — see Pending.)

---

## 4. Pre-flight Checks Before Any Apply

### 4.1 Leftovers from last year (should all be "NotFound")

```bash
export AWS_PAGER=""
aws rds describe-db-instances --db-instance-identifier qreport-vapt --region ap-southeast-2
aws secretsmanager describe-secret --secret-id vapt-DB-Credentials --region ap-southeast-2
aws elasticache describe-replication-groups --replication-group-id qreport-vapt-redis --region ap-southeast-2
aws wafv2 list-ip-sets --scope REGIONAL --region ap-southeast-2 --query "IPSets[?contains(Name,'vapt')].Name"
```

### 4.2 Is the DB engine version offered?

```bash
aws rds describe-db-engine-versions --engine mysql --engine-version 8.4.8 --region ap-southeast-2 --query 'DBEngineVersions[].EngineVersion'
```

### 4.3 Config review — bugs we caught before applying

| File | Problem | Why it would fail | Fix |
|---|---|---|---|
| `common-resources/config/vapt.tfvars` | `mysql_version = "8.0.42"` but default param group `default.mysql8.4` | Engine/parameter-group family mismatch → RDS create fails (plan doesn't catch it) | `mysql_version = "8.4.8"` |
| `env-resources/config/vapt.tfvars` | `ec2_instance_type = "m5.x2large"` | Not a real type | `m5.2xlarge` |
| `common-resources/config/vapt.tfvars` | `custom_db_sg_rules` commented out | ✅ Correct — prod SG IDs belong to prod VPC; can't be referenced from VAPT VPC | Keep commented |
| `vpc_peering_routes`, `rds_parameter_group_name` | Prod-only | Prod peering / custom group | Leave out for VAPT |

**Tip:** `terraform plan` only checks syntax and what the API reveals at plan time. Things like "does this instance type exist" or "does this key pair exist" often fail only at **apply**.

---

## 5. EC2 Key Pairs — Concepts & Fixes

### 5.1 How key pairs work

| Half | Where | Used for |
|---|---|---|
| Public key | Stored in **EC2 → Key Pairs** (per region) | Copied into `~/.ssh/authorized_keys` on the server **at launch** |
| Private key (`.pem`) | **Your laptop** | Proves who you are when SSHing |

- Terraform only references the key **by name** (`key_name = "qreport-vapt-clamav"`).
- Key pairs are **separate objects** — `terraform destroy` does **not** delete them (they were created by hand).
- Replacing a key pair later doesn't change **running** servers (the key was copied at launch).

### 5.2 Check key pairs exist

```bash
aws ec2 describe-key-pairs --key-names qreport-vapt-clamav qreport-vapt-app --region ap-southeast-2 \
  --query 'KeyPairs[].{Name:KeyName,Created:CreateTime}' --output table
```

### 5.3 Does my local `.pem` match AWS?

AWS-**created** keys show a 40-hex-char SHA-1 fingerprint of the private key:
```bash
openssl pkcs8 -in key.pem -inform PEM -outform DER -topk8 -nocrypt | openssl sha1 -c
```
Universal method (works for any key) — compare public-key fingerprints:
```bash
K="/Users/jayamaran/Desktop/vapt pem key/qreport-vapt-clamav.pem"
echo -n "AWS  : "; aws ec2 describe-key-pairs --key-names qreport-vapt-clamav --include-public-key \
  --region ap-southeast-2 --query 'KeyPairs[0].PublicKey' --output text | ssh-keygen -lf - | awk '{print $2}'
echo -n "Local: "; ssh-keygen -y -f "$K" | ssh-keygen -lf - | awk '{print $2}'
```
**Clue we used:** local `.pem` files were dated **17 Sep 2025**, but AWS keys were created **6 Oct 2025** — a file can't contain a key created later → mismatch. Fingerprints confirmed it.

### 5.4 Replace AWS key pairs with your local keys (safely)

```bash
export AWS_PROFILE=q-report AWS_PAGER=""
cd ~/Downloads/qrport-pems
for k in qreport-vapt-app qreport-vapt-clamav; do
  chmod 400 "$k.pem" &&
  ssh-keygen -y -f "$k.pem" > "/tmp/$k.pub" &&
  aws ec2 delete-key-pair --key-name "$k" --region ap-southeast-2 &&
  aws ec2 import-key-pair --key-name "$k" --public-key-material "fileb:///tmp/$k.pub" --region ap-southeast-2 &&
  echo "✅ $k replaced" || echo "❌ $k failed"
done
```
**Why this order:** generate the public key **first**, delete only if that succeeded. `ssh-keygen` refuses keys with open permissions ("UNPROTECTED PRIVATE KEY FILE"), so `chmod 400` first — otherwise you'd delete the AWS key and fail to import.

**Tips**
- Only do this when **no running instances** depend on the old key and nobody else holds it (deletion is permanent).
- After import, AWS shows a different fingerprint format (MD5 of public key) — that's normal.
- ⚠️ Keep `.pem` files **out of Git repos** (our `~/Downloads/qrport-pems` was inside a git repo — risky).

---

## 6. Choosing the AMI

### 6.1 Inspect AMIs

```bash
aws ec2 describe-images --image-ids ami-0c462b53550d4fca8 ami-00fde50da249b5ff7 --region ap-southeast-2 \
  --query 'Images[].{Id:ImageId,Name:Name,Owner:OwnerId,Created:CreationDate,Deprecated:DeprecationTime,State:State}' --output table
```
| Owner | Meaning |
|---|---|
| `137112412989` | Amazon (stock image) |
| Your account `463949419568` | Custom image (may have software pre-installed) |

Findings: VAPT's AMI was an old AL2023 build (deprecated Dec 2025); prod's ClamAV runs **Amazon Linux 2**, which reached end of support (June 2026).

### 6.2 Get the latest Amazon Linux 2023 AMI

```bash
aws ssm get-parameter --region ap-southeast-2 \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' --output text
```
Result used: `ami-0720cb7af233b0529`.

**Tips**
- There is no "Amazon Linux 3" — the successor of AL2 is **AL2023**.
- Changing `ami` on an existing `aws_instance` → Terraform **destroys and recreates** the server (new IP, manual setup lost). Decide before the first apply.
- Prod AL2 → AL2023 upgrade = separate planned change; consider pinning `private_ip` so `CLAMAV_HOST` doesn't change.

---

## 7. Common-Resources — Staged Apply

Folder: `APP/sydney/common-resources`, workspace `vapt`.

### 7.1 Dependency chain

```
VPC ─► subnets ─► IGW / NAT / route tables
          │
          ├─► DB subnet group ─► RDS ─► DB secret
          ├─► Security groups ─► ClamAV EC2, Redis, ALB
          └─► S3 log bucket + WAF ─► ALB ─► listeners + WAF association
```

### 7.2 Why the doc's "DB Subnet Group error" happens

The VPC module reads subnets with a **data source**:
```hcl
data "aws_subnets" "private_subnets" { filter { name = "vpc-id" values = [aws_vpc.vpc.id] } tags = { Type = "private" } }
```
It depends only on the VPC, not on the subnets → on a first run it can read **0 subnets** → `InvalidSubnet: Subnet IDs are required`. **Fix:** create subnets first (stage 1), then the subnet group (stage 2).

### 7.3 The stages (commands)

Pattern for every stage:
```bash
date
terraform plan -var-file=config/vapt.tfvars <targets> -out=NN-name.tfplan   # review
date
terraform apply NN-name.tfplan
date
```

| # | Targets | Created |
|---|---|---|
| 1 Network | `module.vpc.aws_route_table_association.public_rt_assn`, `module.vpc.aws_route.ig_rule`, `module.vpc.aws_route_table_association.private_rt_assn` | VPC, 3 public + 3 private subnets, IGW, EIP, NAT, route tables (18) |
| 2 DB subnet group | `module.vpc.aws_db_subnet_group.db_subnet_group` | 1 |
| 3 Security groups | `aws_security_group.lb_sg`, `aws_security_group.db_sg`, `aws_security_group.clamav` (Redis SG skipped) | 3 |
| 4 S3 ALB logs | `aws_s3_bucket_policy.allow_access_from_alb`, `aws_s3_bucket_versioning.qreport-alb-logs_versioning`, `aws_s3_bucket_public_access_block.alb_block_public_access` | 4 |
| 5 WAF | `aws_wafv2_web_acl_logging_configuration.waf_logging` | 2 IP sets, rule group, web ACL, log group, logging (6) |
| 6 ALB | `aws_lb_listener.port_80`, `aws_lb_listener.port_443`, `aws_wafv2_web_acl_association.alb_association` | 4 |
| 7 ClamAV | `aws_instance.clamav` | 1 |
| 8 RDS | `aws_secretsmanager_secret_version.rds_credentials` | password, secret, DB, secret value (4) |
| 9 Redis | `aws_elasticache_replication_group.redis` | **Skipped** (decision pending) |
| 10 Final | full plan, no targets | Showed only 3 Redis resources left → 41 + 3 = 44 ✅ matches original plan |

### 7.4 Actual timeline (13 Oct 2026, IST)

| # | Task | Start | End | Duration | Result |
|---|---|---|---|---|---|
| 1 | Network | 11:05:19 | 11:07:30 | 2m 11s (NAT alone 1m 19s) | 18 added |
| 2 | DB subnet group | 11:36:05 | 11:36:36 | 31s | 1 added |
| 3 | Security groups | ~11:41 | 11:42:00 | < 43s (8s create) | 3 added |
| 4 | S3 ALB logs | 11:43:42 | 11:44:09 | 27s | 4 added |
| 5 | WAF | 11:46:35 | 11:46:59 | 24s | 6 added |
| 6 | ALB | 11:49:13 | 11:52:01 | 2m 48s (ALB alone 2m 30s) | 4 added |
| 7 | ClamAV EC2 | 11:57:12 | 11:57:43 | 31s | 1 added |
| 8 | RDS + secret | 11:59:37 | 12:11:50 | 12m 13s (RDS 11m 57s) | 4 added |
| | **Total provisioning** | | | **≈ 19m 50s** | **41 resources** |

**Observations**
- Slowest resources: **RDS (~12 min)**, **ALB (~2.5 min)**, **NAT Gateway (~1.3 min)**. Everything else takes seconds.
- Elapsed wall-clock (11:05 → 12:12) was ~67 min because of review time between stages — report both numbers and say which is which.
- S3 bucket name has a random suffix because the code uses `bucket_prefix` (bucket names are globally unique).

### 7.5 Where to see things in the console

| Resource | Console path |
|---|---|
| DB subnet group | **RDS → Subnet groups** (not VPC!) |
| Snapshots | RDS → Snapshots → Manual |
| WAF | WAF & Shield → Web ACLs (region Sydney) |
| ALB listeners/rules | EC2 → Load balancers → qreport-vapt-ALB → Listeners |

### 7.6 Warnings seen (all harmless)

- `vpc = true` deprecated on `aws_eip` → use `domain = "vpc"` (shared module; fix later).
- `aws_s3_bucket` inline encryption deprecated → use `aws_s3_bucket_server_side_encryption_configuration`.
- `dynamodb_table` deprecated → `use_lockfile`.

---

## 8. SSH into Private Servers via SSM

Servers are in **private subnets with no public IP** — plain `ssh user@ip` from a laptop can't reach them. **AWS Systems Manager (SSM)** tunnels SSH through AWS. No port 22 needs to be open.

### 8.1 One-time setup

```bash
brew install --cask session-manager-plugin
session-manager-plugin --version
```

### 8.2 Is the instance registered with SSM?

```bash
aws ssm describe-instance-information --region ap-southeast-2 \
  --filters Key=InstanceIds,Values=i-0d8bcf92c5230aa6f \
  --query 'InstanceInformationList[].{Id:InstanceId,Ping:PingStatus}' --output table
```
Needs: SSM agent (built into Amazon Linux), instance role with `AmazonSSMManagedInstanceCore`, outbound internet (NAT).

### 8.3 SSH through SSM

```bash
ssh -i "/Users/jayamaran/Desktop/vapt pem key/qreport-vapt-clamav.pem" \
  -o ProxyCommand="aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters portNumber=%p --region ap-southeast-2" \
  ec2-user@i-0d8bcf92c5230aa6f
```
- Use the **instance ID** as the host, user `ec2-user`.
- Quote paths with spaces.
- One-shot test: add `'echo "SSH OK as $(whoami) on $(hostname)"'` at the end.

### 8.4 Shortcut in `~/.ssh/config`

```
Host vapt-clamav
    HostName i-0d8bcf92c5230aa6f
    User ec2-user
    IdentityFile "/Users/jayamaran/Desktop/vapt pem key/qreport-vapt-clamav.pem"
    ProxyCommand aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters portNumber=%p --region ap-southeast-2
```
Then `ssh vapt-clamav`.

### 8.5 Copy files from a server to your Mac (run on the **Mac**)

```bash
scp -i "/Users/jayamaran/Desktop/vapt pem key/qreport-vapt-app.pem" \
  -o ProxyCommand="aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters portNumber=%p --region ap-southeast-2" \
  ec2-user@i-0622582167146e466:~/file.zip ~/Desktop/
```

### 8.6 Browser alternative

EC2 → Instances → select → **Connect → Session Manager**. You become `ssm-user`; use `sudo -i` for root.

**Note:** AWS CLI **on a server** uses the server's IAM role, not your user — e.g. `rds:DescribeDBInstances` was denied on the APP server. Run AWS lookups from your Mac.

---

## 9. ClamAV Server Setup & Testing

### 9.1 Why manual

`clamav.tf` has **no user data** → Terraform launches a bare AL2023 server. ClamAV must be installed by hand (doc Step 5).

### 9.2 Install & configure

```bash
sudo -i
dnf update -y
dnf install -y clamav1.4 clamd1.4
freshclam                                   # download virus database
cp /usr/lib/systemd/system/clamd@.service /etc/systemd/system/clamd.service
```

**Edit 1 — service file:** `%i` only works for template units (`clamd@scan.service`); our copy is `clamd.service`, so replace it:
```bash
sed -i 's|%i.conf|scan.conf|' /etc/systemd/system/clamd.service
grep ExecStart /etc/systemd/system/clamd.service
# ExecStart = /usr/sbin/clamd -c /etc/clamd.d/scan.conf
```

**Edit 2 — scan.conf:**
```bash
sed -i 's/^User clamscan/User root/' /etc/clamd.d/scan.conf
sed -i 's/^#TCPSocket 3310/TCPSocket 3310/' /etc/clamd.d/scan.conf
grep -nE '^#?(User|TCPSocket|TCPAddr|LocalSocket)' /etc/clamd.d/scan.conf
grep -n '^Example' /etc/clamd.d/scan.conf     # must print nothing (an active Example line blocks startup)
```
Keep `#TCPAddr localhost` **commented** → clamd listens on all interfaces so app servers can reach it.

**Start:**
```bash
systemctl daemon-reload
systemctl enable clamd.service --now
systemctl status clamd.service --no-pager
ss -tlnp | grep 3310          # expect 0.0.0.0:3310
systemctl enable clamav-freshclam.service --now   # keep virus definitions updated
```

**Navigation lessons:** `cd /etc/systemmd` (typo) → `systemd`; `cd scan.conf` fails because it's a **file** — use the full path with `sed`/`vi`.

### 9.3 Testing

```bash
# Ping / version over TCP (exactly how the app talks to it)
exec 3<>/dev/tcp/10.5.3.144/3310; printf 'nPING\n' >&3; cat <&3; exec 3>&-      # PONG
exec 3<>/dev/tcp/10.5.3.144/3310; printf 'nVERSION\n' >&3; cat <&3; exec 3>&-   # ClamAV 1.4.6/...
```

**EICAR test** (harmless standard "virus" string). Use base64 so copy-paste can't corrupt it:
```bash
echo 'WDVPIVAlQEFQWzRcUFpYNTQoUF4pN0NDKTd9JEVJQ0FSLVNUQU5EQVJELUFOVElWSVJVUy1URVNULUZJTEUhJEgrSCo=' | base64 -d > /tmp/eicar.txt
md5sum /tmp/eicar.txt     # must be 44d88612fea8a8f36de82e1278abb02f
```
Scan via the INSTREAM protocol (what the app does):
```bash
python3 - <<'EOF'
import socket, struct
data = open('/tmp/eicar.txt', 'rb').read()
s = socket.create_connection(('10.5.3.144', 3310))
s.sendall(b'zINSTREAM\0' + struct.pack('!L', len(data)) + data + struct.pack('!L', 0))
print(s.recv(4096))
EOF
rm -f /tmp/eicar.txt
# expect: b'stream: Eicar-Test-Signature FOUND\x00'
```

**Lessons**
- Our first EICAR string was missing a `$` (`...FILE!$H+H*`) → `stream: OK` was the **correct** answer for a non-virus. Always verify the md5 first.
- File size clue: correct EICAR = 68 bytes (+1 newline with `echo`).
- `clamdscan` client showed "Total errors" with no message — client config issue, not clamd. The app uses TCP, so test with INSTREAM.
- ClamAV private IP `10.5.3.144` → goes into CICD `CLAMAV_HOST`.

---

## 10. Env-Resources — Staged Apply

Folder: `APP/sydney/env-resources`, workspace `vapt`.

### 10.1 Stages

| # | Targets | Created |
|---|---|---|
| E1 IAM | `aws_iam_instance_profile.ecsInstanceProfile`, the 4 `aws_iam_role_policy_attachment.*` | 4 roles, 11 attachments, 1 profile (16) |
| E2 ECR/logs/secret | `aws_ecr_repository.ecr_repo`, `aws_cloudwatch_log_group.log_group`, `aws_secretsmanager_secret.secrets_manager` | 3 (secret is **empty**) |
| E3 SG | `aws_security_group.ecs_instance_sg` | 1 |
| E4 Cluster/ASG | `aws_autoscaling_group.auto_scaling_group` | cluster, launch template, ASG (3) |
| E5 TG + rules | `aws_lb_listener_rule.rules_in_lb_80`, `aws_lb_listener_rule.rules_in_lb_443` | TG + 2 rules (3) |
| E6 Services | `module.qreport_ecs_service` | 2 task defs + 2 services (4) |
| E7 Final | full plan | "No changes" ✅ (30 resources) |

**Ordering trick:** the doc says "APP service may fail on first apply". Cause: an ECS service with a load balancer requires its target group to be attached to a listener rule. Creating rules (E5) **before** services (E6) avoided it.

### 10.2 Timeline (13 Oct 2026, IST)

| # | Task | Start | End | Duration |
|---|---|---|---|---|
| E1 | IAM | 14:33:23 | 14:33:38 | 15s |
| E2 | ECR/logs/secret | 14:36:18 | 14:36:31 | 13s |
| E4 | Cluster/LT/ASG | ~14:41:19 | 14:42:01 | 42s (+ manual ECS fix, joined 14:49:47) |
| E5 | TG + rules | 14:54:22 | 14:54:40 | 18s |
| E6 | Task defs + services | ~15:00:22 | 15:01:00 | < 38s |
| E8 | User-data fix + instance replacement | ~15:14 | ~15:25 | ✅ joined automatically |

### 10.3 Launch template facts

- Terraform creates it (`qreport-vapt`, version 1, then 2, 3 after fixes).
- AMI = latest **ECS-optimized AL2023** (`data.aws_ami.ecs_ami`).
- ASG uses `version = "$Latest"`; instance refresh triggers only on **tag** changes → a new LT version does **not** replace running instances automatically.
- ASG min = desired = max = 1 → always exactly one instance.

### 10.4 Useful checks

```bash
# Is the instance registered in the cluster?
aws ecs describe-clusters --clusters qreport-vapt --region ap-southeast-2 --query 'clusters[].registeredContainerInstancesCount' --output text

# Which instance, agent connected?
aws ecs describe-container-instances --cluster qreport-vapt --region ap-southeast-2 \
  --container-instances $(aws ecs list-container-instances --cluster qreport-vapt --region ap-southeast-2 --query 'containerInstanceArns' --output text) \
  --query 'containerInstances[].{EC2:ec2InstanceId,Status:status,Agent:agentConnected}' --output table

# ASG instances & capacity
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names qreport-vapt --region ap-southeast-2 \
  --query 'AutoScalingGroups[].{Min:MinSize,Desired:DesiredCapacity,Max:MaxSize,Instances:Instances[].[InstanceId,LifecycleState]}' --output json

# ASG history (launch/terminate events)
aws autoscaling describe-scaling-activities --auto-scaling-group-name qreport-vapt --region ap-southeast-2 --max-items 5 \
  --query 'Activities[].[StartTime,StatusCode,Description]' --output table

# Which LT version did an instance use?
aws ec2 describe-instances --instance-ids <id> --region ap-southeast-2 \
  --query 'Reservations[].Instances[].Tags[?Key==`aws:ec2launchtemplate:version`].Value' --output text
```
"Container instance cannot be empty" error = the inner `list-container-instances` returned nothing → 0 instances registered.

---

## 11. ECS User-Data Problems

### 11.1 Problem 1 — the boot deadlock

Original `userdata.sh` ended with:
```bash
systemctl daemon-reexec
systemctl enable ecs
systemctl start ecs      # ← deadlock
```
On the AL2023 ECS AMI, `ecs.service` waits for cloud-init to finish, while cloud-init (running this script) waits for `ecs.service` to start → neither proceeds.

**Diagnosis on the instance:**
```bash
cat /etc/ecs/ecs.config                  # ECS_CLUSTER=qreport-vapt ✅
sudo systemctl status ecs --no-pager     # inactive (dead)
sudo cloud-init status                   # status: running (stuck)
sudo systemctl list-jobs                 # cloud-final running, ecs.service waiting
```
**Unblock without reboot:**
```bash
sudo pkill -f "systemctl start ecs"      # cloud-init finishes → ecs starts
```
(cloud-init then shows `status: error` — harmless, from the killed command.)

### 11.2 Problem 2 — blank lines before the shebang

After switching to the prod-style script, a new instance **still** didn't join. `cat -vet userdata.sh` showed two empty lines before `#!/bin/bash`. cloud-init only runs user data as a script if the **first characters are `#!`** → script skipped → no `ecs.config` → agent joined the **`default`** cluster.

```bash
cat -vet userdata.sh          # macOS: use -vet, not -A (shows $ at line ends)
sed -i '' '/./,$!d' userdata.sh   # delete leading blank lines (macOS sed)
head -2 userdata.sh           # first line must be #!/bin/bash
```

### 11.3 Final `userdata.sh`

```bash
#!/bin/bash
set -ex

# 1. Configure ECS cluster settings FIRST (before running any package installs)
cat << 'EOF' >> /etc/ecs/ecs.config
ECS_CLUSTER=${ECS_CLUSTER_NAME}
ECS_IMAGE_PULL_BEHAVIOR=always
ECS_ENABLE_CONTAINER_METADATA=true
ECS_LOGLEVEL=info
EOF

# 2. Install MariaDB / MySQL client non-interactively
dnf install -y mariadb105
```
- `${ECS_CLUSTER_NAME}` is filled by Terraform's `templatefile()` **before** bash runs, so it works even inside `<< 'EOF'`.
- No `systemctl` lines — the ECS AMI starts the agent itself.
- No `dnf update` — slower boot and can upgrade Docker/agent mid-boot.
- `set -ex` → stop on errors, log each command to `/var/log/cloud-init-output.log`.

### 11.4 Rolling out a fix

```bash
terraform plan -var-file=config/vapt.tfvars -out=env-userdata.tfplan   # 1 to change: launch template
terraform apply env-userdata.tfplan                                    # new LT version
aws autoscaling terminate-instance-in-auto-scaling-group --instance-id <id> --no-should-decrement-desired-capacity --region ap-southeast-2
```
**Trap we hit:** an instance refresh with `min_healthy_percentage = 100` considered the broken new instance "healthy" (ASG health type = EC2 only, not ECS registration) and **terminated the good old one**. Watch refreshes carefully; cancel if needed:
```bash
aws autoscaling cancel-instance-refresh --auto-scaling-group-name qreport-vapt --region ap-southeast-2
```

---

## 12. Testing the ALB & WAF

### 12.1 Test without DNS (fake the hostname)

```bash
UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0 Safari/537.36"
ALB=qreport-vapt-ALB-652950571.ap-southeast-2.elb.amazonaws.com
curl -sk -o /dev/null -w "%{http_code}\n" -A "$UA" -H "Host: test-vapt.qreport.com.au" https://$ALB/
curl -sk -o /dev/null -w "%{http_code}\n" -A "$UA" -H "Host: something-else.com" https://$ALB/
```
| Code | Meaning |
|---|---|
| 503 | Rule matched, no healthy targets yet (expected before app runs) |
| 404 (plain text) | No rule matched → ALB default action |
| 403 | **WAF blocked** |
| App-branded 404 | App reached, but that path/host isn't valid in the app |

### 12.2 Why `curl` got 403

The web ACL includes **AWS Bot Control**, which blocks HTTP-library User-Agents (`curl/…`). Sending a browser User-Agent passed. **Tell testers:** scanners (Burp, ZAP, Nessus) will be blocked the same way — decide WAF on/off or allow-list their IPs.

### 12.3 WAF logs (delivered with ~5 min delay)

```bash
aws logs filter-log-events --log-group-name aws-waf-logs-qreport-vapt --region ap-southeast-2 --limit 5 --output json \
  | jq -r '.events[].message | fromjson | [.action, .terminatingRuleId] | @tsv'
```

### 12.4 Get the ALB DNS name

```bash
aws elbv2 describe-load-balancers --names qreport-vapt-ALB --region ap-southeast-2 --query 'LoadBalancers[].DNSName' --output text
```

⚠️ The `*.qreport.com.au` certificate showed **"expires in 45 days"** (~28 Nov 2026) — likely shared with prod. Renew & re-import in time.

---

## 13. Connectivity Validation

(Doc section 6 — run from the APP server.)

| Check | Command | Result |
|---|---|---|
| App → RDS 3306 | `timeout 5 bash -c "</dev/tcp/$DB/3306" && echo OPEN` then `mysql -h $DB -u admin -p -e 'SELECT VERSION(), CURRENT_USER();'` | ✅ OPEN, 8.4.8, admin@% |
| App → ClamAV 3310 | `exec 3<>/dev/tcp/10.5.3.144/3310; printf 'nPING\n' >&3; cat <&3; exec 3>&-` | ✅ PONG |
| ALB → App (SG rule) | `aws ec2 describe-security-groups --group-ids sg-01293abe7354c9fd4 --query 'SecurityGroups[].IpPermissions[].UserIdGroupPairs[].GroupId' --output text` | ✅ `sg-03a53ec66968bb7d9` (ALB SG) |
| ALB → App (health) | `aws elbv2 describe-target-health --target-group-arn <tg-arn> --query 'TargetHealthDescriptions[].TargetHealth.State' --output text` | `healthy` after deploy |

**Tip:** `/dev/tcp/HOST/PORT` is a bash built-in — a quick port test when `nc`/`telnet` aren't installed. Note the doc says "ClamAV → App", but really the **app connects to ClamAV**.

---

## 14. Secrets Manager — The Two Secrets

| Secret | Created by | Filled by | Contents |
|---|---|---|---|
| `vapt-DB-Credentials` | common-resources (stage 8) | **Terraform automatically** | MYSQL_ROOT_USER, MYSQL_ROOT_PASSWORD (random), DB_HOST (`host:3306`), DB_PORT |
| `qreport-vapt-app` | env-resources (E2) | **You + dev team** | All app settings (DB, third-party keys, URLs, SECRET_KEY_BASE…) |

### 14.1 How the app uses `qreport-vapt-app`

- **Build time:** `buildspec.yml` reads some keys → empty secret made CodeBuild fail with `can't find the specified secret value for staging label: AWSCURRENT`.
- **Run time:** the task definition passes only `SECRET_ID=qreport-vapt-app`; the container's `startup.sh` reads **all** keys at boot. → Change a value later? Just **force a new deployment**, no rebuild.

### 14.2 Build a template from prod (keys only)

```bash
aws secretsmanager get-secret-value --secret-id qreport-production-app --region ap-southeast-2 \
  --query SecretString --output text | jq 'with_entries(.value = "")' > ~/Desktop/vapt-app-secret-template.json
```
Fill DB values yourself; dev team fills third-party keys with **test/sandbox** values (never prod keys in VAPT).

### 14.3 Save it

Console: Secrets Manager → secret → Retrieve secret value → Edit → **Plaintext** → paste JSON → Save.
CLI:
```bash
jq . file.json                       # validates JSON first
aws secretsmanager put-secret-value --secret-id qreport-vapt-app --region ap-southeast-2 --secret-string file://$HOME/Desktop/file.json
rm file.json
```
Format: `{"KEY":"value","KEY2":"value2"}` — every key and value in double quotes.

### 14.4 Fix DB password + add SECRET_KEY_BASE without printing anything

```bash
DBPASS=$(aws secretsmanager get-secret-value --secret-id vapt-DB-Credentials --region ap-southeast-2 --query SecretString --output text | jq -r .MYSQL_ROOT_PASSWORD)
NEW=$(aws secretsmanager get-secret-value --secret-id qreport-vapt-app --region ap-southeast-2 --query SecretString --output text \
  | jq -c --arg p "$DBPASS" --arg k "$(openssl rand -hex 64)" '.DB_PASSWORD=$p | .SECRET_KEY_BASE //= $k')
aws secretsmanager put-secret-value --secret-id qreport-vapt-app --region ap-southeast-2 --secret-string "$NEW" --query VersionId --output text
unset DBPASS NEW
```
- `//=` in jq = "set only if missing".
- **SECRET_KEY_BASE** = random text Rails uses to sign cookies/sessions. Generate anywhere (`openssl rand -hex 64`); it isn't tied to a machine. Use a **new** value for VAPT; changing it later logs everyone out.
- `DB_HOST` in the app secret must be the hostname **without** `:3306`.

---

## 15. Database Part 1 — Old Snapshot & Sharing a Dump

### 15.1 About snapshots

- `final-qreport-vapt-d7b9ddca` = **manual** RDS snapshot from **13 Dec 2022** (MySQL 8.0.28) — likely the final snapshot when last VAPT's DB was deleted.
- Stored in AWS-managed storage (not your S3 buckets). Manual snapshots never expire.
- You **can't download** a snapshot file — restore it to a DB, then dump.

```bash
aws rds describe-db-snapshots --db-snapshot-identifier final-qreport-vapt-d7b9ddca --region ap-southeast-2 \
  --query 'DBSnapshots[].{Created:SnapshotCreateTime,Engine:EngineVersion,SizeGB:AllocatedStorage,Status:Status}' --output table
```

### 15.2 Restore → temporary DB

```bash
aws rds restore-db-instance-from-db-snapshot --region ap-southeast-2 \
  --db-instance-identifier qreport-vapt-restore-tmp \
  --db-snapshot-identifier final-qreport-vapt-d7b9ddca \
  --db-instance-class db.m5.large \
  --db-subnet-group-name qreport-vapt-private_subnet_db_group \
  --vpc-security-group-ids sg-0eb26e363b5ef2e97 \
  --no-publicly-accessible --no-multi-az --query 'DBInstance.DBInstanceStatus' --output text
aws rds wait db-instance-available --db-instance-identifier qreport-vapt-restore-tmp --region ap-southeast-2
aws rds modify-db-instance --db-instance-identifier qreport-vapt-restore-tmp --master-user-password '<TEMP_PASSWORD>' --apply-immediately --region ap-southeast-2
```
**What happened:** RDS auto-**upgraded** the retired 8.0.28 → **8.0.35** (status `upgrading`, then `backing-up`). The dump failed with error 2002 (connection timed out) during the upgrade. Wait until `available`:
```bash
aws rds describe-db-instances --db-instance-identifier qreport-vapt-restore-tmp --region ap-southeast-2 \
  --query 'DBInstances[].{Status:DBInstanceStatus,Version:EngineVersion}' --output table
```

### 15.3 Dump with progress, zip with password, copy to Mac

On the APP server:
```bash
OLD=qreport-vapt-restore-tmp.c5g4ncau2qu6.ap-southeast-2.rds.amazonaws.com
mysql -h $OLD -u admin -p -e 'SHOW DATABASES;'
sudo dnf install -y pv zip
mysqldump -h $OLD -u admin -p --single-transaction --routines --triggers --databases qreport_vapt | pv > ~/vapt_2022_dump.sql
tail -1 ~/vapt_2022_dump.sql          # "-- Dump completed on ..."
zip -e ~/vapt_2022_dump.zip ~/vapt_2022_dump.sql   # asks for a password twice
```
Result: 2.7 GB SQL in ~47 s → 345 MB zip. Copy with `scp` **from the Mac** (section 8.5).

**Tips**
- `mysqldump` prints nothing while working and the password prompt shows no characters — use `pv` to see progress.
- `Ctrl+C` on `mysqldump` is safe (read-only); delete the partial file and rerun.
- Share the zip password through a **different channel**.
- An S3 pre-signed link is a safer alternative (expires): `aws s3 presign s3://bucket/key --expires-in 86400`.

### 15.4 Cleanup & verification

```bash
rm ~/vapt_2022_dump.sql ~/vapt_2022_dump.zip                     # server
ls -lh ~/vapt_2022_dump.* 2>/dev/null || echo "✅ none"
aws rds delete-db-instance --db-instance-identifier qreport-vapt-restore-tmp --skip-final-snapshot --delete-automated-backups --region ap-southeast-2
aws rds describe-db-instances --db-instance-identifier qreport-vapt-restore-tmp --region ap-southeast-2   # DBInstanceNotFound = deleted
```

---

## 16. Database Part 2 — Masking with AWS Batch

### 16.1 Never load raw prod data into VAPT

VAPT is built to be attacked, often by a third party (CyberCX). Real customer data there = data-protection risk. Pipeline:

```
prod RDS ─(every 3h backup job)─► s3://qreport-prod-dbbackup/DB-BACKUP/mysqlbackup-DD-MM-YYYY-H:0:8.sql.gz  (RAW)
                                          │  AWS Batch: mysql-data-obfuscator
                                          ▼
                           s3://qreport-sydney-prod-scrubbed-db-dumps/<UTC-ts>_obfuscated_dump.sql.gz  (MASKED)
                                          │  download on APP server → mysql
                                          ▼
                                   qreport-vapt RDS
```

### 16.2 Discover the pieces

```bash
aws s3 ls s3://qreport-prod-dbbackup/DB-BACKUP/ --human-readable | sort | tail -5          # raw dumps (times local; file names UTC)
aws s3 ls s3://qreport-sydney-prod-scrubbed-db-dumps/ --human-readable | sort | tail -5     # masked dumps
aws batch describe-job-queues --region ap-southeast-2 --query 'jobQueues[].{Queue:jobQueueName,State:state,Status:status,CE:computeEnvironmentOrder[0].computeEnvironment}' --output table
aws batch describe-compute-environments --region ap-southeast-2 --query 'computeEnvironments[].{Name:computeEnvironmentName,State:state,Status:status}' --output table
aws batch describe-job-definitions --job-definition-name prod-mysql-db-dump-obfuscator-job-definition --status ACTIVE --region ap-southeast-2
```
- Queue: `prod-mysql-db-dump-obfuscator-job-queue` → CE `batch-prod-al2023` (EC2).
- Job definition: `prod-mysql-db-dump-obfuscator-job-definition:4`, image `mysql-data-obfuscator:latest`, env `S3_DUMP_FOLDER_URI`, `DESTINATION_BUCKET_NAME`.

### 16.3 Check the job's permissions (least privilege)

```bash
aws iam list-attached-role-policies --role-name prod-mysql-db-dump-obfuscator-role --output table
aws iam list-role-policies --role-name prod-mysql-db-dump-obfuscator-role --output table
aws iam get-role-policy --role-name prod-mysql-db-dump-obfuscator-role --policy-name qreport-sydney-prod-scrubbed-db-dumps-bucket-access --query PolicyDocument --output json
```
Input bucket: `GetObject`/`ListBucket` only → the job **cannot damage** raw dumps. Output: `PutObject`.

### 16.4 Look inside the container image (without running it)

```bash
aws ecr get-login-password --region ap-southeast-2 | docker login --username AWS --password-stdin 463949419568.dkr.ecr.ap-southeast-2.amazonaws.com
IMG=463949419568.dkr.ecr.ap-southeast-2.amazonaws.com/mysql-data-obfuscator:latest
docker pull $IMG
docker inspect $IMG --format 'Entrypoint: {{.Config.Entrypoint}}  Cmd: {{.Config.Cmd}}'
docker run --rm -it --platform linux/amd64 --entrypoint sh $IMG     # opens a shell; nothing runs
# inside:  ls -la /opt ; cat /opt/mysql_data_obfuscator.sh ; cat /opt/shard.yml
docker run --rm --platform linux/amd64 --entrypoint cat $IMG /opt/obfuscator.cr > ~/vapt-logs/obfuscator.cr
```
- `--entrypoint` overrides the start command → safe inspection.
- Platform warning on Apple Silicon (amd64 vs arm64) is harmless.
- Image last pushed **22 Dec 2023** → masking rules are ~3 years old.

### 16.5 What the script does

`/opt/mysql_data_obfuscator.sh`:
1. If `S3_DUMP_FILE_URI` set → download that file; else list `S3_DUMP_FOLDER_URI`, `sort | tail -1` → **newest** file.
2. `gunzip` → `crystal obfuscator.cr < dump.sql > <UTC-timestamp>_obfuscated_dump.sql` → `gzip` → upload.
3. Every step logs `$(date) ...` → exact timings in CloudWatch.

### 16.6 Masking rules (Triki config in `obfuscator.cr`)

| Rule | Effect | Example |
|---|---|---|
| `:keep` | Unchanged (real) | username, password, tokens |
| `:null` | NULL | phone, secondary_email |
| `{ :type => :fixed, :string => "…" }` | Fixed fake value | email → `xxxxx@yyy.com` |
| Lambda `row[...][0,2] + "******"` | First 2 letters kept | `Jonathan` → `Jo******` |

Turn the config into a list:
```bash
awk '/^  "[a-z_0-9]+" => \{/{t=$1; gsub(/"/,"",t)} /^    "[a-z_0-9]+" +=>/{c=$1; gsub(/"/,"",c); a=($0~/:keep/)?"KEEP":($0~/:null/)?"NULL":"MASKED"; print t"."c, a}' \
  ~/vapt-logs/obfuscator.cr > ~/vapt-logs/mask-rules.txt
grep -v KEEP ~/vapt-logs/mask-rules.txt                                       # masked columns
grep KEEP ~/vapt-logs/mask-rules.txt | grep -iE 'email|phone|name|addr|token|secret|password|card|ip_address' # sensitive but kept
```
Tables **not in the config** are passed through unchanged with log warnings `… was not specified in the config`:
```bash
grep -oE 'Deprecated: [a-z_0-9]+ was not specified' ~/vapt-logs/masking-job-log.txt | awk '{print $2}' | sort -u
```
Gaps found: `confirm_email`, `full_name`, `card_name`, `preferred_phone_number`, `username`, `password`, API/payment tokens, `ip_address`; unconfigured tables like `conversations`, `google_credentials`, `*_tokens`, `payment_histories`. → **Decision taken: no additional masking for this VAPT** (documented decision; raise for next year to update `obfuscator.cr`).

### 16.7 Run the job (pin the exact dump)

```bash
aws s3 ls s3://qreport-prod-dbbackup/DB-BACKUP/ | sort | tail -1     # latest raw dump
date
aws batch submit-job --job-name vapt-obfuscate-$(date +%Y%m%d-%H%M) \
  --job-queue prod-mysql-db-dump-obfuscator-job-queue \
  --job-definition prod-mysql-db-dump-obfuscator-job-definition:4 --region ap-southeast-2 \
  --container-overrides '{"environment":[{"name":"S3_DUMP_FILE_URI","value":"s3://qreport-prod-dbbackup/DB-BACKUP/mysqlbackup-14-10-2026-3:0:8.sql.gz"}]}'
```
Our run: job `vapt-obfuscate-20261014-0912`, id `670fe12b-c253-4050-9d72-c8a9f41deeed`.

### 16.8 Monitor & collect evidence

```bash
JOB=670fe12b-c253-4050-9d72-c8a9f41deeed
aws batch describe-jobs --jobs $JOB --region ap-southeast-2 --query 'jobs[].{Status:status,Reason:statusReason,Log:container.logStreamName}' --output table
# SUBMITTED → RUNNABLE (EC2 launching, 3-8 min) → STARTING → RUNNING → SUCCEEDED

# Readable timeline (epoch ms → IST)
aws batch describe-jobs --jobs $JOB --region ap-southeast-2 --output json | jq -r '.jobs[] |
 "Submitted: \(.createdAt/1000 + 19800 | todate)\nStarted:   \(.startedAt/1000 + 19800 | todate)\nFinished:  \(.stoppedAt/1000 + 19800 | todate)\nRun time:  \((.stoppedAt - .startedAt)/60000 | floor) minutes"'

# Full log
LS=$(aws batch describe-jobs --jobs $JOB --region ap-southeast-2 --query 'jobs[0].container.logStreamName' --output text)
aws logs get-log-events --log-group-name /aws/batch/job --log-stream-name "$LS" --start-from-head --region ap-southeast-2 \
  --query 'events[].message' --output text | tr '\t' '\n' > ~/vapt-logs/masking-job-log.txt
```
- RUNNABLE for a while + compute environment `Desired vCPUs` going up = EC2 is launching (normal).
- `get-log-events` without `--start-from-head` can return **empty** — always add it.
- Log group `/aws/batch/job` retention = **Never expire** — past runs' logs (Feb/Apr/Aug 2026) are still there.
- `jq ... + 19800` shifts UTC to IST (output still shows `Z`, but the time is IST).

**Our masking timeline (IST):** submitted 09:12:21 → started 09:14:13 → masking done 10:38:24 → compressed 10:40:43 → uploaded 10:40:49 → **86 minutes**. Output: `2026_10_14_03_44_13_obfuscated_dump.sql.gz` (683 MB).

---

## 17. Database Part 3 — Loading the Masked Dump

Run on the APP server inside `tmux`:
```bash
sudo dnf install -y tmux pv
tmux new -s dbload           # detach: Ctrl+b then d ; reattach: tmux attach -t dbload
F=2026_10_14_03_44_13_obfuscated_dump.sql.gz
DB=qreport-vapt.c5g4ncau2qu6.ap-southeast-2.rds.amazonaws.com
aws s3 cp s3://qreport-sydney-prod-scrubbed-db-dumps/$F ~/          # 683 MB in ~3s
zcat ~/$F | grep -m3 -E '^(CREATE DATABASE|USE )'                    # nothing = loads into the DB you name
```

### 17.1 Final working load command

```bash
date
zcat ~/$F | grep -v 'triki: Deprecated' | sed -E 's/DEFINER=`[^`]+`@`[^`]+`//g' | pv | mysql -h $DB -u admin -p qreport_vapt
date
```

| Filter | Why |
|---|---|
| `sed … DEFINER=…` | Views/routines had `DEFINER=<prod user>`. On RDS, `admin` lacks SUPER → `ERROR 1227 … SUPER or ALLOW_NONEXISTENT_DEFINER`. Removing DEFINER makes the loader the owner |
| `grep -v 'triki: Deprecated'` | The obfuscator wrote its **warning lines into the dump file** → `ERROR 1064 … near '2026-10-14T03:45:11Z WARN - triki'`. (Bug to report to dev team) |

Re-running the whole load is safe — the dump drops and recreates each table.

### 17.2 Verify

```bash
zcat ~/$F | grep -c '^CREATE TABLE'        # 176
zcat ~/$F | grep -c 'VIEW `'              # 6
mysql -h $DB -u admin -p -e "SELECT table_type, COUNT(*) n FROM information_schema.tables WHERE table_schema='qreport_vapt' GROUP BY table_type;"
# BASE TABLE 176, VIEW 6  ✅ matches
rm ~/$F
```

### 17.3 Snapshot for next year

```bash
aws rds create-db-snapshot --db-instance-identifier qreport-vapt --db-snapshot-identifier qreport-vapt-masked-20261014 --region ap-southeast-2
aws rds describe-db-snapshots --db-snapshot-identifier qreport-vapt-masked-20261014 --region ap-southeast-2 \
  --query 'DBSnapshots[].{Status:Status,Progress:PercentProgress}' --output table
```
(`SizeGB 100` = allocated disk, not data size. First snapshot is a full copy → slower.)

### 17.4 Load timeline (14 Oct, IST)

| Task | Start | End | Result |
|---|---|---|---|
| Download masked dump | 12:31:08 | 12:31:11 | 3 s |
| Load attempt 1 | 12:33:29 | 12:50:36 | ❌ ERROR 1227 (DEFINER) |
| Load attempt 2 | ~12:5x | 13:12:30 | ✅ data complete; stopped at triki text at end |
| Verify | — | — | 176 tables, 6 views ✅ |

---

## 18. CI/CD — CodePipeline & CodeBuild

### 18.1 Structure

- `CICD/common-resources` — **shared** by all environments (IAM roles with `*` + PowerUserAccess, artifact bucket). State in Singapore bucket. **Never apply for VAPT.**
- `CICD/env-specific/sydney` — per-workspace CodeBuild project + CodePipeline.

### 18.2 What we did

```bash
cd CICD/env-specific/sydney
terraform workspace select vapt          # it was on staging!
# vapt.tfvars: CLAMAV_HOST="10.5.3.144"
terraform plan -var-file=config/vapt.tfvars \
  -target=aws_codepipeline.qreport-codepipeline \
  -target=aws_codestarnotifications_notification_rule.qreport-codebuild-notification \
  -target=aws_sns_topic_policy.codestar-sns -out=cicd-vapt.tfplan
terraform apply cicd-vapt.tfplan
```
- Pipeline **already existed from last year** → plan showed only `CLAMAV_HOST 10.5.3.117 → 10.5.3.144`.
- `-target` used to **exclude** an uncommitted `codebuild-shopify.tf` (someone's staging work) that would have created a Shopify build for VAPT.
- Pipeline = Source (GitHub `vapt` branch) → Build (CodeBuild pushes to ECR). No separate Deploy stage.
- Existing pipeline doesn't auto-run → **Release change** in console, or:
  ```bash
  aws codepipeline start-pipeline-execution --name vapt-qreport --region ap-southeast-2
  aws codepipeline get-pipeline-state --name vapt-qreport --region ap-southeast-2 --query 'stageStates[].{Stage:stageName,Status:latestExecution.status}' --output table
  ```

### 18.3 Reading CodeBuild logs

```bash
B=$(aws codebuild list-builds-for-project --project-name vapt-qreport --region ap-southeast-2 --query 'ids[0]' --output text)
aws codebuild batch-get-builds --ids $B --region ap-southeast-2 --query 'builds[0].logs.[groupName,streamName]' --output text
aws logs filter-log-events --log-group-name /aws/codebuild/vapt-qreport --log-stream-names <stream> \
  --filter-pattern '?"rake aborted" ?Error ?error ?ERROR' --region ap-southeast-2 --query 'events[].message' --output text | head -40
```

### 18.4 Build failures & fixes

| Error | Meaning | Fix | Owner |
|---|---|---|---|
| `Secrets Manager can't find the specified secret value for staging label: AWSCURRENT` | `qreport-vapt-app` exists but is empty | Fill the secret | You |
| `NameError: uninitialized constant ENV_PATH::PORTAL_URL` (during `rake assets:precompile`) | App code has no `PORTAL_URL` for `vapt` env | Add VAPT values in `vapt` branch | Dev team |
| `docker push … image does not exist` | Consequence of failed build | — | — |

Success check:
```bash
aws ecr list-images --repository-name qreport-vapt --region ap-southeast-2 --output table
# latest + vapt-2026-10-14.08.12.16 (same digest)
```

---

## 19. Starting the App on ECS & Troubleshooting

### 19.1 Scale up (via Terraform)

In `env-resources/config/vapt.tfvars` set `desired_count = 1` for `qreport-app` and `qreport-dj`, then plan/apply (expect only `desired_count` changes). Quick alternative:
```bash
aws ecs update-service --cluster qreport-vapt --service qreport-app --desired-count 1 --region ap-southeast-2
aws ecs update-service --cluster qreport-vapt --service qreport-app --force-new-deployment --region ap-southeast-2   # re-read secrets
```
Set to 0 while fixing problems to stop a crash loop.

### 19.2 Status & stopped tasks

```bash
aws ecs describe-services --cluster qreport-vapt --services qreport-app qreport-dj --region ap-southeast-2 \
  --query 'services[].{Name:serviceName,Desired:desiredCount,Running:runningCount,Pending:pendingCount,LastEvent:events[0].message}' --output table

T=$(aws ecs list-tasks --cluster qreport-vapt --desired-status STOPPED --region ap-southeast-2 --query 'taskArns[:4]' --output text)
aws ecs describe-tasks --cluster qreport-vapt --tasks ${=T} --region ap-southeast-2 \
  --query 'tasks[].{Def:taskDefinitionArn,StoppedReason:stoppedReason,Exit:containers[0].exitCode}' --output table

aws logs tail qreport-vapt --since 30m --region ap-southeast-2 | tail -60
```

| Exit code | Usually means |
|---|---|
| 1 | App error at startup → read the app log |
| 137 | Killed — out of memory |
| `CannotPullContainerError` | Image missing / ECR access |

Task IDs must be 32 chars — copied IDs from screenshots get truncated; list them via CLI instead.

### 19.3 App startup failures we fixed

| Log line | Cause | Fix |
|---|---|---|
| `Missing secret_key_base for 'vapt' environment` | No `SECRET_KEY_BASE` in secret | `openssl rand -hex 64` → add to secret |
| `couldn't connect to database` | `DB_PASSWORD` in app secret ≠ real DB password | Copy `MYSQL_ROOT_PASSWORD` into it (section 14.4) |
| `qreport-dj` 0/0 tasks | Its desired count was 0 | Set to 1 |

`qreport-dj` = Delayed Job background worker (emails, async jobs).

---

## 20. DNS & Accessing the App

### 20.1 Who hosts the DNS?

```bash
dig NS qreport.com.au +short        # ns49/ns50.domaincontrol.com → GoDaddy
```

### 20.2 Check existing records (no access needed)

```bash
for h in app-vapt jcalc-vapt portal-vapt; do
  echo "== $h.qreport.com.au"; dig +short $h.qreport.com.au CNAME; dig +short $h.qreport.com.au A
done
```
Finding: `portal-vapt.qreport.com.au` → CNAME to **last year's** ALB `qreport-vapt-alb-572845299…` (no longer exists). New ALB: `qreport-vapt-ALB-652950571…`.

**Request to GoDaddy owner:** update CNAME `portal-vapt` → `qreport-vapt-ALB-652950571.ap-southeast-2.elb.amazonaws.com` (Name field = only `portal-vapt`). Add `jcalc-vapt` if JewelCalc is needed.

### 20.3 Temporary access via hosts file

```bash
dig +short qreport-vapt-ALB-652950571.ap-southeast-2.elb.amazonaws.com | head -1     # e.g. 52.62.135.9
sudo sh -c 'echo "52.62.135.9  portal-vapt.qreport.com.au" >> /etc/hosts'
# open https://portal-vapt.qreport.com.au/login
sudo sed -i '' '/portal-vapt.qreport.com.au/d' /etc/hosts      # remove after testing
```
- `<host>` in examples is a **placeholder** — typing it literally just searches Google.
- ALB IPs change → hosts entries are temporary.
- `app-vapt` reached the app (ALB accepts any `*-vapt` name) but the app's own 404 appeared at `/`; use the real `PORTAL_URL` host (`portal-vapt`) so redirects/links work.

### 20.4 Tester script (Mac/Linux) — resolves current IP each run

```bash
#!/bin/bash
HOST="portal-vapt.qreport.com.au"
ALB="qreport-vapt-ALB-652950571.ap-southeast-2.elb.amazonaws.com"
IP=$(dig +short "$ALB" | grep -E '^[0-9.]+$' | head -1)
[ -z "$IP" ] && { echo "Could not find the VAPT server IP."; exit 1; }
sudo sed -i.bak "/$HOST/d" /etc/hosts
echo "$IP  $HOST" | sudo tee -a /etc/hosts
sudo dscacheutil -flushcache 2>/dev/null; sudo killall -HUP mDNSResponder 2>/dev/null
echo "Done. Open: https://$HOST/login"
```

### 20.5 Tester script (Windows PowerShell as Administrator)

```powershell
$HostName = "portal-vapt.qreport.com.au"
$Alb = "qreport-vapt-ALB-652950571.ap-southeast-2.elb.amazonaws.com"
$ip = (Resolve-DnsName $Alb -Type A | Select-Object -First 1).IPAddress
if (-not $ip) { Write-Host "Could not find the VAPT server IP."; exit 1 }
$hosts = "$env:SystemRoot\System32\drivers\etc\hosts"
(Get-Content $hosts) | Where-Object { $_ -notmatch [regex]::Escape($HostName) } | Set-Content $hosts
Add-Content $hosts "$ip  $HostName"
ipconfig /flushdns | Out-Null
Write-Host "Done. Open: https://$HostName/login"
```

---

## 21. Audit Timeline & Logging

### 21.1 What auditors asked
- How long to mask data? How long to restore masked DB? How long to set up servers? Logs for each.

### 21.2 Evidence sources

| Source | Retention | Gives |
|---|---|---|
| Your `date` before/after each command | Your notes | Start/end per task |
| Terraform output `Creation complete after Xs` | Terminal | Per-resource time |
| Prompt's right side (e.g. `2m 11s`) | Terminal | Last command duration |
| Batch `describe-jobs` + `/aws/batch/job` logs | Never expire | Masking timings |
| CloudTrail Event history | 90 days (no trail in this account) | API calls (who/when) |
| RDS events | 14 days | Restore start/finish |

Save copies under `~/vapt-logs/`. Optional timestamped logging helper:
```bash
tlog() { while IFS= read -r l; do printf '%s %s\n' "$(date '+%F %T')" "$l"; done | tee "$1"; }
{ echo START; terraform apply -no-color x.tfplan; echo "EXIT=$?"; } 2>&1 | tlog ~/vapt-logs/x.log
```

### 21.3 Findings for the report
- No CloudTrail trail exists → last year's activity isn't recoverable from AWS; ask Madheshwaran for his records.
- The old snapshot was from Dec 2022 → replaced by a fresh masked copy.

---

## 22. Troubleshooting Cheat Sheet

| Error / symptom | Cause | Fix |
|---|---|---|
| `No valid credential sources found … IMDS` | Wrong/missing `AWS_PROFILE` (case!) | `export AWS_PROFILE=q-report` |
| `(END)` in terminal | AWS CLI pager | `q`, or `export AWS_PAGER=""` |
| `InvalidSubnet: Subnet IDs are required` | Data source read subnets before they existed | Create subnets first (staged apply) |
| `Permission denied (publickey)` | `.pem` ≠ AWS key pair | Compare fingerprints, re-import key |
| `UNPROTECTED PRIVATE KEY FILE` | `.pem` permissions too open | `chmod 400 key.pem` |
| `SessionManagerPlugin is not found` | Plugin missing | `brew install --cask session-manager-plugin` |
| `TargetNotConnected` | SSM agent not online yet | Wait; check role & NAT |
| ClamAV `%i.conf` not found | Template unit copied as normal unit | Replace `%i` with `scan` |
| EICAR returns OK | Corrupted test string | Use base64 + md5 check |
| ECS instance never joins | User-data deadlock / blank lines before `#!` | Prod-style user data, shebang on line 1 |
| `Container instance cannot be empty` | 0 instances registered | Check ASG + user data |
| curl 403 from ALB | WAF Bot Control blocks curl UA | Use browser UA |
| `ERROR 2002 Can't connect` to restored DB | DB upgrading/backing-up | Wait for `available` |
| `ERROR 1227 … SUPER` | DEFINER in dump | `sed -E 's/DEFINER=…//g'` |
| `ERROR 1064 near '… WARN - triki'` | Log text inside dump | `grep -v 'triki: Deprecated'` |
| `taskId length should be one of [32,36]` | Truncated ID | List tasks via CLI |
| `Unexpected number of separators` | zsh didn't split variable | `${=T}` |
| CodeBuild `AWSCURRENT` secret error | Empty secret | Fill secret |
| `uninitialized constant ENV_PATH::PORTAL_URL` | Missing env config in app code | Dev team |
| `Missing secret_key_base` | Secret lacks key | Add `SECRET_KEY_BASE` |
| `couldn't connect to database` | Wrong DB password in app secret | Copy from `vapt-DB-Credentials` |
| Browser shows Google search for `https://<host>` | Placeholder typed literally | Use real hostname |

---

## 23. Pending Items & Teardown Checklist

### 23.1 Still open
- [ ] **Redis:** keep or skip (`enable_redis` toggle). Don't apply `10-final.tfplan` until decided.
- [ ] **GoDaddy DNS:** update `portal-vapt` CNAME to the new ALB (+ `jcalc-vapt` if needed).
- [ ] **CyberCX test users:** dev team creates them.
- [ ] **WAF policy for testers:** WAF on vs. allow-list tester IPs.
- [ ] **Masking gaps:** decision was "no extra masking" — record it; ask dev team to update `obfuscator.cr` (new tables, kept passwords/tokens) and fix warnings being written into the dump.
- [ ] **Passwords:** consider resetting user passwords to a test password (hashes were kept).
- [ ] **Commit code:** `userdata.sh` fix, `vapt.tfvars` changes (both stacks, CICD `CLAMAV_HOST`).
- [ ] **Delete `*.tfplan` files** in all folders.
- [ ] **Prod drift:** put prod's manual launch-template changes into `production.tfvars`/code.
- [ ] **Certificate:** `*.qreport.com.au` expires ~28 Nov 2026 — renew.
- [ ] **Prod ClamAV:** plan AL2 → AL2023 migration (test on VAPT first).
- [ ] Remove hosts-file entries on your Mac after testing.

### 23.2 Teardown after VAPT
1. Manual snapshot of `qreport-vapt` (the code has `skip_final_snapshot = true`).
2. Disable **deletion protection** on RDS and ALB (both `true` in code) — otherwise destroy fails.
3. Set ECS desired counts to 0.
4. `terraform destroy -var-file=config/vapt.tfvars` in **env-resources** first.
5. Then **common-resources** (same workspace `vapt`).
6. CICD env-specific `vapt` pipeline — destroy or leave (it was left from last year).
7. Key pairs and snapshots remain (by design).
8. Verify nothing VAPT-tagged remains; costs (RDS, NAT, ALB) stop.

---

## 24. Glossary

| Term | Meaning |
|---|---|
| **VAPT** | Vulnerability Assessment & Penetration Testing — security testers attack a copy of prod |
| **Workspace** | Terraform's way to keep separate state per environment in the same code |
| **State** | Terraform's record of what it created (S3 here) |
| **`-target`** | Plan/apply only selected resources (+ their dependencies) |
| **AMI** | Machine image a server boots from |
| **AL2 / AL2023** | Amazon Linux 2 (EOL 2026) / Amazon Linux 2023 |
| **ASG** | Auto Scaling Group — keeps N servers running |
| **Launch template** | Server recipe (AMI, type, key, user data) |
| **User data** | Script that runs once on a server's first boot (cloud-init) |
| **ECS** | Elastic Container Service — runs Docker containers on the EC2 instance |
| **Task / Service** | One running container group / keeps N tasks running |
| **Target group** | List of backends the ALB forwards to, with health checks |
| **ALB** | Application Load Balancer |
| **WAF / Bot Control** | Web firewall / managed rule set that blocks bots & tools like curl |
| **NAT Gateway** | Lets private servers reach the internet outbound only |
| **SSM Session Manager** | Remote shell into servers without SSH ports or public IPs |
| **Secrets Manager** | Stores secrets; apps fetch them at runtime |
| **AWSCURRENT** | Label of a secret's current version (missing = secret empty) |
| **RDS snapshot** | Point-in-time copy of a database (manual ones never expire) |
| **mysqldump** | Exports a MySQL DB as SQL text |
| **DEFINER** | Owner of a view/routine in MySQL; restricted on RDS |
| **AWS Batch** | Runs one-off container jobs on auto-launched EC2 |
| **Triki** | Crystal library that masks MySQL dumps by table/column rules |
| **EICAR** | Harmless standard test string every antivirus flags |
| **INSTREAM** | ClamAV TCP command to scan data sent over the socket |
| **CNAME** | DNS alias pointing one name to another |
| **`/etc/hosts`** | Local file that overrides DNS on one computer |
| **SECRET_KEY_BASE** | Rails' secret for signing cookies/sessions |
| **tmux** | Keeps terminal sessions running after you disconnect |
| **pv** | Shows progress of data flowing through a pipe |

---

*Built 13–14 Oct 2026 — QReport VAPT, AWS account 463949419568, region ap-southeast-2.*
