# Terraform `cidrsubnet` Function - Complete Guide

## Table of Contents
1. [What is cidrsubnet?](#what-is-cidrsubnet)
2. [Function Syntax](#function-syntax)
3. [How the Calculation Works](#how-the-calculation-works)
4. [Step-by-Step Example](#step-by-step-example)
5. [Visual Breakdown](#visual-breakdown)
6. [Practical Terraform Example](#practical-terraform-example)
7. [IP Assignment in AWS](#ip-assignment-in-aws)
8. [Key Formulas](#key-formulas)
9. [Real-World Use Case](#real-world-use-case)

---

## What is cidrsubnet?

The `cidrsubnet` function in Terraform dynamically calculates subnet CIDR blocks by dividing a larger network range into smaller subnets using bitwise arithmetic. Its primary purpose is to automate the subnetting of IP ranges when provisioning cloud infrastructure.

**Key Benefits:**
- Automates subnet allocation and avoids manual errors
- Scales dynamically for any number of subnets or zones
- Integrates seamlessly with Terraform resource creation patterns
- Guarantees non-overlapping subnet ranges

---

## Function Syntax

```hcl
cidrsubnet(prefix, newbits, netnum)
```

### Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| **prefix** | The base CIDR block (your VPC range) | `"10.0.0.0/16"` |
| **newbits** | Number of extra bits to add to the network mask | `4` |
| **netnum** | Index of the subnet you want to create (starts at 0) | `0`, `1`, `2`, etc. |

---

## How the Calculation Works

### Basic Concept

**Starting Point:** Your VPC has `10.0.0.0/16`, which contains 65,536 IP addresses.

**Goal:** Divide this into smaller `/20` subnets, each containing 4,096 IPs.

**Process:**
1. Add `newbits` to the original prefix length
2. Calculate how many subnets can be created
3. Calculate the size of each subnet
4. Determine the starting address for the requested subnet index

---

## Step-by-Step Example

Let's calculate: `cidrsubnet("10.0.0.0/16", 4, 2)`

### Step 1: Calculate the New Subnet Mask
- Original prefix: `/16`
- New bits to add: `4`
- **New prefix length:** `/16 + 4 = /20`
- Each subnet will be a `/20` network

### Step 2: Determine How Many Subnets Possible
- Formula: `2^newbits = 2^4 = 16 subnets`
- You can create subnet indexes **0 through 15**

### Step 3: Calculate Size of Each Subnet
- Formula: `2^(32 - new_prefix_length)`
- Calculation: `2^(32 - 20) = 2^12 = 4,096 IP addresses per subnet`
- **Usable IPs:** 4,094 (excluding network and broadcast addresses)

### Step 4: Calculate Starting Address for Subnet Index 2
- Formula: `Base network address + (netnum × subnet_size)`
- Calculation: `10.0.0.0 + (2 × 4,096) = 10.0.0.0 + 8,192`
- Converting to IP: `8,192 ÷ 256 = 32` (this goes in the third octet)
- **Result:** `10.0.32.0/20`

### Step 5: Final Result
```
cidrsubnet('10.0.0.0/16', 4, 2) = 10.0.32.0/20
```

**This subnet contains:**
- Network address: `10.0.32.0`
- First usable IP: `10.0.32.1`
- Last usable IP: `10.0.47.254`
- Broadcast address: `10.0.47.255`
- Total IPs: `4,096`
- Usable IPs: `4,094`

---

## Visual Breakdown

### All 16 Possible Subnets from 10.0.0.0/16

```
VPC: 10.0.0.0/16 (65,536 IPs)
├── Subnet 0:  10.0.0.0/20    (10.0.0.0   - 10.0.15.255)
├── Subnet 1:  10.0.16.0/20   (10.0.16.0  - 10.0.31.255)
├── Subnet 2:  10.0.32.0/20   (10.0.32.0  - 10.0.47.255)  ← Example above
├── Subnet 3:  10.0.48.0/20   (10.0.48.0  - 10.0.63.255)
├── Subnet 4:  10.0.64.0/20   (10.0.64.0  - 10.0.79.255)
├── Subnet 5:  10.0.80.0/20   (10.0.80.0  - 10.0.95.255)
├── Subnet 6:  10.0.96.0/20   (10.0.96.0  - 10.0.111.255)
├── Subnet 7:  10.0.112.0/20  (10.0.112.0 - 10.0.127.255)
├── Subnet 8:  10.0.128.0/20  (10.0.128.0 - 10.0.143.255)
├── Subnet 9:  10.0.144.0/20  (10.0.144.0 - 10.0.159.255)
├── Subnet 10: 10.0.160.0/20  (10.0.160.0 - 10.0.175.255)
├── Subnet 11: 10.0.176.0/20  (10.0.176.0 - 10.0.191.255)
├── Subnet 12: 10.0.192.0/20  (10.0.192.0 - 10.0.207.255)
├── Subnet 13: 10.0.208.0/20  (10.0.208.0 - 10.0.223.255)
├── Subnet 14: 10.0.224.0/20  (10.0.224.0 - 10.0.239.255)
└── Subnet 15: 10.0.240.0/20  (10.0.240.0 - 10.0.255.255)
```

### First 3 Subnets - Detailed Breakdown

| Subnet | CIDR | IP Range | Total IPs | Usable IPs |
|--------|------|----------|-----------|------------|
| **Subnet 0** | `10.0.0.0/20` | `10.0.0.0` → `10.0.15.255` | 4,096 | 4,094 |
| **Subnet 1** | `10.0.16.0/20` | `10.0.16.0` → `10.0.31.255` | 4,096 | 4,094 |
| **Subnet 2** | `10.0.32.0/20` | `10.0.32.0` → `10.0.47.255` | 4,096 | 4,094 |

### Overlap Check
✅ **NO OVERLAPS** - All subnets are completely separate!
- Each subnet has its own exclusive IP range
- No IP address exists in more than one subnet
- Subnets are sequential and non-overlapping

---

## Practical Terraform Example

### Your Terraform Code

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "public" {
  count             = length(data.aws_availability_zones.available.names)
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.env}-${local.pub_subnet_name}-${count.index + 1}"
    Type = "Public"
  }
}
```

### How It Works in Practice

Assuming you have **3 Availability Zones** and `var.vpc_cidr = "10.0.0.0/16"`:

**Terraform creates:**

```
Subnet 0 (count.index = 0):
  cidrsubnet(var.vpc_cidr, 4, 0)
  → cidrsubnet('10.0.0.0/16', 4, 0)
  → 10.0.0.0/20
  → Placed in us-east-1a

Subnet 1 (count.index = 1):
  cidrsubnet(var.vpc_cidr, 4, 1)
  → cidrsubnet('10.0.0.0/16', 4, 1)
  → 10.0.16.0/20
  → Placed in us-east-1b

Subnet 2 (count.index = 2):
  cidrsubnet(var.vpc_cidr, 4, 2)
  → cidrsubnet('10.0.0.0/16', 4, 2)
  → 10.0.32.0/20
  → Placed in us-east-1c
```

### Each Subnet Automatically Gets:
✓ A unique, non-overlapping CIDR block  
✓ 4,094 usable IP addresses  
✓ Placement in a different Availability Zone  
✓ No manual calculation needed!

---

## IP Assignment in AWS

### How AWS Assigns IPs to Resources

When you launch resources (EC2 instances, RDS databases, etc.) in a subnet:

1. **You choose the subnet** where the resource will be placed
2. **AWS automatically assigns a random, available IP** from that subnet's range
3. **The IP is unique** and won't conflict with other resources
4. **You can optionally specify a static IP** if needed

### Example:
If you launch an EC2 instance in Subnet 1 (`10.0.16.0/20`):
- AWS will assign any available IP between `10.0.16.1` and `10.0.31.254`
- Example assigned IPs: `10.0.20.145`, `10.0.25.78`, `10.0.31.200`
- Each IP belongs to **exactly one subnet** - no overlaps!

### Reserved IPs
AWS reserves 5 IPs in each subnet:
- **First IP** (e.g., `10.0.16.0`): Network address
- **Second IP** (e.g., `10.0.16.1`): VPC router
- **Third IP** (e.g., `10.0.16.2`): DNS server
- **Fourth IP** (e.g., `10.0.16.3`): Reserved for future use
- **Last IP** (e.g., `10.0.31.255`): Broadcast address

---

## Key Formulas

### Essential Calculations

| Formula | Purpose | Example |
|---------|---------|---------|
| `New Prefix = Original Prefix + newbits` | Calculate subnet mask | `/16 + 4 = /20` |
| `Subnet Count = 2^newbits` | Total possible subnets | `2^4 = 16` |
| `Subnet Size = 2^(32 - New Prefix)` | IPs per subnet | `2^(32-20) = 4,096` |
| `Start Address = Base + (netnum × Subnet Size)` | Calculate subnet start | `10.0.0.0 + (2 × 4,096)` |

### Quick Reference Table

| newbits | New Prefix (from /16) | Subnets Created | IPs per Subnet |
|---------|----------------------|-----------------|----------------|
| 1 | /17 | 2 | 32,768 |
| 2 | /18 | 4 | 16,384 |
| 3 | /19 | 8 | 8,192 |
| **4** | **/20** | **16** | **4,096** |
| 5 | /21 | 32 | 2,048 |
| 6 | /22 | 64 | 1,024 |
| 7 | /23 | 128 | 512 |
| 8 | /24 | 256 | 256 |

---

## Real-World Use Case

### Complete VPC Setup with Public and Private Subnets

```hcl
############## Locals ###########################################
locals {
  pub_subnet_name = "Public-Jai"
  pvt_subnet_name = "Private-Jai"
}

################# Availability Zone #############################
data "aws_availability_zones" "available" {
  state = "available"
}

################# Creating VPC ###################################
resource "aws_vpc" "my_vpc" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  
  tags = {
    Name = "${var.env}-vpc-${var.prefix}"
  }
}

################ Creating PUBLIC SUBNET ###########################
resource "aws_subnet" "public" {
  count             = length(data.aws_availability_zones.available.names)
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.env}-${local.pub_subnet_name}-${count.index + 1}"
    Type = "Public"
  }
}

################ Creating PRIVATE SUBNET #################################
resource "aws_subnet" "private" {
  count             = length(data.aws_availability_zones.available.names)
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index + length(data.aws_availability_zones.available.names))
  availability_zone = data.aws_availability_zones.available.names[count.index]
 
  tags = {
    Name = "${var.env}-${local.pvt_subnet_name}-${count.index + 1}"
    Type = "Private"
  }
}
```

### What This Creates (with 3 AZs)

**Public Subnets:**
- Subnet 0: `10.0.0.0/20` in us-east-1a
- Subnet 1: `10.0.16.0/20` in us-east-1b
- Subnet 2: `10.0.32.0/20` in us-east-1c

**Private Subnets:**
- Subnet 3: `10.0.48.0/20` in us-east-1a
- Subnet 4: `10.0.64.0/20` in us-east-1b
- Subnet 5: `10.0.80.0/20` in us-east-1c

### Why This Pattern Works
- **Dynamic:** Adapts to any region with any number of AZs
- **Scalable:** Works with any VPC CIDR range
- **Organized:** Public and private subnets are clearly separated
- **Fault-tolerant:** Distributes across multiple AZs
- **Maintainable:** No hardcoded IP addresses

---

## Summary

### Key Takeaways

1. **`cidrsubnet` automates subnet creation** - No manual IP calculations needed
2. **Each subnet is a complete IP range** - Not a single IP, but thousands of IPs
3. **Subnets never overlap** - Mathematically guaranteed to be separate
4. **AWS assigns random IPs** - From the available pool in each subnet
5. **Perfect for multi-AZ deployments** - Combine with `count` for automatic distribution

### Think of It Like This:
- Your VPC is like a **city** (10.0.0.0/16)
- Each subnet is a **neighborhood** (10.0.0.0/20, 10.0.16.0/20, etc.)
- Each IP is a **house address** in that neighborhood
- No house can be in two neighborhoods at once!

---

## Additional Resources

- **Terraform Documentation:** [cidrsubnet function](https://developer.hashicorp.com/terraform/language/functions/cidrsubnet)
- **AWS VPC Guide:** [Subnets for your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)
- **CIDR Calculator:** Use online tools to visualize subnet ranges

---

**Created:** October 28, 2025  
**Last Updated:** October 28, 2025