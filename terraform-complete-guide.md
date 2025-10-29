# Complete Terraform Tutorial: Comprehensive Practical Guide

## Table of Contents

1. [Introduction to Terraform](#introduction-to-terraform)
2. [What is Terraform?](#what-is-terraform)
3. [Why Use Terraform?](#why-use-terraform)
4. [Infrastructure as Code (IaC)](#infrastructure-as-code-iac)
5. [Terraform Architecture](#terraform-architecture)
6. [Core Components](#core-components)
7. [Installing Terraform](#installing-terraform)
8. [Terraform Basics](#terraform-basics)
9. [HashiCorp Configuration Language (HCL)](#hashicorp-configuration-language-hcl)
10. [Terraform Workflow](#terraform-workflow)
11. [Providers](#providers)
12. [Resources](#resources)
13. [Variables](#variables)
14. [Outputs](#outputs)
15. [Data Sources](#data-sources)
16. [State Management](#state-management)
17. [Modules](#modules)
18. [Workspaces](#workspaces)
19. [Remote State](#remote-state)
20. [Provisioners](#provisioners)
21. [Terraform Functions](#terraform-functions)
22. [Best Practices](#best-practices)
23. [CI/CD Integration](#cicd-integration)
24. [Security Best Practices](#security-best-practices)
25. [Troubleshooting](#troubleshooting)
26. [Advantages and Disadvantages](#advantages-and-disadvantages)
27. [Top 100 Terraform Interview Questions](#top-100-terraform-interview-questions)

---

## Introduction to Terraform

Terraform is the industry-leading Infrastructure as Code (IaC) tool developed by HashiCorp. It enables DevOps teams to define, provision, and manage infrastructure across multiple cloud providers using a simple, declarative configuration language.

**Created:** 2014 by HashiCorp  
**Written in:** Go (Golang)  
**Type:** Infrastructure as Code (IaC) Tool  
**License:** Mozilla Public License 2.0 (previously BSL)

Terraform has revolutionized how organizations manage infrastructure, making it easier to version control, collaborate on, and automate infrastructure deployments.

---

## What is Terraform?

Terraform is an open-source infrastructure as code software tool that provides a consistent CLI workflow to manage hundreds of cloud services. It codifies APIs into declarative configuration files that can be shared amongst team members, treated as code, edited, reviewed, and versioned.

### Key Characteristics

- **Declarative Approach**: Define "what" you want, not "how" to create it
- **Cloud Agnostic**: Works with AWS, Azure, GCP, and 100+ providers
- **State Management**: Tracks infrastructure state intelligently
- **Execution Plans**: Preview changes before applying
- **Resource Graph**: Understands resource dependencies
- **Change Automation**: Automates infrastructure changes safely
- **Open Source**: Community-driven with enterprise options

### How Terraform Works

```
┌──────────────┐
│ Config Files │  (.tf files)
│   (HCL)      │
└──────┬───────┘
       │
       ↓
┌──────────────┐
│ Terraform    │
│   Core       │ ──→ Reads config
└──────┬───────┘     Creates execution plan
       │
       ↓
┌──────────────┐
│  Providers   │ ──→ AWS, Azure, GCP, etc.
└──────┬───────┘
       │
       ↓
┌──────────────┐
│ Cloud/Infra  │  Infrastructure is created
└──────────────┘
```

---

## Why Use Terraform?

### Use Cases

Terraform excels in:

1. **Multi-Cloud Deployments**: Manage infrastructure across AWS, Azure, GCP simultaneously
2. **Infrastructure Automation**: Eliminate manual configuration
3. **Disaster Recovery**: Quickly recreate infrastructure from code
4. **Environment Provisioning**: Create dev, staging, prod environments consistently
5. **Compliance & Governance**: Enforce infrastructure policies
6. **Cost Management**: Track and optimize infrastructure costs
7. **Microservices Infrastructure**: Manage complex service architectures
8. **Hybrid Cloud**: Manage both cloud and on-premises infrastructure

### When to Use Terraform

✅ **Use Terraform when you need:**
- Multi-cloud infrastructure management
- Version-controlled infrastructure
- Reproducible environments
- Infrastructure automation
- Team collaboration on infrastructure
- Immutable infrastructure patterns
- Complex dependency management

❌ **Consider alternatives when:**
- You need runtime configuration management (use Ansible, Chef)
- You're locked into a single cloud (CloudFormation might be simpler for AWS-only)
- You need application deployment (use Kubernetes, Docker Compose)
- You want actual programming languages (use Pulumi)

---

## Infrastructure as Code (IaC)

Infrastructure as Code (IaC) is the practice of managing and provisioning infrastructure through machine-readable definition files, rather than physical hardware configuration or interactive configuration tools.

### Benefits of IaC

**1. Version Control**
- Track every infrastructure change
- Rollback to previous versions
- Collaborate using Git workflows

**2. Consistency**
- Same configuration every time
- Eliminate configuration drift
- Reproducible infrastructure

**3. Speed**
- Automated provisioning
- Rapid environment creation
- Faster disaster recovery

**4. Documentation**
- Code is self-documenting
- Easy to understand infrastructure
- Clear audit trails

**5. Cost Optimization**
- Identify unused resources
- Standardize infrastructure sizes
- Automate resource cleanup

### IaC vs Traditional Infrastructure

| Traditional | Infrastructure as Code |
|-------------|------------------------|
| Manual configuration | Automated provisioning |
| Prone to errors | Consistent and repeatable |
| Difficult to replicate | Easy to duplicate |
| No version control | Full version history |
| Slow provisioning | Rapid deployment |
| Undocumented | Self-documenting |

---

## Terraform Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────┐
│          User Configuration Files               │
│              (*.tf files - HCL)                 │
└────────────────┬────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────┐
│            Terraform Core                       │
│  ┌──────────────────────────────────────────┐  │
│  │  - Configuration Parser                   │  │
│  │  - Dependency Graph Builder              │  │
│  │  - State Manager                         │  │
│  │  - Execution Planner                     │  │
│  └──────────────────────────────────────────┘  │
└────────────────┬────────────────────────────────┘
                 │
    ┌────────────┴────────────┐
    ↓                         ↓
┌──────────┐           ┌──────────┐
│ Providers│           │  State   │
│ (Plugins)│           │  Backend │
└────┬─────┘           └──────────┘
     │
     ↓
┌──────────────────────────────────┐
│  Cloud Providers / Services      │
│  (AWS, Azure, GCP, Kubernetes)   │
└──────────────────────────────────┘
```

### Core Components

**1. Terraform Core**
- Written in Go
- Reads configuration files
- Builds dependency graph
- Creates execution plans
- Manages state
- Orchestrates provider operations

**2. Providers (Plugins)**
- Interface with APIs
- Define resources
- Implement CRUD operations
- Communicate via RPC

**3. State**
- Tracks resource metadata
- Maps config to real infrastructure
- Stores resource dependencies
- Enables change detection

**4. Configuration Files**
- Written in HCL
- Define desired infrastructure
- Declare resources and dependencies
- Configure provider settings

---

## Core Components

### 1. Configuration Files (.tf)

Terraform uses configuration files with `.tf` extension written in HCL (HashiCorp Configuration Language).

**Standard file structure:**
```
project/
├── main.tf           # Primary resources
├── variables.tf      # Input variables
├── outputs.tf        # Output values
├── providers.tf      # Provider configuration
├── terraform.tfvars  # Variable values
└── versions.tf       # Version constraints
```

### 2. Terraform Block

Defines Terraform settings.

```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

### 3. Provider Block

Configures the provider (cloud platform).

```hcl
provider "aws" {
  region = "us-east-1"
  
  default_tags {
    tags = {
      Environment = "Production"
      ManagedBy   = "Terraform"
    }
  }
}
```

### 4. Resource Block

Defines infrastructure components.

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "WebServer"
  }
}
```

### 5. Data Block

Fetches information from existing resources.

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}
```

---

## Installing Terraform

### Installation on Windows

#### Method 1: Manual Installation

**Step 1: Download Terraform**
1. Visit: https://developer.hashicorp.com/terraform/install
2. Select Windows tab
3. Download AMD64 or 386 (32-bit) ZIP file

**Step 2: Extract the ZIP**
```powershell
# Create directory
mkdir C:\terraform
# Extract terraform.exe to C:\terraform
```

**Step 3: Add to PATH**
1. Open System Properties → Environment Variables
2. Edit `Path` variable
3. Add `C:\terraform`
4. Click OK

**Step 4: Verify Installation**
```powershell
terraform version
```

#### Method 2: Using Chocolatey

```powershell
# Install Chocolatey (if not installed)
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install Terraform
choco install terraform

# Verify
terraform --version
```

### Installation on Ubuntu/Linux

#### Method 1: Using APT Repository

```bash
# Update system
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

# Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | \
    gpg --dearmor | \
    sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Verify key fingerprint
gpg --no-default-keyring \
    --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
    --fingerprint

# Add HashiCorp repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
    https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
    sudo tee /etc/apt/sources.list.d/hashicorp.list

# Update and install
sudo apt update
sudo apt-get install terraform

# Verify installation
terraform -version
```

#### Method 2: Manual Installation (Binary)

```bash
# Download Terraform (replace with latest version)
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip

# Install unzip if needed
sudo apt-get install unzip

# Unzip to /usr/local/bin
sudo unzip terraform_1.6.0_linux_amd64.zip -d /usr/local/bin/

# Verify installation
terraform --version

# Clean up
rm terraform_1.6.0_linux_amd64.zip
```

### Installation on macOS

#### Method 1: Using Homebrew

```bash
# Install Homebrew (if not installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Tap HashiCorp repository
brew tap hashicorp/tap

# Install Terraform
brew install hashicorp/tap/terraform

# Verify installation
terraform --version

# Update Terraform
brew upgrade hashicorp/tap/terraform
```

#### Method 2: Manual Installation

```bash
# Create directory
sudo mkdir -p /opt/terraform

# Navigate to directory
cd /opt/terraform

# Download Terraform (macOS ARM64 or AMD64)
sudo curl -O https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_darwin_arm64.zip

# Unzip
sudo unzip terraform_1.6.0_darwin_arm64.zip

# Add to PATH (add to ~/.bash_profile or ~/.zshrc)
echo 'export PATH="/opt/terraform:$PATH"' >> ~/.zshrc

# Reload shell configuration
source ~/.zshrc

# Verify
terraform --version
```

### Docker Installation

```bash
# Pull Terraform Docker image
docker pull hashicorp/terraform:latest

# Run Terraform commands
docker run --rm -v $(pwd):/workspace -w /workspace hashicorp/terraform init
docker run --rm -v $(pwd):/workspace -w /workspace hashicorp/terraform plan

# Create alias for convenience
alias terraform='docker run --rm -v $(pwd):/workspace -w /workspace hashicorp/terraform'
```

### Enable Tab Completion

```bash
# For Bash
terraform -install-autocomplete

# For Zsh
terraform -install-autocomplete

# Reload shell
source ~/.bashrc  # or ~/.zshrc
```

---

## Terraform Basics

### Creating Your First Terraform Project

#### Step 1: Create Project Directory

```bash
mkdir terraform-demo
cd terraform-demo
```

#### Step 2: Create Configuration File (main.tf)

```hcl
# main.tf

terraform {
  required_version = ">= 1.0"
  
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

resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "TerraformExample"
  }
}
```

#### Step 3: Initialize Terraform

```bash
terraform init
```

**What happens:**
- Downloads provider plugins
- Initializes backend
- Creates `.terraform` directory
- Creates `.terraform.lock.hcl` (dependency lock file)

#### Step 4: Format Code

```bash
terraform fmt
```

Automatically formats Terraform configuration files to a canonical format.

#### Step 5: Validate Configuration

```bash
terraform validate
```

Checks configuration for syntax errors.

#### Step 6: Plan Changes

```bash
terraform plan
```

**Output shows:**
- Resources to be created (+)
- Resources to be modified (~)
- Resources to be destroyed (-)

#### Step 7: Apply Changes

```bash
terraform apply
```

Type `yes` to confirm and create infrastructure.

**Auto-approve (use with caution):**
```bash
terraform apply -auto-approve
```

#### Step 8: View State

```bash
terraform show
```

Displays current state of infrastructure.

#### Step 9: Destroy Infrastructure

```bash
terraform destroy
```

Type `yes` to confirm and destroy all resources.

### Basic Terraform Commands

```bash
# Initialize working directory
terraform init

# Format configuration files
terraform fmt

# Validate configuration
terraform validate

# Create execution plan
terraform plan

# Apply changes
terraform apply

# Destroy infrastructure
terraform destroy

# Show current state
terraform show

# List resources in state
terraform state list

# Show resource from state
terraform state show <resource_address>

# Refresh state
terraform refresh

# Output values
terraform output

# Get providers
terraform providers

# View dependency graph
terraform graph | dot -Tpng > graph.png

# Version information
terraform version

# Console for expression evaluation
terraform console
```

---

## HashiCorp Configuration Language (HCL)

HCL (HashiCorp Configuration Language) is a declarative language designed to be both human-readable and machine-friendly.

### Basic Syntax

```hcl
# Comments start with #

// Or double slashes

/* Multi-line
   comments */

# Block syntax
<BLOCK_TYPE> "<BLOCK_LABEL>" "<BLOCK_LABEL>" {
  # Block body
  <IDENTIFIER> = <EXPRESSION>
}
```

### Resource Syntax

```hcl
resource "resource_type" "resource_name" {
  argument1 = value1
  argument2 = value2
  
  nested_block {
    nested_argument = value
  }
}
```

### Data Types

**1. String**
```hcl
name = "example"
multi_line = <<EOF
This is a
multi-line string
EOF
```

**2. Number**
```hcl
port = 8080
count = 3
```

**3. Boolean**
```hcl
enabled = true
public = false
```

**4. List**
```hcl
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
```

**5. Map**
```hcl
tags = {
  Name        = "WebServer"
  Environment = "Production"
}
```

**6. Set**
```hcl
security_groups = toset(["sg-123", "sg-456"])
```

**7. Object**
```hcl
server_config = {
  instance_type = "t2.micro"
  ami           = "ami-123456"
  count         = 2
}
```

**8. Tuple**
```hcl
mixed_list = ["string", 123, true]
```

### Expressions

**1. String Interpolation**
```hcl
name = "server-${var.environment}"
```

**2. Arithmetic**
```hcl
total = var.price * var.quantity
```

**3. Comparison**
```hcl
is_prod = var.environment == "production"
```

**4. Logical**
```hcl
enabled = var.feature_enabled && var.environment == "prod"
```

**5. Conditional**
```hcl
instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"
```

**6. For Expression**
```hcl
upper_names = [for name in var.names : upper(name)]
```

**7. Splat Expression**
```hcl
instance_ids = aws_instance.servers[*].id
```

---

## Terraform Workflow

The standard Terraform workflow consists of the following stages:

### 1. Write Configuration

Create `.tf` files defining your infrastructure.

```hcl
# main.tf
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

### 2. Initialize (terraform init)

```bash
terraform init
```

**Purpose:**
- Download provider plugins
- Initialize backend
- Download modules
- Create dependency lock file

### 3. Validate (terraform validate)

```bash
terraform validate
```

**Purpose:**
- Check syntax
- Validate configuration
- Ensure references are correct

### 4. Plan (terraform plan)

```bash
terraform plan
```

**Purpose:**
- Preview changes
- Show execution plan
- Detect errors before applying
- Review what will be created/modified/destroyed

**Output example:**
```
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami           = "ami-0c55b159cbfafe1f0"
      + instance_type = "t2.micro"
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

### 5. Apply (terraform apply)

```bash
terraform apply
```

**Purpose:**
- Execute the plan
- Create/modify/delete resources
- Update state file

### 6. Verify Infrastructure

Check the created resources in your cloud console or using CLI.

### 7. Update Configuration

Modify `.tf` files as needed.

### 8. Plan and Apply Again

Review and apply changes incrementally.

### 9. Destroy (terraform destroy)

```bash
terraform destroy
```

**Purpose:**
- Delete all managed resources
- Clean up infrastructure

### Complete Workflow Diagram

```
┌──────────────┐
│    Write     │
│    Code      │
└──────┬───────┘
       │
       ↓
┌──────────────┐
│  terraform   │
│    init      │
└──────┬───────┘
       │
       ↓
┌──────────────┐
│  terraform   │
│   validate   │
└──────┬───────┘
       │
       ↓
┌──────────────┐
│  terraform   │
│    plan      │
└──────┬───────┘
       │
       ↓
┌──────────────┐
│  terraform   │
│    apply     │
└──────┬───────┘
       │
       ↓
┌──────────────┐
│ Infrastructure│
│   Running    │
└──────┬───────┘
       │
       ↓
┌──────────────┐
│  terraform   │
│   destroy    │
└──────────────┘
```

---

## Providers

Providers are plugins that allow Terraform to interact with cloud providers, SaaS providers, and other APIs.

### Configuring Providers

```hcl
# Single provider
provider "aws" {
  region = "us-east-1"
}

# Provider with alias (multiple regions)
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# Using aliased provider
resource "aws_instance" "west_server" {
  provider = aws.west
  
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

### Popular Providers

**1. AWS Provider**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region     = "us-east-1"
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
}
```

**2. Azure Provider**
```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}
```

**3. Google Cloud Provider**
```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = "us-central1"
  zone    = "us-central1-a"
}
```

**4. Kubernetes Provider**
```hcl
provider "kubernetes" {
  config_path = "~/.kube/config"
}
```

**5. Docker Provider**
```hcl
provider "docker" {
  host = "unix:///var/run/docker.sock"
}
```

### Provider Authentication

**AWS Authentication Methods:**

1. **Environment Variables**
```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"
```

2. **Shared Credentials File**
```hcl
provider "aws" {
  region                  = "us-east-1"
  shared_credentials_files = ["~/.aws/credentials"]
  profile                 = "default"
}
```

3. **IAM Role (for EC2)**
```hcl
provider "aws" {
  region = "us-east-1"
  # Automatically uses IAM role
}
```

### Version Constraints

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Allow 5.x versions
    }
    
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 3.0, < 4.0"  # Range
    }
    
    google = {
      source  = "hashicorp/google"
      version = "= 5.1.0"  # Exact version
    }
  }
}
```

**Version Operators:**
- `=` - Exact version
- `!=` - Exclude specific version
- `>`, `>=`, `<`, `<=` - Comparison
- `~>` - Pessimistic constraint (allow rightmost version component to increment)

---

## Resources

Resources are the most important element in Terraform. Each resource block describes one or more infrastructure objects.

### Resource Syntax

```hcl
resource "resource_type" "resource_name" {
  argument1 = value1
  argument2 = value2
  
  nested_block {
    nested_argument = value
  }
}
```

### Resource Examples

**1. AWS EC2 Instance**
```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.main.id
  
  tags = {
    Name = "WebServer"
    Environment = "Production"
  }
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              EOF
}
```

**2. AWS S3 Bucket**
```hcl
resource "aws_s3_bucket" "static_site" {
  bucket = "my-static-website-bucket"
  
  tags = {
    Name        = "StaticSite"
    Environment = "Production"
  }
}

resource "aws_s3_bucket_versioning" "static_site" {
  bucket = aws_s3_bucket.static_site.id
  
  versioning_configuration {
    status = "Enabled"
  }
}
```

**3. AWS VPC**
```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "MainVPC"
  }
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  
  tags = {
    Name = "PublicSubnet"
  }
}
```

**4. AWS Security Group**
```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-security-group"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "WebSecurityGroup"
  }
}
```

### Resource Meta-Arguments

**1. depends_on**

Explicitly specify dependencies.

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  depends_on = [aws_db_instance.database]
}
```

**2. count**

Create multiple similar resources.

```hcl
resource "aws_instance" "server" {
  count = 3
  
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "Server-${count.index}"
  }
}

# Access: aws_instance.server[0], aws_instance.server[1], etc.
```

**3. for_each**

Create multiple resources from a map or set.

```hcl
resource "aws_instance" "server" {
  for_each = {
    web  = "t2.micro"
    app  = "t2.small"
    db   = "t2.medium"
  }
  
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value
  
  tags = {
    Name = "${each.key}-server"
  }
}

# Access: aws_instance.server["web"], aws_instance.server["app"], etc.
```

**4. lifecycle**

Customize resource behavior.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  lifecycle {
    create_before_destroy = true  # Create new before destroying old
    prevent_destroy       = true  # Prevent accidental deletion
    ignore_changes        = [tags]  # Ignore changes to tags
  }
}
```

### Resource Addressing

```hcl
# Resource type.resource name
aws_instance.web_server

# With count
aws_instance.server[0]
aws_instance.server[1]

# With for_each
aws_instance.server["web"]
aws_instance.server["app"]

# Module resources
module.vpc.aws_subnet.public
```

---

## Variables

Variables allow you to parameterize your Terraform configurations, making them reusable and flexible.

### Declaring Variables

**variables.tf**
```hcl
# String variable
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

# Number variable
variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  default     = 2
}

# Boolean variable
variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}

# List variable
variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
}

# Map variable
variable "instance_types" {
  description = "Instance types for different environments"
  type        = map(string)
  default = {
    dev  = "t2.micro"
    prod = "t2.large"
  }
}

# Object variable
variable "server_config" {
  description = "Server configuration"
  type = object({
    instance_type = string
    disk_size     = number
    enable_backup = bool
  })
  default = {
    instance_type = "t2.micro"
    disk_size     = 20
    enable_backup = false
  }
}

# Variable with validation
variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# Sensitive variable
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}
```

### Using Variables

```hcl
# In resources
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_types[var.environment]
  
  tags = {
    Name        = "Server-${var.environment}"
    Environment = var.environment
  }
}
```

### Variable Types

1. **Primitive Types:**
   - `string`
   - `number`
   - `bool`

2. **Collection Types:**
   - `list(<TYPE>)` - Ordered collection
   - `set(<TYPE>)` - Unordered unique collection
   - `map(<TYPE>)` - Key-value pairs

3. **Structural Types:**
   - `object({...})` - Collection with named attributes
   - `tuple([...])` - Fixed-length ordered collection

### Assigning Variable Values

**1. Command Line**
```bash
terraform apply -var="instance_count=3" -var="environment=prod"
```

**2. Variable Definition Files (terraform.tfvars)**
```hcl
# terraform.tfvars
aws_region     = "us-west-2"
instance_count = 5
environment    = "production"
```

**3. Environment Variables**
```bash
export TF_VAR_aws_region="us-west-2"
export TF_VAR_instance_count=5
```

**4. Variable Files with Custom Names**
```bash
terraform apply -var-file="production.tfvars"
```

**Example production.tfvars:**
```hcl
environment    = "prod"
instance_count = 10
instance_type  = "t2.large"
```

### Variable Precedence (lowest to highest)

1. Environment variables
2. `terraform.tfvars`
3. `terraform.tfvars.json`
4. `*.auto.tfvars` or `*.auto.tfvars.json`
5. `-var` and `-var-file` command-line options

---

## Outputs

Output values make information about your infrastructure available for other Terraform configurations or external systems.

### Declaring Outputs

**outputs.tf**
```hcl
# Simple output
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

# Output with sensitivity
output "db_password" {
  description = "Database password"
  value       = aws_db_instance.main.password
  sensitive   = true
}

# Output from list
output "server_ips" {
  description = "IP addresses of servers"
  value       = aws_instance.server[*].public_ip
}

# Output from map
output "instance_details" {
  description = "Instance details"
  value = {
    id         = aws_instance.web.id
    public_ip  = aws_instance.web.public_ip
    private_ip = aws_instance.web.private_ip
  }
}

# Conditional output
output "load_balancer_dns" {
  description = "Load balancer DNS name"
  value       = var.create_lb ? aws_lb.main[0].dns_name : "N/A"
}
```

### Viewing Outputs

```bash
# Show all outputs after apply
terraform output

# Show specific output
terraform output instance_id

# Show as JSON
terraform output -json

# Show sensitive outputs
terraform output -json | jq .db_password.value
```

### Using Outputs in Other Modules

```hcl
# In module
module "vpc" {
  source = "./modules/vpc"
}

# Use module output
resource "aws_instance" "web" {
  subnet_id = module.vpc.subnet_id
}
```

### Output Example

**Complete example:**
```hcl
# main.tf
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "WebServer"
  }
}

resource "aws_eip" "web" {
  instance = aws_instance.web.id
  domain   = "vpc"
}

# outputs.tf
output "instance_info" {
  description = "Complete instance information"
  value = {
    instance_id = aws_instance.web.id
    public_ip   = aws_eip.web.public_ip
    private_ip  = aws_instance.web.private_ip
    az          = aws_instance.web.availability_zone
  }
}
```

**Output:**
```bash
$ terraform output
instance_info = {
  "az" = "us-east-1a"
  "instance_id" = "i-0123456789abcdef0"
  "private_ip" = "10.0.1.10"
  "public_ip" = "52.12.34.56"
}
```

---

## Data Sources

Data sources allow Terraform to fetch information from existing infrastructure or external sources.

### Data Source Syntax

```hcl
data "data_source_type" "name" {
  # Query parameters
}

# Use data source
resource "..." "..." {
  attribute = data.data_source_type.name.attribute
}
```

### Common Data Sources

**1. AWS AMI**
```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
}
```

**2. AWS VPC**
```hcl
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}
```

**3. AWS Availability Zones**
```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "primary" {
  count             = 3
  availability_zone = data.aws_availability_zones.available.names[count.index]
  cidr_block        = "10.0.${count.index}.0/24"
  vpc_id            = aws_vpc.main.id
}
```

**4. AWS IAM Policy Document**
```hcl
data "aws_iam_policy_document" "s3_policy" {
  statement {
    actions = [
      "s3:GetObject",
      "s3:ListBucket"
    ]
    
    resources = [
      aws_s3_bucket.main.arn,
      "${aws_s3_bucket.main.arn}/*"
    ]
    
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::123456789012:root"]
    }
  }
}
```

**5. Terraform Remote State**
```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  
  config = {
    bucket = "my-terraform-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.vpc.outputs.subnet_id
}
```

**6. External Data**
```hcl
data "external" "example" {
  program = ["python", "${path.module}/get-data.py"]
  
  query = {
    id = "12345"
  }
}

output "result" {
  value = data.external.example.result
}
```

**7. HTTP Data**
```hcl
data "http" "my_ip" {
  url = "https://api.ipify.org?format=json"
  
  request_headers = {
    Accept = "application/json"
  }
}

locals {
  my_ip = jsondecode(data.http.my_ip.response_body).ip
}
```

---

## State Management

Terraform state is a critical component that tracks the current state of your infrastructure.

### What is Terraform State?

The state file (`terraform.tfstate`) is a JSON file that contains:
- Resource metadata
- Resource dependencies
- Resource attributes
- Provider configuration
- Terraform version used

### Local State

By default, Terraform stores state locally in `terraform.tfstate`.

**Structure:**
```
project/
├── main.tf
├── terraform.tfstate       # Current state
└── terraform.tfstate.backup  # Previous state
```

### State Commands

```bash
# List resources in state
terraform state list

# Show resource details
terraform state show aws_instance.web

# Pull current state
terraform state pull

# Remove resource from state (doesn't destroy)
terraform state rm aws_instance.web

# Move resource to different address
terraform state mv aws_instance.web aws_instance.server

# Replace provider in state
terraform state replace-provider hashicorp/aws registry.terraform.io/hashicorp/aws
```

### Import Existing Resources

```bash
# Import existing AWS instance
terraform import aws_instance.web i-1234567890abcdef0

# Import with module
terraform import module.vpc.aws_vpc.main vpc-12345678
```

**Example import:**
```hcl
# 1. Create resource configuration
resource "aws_instance" "imported" {
  # Configuration to match existing instance
}

# 2. Import
terraform import aws_instance.imported i-1234567890abcdef0

# 3. Run terraform plan to ensure match
terraform plan
```

### State Locking

State locking prevents concurrent operations that could corrupt state.

**Supported backends with locking:**
- AWS S3 + DynamoDB
- Azure Blob Storage
- Google Cloud Storage
- Terraform Cloud
- Consul

**Manual lock/unlock:**
```bash
# Force unlock (use with caution)
terraform force-unlock <LOCK_ID>
```

### Best Practices

1. **Never edit state files manually**
2. **Use remote state for team collaboration**
3. **Enable state locking**
4. **Use workspaces for environments**
5. **Back up state files regularly**
6. **Encrypt state files (contain sensitive data)**
7. **Use version control for configuration, NOT state**

---

## Modules

Modules are containers for multiple resources that are used together. They enable code reuse and organization.

### Module Structure

```
modules/
└── vpc/
    ├── main.tf        # Resources
    ├── variables.tf   # Input variables
    ├── outputs.tf     # Output values
    └── README.md      # Documentation
```

### Creating a Module

**modules/vpc/main.tf:**
```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = var.enable_dns_support
  
  tags = merge(
    var.tags,
    {
      Name = var.name
    }
  )
}

resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]
  
  map_public_ip_on_launch = true
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-public-${count.index + 1}"
    }
  )
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-igw"
    }
  )
}
```

**modules/vpc/variables.tf:**
```hcl
variable "name" {
  description = "Name prefix for resources"
  type        = string
}

