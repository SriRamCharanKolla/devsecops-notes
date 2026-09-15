# Tuesday, 3 March 2026
# Session 38 - Bastion Host Architecture, Security Group Chaining, AWS SSM Parameter Store Decoupling & VPC Peering Verification
## Comprehensive Class Notes, Multi-Tier Infrastructure Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [Project Infrastructure vs Application Infrastructure](#project-infrastructure-vs-application-infrastructure)
   - [Remote Access: VPN vs Bastion Host (Jump Host)](#remote-access-vpn-vs-bastion-host-jump-host)
   - [Security Group Chaining (SG ID vs Hardcoded IP Reference)](#security-group-chaining-sg-id-vs-hardcoded-ip-reference)
   - [VPC Peering End-to-End Connectivity Verification](#vpc-peering-end-to-end-connectivity-verification)
   - [Enterprise Decoupling via AWS SSM Parameter Store](#enterprise-decoupling-via-aws-ssm-parameter-store)
   - [The 14 Enterprise Security Groups in RoboShop](#the-14-enterprise-security-groups-in-roboshop)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct File Links](#directory-structure--direct-file-links)
   - [Architectural Diagrams](#architectural-diagrams)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Layer 1: Publishing VPC Outputs to SSM (`roboshop-infra-dev/00-vpc/`)](#layer-1-publishing-vpc-outputs-to-ssm-roboshop-infra-dev00-vpc)
     - [`parameters.tf`](#parameterstf-in-00-vpc)
   - [Layer 2: Consuming VPC & Creating SGs (`roboshop-infra-dev/10-sg/`)](#layer-2-consuming-vpc--creating-sgs-roboshop-infra-dev10-sg)
     - [`data.tf`](#datatf-in-10-sg)
     - [`variables.tf`](#variablestf-in-10-sg)
     - [`main.tf`](#maintf-in-10-sg)
     - [`parameters.tf`](#parameterstf-in-10-sg)
   - [Child Module: Reusable Security Group (`terraform-aws-sg/`)](#child-module-reusable-security-group-terraform-aws-sg)
     - [`main.tf`](#maintf-in-terraform-aws-sg)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Provisioning `00-vpc` and Exporting SSM Parameters](#1-provisioning-00-vpc-and-exporting-ssm-parameters)
   - [2. Provisioning `10-sg` and Exporting Security Group IDs](#2-provisioning-10-sg-and-exporting-security-group-ids)
   - [3. Hands-on Bastion-to-Private SSH & Telnet Testing](#3-hands-on-bastion-to-private-ssh--telnet-testing)
   - [4. Testing Inter-VPC Peering Connectivity](#4-testing-inter-vpc-peering-connectivity)
   - [5. Clean Up (Teardown)](#5-clean-up-teardown)
5. [Commands & CLI Flags Reference Table](#5-commands--cli-flags-reference-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### Project Infrastructure vs Application Infrastructure
In modern enterprise DevOps, infrastructure is divided into two distinct lifecycle tiers:
1. **Project / Platform Infrastructure (One-time, Long-lived)**:
   - VPC, Subnets, Gateways (IGW, NAT), Route Tables, Peering Connections.
   - Created once; modified very rarely (e.g., quarterly or semi-annually).
   - High blast radius: Destruction disrupts all hosted environments and services.
2. **Application Infrastructure (Continuous, High Churn)**:
   - EC2 Instances, Autoscaling Groups, ALB Target Groups, Listener Rules, Databases.
   - Deployed and destroyed frequently by feature teams (daily/hourly CI/CD releases).

---

### Remote Access: VPN vs Bastion Host (Jump Host)
To manage EC2 instances residing in private subnets, administrators cannot connect directly over the internet. Two primary access patterns exist:
1. **Client VPN / Site-to-Site VPN**:
   - Encrypted tunnel directly into the VPC network.
   - Requires dedicated VPN client software, certificates, and ongoing management overhead.
2. **Bastion Host (Jump Host)**:
   - A single, hardened EC2 instance provisioned in a **Public Subnet**.
   - Acts as a proxy gateway to reach private workloads:
     $$\text{Admin Laptop} \xrightarrow{\text{SSH (Port 22)}} \text{Bastion Host (Public IP)} \xrightarrow{\text{SSH (Port 22)}} \text{Private EC2 (Private IP)}$$
   - Direct inbound internet access to all other EC2 instances in private and database subnets is **completely blocked**.

---

### Security Group Chaining (SG ID vs Hardcoded IP Reference)
When configuring firewall permissions between the Bastion host and private EC2 instances, engineers face a critical architectural decision:

#### The Anti-Pattern: Hardcoding Bastion's Private IP:
```hcl
# BAD PRACTICE (Brittle):
ingress {
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["10.0.1.94/32"] # Bastion's current private IP (/32 = single IP)
}
```
- **Why this fails in production**: If the Bastion instance is stopped, restarted, or auto-scaled, its private IP may change! Administrators lose access and must manually update security group rules in an emergency.

#### The Enterprise Best Practice: Security Group Chaining:
```hcl
# BEST PRACTICE (Dynamic & Resilient):
ingress {
  from_port       = 22
  to_port         = 22
  protocol        = "tcp"
  security_groups = [aws_security_group.bastion.id] # Source is the Bastion SG ID!
}
```
- Instead of an IP address, the **Security Group ID** of the Bastion is declared as the traffic source.
- AWS automatically authorizes traffic from **any instance** attached to the Bastion Security Group, completely immune to IP changes!

---

### VPC Peering End-to-End Connectivity Verification
After establishing a VPC Peering connection and adding bidirectional route table entries:
1. Connecting from Bastion to Private server in Custom VPC:
   ```bash
   telnet <roboshop-dev-private-ip> 22
   ```
2. If `telnet` times out or fails:
   - Verify that the target instance's Security Group allows inbound traffic on port 22 from the caller's Security Group or source VPC CIDR.
3. Connecting from Custom VPC to Default VPC:
   ```bash
   telnet <default-vpc-server-private-ip> 22
   ```
   - Requires updating the Default VPC server's Security Group to authorize inbound port 22 traffic originating from the Roboshop VPC Security Group ID or `10.0.0.0/16` CIDR.

---

### Enterprise Decoupling via AWS SSM Parameter Store
In large projects, provisioning everything in a single massive Terraform directory creates state locking bottlenecks and high risk. Instead, architectures are divided into modular layers:
- `00-vpc` $\rightarrow$ `10-sg` $\rightarrow$ `20-sg-rules` $\rightarrow$ `30-bastion` $\rightarrow$ `40-databases` ...

#### The Parameter Store Pattern:
```
+-----------------------------------------------------------------------------------+
| LAYER 00-VPC                                                                      |
| - Provisions VPC, Subnets, Gateways                                               |
| - Publishes outputs to AWS Systems Manager Parameter Store:                       |
|   /roboshop/dev/vpc_id = "vpc-0abc12345"                                          |
|   /roboshop/dev/public_subnet_ids = "subnet-1,subnet-2"                           |
+-----------------------------------------------------------------------------------+
                                         │
                                         ▼ Published to AWS Cloud Parameter Store
+-----------------------------------------------------------------------------------+
| LAYER 10-SG                                                                       |
| - Queries SSM: data "aws_ssm_parameter" "vpc_id"                                  |
| - Provisions all 14 Security Groups using terraform-aws-sg module                 |
| - Publishes Security Group IDs back to SSM:                                       |
|   /roboshop/dev/catalogue_sg_id = "sg-0123"                                       |
|   /roboshop/dev/bastion_sg_id = "sg-0456"                                         |
+-----------------------------------------------------------------------------------+
                                         │
                                         ▼
+-----------------------------------------------------------------------------------+
| LAYER 30-BASTION                                                                  |
| - Queries SSM: data "aws_ssm_parameter" "bastion_sg_id"                            |
| - Launches Bastion EC2 into public subnet without any hardcoded IDs!              |
+-----------------------------------------------------------------------------------+
```

- **Benefits**:
  - Independent state files per layer (fast execution, zero cross-layer blast radius).
  - No need to read remote state files directly (`terraform_remote_state` anti-pattern).
  - Standardized parameter naming hierarchy: `/${project}/${environment}/${parameter_name}`.

---

### The 14 Enterprise Security Groups in RoboShop
In `roboshop-infra-dev/10-sg`, 14 distinct security groups are provisioned to enforce least-privilege microsegmentation:
1. **Databases (4 SGs)**: `mongodb`, `redis`, `mysql`, `rabbitmq`
2. **Backend Services (5 SGs)**: `catalogue`, `user`, `cart`, `shipping`, `payment`
3. **Internal Load Balancer (1 SG)**: `backend_alb`
4. **Frontend UI (1 SG)**: `frontend`
5. **External Load Balancer (1 SG)**: `frontend_alb`
6. **Management & Access (2 SGs)**: `bastion`, `openvpn`

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct File Links
Click any file link below to view it directly in your IDE:

#### Project 1: Multi-Layer Foundation (`roboshop-infra-dev`)
- **VPC Layer (00-vpc)**: [`roboshop-infra-dev/00-vpc/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/00-vpc) | [Relative Path](../../roboshop-infra-dev/00-vpc)
  - [`00-vpc/main.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/00-vpc/main.tf#L1-L7) | [Relative Link](../../roboshop-infra-dev/00-vpc/main.tf)
  - [`00-vpc/parameters.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/00-vpc/parameters.tf#L1-L22) | [Relative Link](../../roboshop-infra-dev/00-vpc/parameters.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/00-vpc/parameters.tf)
  - [`00-vpc/variables.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/00-vpc/variables.tf#L1-L6) | [Relative Link](../../roboshop-infra-dev/00-vpc/variables.tf)
  - [`00-vpc/outputs.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/00-vpc/outputs.tf#L1-L14) | [Relative Link](../../roboshop-infra-dev/00-vpc/outputs.tf)
- **Security Group Layer (10-sg)**: [`roboshop-infra-dev/10-sg/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/10-sg) | [Relative Path](../../roboshop-infra-dev/10-sg)
  - [`10-sg/data.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/10-sg/data.tf#L1-L2) | [Relative Link](../../roboshop-infra-dev/10-sg/data.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/10-sg/data.tf)
  - [`10-sg/main.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/10-sg/main.tf#L1-L9) | [Relative Link](../../roboshop-infra-dev/10-sg/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/10-sg/main.tf)
  - [`10-sg/variables.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/10-sg/variables.tf#L1-L26) | [Relative Link](../../roboshop-infra-dev/10-sg/variables.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/10-sg/variables.tf)
  - [`10-sg/parameters.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/10-sg/parameters.tf#L1-L11) | [Relative Link](../../roboshop-infra-dev/10-sg/parameters.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/10-sg/parameters.tf)

#### Project 2: Reusable Security Group Child Module
- **Directory**: [`terraform-aws-sg/`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-aws-sg) | [Relative Path](../../terraform-aws-sg)
- **Main SG Logic**: [`terraform-aws-sg/main.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-aws-sg/main.tf#L1-L20) | [Relative Link](../../terraform-aws-sg/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terraform-aws-sg/blob/main/main.tf)
- **Variables**: [`terraform-aws-sg/variable.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-aws-sg/variable.tf#L1-L19) | [Relative Link](../../terraform-aws-sg/variable.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terraform-aws-sg/blob/main/variable.tf)

---

### Architectural Diagrams

#### A. Bastion Host Jump Proxy Traffic Flow
```
+--------------------+
|  Developer Laptop  |
|  (SSH Client)      |
+--------------------+
          │
          │ 1. SSH:22 to Bastion Public IP
          ▼
+───────────────────────────────────────────────────────────────────────────────────+
| ROBOSHOP DEV VPC (10.0.0.0/16)                                                    |
|                                                                                   |
|   +───────────────────────────────────+                                           |
|   | PUBLIC SUBNET (10.0.1.0/24)       |                                           |
|   |                                   |                                           |
|   |  Bastion Host (Jump Host)         |                                           |
|   |  Public IP:  54.78.98.123         |                                           |
|   |  Private IP: 10.0.1.94            |                                           |
|   |  SG: "roboshop-dev-bastion"       |                                           |
|   +───────────────────────────────────+                                           |
|                     │                                                             |
|                     │ 2. SSH:22 to Private IP (10.0.11.233)                       |
|                     │    Authorized via SG Reference (source = bastion_sg_id)     |
|                     ▼                                                             |
|   +───────────────────────────────────+                                           |
|   | PRIVATE SUBNET (10.0.11.0/24)     |                                           |
|   |                                   |                                           |
|   |  Backend Instance (Catalogue/User)|                                           |
|   |  Private IP: 10.0.11.233          |                                           |
|   |  SG: "roboshop-dev-catalogue"     |                                           |
|   +───────────────────────────────────+                                           |
+───────────────────────────────────────────────────────────────────────────────────+
```

#### B. SSM Parameter Store Decoupled Pipeline
```
┌─────────────────────────────────┐               ┌─────────────────────────────────┐
│       00-vpc (Platform)         │               │     10-sg (Security Team)       │
│                                 │               │                                 │
│  aws_ssm_parameter.vpc_id       │               │  data.aws_ssm_parameter.vpc_id  │
│  "/${project}/${env}/vpc_id"    │               │  (Reads VPC ID dynamically)     │
└────────────────┬────────────────┘               └───────────────▲─────────────────┘
                 │                                                │
                 │ Writes Parameter                               │ Queries Parameter
                 ▼                                                │
+─────────────────────────────────────────────────────────────────┴─────────────────+
|                       AWS SYSTEMS MANAGER (SSM) PARAMETER STORE                   |
|                                                                                   |
|   Key: /roboshop/dev/vpc_id              -> Value: "vpc-01a2b3c4d5e6"             |
|   Key: /roboshop/dev/public_subnet_ids   -> Value: "subnet-111,subnet-222"        |
|   Key: /roboshop/dev/catalogue_sg_id     -> Value: "sg-0abc1234"                  |
|   Key: /roboshop/dev/bastion_sg_id       -> Value: "sg-0def5678"                  |
+───────────────────────────────────────────────────────────────────────────────────+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Layer 1: Publishing VPC Outputs to SSM ([`roboshop-infra-dev/00-vpc/parameters.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/00-vpc/parameters.tf#L1-L22) | [Relative](../../roboshop-infra-dev/00-vpc/parameters.tf))

```hcl
1: resource "aws_ssm_parameter" "vpc_id" {
2:   name  = "/${var.project}/${var.environment}/vpc_id"
3:   type  = "String"
4:   value = module.vpc.vpc_id
5: }
6: 
7: resource "aws_ssm_parameter" "public_subnet_ids" {
8:   name  = "/${var.project}/${var.environment}/public_subnet_ids"
9:   type  = "StringList"
10:   value = join(",", module.vpc.public_subnet_ids)
11: }
13: resource "aws_ssm_parameter" "private_subnet_ids" {
14:   name  = "/${var.project}/${var.environment}/private_subnet_ids"
15:   type  = "StringList"
16:   value = join(",", module.vpc.private_subnet_ids)
17: }
19: resource "aws_ssm_parameter" "database_subnet_ids" {
20:   name  = "/${var.project}/${var.environment}/database_subnet_ids"
21:   type  = "StringList"
22:   value = join(",", module.vpc.database_subnet_ids)
23: }
```

##### Line-by-Line Breakdown:
- **Lines 1-5 (`aws_ssm_parameter "vpc_id"`)**:
  - `name`: Hierarchical string path `"/roboshop/dev/vpc_id"`.
  - `type = "String"`: Single string data type in SSM.
  - `value = module.vpc.vpc_id`: Pulls the VPC ID attribute exported by the child module.
- **Lines 7-11 (`aws_ssm_parameter "public_subnet_ids"`)**:
  - `type = "StringList"`: Built-in SSM type for comma-separated values.
  - `join(",", module.vpc.public_subnet_ids)`: Built-in HCL function converting list `["subnet-1", "subnet-2"]` into `"subnet-1,subnet-2"`.

---

### Layer 2: Consuming VPC & Creating SGs ([`roboshop-infra-dev/10-sg/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/10-sg) | [Relative](../../roboshop-infra-dev/10-sg))

#### [`data.tf` in `10-sg/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/10-sg/data.tf#L1-L2) | [Relative](../../roboshop-infra-dev/10-sg/data.tf)

```hcl
1: data "aws_ssm_parameter" "vpc_id" {
2:   name = "/${var.project}/${var.environment}/vpc_id"
3: }
```
- Completely decouples the SG layer from the VPC layer. Reads the VPC ID directly from AWS Systems Manager without loading `00-vpc` state!

---

#### [`variables.tf` in `10-sg/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/10-sg/variables.tf#L1-L26) | [Relative](../../roboshop-infra-dev/10-sg/variables.tf)

```hcl
9: variable "sg_names" {
10:   type = list
11:   default = [
12:     # Databases
13:     "mongodb", "redis", "mysql", "rabbitmq",
14:     # Backend
15:     "catalogue", "user", "cart", "shipping", "payment",
16:     # Backed ALB
17:     "backend_alb",
18:     # Frontend
19:     "frontend",
20:     # Frontend ALB
21:     "frontend_alb",
22:     # Bastion
23:     "bastion",
24:     # Openvpn
25:     "openvpn"
26:   ]
27: }
```
- Centralized inventory declaring all 14 RoboShop component security groups.

---

#### [`main.tf` in `10-sg/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/10-sg/main.tf#L1-L9) | [Relative](../../roboshop-infra-dev/10-sg/main.tf)

```hcl
1: module "sg" {
2:   count       = length(var.sg_names)
3:   source      = "git::https://github.com/SriRamCharanKolla/terraform-aws-sg.git"
4:   project     = var.project
5:   environment = var.environment
6:   sg_name     = replace(var.sg_names[count.index], "_", "-")
7:   vpc_id      = local.vpc_id
8: }
```
- **Line 2 (`count = length(var.sg_names)`)**: Dynamically iterates over all 14 items.
- **Line 3 (`source = "git::https://..."`)**: Consumes the reusable child module directly from GitHub.
- **Line 6 (`replace(...)`)**: Replaces programmatic underscores with human-readable hyphens (`"backend_alb"` $\rightarrow$ `"backend-alb"`).
- **Line 7 (`vpc_id = local.vpc_id`)**: Passes the dynamically fetched SSM parameter value.

---

#### [`parameters.tf` in `10-sg/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/10-sg/parameters.tf#L1-L11) | [Relative](../../roboshop-infra-dev/10-sg/parameters.tf)

```hcl
7: resource "aws_ssm_parameter" "sg_id" {
8:   count = length(var.sg_names)
9:   name  = "/${var.project}/${var.environment}/${var.sg_names[count.index]}_sg_id"
10:   type  = "String"
11:   value = module.sg[count.index].sg_id
12: }
```
- Exports all 14 created Security Group IDs back to SSM Parameter Store (e.g., `"/roboshop/dev/catalogue_sg_id"`), ready for downstream compute and ALB modules to consume!

---

### Child Module: Reusable Security Group ([`terraform-aws-sg/main.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-aws-sg/main.tf#L1-L20) | [Relative](../../terraform-aws-sg/main.tf))

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
12: 
13:   tags = merge(
14:     var.sg_tags,
15:     local.common_tags,
16:     {
17:       Name = "${var.project}-${var.environment}-${var.sg_name}"
18:     }
19:   )
20: }
```
- Follows the single-resource standard name `"main"`.
- Sets up default allow-all egress, standardized naming, and parameter-driven VPC association.

---

## 4. Step-by-Step Hands-on Execution Walkthrough

All commands below are directly copyable:

### 1. Provisioning `00-vpc` and Exporting SSM Parameters
```bash
# Navigate to VPC foundation layer
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/00-vpc

# Initialize and apply
terraform init
terraform apply -auto-approve

# Verify parameters in AWS SSM Parameter Store
aws ssm get-parameters-by-path \
  --path "/roboshop/dev" \
  --query "Parameters[*].{Name:Name,Type:Type,Value:Value}" \
  --output table
```

---

### 2. Provisioning `10-sg` and Exporting Security Group IDs
```bash
# Navigate to Security Group layer
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/10-sg

# Initialize (downloads terraform-aws-sg module from Git)
terraform init

# Apply all 14 Security Groups
terraform apply -auto-approve

# Inspect created security groups in SSM
aws ssm get-parameters-by-path \
  --path "/roboshop/dev" \
  --query "Parameters[?contains(Name, 'sg_id')].{Name:Name,Value:Value}" \
  --output table
```

---

### 3. Hands-on Bastion-to-Private SSH & Telnet Testing
Follow this exact sequence to test access from your local machine:

```bash
# 1. SSH into the Bastion Host using its Public IP
ssh -i /path/to/key.pem ec2-user@<BASTION_PUBLIC_IP>

# 2. Inside Bastion: Attempt telnet to Private EC2 instance on Port 22
telnet <PRIVATE_EC2_IP> 22
# Result: Connection will time out or be refused if SG rule is missing!

# 3. Add Security Group Ingress Rule (via Console or Terraform):
#    Authorize Type: SSH (Port 22), Source: <BASTION_SG_ID>

# 4. Re-run telnet inside Bastion
telnet <PRIVATE_EC2_IP> 22
# Result: Connected! SSH banner (SSH-2.0-OpenSSH...) displays successfully!

# 5. SSH from Bastion into Private EC2
ssh ec2-user@<PRIVATE_EC2_IP>
```

---

### 4. Testing Inter-VPC Peering Connectivity
```bash
# Inside Private EC2 in Roboshop VPC (10.0.11.x):
# Attempt connection to Management server in Default VPC (172.31.x.x)
telnet <DEFAULT_VPC_SERVER_PRIVATE_IP> 22

# If connection hangs:
# Update the Default VPC server's Security Group to allow Port 22 from the caller's Security Group or 10.0.0.0/16 CIDR!
```

---

### 5. Clean Up (Teardown)
> [!NOTE]
> Always destroy in reverse dependency order: First `10-sg`, then `00-vpc`!

```bash
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/10-sg
terraform destroy -auto-approve

cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/00-vpc
terraform destroy -auto-approve
```

---

## 5. Commands & CLI Flags Reference Table

| Command / Tool | Syntax Example | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `aws ssm get-parameter` | `aws ssm get-parameter --name "/roboshop/dev/vpc_id"` | Queries the value of a specific parameter from SSM. |
| `aws ssm get-parameters-by-path` | `aws ssm get-parameters-by-path --path "/roboshop/dev"` | Recursively lists all parameters under an environment namespace. |
| `telnet <host> <port>` | `telnet 10.0.11.233 22` | Verifies TCP connectivity and handshake to a remote port. |
| `nc -zv <host> <port>` | `nc -zv 10.0.11.233 22` | Fast netcat port check without interactive telnet banner. |
| `replace()` Function | `replace("backend_alb", "_", "-")` | Built-in HCL function converting underscores to hyphens. |

---

## 6. Official Documentation & References
- **AWS SSM Parameter Store Guide**: [docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- **Terraform `aws_ssm_parameter` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ssm_parameter](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ssm_parameter)
- **Terraform `aws_ssm_parameter` Data Source**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ssm_parameter](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ssm_parameter)
- **AWS Security Group Rules Reference Source**: [docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html)

---

## 7. High-Yield Interview Questions & Answers

### Q1: Why should you reference a Security Group ID instead of an IP/CIDR in firewall ingress rules?
**Answer**:
1. **Dynamic Resiliency**: In cloud environments, instances in autoscaling groups or ephemeral test environments frequently receive new private IP addresses upon launch or reboot. If rules use hardcoded IPs (`10.0.1.94/32`), access breaks as soon as the instance restarts.
2. **Security Group Chaining**: When you specify `security_groups = [aws_security_group.bastion.id]`, AWS automatically allows traffic from **any current or future EC2 instance attached to that security group**, regardless of IP changes or autoscaling events.
3. **Least Privilege & Simplicity**: Eliminates the need to maintain hundreds of dynamic IP addresses across enterprise firewall rules.

---

### Q2: How do you decouple multi-tier Terraform layers without sharing monolithic state files?
**Answer**:
By using **AWS Systems Manager (SSM) Parameter Store**:
- The upstream layer (`00-vpc`) creates resources and writes their exported attributes into SSM parameters under a standard namespace (`/${project}/${environment}/vpc_id`).
- Downstream layers (`10-sg`, `30-bastion`) query SSM using `data "aws_ssm_parameter"`.
- This eliminates monolithic state files, reduces blast radius, prevents cross-team state lock contention, and avoids brittle `terraform_remote_state` data sources.

---

### Q3: What is the difference between an AWS SSM Parameter of type `String` vs `StringList`?
**Answer**:
- **`String`**: Holds a single text value (e.g., `"vpc-0123456789abcdef0"`).
- **`StringList`**: Holds a comma-separated list of values (e.g., `"subnet-1a,subnet-1b,subnet-1c"`). When consumed in Terraform, it can be parsed back into an HCL list using the `split(",", data.aws_ssm_parameter.subnets.value)` function.

---

### Q4: How does an administrator connect from their laptop to a private database server using a Bastion Host?
**Answer**:
There are two standard methods:
1. **SSH Jump Proxy (`-J` flag)**:
   `ssh -J ec2-user@<BASTION_PUBLIC_IP> ec2-user@<PRIVATE_DB_IP>`
2. **SSH Agent Forwarding (`-A` flag)**:
   Add private key to local agent (`ssh-add key.pem`), connect to Bastion (`ssh -A ec2-user@<BASTION_PUBLIC_IP>`), then SSH directly to private IP without storing private keys on the Bastion server.

---

## 8. Production Mistakes & Troubleshooting Guide

1. **Hardcoding Private IPs in Security Group Rules**:
   - *Problem*: Bastion instance reboot changes its private IP from `10.0.1.94` to `10.0.1.182`, breaking all SSH access to private microservices.
   - *Fix*: Always use Security Group Chaining (`security_groups = [module.bastion_sg.id]`).
2. **SSM Parameter Path Inconsistency**:
   - *Error*: `Error: ParameterNotFound: Parameter /roboshop/dev/vpc_id not found.`
   - *Cause*: Typo in parameter name (e.g., `_` vs `-` or missing leading slash).
   - *Fix*: Standardize path structure across all layers: `/${var.project}/${var.environment}/${parameter_name}`.
3. **Leaving Storing Private Keys on the Bastion Server**:
   - *Security Risk*: Storing `private-key.pem` on the Bastion file system means that if the Bastion is compromised, all private servers are compromised.
   - *Best Practice*: Use **SSH Agent Forwarding** (`ssh-agent` and `ssh -A`) or AWS Systems Manager Session Manager (SSM Session Manager) to connect without managing SSH keys or open port 22!

---

## 9. Session Metadata & Timestamps

- **Project Infra vs Application Infra**: `00:00 - 20:00`
- **Bastion Host & Security Group Chaining**: `20:00 - 45:00`
- **VPC Peering Connectivity & Telnet Verification**: `45:00 - 01:05:00`
- **SSM Parameter Store Decoupled Architecture**: `01:05:00 - 01:25:00`
- **Session Q&A**: `01:28:40`

### Doubts & AI Clarification Link
- [Session 38 AI Clarification Chat](https://chat.z.ai/c/f3862eee-15cb-49ca-84e8-41702b207049)

---

### Technical Interview Coding Challenge (Techolution.com)
**Question**:
*Design a type-safe User model in TypeScript for create, update, and fetch operations, ensuring proper use of generics and utility types without using `any`.*

**Production TypeScript Implementation**:
```typescript
// 1. Base Entity Interface
interface BaseEntity {
  readonly id: string;
  readonly createdAt: Date;
  updatedAt: Date;
}

// 2. Core Domain Model
export interface User extends BaseEntity {
  name: string;
  email: string;
  age?: number;
  role: 'admin' | 'developer' | 'viewer';
  isActive: boolean;
}

// 3. Create DTO: Omit auto-generated metadata fields
export type CreateUserDTO = Omit<User, keyof BaseEntity>;

// 4. Update DTO: Partial of CreateUserDTO (id is immutable, updates are optional)
export type UpdateUserDTO = Partial<CreateUserDTO>;

// 5. Readonly Fetch Projection (Immutable representation)
export type FetchUserResponse = Readonly<User>;

// 6. Generic Type-Safe Repository Interface
export interface UserRepository<T extends BaseEntity> {
  create(payload: Omit<T, keyof BaseEntity>): Promise<T>;
  update(id: string, payload: Partial<Omit<T, keyof BaseEntity>>): Promise<T>;
  findById(id: string): Promise<T | null>;
  list(filter?: Partial<T>): Promise<T[]>;
}
```