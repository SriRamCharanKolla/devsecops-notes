# Monday, 9 March 2026
# Session 40 - Database Provisioning, Terraform-Ansible Integration, `terraform_data` Provisioners & Route53 Private DNS
## Comprehensive Class Notes, Multi-Tier Infrastructure Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [The Three Fundamental DevOps Cloud Access Pillars](#the-three-fundamental-devops-cloud-access-pillars)
   - [Database Provisioning Workflow](#database-provisioning-workflow)
   - [Resource Lifecycle Evolution: `null_resource` vs Modern `terraform_data`](#resource-lifecycle-evolution-null_resource-vs-modern-terraform_data)
   - [Architectural Patterns: Orchestrating Ansible with Terraform](#architectural-patterns-orchestrating-ansible-with-terraform)
     - [Option A: Remote Execution from Bastion](#option-a-remote-execution-from-bastion)
     - [Option B: Self-Bootstrapping via Localhost Execution (Selected Pattern)](#option-b-self-bootstrapping-via-localhost-execution-selected-pattern)
   - [Deep Architectural Comparison: `user_data` vs Provisioners (`remote-exec`)](#deep-architectural-comparison-user_data-vs-provisioners-remote-exec)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct File Links](#directory-structure--direct-file-links)
   - [Architectural Diagrams](#architectural-diagrams)
     - [A. Bastion-to-Database Multi-Tier Provisioning Flow](#a-bastion-to-database-multi-tier-provisioning-flow)
     - [B. `terraform_data` Provisioner & Ansible Bootstrap Lifecycle](#b-terraform_data-provisioner--ansible-bootstrap-lifecycle)
     - [C. Private DNS Resolution Architecture via Route53](#c-private-dns-resolution-architecture-via-route53)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Layer 5: Databases Infrastructure (`roboshop-infra-dev/40-databases/`)](#layer-5-databases-infrastructure-roboshop-infra-dev40-databases)
     - [`bootstrap.sh`](#bootstrapsh-in-40-databases)
     - [`data.tf`](#datatf-in-40-databases)
     - [`locals.tf`](#localstf-in-40-databases)
     - [`main.tf`](#maintf-in-40-databases)
     - [`iam.tf`](#iamtf-in-40-databases)
     - [`r53.tf`](#r53tf-in-40-databases)
     - [`parameter.tf`](#parametertf-in-40-databases)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Robust Multi-Layer Deployment Script with Error Trapping](#1-robust-multi-layer-deployment-script-with-error-trapping)
   - [2. Deploying `40-databases` from Bastion Host](#2-deploying-40-databases-from-bastion-host)
   - [3. Verifying Database Listening Ports via SSH](#3-verifying-database-listening-ports-via-ssh)
   - [4. Validating Route53 Private DNS Resolution](#4-validating-route53-private-dns-resolution)
   - [5. Clean Up (Teardown Script)](#5-clean-up-teardown-script)
5. [Commands & CLI Flags Reference Table](#5-commands--cli-flags-reference-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### The Three Fundamental DevOps Cloud Access Pillars
When automating production-grade cloud environments, an orchestration engine (like the Bastion server or CI/CD agent) requires 3 core permissions:
1. **S3 State Access**: Permissions to read remote state files, create new locks, and update state representations (`s3:GetObject`, `s3:PutObject`, `s3:ListBucket`).
2. **EC2 Lifecycle Access**: Permissions to launch instances, query AMIs, attach EBS volumes, and configure network interfaces (`ec2:RunInstances`, `ec2:Describe*`, `ec2:TerminateInstances`).
3. **Route53 DNS Management**: Permissions to create, update, and delete private DNS hosted zone records for dynamic service discovery (`route53:ChangeResourceRecordSets`, `route53:ListHostedZones`).

---

### Database Provisioning Workflow
Provisioning stateful backend data stores involves a synchronized 2-step pipeline:
1. **Infrastructure Provisioning (Terraform)**:
   - Allocates the EC2 instance inside the private database subnet (`10.0.21.0/24`).
   - Attaches the dedicated Security Group (`roboshop-dev-mongodb`, `roboshop-dev-redis`, etc.).
   - Connects to the instance and initiates configuration.
2. **DNS & Service Discovery (Route53)**:
   - Obtains the dynamic private IP assigned by AWS (`aws_instance.mongodb.private_ip`).
   - Creates a Route53 private record: `mongodb-dev.aitechapp.fun` $\rightarrow$ `10.0.21.x`.
   - Microservices in the VPC can now connect using human-readable hostnames instead of volatile IP addresses.

---

### Resource Lifecycle Evolution: `null_resource` vs Modern `terraform_data`

| Feature | Legacy `null_resource` | Modern `terraform_data` (Terraform $\ge$ 1.4) |
| :--- | :--- | :--- |
| **Provider Dependency** | Requires external `hashicorp/null` provider plugin. | **Built into Terraform Core** (Zero external provider downloads). |
| **Lifecycle Semantics** | Deviates from standard resource lifecycle. | Follows standard resource lifecycle (plan, apply, destroy). |
| **Trigger Mechanism** | `triggers = { id = ... }` (Map of strings only). | `triggers_replace = [ ... ]` (Accepts any data type / expressions). |
| **State Storage** | Stores arbitrary string keys. | Can store arbitrary computed values in state without infrastructure. |
| **Primary Use Case** | Running provisioners (`local-exec`, `remote-exec`). | Running provisioners and orchestrating replacements cleanly. |

`terraform_data` does not create any physical cloud infrastructure in AWS. Instead, it acts as a lifecycle coordinator inside Terraform to trigger actions whenever tracked upstream attributes change (such as an EC2 instance ID changing after recreation).

---

### Architectural Patterns: Orchestrating Ansible with Terraform

#### Option A: Remote Execution from Bastion
- Install Ansible on the Bastion server.
- Bastion manages an SSH inventory pointing to private database IPs.
- *Downsides*: Requires complex SSH key management across servers, maintains state on Bastion, susceptible to network blips during playbook execution.

#### Option B: Self-Bootstrapping via Localhost Execution (Selected Pattern)
- Terraform provisions the instance and pushes a minimal bootstrap script (`bootstrap.sh`) via the `file` provisioner.
- Terraform executes the script on the target instance via `remote-exec`.
- The instance installs Ansible locally (`dnf install ansible -y`), clones the centralized playbook repository (`ansible-roboshop-roles-tf`), and executes against `localhost`:
  ```bash
  ansible-playbook -e component=$component -e env=$environment roboshop.yaml
  ```
- *Benefits*: Completely self-contained, zero network SSH inventory complexity, scales effortlessly.

---

### Deep Architectural Comparison: `user_data` vs Provisioners (`remote-exec`)

```
+-----------------------------------------------------------------------------------------+
| FEATURE                 | USER_DATA (Cloud-Init)          | PROVISIONERS (remote-exec)  |
+-------------------------+---------------------------------+-----------------------------+
| Execution Timing        | Async at EC2 OS boot time       | Synchronous during apply    |
| Terraform Awareness     | Blind (Terraform doesn't know)  | Aware (Direct pipeline lock)|
| On Failure Behavior     | Terraform reports SUCCESS       | Terraform reports FAILURE   |
| Terminal Logs           | Hidden in /var/log/cloud-init   | Live streamed to terminal   |
| Network Dependency      | Outbound internet (for packages)| Inbound SSH:22 from caller  |
| Ideal Use Case          | Simple agent installs, mount disk| Orchestrated multi-tier apps|
+-----------------------------------------------------------------------------------------+
```

1. **`user_data`**:
   - Managed entirely by AWS EC2 and Linux `cloud-init`.
   - If a command inside `user_data` crashes (e.g. `dnf install` fails), AWS still reports the instance as healthy, and Terraform exits with **Success**.
   - Debugging requires logging into the server and inspecting `/var/log/cloud-init-output.log`.
2. **Provisioners (`remote-exec`)**:
   - Managed directly by the Terraform CLI over an SSH connection.
   - If a command inside `remote-exec` fails (e.g., Ansible playbook syntax error), **Terraform immediately fails the apply** and marks the resource as tainted.
   - You get immediate real-time console feedback, allowing CI/CD pipelines to halt on configuration errors.

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct File Links
Click any file link below to view it directly in your IDE:

#### Project: Databases & Configuration Management (`roboshop-infra-dev/40-databases`)
- **Directory**: [roboshop-infra-dev/40-databases/](../../roboshop-infra-dev/40-databases)
  - [40-databases/provider.tf](../../roboshop-infra-dev/40-databases/provider.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/40-databases/provider.tf)
  - [40-databases/main.tf](../../roboshop-infra-dev/40-databases/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/40-databases/main.tf)
  - [40-databases/bootstrap.sh](../../roboshop-infra-dev/40-databases/bootstrap.sh) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/40-databases/bootstrap.sh)
  - [40-databases/data.tf](../../roboshop-infra-dev/40-databases/data.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/40-databases/data.tf)
  - [40-databases/locals.tf](../../roboshop-infra-dev/40-databases/locals.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/40-databases/locals.tf)
  - [40-databases/iam.tf](../../roboshop-infra-dev/40-databases/iam.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/40-databases/iam.tf)
  - [40-databases/r53.tf](../../roboshop-infra-dev/40-databases/r53.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/40-databases/r53.tf)
  - [40-databases/parameter.tf](../../roboshop-infra-dev/40-databases/parameter.tf)
  - [40-databases/variables.tf](../../roboshop-infra-dev/40-databases/variables.tf)
  - [40-databases/mysql-iam-policy.json](../../roboshop-infra-dev/40-databases/mysql-iam-policy.json)

---

### Architectural Diagrams

#### A. Bastion-to-Database Multi-Tier Provisioning Flow
```
+-----------------------------------------------------------------------------------+
| ROBOSHOP DEV VPC (10.0.0.0/16)                                                    |
|                                                                                   |
|   +───────────────────────────────────+                                           |
|   | PUBLIC SUBNET (10.0.1.0/24)       |                                           |
|   |                                   |                                           |
|   |  BASTION HOST                     |                                           |
|   |  - Terraform CLI installed        |                                           |
|   |  - Assumes IAM Role               |                                           |
|   |  - Executes `40-databases` apply  |                                           |
|   +───────────────────────────────────+                                           |
|                     │                                                             |
|                     │ 1. SSH:22 to Private IP via Security Group Chaining        |
|                     │ 2. Transfers `bootstrap.sh` via file provisioner            |
|                     │ 3. Executes `remote-exec` triggers                          |
|                     ▼                                                             |
|   +───────────────────────────────────────────────────────────────────────────+   |
|   | DATABASE SUBNET (10.0.21.0/24)                                            |   |
|   |                                                                           |   |
|   |  +--------------------+  +--------------------+  +--------------------+   |   |
|   |  | MONGODB (27017)    |  | REDIS (6379)       |  | MYSQL (3306)       |   |   |
|   |  | Private: 10.0.21.10|  | Private: 10.0.21.20|  | Private: 10.0.21.30|   |   |
|   |  +--------------------+  +--------------------+  +--------------------+   |   |
|   |                                                                           |   |
|   |  +--------------------+                                                   |   |
|   |  | RABBITMQ (5672)    |                                                   |   |
|   |  | Private: 10.0.21.40|                                                   |   |
|   |  +--------------------+                                                   |   |
|   +───────────────────────────────────────────────────────────────────────────+   |
+-----------------------------------------------------------------------------------+
```

---

#### B. `terraform_data` Provisioner & Ansible Bootstrap Lifecycle
```
+---------------------------------------------------------------------------------------+
| 1. Terraform launches `aws_instance.mongodb`                                          |
|    - Assigns AMI, Subnet, and Security Group                                          |
+---------------------------------------------------------------------------------------+
                                          │
                                          ▼
+---------------------------------------------------------------------------------------+
| 2. `terraform_data.mongodb` triggers on `triggers_replace = [aws_instance.mongodb.id]`|
|    - Establishes SSH connection to `aws_instance.mongodb.private_ip`                   |
+---------------------------------------------------------------------------------------+
                                          │
                                          ▼
+---------------------------------------------------------------------------------------+
| 3. `provisioner "file"` uploads `bootstrap.sh`                                        |
|    - Source: Local `bootstrap.sh`                                                     |
|    - Destination: `/tmp/bootstrap.sh` on target VM                                    |
+---------------------------------------------------------------------------------------+
                                          │
                                          ▼
+---------------------------------------------------------------------------------------+
| 4. `provisioner "remote-exec"` executes:                                              |
|    - `chmod +x /tmp/bootstrap.sh`                                                     |
|    - `sudo sh /tmp/bootstrap.sh mongodb dev`                                          |
+---------------------------------------------------------------------------------------+
                                          │
                                          ▼
+---------------------------------------------------------------------------------------+
| 5. Inside Target VM (`bootstrap.sh` execution):                                       |
|    - `dnf install ansible -y`                                                         |
|    - `git clone https://github.com/.../ansible-roboshop-roles-tf.git`                 |
|    - `ansible-playbook -e component=mongodb -e env=dev roboshop.yaml` (on localhost)  |
|    - Service starts on Port 27017!                                                    |
+---------------------------------------------------------------------------------------+
```

---

#### C. Private DNS Resolution Architecture via Route53
```
+───────────────────────────────────────────────────────────────────────────────────+
| ROUTE53 PRIVATE HOSTED ZONE ("aitechapp.fun")                                     |
|                                                                                   |
|   Record Name                              Type    Value (Target)                 |
|   ---------------------------------------  ------  ----------------------------   |
|   mongodb-dev.aitechapp.fun                A       aws_instance.mongodb.private_ip|
|   redis-dev.aitechapp.fun                  A       aws_instance.redis.private_ip  |
|   mysql-dev.aitechapp.fun                  A       aws_instance.mysql.private_ip  |
|   rabbitmq-dev.aitechapp.fun               A       aws_instance.rabbitmq.private_ip|
+───────────────────────────────────────────────────────────────────────────────────+
                                          ▲
                                          │ Private DNS queries resolved inside VPC
                                          │
+───────────────────────────────────────────────────────────────────────────────────+
| BACKEND APPLICATION TIER (Catalogue, User, Cart, Shipping, Payment)               |
| - Connects directly to `mongodb-dev.aitechapp.fun:27017`                          |
| - Zero hardcoded IP addresses in microservice config files!                       |
+───────────────────────────────────────────────────────────────────────────────────+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Layer 5: Databases Infrastructure (40-databases)
> **Layer Directory**: [roboshop-infra-dev/40-databases/](../../roboshop-infra-dev/40-databases)

#### bootstrap.sh in 40-databases/
> **File**: [roboshop-infra-dev/40-databases/bootstrap.sh](../../roboshop-infra-dev/40-databases/bootstrap.sh)

```bash
1: #!/bin/bash
2: 
3: component=$1
4: environment=$2
5: dnf install ansible -y
6: 
7: cd /home/ec2-user
8: git clone https://github.com/SriRamCharanKolla/ansible-roboshop-roles-tf.git
9: 
10: cd ansible-roboshop-roles-tf
11: git pull
12: ansible-playbook -e component=$component -e env=$environment roboshop.yaml
```

##### Line-by-Line Breakdown:
- **Lines 3-4 (`component=$1`, `environment=$2`)**: Ingests command line positional arguments passed from Terraform's `remote-exec` (e.g. `mongodb` and `dev`).
- **Line 5 (`dnf install ansible -y`)**: Installs the Ansible automation engine directly on the RHEL 9 instance.
- **Lines 7-11 (`git clone`, `git pull`)**: Pulls the centralized, version-controlled Ansible roles repository.
- **Line 12 (`ansible-playbook`)**: Runs the Ansible playbook against `localhost` with extra vars (`-e`), configuring the database server without requiring an external Ansible control machine!

---

#### data.tf in 40-databases/
> **File**: [roboshop-infra-dev/40-databases/data.tf](../../roboshop-infra-dev/40-databases/data.tf)

```hcl
1: data "aws_ami" "joindevops" {
2:   most_recent = true
3:   owners      = ["973714476881"]
4: 
5:   filter {
6:     name   = "name"
7:     values = ["Redhat-9-DevOps-Practice"]
8:   }
9:   ...
19: }
20: 
21: data "aws_ssm_parameter" "database_subnet_ids" {
22:   name = "/${var.project}/${var.environment}/database_subnet_ids"
23: }
24: 
25: data "aws_ssm_parameter" "mongodb_sg_id" {
26:   name = "/${var.project}/${var.environment}/mongodb_sg_id"
27: }
...
37: data "aws_ssm_parameter" "rabbitmq_sg_id" {
38:   name = "/${var.project}/${var.environment}/rabbitmq_sg_id"
39: }
```
- Completely decouples database creation from the upstream VPC and SG layers:
  - Fetches the `database_subnet_ids` parameter published by `00-vpc`.
  - Queries all 4 database Security Group IDs (`mongodb_sg_id`, `redis_sg_id`, `mysql_sg_id`, `rabbitmq_sg_id`) published by `10-sg`.

---

#### locals.tf in 40-databases/
> **File**: [roboshop-infra-dev/40-databases/locals.tf](../../roboshop-infra-dev/40-databases/locals.tf)

```hcl
1: locals {
2:     ami_id = data.aws_ami.joindevops.id
3:     common_tags = {
4:         Project     = var.project
5:         Environment = var.environment
6:         Terraform   = "true"
7:     }
8:     # database subnet in 1a AZ
9:     database_subnet_id = split(",", data.aws_ssm_parameter.database_subnet_ids.value)[0]
10:     mongodb_sg_id      = data.aws_ssm_parameter.mongodb_sg_id.value
11:     redis_sg_id        = data.aws_ssm_parameter.redis_sg_id.value
12:     mysql_sg_id        = data.aws_ssm_parameter.mysql_sg_id.value
13:     rabbitmq_sg_id     = data.aws_ssm_parameter.rabbitmq_sg_id.value
14:     mysql_role_name = join("-", [
15:             for name in ["${var.project}","${var.environment}", "mysql"] : title(name)
16:         ])
17:     mysql_policy_name = join("", [
18:             for name in ["${var.project}","${var.environment}", "mysql"] : title(name)
19:         ])
20: }
```

##### Line-by-Line Breakdown:
- **Line 9 (`database_subnet_id`)**: Converts the SSM `StringList` into a list using `split()` and extracts index `0` (`database-us-east-1a`).
- **Lines 14-16 (`mysql_role_name`)**: Uses a `for` expression with `title()` to format `"Roboshop-Dev-Mysql"`.
- **Lines 17-19 (`mysql_policy_name`)**: Generates `"RoboshopDevMysql"` without dashes for IAM policy naming conventions.

---

#### main.tf in 40-databases/
> **File**: [roboshop-infra-dev/40-databases/main.tf](../../roboshop-infra-dev/40-databases/main.tf)

```hcl
1: resource "aws_instance" "mongodb" {
2:   ami                    = local.ami_id
3:   instance_type          = "t3.micro"
4:   subnet_id              = local.database_subnet_id
5:   vpc_security_group_ids = [local.mongodb_sg_id]
6: 
7:   tags = merge(
8:     { Name = "${var.project}-${var.environment}-mongodb" },
9:     local.common_tags
10:   )
11: }
12: 
13: resource "terraform_data" "mongodb" {
14:   triggers_replace = [
15:     aws_instance.mongodb.id
16:   ]
17: 
18:   connection {
19:     type     = "ssh"
20:     user     = "ec2-user"
21:     password = "DevOps321"
22:     host     = aws_instance.mongodb.private_ip
23:   }
24: 
25:   provisioner "file" {
26:     source      = "bootstrap.sh"
27:     destination = "/tmp/bootstrap.sh"
28:   }
29: 
30:   provisioner "remote-exec" {
31:     inline = [
32:       "chmod +x /tmp/bootstrap.sh",
33:       "sudo sh /tmp/bootstrap.sh mongodb ${var.environment}"
34:     ]
35:   }
36: }
```

##### Line-by-Line Breakdown:
- **Lines 1-11 (`aws_instance "mongodb"`)**: Deploys the MongoDB EC2 instance into the private database subnet with its dedicated security group.
- **Lines 13-16 (`resource "terraform_data" "mongodb"`)**: Uses the built-in `terraform_data` resource. `triggers_replace = [aws_instance.mongodb.id]` guarantees that if the EC2 instance is recreated, Terraform automatically re-runs the provisioner!
- **Lines 18-23 (`connection`)**: Configures the SSH connection parameters over the private network.
- **Lines 25-28 (`provisioner "file"`)**: Transfers the local `bootstrap.sh` script to the remote machine's `/tmp` directory.
- **Lines 30-35 (`provisioner "remote-exec"`)**: Grants execute permissions and runs the bootstrap script with `mongodb` and `dev` arguments.

---

#### iam.tf in 40-databases/
> **File**: [roboshop-infra-dev/40-databases/iam.tf](../../roboshop-infra-dev/40-databases/iam.tf)

```hcl
1: resource "aws_iam_role" "mysql" {
2:   name = local.mysql_role_name
3: 
4:   assume_role_policy = jsonencode({
5:     Version = "2012-10-17"
6:     Statement = [
7:       {
8:         Action    = "sts:AssumeRole"
9:         Effect    = "Allow"
10:         Principal = { Service = "ec2.amazonaws.com" }
11:       },
12:     ]
13:   })
14: }
15: 
16: resource "aws_iam_policy" "mysql" {
17:   name        = local.mysql_policy_name
18:   description = "A policy for Mysql EC2 instance"
19:   policy      = templatefile("mysql-iam-policy.json", {
20:     environment = var.environment
21:   })
22: }
23: 
24: resource "aws_iam_role_policy_attachment" "mysql" {
25:   role       = aws_iam_role.mysql.name
26:   policy_arn = aws_iam_policy.mysql.arn
27: }
28: 
29: resource "aws_iam_instance_profile" "mysql" {
30:   name = "${var.project}-${var.environment}-mysql"
31:   role = aws_iam_role.mysql.name
32: }
```
- Demonstrates parameterized policy creation using `templatefile("mysql-iam-policy.json", { environment = var.environment })`.
- Provides the MySQL instance with permission to read SSM parameters and secrets without hardcoding credentials!

---

#### r53.tf in 40-databases/
> **File**: [roboshop-infra-dev/40-databases/r53.tf](../../roboshop-infra-dev/40-databases/r53.tf)

```hcl
1: resource "aws_route53_record" "mongodb" {
2:   zone_id         = var.zone_id
3:   name            = "mongodb-${var.environment}.${var.domain_name}"
4:   type            = "A"
5:   ttl             = "1"
6:   records         = [aws_instance.mongodb.private_ip]
7:   allow_overwrite = true
8: }
```
- **Line 3 (`name = "mongodb-${var.environment}.${var.domain_name}"`)**: Establishes standard environment-aware private DNS record (e.g., `mongodb-dev.aitechapp.fun`).
- **Line 5 (`ttl = "1"`)**: Configures 1-second TTL for immediate failover and cache busting during development cycles.
- **Line 6 (`records = [aws_instance.mongodb.private_ip]`)**: Maps directly to the instance's private IP address.
- **Line 7 (`allow_overwrite = true`)**: Prevents Terraform collision errors if a stale DNS record already exists in the zone.

---

## 4. Step-by-Step Hands-on Execution Walkthrough

All commands below are directly copyable:

### 1. Robust Multi-Layer Deployment Script with Error Trapping
To prevent manual directory switching and capture errors immediately:

```bash
# Execute multi-layer apply loop from roboshop-infra-dev root:
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev

for i in 00-vpc/ 10-sg/ 20-sg-rules/ 30-bastion/; do
  echo "========================================="
  echo "🚀 Deploying Layer: $i"
  echo "========================================="

  cd $i || exit 1

  # Initialize only if .terraform directory is absent
  if [ ! -d ".terraform" ]; then
    terraform init
  fi

  terraform apply -auto-approve
  if [ $? -ne 0 ]; then
    echo "❌ Error occurred in layer: $i"
    exit 1
  fi

  cd ..
done
```

---

### 2. Deploying `40-databases` from Bastion Host
```bash
# 1. Connect to Bastion Server using its Public IP
ssh ec2-user@<BASTION_PUBLIC_IP>

# 2. Inside Bastion: Navigate to home directory and clone repo
cd /home/ec2-user
git clone https://github.com/SriRamCharanKolla/roboshop-infra-dev.git
cd roboshop-infra-dev/40-databases

# 3. Initialize and apply database layer
terraform init
terraform plan
terraform apply -auto-approve

# Watch live output as Terraform provisions instances, uploads bootstrap.sh,
# installs Ansible, and runs playbooks against localhost in real time!
```

---

### 3. Verifying Database Listening Ports via SSH
From inside the Bastion host, connect to each database private IP and verify that the application daemon is running:

```bash
# Test MongoDB (Default Port 27017)
ssh ec2-user@<MONGODB_PRIVATE_IP> "netstat -lntp | grep 27017"
# Expected Output: tcp 0 0 0.0.0.0:27017 0.0.0.0:* LISTEN <pid>/mongod

# Test Redis (Default Port 6379)
ssh ec2-user@<REDIS_PRIVATE_IP> "netstat -lntp | grep 6379"
# Expected Output: tcp 0 0 0.0.0.0:6379 0.0.0.0:* LISTEN <pid>/redis-server

# Test MySQL (Default Port 3306)
ssh ec2-user@<MYSQL_PRIVATE_IP> "netstat -lntp | grep 3306"
# Expected Output: tcp 0 0 0.0.0.0:3306 0.0.0.0:* LISTEN <pid>/mysqld

# Test RabbitMQ (Default Port 5672)
ssh ec2-user@<RABBITMQ_PRIVATE_IP> "netstat -lntp | grep 5672"
# Expected Output: tcp 0 0 0.0.0.0:5672 0.0.0.0:* LISTEN <pid>/beam.smp
```

---

### 4. Validating Route53 Private DNS Resolution
From inside the Bastion host or any instance in the VPC:

```bash
# Resolve private DNS hostnames
dig +short mongodb-dev.aitechapp.fun
dig +short redis-dev.aitechapp.fun
dig +short mysql-dev.aitechapp.fun
dig +short rabbitmq-dev.aitechapp.fun

# Check connection directly using hostname:
nc -zv mongodb-dev.aitechapp.fun 27017
nc -zv redis-dev.aitechapp.fun 6379
```

---

### 5. Clean Up (Teardown Script)
> [!IMPORTANT]
> Always destroy infrastructure in **strict reverse dependency order**!

```bash
# Inside Bastion: First destroy databases
cd /home/ec2-user/roboshop-infra-dev/40-databases
terraform destroy -auto-approve

# From local workstation: Destroy remaining infrastructure layers
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev

for i in 30-bastion/ 20-sg-rules/ 10-sg/ 00-vpc/; do
  echo "========================================="
  echo "💥 Destroying Layer: $i"
  echo "========================================="

  cd $i || exit 1

  if [ ! -d ".terraform" ]; then
    terraform init
  fi

  terraform destroy -auto-approve
  if [ $? -ne 0 ]; then
    echo "❌ Error destroying layer: $i"
    exit 1
  fi

  cd ..
done
```

---

## 5. Commands & CLI Flags Reference Table

| Command / Tool | Syntax Example | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `terraform_data` | `resource "terraform_data" "db"` | Built-in resource used to trigger provisioners without external provider plugins. |
| `triggers_replace` | `triggers_replace = [aws_instance.db.id]` | Forces recreation of `terraform_data` whenever the listed attribute changes. |
| `templatefile()` | `templatefile("policy.json", { env = "dev" })` | Reads a file and renders template variables with dynamic Terraform inputs. |
| `title()` | `title("mysql")` $\rightarrow$ `"Mysql"` | Built-in string function capitalizing the first letter of each word. |
| `netstat -lntp` | `netstat -lntp` | Displays all listening (`-l`) numeric (`-n`) TCP (`-t`) ports with process IDs (`-p`). |
| `df -hT` | `df -hT` | Reports disk space usage in human-readable format (`-h`) with filesystem type (`-T`). |
| `terraform state list` | `terraform state list \| grep iam` | Queries and filters all resources tracked in the current state file. |
| `terraform destroy -target` | `terraform destroy -target=aws_iam_role.mysql` | Surgically destroys a single specific resource without impacting the rest of the stack. |

---

## 6. Official Documentation & References
- **Terraform `terraform_data` Resource**: [developer.hashicorp.com/terraform/language/resources/terraform-data](https://developer.hashicorp.com/terraform/language/resources/terraform-data)
- **Terraform Provisioners Overview**: [developer.hashicorp.com/terraform/language/resources/provisioners/syntax](https://developer.hashicorp.com/terraform/language/resources/provisioners/syntax)
- **Terraform `templatefile` Function**: [developer.hashicorp.com/terraform/language/functions/templatefile](https://developer.hashicorp.com/terraform/language/functions/templatefile)
- **AWS Route53 `aws_route53_record` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route53_record](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route53_record)
- **Ansible Running on Localhost**: [docs.ansible.com/ansible/latest/inventory_guide/connection_details.html#running-on-localhost](https://docs.ansible.com/ansible/latest/inventory_guide/connection_details.html#running-on-localhost)

---

## 7. High-Yield Interview Questions & Answers

### Q1: Why did HashiCorp introduce `terraform_data` in Terraform 1.4 to replace `null_resource`?
**Answer**:
1. **Core Built-in**: `null_resource` required downloading an external provider plugin (`hashicorp/null`) from the Terraform Registry during `terraform init`. `terraform_data` is built directly into Terraform core with zero provider download overhead.
2. **Type Flexibility with `triggers_replace`**: `null_resource` triggers only accepted string maps (`map(string)`). `triggers_replace` accepts any data type (lists, objects, booleans, resource IDs).
3. **Clean Lifecycle Semantics**: `terraform_data` adheres strictly to standard Terraform resource lifecycle semantics, storing arbitrary calculated values across state transitions without needing dummy resources.

---

### Q2: Compare `user_data` vs Provisioners (`remote-exec`): When should you choose one over the other?
**Answer**:
- **Use `user_data` when**:
  - You need simple, static, fire-and-forget initialization (e.g., expanding disks, installing standard monitoring agents like CloudWatch/Datadog).
  - You do not want Terraform to block or fail if an external mirror is slow.
  - Instances are deployed in an Auto Scaling Group (ASG) where Terraform is not present during scale-out events.
- **Use Provisioners (`remote-exec`) when**:
  - You require **synchronous feedback** during `terraform apply`. If configuration fails, the pipeline must immediately halt.
  - You want to stream Ansible or configuration output directly to CI/CD logs.
  - Upstream resources require subsequent configuration actions before downstream resources can proceed.

---

### Q3: What happens when a provisioner fails during `terraform apply`?
**Answer**:
- If a provisioner fails (returns a non-zero exit code), Terraform marks the resource as **tainted** (or in Terraform $\ge$ 0.14, flags it for replacement during the next plan).
- Terraform does not roll back the created EC2 instance; instead, the instance remains running in AWS, but Terraform marks it as corrupted because its configuration did not complete successfully.
- On the next `terraform apply`, Terraform will destroy the tainted instance and create a brand-new one to retry the provisioner.

---

### Q4: How does Ansible execute on `localhost` without an inventory file?
**Answer**:
When Ansible is invoked on the managed machine itself via:
```bash
ansible-playbook -e component=mongodb -e env=dev roboshop.yaml
```
Ansible recognizes that the target is the local node. In `roboshop.yaml`, the target play specifies `hosts: localhost` and `connection: local`. Ansible bypasses SSH entirely, executing Python modules and shell commands directly through local system calls with root privileges (`become: yes`).

---

## 8. Production Mistakes & Troubleshooting Guide

### 1. Multi-Layer Loop Missing Trailing Slashes & Error Trapping
- **Problem**: Executing `for i in 00-vpc/ 10-sg/ 20-sg-rules 30-bastion/; do cd $i; terraform apply -auto-approve; cd ..; done` failed because `20-sg-rules` missed a trailing slash, and without checking `$?`, failures in one layer cascaded into subsequent layers.
- **Fix**: Always enforce directory validation (`cd $i || exit 1`) and verify exit codes (`if [ $? -ne 0 ]; then exit 1; fi`).

---

### 2. Playbook Filename Typo in `bootstrap.sh`
- **Error**:
  ```text
  terraform_data.mongodb (remote-exec): ERROR! the playbook: roboshop.ymal could not be found
  ```
- **Cause**: Typo in file extension (`.ymal` instead of `.yaml`).
- **Fix**: Verify exact spelling in `bootstrap.sh`:
  ```bash
  ansible-playbook -e component=$component -e env=$environment roboshop.yaml
  ```

---

### 3. Collision with Pre-existing IAM Role / Instance Profile
- **Error**:
  ```text
  Error: adding IAM Role (arn:aws:iam::...:role/Roboshop-Dev-Mysql) to IAM Instance Profile (roboshop-dev-mysql):
  operation error IAM: AddRoleToInstanceProfile, StatusCode: 400, api error ValidationError: The specified value for roleName is invalid.
  ```
- **Cause**: The IAM Role `Roboshop-Dev-Mysql` was previously created manually in the AWS Console. When Terraform tried to create the role or bind it to an instance profile, AWS returned a conflict.
- **Fix**:
  1. Inspect state:
     ```bash
     terraform state list | grep iam
     ```
  2. Surgically destroy state references or delete the conflicting AWS Console role:
     ```bash
     terraform destroy -target=aws_iam_role.mysql
     terraform destroy -target=aws_iam_instance_profile.mysql
     ```

---

## 9. Session Metadata & Timestamps

- **Database Provisioning & S3/EC2/R53 Access Pillars**: `00:00 - 20:00`
- **`null_resource` vs `terraform_data` & Provisioner Triggers**: `20:00 - 45:00`
- **Terraform-Ansible Integration via `bootstrap.sh`**: `45:00 - 01:10:00`
- **Route53 Private DNS Automation & Layer Execution**: `01:10:00 - 01:28:42`
- **Session Q&A**: `01:28:42`

### Doubts & AI Clarification Link
- [Session 40 AI Clarification Chat](https://chat.z.ai/c/session-40-clarification)