variable "cidr_block" {
  description = "CIDR block for VPC"
  type        = string
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
}

variable "enable_dns_hostnames" {
  description = "Enable DNS hostnames"
  type        = bool
  default     = true
}

variable "enable_dns_support" {
  description = "Enable DNS support"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default     = {}
}
```

**modules/vpc/outputs.tf:**
```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "internet_gateway_id" {
  description = "Internet Gateway ID"
  value       = aws_internet_gateway.main.id
}
```

### Using a Module

**main.tf:**
```hcl
module "vpc" {
  source = "./modules/vpc"
  
  name               = "production-vpc"
  cidr_block         = "10.0.0.0/16"
  public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# Use module outputs
resource "aws_instance" "web" {
  subnet_id = module.vpc.public_subnet_ids[0]
  # ... other configuration
}
```

### Module Sources

**1. Local Path**
```hcl
module "vpc" {
  source = "./modules/vpc"
}
```

**2. Terraform Registry**
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}
```

**3. GitHub**
```hcl
module "vpc" {
  source = "github.com/terraform-aws-modules/terraform-aws-vpc"
}

# Specific branch
module "vpc" {
  source = "github.com/terraform-aws-modules/terraform-aws-vpc?ref=v5.0.0"
}
```

**4. Git**
```hcl
module "vpc" {
  source = "git::https://github.com/username/terraform-modules.git//vpc?ref=v1.0.0"
}
```

**5. S3**
```hcl
module "vpc" {
  source = "s3::https://s3.amazonaws.com/my-bucket/vpc-module.zip"
}
```

### Module Best Practices

1. **Keep modules focused and single-purpose**
2. **Document inputs and outputs clearly**
3. **Use semantic versioning**
4. **Provide examples**
5. **Include README.md**
6. **Test modules independently**
7. **Use variables for customization**
8. **Avoid hardcoding values**
9. **Return useful outputs**
10. **Follow naming conventions**

### Complete Module Example

**modules/ec2-instance/main.tf:**
```hcl
resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id
  
  vpc_security_group_ids = var.security_group_ids
  
  user_data = var.user_data
  
  root_block_device {
    volume_size = var.root_volume_size
    volume_type = var.root_volume_type
    encrypted   = var.encrypt_root_volume
  }
  
  tags = merge(
    var.tags,
    {
      Name = var.name
    }
  )
}

resource "aws_eip" "this" {
  count = var.associate_public_ip ? 1 : 0
  
  instance = aws_instance.this.id
  domain   = "vpc"
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-eip"
    }
  )
}
```

**Usage:**
```hcl
module "web_server" {
  source = "./modules/ec2-instance"
  
  name          = "web-server"
  ami_id        = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  subnet_id     = module.vpc.public_subnet_ids[0]
  
  security_group_ids = [aws_security_group.web.id]
  
  associate_public_ip = true
  
  tags = {
    Environment = "Production"
  }
}
```

---

## Workspaces

Workspaces allow you to manage multiple environments (dev, staging, prod) with the same configuration.

### What are Workspaces?

Each workspace has its own state file, allowing you to use the same Terraform configuration for multiple environments.

### Workspace Commands

```bash
# List workspaces (* indicates current)
terraform workspace list

# Create new workspace
terraform workspace new dev

# Switch workspace
terraform workspace select prod

# Show current workspace
terraform workspace show

# Delete workspace (cannot delete current workspace)
terraform workspace delete staging
```

### Using Workspaces

**main.tf:**
```hcl
locals {
  environment = terraform.workspace
  
  instance_type = {
    dev     = "t2.micro"
    staging = "t2.small"
    prod    = "t2.large"
  }
  
  instance_count = {
    dev     = 1
    staging = 2
    prod    = 5
  }
}

resource "aws_instance" "server" {
  count = local.instance_count[local.environment]
  
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type[local.environment]
  
  tags = {
    Name        = "server-${local.environment}-${count.index + 1}"
    Environment = local.environment
  }
}
```

### Workspace Workflow

```bash
# Create development workspace
terraform workspace new dev
terraform apply

# Create production workspace
terraform workspace new prod
terraform apply

# Switch back to dev
terraform workspace select dev
terraform plan

# List all workspaces
terraform workspace list
#   default
#   dev
# * prod
```

### Best Practices

**When to use workspaces:**
- ✅ Same infrastructure, different environments
- ✅ Quick environment isolation
- ✅ Simple use cases

**When NOT to use workspaces:**
- ❌ Different infrastructure per environment
- ❌ Different cloud accounts/subscriptions
- ❌ Complex environment differences

**Alternative to workspaces:**
Use separate directories:
```
environments/
├── dev/
│   ├── main.tf
│   └── terraform.tfvars
├── staging/
│   ├── main.tf
│   └── terraform.tfvars
└── prod/
    ├── main.tf
    └── terraform.tfvars
```

---

## Remote State

Remote state allows teams to share Terraform state and enables state locking.

### Why Remote State?

**Problems with local state:**
- Cannot be shared with team
- Risk of data loss
- No state locking
- No collaboration

**Benefits of remote state:**
- Centralized storage
- State locking
- Team collaboration
- Encryption
- Versioning
- Access control

### Configuring Remote State

**1. AWS S3 Backend**

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

**Setup DynamoDB table for locking:**
```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**2. Azure Blob Storage Backend**

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-rg"
    storage_account_name = "terraformstatestorage"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

**3. Google Cloud Storage Backend**

```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state-bucket"
    prefix = "prod"
  }
}
```

**4. Terraform Cloud Backend**

```hcl
terraform {
  cloud {
    organization = "my-org"
    
    workspaces {
      name = "production"
    }
  }
}
```

**5. Consul Backend**

```hcl
terraform {
  backend "consul" {
    address = "consul.example.com:8500"
    path    = "terraform/state"
    lock    = true
  }
}
```

### Migrating to Remote State

**Step 1: Add backend configuration**
```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

