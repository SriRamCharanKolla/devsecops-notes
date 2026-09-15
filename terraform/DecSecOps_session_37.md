# Monday, 2 March 2026
# Session 37 - VPC Module Completion, VPC Peering Architecture, Bidirectional Routing & Project vs Application Infrastructure
## Comprehensive Class Notes, Enterprise VPC Peering Code Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [VPC Architecture Recap](#vpc-architecture-recap)
   - [The VPC Isolation Boundary](#the-vpc-isolation-boundary)
   - [VPC Peering Deep-Dive](#vpc-peering-deep-dive)
   - [The Non-Overlapping CIDR Rule (Hard Constraint)](#the-non-overlapping-cidr-rule-hard-constraint)
   - [Peering Roles: Requester vs Accepter](#peering-roles-requester-vs-accepter)
   - [Transitive Peering Limitation](#transitive-peering-limitation)
   - [Project Infrastructure vs Application Infrastructure](#project-infrastructure-vs-application-infrastructure)
   - [AWS Console Step-by-Step Manual Peering Guide](#aws-console-step-by-step-manual-peering-guide)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct File Links](#directory-structure--direct-file-links)
   - [Architectural Diagram: Bidirectional VPC Peering](#architectural-diagram-bidirectional-vpc-peering)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [VPC Peering Connection (`peering.tf`: Lines 1-27)](#vpc-peering-connection-peeringtf-lines-1-27)
   - [Custom VPC Routes to Default VPC (`peering.tf`: Lines 29-49)](#custom-vpc-routes-to-default-vpc-peeringtf-lines-29-49)
   - [Default VPC Return Route (`peering.tf`: Lines 50-55)](#default-vpc-return-route-peeringtf-lines-50-55)
   - [Data Sources for Default VPC & Route Table (`data.tf`)](#data-sources-for-default-vpc--route-table-datatf)
   - [Module Consumer Implementation (`vpc-module-test/vpc.tf`)](#module-consumer-implementation-vpc-module-testvpctf)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Testing VPC Peering via Test Harness](#1-testing-vpc-peering-via-test-harness)
   - [2. Verifying Peering & Routing with AWS CLI](#2-verifying-peering--routing-with-aws-cli)
   - [3. Clean Up (Teardown)](#3-clean-up-teardown)
5. [Commands & CLI Flags Reference Table](#5-commands--cli-flags-reference-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### VPC Architecture Recap
The foundational components of an enterprise AWS network:
1. **VPC**: Isolated virtual data center defined by a primary CIDR (e.g., `10.0.0.0/16`).
2. **Internet Gateway (IGW)**: Scaled two-way gateway attached to the VPC for internet ingress and egress.
3. **Subnets**: Public (`10.0.1.0/24`, `10.0.2.0/24`), Private (`10.0.11.0/24`, `10.0.12.0/24`), and Database (`10.0.21.0/24`, `10.0.22.0/24`).
4. **Route Tables**: Separate routing rules for Public, Private, and Database subnets.
5. **Associations**: Binding subnets to their respective route tables.
6. **Elastic IP (EIP)**: Static public IPv4 address allocated for the NAT Gateway.
7. **NAT Gateway**: Deployed in a Public Subnet to provide secure, one-way outbound (egress) internet access for private workloads.
   - **Public Route Table**: `0.0.0.0/0` $\rightarrow$ `Internet Gateway (IGW)`
   - **Private Route Table**: `0.0.0.0/0` $\rightarrow$ `NAT Gateway`

---

### The VPC Isolation Boundary
- By default, **two VPCs in AWS cannot communicate with each other**, even if they are created within the same AWS account and same AWS Region!
- A backend server with private IP `10.0.11.34` inside Custom VPC (`10.0.0.0/16`) cannot ping or route packets to a server with private IP `172.31.1.3` inside the Default VPC (`172.31.0.0/16`).
- This isolation is a fundamental AWS security boundary.

---

### VPC Peering Deep-Dive
- **What is VPC Peering?**
  - A dedicated point-to-point networking connection between two VPCs that routes traffic privately using internal IPv4 or IPv6 addresses.
  - Traffic never traverses the public internet; packets remain entirely within the high-speed AWS global network backbone, ensuring high throughput, low latency, and zero exposure to external snoopers.
- **The 4 Supported Peering Topologies**:
  1. **Same Region, Same AWS Account** (e.g., Roboshop Dev VPC $\leftrightarrow$ Default AWS VPC).
  2. **Different Region, Same AWS Account** (Inter-Region Peering: `us-east-1` $\leftrightarrow$ `ap-south-1`).
  3. **Different AWS Account, Same Region** (Cross-Account Peering: Corp Shared Services Account $\leftrightarrow$ App Account).
  4. **Different AWS Account, Different Region** (Cross-Account, Cross-Region Peering).

---

### The Non-Overlapping CIDR Rule (Hard Constraint)
> [!CRITICAL]
> **VPC Peering CANNOT be established if the CIDR blocks of the two VPCs overlap!**
> - **Valid Peering**: VPC-1 (`10.0.0.0/16`) $\leftrightarrow$ VPC-2 (`172.31.0.0/16`) $\rightarrow$ **Permitted** (No overlap).
> - **Invalid Peering**: VPC-1 (`10.0.0.0/16`) $\leftrightarrow$ VPC-2 (`10.0.0.0/16`) $\rightarrow$ **AWS API Rejection** (`InvalidCIDRBlock.Overlap`).
> If CIDRs overlap, routers cannot determine whether `10.0.1.5` resides locally or across the peering link!

---

### Peering Roles: Requester vs Accepter
1. **Requester VPC**:
   - The VPC initiating the peering connection request (e.g., `roboshop-dev`).
2. **Accepter VPC**:
   - The VPC receiving and approving the connection request (e.g., the `default` VPC).
3. **Auto-Accept Rule**:
   - When both VPCs reside within the **same AWS account**, Terraform can automatically accept the connection (`auto_accept = true`).
   - When peering across **different AWS accounts**, the peering connection remains in `pending-acceptance` state until an authorized IAM identity in the accepter account approves it.

---

### Transitive Peering Limitation
VPC Peering is strictly non-transitive:
```
[VPC A] <==== Peered ====> [VPC B] <==== Peered ====> [VPC C]
   │                                                     │
   └─────────────── CANNOT COMMUNICATE ──────────────────┘
```
- If VPC A is peered with VPC B, and VPC B is peered with VPC C:
  - Traffic **CANNOT** flow from VPC A to VPC C through VPC B!
  - You must either create a direct peering link between VPC A and VPC C, or deploy an **AWS Transit Gateway (TGW)** to act as a centralized hub-and-spoke router.

---

### Project Infrastructure vs Application Infrastructure
In enterprise DevOps, infrastructure is divided into two distinct lifecycle tiers:

```
+-----------------------------------------------------------------------------------+
| 1. PROJECT / FOUNDATION INFRASTRUCTURE (Managed by Platform / Core Network Team)  |
| --------------------------------------------------------------------------------- |
| - VPC, Subnets, Internet Gateways, NAT Gateways, Route Tables, VPC Peering        |
| - Lifecycle: One-time creation, long-lived, changes rarely (Quarterly / Yearly)  |
| - High blast radius: Destruction disrupts all hosted services                     |
+-----------------------------------------------------------------------------------+
                                         │
                                         ▼
+-----------------------------------------------------------------------------------+
| 2. APPLICATION INFRASTRUCTURE (Managed by DevOps / Product Feature Teams)         |
| --------------------------------------------------------------------------------- |
| - EC2 Instances, Auto Scaling Groups, ALB Target Groups, Helm Releases, Pods      |
| - Lifecycle: Continuous deployment, ephemeral, daily/hourly releases              |
| - Low blast radius: Easily recreated via CI/CD automation                         |
+-----------------------------------------------------------------------------------+
```

---

### AWS Console Step-by-Step Manual Peering Guide
To manually configure VPC Peering via the AWS Console:
1. Navigate to **VPC Console** $\rightarrow$ Click **Peering Connections** in the left navigation pane.
2. Click **Create peering connection**.
3. Name: `roboshop-dev-default`.
4. **VPC ID (Requester)**: Select your custom VPC (`roboshop-dev`).
5. **Select another VPC to peer with**:
   - Choose *My account* (or *Another account* if peering across organizations).
   - Region: Select *This region (`us-east-1`)*.
6. **VPC ID (Accepter)**: Select the Default VPC ID.
7. Click **Create peering connection**.
8. In the Peering connections list, select the newly created connection $\rightarrow$ Click **Actions** $\rightarrow$ Select **Accept request**.
9. **Configure Bidirectional Routes**:
   - Go to **Route Tables** $\rightarrow$ Select `roboshop-dev-public`, `roboshop-dev-private`, and `roboshop-dev-database` $\rightarrow$ Edit Routes $\rightarrow$ Add Route: Destination `172.31.0.0/16` $\rightarrow$ Target: Peering Connection (`pcx-xxxx`).
   - Select the **Default VPC Main Route Table** $\rightarrow$ Edit Routes $\rightarrow$ Add Route: Destination `10.0.0.0/16` $\rightarrow$ Target: Peering Connection (`pcx-xxxx`).

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct File Links
Click any link below to open the corresponding project file directly in your IDE:

#### Project 1: Reusable Enterprise VPC Module (with Peering)
- **Directory**: [terraform-aws-vpc/](../../terraform-aws-vpc)
- **Peering Logic & Routing**: [terraform-aws-vpc/peering.tf](../../terraform-aws-vpc/peering.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terrafrom-aws-vpc/blob/main/peering.tf)
- **Default VPC Data Queries**: [terraform-aws-vpc/data.tf](../../terraform-aws-vpc/data.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terrafrom-aws-vpc/blob/main/data.tf)
- **Variables & Toggles**: [terraform-aws-vpc/variables.tf](../../terraform-aws-vpc/variables.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terrafrom-aws-vpc/blob/main/variables.tf)
- **Core VPC Code**: [terraform-aws-vpc/main.tf](../../terraform-aws-vpc/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terrafrom-aws-vpc/blob/main/main.tf)

#### Project 2: VPC Module Test Consumer
- **Directory**: [vpc-module-test/](../../vpc-module-test)
- **Root Invocation with Peering**: [vpc-module-test/vpc.tf](../../vpc-module-test/vpc.tf)
- **Provider**: [vpc-module-test/provider.tf](../../vpc-module-test/provider.tf)
- **Variables**: [vpc-module-test/variables.tf](../../vpc-module-test/variables.tf)

---

### Architectural Diagram: Bidirectional VPC Peering

```
+---------------------------------------------------------------------------------------------------+
|                                      AWS ACCOUNT (us-east-1)                                      |
|                                                                                                   |
|   +---------------------------------------+             +-------------------------------------+   |
|   |      ROBOSHOP CUSTOM DEV VPC          |             |          AWS DEFAULT VPC            |   |
|   |         (10.0.0.0/16)                 |             |          (172.31.0.0/16)            |   |
|   |                                       |             |                                     |   |
|   |  - Public Subnet:   10.0.1.0/24       |             |  - Default Subnet: 172.31.1.0/24    |   |
|   |  - Private Subnet:  10.0.11.0/24      |             |  - Management Workstation / Bastion |   |
|   |  - Database Subnet: 10.0.21.0/24      |             |    (IP: 172.31.1.3)                 |   |
|   +---------------------------------------+             +-------------------------------------+   |
|                       │                                                     │                     |
|                       │                                                     │                     |
|                       ▼                                                     ▼                     |
|   +───────────────────────────────────────────────────────────────────────────────────────────+   |
|   |                     VPC PEERING CONNECTION: "roboshop-dev-default"                        |   |
|   |                              (aws_vpc_peering_connection.default)                         |   |
|   |                              pcx-0abc123456789 (Status: ACTIVE)                           |   |
|   +───────────────────────────────────────────────────────────────────────────────────────────+   |
|                       ▲                                                     ▲                     |
|                       │                                                     │                     |
|          ROUTE: 172.31.0.0/16 -> pcx-xxxx                       ROUTE: 10.0.0.0/16 -> pcx-xxxx    |
|          (In Public, Private & DB Route Tables)                 (In Default VPC Main Route Table) |
+---------------------------------------------------------------------------------------------------+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### VPC Peering Connection ([`peering.tf`: Lines 1-27](../../terraform-aws-vpc/peering.tf#L1-L27))

```hcl
1: resource "aws_vpc_peering_connection" "default" {
2:   count = var.is_peering_required ? 1 : 0
3: 
4:   # Acceptor
5:   peer_vpc_id = data.aws_vpc.default.id
6: 
7:   # Requestor
8:   vpc_id      = aws_vpc.main.id
9: 
10:   auto_accept = true
11: 
12:   accepter {
13:     allow_remote_vpc_dns_resolution = true
14:   }
15: 
16:   requester {
17:     allow_remote_vpc_dns_resolution = true
18:   }
19: 
20:   tags = merge(
21:     local.common_tags,
22:     {
23:       Name = "${var.project}-${var.environment}-default"
24:     }
25:   )
26: }
```

##### Line-by-Line Breakdown:
- **Line 2 (`count = var.is_peering_required ? 1 : 0`)**: Conditional toggle. Only provisions peering if consumer sets `is_peering_required = true`.
- **Line 5 (`peer_vpc_id = data.aws_vpc.default.id`)**: Dynamically resolves the Accepter VPC ID from data source query.
- **Line 8 (`vpc_id = aws_vpc.main.id`)**: Sets the Requester VPC ID to the newly created Roboshop custom VPC.
- **Line 10 (`auto_accept = true`)**: Automatically accepts the peering connection on AWS because both VPCs belong to the same AWS account.
- **Lines 12-18 (`allow_remote_vpc_dns_resolution = true`)**: Enables bidirectional private DNS resolution across peered VPCs. Allows EC2 instances in the default VPC to resolve private hostnames in the custom VPC.

---

### Custom VPC Routes to Default VPC ([`peering.tf`: Lines 29-49](../../terraform-aws-vpc/peering.tf#L29-L49))

```hcl
29: resource "aws_route" "public_peering" {
30:   count                     = var.is_peering_required ? 1 : 0
31:   route_table_id            = aws_route_table.public.id
32:   destination_cidr_block    = data.aws_vpc.default.cidr_block
33:   vpc_peering_connection_id = aws_vpc_peering_connection.default[count.index].id
34: }
36: resource "aws_route" "private_peering" {
37:   count                     = var.is_peering_required ? 1 : 0
38:   route_table_id            = aws_route_table.private.id
39:   destination_cidr_block    = data.aws_vpc.default.cidr_block
40:   vpc_peering_connection_id = aws_vpc_peering_connection.default[count.index].id
41: }
43: resource "aws_route" "database_peering" {
44:   count                     = var.is_peering_required ? 1 : 0
45:   route_table_id            = aws_route_table.database.id
46:   destination_cidr_block    = data.aws_vpc.default.cidr_block
47:   vpc_peering_connection_id = aws_vpc_peering_connection.default[count.index].id
48: }
```
- Adds routing rules to Public, Private, and Database Route Tables.
- **`destination_cidr_block = data.aws_vpc.default.cidr_block`**: Directs all packets destined for `172.31.0.0/16` across the peering connection (`vpc_peering_connection_id`).

---

### Default VPC Return Route ([`peering.tf`: Lines 50-55](../../terraform-aws-vpc/peering.tf#L50-L55))

```hcl
50: resource "aws_route" "default_peering" {
51:   count                     = var.is_peering_required ? 1 : 0
52:   route_table_id            = data.aws_route_table.default.id
53:   destination_cidr_block    = var.vpc_cidr
54:   vpc_peering_connection_id = aws_vpc_peering_connection.default[count.index].id
55: }
```
- **Crucial Return Route**: Networking requires bidirectional routing!
- Injects a route into the **Default VPC's Main Route Table** directing packets destined for `10.0.0.0/16` back through the peering connection.
- Without this return route, packets from Default VPC reach Custom VPC, but return packets are dropped!

---

### Data Sources for Default VPC & Route Table ([`data.tf`](../../terraform-aws-vpc/data.tf))

```hcl
5: data "aws_vpc" "default" {
6:   default = true
7: }
8: 
9: data "aws_route_table" "default" {
10:   vpc_id = data.aws_vpc.default.id
11:   filter {
12:     name   = "association.main"
13:     values = ["true"]
14:   }
15: }
```
- Queries the default VPC and its primary main route table dynamically without hardcoding IDs.

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
- Sets `is_peering_required = true` to activate all peering resources and bidirectional routes.

---

## 4. Step-by-Step Hands-on Execution Walkthrough

All commands below are directly copyable:

### 1. Testing VPC Peering via Test Harness
```bash
# Navigate to test consumer directory
cd /Users/sriramcharankolla/Desktop/DevOps/vpc-module-test

# Initialize Terraform (fetches updated child module code)
terraform init

# Validate syntax
terraform validate

# Plan and verify that Peering Connection + 4 Peering Routes are included
terraform plan

# Apply infrastructure to AWS
terraform apply -auto-approve
```

---

### 2. Verifying Peering & Routing with AWS CLI
```bash
# 1. Check Peering Connection State (Expected: active)
aws ec2 describe-vpc-peering-connections \
  --filters "Name=tag:Name,Values=roboshop-dev-default" \
  --query "VpcPeeringConnections[*].{PeeringId:VpcPeeringConnectionId,Status:Status.Code,Requester:RequesterVpcInfo.VpcId,Accepter:AccepterVpcInfo.VpcId}" \
  --output table

# 2. Audit Custom VPC Route Tables for Peering Target
aws ec2 describe-route-tables \
  --filters "Name=tag:Project,Values=roboshop" \
  --query "RouteTables[*].{Name:Tags[?Key=='Name'].Value|[0],Routes:Routes[?VpcPeeringConnectionId!=null]}" \
  --output json

# 3. Audit Default VPC Route Table for Return Route
aws ec2 describe-route-tables \
  --filters "Name=association.main,Values=true" \
  --query "RouteTables[?VpcId!=''].{VpcId:VpcId,Routes:Routes[?DestinationCidrBlock=='10.0.0.0/16']}" \
  --output table
```

---

### 3. Clean Up (Teardown)
```bash
terraform destroy -auto-approve
```

---

## 5. Commands & CLI Flags Reference Table

| Command / Tool | Syntax Example | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `aws ec2 describe-vpc-peering-connections` | `aws ec2 describe-vpc-peering-connections` | Lists peering connections, connection IDs (`pcx-xxxx`), and statuses. |
| `aws ec2 accept-vpc-peering-connection` | `aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-...` | Manually accepts a cross-account peering connection from accepter account. |
| `is_peering_required` | `is_peering_required = true` | Boolean toggle meta-argument controlling conditional resource creation in Terraform. |
| `auto_accept` argument | Inside `aws_vpc_peering_connection` | Automatically activates connection when both VPCs are in the same AWS account. |

---

## 6. Official Documentation & References
- **AWS VPC Peering Guide**: [docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)
- **VPC Peering Limitations & Unsupported Configurations**: [docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html#vpc-peering-limitations](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html#vpc-peering-limitations)
- **Terraform `aws_vpc_peering_connection` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc_peering_connection](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc_peering_connection)
- **Terraform `aws_route` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route)

---

## 7. High-Yield Interview Questions & Answers

### Q1: Can two VPCs with overlapping CIDR blocks peer with each other?
**Answer**:
**No.** AWS strictly prohibits VPC Peering between VPCs with matching or overlapping CIDR blocks.
If CIDRs overlap (e.g., both are `10.0.0.0/16`), routers cannot distinguish whether an IP address belongs to the local subnet or the remote peered subnet, creating unresolvable routing ambiguity.
To interconnect overlapping networks, organizations must implement Private NAT Gateways or AWS PrivateLink.

---

### Q2: Is VPC Peering transitive? Explain with an example.
**Answer**:
**No, VPC Peering is non-transitive.**
If VPC A is peered with VPC B, and VPC B is peered with VPC C, traffic from VPC A **cannot** travel through VPC B to reach VPC C.
To enable communication between A and C, you must either create a direct peering connection between A and C, or migrate to a centralized **AWS Transit Gateway (TGW)**.

---

### Q3: Why is creating an `aws_vpc_peering_connection` resource alone not enough to enable communication?
**Answer**:
A peering connection merely establishes the physical network conduit between two VPCs.
Traffic will **not** flow until **bidirectional routing rules** are explicitly added to the Route Tables on **both** sides:
1. The Requester VPC route tables must have a route for the Accepter VPC CIDR pointing to the `pcx-xxxx` peering ID.
2. The Accepter VPC route table must have a return route for the Requester VPC CIDR pointing to the `pcx-xxxx` peering ID.
Without both routes, packets either cannot leave the source or return packets are dropped.

---

### Q4: What is the architectural difference between Project Infrastructure and Application Infrastructure?
**Answer**:
- **Project / Foundation Infrastructure**: The underlying network and core platform (VPCs, Subnets, NAT/Internet Gateways, Route Tables, Peering). Provisioned once, long-lived, rarely modified, managed by Platform Engineers.
- **Application Infrastructure**: The compute, storage, and runtime workloads (EC2 instances, Auto Scaling Groups, RDS databases, Kubernetes pods, ALB listener rules). Continuously updated, ephemeral, managed by Application/DevOps feature teams.

---

## 8. Production Mistakes & Troubleshooting Guide

1. **Missing the Return Route in Accepter VPC**:
   - *Symptom*: Pings or TCP SYN packets reach the remote EC2 instance (visible in tcpdump), but the client receives no response.
   - *Cause*: Route was added to Requester VPC route table, but Accepter VPC route table has no route for the Requester CIDR.
   - *Fix*: Always add bidirectional routes (as implemented in `peering.tf`).
2. **Peering Stuck in `pending-acceptance`**:
   - *Cause*: Attempting cross-account peering with `auto_accept = true`. `auto_accept` only works for single-account peering.
   - *Fix*: In cross-account scenarios, use an `aws_vpc_peering_connection_accepter` resource authenticated with provider credentials from the second account.
3. **Security Group Ingress Blocking Peered Traffic**:
   - *Symptom*: Routing is correctly configured, but connection times out.
   - *Cause*: The destination EC2 instance's Security Group only allows traffic from `0.0.0.0/0` on port 80/443, blocking internal ports from the peered VPC CIDR.
   - *Fix*: Add an ingress rule allowing appropriate ports from the remote VPC CIDR (`172.31.0.0/16`).

---

## 9. Session Metadata & Timestamps

- **VPC Module Recap**: `00:00 - 34:00`
- **VPC Peering Architecture & Concepts**: `34:00`
- **Overlapping CIDR & Transitive Constraints**: `45:00 - 01:00:00`
- **Project vs Application Infrastructure**: `01:00:00 - 01:14:00`
- **Session Q&A**: `01:14:30`

### Doubts & AI Clarification Link
- [Session 37 AI Clarification Chat](https://chat.z.ai/c/566b2042-5791-4a86-8ddb-484d8f10d276)