# AWS VPC Terraform Module - Complete Technical Guide

## Table of Contents
1. [VPC Module Overview](#vpc-module-overview)
2. [Understanding AWS VPC Architecture](#understanding-aws-vpc-architecture)
3. [Module Components Breakdown](#module-components-breakdown)
4. [How VPC Routing Works](#how-vpc-routing-works)
5. [Configuration Guide](#configuration-guide)
6. [Real-Time Examples](#real-time-examples)
7. [Networking Concepts Deep Dive](#networking-concepts-deep-dive)

---

## VPC Module Overview

Your VPC module is a **Terraform Infrastructure-as-Code (IaC)** configuration that creates a **production-grade AWS Virtual Private Cloud (VPC)** with the following infrastructure:

- **1 VPC** with configurable CIDR block
- **Multiple Private Subnets** (dynamically created)
- **1 Internet Gateway** for public internet connectivity
- **2 Route Tables** (Main and Public)
- **1 Public Route** connecting to the Internet Gateway
- **Subnet-to-Route Table Associations** for traffic management

### Key Principle: Separation of Concerns

This module follows AWS best practices by:
- Creating **separate route tables** for different subnet types
- Using **dynamic resource creation** with `for_each` to reduce code duplication
- Maintaining **explicit routing logic** to control traffic flow
- Supporting **multi-AZ deployments** for high availability

---

## Understanding AWS VPC Architecture

### What is a VPC?

A **Virtual Private Cloud (VPC)** is a logically isolated section of AWS cloud infrastructure where you can launch your resources (EC2 instances, RDS databases, etc.). It's like having your own private data center within AWS.

```
┌─────────────────────────────────────────────────────┐
│            AWS Region (us-east-1)                   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  VPC (10.0.0.0/16)                           │   │
│  │                                              │   │
│  │  ┌──────────────┐    ┌──────────────┐       │   │
│  │  │ Private      │    │ Private      │       │   │
│  │  │ Subnet (AZ-A)│    │ Subnet (AZ-B)│       │   │
│  │  │ 10.0.1.0/24  │    │ 10.0.2.0/24  │       │   │
│  │  └──────────────┘    └──────────────┘       │   │
│  │         │                    │                │   │
│  │         └────────────────────┘                │   │
│  │               ↓ (Implicit Router)             │   │
│  │         Route Tables                          │   │
│  │         (Local routing)                       │   │
│  │                                              │   │
│  │  ┌──────────────────────────────────────┐   │   │
│  │  │ Internet Gateway                     │   │   │
│  │  │ (IGW) - 0.0.0.0/0 route             │   │   │
│  │  └──────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────┘   │
│                    ↓                                 │
│         ┌──────────────────────┐                    │
│         │   Internet (WWW)     │                    │
│         │   0.0.0.0/0          │                    │
│         └──────────────────────┘                    │
└─────────────────────────────────────────────────────┘
```

### Core VPC Components

| Component | Purpose | Example in Your Module |
|-----------|---------|----------------------|
| **CIDR Block** | IP address range for entire VPC | `var.vpc_cidr` (e.g., `10.0.0.0/16`) |
| **Subnets** | Subdivisions of VPC within specific AZs | `aws_subnet.private` with multiple CIDR blocks |
| **Route Table** | Rules for routing traffic from subnet | `aws_route_table.main`, `aws_route_table.public` |
| **Routes** | Specific forwarding rules in route tables | `aws_route.public_internet_route` |
| **Internet Gateway** | Connection point to internet | `aws_internet_gateway.this` |
| **Network Interfaces** | Virtual network cards for EC2 instances | Created automatically when instances launch |

---

## Module Components Breakdown

### 1. VPC Resource Block

```hcl
resource "aws_vpc" "this" {
  cidr_block                           = var.vpc_cidr
  enable_dns_support                   = var.status_enable_dns_support
  enable_dns_hostnames                 = var.enable_dns_hostnames
  instance_tenancy                     = "default"
  assign_generated_ipv6_cidr_block     = false
  enable_network_address_usage_metrics = false

  tags = var.tags
}
```

#### Parameter Explanation

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `cidr_block` | `var.vpc_cidr` | Defines IP address range (e.g., `10.0.0.0/16` = 65,536 IP addresses) |
| `enable_dns_support` | Variable | Allows instances to use AWS DNS (Route 53) for domain name resolution |
| `enable_dns_hostnames` | Variable | Enables instances to receive public DNS hostnames |
| `instance_tenancy` | `"default"` | Instances run on shared hardware (vs. `dedicated` for isolation) |
| `assign_generated_ipv6_cidr_block` | `false` | IPv6 not enabled (only IPv4 used in this module) |
| `enable_network_address_usage_metrics` | `false` | Disables CloudWatch metrics for IP usage tracking |
| `tags` | `var.tags` | Metadata labels for cost allocation and organization |

#### How It Works

When Terraform applies this configuration:
1. AWS creates a new VPC with the specified CIDR block
2. An **implicit router** is automatically created within the VPC
3. A **main route table** is automatically created for the VPC
4. A **network ACL** (Network Access Control List) is automatically created for security
5. A **DHCP option set** is automatically associated for IP address assignment

---

### 2. Private Subnets (Dynamic Creation)

```hcl
resource "aws_subnet" "private" {
  for_each = var.private_subnets

  vpc_id                  = aws_vpc.this.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = each.value.map_public_ip_on_launch

  tags = each.value.tags
}
```

#### Understanding `for_each`

The `for_each` loop creates **multiple subnets dynamically** based on input variables.

**Example Input Variable:**
```hcl
variable "private_subnets" {
  type = map(object({
    cidr                      = string
    az                        = string
    map_public_ip_on_launch   = bool
    tags                      = map(string)
  }))

  default = {
    "private-subnet-1a" = {
      cidr                    = "10.0.1.0/24"
      az                      = "us-east-1a"
      map_public_ip_on_launch = false
      tags = {
        Name = "Private Subnet AZ-A"
        Type = "Private"
      }
    }
    "private-subnet-1b" = {
      cidr                    = "10.0.2.0/24"
      az                      = "us-east-1b"
      map_public_ip_on_launch = false
      tags = {
        Name = "Private Subnet AZ-B"
        Type = "Private"
      }
    }
  }
}
```

#### Loop Mechanics

When Terraform processes the `for_each`:
```
Iteration 1: each.key = "private-subnet-1a"
             each.value = {cidr: "10.0.1.0/24", az: "us-east-1a", ...}

Iteration 2: each.key = "private-subnet-1b"
             each.value = {cidr: "10.0.2.0/24", az: "us-east-1b", ...}
```

**Result:** 2 separate `aws_subnet` resources are created, accessible as:
- `aws_subnet.private["private-subnet-1a"]`
- `aws_subnet.private["private-subnet-1b"]`

#### Parameter Explanation

| Parameter | Example Value | Purpose |
|-----------|-----------------|---------|
| `vpc_id` | `aws_vpc.this.id` | Links subnet to the VPC |
| `cidr_block` | `"10.0.1.0/24"` | Defines subnet IP range (256 addresses, 251 usable) |
| `availability_zone` | `"us-east-1a"` | Physical data center location for redundancy |
| `map_public_ip_on_launch` | `false` | Prevents auto-assignment of public IPs (private subnet) |
| `tags` | Map of key-value pairs | For resource identification and cost tracking |

#### CIDR Block Breakdown

For subnet `10.0.1.0/24`:
```
Network Address:        10.0.1.0      (Reserved)
AWS Router:             10.0.1.1      (Reserved)
DNS Server:             10.0.1.2      (Reserved)
Reserved AWS:           10.0.1.3      (Reserved)
Usable IPs:             10.0.1.4 - 10.0.1.254  (251 available for resources)
Broadcast Address:      10.0.1.255    (Reserved)
```

---

### 3. Internet Gateway (Dynamic)

```hcl
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = var.igw_tags
}
```

#### What is an Internet Gateway?

An **Internet Gateway (IGW)** is a VPC component that:
1. **Connects** your VPC to the public internet
2. **Routes** internet traffic to resources within the VPC
3. **Translates** private IPs to public IPs (with Elastic IPs)
4. **Highly available** - AWS manages redundancy automatically

#### How It Works

```
┌─────────────────────────────────────┐
│   EC2 Instance                      │
│   Private IP: 10.0.1.50             │
│   Public IP: 52.12.34.56 (Elastic)  │
└──────────┬──────────────────────────┘
           │
           ↓ (NAT mapping)
┌──────────────────────────────────────┐
│   Internet Gateway                   │
│   ┌──────────────────────────────────┤
│   │ Translation Table:               │
│   │ 10.0.1.50 ↔ 52.12.34.56         │
│   │ 10.0.1.51 ↔ 52.12.34.57         │
│   └──────────────────────────────────┤
└──────────┬──────────────────────────┘
           │
           ↓
    ┌─────────────┐
    │  Internet   │
    │  (0.0.0.0/0)│
    └─────────────┘
```

---

### 4. Main Route Table

```hcl
resource "aws_route_table" "main" {
  vpc_id = aws_vpc.this.id
  tags   = var.main_rt_tags
}
```

#### What is the Main Route Table?

The **main route table** is automatically created with every VPC and is implicitly associated with all subnets unless explicitly overridden.

**Default Main Route Table Content:**
```
Destination      Target    Status
10.0.0.0/16      local     active
```

This **local route** enables:
- All instances within the VPC to communicate with each other
- Instances in `10.0.1.0/24` to reach instances in `10.0.2.0/24`
- No internet access (no route to `0.0.0.0/0`)

#### When to Use Main Route Table

AWS best practice is to **keep the main route table private** (no internet route) and explicitly associate custom route tables with public subnets.

---

### 5. Public Route Table

```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  tags   = var.public_rt_tags
}
```

#### Purpose

Creates a **custom route table** that will be explicitly associated with subnets that need internet access.

**This route table is initially empty and receives routes via separate `aws_route` resources.**

---

### 6. Public Route (Internet Access)

```hcl
resource "aws_route" "public_internet_route" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.this.id
}
```

#### Parameter Explanation

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `route_table_id` | `aws_route_table.public.id` | Which route table gets this route |
| `destination_cidr_block` | `"0.0.0.0/0"` | **All IPv4 traffic** (0.0.0.0 = 0, /0 = all 32 bits) |
| `gateway_id` | `aws_internet_gateway.this.id` | Send matching traffic to Internet Gateway |

#### What This Route Does

```
Packet Destination IP:  8.8.8.8 (Google DNS)
├─ Does it match 10.0.0.0/16? NO
└─ Does it match 0.0.0.0/0?   YES ✓
   └─ Action: Send to Internet Gateway
      └─ IGW translates and routes to internet
```

#### Route Selection Algorithm (Longest Prefix Match)

AWS uses the **longest prefix match** to select routes:

```
Route Table Entries:
1. 10.0.0.0/16   → local
2. 0.0.0.0/0     → IGW

Packet to 10.0.1.50:
├─ Matches /16? YES (length 16)
├─ Matches /0?  YES (length 0)
└─ Select: /16 (longer) → local (stay in VPC)

Packet to 8.8.8.8:
├─ Matches /16? NO
├─ Matches /0?  YES (length 0)
└─ Select: /0 → IGW (send to internet)
```

---

### 7. Route Table Associations

```hcl
resource "aws_route_table_association" "public_assoc" {
  for_each = var.subnet_associations

  subnet_id      = each.value.subnet_id
  route_table_id = aws_route_table.public.id
}
```

#### What Does This Do?

Explicitly associates subnets with the **public route table**, overriding the implicit main route table association.

**Example Input:**
```hcl
variable "subnet_associations" {
  type = map(object({
    subnet_id = string
  }))

  default = {
    "public-assoc-1a" = {
      subnet_id = aws_subnet.private["private-subnet-1a"].id
    }
    "public-assoc-1b" = {
      subnet_id = aws_subnet.private["private-subnet-1b"].id
    }
  }
}
```

#### Association Flow

```
Before Association:
┌──────────────┐
│ Subnet 1     │ → (implicitly) → Main Route Table (local only)
└──────────────┘

After Association:
┌──────────────┐
│ Subnet 1     │ → (explicitly) → Public Route Table (local + IGW)
└──────────────┘
```

#### Key Points

1. **One subnet = One explicit association** (implicit main association is removed)
2. **One subnet = Many route tables possible** (but only one active at a time)
3. **Multiple subnets = One route table** (efficient shared routing)

---

## How VPC Routing Works

### Packet Flow Through Route Table

When an EC2 instance sends a packet, this is what happens:

#### Step 1: Instance Generates Packet
```
Source:      10.0.1.50 (Instance in Private Subnet 1A)
Destination: 8.8.8.8   (Google DNS)
Protocol:    ICMP (ping)
```

#### Step 2: Instance Consults Subnet's Route Table
The instance looks up which route table is associated with its subnet.

**For private-subnet-1a:**
```
Route Table: "public" (explicit association)
Routes:
  - 10.0.0.0/16   → local (target VPC)
  - 0.0.0.0/0     → IGW   (target Internet)
```

#### Step 3: Route Matching Algorithm
```
Does destination 8.8.8.8 match 10.0.0.0/16?
├─ Check network bits: 8.8.8.8 in binary = 00001000.00001000.00001000.00001000
├─ 10.0.0.0 in binary   = 00001010.00000000.00000000.00000000
├─ First 16 bits match? NO
└─ Result: NO MATCH

Does destination 8.8.8.8 match 0.0.0.0/0?
├─ 0.0.0.0/0 matches EVERYTHING (all 0 bits = matches any 32 bits)
└─ Result: YES MATCH ✓

Longest Prefix Match Decision:
├─ Route 1: /16 (matches? NO)
├─ Route 2: /0  (matches? YES)
└─ Selected: Route 2 (/0 is most specific available match)
```

#### Step 4: Forward to Target
```
Selected Route: 0.0.0.0/0 → IGW
Action: Forward packet to Internet Gateway
```

#### Step 5: Internet Gateway Processing
```
IGW Operations:
├─ Extract destination IP: 8.8.8.8
├─ Check: "Does this belong to my VPC?" NO
├─ Action: Enable internet routing
├─ NAT (if using Elastic IP): 10.0.1.50 → 52.12.34.56
└─ Forward packet to internet backbone
```

#### Step 6: Return Traffic
```
Response from 8.8.8.8 arrives at IGW (destination: 52.12.34.56)
IGW:
├─ Reverse NAT: 52.12.34.56 → 10.0.1.50
├─ Check route table of destination subnet
├─ See local route for 10.0.0.0/16
└─ Forward to EC2 instance (10.0.1.50)
```

### Complete Route Table Example

```hcl
# Main Route Table (Private Subnets)
Route Table: main
┌────────────────┬──────────┐
│ Destination    │ Target   │
├────────────────┼──────────┤
│ 10.0.0.0/16    │ local    │
└────────────────┴──────────┘
Effect: ✓ Intra-VPC traffic
        ✗ Internet access

# Public Route Table
Route Table: public
┌────────────────┬──────────┐
│ Destination    │ Target   │
├────────────────┼──────────┤
│ 10.0.0.0/16    │ local    │
│ 0.0.0.0/0      │ IGW      │
└────────────────┴──────────┘
Effect: ✓ Intra-VPC traffic
        ✓ Internet access (with public IP)
```

---

## Configuration Guide

### Real-Time Configuration Example

#### Step 1: Define Variables

Create `variables.tf`:

```hcl
variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

variable "status_enable_dns_support" {
  type        = bool
  description = "Enable DNS support"
  default     = true
}

variable "enable_dns_hostnames" {
  type        = bool
  description = "Enable DNS hostnames"
  default     = true
}

variable "tags" {
  type        = map(string)
  description = "Common tags for all resources"
  default = {
    Environment = "production"
    Project     = "MyApp"
    ManagedBy   = "Terraform"
  }
}

variable "igw_tags" {
  type        = map(string)
  description = "Tags for Internet Gateway"
  default = {
    Name = "MyApp-IGW"
  }
}

variable "main_rt_tags" {
  type        = map(string)
  description = "Tags for main route table"
  default = {
    Name = "MyApp-Main-RT"
    Type = "Private"
  }
}

variable "public_rt_tags" {
  type        = map(string)
  description = "Tags for public route table"
  default = {
    Name = "MyApp-Public-RT"
    Type = "Public"
  }
}

variable "private_subnets" {
  type = map(object({
    cidr                    = string
    az                      = string
    map_public_ip_on_launch = bool
    tags                    = map(string)
  }))
  description = "Configuration for private subnets"
  default = {
    "private-subnet-1a" = {
      cidr                    = "10.0.1.0/24"
      az                      = "us-east-1a"
      map_public_ip_on_launch = false
      tags = {
        Name = "Private Subnet AZ-A"
        Type = "Private"
      }
    }
    "private-subnet-1b" = {
      cidr                    = "10.0.2.0/24"
      az                      = "us-east-1b"
      map_public_ip_on_launch = false
      tags = {
        Name = "Private Subnet AZ-B"
        Type = "Private"
      }
    }
    "private-subnet-1c" = {
      cidr                    = "10.0.3.0/24"
      az                      = "us-east-1c"
      map_public_ip_on_launch = false
      tags = {
        Name = "Private Subnet AZ-C"
        Type = "Private"
      }
    }
  }
}

variable "subnet_associations" {
  type = map(object({
    subnet_id = string
  }))
  description = "Subnets to associate with public route table"
  default = {
    "public-assoc-1a" = {
      subnet_id = "" # Will be filled dynamically
    }
    "public-assoc-1b" = {
      subnet_id = "" # Will be filled dynamically
    }
  }
}
```

#### Step 2: Create Main Module

Create `main.tf` with your module code (as provided).

#### Step 3: Create terraform.tfvars for Customization

```hcl
vpc_cidr = "10.0.0.0/16"

private_subnets = {
  "app-subnet-1a" = {
    cidr                    = "10.0.10.0/24"
    az                      = "us-east-1a"
    map_public_ip_on_launch = false
    tags = {
      Name        = "App Tier - AZ-A"
      Type        = "Private"
      Environment = "production"
    }
  }
  "app-subnet-1b" = {
    cidr                    = "10.0.11.0/24"
    az                      = "us-east-1b"
    map_public_ip_on_launch = false
    tags = {
      Name        = "App Tier - AZ-B"
      Type        = "Private"
      Environment = "production"
    }
  }
  "db-subnet-1a" = {
    cidr                    = "10.0.20.0/24"
    az                      = "us-east-1a"
    map_public_ip_on_launch = false
    tags = {
      Name        = "Database Tier - AZ-A"
      Type        = "Private"
      Environment = "production"
    }
  }
  "db-subnet-1b" = {
    cidr                    = "10.0.21.0/24"
    az                      = "us-east-1b"
    map_public_ip_on_launch = false
    tags = {
      Name        = "Database Tier - AZ-B"
      Type        = "Private"
      Environment = "production"
    }
  }
}

subnet_associations = {
  "app-assoc-1a" = {
    subnet_id = aws_subnet.private["app-subnet-1a"].id
  }
  "app-assoc-1b" = {
    subnet_id = aws_subnet.private["app-subnet-1b"].id
  }
}
```

#### Step 4: Deploy with Terraform

```bash
# Initialize Terraform (downloads AWS provider)
terraform init

# Plan the deployment (shows what will be created)
terraform plan

# Apply the configuration (creates AWS resources)
terraform apply

# Output created resource IDs
terraform output
```

---

## Real-Time Examples

### Example 1: Multi-Tier Application Network

**Architecture Goal:** Create a 3-tier network for a web application

```
┌─────────────────────────────────────────────────┐
│          VPC (10.0.0.0/16)                      │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │ Public Tier (Auto Scaling Web Servers)   │   │
│  │ Subnets: 10.0.100.0/24 (AZ-A)           │   │
│  │          10.0.101.0/24 (AZ-B)           │   │
│  │ Route Table: public (0.0.0.0/0 → IGW)   │   │
│  └──────────────────────────────────────────┘   │
│                      ↓                           │
│  ┌──────────────────────────────────────────┐   │
│  │ App Tier (Protected Servers)             │   │
│  │ Subnets: 10.0.10.0/24 (AZ-A)            │   │
│  │          10.0.11.0/24 (AZ-B)            │   │
│  │ Route Table: private (local only)        │   │
│  └──────────────────────────────────────────┘   │
│                      ↓                           │
│  ┌──────────────────────────────────────────┐   │
│  │ Database Tier (RDS Multi-AZ)             │   │
│  │ Subnets: 10.0.20.0/24 (AZ-A)            │   │
│  │          10.0.21.0/24 (AZ-B)            │   │
│  │ Route Table: private (local only)        │   │
│  └──────────────────────────────────────────┘   │
│                                                 │
│  Internet Gateway (IGW)                         │
│  ├─ Public subnets can reach internet           │
│  └─ Private subnets stay internal               │
└─────────────────────────────────────────────────┘
```

**Terraform Configuration:**

```hcl
variable "private_subnets" {
  type = map(object({
    cidr                    = string
    az                      = string
    map_public_ip_on_launch = bool
    tags                    = map(string)
  }))

  default = {
    # Public Tier
    "public-web-1a" = {
      cidr                    = "10.0.100.0/24"
      az                      = "us-east-1a"
      map_public_ip_on_launch = true  # Public IPs assigned
      tags = {
        Name = "Public Web Tier - AZ-A"
        Tier = "Public"
      }
    }
    "public-web-1b" = {
      cidr                    = "10.0.101.0/24"
      az                      = "us-east-1b"
      map_public_ip_on_launch = true
      tags = {
        Name = "Public Web Tier - AZ-B"
        Tier = "Public"
      }
    }

    # App Tier
    "private-app-1a" = {
      cidr                    = "10.0.10.0/24"
      az                      = "us-east-1a"
      map_public_ip_on_launch = false  # No public IPs
      tags = {
        Name = "Private App Tier - AZ-A"
        Tier = "App"
      }
    }
    "private-app-1b" = {
      cidr                    = "10.0.11.0/24"
      az                      = "us-east-1b"
      map_public_ip_on_launch = false
      tags = {
        Name = "Private App Tier - AZ-B"
        Tier = "App"
      }
    }

    # Database Tier
    "private-db-1a" = {
      cidr                    = "10.0.20.0/24"
      az                      = "us-east-1a"
      map_public_ip_on_launch = false
      tags = {
        Name = "Private DB Tier - AZ-A"
        Tier = "Database"
      }
    }
    "private-db-1b" = {
      cidr                    = "10.0.21.0/24"
      az                      = "us-east-1b"
      map_public_ip_on_launch = false
      tags = {
        Name = "Private DB Tier - AZ-B"
        Tier = "Database"
      }
    }
  }
}

# Only public subnets associated with public route table
variable "subnet_associations" {
  type = map(object({
    subnet_id = string
  }))

  default = {
    "assoc-public-web-1a" = {
      subnet_id = aws_subnet.private["public-web-1a"].id
    }
    "assoc-public-web-1b" = {
      subnet_id = aws_subnet.private["public-web-1b"].id
    }
  }
}
```

### Example 2: Packet Routing Simulation

**Scenario:** Web server in public subnet sends traffic to RDS in private subnet

```
EC2 (public-web-1a): 10.0.100.50
Task: Connect to RDS database at 10.0.20.10

Step 1: Check Route Table (Associated: public)
┌────────────────┬──────────┐
│ Destination    │ Target   │
├────────────────┼──────────┤
│ 10.0.0.0/16    │ local    │ ← Destination 10.0.20.10 matches here!
│ 0.0.0.0/0      │ IGW      │
└────────────────┴──────────┘

Step 2: Longest Prefix Match
├─ 10.0.0.0/16 (/16) matches 10.0.20.10? YES ✓
└─ 0.0.0.0/0 (/0) matches 10.0.20.10? YES (but less specific)
   └─ Winner: 10.0.0.0/16 (/16 is longer/more specific)

Step 3: Route Action
Target: local → Send packet within VPC to 10.0.20.10

Step 4: Delivery
EC2 in public-web-1a (10.0.100.50)
                    ↓
         AWS Virtual Router
                    ↓
RDS in private-db-1a (10.0.20.10)

Result: Database traffic stays within VPC (secure, no internet routing)
```

---

## Networking Concepts Deep Dive

### CIDR Notation Explained

#### What is CIDR?

**Classless Inter-Domain Routing (CIDR)** is a method of allocating IP addresses and specifying IP address ranges using a compact notation.

#### CIDR Notation: `a.b.c.d/n`

```
10.0.0.0/16
│ │ │ │ │└─ Prefix Length (number of network bits)
│ │ │ │
│ └─ IP Address (4 octets = 32 bits total)

/16 means: First 16 bits are network, last 16 bits are host
         : 2^16 = 65,536 total IP addresses
         : 2^(32-16) = 2^16 addresses available
```

#### Subnet Mask Conversion

```
/16 CIDR → 255.255.0.0 Subnet Mask

CIDR /24 breakdown:
10.0.1.0/24
├─ Binary: 00001010.00000000.00000001.[00000000-11111111]
├─ Network bits: 24 bits = First 3 octets (10.0.1.0)
├─ Host bits: 8 bits = Last octet (0-255 = 256 addresses)
└─ Usable: 256 - 5 reserved = 251 addresses

Reserved IP addresses in 10.0.1.0/24:
├─ 10.0.1.0       → Network address
├─ 10.0.1.1       → AWS VPC router (implicit gateway)
├─ 10.0.1.2       → AWS DNS server
├─ 10.0.1.3       → Reserved by AWS
├─ 10.0.1.4-254   → Available for instances
└─ 10.0.1.255     → Broadcast address
```

#### CIDR Block Sizing Guide

```
┌─────────┬──────────────┬──────────────┬──────────────┐
│ CIDR    │ Subnet Mask  │ Total Hosts  │ Usable IPs   │
├─────────┼──────────────┼──────────────┼──────────────┤
│ /16     │ 255.255.0.0  │ 65,536       │ 65,531       │
│ /17     │ 255.255.128.0│ 32,768       │ 32,763       │
│ /18     │ 255.255.192.0│ 16,384       │ 16,379       │
│ /19     │ 255.255.224.0│ 8,192        │ 8,187        │
│ /20     │ 255.255.240.0│ 4,096        │ 4,091        │
│ /21     │ 255.255.248.0│ 2,048        │ 2,043        │
│ /22     │ 255.255.252.0│ 1,024        │ 1,019        │
│ /23     │ 255.255.254.0│ 512          │ 507          │
│ /24     │ 255.255.255.0│ 256          │ 251          │
│ /25     │ 255.255.255.1│ 128          │ 123          │
│ /26     │ 255.255.255.2│ 64           │ 59           │
│ /27     │ 255.255.255.3│ 32           │ 27           │
│ /28     │ 255.255.255.4│ 16           │ 11           │
└─────────┴──────────────┴──────────────┴──────────────┘
```

### Availability Zones (AZ) Distribution

#### What are Availability Zones?

**Availability Zones (AZs)** are isolated data centers within an AWS Region, each with independent power, cooling, and networking.

#### Multi-AZ Deployment Benefits

```
┌──────────────────────────────────────────────────────────┐
│         AWS Region (us-east-1)                           │
├──────────────────┬──────────────────┬──────────────────┤
│   Availability   │  Availability    │  Availability    │
│   Zone A         │  Zone B          │  Zone C          │
│ (us-east-1a)    │ (us-east-1b)    │ (us-east-1c)    │
│                 │                  │                  │
│ ┌─────────────┐ │ ┌──────────────┐ │ ┌──────────────┐ │
│ │ Subnet 1A   │ │ │ Subnet 1B    │ │ │ Subnet 1C    │ │
│ │ 10.0.1.0/24 │ │ │ 10.0.2.0/24  │ │ │ 10.0.3.0/24  │ │
│ │ EC2, RDS    │ │ │ EC2, RDS     │ │ │ EC2, RDS     │ │
│ └─────────────┘ │ └──────────────┘ │ └──────────────┘ │
│       ↓         │        ↓         │        ↓         │
│    Data        │     Data         │     Data        │
│   Center 1     │    Center 2      │    Center 3     │
│                │                  │                  │
├──────────────────┴──────────────────┴──────────────────┤
│              AWS Network Backbone                       │
└──────────────────────────────────────────────────────────┘

Benefits:
✓ High Availability: If AZ-A fails, AZ-B and AZ-C continue
✓ Low Latency: AZs connected via dedicated high-speed network
✓ Disaster Recovery: Geographic distribution of data
✓ Load Distribution: Spread traffic across multiple AZs
```

### Dynamic Resource Creation Best Practices

#### Why Use `for_each`?

```hcl
# ❌ Bad: Code duplication (not scalable)
resource "aws_subnet" "subnet_1a" {
  vpc_id            = aws_vpc.this.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  tags = { Name = "Subnet 1A" }
}

resource "aws_subnet" "subnet_1b" {
  vpc_id            = aws_vpc.this.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"
  tags = { Name = "Subnet 1B" }
}

resource "aws_subnet" "subnet_1c" {
  vpc_id            = aws_vpc.this.id
  cidr_block        = "10.0.3.0/24"
  availability_zone = "us-east-1c"
  tags = { Name = "Subnet 1C" }
}

# ✓ Good: DRY principle (Don't Repeat Yourself)
resource "aws_subnet" "private" {
  for_each = var.private_subnets

  vpc_id            = aws_vpc.this.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
  tags              = each.value.tags
}
```

#### Accessing `for_each` Created Resources

```hcl
# Access specific subnet
resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.private["private-subnet-1a"].id
  route_table_id = aws_route_table.public.id
}

# Access all subnets in a loop
output "all_subnet_ids" {
  value = {
    for key, subnet in aws_subnet.private :
    key => subnet.id
  }
}

# Filter subnets (example: only private subnets)
output "private_subnet_cidrs" {
  value = {
    for key, subnet in aws_subnet.private :
    key => subnet.cidr_block
    if contains(["private"], subnet.tags["Type"])
  }
}
```

### Security: Private vs. Public Subnets

#### Private Subnet Characteristics

```
Private Subnet: No direct internet access
┌────────────────────────────────────┐
│ 10.0.1.0/24                        │
│ Route Table: main                  │
├─────────────────┬──────────────────┤
│ Destination     │ Target           │
├─────────────────┼──────────────────┤
│ 10.0.0.0/16     │ local            │
│ (no other route)│                  │
└─────────────────┴──────────────────┘

Implications:
❌ Instances cannot reach internet (0.0.0.0/0 route missing)
❌ Internet cannot reach instances directly
✓ Instances can reach each other (within VPC)
✓ Secure for databases, app servers, etc.
```

#### Public Subnet Characteristics

```
Public Subnet: Internet access via IGW
┌────────────────────────────────────┐
│ 10.0.100.0/24                      │
│ Route Table: public                │
├─────────────────┬──────────────────┤
│ Destination     │ Target           │
├─────────────────┼──────────────────┤
│ 10.0.0.0/16     │ local            │
│ 0.0.0.0/0       │ IGW              │
└─────────────────┴──────────────────┘

Implications:
✓ Instances can reach internet
✓ Internet can reach instances (with public IP)
✓ Suitable for web servers, ALBs, etc.
⚠ Requires careful security group configuration
```

---

## Terraform Execution Flow

### How Terraform Applies This Configuration

```
1. Parse Configuration
   ├─ Read main.tf, variables.tf, terraform.tfvars
   └─ Validate syntax and resource types

2. Build Resource Graph
   ├─ aws_vpc.this (no dependencies)
   ├─ aws_internet_gateway.this (depends on VPC)
   ├─ aws_subnet.private (depends on VPC, creates N resources via for_each)
   ├─ aws_route_table.main (depends on VPC)
   ├─ aws_route_table.public (depends on VPC)
   ├─ aws_route.public_internet_route (depends on public route table + IGW)
   └─ aws_route_table_association.public_assoc (depends on subnets + public RT)

3. Plan Phase (terraform plan)
   ├─ Query AWS for existing resources
   ├─ Compare with Terraform state
   ├─ Generate plan (add/modify/delete)
   └─ Display changes for review

4. Apply Phase (terraform apply)
   ├─ Create AWS VPC
   ├─ Create IGW and attach to VPC
   ├─ Create all subnets (parallel execution, for_each)
   ├─ Create route tables
   ├─ Add routes to public route table
   ├─ Associate subnets with route tables
   └─ Update Terraform state file

5. Output Phase
   ├─ Display resource IDs
   ├─ Generate outputs (if defined in outputs.tf)
   └─ Write state to terraform.tfstate
```

---

## Troubleshooting Guide

### Common Issues and Solutions

#### Issue 1: "InvalidParameterValue" - CIDR block overlap

```
Error: subnet cidr block overlap
Reason: Subnet CIDR blocks cannot overlap within same VPC

Solution:
vpc_cidr = 10.0.0.0/16

Correct subnets:
├─ 10.0.1.0/24    ✓
├─ 10.0.2.0/24    ✓
└─ 10.0.3.0/24    ✓

Incorrect (overlap):
├─ 10.0.1.0/24    ✗
└─ 10.0.1.128/25  ✗ (overlaps with 10.0.1.0/24)
```

#### Issue 2: Instances in private subnet cannot reach internet

```
Problem: You expect instances to have internet access

Diagnosis:
1. Check subnet's route table
   ├─ Is there a route with target 0.0.0.0/0? NO
   └─ Then subnet is truly private

2. Check IGW attachment
   ├─ aws_internet_gateway.this.vpc_id = aws_vpc.this.id ✓
   └─ If missing, IGW not attached

3. Check public IP
   ├─ Instance needs public IP or Elastic IP
   └─ map_public_ip_on_launch = true (or assign manually)

Solution: Either:
a) Add IGW route (make subnet public): Update route table
b) Add NAT Gateway (keep subnet private): New resource needed
```

#### Issue 3: "DependencyViolation" - Cannot delete VPC

```
Error: The VPC 'vpc-xxx' has dependencies and cannot be destroyed

Reason: Resources must be deleted in dependency order

Solution: terraform destroy automatically handles order
If manual deletion needed:
1. Delete route table associations
2. Delete routes
3. Delete route tables
4. Detach internet gateway
5. Delete subnets
6. Delete VPC
```

---

## Quick Reference: Terraform Commands

```bash
# Initialize (first time setup)
terraform init

# Validate configuration
terraform validate

# Format code (HCL style guide)
terraform fmt -recursive

# Plan changes
terraform plan
terraform plan -out=tfplan

# Apply configuration
terraform apply
terraform apply tfplan

# Show current state
terraform show
terraform state list
terraform state show aws_vpc.this

# Destroy resources
terraform destroy

# Output values
terraform output
terraform output vpc_id
```

---

## Summary

### Module Purpose
This VPC module creates a **complete, production-ready VPC network** with:
- **VPC isolation** for your AWS resources
- **Multi-subnet architecture** for resource organization
- **Internet connectivity** via IGW for public subnets
- **Traffic routing** with intelligent route tables
- **High availability** via multi-AZ deployment

### Key Concepts Recap

| Concept | Purpose | Your Module |
|---------|---------|------------|
| **VPC** | Network isolation | `aws_vpc.this` |
| **Subnet** | AZ-specific address ranges | `aws_subnet.private` (for_each) |
| **Route Table** | Traffic routing rules | `aws_route_table.main/public` |
| **Routes** | Specific forwarding rules | `aws_route.public_internet_route` |
| **IGW** | Internet connectivity | `aws_internet_gateway.this` |
| **Association** | Links subnets to route tables | `aws_route_table_association` |

### Best Practices Applied

✓ **Separation of concerns**: Main RT (private) vs Public RT (internet)
✓ **Code reusability**: `for_each` for dynamic subnet creation
✓ **Multi-AZ**: Subnets span multiple availability zones
✓ **Explicit associations**: Subnets explicitly tied to route tables
✓ **Tagging**: All resources tagged for organization and billing

---

## Additional Resources

- **AWS VPC Documentation**: https://docs.aws.amazon.com/vpc/
- **Terraform AWS Provider**: https://registry.terraform.io/providers/hashicorp/aws/latest
- **VPC CIDR Calculator**: https://cidr.xyz/
- **AWS Whitepapers**: AWS Well-Architected Framework