**Step 2: Initialize with migration**
```bash
terraform init -migrate-state
```

Terraform will ask if you want to copy existing state to the new backend.

### Accessing Remote State

```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  
  config = {
    bucket = "my-terraform-state"
    key    = "vpc/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use remote state outputs
resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.vpc.outputs.private_subnet_id
}
```

### Complete S3 Backend Setup

**backend.tf:**
```hcl
# Create S3 bucket for state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-company-terraform-state"
  
  lifecycle {
    prevent_destroy = true
  }
}

# Enable versioning
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Enable encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# DynamoDB for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

---

## Provisioners

Provisioners are used to execute scripts or commands on resources after creation. **Use them as a last resort.**

### Types of Provisioners

**1. local-exec**

Executes commands on the machine running Terraform.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> ip_addresses.txt"
  }
}
```

**2. remote-exec**

Executes commands on the remote resource via SSH or WinRM.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  key_name      = "my-key"
  
  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }
  
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
      "sudo systemctl start nginx"
    ]
  }
}
```

**3. file**

Copies files or directories to the remote resource.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }
  
  provisioner "file" {
    source      = "app.conf"
    destination = "/tmp/app.conf"
  }
  
  provisioner "file" {
    content     = "ami used: ${self.ami}"
    destination = "/tmp/ami_info.txt"
  }
}
```

