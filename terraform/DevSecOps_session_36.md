# Friday, 27 February 2026
# Session 36 - AWS VPC Deep Dive: Manual vs Automated Creation, 3-Tier Subnets, Route Tables, NAT Gateway & Elastic IP
## Comprehensive Class Notes, Enterprise VPC Module Code Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [Multi-AZ Network Architecture](#multi-az-network-architecture)
   - [3-Tier Subnet Topology (Public, Private, Database)](#3-tier-subnet-topology-public-private-database)
   - [Network Traffic Flows: Ingress vs Egress](#network-traffic-flows-ingress-vs-egress)
   - [NAT Gateway & Elastic IP Deep-Dive](#nat-gateway--elastic-ip-deep-dive)
   - [AWS Console Manual Workflow vs Terraform Automation](#aws-console-manual-workflow-vs-terraform-automation)
   - [AWS Cost Considerations (Free vs Paid VPC Components)](#aws-cost-considerations-free-vs-paid-vpc-components)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct File Links](#directory-structure--direct-file-links)
   - [Architectural Diagram: 3-Tier VPC Packet Flow](#architectural-diagram-3-tier-vpc-packet-flow)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [VPC & Internet Gateway (`main.tf`: Lines 1-13)](#vpc--internet-gateway-maintf-lines-1-13)
   - [3-Tier Subnets with Dynamic AZ Mapping (`main.tf`: Lines 16-65)](#3-tier-subnets-with-dynamic-az-mapping-maintf-lines-16-65)
   - [Route Tables Creation (`main.tf`: Lines 67-104)](#route-tables-creation-maintf-lines-67-104)
   - [Elastic IP & Public NAT Gateway (`main.tf`: Lines 106-139)](#elastic-ip--public-nat-gateway-maintf-lines-106-139)
   - [Private & Database Routing via NAT Gateway (`main.tf`: Lines 141-152)](#private--database-routing-via-nat-gateway-maintf-lines-141-152)
   - [Subnet Route Table Associations (`main.tf`: Lines 154-170)](#subnet-route-table-associations-maintf-lines-154-170)
   - [Module Consumer Implementation (`vpc-module-test/vpc.tf`)](#module-consumer-implementation-vpc-module-testvpctf)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Testing the Enterprise VPC Module via Test Harness](#1-testing-the-enterprise-vpc-module-via-test-harness)
   - [2. Validating NAT Gateway & Route Tables with AWS CLI](#2-validating-nat-gateway--route-tables-with-aws-cli)
   - [3. Clean Up (Teardown)](#3-clean-up-teardown)
5. [Commands & CLI Flags Reference Table](#5-commands--cli-flags-reference-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### Multi-AZ Network Architecture
In enterprise production design, high availability is achieved by spanning resources across at least **two Availability Zones (AZs)** within a region:
- **Region**: `us-east-1` (N. Virginia)
- **Selected AZs**: `us-east-1a`, `us-east-1b`
- **Rule**: Exactly 1 subnet per tier per AZ to ensure failure isolation if an entire AWS data center loses power.

---

### 3-Tier Subnet Topology (Public, Private, Database)
The network CIDR `10.0.0.0/16` (65,536 total IPs) is partitioned into three distinct functional tiers:

```
+---------------------------------------------------------------------------------------------------+
| TIER             | AZ: us-east-1a     | AZ: us-east-1b     | PURPOSE / HOSTED WORKLOADS           |
| ---------------- | ------------------ | ------------------ | -----------------------------------  |
| 1. Public        | 10.0.1.0/24        | 10.0.2.0/24        | Application Load Balancers, Bastion, |
|                  | (251 usable IPs)   | (251 usable IPs)   | Public NAT Gateway                   |
| 2. Private       | 10.0.11.0/24       | 10.0.12.0/24       | Backend Microservices (Catalogue,    |
|                  | (251 usable IPs)   | (251 usable IPs)   | Cart, User, Shipping, Payment)       |
| 3. Database      | 10.0.21.0/24       | 10.0.22.0/24       | Data Stores (MongoDB, MySQL, Redis,  |
|                  | (251 usable IPs)   | (251 usable IPs)   | RabbitMQ, AWS RDS Instances)         |
+---------------------------------------------------------------------------------------------------+
```

---

### Network Traffic Flows: Ingress vs Egress
- **Ingress (Inbound Traffic)**:
  - Data or connection requests entering the VPC from the outside world.
  - *Example*: External internet users browsing `https://roboshop.aitechapp.fun` sending HTTPS packets on port 443 into the public Application Load Balancer.
- **Egress (Outbound Traffic)**:
  - Data or packets originating inside your VPC/servers going outward to the internet.
  - *Example*: Running `sudo dnf install mysql-server -y` inside a private EC2 instance. The server initiates an outbound TCP request on port 80/443 to Red Hat repository mirrors on the public internet to download RPM packages.

---

### NAT Gateway & Elastic IP Deep-Dive
- **The Private Subnet Dilemma**:
  - Private EC2 instances (databases, backend microservices) do **not** have public IP addresses and cannot communicate directly with the Internet Gateway (IGW).
  - However, they must download OS security patches, install software packages, and communicate with external payment gateways or third-party APIs.
- **How NAT (Network Address Translation) Gateway Works**:
  - A managed AWS service that translates private IP addresses (`10.0.11.x`) into its own single public Elastic IP before forwarding packets to the Internet Gateway.
  - **Critical Placement Rule**: The NAT Gateway **MUST be created inside a Public Subnet** (`roboshop-public-us-east-1a`) because it needs direct access to the Internet Gateway!
  - **One-Way Connectivity**: The NAT Gateway allows private instances to connect *out* to the internet, but **strictly blocks any incoming connection** initiated from the internet.
- **Elastic IP (EIP) Requirement**:
  - An Elastic IP is a static, public IPv4 address allocated to your AWS account.
  - Allocating an Elastic IP is **mandatory** when provisioning a public NAT Gateway.

---

### AWS Console Manual Workflow vs Terraform Automation
Understanding the manual AWS Management Console steps illuminates why Infrastructure as Code is so powerful:

#### Manual AWS Console Steps (As Covered in Class):
1. **Create VPC**: Search for VPC $\rightarrow$ Click *Create VPC* $\rightarrow$ Enter Name `roboshop-dev`, IPv4 CIDR `10.0.0.0/16` $\rightarrow$ Create.
2. **Create & Attach Internet Gateway**: Go to *Internet Gateways* $\rightarrow$ *Create internet gateway* $\rightarrow$ Name `roboshop-dev-igw` $\rightarrow$ Click *Attach to VPC* $\rightarrow$ Select `roboshop-dev`.
3. **Create Subnets**: Go to *Subnets* $\rightarrow$ Create 6 subnets across AZs `us-east-1a` and `us-east-1b` with respective CIDRs (`10.0.1.0/24`, `10.0.2.0/24`, `10.0.11.0/24`...).
4. **Allocate Elastic IP**: Go to *Elastic IPs* $\rightarrow$ Click *Allocate Elastic IP address* in `us-east-1`.
5. **Create NAT Gateway**: Go to *NAT Gateways* $\rightarrow$ *Create NAT gateway* $\rightarrow$ Select public subnet `roboshop-public-us-east-1a` $\rightarrow$ Connectivity: *Public* $\rightarrow$ Choose allocated Elastic IP ID.
6. **Create Route Tables & Associations**:
   - Create Public Route Table $\rightarrow$ Associate public subnets $\rightarrow$ Add Route: `0.0.0.0/0` $\rightarrow$ Target: **Internet Gateway (IGW)**.
   - Create Private Route Table $\rightarrow$ Associate private subnets $\rightarrow$ Add Route: `0.0.0.0/0` $\rightarrow$ Target: **NAT Gateway**.
   - Create Database Route Table $\rightarrow$ Associate database subnets $\rightarrow$ Add Route: `0.0.0.0/0` $\rightarrow$ Target: **NAT Gateway** (or leave strictly local).

*In Terraform, all of the above 6 complex, error-prone manual console steps are automated in a single reusable module call with 6 lines of HCL!*

---

### AWS Cost Considerations (Free vs Paid VPC Components)
- **100% FREE in AWS**:
  - Creating a VPC.
  - Creating Subnets.
  - Creating and attaching an Internet Gateway (IGW).
  - Creating Route Tables and Subnet Associations.
  - Security Groups and Network ACLs.
- **PAID Components (Billed Hourly + Data Processing)**:
  - **NAT Gateway**: Incurs an hourly charge (~$0.045/hr $\approx$ $32/month per NAT Gateway) + data processing fee per GB transferred.
  - **Elastic IP**: Free as long as it is attached to an active running resource (like a NAT Gateway or running EC2 instance). **AWS charges for unattached or idle Elastic IPs** to prevent IPv4 hoarding!

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct File Links
Click any link below to open the corresponding project file directly in your IDE:

#### Project 1: Reusable Enterprise VPC Child Module
- **Directory**: [terraform-aws-vpc/](../../terraform-aws-vpc)
- **VPC Infrastructure Code**: [terraform-aws-vpc/main.tf](../../terraform-aws-vpc/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terrafrom-aws-vpc/blob/main/main.tf)
- **Variables & Validations**: [terraform-aws-vpc/variables.tf](../../terraform-aws-vpc/variables.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terrafrom-aws-vpc/blob/main/variables.tf)
- **Locals & Computed Tags**: [terraform-aws-vpc/locals.tf](../../terraform-aws-vpc/locals.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terrafrom-aws-vpc/blob/main/locals.tf)
- **AZ Query Data Source**: [terraform-aws-vpc/data.tf](../../terraform-aws-vpc/data.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terrafrom-aws-vpc/blob/main/data.tf)
- **Outputs**: [terraform-aws-vpc/outputs.tf](../../terraform-aws-vpc/outputs.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terrafrom-aws-vpc/blob/main/outputs.tf)

#### Project 2: VPC Module Test Consumer (Root Module)
- **Directory**: [vpc-module-test/](../../vpc-module-test)
- **Root Invocation**: [vpc-module-test/vpc.tf](../../vpc-module-test/vpc.tf)
- **Variables**: [vpc-module-test/variables.tf](../../vpc-module-test/variables.tf)
- **Provider**: [vpc-module-test/provider.tf](../../vpc-module-test/provider.tf)
- **Outputs**: [vpc-module-test/outputs.tf](../../vpc-module-test/outputs.tf)

---

### Architectural Diagram: 3-Tier VPC Packet Flow

```
                                  PUBLIC INTERNET
                                         │
                                         │ Inbound & Outbound Internet Traffic
                                         ▼
                      +-------------------------------------+
                      |     Internet Gateway (IGW: main)    |
                      +-------------------------------------+
                                         │
        ┌────────────────────────────────┴────────────────────────────────┐
        │ Route: 0.0.0.0/0 -> IGW                                         │ Route: 0.0.0.0/0 -> IGW
        ▼                                                                 ▼
+─────────────────────────────────+             +─────────────────────────────────+
|   PUBLIC SUBNET (us-east-1a)    |             |   PUBLIC SUBNET (us-east-1b)    |
|   10.0.1.0/24                   |             |   10.0.2.0/24                   |
|                                 |             |                                 |
|   [NAT GATEWAY (with EIP)]      |             |   [Public ALB Node B]           |
|   (Translates Private IPs)      |             |                                 |
+─────────────────────────────────+             +─────────────────────────────────+
        ▲                                                                 ▲
        │ Egress Outbound Only                                            │ Egress Outbound Only
        │ (0.0.0.0/0 -> NAT Gateway)                                      │ (0.0.0.0/0 -> NAT Gateway)
        │                                                                 │
+─────────────────────────────────+             +─────────────────────────────────+
|   PRIVATE SUBNET (us-east-1a)   |             |   PRIVATE SUBNET (us-east-1b)   |
|   10.0.11.0/24                  |             |   10.0.12.0/24                  |
|   [Catalogue, Cart, User...]    |             |   [Shipping, Payment...]        |
+─────────────────────────────────+             +─────────────────────────────────+
        │                                                                 │
        ▼ Internal Communication Only                                     ▼ Internal Communication Only
+─────────────────────────────────+             +─────────────────────────────────+
|   DATABASE SUBNET (us-east-1a)  |             |   DATABASE SUBNET (us-east-1b)  |
|   10.0.21.0/24                  |             |   10.0.22.0/24                  |
|   [MongoDB, MySQL Primary]      |             |   [Redis, MySQL Replica]        |
+─────────────────────────────────+             +─────────────────────────────────+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### VPC & Internet Gateway ([`main.tf`: Lines 1-13](../../terraform-aws-vpc/main.tf#L1-L13))

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
```
- **Line 1 (`resource "aws_vpc" "main"`)**: Provisions the VPC. Named `"main"` following HashiCorp single-resource convention.
- **Line 4 (`enable_dns_hostnames = true`)**: Required so that instances launched inside the VPC receive public and private DNS records (e.g., `ec2-xx-xx.compute-1.amazonaws.com`).
- **Line 10 (`vpc_id = aws_vpc.main.id`)**: Explicitly attaches the Internet Gateway to the VPC via implicit DAG dependency.

---

### 3-Tier Subnets with Dynamic AZ Mapping ([`main.tf`: Lines 16-65](../../terraform-aws-vpc/main.tf#L16-L65))

```hcl
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
34: resource "aws_subnet" "private" {
35:   count             = length(var.private_subnet_cidrs)
36:   vpc_id            = aws_vpc.main.id
37:   cidr_block        = var.private_subnet_cidrs[count.index]
38:   availability_zone = local.az_names[count.index]
39:   # map_public_ip_on_launch defaults to false
40:   tags = merge(...)
41: }
51: resource "aws_subnet" "database" {
52:   count             = length(var.database_subnet_cidrs)
53:   vpc_id            = aws_vpc.main.id
54:   cidr_block        = var.database_subnet_cidrs[count.index]
55:   availability_zone = local.az_names[count.index]
56:   tags = merge(...)
57: }
```
- **Line 20 (`availability_zone = local.az_names[count.index]`)**:
  - Dynamically assigns `us-east-1a` to index `0`, and `us-east-1b` to index `1`.
  - Ensures subnets are evenly balanced across physical AWS availability zones.
- **Line 21 (`map_public_ip_on_launch = true`)**:
  - Activates automatic public IPv4 address assignment for instances in public subnets.
  - Notice this is **omitted** from private and database subnets, ensuring complete isolation from public addressing.

---

### Route Tables Creation ([`main.tf`: Lines 67-104](../../terraform-aws-vpc/main.tf#L67-L104))

```hcl
67: resource "aws_route_table" "public" {
68:   vpc_id = aws_vpc.main.id
69:   tags   = merge(local.common_tags, { Name = "${var.project}-${var.environment}-public" }, var.public_route_table_tags)
70: }
80: resource "aws_route_table" "private" {
81:   vpc_id = aws_vpc.main.id
82:   tags   = merge(local.common_tags, { Name = "${var.project}-${var.environment}-private" }, var.private_route_table_tags)
83: }
93: resource "aws_route_table" "database" {
94:   vpc_id = aws_vpc.main.id
95:   tags   = merge(local.common_tags, { Name = "${var.project}-${var.environment}-database" }, var.database_route_table_tags)
96: }
```
- Provisions three dedicated route tables to keep routing rules completely separated by network tier.

---

### Elastic IP & Public NAT Gateway ([`main.tf`: Lines 106-139](../../terraform-aws-vpc/main.tf#L106-L139))

```hcl
106: resource "aws_route" "public" {
107:   route_table_id         = aws_route_table.public.id
108:   destination_cidr_block = "0.0.0.0/0"
109:   gateway_id             = aws_internet_gateway.main.id
110: }
112: resource "aws_eip" "nat" {
113:   domain = "vpc"
114:   tags   = merge(local.common_tags, { Name = "${var.project}-${var.environment}-nat" }, var.eip_tags)
115: }
124: resource "aws_nat_gateway" "main" {
125:   allocation_id = aws_eip.nat.id
126:   subnet_id     = aws_subnet.public[0].id # Placed in us-east-1a Public Subnet
127:   tags          = merge(local.common_tags, { Name = "${var.project}-${var.environment}" }, var.nat_gateway_tags)
138:   depends_on    = [aws_internet_gateway.main]
139: }
```
- **Lines 106-110 (`aws_route.public`)**: Routes all non-local traffic (`0.0.0.0/0`) from public subnets out to the **Internet Gateway**.
- **Lines 112-115 (`aws_eip.nat`)**: Allocates a static public IP in the VPC domain.
- **Line 126 (`subnet_id = aws_subnet.public[0].id`)**: Critical architecture rule: NAT Gateway is physically hosted in the **first Public Subnet**.
- **Line 138 (`depends_on = [aws_internet_gateway.main]`)**: Explicit dependency ensuring the Internet Gateway is fully initialized before AWS attempts to bind the NAT Gateway to the network.

---

### Private & Database Routing via NAT Gateway ([`main.tf`: Lines 141-152](../../terraform-aws-vpc/main.tf#L141-L152))

```hcl
141: resource "aws_route" "private" {
142:   route_table_id         = aws_route_table.private.id
143:   destination_cidr_block = "0.0.0.0/0"
144:   nat_gateway_id         = aws_nat_gateway.main.id
145: }
147: resource "aws_route" "database" {
148:   route_table_id         = aws_route_table.database.id
149:   destination_cidr_block = "0.0.0.0/0"
150:   nat_gateway_id         = aws_nat_gateway.main.id
151: }
```
- **Lines 141-145**: Directs all outbound internet requests (`0.0.0.0/0`) from private subnets to the **NAT Gateway**, enabling secure package downloads (`dnf install`) without exposing servers to incoming traffic.

---

### Subnet Route Table Associations ([`main.tf`: Lines 154-170](../../terraform-aws-vpc/main.tf#L154-L170))

```hcl
154: resource "aws_route_table_association" "public" {
155:   count          = length(var.public_subnet_cidrs)
156:   subnet_id      = aws_subnet.public[count.index].id
157:   route_table_id = aws_route_table.public.id
158: }
160: resource "aws_route_table_association" "private" {
161:   count          = length(var.private_subnet_cidrs)
162:   subnet_id      = aws_subnet.private[count.index].id
163:   route_table_id = aws_route_table.private.id
164: }
166: resource "aws_route_table_association" "database" {
167:   count          = length(var.database_subnet_cidrs)
168:   subnet_id      = aws_subnet.database[count.index].id
169:   route_table_id = aws_route_table.database.id
170: }
```
- Binds each of the 6 subnets to its corresponding tier route table.

---

### Module Consumer Implementation ([`vpc-module-test/vpc.tf`](../../vpc-module-test/vpc.tf))

```hcl
1: module "vpc" {
2:   source              = "../terraform-aws-vpc"
3:   project             = var.project
4:   environment         = var.environment
5:   is_peering_required = true
6: }
```
- The consumer module invokes the entire architecture in 6 lines of code.

---

## 4. Step-by-Step Hands-on Execution Walkthrough

All commands below are directly copyable:

### 1. Testing the Enterprise VPC Module via Test Harness
```bash
# Navigate to the test consumer directory
cd /Users/sriramcharankolla/Desktop/DevOps/vpc-module-test

# Initialize (downloads and links local child module)
terraform init

# Code style and syntax validation
terraform fmt
terraform validate

# Plan infrastructure changes (Inspect all 22 resources to add)
terraform plan

# Deploy infrastructure to AWS
terraform apply -auto-approve
```

---

### 2. Validating NAT Gateway & Route Tables with AWS CLI
```bash
# 1. Verify VPC state and DNS settings
aws ec2 describe-vpcs \
  --filters "Name=tag:Project,Values=roboshop" \
  --query "Vpcs[*].{VpcId:VpcId,CidrBlock:CidrBlock,State:State}" \
  --output table

# 2. Verify all 6 Subnets and their available IP counts (251 IPs each)
aws ec2 describe-subnets \
  --filters "Name=tag:Project,Values=roboshop" \
  --query "Subnets[*].{Name:Tags[?Key=='Name'].Value|[0],CIDR:CidrBlock,AZ:AvailabilityZone,AvailableIPs:AvailableIpAddressCount}" \
  --output table

# 3. Verify NAT Gateway is Active and Bound to an Elastic IP
aws ec2 describe-nat-gateways \
  --filter "Name=tag:Project,Values=roboshop" \
  --query "NatGateways[*].{NatGatewayId:NatGatewayId,State:State,SubnetId:SubnetId,PublicIP:NatGatewayAddresses[0].PublicIp}" \
  --output table
```

---

### 3. Clean Up (Teardown)
> [!CAUTION]
> Always destroy the test VPC after validation to avoid accumulating hourly NAT Gateway charges!

```bash
terraform destroy -auto-approve
```

---

## 5. Commands & CLI Flags Reference Table

| Command / Tool | Syntax Example | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `terraform get -update` | `terraform get -update` | Synchronizes and updates child modules referenced in the root configuration. |
| `aws ec2 describe-nat-gateways` | `aws ec2 describe-nat-gateways` | Audits state, public IP, and hosting subnet of all NAT Gateways. |
| `aws ec2 describe-route-tables` | `aws ec2 describe-route-tables --filters "Name=vpc-id,Values=..."` | Audits routing destinations (`0.0.0.0/0 -> igw` vs `0.0.0.0/0 -> nat`). |
| `depends_on` meta-argument | `depends_on = [aws_internet_gateway.main]` | Enforces explicit creation ordering when implicit references are absent. |

---

## 6. Official Documentation & References
- **AWS VPC Architecture Guide**: [docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
- **AWS NAT Gateways Documentation**: [docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- **AWS Elastic IP Addresses (EIP)**: [docs.aws.amazon.com/vpc/latest/userguide/vpc-eips.html](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-eips.html)
- **Terraform `aws_route_table` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table)
- **Terraform `aws_nat_gateway` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/nat_gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/nat_gateway)

---

## 7. High-Yield Interview Questions & Answers

### Q1: Why must an AWS NAT Gateway always be deployed in a Public Subnet?
**Answer**:
A NAT Gateway functions as a proxy that translates internal private IP addresses to its own public Elastic IP address so traffic can reach external servers.
Because the NAT Gateway itself must communicate directly with the **Internet Gateway (IGW)**, it **must reside in a subnet that has a route to the IGW (`0.0.0.0/0 -> igw-xxxx`)**, which by definition is a **Public Subnet**.
If you deploy a NAT Gateway in a private subnet, it cannot communicate with the internet, creating a black-hole routing loop.

---

### Q2: What is the architectural difference between an Internet Gateway (IGW) and a NAT Gateway?
**Answer**:
- **Internet Gateway (IGW)**:
  - Horizontally scaled, redundant, highly available VPC component that provides **two-way routing** (both Ingress from internet and Egress to internet).
  - Designed for instances with Public IPs in Public Subnets.
  - **Free of charge**.
- **NAT Gateway**:
  - Managed network translation device providing **one-way egress routing** (outbound internet access only).
  - Designed for private instances with no public IPs. Strictly blocks inbound internet connections.
  - **Incurs hourly and data processing charges**.

---

### Q3: Why is `depends_on = [aws_internet_gateway.main]` recommended inside the `aws_nat_gateway` resource?
**Answer**:
The NAT Gateway uses an Elastic IP to communicate with external endpoints through the VPC's Internet Gateway.
However, in HCL, the `aws_nat_gateway` resource only directly references `aws_eip.nat.id` and `aws_subnet.public[0].id`—it contains no direct attribute reference to `aws_internet_gateway.main.id`.
Without `depends_on`, Terraform's DAG engine might attempt to create the NAT Gateway before the Internet Gateway attachment finishes on AWS, causing provisioning timeouts or routing failures.

---

### Q4: How do instances in a Private Subnet resolve domain names like `google.com` or `repo.mysql.com`?
**Answer**:
Every AWS subnet reserves the `.2` IP address (e.g., `10.0.11.2` in `10.0.11.0/24`) for the **Amazon Route 53 Resolver** (formerly AmazonProvidedDNS).
Private EC2 instances send DNS queries directly to `10.0.11.2` via internal VPC routing. Once resolved to an external IP, the instance opens a connection to that IP through the NAT Gateway.

---

## 8. Production Mistakes & Troubleshooting Guide

1. **Deploying NAT Gateway in a Private Subnet (Black-Hole Routing)**:
   - *Symptom*: Private EC2 instances cannot install packages (`curl` or `dnf` hangs indefinitely).
   - *Cause*: NAT Gateway was provisioned inside `aws_subnet.private[0].id` instead of `aws_subnet.public[0].id`.
   - *Fix*: Ensure `subnet_id = aws_subnet.public[0].id` in `aws_nat_gateway`.
2. **Idle Elastic IP Charges**:
   - *Problem*: Destroying EC2 instances or NAT Gateways without releasing the associated Elastic IP causes AWS to bill an idle IP surcharge.
   - *Fix*: Always track EIPs in Terraform state so `terraform destroy` purges both the NAT Gateway and the Elastic IP simultaneously.
3. **Missing Subnet Route Table Associations**:
   - *Symptom*: Instances in custom subnets default to the VPC "Main Route Table", bypassing custom IGW/NAT routes.
   - *Fix*: Always create explicit `aws_route_table_association` resources for all subnets.

---

## 9. Session Metadata & Timestamps

- **Multi-AZ Architecture & Ingress/Egress**: `00:00 - 25:00`
- **NAT Gateway & Elastic IP Architecture**: `25:00 - 45:00`
- **AWS Console Step-by-Step Manual Guide**: `45:00 - 01:10:00`
- **Terraform VPC Module Implementation**: `01:10:00 - 01:25:00`
- **Session Q&A**: `01:31:40`

### Doubts & AI Clarification Link
- [Session 36 AI Clarification Chat](https://chat.z.ai/c/f3862eee-15cb-49ca-84e8-41702b207049)