# Friday, 27 February 2026
# Session 35 - Terraform Naming Conventions, Module Design Standards & AWS VPC Networking Fundamentals (IPv4, CIDR, Subnets & Gateways)
## Comprehensive Class Notes, Module Code Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [Multi-Environment Recap (Workspaces vs Separate Repos)](#multi-environment-recap-workspaces-vs-separate-repos)
   - [Why Terraform Modules?](#why-terraform-modules)
   - [Terraform Naming Conventions & Best Practices](#terraform-naming-conventions--best-practices)
   - [Virtual Private Cloud (VPC) Core Concepts](#virtual-private-cloud-vpc-core-concepts)
   - [IPv4 Addressing & Binary Math Deep-Dive](#ipv4-addressing--binary-math-deep-dive)
   - [CIDR Notation & Subnetting Calculations](#cidr-notation--subnetting-calculations)
   - [The 5 AWS Reserved IP Addresses](#the-5-aws-reserved-ip-addresses)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct File Links](#directory-structure--direct-file-links)
   - [Architectural Diagram: 3-Tier Enterprise VPC](#architectural-diagram-3-tier-enterprise-vpc)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Project 1: Enterprise VPC Module (`terraform-aws-vpc/`)](#project-1-enterprise-vpc-module-terraform-aws-vpc)
     - [`main.tf` (VPC, IGW, 3-Tier Subnets)](#maintf-in-terraform-aws-vpc)
     - [`variables.tf` (Validation & CIDR Lists)](#variablestf-in-terraform-aws-vpc)
     - [`locals.tf` & `data.tf`](#localstf--datatf-in-terraform-aws-vpc)
   - [Project 2: Security Group Module (`terraform-aws-sg/`)](#project-2-security-group-module-terraform-aws-sg)
     - [`main.tf`](#maintf-in-terraform-aws-sg)
   - [Project 3: VPC Module Consumer (`vpc-module-test/`)](#project-3-vpc-module-consumer-vpc-module-test)
     - [`vpc.tf`](#vpctf-in-vpc-module-test)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Testing the Reusable VPC Module](#1-testing-the-reusable-vpc-module)
   - [2. Inspecting Subnets and Gateways via AWS CLI](#2-inspecting-subnets-and-gateways-via-aws-cli)
   - [3. Clean Up (Teardown)](#3-clean-up-teardown)
5. [Commands & CLI Flags Reference Table](#5-commands--cli-flags-reference-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### Multi-Environment Recap (Workspaces vs Separate Repos)
- **Workspaces Recap**:
  - `terraform.workspace` variable: `dev`, `uat`, `prod`.
  - Commands: `terraform workspace new dev`, `terraform workspace select dev`.
  - Disadvantages: Mistakes can bleed into production, shared AWS account, blast radius is very high.
- **Tfvars & Backends Recap**:
  - `terraform init -reconfigure -backend-config=dev/backend.config`
  - `terraform plan -var-file=dev/dev.tfvars`
  - `terraform apply -auto-approve -var-file=dev/terraform.tfvars`
  - CI/CD automation tools (Jenkins, GitLab CI) are critical to eliminate human typing mistakes.
- **Separate Repositories (Enterprise Isolation)**:
  - `roboshop-infra-dev` (Non-Prod AWS Account)
  - `roboshop-infra-prod` (Prod AWS Account)
  - Disadvantage: Duplicate code across repositories.
  - **The Solution**: Reusable **Terraform Modules**.

---

### Why Terraform Modules?
A module is a container for multiple resources that are used together.
1. **Code Reuse**: Write once, deploy across dozens of microservices.
2. **Enforce Standards**: Embed corporate tagging, encryption, and network security policies into the module so consumers cannot create insecure infra.
3. **Centralized Updates**: Bug fixes and security patches are released centrally in the module repository.
4. **Simplified Maintenance**: Reduces thousands of lines of copy-pasted HCL down to clean 10-line `module` invocation blocks.

---

### Terraform Naming Conventions & Best Practices
Adhering to official HashiCorp standard conventions:

1. **Programmatic Identifiers vs Human Display Names**:
   - **Underscores (`_`)** are for internal program code identifiers:
     - `module "name_of_module" { ... }`
     - `resource "aws_instance" "catalogue" { ... }`
     - Variable names: `ami_id`, `vpc_id`, `sg_ids`.
   - **Hyphens (`-`)** are for human-facing cloud resource names and tags:
     - `Name = "roboshop-dev-catalogue"`
     - Security group name: `allow-all-terraform`.
2. **The "Single Resource" Rule (`this` or `main`)**:
   - If a child module contains only **one primary resource** of a given type, name that resource `"this"` or `"main"`:
     ```hcl
     # Inside module terraform-aws-instance/ec2.tf:
     resource "aws_instance" "this" { ... }

     # Inside module terraform-aws-vpc/main.tf:
     resource "aws_vpc" "main" { ... }
     ```
   - *Why?* When consumers query module outputs, writing `aws_instance.this.id` or `aws_vpc.main.id` avoids stuttering names like `aws_instance.instance.id` or `aws_vpc.vpc.id`.
3. **Singular vs Plural Variable Names**:
   - **1 Resource / Single Item $\rightarrow$ Singular Name**:
     - `ami_id`, `vpc_id`, `instance_type`, `environment`.
   - **Multiple Items / List / Set $\rightarrow$ Plural Name**:
     - `sg_ids`, `public_subnet_cidrs`, `private_subnet_cidrs`, `az_names`.

---

### Virtual Private Cloud (VPC) Core Concepts
- **What is a VPC?**
  - An isolated virtual network dedicated to your AWS account within a specific AWS Region.
  - It resembles a physical data center that you control, but with cloud elasticity.
- **The Village Construction Analogy**:
  - **Data Center / Region** $\rightarrow$ Selecting the geographic city/state.
  - **VPC** $\rightarrow$ Selecting an isolated village boundary with a distinct perimeter fence.
  - **Subnets** $\rightarrow$ Dividing the village into individual streets for organization, traffic control, and safety.
  - **Servers (EC2 Instances)** $\rightarrow$ Constructing houses on those streets with door numbers.
  - **Internet Gateway (IGW) / Modem** $\rightarrow$ The main grand arch/gate connecting the village roads to the public world outside.
  - **CIDR Block** $\rightarrow$ The postal pincode defining the total range of house numbers available.
- **Subnet Types**:
  - **Public Subnet**: Connected directly to the Internet Gateway (IGW). Instances receive Public IPs; accessible by end users (e.g., Load Balancers, Web Frontends).
  - **Private Subnet**: No direct route to the IGW. Isolated from inbound internet traffic; outbound traffic routes via a NAT Gateway (e.g., Backend microservices like Catalogue, Cart, User).
  - **Database Subnet**: Completely private subnets with strict security groups hosting databases (MongoDB, MySQL, Redis, RabbitMQ).

---

### IPv4 Addressing & Binary Math Deep-Dive

#### Number Systems:
- **Unary**: `1`, `11`, `111` (Base 1)
- **Binary**: `0`, `1` (Base 2)
- **Decimal**: `0`, `1`, `2`, `3`, `4`, `5`, `6`, `7`, `8`, `9` (Base 10)

#### Place Value in Decimal:
Consider number `168`:
- $8 \times 10^0 = 8$
- $6 \times 10^1 = 60$
- $1 \times 10^2 = 100$
- Total = $100 + 60 + 8 = 168$.

#### IPv4 Structure:
- Total size: **32 bits**.
- Divided into **4 Octets** (each octet contains 8 bits, separated by dots):
  `Octet1 . Octet2 . Octet3 . Octet4`
- Each octet represents a decimal value from `0` to `255` ($2^8 = 256$ possibilities).

#### Binary to Decimal Conversion Table (8-bit Octet):
```text
Bit Position:    7     6     5     4     3     2     1     0
Power of 2:     2^7   2^6   2^5   2^4   2^3   2^2   2^1   2^0
Decimal Value:  128   64    32    16     8     4     2     1
```

#### Example 1: Converting `10110111` to Decimal:
$$1 \times 128 + 0 \times 64 + 1 \times 32 + 1 \times 16 + 0 \times 8 + 1 \times 4 + 1 \times 2 + 1 \times 1$$
$$= 128 + 0 + 32 + 16 + 0 + 4 + 2 + 1 = 183$$

#### Example 2: Converting `192.168.145.132` to Binary:
- $192 = 128 + 64 \rightarrow \mathbf{11000000}$
- $168 = 128 + 32 + 8 \rightarrow \mathbf{10101000}$
- $145 = 128 + 16 + 1 \rightarrow \mathbf{10010001}$
- $132 = 128 + 4 \rightarrow \mathbf{10000100}$
- **Full Binary Representation**:
  `11000000.10101000.10010001.10000100`

#### Total IPv4 Address Space:
$$2^{32} = 4,294,967,296 \text{ (approximately 4.29 Billion Addresses)}$$

---

### CIDR Notation & Subnetting Calculations
CIDR stands for **Classless Inter-Domain Routing**.

$$\text{IP Address} = \text{Network Bits (Prefix)} + \text{Host Bits}$$

- The `/` slash notation indicates how many bits are locked for the **Network**, leaving the remainder for **Hosts**:
  $$\text{Host Bits} = 32 - \text{Prefix}$$
  $$\text{Total IP Addresses} = 2^{\text{Host Bits}}$$

#### 1. `/16` CIDR Block (e.g., `10.0.0.0/16`):
- Network Bits = `16`
- Host Bits = $32 - 16 = 16$
- Total IPs = $2^{16} = \mathbf{65,536 \text{ addresses}}$
- Standard size for an AWS VPC.

#### 2. `/24` CIDR Block (e.g., `10.0.1.0/24` or Home Wi-Fi `192.168.1.0/24`):
- Network Bits = `24`
- Host Bits = $32 - 24 = 8$
- Total IPs = $2^8 = \mathbf{256 \text{ addresses}}$
- Standard size for AWS Subnets.

#### 3. Partitioning a `/16` VPC into `/24` Subnets:
From a single `10.0.0.0/16` VPC, you can carve out **256 distinct `/24` subnets**:
- Subnet 1: `10.0.0.0/24`
- Subnet 2: `10.0.1.0/24`
- Subnet 3: `10.0.2.0/24`
- ...
- Subnet 256: `10.0.255.0/24`

---

### The 5 AWS Reserved IP Addresses
In any AWS subnet (unlike standard on-prem networking which reserves only 2: Network & Broadcast), **AWS automatically reserves 5 IP addresses**:

| Reserved IP | Example (`10.0.1.0/24`) | Purpose in AWS |
| :--- | :--- | :--- |
| **First IP (.0)** | `10.0.1.0` | **Network Address**: Identifies the subnet boundary. |
| **Second IP (.1)** | `10.0.1.1` | **VPC Router**: Default gateway for internal routing. |
| **Third IP (.2)** | `10.0.1.2` | **Amazon Route 53 DNS Resolver**: Internal AWS DNS server. |
| **Fourth IP (.3)** | `10.0.1.3` | **Future Use**: Reserved by AWS for future expansion. |
| **Last IP (.255)** | `10.0.1.255` | **Network Broadcast**: AWS does not support broadcast, but reserves it. |

> [!IMPORTANT]
> **Usable IP Formula in AWS**:
> $$\text{Usable IPs} = 2^{(32 - \text{Prefix})} - 5$$
> For a `/24` subnet: $256 - 5 = \mathbf{251 \text{ usable IP addresses}}$.
> For a `/28` subnet (AWS minimum allowed): $16 - 5 = \mathbf{11 \text{ usable IP addresses}}$.

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct File Links
Session 35 concepts are implemented in real code in your workspace. Click below to open them directly in your IDE:

#### Project 1: Reusable VPC Child Module
- **Directory**: [terraform-aws-vpc/](../../terraform-aws-vpc)
- **VPC & Subnets**: [terraform-aws-vpc/main.tf](../../terraform-aws-vpc/main.tf)
- **Variables**: [terraform-aws-vpc/variables.tf](../../terraform-aws-vpc/variables.tf)
- **Locals**: [terraform-aws-vpc/locals.tf](../../terraform-aws-vpc/locals.tf)
- **Data Source (AZs)**: [terraform-aws-vpc/data.tf](../../terraform-aws-vpc/data.tf)
- **Outputs**: [terraform-aws-vpc/outputs.tf](../../terraform-aws-vpc/outputs.tf)
- **Module Architecture Docs**: [terraform-aws-vpc/README.md](../../terraform-aws-vpc/README.md)

#### Project 2: Security Group Child Module
- **Directory**: [terraform-aws-sg/](../../terraform-aws-sg)
- **SG Resource**: [terraform-aws-sg/main.tf](../../terraform-aws-sg/main.tf)
- **Variables**: [terraform-aws-sg/variable.tf](../../terraform-aws-sg/variable.tf)

#### Project 3: VPC Module Consumer (Root Test Module)
- **Directory**: [vpc-module-test/](../../vpc-module-test)
- **Module Invocation**: [vpc-module-test/vpc.tf](../../vpc-module-test/vpc.tf)
- **Provider**: [vpc-module-test/provider.tf](../../vpc-module-test/provider.tf)
- **Variables**: [vpc-module-test/variables.tf](../../vpc-module-test/variables.tf)

---

### Architectural Diagram: 3-Tier Enterprise VPC

```
+---------------------------------------------------------------------------------------------------+
|                                 AWS REGION: us-east-1 (VPC CIDR: 10.0.0.0/16)                     |
|                                                                                                   |
|                               +-----------------------------------+                               |
|                               |    Internet Gateway (IGW: main)   |                               |
|                               +-----------------------------------+                               |
|                                                 ▲                                                 |
|                   ┌─────────────────────────────┴─────────────────────────────┐                   |
|                   ▼                                                           ▼                   |
|    +-----------------------------+                             +-----------------------------+    |
|    | Availability Zone us-east-1a|                             | Availability Zone us-east-1b|    |
|    |                             |                             |                             |    |
|    |  PUBLIC SUBNET (Tier 1)     |                             |  PUBLIC SUBNET (Tier 1)     |    |
|    |  CIDR: 10.0.1.0/24 (251 IPs)|                             |  CIDR: 10.0.2.0/24 (251 IPs)|    |
|    |  [ALB / Public Facing UI]   |                             |  [ALB / Public Facing UI]   |    |
|    +-----------------------------+                             +-----------------------------+    |
|                   │                                                           │                   |
|                   ▼                                                           ▼                   |
|    +-----------------------------+                             +-----------------------------+    |
|    |  PRIVATE SUBNET (Tier 2)    |                             |  PRIVATE SUBNET (Tier 2)    |    |
|    |  CIDR: 10.0.11.0/24 (251 IPs|                             |  CIDR: 10.0.12.0/24 (251 IPs|    |
|    |  [Catalogue, Cart, User...] |                             |  [Catalogue, Cart, User...] |    |
|    +-----------------------------+                             +-----------------------------+    |
|                   │                                                           │                   |
|                   ▼                                                           ▼                   |
|    +-----------------------------+                             +-----------------------------+    |
|    |  DATABASE SUBNET (Tier 3)   |                             |  DATABASE SUBNET (Tier 3)   |    |
|    |  CIDR: 10.0.21.0/24 (251 IPs|                             |  CIDR: 10.0.22.0/24 (251 IPs|    |
|    |  [MongoDB, MySQL, Redis]    |                             |  [MongoDB, MySQL, Redis]    |    |
|    +-----------------------------+                             +-----------------------------+    |
|                                                                                                   |
+---------------------------------------------------------------------------------------------------+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Project 1: Enterprise VPC Module (`terraform-aws-vpc/`)

#### [`main.tf` in `terraform-aws-vpc/`](../../terraform-aws-vpc/main.tf)

```hcl
1: resource "aws_vpc" "main" {
2:   cidr_block           = var.vpc_cidr
3:   instance_tenancy     = "default"
4:   enable_dns_hostnames = true
5: 
6:   tags = local.vpc_final_tags
7: }
8: 
9: resource "aws_internet_gateway" "main" {
10:   vpc_id = aws_vpc.main.id # VPC association
11: 
12:   tags = local.igw_final_tags
13: }
14: 
15: # Public Subnets
16: resource "aws_subnet" "public" {
17:   count                   = length(var.public_subnet_cidrs)
18:   vpc_id                  = aws_vpc.main.id
19:   cidr_block              = var.public_subnet_cidrs[count.index]
20:   availability_zone       = local.az_names[count.index]
21:   map_public_ip_on_launch = true
22: 
23:   tags = merge(
24:     local.common_tags,
25:     {
26:       Name = "${var.project}-${var.environment}-public-${local.az_names[count.index]}"
27:     },
28:     var.public_subnet_tags
29:   )
30: }
```

##### Line-by-Line Breakdown:
- **Lines 1-7 (`resource "aws_vpc" "main"`)**:
  - `main`: Adheres to HashiCorp standard convention (single primary resource named `"main"` or `"this"`).
  - `enable_dns_hostnames = true`: Required for EC2 instances to receive AWS public and private DNS hostnames.
- **Lines 9-13 (`resource "aws_internet_gateway" "main"`)**:
  - Attached directly to the newly created VPC via `vpc_id = aws_vpc.main.id`.
- **Lines 16-30 (`resource "aws_subnet" "public"`)**:
  - `count = length(var.public_subnet_cidrs)`: Dynamically generates subnets across list items (`10.0.1.0/24`, `10.0.2.0/24`).
  - `availability_zone = local.az_names[count.index]`: Automatically spreads subnets across multiple AZs (`us-east-1a`, `us-east-1b`) using data source queries.
  - `map_public_ip_on_launch = true`: Designates these subnets as **Public**, automatically assigning public IPs to instances launched here.
  - `tags`: Merges common enterprise tags with dynamically computed name: `${var.project}-${var.environment}-public-${local.az_names[count.index]}`.

---

#### [`variables.tf` in `terraform-aws-vpc/`](../../terraform-aws-vpc/variables.tf)

```hcl
1: variable "project" {
2:   type = string
3: }
4: 
5: variable "environment" {
6:   type = string
7:   validation {
8:     condition     = contains(["dev", "qa", "uat", "prod"], var.environment)
9:     error_message = "Environments should be one of dev, qa, uat or prod"
10:   }
11: }
12: 
13: variable "vpc_cidr" {
14:   type    = string
15:   default = "10.0.0.0/16"
16: }
17: 
28: variable "public_subnet_cidrs" {
29:   type    = list(any)
30:   default = ["10.0.1.0/24", "10.0.2.0/24"]
31: }
```
- **Lines 7-10 (`validation` block)**: Enforces input validation. Prevents invalid environment names at plan time before touching AWS APIs.
- **Line 15 (`default = "10.0.0.0/16"`)**: Default `/16` network CIDR offering 65,536 total IPs.

---

### Project 2: Security Group Module (`terraform-aws-sg/`)

#### [`main.tf` in `terraform-aws-sg/`](../../terraform-aws-sg/main.tf)

```hcl
1: resource "aws_security_group" "main" {
2:   name        = "${var.project}-${var.environment}-${var.sg_name}"
3:   description = "Allow TLS inbound traffic for ${var.project} in ${var.environment} for component ${var.sg_name}"
4:   vpc_id      = var.vpc_id
5: 
6:   egress {
7:     from_port   = 0
8:     to_port     = 0
9:     protocol    = "-1"
10:     cidr_blocks = ["0.0.0.0/0"]
11:   }
12: }
```
- Implements singular naming (`"main"`).
- Dynamic name interpolation: `name = "${var.project}-${var.environment}-${var.sg_name}"`.

---

### Project 3: VPC Module Consumer (`vpc-module-test/`)

#### [`vpc.tf` in `vpc-module-test/`](../../vpc-module-test/vpc.tf)

```hcl
1: module "vpc" {
2:   source              = "../terraform-aws-vpc"
3:   project             = var.project
4:   environment         = var.environment
5:   is_peering_required = true
6: }
```
- Calls the reusable VPC module via a relative path.
- In just 6 lines of code, provisions an entire enterprise-grade multi-tier VPC!

---

## 4. Step-by-Step Hands-on Execution Walkthrough

### 1. Testing the Reusable VPC Module
```bash
# 1. Navigate to the VPC test module consumer directory
cd /Users/sriramcharankolla/Desktop/DevOps/vpc-module-test

# 2. Initialize Terraform (Links the local VPC child module)
terraform init

# 3. Format and validate
terraform fmt
terraform validate

# 4. Generate execution plan
terraform plan
```

---

### 2. Inspecting Subnets and Gateways via AWS CLI
```bash
# Query the created VPC details
aws ec2 describe-vpcs \
  --filters "Name=tag:Project,Values=roboshop" \
  --query "Vpcs[*].{VpcId:VpcId,CidrBlock:CidrBlock,State:State}" \
  --output table

# Query created Subnets and their Available IP counts
aws ec2 describe-subnets \
  --filters "Name=tag:Project,Values=roboshop" \
  --query "Subnets[*].{SubnetId:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone,AvailableIPs:AvailableIpAddressCount}" \
  --output table
```
*Expected AvailableIPs on a freshly provisioned `/24` subnet: `251` (proving the 5 AWS reserved IPs).*

---

### 3. Clean Up (Teardown)
```bash
terraform destroy -auto-approve
```

---

## 5. Commands & CLI Flags Reference Table

| Command / Tool | Syntax Example | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `aws ec2 describe-vpcs` | `aws ec2 describe-vpcs --vpc-ids vpc-01234` | Queries details, CIDR block, and state of a VPC. |
| `aws ec2 describe-subnets` | `aws ec2 describe-subnets --filters "Name=vpc-id,Values=..."` | Audits available IP addresses and AZ mapping across subnets. |
| `terraform validate` | `terraform validate` | Checks HCL syntax, block arguments, and custom variable validations offline. |
| `validation` block | Inside `variable "environment"` | Enforces strict string constraints (`dev`, `qa`, `uat`, `prod`). |

---

## 6. Official Documentation & References
- **AWS VPC Fundamentals Guide**: [docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
- **Subnet Sizing & AWS Reserved IPs**: [docs.aws.amazon.com/vpc/latest/userguide/subnet-sizing-ipv4.html](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-sizing-ipv4.html)
- **HashiCorp Terraform Standard Module Structure**: [developer.hashicorp.com/terraform/language/modules/develop/structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure)
- **Custom Variable Validation**: [developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules](https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules)

---

## 7. High-Yield Interview Questions & Answers

### Q1: Why does AWS reserve 5 IP addresses in every subnet, and what are their specific roles?
**Answer**:
In every subnet created in an AWS VPC, 5 IP addresses are automatically reserved and cannot be assigned to EC2 instances:
1. **First IP (`.0`)**: Network Address.
2. **Second IP (`.1`)**: VPC Local Router (default gateway).
3. **Third IP (`.2`)**: Amazon DNS server (AmazonProvidedDNS / Route 53 Resolver).
4. **Fourth IP (`.3`)**: Reserved by AWS for future functionality.
5. **Last IP (`.255` in a `/24`)**: Network Broadcast address (AWS does not support broadcast, but reserves the address).
Therefore, a `/24` subnet ($256$ total addresses) provides exactly **251 usable IP addresses**.

---

### Q2: What is CIDR notation and what are the minimum and maximum subnet sizes supported by AWS?
**Answer**:
CIDR (Classless Inter-Domain Routing) defines an IP block using prefix notation (`IP/Prefix`), where the prefix dictates the number of bits allocated to the Network portion, leaving the rest for Hosts.
- **Maximum VPC size**: `/16` (providing $2^{16} = 65,536$ addresses).
- **Minimum Subnet size**: `/28` (providing $2^4 = 16$ addresses, of which only **11 are usable** after subtracting the 5 AWS reserved IPs).

---

### Q3: What is the HashiCorp recommended naming convention for resources inside a reusable child module?
**Answer**:
If a child module provisions only one primary resource of a given type, it should be named `"this"` or `"main"` (e.g., `resource "aws_vpc" "main"` or `resource "aws_instance" "this"`).
This standard ensures that callers accessing outputs or referencing attributes write clean expressions like `module.vpc.id` or `aws_instance.this.private_ip`, avoiding redundant stuttering names like `aws_vpc.vpc.id`.

---

### Q4: What is the architectural difference between a Public Subnet and a Private Subnet in AWS?
**Answer**:
- **Public Subnet**: Its Route Table contains an explicit route to an **Internet Gateway (IGW)** (`0.0.0.0/0 -> igw-xxxx`). Instances launched here have `map_public_ip_on_launch = true` and are reachable directly from the internet.
- **Private Subnet**: Its Route Table has **no direct route to an Internet Gateway**. Outbound internet communication (e.g., software downloads) is directed through a **NAT Gateway** located in a public subnet, keeping the instances protected from inbound internet traffic.

---

## 8. Production Mistakes & Troubleshooting Guide

1. **Sizing Subnets Too Small (`/28` Exhaustion)**:
   - *Problem*: Creating a `/28` subnet provides only 11 usable IPs. Adding load balancers, RDS instances, and autoscaling quickly exhausts available IPs, causing instance launch failures.
   - *Best Practice*: Use `/24` (251 usable IPs) as the standard baseline size for production subnets.
2. **Missing `enable_dns_hostnames = true` on Custom VPCs**:
   - *Problem*: Instances launched in custom VPCs receive Private IPs but no private DNS names (`ip-10-0-1-x.ec2.internal`).
   - *Fix*: Always set `enable_dns_hostnames = true` in the `aws_vpc` resource.
3. **Overlapping CIDR Blocks**:
   - *Problem*: Creating two VPCs with the same CIDR (`10.0.0.0/16`) prevents VPC Peering, Transit Gateway attachments, or VPN connections between them.
   - *Fix*: Plan non-overlapping IP schemas across enterprise environments (e.g., VPC-A: `10.0.0.0/16`, VPC-B: `10.1.0.0/16`).

---

## 9. Session Metadata & Timestamps

- **Terraform Naming Standards & Best Practices**: `00:00 - 25:00`
- **Module Architecture Recap**: `25:00 - 45:00`
- **Virtual Private Cloud (VPC) Deep-Dive**: `45:04`
- **IPv4 Binary System & Math**: `45:04 - 01:10:00`
- **CIDR & Subnetting Calculations**: `01:10:00 - 01:25:00`
- **Session Q&A**: `01:27:21`

### Doubts & AI Clarification Link
- [Session 35 AI Clarification Chat](https://chatgpt.com/share/69a067dc-dd60-8007-94a0-5cab91cbe883)