### Provisioner Behavior

**on_failure:**
```hcl
provisioner "remote-exec" {
  # ...
  
  on_failure = continue  # or fail (default)
}
```

**when:**
```hcl
# Run on resource creation (default)
provisioner "local-exec" {
  when    = create
  command = "echo Created"
}

# Run on resource destruction
provisioner "local-exec" {
  when    = destroy
  command = "echo Destroyed"
}
```

### Best Practices

**⚠️ Provisioners are a last resort:**
1. **Prefer cloud-init/user_data** for initial configuration
2. **Use configuration management tools** (Ansible, Chef, Puppet)
3. **Use Packer** to pre-build images
4. **Provisioners make Terraform less declarative**

**When to use provisioners:**
- Bootstrapping initial configuration
- Running scripts that can't be done via user_data
- Triggering external systems

---

## Terraform Functions

Terraform includes built-in functions for transforming and combining values.

### Numeric Functions

```hcl
# abs - Absolute value
abs(-5)  # 5

# ceil - Round up
ceil(4.3)  # 5

# floor - Round down
floor(4.9)  # 4

# max - Maximum value
max(5, 12, 9)  # 12

# min - Minimum value
min(5, 12, 9)  # 5

# pow - Power
pow(3, 2)  # 9
```

### String Functions

