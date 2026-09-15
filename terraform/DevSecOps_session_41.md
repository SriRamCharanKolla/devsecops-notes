# Friday, 13 March 2026
# Session 41 - IAM Roles for MySQL, SSM Password Retrieval, Corporate Analogy for Load Balancers & Application Load Balancer (ALB) Architecture
## Comprehensive Class Notes, Multi-Tier Infrastructure Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [Securing Database Credentials: Least-Privilege IAM for MySQL](#securing-database-credentials-least-privilege-iam-for-mysql)
   - [HCL String Manipulation: `title()` and `join()`](#hcl-string-manipulation-title-and-join)
   - [The Corporate Mental Model for Cloud Architecture](#the-corporate-mental-model-for-cloud-architecture)
     - [Corporate Hierarchy vs AWS Cloud Components](#corporate-hierarchy-vs-aws-cloud-components)
     - [How Load Balancing Mirrors Project Management](#how-load-balancing-mirrors-project-management)
     - [How Auto Scaling Mirrors Human Resources (HR)](#how-auto-scaling-mirrors-human-resources-hr)
   - [Application Load Balancer (ALB) Core Concepts: Listeners, Rules & Target Groups](#application-load-balancer-alb-core-concepts-listeners-rules--target-groups)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct File Links](#directory-structure--direct-file-links)
   - [Architectural Diagrams](#architectural-diagrams)
     - [A. The Corporate Analogy to AWS Cloud Architecture](#a-the-corporate-analogy-to-aws-cloud-architecture)
     - [B. Application Load Balancer Host & Path-Based Routing Mesh](#b-application-load-balancer-host--path-based-routing-mesh)
     - [C. MySQL IAM Instance Profile & SSM Secret Retrieval Flow](#c-mysql-iam-instance-profile--ssm-secret-retrieval-flow)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Layer 5: Database IAM & Policy Rendering (`roboshop-infra-dev/40-databases/`)](#layer-5-database-iam--policy-rendering-roboshop-infra-dev40-databases)
     - [`iam.tf`](#iamtf-in-40-databases)
     - [`mysql-iam-policy.json`](#mysql-iam-policyjson-in-40-databases)
     - [`main.tf`](#maintf-in-40-databases)
   - [Layer 6: Internal Application Load Balancer (`roboshop-infra-dev/50-backend-alb/`)](#layer-6-internal-application-load-balancer-roboshop-infra-dev50-backend-alb)
     - [`main.tf`](#maintf-in-50-backend-alb)
     - [`locals.tf`](#localstf-in-50-backend-alb)
     - [`parameters.tf`](#parameterstf-in-50-backend-alb)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Multi-Layer Automation Pipeline](#1-multi-layer-automation-pipeline)
   - [2. Deploying MySQL with IAM Role for SSM Password Access](#2-deploying-mysql-with-iam-role-for-ssm-password-access)
   - [3. Resolving Python `boto3` / `botocore` Dependency on RHEL 9](#3-resolving-python-boto3--botocore-dependency-on-rhel-9)
   - [4. Creating and Testing the Backend Application Load Balancer](#4-creating-and-testing-the-backend-application-load-balancer)
   - [5. Clean Up (Teardown)](#5-clean-up-teardown)
5. [Commands & CLI Flags Reference Table](#5-commands--cli-flags-reference-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### Securing Database Credentials: Least-Privilege IAM for MySQL
In production systems, database root passwords must **never** be hardcoded in Git repositories, Terraform variables, or plain-text configuration files:
1. The administrative root password is pre-created in AWS Systems Manager Parameter Store under `/roboshop/dev/mysql_root_password` as a `SecureString` or `String`.
2. A dedicated IAM Role (`Roboshop-Dev-Mysql`) is provisioned with an inline/managed policy strictly granting:
   - `ssm:GetParameter` on the exact parameter ARN: `arn:aws:ssm:us-east-1:<ACCOUNT_ID>:parameter/roboshop/${environment}/mysql_root_password`.
   - `ssm:DescribeParameters` on `*`.
3. The IAM Role is attached to the MySQL EC2 instance via an **IAM Instance Profile** (`aws_iam_instance_profile.mysql`).
4. When the Ansible setup playbook runs inside MySQL, Ansible calls AWS SSM using Python to dynamically fetch the root password in memory, sets the MySQL root user credentials, and finishes bootstrapping without writing the secret to disk.

---

### HCL String Manipulation: `title()` and `join()`
AWS IAM roles and policies often require PascalCase formatting (`Roboshop-Dev-Mysql`) while Terraform variables use lowercase (`roboshop`, `dev`, `mysql`):
```hcl
mysql_role_name = join("-", [
  for name in ["${var.project}", "${var.environment}", "mysql"] : title(name)
])
# Result: "Roboshop-Dev-Mysql"

mysql_policy_name = join("", [
  for name in ["${var.project}", "${var.environment}", "mysql"] : title(name)
])
# Result: "RoboshopDevMysql"
```
- `title()`: Converts the first letter of each word to uppercase and the rest to lowercase.
- `join(separator, list)`: Concatenates the string elements of a list using the specified separator.

---

### The Corporate Mental Model for Cloud Architecture

#### Corporate Hierarchy vs AWS Cloud Components
To master cloud architecture, examine how enterprise technology companies operate:

```
+-----------------------------------------------------------------------------------------+
| CORPORATE ORGANIZATION                | AWS CLOUD EQUIVALENT                            |
+---------------------------------------+-------------------------------------------------+
| Delivery Manager (DM) / Team Lead     | Application Load Balancer (ALB)                 |
| Engineering Teams (Frontend, Backend) | Target Groups (e.g. Catalogue TG, User TG)      |
| Individual Team Members (Engineers)   | EC2 Instances / Containers / Pods               |
| Office Reception Desk                 | Load Balancer Listener (Port 80 / Port 443)     |
| Department Mail Routing Rules         | Load Balancer Listener Rules (Host/Path Routing)|
| Daily Standup / Liveness Check        | Target Group Health Checks (`/health` HTTP 200) |
| Work Assignment Algorithm             | Round Robin / Least Outstanding Requests        |
| Human Resources (HR Department)       | Auto Scaling Group (ASG)                        |
| Job Description (JD)                  | Launch Template / Launch Configuration          |
| Resignation / Notice Period           | Scale-In Cooldown / Deregistration Delay        |
+-----------------------------------------------------------------------------------------+
```

---

#### How Load Balancing Mirrors Project Management
1. **The Client Request**: A user submits a query to the company (`https://catalogue.daws88s.online/categories` or `https://user.daws88s.online/submit`).
2. **The Delivery Manager (Load Balancer)**:
   - Evaluates the incoming traffic.
   - Inspects the request path and domain name against predefined **Listener Rules**.
   - Determines which team is responsible for the task:
     - `catalogue.daws88s.online` $\rightarrow$ Forward to **Catalogue Target Group**.
     - `user.daws88s.online` $\rightarrow$ Forward to **User Target Group**.
3. **Health Checking (Daily Standups)**:
   - The Delivery Manager constantly checks: "Is this engineer occupied? Are they available and healthy?"
   - If an engineer (EC2 instance) is overloaded or sick (fails health checks), the Delivery Manager stops assigning tasks to them and routes traffic only to healthy engineers.
4. **Workload Distribution (Round Robin)**:
   - Request 1 $\rightarrow$ Instance 1
   - Request 2 $\rightarrow$ Instance 2
   - Request 3 $\rightarrow$ Instance 3
   - Request 4 $\rightarrow$ Instance 4
   - Request 5 $\rightarrow$ Instance 1 (cycle repeats evenly).

---

#### How Auto Scaling Mirrors Human Resources (HR)
1. **Workload Spike (Overtime)**:
   - Suppose the 4 engineers in the team work 8-hour days, but client demand surges and average workload exceeds 80% capacity (CPU utilization $> 80\%$).
2. **The DM Informs HR (Alarm Triggers ASG)**:
   - The Delivery Manager flags that the team is exhausted.
   - CloudWatch Alarm notifies the Auto Scaling Group (HR).
3. **Hiring Based on Job Description (Launch Template)**:
   - HR doesn't recruit blindly; they refer to a standardized **Job Description (Launch Template)** specifying exact qualifications: AMI ID, Instance Type (`t3.micro`), Security Groups, and User Data bootstrap scripts.
   - HR provisions a brand-new instance and assigns it directly to the **Target Group (Team)**.
4. **Demand Subsides (Scale-In / Notice Period)**:
   - When workload drops, HR gracefully offboards excess capacity.
   - The instance enters **Deregistration Delay (Notice Period)**: It finishes in-flight customer orders before being terminated.

---

### Application Load Balancer (ALB) Core Concepts: Listeners, Rules & Target Groups

1. **Application Load Balancer (ALB)**:
   - Operates at **Layer 7 (Application Layer)** of the OSI model.
   - Can inspect HTTP/HTTPS headers, hostnames, query strings, and paths.
   - Can be **Internet-Facing** (public subnets) or **Internal** (private subnets).
2. **Listener**:
   - A process that checks for connection requests using protocol and port (e.g. HTTP on Port 80, HTTPS on Port 443).
3. **Target Group**:
   - Routes requests to one or more registered targets (EC2 instances, ECS containers, or IP addresses).
   - Manages independent health check configurations per microservice.
4. **Listener Rules**:
   - Priority-ordered conditions (Path pattern `/api/*`, Host header `catalogue.*`) that dictate which Target Group receives the traffic.

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct File Links
Click any file link below to view it directly in your IDE:

#### Project 1: Database Tier & IAM (`roboshop-infra-dev/40-databases`)
- **Directory**: [`roboshop-infra-dev/40-databases/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/40-databases) | [Relative Path](../../roboshop-infra-dev/40-databases)
  - [`40-databases/iam.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/40-databases/iam.tf#L1-L44) | [Relative Link](../../roboshop-infra-dev/40-databases/iam.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/40-databases/iam.tf)
  - [`40-databases/mysql-iam-policy.json`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/40-databases/mysql-iam-policy.json#L1-L17) | [Relative Link](../../roboshop-infra-dev/40-databases/mysql-iam-policy.json) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/40-databases/mysql-iam-policy.json)
  - [`40-databases/main.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/40-databases/main.tf#L79-L92) | [Relative Link](../../roboshop-infra-dev/40-databases/main.tf#L79-L92) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/40-databases/main.tf)
  - [`40-databases/bootstrap.sh`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/40-databases/bootstrap.sh#L1-L12) | [Relative Link](../../roboshop-infra-dev/40-databases/bootstrap.sh)

#### Project 2: Internal Application Load Balancer (`roboshop-infra-dev/50-backend-alb`)
- **Directory**: [`roboshop-infra-dev/50-backend-alb/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb) | [Relative Path](../../roboshop-infra-dev/50-backend-alb)
  - [`50-backend-alb/main.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb/main.tf#L1-L46) | [Relative Link](../../roboshop-infra-dev/50-backend-alb/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/50-backend-alb/main.tf)
  - [`50-backend-alb/data.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb/data.tf#L1-L10) | [Relative Link](../../roboshop-infra-dev/50-backend-alb/data.tf)
  - [`50-backend-alb/locals.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb/locals.tf#L1-L10) | [Relative Link](../../roboshop-infra-dev/50-backend-alb/locals.tf)
  - [`50-backend-alb/parameters.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb/parameters.tf#L1-L7) | [Relative Link](../../roboshop-infra-dev/50-backend-alb/parameters.tf)

---

### Architectural Diagrams

#### A. The Corporate Analogy to AWS Cloud Architecture
```
+───────────────────────────────────────────────────────────────────────────────────+
| DELIVERY MANAGER (ALB - Layer 7)                                                  |
| Checks: Reception Listener (Port 80/443), Routes requests according to Rules       |
+───────────────────────────────────────────────────────────────────────────────────+
                                    │
         ┌──────────────────────────┴──────────────────────────┐
         ▼                                                     ▼
+───────────────────────────────────+ +───────────────────────────────────+
| CATALOGUE TEAM (Target Group 1)   | | USER TEAM (Target Group 2)        |
| - Health Check: GET /health (200) | | - Health Check: GET /health (200) |
|                                   | |                                   |
| [Eng 1]  [Eng 2]  [Eng 3]  [Eng 4]| | [Eng 1]  [Eng 2]  [Eng 3]  [Eng 4]|
| (EC2)    (EC2)    (EC2)    (EC2)  | | (EC2)    (EC2)    (EC2)    (EC2)  |
+───────────────────────────────────+ +───────────────────────────────────+
                  ▲                                     ▲
                  │                                     │
                  └──────────────────┬──────────────────┘
                                     │
                      +─────────────────────────────+
                      | HR DEPARTMENT (Auto Scaling)|
                      | - Hires via Job Description |
                      |   (Launch Template)         |
                      | - Recruits when CPU > 80%   |
                      | - Offboards on cool-down    |
                      +─────────────────────────────+
```

---

#### B. Application Load Balancer Host & Path-Based Routing Mesh
```
                                 Client Requests
                                        │
                                        ▼
                   +─────────────────────────────────────────+
                   | ROUTE53 PRIVATE DNS ALIAS               |
                   | *.backend-alb-dev.aitechapp.fun         |
                   +─────────────────────────────────────────+
                                        │
                                        ▼
                   +─────────────────────────────────────────+
                   | INTERNAL APPLICATION LOAD BALANCER      |
                   | `roboshop-dev` (Private Subnets)        |
                   | Listener: HTTP:80                       |
                   +─────────────────────────────────────────+
                                        │
           ┌────────────────────────────┼────────────────────────────┐
           │ Rule: Host catalogue.*     │ Rule: Host user.*          │ Default Action:
           ▼                            ▼                            ▼
+──────────────────────+     +──────────────────────+     +──────────────────────+
| TARGET GROUP:        |     | TARGET GROUP:        |     | FIXED-RESPONSE:      |
| catalogue (Port 8080)|     | user (Port 8080)     |     | HTTP 200             |
| [EC2 Instance: 1a]   |     | [EC2 Instance: 1a]   |     | "Hi, I am from HTTP  |
| [EC2 Instance: 1b]   |     | [EC2 Instance: 1b]   |     |  Backend ALB"        |
+──────────────────────+     +──────────────────────+     +──────────────────────+
```

---

#### C. MySQL IAM Instance Profile & SSM Secret Retrieval Flow
```
+───────────────────────────────────────────────────────────────────────────────────+
| AWS SYSTEMS MANAGER (SSM) PARAMETER STORE                                         |
| Parameter: `/roboshop/dev/mysql_root_password`                                     |
| Value: "RoboShop@1"                                                               |
+───────────────────────────────────────────────────────────────────────────────────+
                                          ▲
                                          │ 3. Authorized via attached policy
                                          │    Action: ssm:GetParameter
+───────────────────────────────────────────────────────────────────────────────────+
| MYSQL EC2 INSTANCE (Private Subnet)                                               |
|                                                                                   |
|   1. Bound to IAM Instance Profile: `roboshop-dev-mysql`                          |
|                                                                                   |
|   2. Bootstrap Process (`bootstrap.sh` & Ansible):                                |
|      - `dnf install ansible -y`                                                   |
|      - Installs `python3-boto3` & `python3-botocore` for AWS API lookups          |
|      - Ansible query: lookup('aws_ssm', '/roboshop/dev/mysql_root_password')      |
|      - Ingests password in memory & configures MySQL root password!               |
+───────────────────────────────────────────────────────────────────────────────────+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Layer 5: Database IAM & Policy Rendering ([`roboshop-infra-dev/40-databases/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/40-databases) | [Relative](../../roboshop-infra-dev/40-databases))

#### [`iam.tf` in `40-databases/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/40-databases/iam.tf#L1-L44) | [Relative](../../roboshop-infra-dev/40-databases/iam.tf)

```hcl
1: resource "aws_iam_role" "mysql" {
2:   name = local.mysql_role_name # Resolves to "Roboshop-Dev-Mysql"
3: 
4:   assume_role_policy = jsonencode({
5:     Version = "2012-10-17"
6:     Statement = [
7:       {
8:         Action    = "sts:AssumeRole"
9:         Effect    = "Allow"
10:         Principal = {
11:           Service = "ec2.amazonaws.com"
12:         }
13:       },
14:     ]
15:   })
16: }
17: 
18: resource "aws_iam_policy" "mysql" {
19:   name        = local.mysql_policy_name # Resolves to "RoboshopDevMysql"
20:   description = "A policy for Mysql EC2 instance"
21:   policy      = templatefile("mysql-iam-policy.json", {
22:     environment = var.environment
23:   })
24: }
25: 
26: resource "aws_iam_role_policy_attachment" "mysql" {
27:   role       = aws_iam_role.mysql.name
28:   policy_arn = aws_iam_policy.mysql.arn
29: }
30: 
31: resource "aws_iam_instance_profile" "mysql" {
32:   name = "${var.project}-${var.environment}-mysql"
33:   role = aws_iam_role.mysql.name
34: }
```

##### Line-by-Line Breakdown:
- **Lines 1-16 (`aws_iam_role "mysql"`)**: Defines an IAM role named `Roboshop-Dev-Mysql` allowing the EC2 service principal to assume it via STS.
- **Lines 18-24 (`aws_iam_policy "mysql"`)**: Reads the external JSON policy template and replaces `${environment}` with the active environment variable (`dev`).
- **Lines 26-29 (`aws_iam_role_policy_attachment "mysql"`)**: Attaches the rendered policy to the role.
- **Lines 31-34 (`aws_iam_instance_profile "mysql"`)**: Bridges the IAM role into an instance profile for EC2 consumption.

---

#### [`mysql-iam-policy.json` in `40-databases/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/40-databases/mysql-iam-policy.json#L1-L17) | [Relative](../../roboshop-infra-dev/40-databases/mysql-iam-policy.json)

```json
1: {
2:     "Version": "2012-10-17",
3:     "Statement": [
4:         {
5:             "Sid": "VisualEditor0",
6:             "Effect": "Allow",
7:             "Action": "ssm:GetParameter",
8:             "Resource": "arn:aws:ssm:us-east-1:069416262340:parameter/roboshop/${environment}/mysql_root_password"
9:         },
10:         {
11:             "Sid": "VisualEditor1",
12:             "Effect": "Allow",
13:             "Action": "ssm:DescribeParameters",
14:             "Resource": "*"
15:         }
16:     ]
17: }
```
- **Lines 7-8**: Follows strict **Least Privilege** by restricting `ssm:GetParameter` exclusively to the MySQL root password parameter in the target environment.

---

#### [`main.tf` in `40-databases/` (MySQL Instance Attachment)](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/40-databases/main.tf#L79-L92) | [Relative](../../roboshop-infra-dev/40-databases/main.tf#L79-L92)

```hcl
79: resource "aws_instance" "mysql" {
80:   ami                    = local.ami_id
81:   instance_type          = "t3.micro"
82:   subnet_id              = local.database_subnet_id
83:   vpc_security_group_ids = [local.mysql_sg_id]
84:   iam_instance_profile   = aws_iam_instance_profile.mysql.name
85: 
86:   tags = merge(
87:     { Name = "${var.project}-${var.environment}-mysql" },
88:     local.common_tags
89:   )
90: }
```
- **Line 84 (`iam_instance_profile = aws_iam_instance_profile.mysql.name`)**: Injects the IAM instance profile directly into the MySQL EC2 instance at boot time.

---

### Layer 6: Internal Application Load Balancer ([`roboshop-infra-dev/50-backend-alb/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb) | [Relative](../../roboshop-infra-dev/50-backend-alb))

#### [`main.tf` in `50-backend-alb/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb/main.tf#L1-L46) | [Relative](../../roboshop-infra-dev/50-backend-alb/main.tf)

```hcl
1: resource "aws_lb" "backend_alb" {
2:   name               = "${var.project}-${var.environment}" # "roboshop-dev"
3:   internal           = true                                # Internal ALB inside private subnets!
4:   load_balancer_type = "application"
5:   security_groups    = [local.backend_alb_sg_id]
6:   subnets            = local.private_subnet_ids
7: 
8:   enable_deletion_protection = false                       # Set to false for learning teardowns
9: 
10:   tags = merge(
11:     { Name = "${var.project}-${var.environment}" },
12:     local.common_tags
13:   )
14: }
15: 
16: resource "aws_lb_listener" "http" {
17:   load_balancer_arn = aws_lb.backend_alb.arn
18:   port              = "80"
19:   protocol          = "HTTP"
20: 
21:   default_action {
22:     type = "fixed-response"
23: 
24:     fixed_response {
25:       content_type = "text/html"
26:       message_body = "<h1>Hi, I am from HTTP Backend ALB</h1>"
27:       status_code  = "200"
28:     }
29:   }
30: }
31: 
32: resource "aws_route53_record" "www" {
33:   zone_id = var.zone_id
34:   name    = "*.backend-alb-${var.environment}.${var.domain_name}"
35:   type    = "A"
36: 
37:   alias {
38:     name                   = aws_lb.backend_alb.dns_name
39:     zone_id                = aws_lb.backend_alb.zone_id
40:     evaluate_target_health = true
41:   }
42: }
```

##### Line-by-Line Breakdown:
- **Lines 1-6 (`aws_lb "backend_alb"`)**:
  - `internal = true`: Places the ALB strictly inside private VPC subnets. It does **not** receive a public IP.
  - `load_balancer_type = "application"`: Configures a Layer 7 Application Load Balancer.
  - `subnets = local.private_subnet_ids`: Spans multiple Availability Zones across private subnets for high availability.
- **Lines 16-30 (`aws_lb_listener "http"`)**:
  - Listens on Port 80.
  - `fixed-response`: Returns a static HTTP 200 test page before target groups are attached in downstream component modules.
- **Lines 32-42 (`aws_route53_record "www"`)**:
  - Creates a **Wildcard DNS Alias** `*.backend-alb-dev.aitechapp.fun`.
  - Any microservice URL matching this pattern (e.g. `catalogue-dev.backend-alb-dev.aitechapp.fun`, `user-dev.backend-alb-dev.aitechapp.fun`) automatically resolves directly to the internal ALB!

---

#### [`parameters.tf` in `50-backend-alb/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb/parameters.tf#L1-L7) | [Relative](../../roboshop-infra-dev/50-backend-alb/parameters.tf)

```hcl
1: resource "aws_ssm_parameter" "backend_alb_listener_arn" {
2:   name  = "/${var.project}/${var.environment}/backend_alb_listener_arn"
3:   type  = "String"
4:   value = aws_lb_listener.http.arn
5: }
```
- Exports the listener ARN to SSM so downstream microservice layers (`60-catalogue`, `90-components`) can attach listener rules dynamically!

---

## 4. Step-by-Step Hands-on Execution Walkthrough

All commands below are directly copyable:

### 1. Multi-Layer Automation Pipeline
```bash
# Run deployment across prerequisite foundation layers
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev

for i in 00-vpc/ 10-sg/ 20-sg-rules/ 30-bastion/; do
  echo "========================================="
  echo "🚀 Deploying: $i"
  echo "========================================="
  cd $i || exit 1
  [ ! -d ".terraform" ] && terraform init
  terraform apply -auto-approve || exit 1
  cd ..
done
```

---

### 2. Deploying MySQL with IAM Role for SSM Password Access
```bash
# Connect to Bastion server
ssh ec2-user@<BASTION_PUBLIC_IP>

# Navigate to databases layer
cd /home/ec2-user/roboshop-infra-dev/40-databases

# Pull latest changes and apply
git pull
terraform init
terraform apply -auto-approve
```

---

### 3. Resolving Python `boto3` / `botocore` Dependency on RHEL 9
When Ansible executes the MySQL setup role and attempts to query AWS SSM for the root password, it requires Python AWS SDK libraries on the target MySQL instance:

```bash
# If Ansible throws: "botocore and boto3 are required for this module"
# Ensure the bootstrap.sh script or Ansible role includes:
sudo dnf install -y python3-pip
sudo pip3 install boto3 botocore

# Ensure bootstrap.sh has `git pull` so latest playbooks are pulled:
cat << 'EOF' > bootstrap.sh
#!/bin/bash
component=$1
environment=$2
dnf install ansible -y
cd /home/ec2-user
git clone https://github.com/SriRamCharanKolla/ansible-roboshop-roles-tf.git
cd ansible-roboshop-roles-tf
git pull
ansible-playbook -e component=$component -e env=$environment roboshop.yaml
EOF
```

---

### 4. Creating and Testing the Backend Application Load Balancer
```bash
# Deploy 50-backend-alb layer
cd /home/ec2-user/roboshop-infra-dev/50-backend-alb

terraform init
terraform apply -auto-approve

# Test Backend ALB from inside Bastion:
# Note: Because the ALB is internal, curl MUST be executed from inside the VPC!
curl -i http://roboshop-dev-1234567890.us-east-1.elb.amazonaws.com

# Or using the Route53 wildcard DNS alias:
curl -i http://test.backend-alb-dev.aitechapp.fun

# Expected HTTP Response:
# HTTP/1.1 200 OK
# Content-Type: text/html
# Content-Length: 42
# <h1>Hi, I am from HTTP Backend ALB</h1>
```

---

### 5. Clean Up (Teardown)
> [!IMPORTANT]
> Destroy in reverse order: `50-backend-alb` $\rightarrow$ `40-databases` $\rightarrow$ `30-bastion` $\rightarrow$ `20-sg-rules` $\rightarrow$ `10-sg` $\rightarrow$ `00-vpc`!

```bash
# From Bastion host:
cd /home/ec2-user/roboshop-infra-dev/50-backend-alb
terraform destroy -auto-approve

cd /home/ec2-user/roboshop-infra-dev/40-databases
terraform destroy -auto-approve
```

---

## 5. Commands & CLI Flags Reference Table

| Command / Tool | Syntax Example | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `aws elbv2 describe-load-balancers` | `aws elbv2 describe-load-balancers --names roboshop-dev` | Queries status, DNS name, ARN, and scheme of the created ALB. |
| `aws elbv2 describe-target-groups` | `aws elbv2 describe-target-groups` | Lists all registered target groups and health check parameters. |
| `aws elbv2 describe-listeners` | `aws elbv2 describe-listeners --load-balancer-arn <ARN>` | Inspects ports and default actions configured on ALB listeners. |
| `templatefile()` | `templatefile("policy.json", { env = "dev" })` | Interpolates external JSON files with dynamic Terraform expressions. |
| `fixed-response` | `type = "fixed-response"` | Allows an ALB to return an HTTP status code and message without backend targets. |
| `alias` block in Route53 | `alias { name = aws_lb.alb.dns_name ... }` | Creates AWS-native Route53 Alias records with zero DNS query fees. |

---

## 6. Official Documentation & References
- **AWS Application Load Balancers**: [docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
- **Terraform `aws_lb` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb)
- **Terraform `aws_lb_listener` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener)
- **AWS SSM Parameter Store Ansible Lookup**: [docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ssm_lookup.html](https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ssm_lookup.html)

---

## 7. High-Yield Interview Questions & Answers

### Q1: What is the architectural difference between an Application Load Balancer (ALB) and a Network Load Balancer (NLB)?
**Answer**:
- **Application Load Balancer (Layer 7)**:
  - Inspects HTTP/HTTPS application-layer protocols.
  - Supports advanced routing rules: Host-based (`catalogue.domain.com`), Path-based (`/api/*`), HTTP headers, and query strings.
  - Terminates TLS connections and supports native integration with AWS WAF and Cognito authentication.
- **Network Load Balancer (Layer 4)**:
  - Operates at the transport layer (TCP/UDP/TLS).
  - Designed for ultra-high throughput, millions of requests per second, and ultra-low latency (sub-millisecond).
  - Preserves source client IP addresses natively and supports static Elastic IP addresses per Availability Zone.

---

### Q2: What is the purpose of an ALB "Fixed-Response" Action?
**Answer**:
A fixed-response action enables the Application Load Balancer to respond directly to incoming client requests with an HTTP status code (e.g. 200, 403, 503) and an optional message body **without forwarding the request to any backend target group**.
- **Use Cases**:
  - Maintenance Mode: Returning a 503 "Under Maintenance" HTML page during deployments.
  - Access Control: Returning 403 Forbidden for unauthorized geographic locations or paths.
  - Health/Default Fallback: Returning a baseline HTTP 200 for listener verification before application microservices are registered.

---

### Q3: What is Target Group "Deregistration Delay" (Connection Draining)?
**Answer**:
When an EC2 instance in a target group is being deregistered or scaled in by an Auto Scaling Group, deregistration delay gives the instance time to complete in-flight requests:
- The ALB immediately stops sending **new** connections to the instance.
- The ALB keeps existing active connections open for a configurable duration (default: 300 seconds).
- Once all in-flight connections close or the timeout expires, the instance is safely terminated without returning HTTP 502/504 errors to end users.

---

### Q4: Why should internal microservice load balancers use `internal = true`?
**Answer**:
- Setting `internal = true` instructs AWS to allocate private IP addresses only (from the assigned private subnets) to the load balancer's Elastic Network Interfaces (ENIs).
- The ALB receives an AWS private DNS name that resolves only to private IPs inside the VPC (and peered VPCs).
- It prevents backend databases, catalogue services, and payment microservices from being exposed to the public internet, enforcing network-level isolation.

---

## 8. Production Mistakes & Troubleshooting Guide

### 1. Ansible Missing `botocore` & `boto3` on RHEL 9
- **Problem**: When Ansible runs `lookup('amazon.aws.aws_ssm', ...)`, it fails with `Python module botocore or boto3 is required`.
- **Cause**: Minimal Linux server AMIs do not ship with Python AWS SDK libraries pre-installed.
- **Fix**: In `bootstrap.sh`, install `python3-pip` and run `pip3 install boto3 botocore` before executing `ansible-playbook`.

---

### 2. Missing `git pull` in `bootstrap.sh`
- **Problem**: Changing Ansible role logic in GitHub does not reflect on newly created database instances.
- **Cause**: If `bootstrap.sh` clones into an existing directory without `git pull`, stale local playbooks are executed.
- **Fix**: Add `git pull` immediately after changing directory into the cloned repo in `bootstrap.sh`.

---

### 3. Placing Internal ALB in Public Subnets
- **Problem**: Creating an internal ALB inside public subnets causes confusion and breaks multi-tier isolation.
- **Fix**: Always assign `subnets = local.private_subnet_ids` for internal ALBs, and reserve `public_subnet_ids` solely for external ingress ALBs (`80-frontend-alb`).

---

### 4. Health Check Endpoint Returning HTTP 404
- **Problem**: Instances in target group remain in `Unhealthy` state indefinitely, causing ALB to return `HTTP 502 Bad Gateway`.
- **Cause**: Target group health check path is defaulted to `/`, but the application microservice only responds on `/health`.
- **Fix**: Explicitly specify `health_check { path = "/health", matcher = "200" }` matching the microservice's exact health endpoint.

---

## 9. Session Metadata & Timestamps

- **MySQL IAM User & Policy Setup**: `00:00 - 30:00`
- **Class Assignment Review**: `01:01:47`
- **Application Load Balancer Architecture & Analogy**: `01:18:24`
- **Session Q&A**: `01:27:00`

### Doubts & AI Clarification Link
- [Session 41 AI Clarification Chat](https://chat.z.ai/c/session-41-clarification)