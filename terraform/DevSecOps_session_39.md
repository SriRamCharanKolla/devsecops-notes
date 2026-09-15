# Wednesday, 4 March 2026
# Session 39 - Security Group Rules, Bastion Architecture, AWS IAM Roles & Instance Profiles
## Comprehensive Class Notes, Multi-Tier Infrastructure Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [Microsegmentation: Security Group Ingress Rules Between Instances](#microsegmentation-security-group-ingress-rules-between-instances)
   - [AWS SSM Parameter Types: String vs StringList Handling in Terraform](#aws-ssm-parameter-types-string-vs-stringlist-handling-in-terraform)
   - [Resource Naming & String Formatting (`replace`)](#resource-naming--string-formatting-replace)
   - [AWS Identity & Access Management (IAM) Deep Dive](#aws-identity--access-management-iam-deep-dive)
     - [The IAM Hierarchy: Users, Groups, Roles, Policies](#the-iam-hierarchy-users-groups-roles-policies)
     - [Role-Based Access Control (RBAC) in DevOps Teams](#role-based-access-control-rbac-in-devops-teams)
     - [The Grammar of Cloud Security: Nouns (Resources) vs Verbs (Actions)](#the-grammar-of-cloud-security-nouns-resources-vs-verbs-actions)
     - [IAM User vs IAM Role: Humans vs Non-Humans](#iam-user-vs-iam-role-humans-vs-non-humans)
     - [EC2 IAM Instance Profile: Temporary Credential Delivery](#ec2-iam-instance-profile-temporary-credential-delivery)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct File Links](#directory-structure--direct-file-links)
   - [Architectural Diagrams](#architectural-diagrams)
     - [A. End-to-End Microservice Security Group Ingress Mesh](#a-end-to-end-microservice-security-group-ingress-mesh)
     - [B. EC2 IAM Role & Instance Profile STS AssumeRole Flow](#b-ec2-iam-role--instance-profile-sts-assumerole-flow)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Layer 3: Security Group Rules (`roboshop-infra-dev/20-sg-rules/`)](#layer-3-security-group-rules-roboshop-infra-dev20-sg-rules)
     - [`data.tf`](#datatf-in-20-sg-rules)
     - [`locals.tf`](#localstf-in-20-sg-rules)
     - [`main.tf`](#maintf-in-20-sg-rules)
     - [`sg_rules.yaml`](#sg_rulesyaml-in-20-sg-rules)
   - [Layer 4: Bastion Host & IAM Provisioning (`roboshop-infra-dev/30-bastion/`)](#layer-4-bastion-host--iam-provisioning-roboshop-infra-dev30-bastion)
     - [`data.tf`](#datatf-in-30-bastion)
     - [`locals.tf`](#localstf-in-30-bastion)
     - [`main.tf`](#maintf-in-30-bastion)
     - [`bastion.sh`](#bastionsh-in-30-bastion)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Exporting Subnets from `00-vpc` using `join()`](#1-exporting-subnets-from-00-vpc-using-join)
   - [2. Deploying `20-sg-rules` for Zero-Trust Inter-Service Communication](#2-deploying-20-sg-rules-for-zero-trust-inter-service-communication)
   - [3. Creating IAM Role for Bastion (AWS Console vs Terraform)](#3-creating-iam-role-for-bastion-aws-console-vs-terraform)
   - [4. Deploying `30-bastion` and Validating Instance Profile Access](#4-deploying-30-bastion-and-validating-instance-profile-access)
   - [5. Clean Up (Teardown)](#5-clean-up-teardown)
5. [Commands & CLI Flags Reference Table](#5-commands--cli-flags-reference-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### Microsegmentation: Security Group Ingress Rules Between Instances
In production multi-tier architectures, instances must never communicate over open ports or arbitrary CIDR blocks. Every ingress connection is strictly locked down to the exact caller's Security Group ID:
- **Example Flow**:
  - `bastion` $\rightarrow$ `mongodb`: Port 22 allowed from `bastion_sg_id` only.
  - `catalogue` $\rightarrow$ `mongodb`: Port 27017 allowed from `catalogue_sg_id` only.
  - `user` $\rightarrow$ `mongodb`: Port 27017 allowed from `user_sg_id` only.
  - `user` $\rightarrow$ `redis`: Port 6379 allowed from `user_sg_id` only.
- **Why Separate Rules (`aws_security_group_rule`) from Security Groups (`aws_security_group`)?**:
  - If Security Group A refers to Security Group B in an inline block, and Security Group B refers back to Security Group A, Terraform throws a **Cyclic Dependency Loop Error** (`Cycle: module.sg_a -> module.sg_b -> module.sg_a`).
  - Creating security group containers first (`10-sg`), and applying independent rule attachments (`20-sg-rules`) completely breaks cyclic dependency deadlocks.

---

### AWS SSM Parameter Types: String vs StringList Handling in Terraform
AWS Systems Manager Parameter Store supports three data types: `String`, `StringList`, and `SecureString`.

1. **Publishing Lists to SSM (`00-vpc/parameters.tf`)**:
   - Terraform lists (e.g. `["subnet-048c42aeba89987f1", "subnet-08d03c4487720a4c0"]`) cannot be stored directly as native HCL arrays in SSM.
   - SSM `StringList` requires a comma-separated string: `"subnet-1,subnet-2"`.
   - We use Terraform's built-in `join(",", list)` function:
     ```hcl
     value = join(",", module.vpc.public_subnet_ids)
     ```

2. **Consuming SSM `StringList` in Terraform (`30-bastion/locals.tf`)**:
   - When Terraform queries a `StringList` parameter via `data.aws_ssm_parameter.public_subnet_ids.value`, AWS returns a single comma-separated string: `"subnet-1,subnet-2"`.
   - Terraform converts this string back into a list using `split(",", string)`:
     ```hcl
     # Convert comma-separated string to List and extract the 0th element (AZ 1a)
     public_subnet_id = split(",", data.aws_ssm_parameter.public_subnet_ids.value)[0]
     ```

---

### Resource Naming & String Formatting (`replace`)
AWS resources like Load Balancers, Target Groups, and IAM Roles forbid underscores (`_`) in their names, while internal variable identifiers commonly use underscores:
```hcl
# Input: "backend_alb"
replace(var.sg_names[count.index], "_", "-")
# Output: "backend-alb"
```
This ensures uniform compliance with AWS naming standards without changing variable definitions across modules.

---

### AWS Identity & Access Management (IAM) Deep Dive

#### The IAM Hierarchy: Users, Groups, Roles, Policies
AWS IAM governs authentication ("who you are") and authorization ("what you can do"):
1. **IAM User**: An identity created for a single physical person. Has permanent credentials (console password, access keys).
2. **User Group**: A collection of IAM users used to apply batch permissions (e.g., `roboshop-trainee`, `roboshop-devs`). Users inherit all permissions attached to their groups.
3. **IAM Role**: An identity intended for non-human entities (e.g., EC2 instances, Lambda functions, ECS tasks, cross-account pipelines). Has **no permanent credentials**; uses temporary security tokens via AWS STS (`AssumeRole`).
4. **IAM Policy**: A JSON document formally declaring allowed or denied API actions on specific resources.

---

#### Role-Based Access Control (RBAC) in DevOps Teams
In high-maturity DevOps organizations, permissions are mapped to team engineering tiers:

| Team Role | Operational Responsibility | AWS IAM Permissions Matrix |
| :--- | :--- | :--- |
| **Trainee / Intern** | Observation, log checking, metric review | **Read-Only Access** (`Describe*`, `Get*`, `List*`) |
| **Junior DevOps Engineer** | Deploying non-prod resources, minor fixes | **Read + Create** (`Describe*`, `Get*`, `Create*`, `RunInstances`) |
| **Senior DevOps Engineer** | Upgrading, scaling, modifying topologies | **Read + Create + Update** (`Modify*`, `Update*`, `Attach*`) |
| **Team Lead / Architect** | Architecture overhaul, decommissioning | **Read + Create + Update + Delete** (Full lifecycle access) |
| **Engineering Manager** | Emergency overrides, compliance validation | **Read + Create + Update + Delete** + IAM Auditing |

---

#### The Grammar of Cloud Security: Nouns (Resources) vs Verbs (Actions)
Every cloud interaction translates into a grammatical sentence:
- **Nouns (Resources)**: The cloud entities you want to operate on (`EC2`, `S3`, `VPC`, `RDS`, `IAM`).
- **Verbs (Actions)**: The API calls you execute against those entities:
  - `ec2:RunInstances` (Create EC2)
  - `ec2:DescribeInstances` (Read EC2)
  - `ec2:ModifyInstanceAttribute` (Update EC2)
  - `ec2:TerminateInstances` (Delete EC2)

A security policy simply links **Who** (Principal) can perform **Which Verbs** (Action) on **Which Nouns** (Resource) under **Which Conditions** (Condition).

---

#### IAM User vs IAM Role: Humans vs Non-Humans

| Feature | IAM User | IAM Role |
| :--- | :--- | :--- |
| **Target Audience** | **Humans** (Developers, Administrators) | **Non-Humans / Services** (EC2, Lambda, Pods) |
| **Credential Type** | Permanent (Username/Password, Access Key/Secret Key) | Ephemeral / Temporary (STS Tokens rotated automatically) |
| **Security Risk** | High (Keys can be accidentally committed to GitHub) | Extremely Low (Tokens expire automatically in minutes/hours) |
| **Assignment Mechanism** | Direct or via User Group | Attached to service via **Instance Profile** or `sts:AssumeRole` |

---

#### EC2 IAM Instance Profile: Temporary Credential Delivery
An EC2 instance cannot assume an IAM Role directly through standard user bindings. AWS requires a bridge known as an **Instance Profile**:
1. You create an IAM Role (`aws_iam_role`) specifying `ec2.amazonaws.com` as the trusted service principal in its `assume_role_policy`.
2. You attach managed permission policies (`aws_iam_role_policy_attachment`) to that role.
3. You create an IAM Instance Profile (`aws_iam_instance_profile`) and place the IAM Role inside it.
4. You assign the instance profile to the EC2 instance (`iam_instance_profile = aws_iam_instance_profile.bastion.name`).
5. Inside the running EC2 instance, the AWS CLI and SDKs automatically query the local **Instance Metadata Service (IMDSv2)** at `http://169.254.169.254/latest/meta-data/iam/security-credentials/<RoleName>` to retrieve short-lived session tokens without storing keys on disk!

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct File Links
Click any file link below to view it directly in your IDE:

#### Project: Multi-Tier Foundation (`roboshop-infra-dev`)
- **VPC Layer (`00-vpc/`)**: [roboshop-infra-dev/00-vpc/](../../roboshop-infra-dev/00-vpc)
  - [00-vpc/parameters.tf](../../roboshop-infra-dev/00-vpc/parameters.tf)
- **Security Group Layer (`10-sg/`)**: [roboshop-infra-dev/10-sg/](../../roboshop-infra-dev/10-sg)
  - [10-sg/main.tf](../../roboshop-infra-dev/10-sg/main.tf)
  - [10-sg/parameters.tf](../../roboshop-infra-dev/10-sg/parameters.tf)
- **Security Group Rules Layer (`20-sg-rules/`)**: [roboshop-infra-dev/20-sg-rules/](../../roboshop-infra-dev/20-sg-rules)
  - [20-sg-rules/provider.tf](../../roboshop-infra-dev/20-sg-rules/provider.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/20-sg-rules/provider.tf)
  - [20-sg-rules/main.tf](../../roboshop-infra-dev/20-sg-rules/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/20-sg-rules/main.tf)
  - [20-sg-rules/data.tf](../../roboshop-infra-dev/20-sg-rules/data.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/20-sg-rules/data.tf)
  - [20-sg-rules/locals.tf](../../roboshop-infra-dev/20-sg-rules/locals.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/20-sg-rules/locals.tf)
  - [20-sg-rules/sg_rules.yaml](../../roboshop-infra-dev/20-sg-rules/sg_rules.yaml) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/20-sg-rules/sg_rules.yaml)
- **Bastion Host & Compute Layer (`30-bastion/`)**: [roboshop-infra-dev/30-bastion/](../../roboshop-infra-dev/30-bastion)
  - [30-bastion/provider.tf](../../roboshop-infra-dev/30-bastion/provider.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/30-bastion/provider.tf)
  - [30-bastion/main.tf](../../roboshop-infra-dev/30-bastion/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/30-bastion/main.tf)
  - [30-bastion/data.tf](../../roboshop-infra-dev/30-bastion/data.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/30-bastion/data.tf)
  - [30-bastion/locals.tf](../../roboshop-infra-dev/30-bastion/locals.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/30-bastion/locals.tf)
  - [30-bastion/bastion.sh](../../roboshop-infra-dev/30-bastion/bastion.sh) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/30-bastion/bastion.sh)

---

### Architectural Diagrams

#### A. End-to-End Microservice Security Group Ingress Mesh
```
                   +---------------------------------------+
                   |          INTERNET / ADMIN IP          |
                   +---------------------------------------+
                                       │
                                       │ Port 22 (SSH)
                                       ▼
                       +───────────────────────────────+
                       |    BASTION HOST (30-bastion)   |
                       |    SG: "roboshop-dev-bastion" |
                       +───────────────────────────────+
                         │                   │
      Port 22 (SSH Admin)│                   │ Port 22 (SSH Admin)
                         ▼                   ▼
+─────────────────────────────────+  +─────────────────────────────────+
| BACKEND SERVICES (Catalogue,    |  | DATABASES (MongoDB, Redis,      |
| User, Cart, Shipping, Payment)  |  | MySQL, RabbitMQ)                |
| SG: "roboshop-dev-catalogue",etc|  | SG: "roboshop-dev-mongodb", etc |
+─────────────────────────────────+  +─────────────────────────────────+
                 │                                   ▲
                 │ Port 27017 (MongoDB App Traffic)  │
                 └───────────────────────────────────┘
```

---

#### B. EC2 IAM Role & Instance Profile STS AssumeRole Flow
```
+---------------------------------------------------------------------------------------+
| AWS IAM SYSTEM                                                                        |
|                                                                                       |
|   1. Define IAM Role: "RoboShopDevBastion"                                            |
|      - Trust Entity: ec2.amazonaws.com (sts:AssumeRole)                               |
|      - Attached Policy: AdministratorAccess (or AmazonEC2FullAccess)                  |
|                                                                                       |
|   2. Encapsulate inside Instance Profile:                                             |
|      - aws_iam_instance_profile: "roboshop-dev-bastion"                                |
+---------------------------------------------------------------------------------------+
                                           │
                                           │ 3. Attached at EC2 boot time
                                           ▼
+---------------------------------------------------------------------------------------+
| BASTION EC2 INSTANCE (30-bastion)                                                     |
|                                                                                       |
|   +-------------------------------------------------------------------------------+   |
|   | AWS CLI / Terraform Process                                                   |   |
|   |                                                                               |   |
|   | 4. Queries Link-Local IMDSv2:                                                 |   |
|   |    http://169.254.169.254/latest/meta-data/iam/security-credentials/          |   |
|   |                                                                               |   |
|   | 5. IMDSv2 delivers short-lived credentials from AWS STS:                     |   |
|   |    - AWS_ACCESS_KEY_ID                                                        |   |
|   |    - AWS_SECRET_ACCESS_KEY                                                    |   |
|   |    - AWS_SESSION_TOKEN (Auto-rotated every 6 hours)                           |   |
|   |                                                                               |   |
|   | 6. Executes Terraform deploy commands without any hardcoded credentials!     |   |
|   +-------------------------------------------------------------------------------+   |
+---------------------------------------------------------------------------------------+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Layer 3: Security Group Rules (20-sg-rules)
> **Layer Directory**: [roboshop-infra-dev/20-sg-rules/](../../roboshop-infra-dev/20-sg-rules)

#### data.tf in 20-sg-rules/
> **File**: [roboshop-infra-dev/20-sg-rules/data.tf](../../roboshop-infra-dev/20-sg-rules/data.tf)

```hcl
1: data "http" "my_public_ip_v4" {
2:   url = "https://ipv4.icanhazip.com"
3: }
4: 
5: output "my_ipv4_address" {
6:   value = chomp(data.http.my_public_ip_v4.response_body)
7: }
8: 
9: data "aws_ssm_parameter" "bastion_sg_id" {
10:   name = "/${var.project}/${var.environment}/bastion_sg_id"
11: }
12: 
13: data "aws_ssm_parameter" "mongodb_sg_id" {
14:   name = "/${var.project}/${var.environment}/mongodb_sg_id"
15: }
...
57: data "aws_ssm_parameter" "frontend_alb_sg_id" {
58:   name = "/${var.project}/${var.environment}/frontend_alb_sg_id"
59: }
```

##### Line-by-Line Breakdown:
- **Lines 1-3 (`data "http" "my_public_ip_v4"`)**: Automatically fetches the developer's external outbound public IP address via HTTP GET request to `icanhazip.com`.
- **Lines 5-7 (`output "my_ipv4_address"`)**: Strips trailing newlines using `chomp()` so it can be formatted into a clean `/32` CIDR string.
- **Lines 9-60 (`data "aws_ssm_parameter" "*_sg_id"`)**: Queries all 14 Security Group IDs published by Layer `10-sg` to SSM Parameter Store. Zero hardcoded IDs across the entire project!

---

#### locals.tf in 20-sg-rules/
> **File**: [roboshop-infra-dev/20-sg-rules/locals.tf](../../roboshop-infra-dev/20-sg-rules/locals.tf)

```hcl
1: locals {
2:   my_ip = "${chomp(data.http.my_public_ip_v4.response_body)}/32"
3:   bastion_sg_id = data.aws_ssm_parameter.bastion_sg_id.value
4:   mongodb_sg_id = data.aws_ssm_parameter.mongodb_sg_id.value
...
16:   openvpn_sg_id = data.aws_ssm_parameter.openvpn_sg_id.value
17: }
```
- **Line 2 (`my_ip`)**: Appends `/32` subnet mask to convert the raw public IP into an exact single-host CIDR notation.
- **Lines 3-16**: Unpacks the `.value` property of each SSM parameter data source into compact local variables for concise referencing.

---

#### main.tf in 20-sg-rules/
> **File**: [roboshop-infra-dev/20-sg-rules/main.tf](../../roboshop-infra-dev/20-sg-rules/main.tf)

```hcl
1: # Bastion
2: resource "aws_security_group_rule" "bastion_internet" {
3:   type              = "ingress"
4:   from_port         = 22
5:   to_port           = 22
6:   protocol          = "tcp"
7:   cidr_blocks       = ["0.0.0.0/0"]  # Alternatively: cidr_blocks = [local.my_ip] for strict access
8:   security_group_id = local.bastion_sg_id
9: }
10: 
11: # MongoDB
12: resource "aws_security_group_rule" "mongodb_bastion" {
13:   type                     = "ingress"
14:   from_port                = 22
15:   to_port                  = 22
16:   protocol                 = "tcp"
17:   source_security_group_id = local.bastion_sg_id  # Ingress allowed ONLY from Bastion!
18:   security_group_id        = local.mongodb_sg_id
19: }
20: 
21: resource "aws_security_group_rule" "mongodb_catalogue" {
22:   type                     = "ingress"
23:   from_port                = 27017
24:   to_port                  = 27017
25:   protocol                 = "tcp"
26:   source_security_group_id = local.catalogue_sg_id # Ingress allowed ONLY from Catalogue microservice!
27:   security_group_id        = local.mongodb_sg_id
28: }
29: 
30: resource "aws_security_group_rule" "mongodb_user" {
31:   type                     = "ingress"
32:   from_port                = 27017
33:   to_port                  = 27017
34:   protocol                 = "tcp"
35:   source_security_group_id = local.user_sg_id      # Ingress allowed ONLY from User microservice!
36:   security_group_id        = local.mongodb_sg_id
37: }
```

##### Line-by-Line Breakdown:
- **Lines 2-9 (`bastion_internet`)**: Inbound SSH access to Bastion on port 22. Uses `cidr_blocks = ["0.0.0.0/0"]` (or `local.my_ip` for company VPN/admin restriction).
- **Lines 12-19 (`mongodb_bastion`)**: Administrative SSH access to the MongoDB private instance on port 22, authorized strictly via `source_security_group_id = local.bastion_sg_id`.
- **Lines 21-37 (`mongodb_catalogue` & `mongodb_user`)**: Database client port `27017` opened **only** to the microservices that genuinely require it (`catalogue` and `user`). All other services in the VPC are blocked by default!

---

#### sg_rules.yaml in 20-sg-rules/
> **File**: [roboshop-infra-dev/20-sg-rules/sg_rules.yaml](../../roboshop-infra-dev/20-sg-rules/sg_rules.yaml)

```yaml
1: bastion:
2: - name: bastion_internet
3:   port: 22
4:   src: internet
5:   dest: bastion
6: mongodb:
7: - name: mongodb_bastion
8:   port: 22
9:   src: bastion
10:   dest: mongodb
11: - name: mongodb_catalogue
12:   port: 27017
13:   src: catalogue
14:   dest: mongodb
15: - name: mongodb_user
16:   port: 27017
17:   src: user
18:   dest: mongodb
```
- Demonstrates a data-driven configuration model where rule matrices can be maintained in human-readable YAML and dynamically decoded via `yamldecode()` in Terraform.

---

### Layer 4: Bastion Host & IAM Provisioning (30-bastion)
> **Layer Directory**: [roboshop-infra-dev/30-bastion/](../../roboshop-infra-dev/30-bastion)

#### data.tf in 30-bastion/
> **File**: [roboshop-infra-dev/30-bastion/data.tf](../../roboshop-infra-dev/30-bastion/data.tf)

```hcl
1: data "aws_ami" "joindevops" {
2:   most_recent = true
3:   owners      = ["973714476881"]
4: 
5:   filter {
6:     name   = "name"
7:     values = ["Redhat-9-DevOps-Practice"]
8:   }
9: 
10:   filter {
11:     name   = "root-device-type"
12:     values = ["ebs"]
13:   }
14: 
15:   filter {
16:     name   = "virtualization-type"
17:     values = ["hvm"]
18:   }
19: }
20: 
21: data "aws_ssm_parameter" "public_subnet_ids" {
22:   name = "/${var.project}/${var.environment}/public_subnet_ids"
23: }
24: 
25: data "aws_ssm_parameter" "bastion_sg_id" {
26:   name = "/${var.project}/${var.environment}/bastion_sg_id"
27: }
```
- **Lines 1-19 (`data "aws_ami" "joindevops"`)**: Dynamically resolves the latest Golden AMI ID for RHEL 9 without hardcoding AMI strings across regions.
- **Lines 21-27**: Retrieves the `StringList` of public subnets and the Bastion Security Group ID from SSM Parameter Store.

---

#### locals.tf in 30-bastion/
> **File**: [roboshop-infra-dev/30-bastion/locals.tf](../../roboshop-infra-dev/30-bastion/locals.tf)

```hcl
1: locals {
2:   ami_id = data.aws_ami.joindevops.id
3:   common_tags = {
4:     Project     = var.project
5:     Environment = var.environment
6:     Terraform   = true
7:   }
8:   # public subnet in 1a AZ
9:   public_subnet_id = split(",", data.aws_ssm_parameter.public_subnet_ids.value)[0]
10:   bastion_sg_id    = data.aws_ssm_parameter.bastion_sg_id.value
11: }
```
- **Line 9 (`public_subnet_id = split(...)[0]`)**: Splits the comma-separated `StringList` and selects the first subnet (located in `us-east-1a`) for Bastion placement.

---

#### main.tf in 30-bastion/
> **File**: [roboshop-infra-dev/30-bastion/main.tf](../../roboshop-infra-dev/30-bastion/main.tf)

```hcl
1: resource "aws_instance" "bastion" {
2:   ami                    = local.ami_id
3:   instance_type          = "t3.micro"
4:   subnet_id              = local.public_subnet_id
5:   vpc_security_group_ids = [local.bastion_sg_id]
6:   iam_instance_profile   = aws_iam_instance_profile.bastion.name
7:   user_data              = file("bastion.sh")
8: 
9:   root_block_device {
10:     volume_size = 50
11:     volume_type = "gp3"
12:     tags = merge(
13:       { Name = "${var.project}-${var.environment}-bastion" },
14:       local.common_tags
15:     )
16:   }
17: 
18:   tags = merge(
19:     { Name = "${var.project}-${var.environment}-bastion" },
20:     local.common_tags
21:   )
22: }
23: 
24: resource "aws_iam_role" "bastion" {
25:   name = "RoboShopDevBastion"
26: 
27:   assume_role_policy = jsonencode({
28:     Version = "2012-10-17"
29:     Statement = [
30:       {
31:         Action    = "sts:AssumeRole"
32:         Effect    = "Allow"
33:         Sid       = ""
34:         Principal = {
35:           Service = "ec2.amazonaws.com"
36:         }
37:       },
38:     ]
39:   })
40: }
41: 
42: resource "aws_iam_role_policy_attachment" "bastion" {
43:   role       = aws_iam_role.bastion.name
44:   policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess" # For development; least privilege in production
45: }
46: 
47: resource "aws_iam_instance_profile" "bastion" {
48:   name = "${var.project}-${var.environment}-bastion"
49:   role = aws_iam_role.bastion.name
50: }
```

##### Line-by-Line Breakdown:
- **Lines 1-8 (`aws_instance "bastion"`)**: Launches the Bastion EC2 into the public subnet with its dedicated security group, bootstrap startup script (`bastion.sh`), and attached IAM instance profile.
- **Lines 9-16 (`root_block_device`)**: Provisions an upgraded 50 GB `gp3` root volume to support heavy tooling (Terraform, Git repositories, logs).
- **Lines 24-40 (`aws_iam_role "bastion"`)**: Defines the IAM role with an **Assume Role Trust Policy** explicitly authorizing the `ec2.amazonaws.com` service principal to assume this identity.
- **Lines 42-45 (`aws_iam_role_policy_attachment "bastion"`)**: Binds an AWS Managed Policy to the role (`AdministratorAccess` or `AmazonEC2FullAccess`).
- **Lines 47-50 (`aws_iam_instance_profile "bastion"`)**: Packages the IAM role into an AWS instance profile so EC2 can ingest the credentials via IMDSv2.

---

#### bastion.sh in 30-bastion/
> **File**: [roboshop-infra-dev/30-bastion/bastion.sh](../../roboshop-infra-dev/30-bastion/bastion.sh)

```bash
1: #!/bin/bash
2: 
3: # We are creating 50GB root disk, but only 20GB is partitioned
4: # Remaining 30GB we need to extend using below commands
5: growpart /dev/nvme0n1 4
6: lvextend -r -L +30G /dev/mapper/RootVG-homeVol
7: xfs_growfs /home
8: 
9: # installing Terraform
10: yum install -y yum-utils
11: yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
12: yum -y install terraform
13: 
14: # creating databases
15: cd /home/ec2-user
16: git clone https://github.com/SriRamCharanKolla/roboshop-infra-dev.git
17: chown ec2-user:ec2-user -R roboshop-infra-dev
18: cd roboshop-infra-dev/40-databases
19: terraform init
20: terraform apply -auto-approve
21: cd ../50-backend-alb
22: terraform init
23: terraform apply -auto-approve
...
```

##### Line-by-Line Breakdown:
- **Lines 5-7 (`growpart`, `lvextend`, `xfs_growfs`)**: Resolves the standard RHEL/AWS disk partitioning gap: while AWS attaches a 50 GB block device, the partition table only formats 20 GB. `growpart` extends partition 4, `lvextend` expands the Logical Volume, and `xfs_growfs` expands the filesystem online.
- **Lines 10-12 (Terraform Installation)**: Configures the official HashiCorp yum repository and installs Terraform directly on the Bastion server.
- **Lines 15-23 (Automated Infrastructure Pipeline)**: Clones the infrastructure repository and leverages the Bastion instance's IAM Role to deploy downstream microservice layers (`40-databases`, `50-backend-alb`, etc.) without prompting for AWS credentials!

---

## 4. Step-by-Step Hands-on Execution Walkthrough

All commands below are directly copyable:

### 1. Exporting Subnets from `00-vpc` using `join()`
```bash
# Navigate to VPC foundation layer
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/00-vpc

# Apply parameters containing join(",", module.vpc.public_subnet_ids)
terraform init
terraform apply -auto-approve

# Verify that the parameter is stored as a comma-separated StringList
aws ssm get-parameter \
  --name "/roboshop/dev/public_subnet_ids" \
  --query "Parameter.Value" \
  --output text
# Expected Output: subnet-048c42aeba89987f1,subnet-08d03c4487720a4c0
```

---

### 2. Deploying `20-sg-rules` for Zero-Trust Inter-Service Communication
```bash
# Navigate to Security Group Rules layer
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/20-sg-rules

# Initialize and download providers
terraform init

# Plan and review all individual ingress rules
terraform plan

# Apply all rules into existing Security Group IDs
terraform apply -auto-approve
```

---

### 3. Creating IAM Role for Bastion (AWS Console vs Terraform)

#### Manual Console Creation Method:
1. Open the AWS Management Console $\rightarrow$ Search for **IAM**.
2. In the left navigation pane under **Access Management**, click **Roles** $\rightarrow$ Click **Create role**.
3. Under **Trusted entity type**, select **AWS service**.
4. Under **Use case**, select **EC2** $\rightarrow$ Click **Next**.
5. Under **Add permissions**, search for and select **AmazonEC2FullAccess** (or `AdministratorAccess` for non-prod testing) $\rightarrow$ Click **Next**.
6. Under **Role name**, enter `RoboshopDevBastion` $\rightarrow$ Click **Create role**.

#### Production Terraform Method (Automated in `30-bastion/main.tf`):
```bash
# Handled declaratively by aws_iam_role, aws_iam_role_policy_attachment, and aws_iam_instance_profile!
```

---

### 4. Deploying `30-bastion` and Validating Instance Profile Access
```bash
# Navigate to Bastion layer
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/30-bastion

# Initialize and apply Bastion EC2 + IAM Role + Instance Profile
terraform init
terraform apply -auto-approve

# Get Public IP of Bastion
BASTION_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=roboshop-dev-bastion" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)

echo "Bastion Public IP: $BASTION_IP"

# SSH into Bastion
ssh ec2-user@$BASTION_IP

# Inside Bastion: Verify that IAM Role is active WITHOUT access keys!
aws sts get-caller-identity
# Expected Output:
# {
#     "UserId": "AROA...:i-0123456789abcdef0",
#     "Account": "123456789012",
#     "Arn": "arn:aws:sts::123456789012:assumed-role/RoboShopDevBastion/i-0123456789abcdef0"
# }

# Verify disk partition was expanded by user-data script:
df -h /home
# Expected Output: Size should show 30G+ instead of standard default!
```

---

### 5. Clean Up (Teardown)
> [!IMPORTANT]
> Always destroy layers in reverse dependency order: `30-bastion` $\rightarrow$ `20-sg-rules` $\rightarrow$ `10-sg` $\rightarrow$ `00-vpc`!

```bash
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/30-bastion
terraform destroy -auto-approve

cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/20-sg-rules
terraform destroy -auto-approve

cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/10-sg
terraform destroy -auto-approve

cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/00-vpc
terraform destroy -auto-approve
```

---

## 5. Commands & CLI Flags Reference Table

| Command / Tool | Syntax Example | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `split()` | `split(",", "sub-1,sub-2")[0]` | Built-in HCL function converting a comma-separated string into a list and indexing the element. |
| `join()` | `join(",", ["sub-1", "sub-2"])` | Built-in HCL function concatenating a list of strings into a single comma-separated string for SSM `StringList`. |
| `chomp()` | `chomp("1.2.3.4\n")` | Removes trailing newlines or carriage returns from HTTP responses. |
| `growpart` | `growpart /dev/nvme0n1 4` | Extends a partition table entry on a live Linux disk to occupy newly expanded EBS volume space. |
| `lvextend` | `lvextend -r -L +30G /dev/mapper/RootVG-homeVol` | Extends an LVM logical volume and resizes its filesystem (`-r`) online without unmounting. |
| `xfs_growfs` | `xfs_growfs /home` | Expands an XFS filesystem to fill all allocated underlying logical volume blocks. |
| `aws sts get-caller-identity` | `aws sts get-caller-identity` | Queries the AWS STS endpoint to verify the active identity, account ID, and assumed IAM role ARN. |

---

## 6. Official Documentation & References
- **Terraform `aws_security_group_rule` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule)
- **Terraform `aws_iam_role` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role)
- **Terraform `aws_iam_instance_profile` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_instance_profile](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_instance_profile)
- **AWS EC2 IAM Roles & Instance Profiles**: [docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)
- **AWS Systems Manager Parameter Types**: [docs.aws.amazon.com/systems-manager/latest/userguide/sysman-paramstore-types.html](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-paramstore-types.html)

---

## 7. High-Yield Interview Questions & Answers

### Q1: Why should you use standalone `aws_security_group_rule` resources instead of inline rules inside `aws_security_group`?
**Answer**:
1. **Eliminates Cyclic Dependencies**: When microservice A (e.g., `catalogue`) needs ingress from microservice B (e.g., `user`), and microservice B needs ingress from microservice A, inline rules cause Terraform cycle errors. Standalone rule resources allow creating both empty security group containers first, and attaching the directional rules afterwards.
2. **Layer Decoupling**: Security groups can be provisioned in foundation layers (`10-sg`), while individual application teams can define their own application-specific firewall rules in feature layers (`20-sg-rules`).
3. **State Conflict Prevention**: If one team manages security groups and another manages rules, mixing inline and standalone rules will cause Terraform to continually overwrite and remove rules during plans.

---

### Q2: What is an IAM Instance Profile and why is it required for EC2 instances?
**Answer**:
- An **IAM Role** defines permissions and trust policies, but it cannot be directly associated with an EC2 instance.
- An **IAM Instance Profile** is a dedicated AWS container object that holds exactly one IAM Role.
- When you launch an EC2 instance with `iam_instance_profile = aws_iam_instance_profile.bastion.name`, AWS injects temporary STS credentials for that role into the instance's link-local metadata service (IMDSv2 at `169.254.169.254`).
- This enables tools on the instance (e.g., AWS CLI, Terraform, Docker, Python SDK) to authenticate automatically without hardcoding AWS access keys on the server filesystem.

---

### Q3: How do you handle storing and retrieving a list of Subnet IDs in AWS SSM Parameter Store using Terraform?
**Answer**:
- **Storing**: AWS SSM Parameter Store accepts type `StringList`, but Terraform modules export lists as native arrays `["id-1", "id-2"]`. In `parameters.tf`, we format the list into a comma-separated string using `join(",", module.vpc.public_subnet_ids)`.
- **Retrieving**: When querying the SSM parameter in downstream modules via `data.aws_ssm_parameter`, the returned value is a single string `"id-1,id-2"`. We convert it back into an HCL list using `split(",", data.aws_ssm_parameter.public_subnet_ids.value)`. Specific elements can then be accessed via standard array indexing `[0]` or the `element()` function.

---

### Q4: In an EC2 user-data script, why does expanding an EBS volume from 20 GB to 50 GB still show 20 GB available in Linux?
**Answer**:
Because changing the EBS volume size only expands the underlying virtual physical disk block device (`/dev/nvme0n1`). The operating system's partition table and filesystem do not automatically expand:
1. **Partition Level**: Must execute `growpart /dev/nvme0n1 4` to expand partition 4 to fill the new disk boundary.
2. **Logical Volume Level (LVM)**: Must execute `lvextend -r -L +30G /dev/mapper/RootVG-homeVol` to allocate physical extents to the logical volume.
3. **Filesystem Level**: Must execute `xfs_growfs /home` (for XFS) or `resize2fs` (for EXT4) to expand the actual filesystem metadata.

---

## 8. Production Mistakes & Troubleshooting Guide

1. **Cyclic Dependency Loop Between Security Groups**:
   - *Error*: `Error: Cycle: aws_security_group.sg_a, aws_security_group.sg_b`.
   - *Cause*: Declaring mutual ingress rules inside `aws_security_group` inline blocks.
   - *Fix*: Define empty `aws_security_group` resources in `10-sg`, and create separate `aws_security_group_rule` resources in `20-sg-rules`.
2. **Hardcoding IAM Credentials on Bastion or EC2 Servers**:
   - *Security Risk*: Storing `~/.aws/credentials` on an EC2 instance means anyone who compromises the instance or reads an EBS snapshot acquires long-lived credentials.
   - *Best Practice*: Always attach an **IAM Instance Profile**. Temporary credentials are automatically rotated by AWS STS and cannot be stolen from static files.
3. **Forgetting `chomp()` on HTTP Data Sources**:
   - *Error*: CIDR formatting errors when using `data.http.my_ip`.
   - *Cause*: Web servers return IP addresses with trailing newlines (`"1.2.3.4\n"`), resulting in invalid CIDRs like `"1.2.3.4\n/32"`.
   - *Fix*: Always wrap raw HTTP responses in `chomp()`: `"${chomp(data.http.my_ip.response_body)}/32"`.
4. **SSM Parameter Store `StringList` Type Mismatch**:
   - *Error*: `Error: ParameterValue: The parameter value [subnet-1, subnet-2] is not valid.`
   - *Cause*: Passing a raw HCL list into `value` when `type = "StringList"`.
   - *Fix*: Use `join(",", list)` to ensure a clean comma-separated string.

---

## 9. Session Metadata & Timestamps

- **Security Group Ingress Rules & Microsegmentation**: `00:00 - 25:00`
- **SSM Parameter Store StringList & `join`/`split`**: `25:00 - 45:00`
- **IAM Users, Groups, Roles & RBAC Matrix**: `45:00 - 01:10:00`
- **EC2 IAM Role, Instance Profile & Bastion User-Data**: `01:10:00 - 01:30:10`
- **Session Q&A**: `01:30:10`

### Doubts & AI Clarification Link
- [Session 39 AI Clarification Chat](https://chat.z.ai/c/session-39-clarification)