```hcl
# upper - Convert to uppercase
upper("hello")  # "HELLO"

# lower - Convert to lowercase
lower("HELLO")  # "hello"

# title - Title case
title("hello world")  # "Hello World"

# trimspace - Remove leading/trailing whitespace
trimspace("  hello  ")  # "hello"

# substr - Substring
substr("hello world", 0, 5)  # "hello"

# replace - Replace substring
replace("hello world", "world", "terraform")  # "hello terraform"

# split - Split string into list
split(",", "a,b,c")  # ["a", "b", "c"]

# join - Join list into string
join(",", ["a", "b", "c"])  # "a,b,c"

# format - Format string
format("Hello, %s!", "World")  # "Hello, World!"

# formatlist - Format list
formatlist("server-%s", ["a", "b", "c"])  # ["server-a", "server-b", "server-c"]
```

### Collection Functions

```hcl
# length - Length of list, map, or string
length([1, 2, 3])  # 3

# element - Get element at index
element(["a", "b", "c"], 1)  # "b"

# index - Find index of element
index(["a", "b", "c"], "b")  # 1

# contains - Check if list contains value
contains(["a", "b", "c"], "b")  # true

# concat - Concatenate lists
concat([1, 2], [3, 4])  # [1, 2, 3, 4]

# distinct - Remove duplicates
distinct([1, 2, 2, 3])  # [1, 2, 3]

# flatten - Flatten nested lists
flatten([[1, 2], [3, 4]])  # [1, 2, 3, 4]

# merge - Merge maps
merge({a = 1}, {b = 2})  # {a = 1, b = 2}

# keys - Get map keys
keys({a = 1, b = 2})  # ["a", "b"]

# values - Get map values
values({a = 1, b = 2})  # [1, 2]

# lookup - Get value from map
lookup({a = 1, b = 2}, "a", "default")  # 1

# zipmap - Create map from lists
zipmap(["a", "b"], [1, 2])  # {a = 1, b = 2}
```

### Type Conversion Functions

```hcl
# tostring
tostring(123)  # "123"

# tonumber
tonumber("123")  # 123

# tobool
tobool("true")  # true

# tolist
tolist(toset([1, 2, 2, 3]))  # [1, 2, 3]

# toset
toset([1, 2, 2, 3])  # [1, 2, 3]

# tomap
tomap({a = "1", b = "2"})
```

### Encoding Functions

```hcl
# base64encode
base64encode("Hello")  # "SGVsbG8="

# base64decode
base64decode("SGVsbG8=")  # "Hello"

# jsonencode
jsonencode({a = 1, b = 2})  # '{"a":1,"b":2}'

# jsondecode
jsondecode('{"a":1,"b":2}')  # {a = 1, b = 2}

# yamlencode
yamlencode({a = 1, b = 2})

# yamldecode
yamldecode("a: 1\nb: 2")
```

### Filesystem Functions

```hcl
# file - Read file contents
file("${path.module}/config.txt")

# fileexists - Check if file exists
fileexists("config.txt")  # true or false

# fileset - List files matching pattern
fileset(path.module, "*.tf")

# templatefile - Render template
templatefile("template.tpl", {name = "World"})
```

### Date/Time Functions

```hcl
# timestamp - Current timestamp
timestamp()  # "2024-01-15T10:30:00Z"

# formatdate - Format timestamp
formatdate("DD MMM YYYY hh:mm ZZZ", timestamp())

# timeadd - Add duration
timeadd(timestamp(), "1h")
```

### IP Network Functions

```hcl
# cidrhost - Get IP from CIDR
cidrhost("10.0.0.0/24", 5)  # "10.0.0.5"

# cidrnetmask - Get netmask
cidrnetmask("10.0.0.0/24")  # "255.255.255.0"

# cidrsubnet - Create subnet
cidrsubnet("10.0.0.0/16", 8, 1)  # "10.0.1.0/24"
```

### Practical Examples

```hcl
locals {
  # Create server names
  server_names = [for i in range(1, 4) : format("server-%02d", i)]
  # ["server-01", "server-02", "server-03"]
  
  # Create availability zone mapping
  az_subnets = zipmap(
    data.aws_availability_zones.available.names,
    ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  )
  
  # Filter instances
  prod_instances = [
    for instance in var.instances : instance
    if instance.environment == "prod"
  ]
  
  # Create tags
  common_tags = merge(
    var.tags,
    {
      ManagedBy = "Terraform"
      Timestamp = timestamp()
    }
  )
}
```

---

## Best Practices

### 1. Code Organization

**File Structure:**
```
project/
├── main.tf              # Main resources
├── variables.tf         # Input variables
├── outputs.tf           # Outputs
├── providers.tf         # Provider configuration
├── versions.tf          # Version constraints
├── terraform.tfvars     # Variable values (gitignored)
├── modules/             # Reusable modules
│   ├── vpc/
│   ├── ec2/
│   └── rds/
├── environments/        # Environment-specific configs
│   ├── dev/
│   ├── staging/
│   └── prod/
└── README.md
```

### 2. Naming Conventions

```hcl
# Resources: type_name
resource "aws_instance" "web_server"
resource "aws_s3_bucket" "static_assets"

# Variables: descriptive names
variable "instance_count"
variable "vpc_cidr_block"

# Modules: purpose
module "networking"
module "compute"

# Workspaces: environment
terraform workspace new production
```

### 3. Version Constraints

```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Allow 5.x versions
    }
  }
}
```

### 4. State Management

- ✅ Use remote state for teams
- ✅ Enable state locking
- ✅ Enable encryption
- ✅ Enable versioning
- ❌ Never commit state files to version control
- ❌ Never edit state files manually

### 5. Security

**Sensitive Variables:**
```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

output "db_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = false
}
```

**Environment Variables:**
```bash
export TF_VAR_db_password="secretpassword"
```

**Never hardcode secrets:**
```hcl
# ❌ Bad
resource "aws_db_instance" "main" {
  password = "hardcoded-password"
}

# ✅ Good
resource "aws_db_instance" "main" {
  password = var.db_password
}
```

### 6. Resource Tagging

```hcl
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Project     = var.project_name
    Owner       = var.owner
  }
}

resource "aws_instance" "web" {
  # ...
  
  tags = merge(
    local.common_tags,
    {
      Name = "web-server"
    }
  )
}
```

### 7. Use Modules

```hcl
# Reuse common patterns
module "vpc" {
  source = "./modules/vpc"
  # ...
}

module "web_server" {
  source = "./modules/ec2"
  # ...
}
```

