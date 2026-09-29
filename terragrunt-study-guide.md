# Terragrunt End-to-End Complete Study Guide
> **Target Audience:** Junior DevOps Engineers transitioning from raw Terraform to scalable, enterprise-grade infrastructure management with Terragrunt.

---

## Table of Contents
1. [The "Why": The Problem Terragrunt Solves](#1-the-why-the-problem-terragrunt-solves)
2. [What is Terragrunt? (Mental Model)](#2-what-is-terragrunt-mental-model)
3. [Core Building Blocks & Syntax](#3-core-building-blocks--syntax)
4. [Standard Production Architecture (Live vs Modules)](#4-standard-production-architecture-live-vs-modules)
5. [Deep Dive into Essential Terragrunt Features](#5-deep-dive-into-essential-terragrunt-features)
   - [5.1 The `terraform` Source Block](#51-the-terraform-source-block)
   - [5.2 The `include` Block (Inheritance & DRY)](#52-the-include-block-inheritance--dry)
   - [5.3 Remote State Orchestration (`remote_state`)](#53-remote-state-orchestration-remote_state)
   - [5.4 Generating Code on the Fly (`generate`)](#54-generating-code-on-the-fly-generate)
   - [5.5 Inter-Module Dependencies (`dependency` & `mock_outputs`)](#55-inter-module-dependencies-dependency--mock_outputs)
   - [5.6 Lifecycle Hooks (`before_hook`, `after_hook`)](#56-lifecycle-hooks-before_hook-after_hook)
6. [CLI Mastery & Workflow](#6-cli-mastery--workflow)
7. [Hands-On End-to-End Project (Step-by-Step)](#7-hands-on-end-to-end-project-step-by-step)
8. [Common Junior Pitfalls & How to Avoid Them](#8-common-junior-pitfalls--how-to-avoid-them)
9. [DevOps Interview Questions & Answers](#9-devops-interview-questions--answers)
10. [Quick Syntax Reference & Cheat Sheet](#10-quick-syntax-reference--cheat-sheet)

---

## 1. The "Why": The Problem Terragrunt Solves

As a DevOps engineer using plain Terraform, you quickly run into real-world pain points when managing multiple environments (`dev`, `staging`, `prod`) and multiple regions (`us-east-1`, `eu-west-1`):

### Pain Point 1: Code Duplication (WET - "Write Everything Twice")
In vanilla Terraform, to manage `dev` and `prod`, you often duplicate backend definitions and provider blocks:
```hcl
# In dev/backend.tf
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-states"
    key            = "dev/vpc/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
  }
}

# In prod/backend.tf -- identical except the key!
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-states"
    key            = "prod/vpc/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
  }
}
```
If you have 20 modules across 3 environments, that's **60 duplicate backend files**. If your bucket name or region changes, you have to update all 60 files.

### Pain Point 2: Monolithic State vs. Micro-States (Blast Radius)
- **Monolith:** If you put VPC, EKS, RDS, and Redis in a single state file, running `terraform plan` takes 10 minutes. A mistake in Redis can accidentally destroy your production database or VPC.
- **Micro-states:** Breaking infrastructure into small folders (`vpc/`, `rds/`, `eks/`) isolates blast radius and speeds up runs. However, managing dependencies between these separate folders in vanilla Terraform requires cumbersome `terraform_remote_state` data sources or manual copy-pasting of IDs.

### Pain Point 3: Environment Drift & Variable Copy-Pasting
Managing different values for `dev` vs `prod` using `terraform.tfvars` often results in hundreds of lines of duplicated configurations where only 2 variables actually differ.

---

## 2. What is Terragrunt? (Mental Model)

**Terragrunt is a thin, open-source wrapper for Terraform / OpenTofu** created by Gruntwork.

> **Analogy:**
> - **Terraform Modules** = Blueprint / Recipe (defines *what* AWS resources can be built).
> - **Terragrunt** = Construction Manager / Configuration Orchestrator (decides *where*, *with what parameters*, and *in what order* to build).

```
                     ┌──────────────────────────┐
                     │     User executes:       │
                     │  terragrunt run-all apply│
                     └─────────────┬────────────┘
                                   │
                     ┌─────────────▼────────────┐
                     │        Terragrunt        │
                     │  - Computes DAG (order)  │
                     │  - Generates backend.tf  │
                     │  - Injects variables     │
                     │  - Resolves dependencies │
                     └─────────────┬────────────┘
                                   │
                 ┌─────────────────┴─────────────────┐
                 ▼                                   ▼
        ┌─────────────────┐                 ┌─────────────────┐
        │ terraform apply │                 │ terraform apply │
        │   (vpc state)   │                 │   (rds state)   │
        └─────────────────┘                 └─────────────────┘
```

### What Terragrunt does behind the scenes:
1. Reads your `terragrunt.hcl` files.
2. Creates a temporary caching directory (`.terragrunt-cache/`).
3. Downloads the Terraform code/module defined in `terraform { source = "..." }`.
4. Injects backend configs and provider configs via generation blocks.
5. Injects inputs as Terraform variables (`TF_VAR_*` or generated `.tfvars.json`).
6. Executes standard `terraform` binary commands on your behalf.

---

## 3. Core Building Blocks & Syntax

Terragrunt uses HCL (HashiCorp Configuration Language), just like Terraform, but files are named **`terragrunt.hcl`**.

### The 6 Core Blocks You Must Know:

| Block | What it does | Real-World Use Case |
|---|---|---|
| `terraform` | Specifies where the real `.tf` code lives. | Points to a local folder or Git repo (with tag/branch). |
| `include` | Inherits configuration from a parent `terragrunt.hcl`. | Keeps backend and provider DRY. |
| `remote_state` | Configures remote backend (S3 + DynamoDB, GCS, Azure Blob). | Automatically creates S3 bucket and lock table if missing! |
| `generate` | Writes standard `.tf` files dynamically before running Terraform. | Writes `provider.tf` with dynamic region or account ID. |
| `dependency` | Reads outputs from another isolated Terragrunt unit. | Passes VPC Subnet IDs directly to an RDS or EKS unit. |
| `inputs` | Key-value pairs passed as variables to the underlying module. | `instance_type = "t3.micro"`, `environment = "dev"`. |

---

## 4. Standard Production Architecture (Live vs Modules)

In enterprise DevOps, we strictly separate **Infrastructure Modules** from **Infrastructure Live**.

### Two-Repository Pattern (Best Practice):

```
1. Repositories:
   ├── infra-modules/        (Pure Terraform: reusable modules)
   │   ├── vpc/
   │   │   ├── main.tf
   │   │   ├── variables.tf
   │   │   └── outputs.tf
   │   └── rds/
   │       ├── main.tf
   │       ├── variables.tf
   │       └── outputs.tf
   │
   └── infra-live/           (Pure Terragrunt: environment values & state layout)
       ├── terragrunt.hcl    (Root configuration: backend, providers)
       ├── dev/
       │   ├── env.hcl       (Environment-specific variables: env = "dev")
       │   ├── vpc/
       │   │   └── terragrunt.hcl
       │   └── rds/
       │       └── terragrunt.hcl
       └── prod/
           ├── env.hcl       (Environment-specific variables: env = "prod")
           ├── vpc/
           │   └── terragrunt.hcl
           └── rds/
               └── terragrunt.hcl
```

Notice: In `infra-live/`, **there are no `.tf` files!** Only `terragrunt.hcl` files.

---

## 5. Deep Dive into Essential Terragrunt Features

### 5.1 The `terraform` Source Block
Tells Terragrunt where the actual Terraform code lives.

```hcl
# In infra-live/dev/vpc/terragrunt.hcl

# Option A: Local path (Great for local testing/development)
terraform {
  source = "../../../infra-modules//vpc"
}

# Option B: Remote Git repository (Production standard)
terraform {
  source = "git::git@github.com:myorg/infra-modules.git//vpc?ref=v1.2.0"
}
```

> **Notice the double slash (`//`):**
> This is Terraform standard syntax to specify a subfolder inside a Git repository. Everything before `//` is cloned, and everything after `//` is the working directory.

---

### 5.2 The `include` Block (Inheritance & DRY)
Allows child `terragrunt.hcl` files to inherit settings from parent files.

```hcl
# In infra-live/dev/vpc/terragrunt.hcl

# Exposes all blocks (remote_state, generate, etc.) from the root terragrunt.hcl
include "root" {
  path = find_in_parent_folders("terragrunt.hcl")
}
```

`find_in_parent_folders()` searches upwards in the directory tree until it finds the root `terragrunt.hcl`.

---

### 5.3 Remote State Orchestration (`remote_state`)

Defined once in the **root** `terragrunt.hcl`:

```hcl
# In infra-live/terragrunt.hcl (ROOT)
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "mycompany-terraform-states"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "mycompany-terraform-locks"
  }
}
```

#### Magic of `path_relative_to_include()`:
If the child file is in `dev/vpc/terragrunt.hcl`, the key in S3 automatically becomes:
`dev/vpc/terraform.tfstate`!
If the child is in `prod/rds/terragrunt.hcl`, the key automatically becomes:
`prod/rds/terraform.tfstate`!

> **Terragrunt Superpower:** If the S3 bucket or DynamoDB table does not exist, Terragrunt will **automatically create them** with encryption, versioning, and access logging enabled!

---

### 5.4 Generating Code on the Fly (`generate`)

Use `generate` blocks to produce `.tf` files dynamically into the working directory.

```hcl
# In infra-live/terragrunt.hcl (ROOT)
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "us-east-1"
  default_tags {
    tags = {
      ManagedBy = "Terragrunt"
    }
  }
}
EOF
}
```

When you run `terragrunt plan`, Terragrunt writes a `provider.tf` file into the temporary execution cache folder.

---

### 5.5 Inter-Module Dependencies (`dependency` & `mock_outputs`)

This is arguably **Terragrunt's single greatest feature**.

Imagine `rds` needs `vpc_id` and `private_subnet_ids` from `vpc`.

#### Traditional Terraform:
You have to use `data "terraform_remote_state"` inside your `.tf` module, coupling your reusable module to a specific backend structure.

#### Terragrunt Way:
The module stays completely agnostic, and Terragrunt wires the outputs together in `infra-live`:

```hcl
# In infra-live/dev/rds/terragrunt.hcl

dependency "vpc" {
  config_path = "../vpc"

  # What if VPC has NOT been applied yet (e.g. running 'terragrunt run-all plan' on Day 1)?
  mock_outputs = {
    vpc_id             = "vpc-fake123456"
    private_subnet_ids = ["subnet-fake1", "subnet-fake2"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  vpc_id     = dependency.vpc.outputs.vpc_id
  subnet_ids = dependency.vpc.outputs.private_subnet_ids
  db_name    = "devdb"
}
```

> **Why `mock_outputs` is critical:**
> On a brand-new deployment, `vpc` hasn't run yet, so it has no state or outputs. Without `mock_outputs`, running `terragrunt run-all plan` would crash with `output "vpc_id" not found`. Terragrunt temporarily uses fake values during `plan` and `validate`, and swaps in real values during `apply`!

---

### 5.6 Lifecycle Hooks (`before_hook`, `after_hook`)

Execute custom scripts or binaries before or after Terraform execution:

```hcl
terraform {
  before_hook "tflint" {
    commands     = ["apply", "plan"]
    execute      = ["tflint"]
  }

  after_hook "notify_slack" {
    commands     = ["apply"]
    execute      = ["bash", "-c", "echo 'Deployment succeeded!'"]
    run_on_error = false
  }
}
```

---

## 6. CLI Mastery & Workflow

Every standard `terraform` command works with `terragrunt` by simply replacing `terraform` with `terragrunt`.

### Basic Single-Module Commands:
```bash
# Inside dev/vpc/
terragrunt init       # Initializes backend and downloads providers
terragrunt plan       # Previews changes
terragrunt apply      # Applies changes
terragrunt destroy   # Destroys resources for this module only
terragrunt output     # Displays outputs
```

### Multi-Module Orchestration (`run-all`):
When you want to operate on an entire environment with multiple dependencies:

```bash
# Inside dev/
terragrunt run-all plan
terragrunt run-all apply
terragrunt run-all destroy
```

#### How `run-all` works:
1. Terragrunt scans every child directory containing a `terragrunt.hcl`.
2. It builds a **Directed Acyclic Graph (DAG)** of all dependencies.
3. It executes them concurrently in topological order (e.g., `vpc` first, then `rds` and `eks` in parallel).

```bash
# Visualize the dependency tree!
terragrunt graph-dependencies | dot -Tpng > graph.png
```

#### Handy Flags:
- `--terragrunt-parallelism 4`: Run up to 4 modules concurrently.
- `--terragrunt-non-interactive`: Auto-approve prompts (ideal for CI/CD pipelines).
- `--terragrunt-working-dir <path>`: Run Terragrunt in a target directory without `cd`.

---

## 7. Hands-On End-to-End Project (Step-by-Step)

Let's build a clean, production-ready structure: a **VPC** and an **EC2 Instance** that depends on that VPC.

### Project Layout:
```
my-infra/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── app/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── live/
    ├── terragrunt.hcl              (Root)
    └── dev/
        ├── env.hcl                 (Dev settings)
        ├── vpc/
        │   └── terragrunt.hcl
        └── app/
            └── terragrunt.hcl
```

---

### Step 1: Write Reusable Modules (`modules/`)

#### `modules/vpc/main.tf`
```hcl
variable "cidr_block" { type = string }
variable "env" { type = string }

resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true

  tags = {
    Name = "${var.env}-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.cidr_block, 4, 1)
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.env}-public-subnet"
  }
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_id" {
  value = aws_subnet.public.id
}
```

#### `modules/app/main.tf`
```hcl
variable "subnet_id" { type = string }
variable "instance_type" { type = string }
variable "env" { type = string }

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  tags = {
    Name = "${var.env}-web-server"
  }
}

output "instance_id" {
  value = aws_instance.web.id
}
```

---

### Step 2: Configure the Root `live/terragrunt.hcl`

```hcl
# live/terragrunt.hcl

# Automatically generate AWS provider
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
EOF
}

# Automatically configure remote S3 state
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "mycompany-tf-state-storage-unique123"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "mycompany-tf-locks"
  }
}
```

---

### Step 3: Define Environment Config `live/dev/env.hcl`

```hcl
# live/dev/env.hcl
locals {
  env = "dev"
}
```

---

### Step 4: Configure VPC `live/dev/vpc/terragrunt.hcl`

```hcl
# live/dev/vpc/terragrunt.hcl

include "root" {
  path = find_in_parent_folders("terragrunt.hcl")
}

# Read env variables from env.hcl
locals {
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  env      = local.env_vars.locals.env
}

terraform {
  source = "../../../modules//vpc"
}

inputs = {
  env        = local.env
  cidr_block = "10.0.0.0/16"
}
```

---

### Step 5: Configure App `live/dev/app/terragrunt.hcl`

```hcl
# live/dev/app/terragrunt.hcl

include "root" {
  path = find_in_parent_folders("terragrunt.hcl")
}

locals {
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  env      = local.env_vars.locals.env
}

terraform {
  source = "../../../modules//app"
}

dependency "vpc" {
  config_path = "../vpc"

  mock_outputs = {
    subnet_id = "subnet-00000000000000000"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  env           = local.env
  subnet_id     = dependency.vpc.outputs.subnet_id
  instance_type = "t3.micro"
}
```

---

### Step 6: Deploying the Entire Stack

From the `live/dev/` folder:

```bash
# 1. Preview what will happen across ALL modules
terragrunt run-all plan

# 2. Deploy everything in correct dependency order (VPC first, then App)
terragrunt run-all apply

# 3. Clean up / Tear down when finished
terragrunt run-all destroy
```

---

## 8. Common Junior Pitfalls & How to Avoid Them

### ❌ Pitfall 1: Forgetting `mock_outputs` on Greenfield Deploys
- **Symptom:** You run `terragrunt run-all plan` on a fresh account and get:
  `Module ../vpc has not been applied yet and has no outputs`.
- **Fix:** Always supply `mock_outputs` and specify `mock_outputs_allowed_terraform_commands = ["validate", "plan"]`.

### ❌ Pitfall 2: Confusing Double Slash (`//`) in Git Sources
- **Symptom:** Terragrunt clones the entire repo and tries to run `terraform` at the repo root instead of inside your subfolder.
- **Fix:** Ensure there is a double slash before the folder path:
  `git::https://github.com/org/repo.git//modules/vpc?ref=v1.0.0`

### ❌ Pitfall 3: Stale `.terragrunt-cache`
- **Symptom:** You modified your local module, but Terragrunt seems to run the old code or throws bizarre lockfile errors.
- **Fix:** Terragrunt caches downloads in hidden `.terragrunt-cache` directories. Clear them:
  ```bash
  find . -type d -name ".terragrunt-cache" -prune -exec rm -rf {} +
  ```

### ❌ Pitfall 4: Putting Business Logic in `terragrunt.hcl`
- **Mistake:** Writing resource blocks or complex conditional resources inside `terragrunt.hcl`.
- **Rule:** `terragrunt.hcl` only provides **configuration, dependencies, and inputs**. Resource creation belongs in `.tf` files inside `modules/`.

---

## 9. DevOps Interview Questions & Answers

### Q1: What is the primary difference between Terraform and Terragrunt?
> **Answer:** Terraform is an Infrastructure as Code (IaC) tool that executes definitions of infrastructure resources. Terragrunt is an orchestration tool and wrapper on top of Terraform. Terragrunt solves code duplication (keeping backend and providers DRY), manages isolated remote states per module, and manages dependencies between isolated state files without hardcoding.

### Q2: Why is separating state files per component better than having one large state file?
> **Answer:**
> 1. **Blast Radius Isolation:** An error or lock contention in one service (e.g., Redis) won't prevent deploying or breaking critical components (e.g., VPC or DB).
> 2. **Performance:** `terraform plan` on 10 resources takes 5 seconds; on 500 resources, it can take 15+ minutes.
> 3. **Role-Based Access Control:** Security teams can grant access to the `app` state without granting access to the `networking/vpc` state.

### Q3: How does Terragrunt resolve dependencies when running `run-all apply`?
> **Answer:** Terragrunt parses the `dependency` blocks in each `terragrunt.hcl` file, constructs a Directed Acyclic Graph (DAG) in memory, and applies modules in topological order. Modules without mutual dependencies run concurrently in parallel threads.

### Q4: What are `mock_outputs` used for?
> **Answer:** When running `run-all plan` against an unapplied environment, dependent modules cannot read real outputs from modules that have not been deployed yet. `mock_outputs` inject placeholder values during read-only commands (`plan`, `validate`) so the plan succeeds without errors.

---

## 10. Quick Syntax Reference & Cheat Sheet

```hcl
# Read another HCL file's locals
locals {
  env_data = read_terragrunt_config(find_in_parent_folders("env.hcl"))
}

# Include parent configuration
include "root" {
  path = find_in_parent_folders("terragrunt.hcl")
}

# Target Terraform module
terraform {
  source = "git::git@github.com:foo/bar.git//module?ref=v1.0"
}

# Read output from another module
dependency "vpc" {
  config_path = "../vpc"
  mock_outputs = { id = "vpc-000" }
  mock_outputs_allowed_terraform_commands = ["plan"]
}

# Pass parameters to Terraform module
inputs = {
  subnet_id = dependency.vpc.outputs.id
  env       = local.env_data.locals.env
}
```

---
*Created for Junior DevOps Engineers | Keep your infrastructure modular, DRY, and safe!*