### 8. Code Formatting

```bash
# Format all .tf files
terraform fmt -recursive

# Check formatting
terraform fmt -check
```

### 9. Validation

```bash
# Validate configuration
terraform validate

# Use variable validation
variable "environment" {
  type = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

### 10. Documentation

```hcl
# Document variables
variable "instance_type" {
  description = "EC2 instance type for web servers"
  type        = string
  default     = "t2.micro"
}

# Document outputs
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}
```

---

## CI/CD Integration

### GitHub Actions

**.github/workflows/terraform.yml:**
```yaml
name: Terraform CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.6.0
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Terraform Format
        run: terraform fmt -check
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Validate
        run: terraform validate
      
      - name: Terraform Plan
        if: github.event_name == 'pull_request'
        run: terraform plan -no-color
      
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve
```

### GitLab CI

**.gitlab-ci.yml:**
```yaml
image: hashicorp/terraform:latest

stages:
  - validate
  - plan
  - apply

before_script:
  - terraform init

validate:
  stage: validate
  script:
    - terraform fmt -check
    - terraform validate

plan:
  stage: plan
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - tfplan

apply:
  stage: apply
  script:
    - terraform apply -auto-approve tfplan
  only:
    - main
  when: manual
```

### Jenkins

**Jenkinsfile:**
```groovy
pipeline {
    agent any
    
    environment {
        AWS_CREDENTIALS = credentials('aws-credentials')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Terraform Init') {
            steps {
                sh 'terraform init'
            }
        }
        
        stage('Terraform Validate') {
            steps {
                sh 'terraform validate'
            }
        }
        
        stage('Terraform Plan') {
            steps {
                sh 'terraform plan -out=tfplan'
            }
        }
        
        stage('Terraform Apply') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Apply Terraform changes?', ok: 'Apply'
                sh 'terraform apply tfplan'
            }
        }
    }
}
```

---

## Security Best Practices

### 1. Secrets Management

**Use Secret Managers:**
```hcl
# AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

### 2. Encryption

**S3 Backend Encryption:**
```hcl
terraform {
  backend "s3" {
    bucket  = "terraform-state"
    key     = "prod/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true  # Enable encryption
  }
}
```

### 3. IAM Best Practices

**Least Privilege:**
```hcl
# Specific permissions only
data "aws_iam_policy_document" "terraform" {
  statement {
    actions = [
      "ec2:DescribeInstances",
      "ec2:RunInstances",
      "ec2:TerminateInstances"
    ]
    resources = ["*"]
  }
}
```

### 4. Network Security

```hcl
# Restrict CIDR blocks
resource "aws_security_group" "web" {
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]  # Internal only
  }
}
```

### 5. State File Security

- ✅ Encrypt state files
- ✅ Restrict access with IAM
- ✅ Enable versioning
- ✅ Use state locking
- ❌ Never commit state files

---

## Troubleshooting

### Common Issues

**1. State Lock Error**
```bash
Error: Error acquiring the state lock

# Solution: Force unlock (use with caution)
terraform force-unlock <LOCK_ID>
```

**2. Provider Configuration**
```bash
Error: Provider configuration not present

# Solution: Run terraform init
terraform init
```

**3. Resource Already Exists**
```bash
Error: resource already exists

# Solution: Import existing resource
terraform import aws_instance.web i-1234567890abcdef0
```

**4. Dependency Errors**
```bash
Error: Cycle in resource dependencies

# Solution: Review dependencies and remove cycles
# Use depends_on sparingly
```

**5. Authentication Errors**
```bash
Error: error configuring Terraform AWS Provider

# Solution: Configure credentials
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
```

### Debug Commands

```bash
# Enable detailed logging
export TF_LOG=DEBUG
export TF_LOG_PATH=terraform.log

# Levels: TRACE, DEBUG, INFO, WARN, ERROR
export TF_LOG=TRACE

# Disable logging
unset TF_LOG
```

---

## Advantages and Disadvantages

### Advantages

1. **Cloud Agnostic**
   - Works with AWS, Azure, GCP, and 100+ providers
   - Manage multi-cloud infrastructure from one tool

2. **Declarative Approach**
   - Define desired state, not steps
   - Easier to understand and maintain

3. **Version Control**
   - Infrastructure as code can be versioned
   - Track changes, rollback, collaborate

4. **Execution Plans**
   - Preview changes before applying
   - Reduce errors and surprises

5. **Resource Graph**
   - Automatically handles dependencies
   - Parallel resource creation

6. **State Management**
   - Tracks real infrastructure
   - Detects drift and changes

7. **Modularity**
   - Reusable modules
   - DRY (Don't Repeat Yourself)

8. **Large Community**
   - Extensive documentation
   - Pre-built modules
   - Active support

9. **Immutable Infrastructure**
   - Replace rather than modify
   - More reliable and consistent

10. **Provider Ecosystem**
    - 2000+ providers
    - Supports almost any API

### Disadvantages

1. **State File Management**
   - State files can become complex
   - Need proper backend configuration
   - State corruption risks

2. **Learning Curve**
   - HCL syntax to learn
   - Understanding state management
   - Provider-specific knowledge needed

3. **Limited Configuration Management**
   - Not ideal for runtime configuration
   - Need complementary tools (Ansible, Chef)

4. **Drift Detection**
   - Manual changes outside Terraform cause drift
   - Requires discipline to prevent

5. **Provider Limitations**
   - Quality varies by provider
   - Some resources not fully supported
   - Breaking changes in provider updates

6. **No Rollback**
   - No built-in rollback mechanism
   - Must maintain previous versions manually

7. **State Locking Complexity**
   - Requires additional infrastructure (DynamoDB, etc.)
   - Can cause blocking issues

8. **Testing Challenges**
   - Testing infrastructure changes is complex
   - Requires dedicated test environments

9. **Secrets Management**
   - State files contain sensitive data
   - Need external secret management

10. **Resource Dependencies**
    - Complex dependencies can be hard to manage
    - Circular dependencies cause issues

---

## Top 100 Terraform Interview Questions

### Basic Level (1-30)

**1. What is Terraform?**
Terraform is an open-source Infrastructure as Code (IaC) tool developed by HashiCorp that allows you to define, provision, and manage cloud infrastructure using declarative configuration files written in HCL (HashiCorp Configuration Language).

**2. What is Infrastructure as Code (IaC)?**
IaC is the practice of managing and provisioning infrastructure through code instead of manual processes. It enables version control, automation, consistency, and repeatability in infrastructure management.

**3. What are the main advantages of Terraform?**
- Cloud-agnostic (works with multiple providers)
- Declarative syntax (define desired state)
- Execution plans (preview changes)
- Resource graph (automatic dependency management)
- State management
- Modularity and reusability

**4. What is HCL?**
HCL (HashiCorp Configuration Language) is a declarative language designed by HashiCorp specifically for Terraform. It's human-readable and machine-friendly, used to define infrastructure resources.

**5. What are the main Terraform commands?**
- `terraform init` - Initialize working directory
- `terraform plan` - Create execution plan
- `terraform apply` - Apply changes
- `terraform destroy` - Destroy infrastructure
- `terraform validate` - Validate configuration
- `terraform fmt` - Format code

**6. What is `terraform init`?**
`terraform init` initializes a Terraform working directory by:
- Downloading provider plugins
- Initializing backend
- Downloading modules
- Creating lock files

**7. What is `terraform plan`?**
`terraform plan` creates an execution plan showing what actions Terraform will take to reach the desired state. It previews changes without actually applying them.

**8. What is `terraform apply`?**
`terraform apply` executes the actions proposed in the Terraform plan, creating, updating, or deleting resources to match the desired configuration.

**9. What is `terraform destroy`?**
`terraform destroy` destroys all resources managed by the Terraform configuration, essentially deleting the entire infrastructure.

**10. What are Terraform providers?**
Providers are plugins that enable Terraform to interact with cloud providers, SaaS providers, and other APIs. Examples include AWS, Azure, GCP, Kubernetes, and Docker providers.

**11. What is a Terraform resource?**
A resource is a component of infrastructure, such as an EC2 instance, S3 bucket, or VPC. Resources are defined using resource blocks in Terraform configuration files.

**12. What is a Terraform module?**
A module is a container for multiple resources that are used together. Modules allow code reuse, better organization, and abstraction of infrastructure patterns.

**13. What are Terraform variables?**
Variables are parameters that allow you to customize Terraform configurations without modifying the code. They can be of different types: string, number, bool, list, map, object, etc.

**14. What are Terraform outputs?**
Outputs are values that are exported from a Terraform configuration, making information about resources available for use by other configurations or external systems.

**15. What is the Terraform state file?**
The state file (`terraform.tfstate`) is a JSON file that maps Terraform configuration to real-world resources, tracks metadata, and stores resource dependencies.

**16. What is `terraform.tfvars`?**
`terraform.tfvars` is a file used to assign values to variables. It's automatically loaded by Terraform and commonly used to store environment-specific values.

**17. What is `terraform fmt`?**
`terraform fmt` automatically formats Terraform configuration files to a canonical format and style, ensuring consistency across the codebase.

**18. What is `terraform validate`?**
`terraform validate` checks the syntax and internal consistency of Terraform configuration files without accessing remote services.

**19. What is a data source in Terraform?**
A data source allows Terraform to fetch information from existing infrastructure or external sources, which can then be used in resource configurations.

**20. What is the difference between resource and data source?**
- **Resource**: Creates, updates, or deletes infrastructure
- **Data Source**: Only reads existing infrastructure or external information

**21. What is the terraform.lock.hcl file?**
The dependency lock file records provider versions and ensures consistent provider versions across different runs and team members.

**22. How do you specify provider version constraints?**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

**23. What is the difference between `=` and `~>` in version constraints?**
- `=` - Exact version only
- `~>` - Pessimistic constraint (allows rightmost version component to increment)
  - `~> 5.0` allows `5.0` through `5.x` but not `6.0`

**24. What is `count` in Terraform?**
`count` is a meta-argument that allows you to create multiple instances of a resource based on a number.

**25. What is `for_each` in Terraform?**
`for_each` is a meta-argument that allows you to create multiple instances of a resource based on a map or set, with each instance having a unique key.

**26. What is the difference between `count` and `for_each`?**
- **count**: Uses index-based addressing (`resource[0]`, `resource[1]`)
- **for_each**: Uses key-based addressing (`resource["key"]`), safer when items are reordered

**27. What are lifecycle rules in Terraform?**
Lifecycle rules customize resource behavior:
- `create_before_destroy` - Create new resource before destroying old
- `prevent_destroy` - Prevent resource deletion
- `ignore_changes` - Ignore changes to specific attributes

**28. What is `depends_on`?**
`depends_on` is a meta-argument that explicitly specifies dependencies between resources when Terraform can't automatically infer them.

**29. What is the Terraform workflow?**
1. Write - Create configuration files
2. Init - Initialize working directory
3. Validate - Check configuration
4. Plan - Preview changes
5. Apply - Execute changes
6. Destroy - Tear down infrastructure

**30. What file extensions does Terraform use?**
- `.tf` - Terraform configuration files
- `.tfvars` - Variable definition files
- `.tfstate` - State files
- `.tfstate.backup` - State backup files

### Intermediate Level (31-65)

**31. What is remote state in Terraform?**
Remote state stores the Terraform state file in a shared, remote location (like S3, Azure Blob, GCS) instead of locally, enabling team collaboration and state locking.

**32. What are the benefits of remote state?**
- Team collaboration
- State locking
- Encryption at rest
- Versioning
- Centralized storage
- Disaster recovery

**33. How do you configure S3 as a remote backend?**
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

**34. What is state locking?**
State locking prevents concurrent operations that could corrupt the state file. It ensures only one person can modify the state at a time.

**35. Which backends support state locking?**
- S3 with DynamoDB
- Azure Blob Storage
- Google Cloud Storage
- Terraform Cloud
- Consul
- etcd

**36. What are Terraform workspaces?**
Workspaces allow you to manage multiple environments (dev, staging, prod) with the same configuration, each having its own state file.

**37. How do you create and use workspaces?**
```bash
terraform workspace new dev
terraform workspace select prod
terraform workspace list
terraform workspace show
```

**38. What is the difference between workspaces and separate directories?**
- **Workspaces**: Same code, different state files (good for simple cases)
- **Directories**: Different code, different state (good for complex differences)

**39. What are provisioners in Terraform?**
Provisioners execute scripts or commands on resources after creation. Types include `local-exec`, `remote-exec`, and `file`.

**40. Why are provisioners a last resort?**
Provisioners make Terraform less declarative and can lead to:
- Configuration drift
- State management issues
- Difficult troubleshooting
- Better alternatives exist (cloud-init, Packer, Ansible)

**41. What is `local-exec` provisioner?**
Executes commands on the machine running Terraform.
```hcl
provisioner "local-exec" {
  command = "echo ${self.public_ip} >> ips.txt"
}
```

**42. What is `remote-exec` provisioner?**
Executes commands on the remote resource via SSH or WinRM.
```hcl
provisioner "remote-exec" {
  inline = [
    "sudo apt-get update",
    "sudo apt-get install -y nginx"
  ]
}
```

**43. How do you import existing infrastructure into Terraform?**
```bash
terraform import aws_instance.web i-1234567890abcdef0
```
Then create matching resource configuration and run `terraform plan` to verify.

**44. What is terraform refresh?**
`terraform refresh` updates the state file with the real-world status of resources without modifying infrastructure.

**45. What is taint in Terraform?**
Tainting marks a resource for recreation on the next apply.
```bash
terraform taint aws_instance.web
terraform apply  # Will destroy and recreate
```

**46. What replaced `terraform taint` in newer versions?**
`terraform apply -replace="aws_instance.web"`

**47. What are Terraform modules used for?**
- Code reuse
- Abstraction
- Organization
- Standardization
- Team collaboration
- Versioning

**48. How do you call a module?**
```hcl
module "vpc" {
  source = "./modules/vpc"
  
  cidr_block = "10.0.0.0/16"
  name       = "main-vpc"
}
```

**49. What are module sources?**
- Local paths (`./modules/vpc`)
- Terraform Registry (`terraform-aws-modules/vpc/aws`)
- GitHub (`github.com/user/repo`)
- Git (`git::https://...`)
- HTTP URLs
- S3 buckets

**50. How do you version modules?**
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}
```

**51. What is terraform.tfstate.backup?**
A backup of the previous state file, created automatically before each state modification.

**52. How do you manage secrets in Terraform?**
- Environment variables (`TF_VAR_*`)
- AWS Secrets Manager/Parameter Store
- Azure Key Vault
- HashiCorp Vault
- Mark variables as `sensitive = true`
- Never commit secrets to version control

**53. What is the `sensitive` argument?**
Marks variables and outputs as sensitive, hiding their values in logs and output.
```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

**54. What are dynamic blocks?**
Dynamic blocks generate nested blocks dynamically based on a collection.
```hcl
dynamic "ingress" {
  for_each = var.ingress_rules
  content {
    from_port = ingress.value.from_port
    to_port   = ingress.value.to_port
    protocol  = ingress.value.protocol
  }
}
```

**55. What are locals in Terraform?**
Locals assign names to expressions for reuse within a module.
```hcl
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

**56. What is the difference between variables and locals?**
- **Variables**: Input parameters from outside
- **Locals**: Computed values within the module

**57. What are terraform functions?**
Built-in functions for transforming and combining values:
- String functions: `upper()`, `lower()`, `replace()`
- Collection functions: `concat()`, `merge()`, `flatten()`
- Numeric functions: `min()`, `max()`, `ceil()`
- Date functions: `timestamp()`, `formatdate()`

**58. What is `terraform console`?**
An interactive console for evaluating Terraform expressions.
```bash
terraform console
> var.aws_region
"us-east-1"
```

**59. How do you handle conditional logic in Terraform?**
Use conditional expressions:
```hcl
instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"

resource "aws_eip" "web" {
  count    = var.create_eip ? 1 : 0
  instance = aws_instance.web.id
}
```

**60. What are for expressions?**
Transform collections:
```hcl
upper_names = [for name in var.names : upper(name)]

instance_map = {
  for instance in aws_instance.servers :
  instance.id => instance.private_ip
}
```

**61. What is the splat expression?**
Shorthand for extracting attribute values from all elements.
```hcl
instance_ids = aws_instance.servers[*].id
# Instead of: [for i in aws_instance.servers : i.id]
```

**62. How do you structure a Terraform project?**
```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── versions.tf
├── terraform.tfvars
├── modules/
│   ├── vpc/
│   ├── ec2/
│   └── rds/
└── environments/
    ├── dev/
    ├── staging/
    └── prod/
```

**63. What is terraform graph?**
Generates a visual representation of the resource dependency graph.
```bash
terraform graph | dot -Tpng > graph.png
```

**64. How do you debug Terraform?**
```bash
export TF_LOG=DEBUG
export TF_LOG_PATH=terraform.log
terraform apply
```
Levels: TRACE, DEBUG, INFO, WARN, ERROR

**65. What are terraform_remote_state data sources used for?**
Accessing outputs from another Terraform configuration's state.
```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "terraform-state"
    key    = "vpc/terraform.tfstate"
    region = "us-east-1"
  }
}
```

### Advanced Level (66-100)

**66. What is Terraform Cloud?**
HashiCorp's SaaS offering providing:
- Remote state management
- Remote execution
- Team collaboration
- Policy as code (Sentinel)
- Private module registry
- Cost estimation

**67. What is Sentinel in Terraform?**
Policy as code framework for Terraform Enterprise/Cloud that enforces compliance and governance rules.

**68. What are the three enforcement levels in Sentinel?**
- **Advisory**: Logged but doesn't block
- **Soft Mandatory**: Can be overridden with permissions
- **Hard Mandatory**: Cannot be overridden

**69. How do you perform a state migration?**
```bash
# 1. Update backend configuration
# 2. Run init with migrate flag
terraform init -migrate-state
```

**70. What is the difference between Terraform and CloudFormation?**
- **Terraform**: Cloud-agnostic, HCL syntax, larger community, open-source
- **CloudFormation**: AWS-only, JSON/YAML syntax, tight AWS integration, free

**71. What is the difference between Terraform and Ansible?**
- **Terraform**: Infrastructure provisioning (IaC)
- **Ansible**: Configuration management and application deployment
- Often used together

**72. How do you handle different environments (dev/staging/prod)?**
- Workspaces
- Separate directories
- Separate state files
- Variable files (dev.tfvars, prod.tfvars)
- Terragrunt

**73. What is Terragrunt?**
A thin wrapper for Terraform that provides extra tools for:
- Keeping Terraform code DRY
- Managing remote state
- Working with multiple modules
- Managing multiple environments

**74. How do you test Terraform code?**
- `terraform validate` - Syntax validation
- `terraform plan` - Execution plan review
- Terratest - Automated testing framework (Go)
- Kitchen-Terraform - Integration testing
- Checkov - Static code analysis
- tflint - Linter for Terraform

**75. What is terraform state list?**
Lists all resources in the state file.
```bash
terraform state list
```

**76. What is terraform state show?**
Shows detailed information about a specific resource.
```bash
terraform state show aws_instance.web
```

**77. What is terraform state mv?**
Moves resources in the state file (renaming or moving between modules).
```bash
terraform state mv aws_instance.old aws_instance.new
```

**78. What is terraform state rm?**
Removes resources from state (doesn't destroy actual resources).
```bash
terraform state rm aws_instance.web
```

**79. What is terraform state pull?**
Downloads and outputs the current state.
```bash
terraform state pull > backup.tfstate
```

**80. How do you upgrade Terraform providers?**
```bash
terraform init -upgrade
```

**81. What are provider aliases used for?**
Managing multiple instances of the same provider (e.g., multi-region).
```hcl
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_instance" "west_server" {
  provider = aws.west
}
```

**82. How do you handle circular dependencies?**
- Avoid them by design
- Split into separate resources
- Use data sources instead of direct references
- Restructure dependencies

**83. What is the terraform_data resource?**
A resource that doesn't create infrastructure but can trigger provisioners or track dependencies (replaces null_resource in newer versions).

**84. How do you implement blue-green deployments with Terraform?**
- Use `create_before_destroy` lifecycle
- Deploy new infrastructure
- Switch traffic (load balancer, DNS)
- Destroy old infrastructure

**85. What is the Terraform Registry?**
A public repository of Terraform providers and modules that can be used in configurations.
https://registry.terraform.io

**86. How do you create a private module registry?**
- Terraform Cloud/Enterprise
- Git repositories with specific structure
- Private registry implementation
- Artifact repository (Artifactory, Nexus)

**87. What are the different types of Terraform backends?**
- **Local**: Default, stores state locally
- **Remote**: S3, Azure Blob, GCS, Terraform Cloud
- **Enhanced**: Support remote operations (Terraform Cloud)

**88. What is the difference between Standard and Enhanced backends?**
- **Standard**: Only store state
- **Enhanced**: Store state AND run operations remotely

**89. How do you manage Terraform in a CI/CD pipeline?**
- Use remote state
- Run in containers
- Separate plan and apply stages
- Use approval gates for production
- Store credentials securely
- Use service accounts/roles

**90. What is the recommended way to structure large Terraform projects?**
- Separate by environment (folders)
- Use modules for reusability
- Split by logical boundaries (networking, compute, data)
- Use remote state for cross-module dependencies
- Implement naming conventions
- Document everything

**91. How do you perform zero-downtime deployments?**
- Use `create_before_destroy`
- Blue-green deployments
- Rolling updates
- Health checks before destroying old resources
- Load balancer integration

**92. What are Terraform expressions?**
Expressions compute values:
- Literals: `"hello"`, `123`, `true`
- References: `var.name`, `aws_instance.web.id`
- Operators: `+`, `-`, `==`, `&&`
- Function calls: `upper("hello")`
- Conditionals: `condition ? true_val : false_val`

**93. How do you handle drift in Terraform?**
```bash
# Detect drift
terraform plan

# Reconcile options:
# 1. Update Terraform config to match reality
# 2. Apply to revert manual changes
# 3. Import new resources
```

**94. What is the purpose of `.terraform.lock.hcl`?**
Ensures consistent provider versions across all users and runs by locking provider versions.

**95. How do you share data between modules?**
- Module outputs
- Remote state data sources
- Variables passing
- Locals (within same configuration)

**96. What is the null_resource?**
A resource that doesn't create infrastructure but can trigger provisioners or track dependencies (being replaced by terraform_data).

**97. How do you implement disaster recovery for Terraform?**
- Store state in versioned S3 bucket
- Enable state locking
- Regular state backups
- Document infrastructure
- Keep configuration in version control
- Test recovery procedures

**98. What are the best practices for Terraform state files?**
- Use remote state
- Enable encryption
- Enable versioning
- Use state locking
- Never edit manually
- Never commit to version control
- Regular backups

**99. How do you optimize Terraform performance?**
- Reduce parallelism for rate limits: `terraform apply -parallelism=2`
- Use `-target` for specific resources
- Split large configurations into modules
- Use data sources efficiently
- Minimize provider API calls
- Use local values for repeated expressions

**100. What is the Terraform lifecycle?**
The lifecycle of a Terraform-managed resource:
1. **Create**: Resource doesn't exist, Terraform creates it
2. **Read**: Terraform checks current state
3. **Update**: Resource exists but needs changes
4. **Delete**: Resource should be removed
5. **Replace**: Resource needs recreation (delete + create)

Each provider implements these operations (CRUD) for their resources.

---

## Conclusion

Terraform has become the industry standard for Infrastructure as Code, enabling teams to manage infrastructure efficiently, consistently, and at scale. Its cloud-agnostic nature, declarative approach, and robust ecosystem make it an essential tool for modern DevOps practices.

### Key Takeaways

**Core Strengths:**
- Declarative infrastructure management
- Multi-cloud support
- Strong state management
- Extensive provider ecosystem
- Modular and reusable code
- Version-controlled infrastructure

**Best Use Cases:**
- Multi-cloud deployments
- Infrastructure automation
- Disaster recovery
- Environment provisioning
- Compliance and governance
- Team collaboration

**When to Use Terraform:**
✅ Infrastructure provisioning  
✅ Multi-cloud management  
✅ Version-controlled infrastructure  
✅ Team collaboration  
✅ Immutable infrastructure patterns  

**When to Use Complementary Tools:**
🔄 Ansible/Chef - Runtime configuration  
🔄 Packer - Image building  
🔄 Kubernetes - Container orchestration  
🔄 CloudFormation - AWS-only with deep integration  

### Final Recommendations

1. **Start simple** - Master basics before complex patterns
2. **Use remote state** - Essential for teams
3. **Modularize** - Create reusable components
4. **Test everything** - Validate before production
5. **Document** - Clear documentation saves time
6. **Follow conventions** - Consistent naming and structure
7. **Security first** - Never expose secrets
8. **Version control** - Track all changes
9. **CI/CD integration** - Automate deployment
10. **Keep learning** - Terraform evolves constantly

Terraform expertise is highly valuable in today's cloud-focused job market. Whether you're managing infrastructure for startups or enterprises, Terraform provides the tools and flexibility needed to succeed.

---

**Document Version:** 1.0  
**Last Updated:** October 2025  
**Created For:** DevOps Engineers & Cloud Professionals  

---

## Additional Resources

### Official Documentation
- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [Terraform Registry](https://registry.terraform.io/)
- [Terraform Tutorials](https://developer.hashicorp.com/terraform/tutorials)

### Community Resources
- [Terraform GitHub](https://github.com/hashicorp/terraform)
- [Terraform Community Forum](https://discuss.hashicorp.com/c/terraform-core)
- [Stack Overflow - Terraform](https://stackoverflow.com/questions/tagged/terraform)

### Learning Platforms
- HashiCorp Learn
- A Cloud Guru
- Linux Academy
- Udemy Terraform Courses

### Tools & Extensions
- **terraform-docs** - Generate documentation
- **tflint** - Linter for Terraform
- **Terratest** - Testing framework
- **Terragrunt** - Terraform wrapper
- **Checkov** - Security scanner
- **Infracost** - Cost estimation

### Books
- "Terraform: Up & Running" by Yevgeniy Brikman
- "Terraform in Action" by Scott Winkler
- "Getting Started with Terraform" by Kirill Shirinkin

---

**End of Document**

For the latest Terraform updates and features, always refer to the official HashiCorp documentation.