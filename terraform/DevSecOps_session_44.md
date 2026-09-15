# Friday, 13 March 2026
# Session 44 - Auto Scaling CPU Stress Testing, Request Journey Mesh & Reusable Component Module Architecture
## Comprehensive Class Notes, Multi-Tier Infrastructure Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [The Role of `.terraform.lock.hcl` in Enterprise Version Control](#the-role-of-terraformlockhcl-in-enterprise-version-control)
   - [The 5-Hop Request Journey Through the Cloud Network Mesh](#the-5-hop-request-journey-through-the-cloud-network-mesh)
   - [Production DNS Migration Strategy: Route53 TTL Reduction](#production-dns-migration-strategy-route53-ttl-reduction)
   - [Auto Scaling Dynamics & CPU Stress Testing (`stress-ng`)](#auto-scaling-dynamics--cpu-stress-testing-stress-ng)
   - [Instance Warmup Optimization: Resolving Metrics Lag](#instance-warmup-optimization-resolving-metrics-lag)
   - [Architectural Refactoring: The `terraform-roboshop-component` Module](#architectural-refactoring-the-terraform-roboshop-component-module)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct File Links](#directory-structure--direct-file-links)
   - [Architectural Diagrams](#architectural-diagrams)
     - [A. The 5-Hop Network Request Routing Journey](#a-the-5-hop-network-request-routing-journey)
     - [B. CPU Stress Test & CloudWatch Auto Scaling Control Loop](#b-cpu-stress-test--cloudwatch-auto-scaling-control-loop)
     - [C. Reusable Component Child Module Polymorphism](#c-reusable-component-child-module-polymorphism)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Reusable Module: Component Pipeline (`terraform-roboshop-component/`)](#reusable-module-component-pipeline-terraform-roboshop-component)
     - [`locals.tf` - Dynamic Port & Path Polymorphism](#localstf---dynamic-port--path-polymorphism)
     - [`variables.tf` - Parameterized Contract](#variablestf---parameterized-contract)
     - [`main.tf` - Full Immutable Lifecycle Pipeline](#maintf---full-immutable-lifecycle-pipeline)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Initializing and Applying Catalogue from Bastion](#1-initializing-and-applying-catalogue-from-bastion)
   - [2. Simulating Heavy Traffic with `stress-ng`](#2-simulating-heavy-traffic-with-stress-ng)
   - [3. Observing CloudWatch Metrics & ASG Scale-Out](#3-observing-cloudwatch-metrics--asg-scale-out)
   - [4. Stress Cooldown & Scale-In Validation](#4-stress-cooldown--scale-in-validation)
   - [5. Clean Up (Teardown)](#5-clean-up-teardown)
5. [Commands & CLI Flags Reference Table](#5-commands--cli-flags-reference-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### The Role of `.terraform.lock.hcl` in Enterprise Version Control
The dependency lock file (`.terraform.lock.hcl`) is a first-class citizen of infrastructure-as-code and **must be committed to Git**:
- **Why it matters**: It records the exact version and cryptographic checksum hashes of every provider plugin used (e.g. `hashicorp/aws 5.82.0`).
- **The Risk of Ignoring**: If omitted from Git, when another developer or CI/CD agent pulls the code and runs `terraform init`, Terraform downloads the latest minor version available in the Terraform Registry. A breaking provider change can immediately crash builds or cause accidental resource recreations.
- **Rule of Thumb**: Wherever you clone or pull Terraform code (e.g. your laptop, Bastion server, Jenkins runner), you must run `terraform init` to read the lock file and install the exact verified provider binaries.

---

### The 5-Hop Request Journey Through the Cloud Network Mesh
When an end-user or microservice sends an HTTP request (e.g. `http://catalogue.backend-alb-dev.aitechapp.fun/health`), the traffic traverses 5 synchronized layers:

```
[Hop 1: Route53] -> [Hop 2: ALB Listener] -> [Hop 3: Rule Evaluation] -> [Hop 4: Target Group] -> [Hop 5: Health Check Filter]
```

1. **Hop 1: Route53 DNS Resolution**:
   - The client queries Route53 for `catalogue.backend-alb-dev.aitechapp.fun`.
   - Route53 evaluates the wildcard alias record `*.backend-alb-dev.aitechapp.fun` and returns the private IP addresses of the internal Application Load Balancer nodes.
2. **Hop 2: ALB Listener (Port 80/HTTP)**:
   - The client establishes a TCP connection to port 80 on the internal ALB.
3. **Hop 3: Listener Rule Evaluation**:
   - The ALB evaluates rules in strict priority order (Priority 10 $\rightarrow$ Priority 20 $\rightarrow$ Default).
   - Rule condition matches: `Host Header == catalogue.backend-alb-dev.aitechapp.fun`.
   - Action: Forward to `aws_lb_target_group.catalogue`.
4. **Hop 4: Target Group Dispatch**:
   - The target group receives the forwarded request and routes it to an instance running the Catalogue service on port 8080.
5. **Hop 5: Health Check Filter**:
   - The ALB checks its target health registry. Traffic is dispatched **strictly and exclusively** to instances currently in the `Healthy` state (passing `/health`). Dead or initializing instances receive zero requests.

---

### Production DNS Migration Strategy: Route53 TTL Reduction
In production infrastructure migrations (e.g. switching traffic from an old monolithic data center to a new AWS VPC):
- Standard DNS records typically have a **Time To Live (TTL)** of `300` seconds (5 minutes) or `86400` seconds (1 day).
- **The Problem**: If you change the record during a migration, client browsers and ISP recursive DNS resolvers will continue sending traffic to the old IP until their cached TTL expires.
- **The Best Practice**: Exactly **1 day before the scheduled migration**, reduce the Route53 record TTL to **1 second**:
  ```hcl
  ttl = "1"
  ```
- This ensures that during the cutover window, DNS caches expire almost instantaneously, allowing 100% of global traffic to redirect to the new load balancer within seconds!

---

### Auto Scaling Dynamics & CPU Stress Testing (`stress-ng`)
To validate that an Auto Scaling Group responds correctly under real-world traffic surges, DevOps engineers perform synthetic load testing using `stress-ng`:
- **The Command**:
  ```bash
  stress-ng --cpu 2 --cpu-load 80
  ```
  - Spawns 2 worker threads operating at 80% CPU load each.
- **Metric Progression**:
  1. Linux kernel reports elevated CPU utilization.
  2. The AWS hypervisor detects sustained CPU utilization above the configured 70% threshold.
  3. CloudWatch Alarm breaches the threshold and sends an SNS/API notification to the Auto Scaling Group.
  4. The ASG updates its `desired_capacity` from 1 to 2.
  5. The ASG launches a second EC2 instance using the Golden AMI in the alternate Availability Zone (`us-east-1b`).

---

### Instance Warmup Optimization: Resolving Metrics Lag
- By default, Auto Scaling Groups apply an **`estimated_instance_warmup`** of **300 seconds** (5 minutes).
- **The Issue**: During warmup, metrics from the new instance are not aggregated into the group average. Waiting 5 minutes causes severe scaling lag during rapid traffic spikes.
- **The Optimization**: By setting:
  ```hcl
  estimated_instance_warmup = 120
  ```
  The new instance is allowed to warm up its JVM/NodeJS process for 2 minutes, after which it immediately contributes to CloudWatch metric aggregation, preventing metric thrashing while maintaining responsive elasticity.

---

### Architectural Refactoring: The `terraform-roboshop-component` Module
In `60-catalogue`, writing monolithic HCL meant defining:
- `aws_instance` (Builder)
- `terraform_data` (Ansible provisioner)
- `aws_ec2_instance_state` (Stopped state)
- `aws_ami_from_instance` (Golden AMI)
- `aws_lb_target_group`
- `aws_launch_template`
- `aws_autoscaling_group`
- `aws_autoscaling_policy`
- `aws_lb_listener_rule`
- `terraform_data` (Builder termination)

Duplicating these 210 lines across 10 microservices (`user`, `cart`, `shipping`, `payment`, `frontend`, etc.) violates the **DRY (Don't Repeat Yourself)** principle.
**The Refactoring**: We created the reusable child module [`terraform-roboshop-component`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-roboshop-component) that dynamically handles both internal backend microservices (Port 8080, Backend ALB) and the public frontend web UI (Port 80, Frontend ALB) through polymorphic locals!

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct File Links
Click any file link below to view it directly in your IDE:

#### Project 1: Reusable Component Child Module (`terraform-roboshop-component`)
- **Directory**: [`terraform-roboshop-component/`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-roboshop-component) | [Relative Path](../../terraform-roboshop-component)
  - [`terraform-roboshop-component/main.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-roboshop-component/main.tf#L1-L209) | [Relative Link](../../terraform-roboshop-component/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terraform-roboshop-component/blob/main/main.tf)
  - [`terraform-roboshop-component/locals.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-roboshop-component/locals.tf#L1-L17) | [Relative Link](../../terraform-roboshop-component/locals.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terraform-roboshop-component/blob/main/locals.tf)
  - [`terraform-roboshop-component/variables.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-roboshop-component/variables.tf#L1-L28) | [Relative Link](../../terraform-roboshop-component/variables.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terraform-roboshop-component/blob/main/variables.tf)
  - [`terraform-roboshop-component/data.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-roboshop-component/data.tf#L1-L40) | [Relative Link](../../terraform-roboshop-component/data.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terraform-roboshop-component/blob/main/data.tf)

#### Project 2: Component Implementation (`roboshop-infra-dev/60-catalogue`)
- **Directory**: [`roboshop-infra-dev/60-catalogue/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue) | [Relative Path](../../roboshop-infra-dev/60-catalogue)
  - [`60-catalogue/main.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/main.tf#L1-L211) | [Relative Link](../../roboshop-infra-dev/60-catalogue/main.tf)

---

### Architectural Diagrams

#### A. The 5-Hop Network Request Routing Journey
```
                     +---------------------------------------+
                     | [Hop 1] CLIENT / BROWSER              |
                     | GET http://catalogue.backend-alb...   |
                     +---------------------------------------+
                                         │
                                         ▼
                     +───────────────────────────────────────+
                     | ROUTE53 WILDCARD ALIAS RECORD         |
                     | *.backend-alb-dev.aitechapp.fun       |
                     +───────────────────────────────────────+
                                         │
                                         ▼
                     +───────────────────────────────────────+
                     | [Hop 2] INTERNAL ALB LISTENER (Port 80)|
                     +───────────────────────────────────────+
                                         │
                                         ▼
                     +───────────────────────────────────────+
                     | [Hop 3] RULE EVALUATION (Priority 10) |
                     | Host Header == catalogue.*            |
                     +───────────────────────────────────────+
                                         │
                                         ▼
                     +───────────────────────────────────────+
                     | [Hop 4] TARGET GROUP: catalogue       |
                     | Forward to Port 8080                  |
                     +───────────────────────────────────────+
                                         │
                                         ▼
                     +───────────────────────────────────────+
                     | [Hop 5] HEALTH CHECK FILTER           |
                     | Routes only to Healthy (GET /health)  |
                     +───────────────────────────────────────+
                                         │
                                         ▼
                     +───────────────────────────────────────+
                     | BACKEND EC2 (Catalogue NodeJS Service)|
                     +───────────────────────────────────────+
```

---

#### B. CPU Stress Test & CloudWatch Auto Scaling Control Loop
```
+───────────────────────────────────────────────────────────────────────────────────+
| 1. Execute `stress-ng --cpu 2 --cpu-load 80` on Catalogue Instance                 |
+───────────────────────────────────────────────────────────────────────────────────+
                                          │
                                          ▼
+───────────────────────────────────────────────────────────────────────────────────+
| 2. CloudWatch Metric: `ASGAverageCPUUtilization` climbs to 80% (> 70% target)     |
+───────────────────────────────────────────────────────────────────────────────────+
                                          │
                                          ▼
+───────────────────────────────────────────────────────────────────────────────────+
| 3. Target Tracking Policy evaluates: Requires 2 instances to reduce average load  |
+───────────────────────────────────────────────────────────────────────────────────+
                                          │
                                          ▼
+───────────────────────────────────────────────────────────────────────────────────+
| 4. Auto Scaling Group transitions to "Updating capacity" (Desired: 1 -> 2)       |
|    - Launches Instance 2 into `us-east-1b` from Golden AMI                        |
|    - Target passes `/health` checks -> ALB begins load balancing traffic!         |
+───────────────────────────────────────────────────────────────────────────────────+
                                          │
                                          ▼
+───────────────────────────────────────────────────────────────────────────────────+
| 5. Terminate `stress-ng` -> CPU drops to 5% (< 70%)                               |
|    - ASG initiates Scale-In: Drains connections for 60s & terminates Instance 2  |
+───────────────────────────────────────────────────────────────────────────────────+
```

---

#### C. Reusable Component Child Module Polymorphism
```
                        module "catalogue" / module "frontend"
                                       │
                                       ▼
                   +───────────────────────────────────────+
                   | `terraform-roboshop-component`        |
                   +───────────────────────────────────────+
                                       │
                ┌──────────────────────┴──────────────────────┐
                │ If component == "frontend"                  │ If component == "catalogue"
                ▼                                             ▼
+──────────────────────────────────+       +──────────────────────────────────+
| FRONTEND CONFIGURATION           |       | BACKEND CONFIGURATION            |
| - Target Port: 80                |       | - Target Port: 8080              |
| - Health Check: /                |       | - Health Check: /health          |
| - ALB: Frontend ALB (Public)     |       | - ALB: Backend ALB (Internal)    |
| - Subnet: Public / Private       |       | - Subnet: Private App Subnets    |
+──────────────────────────────────+       +──────────────────────────────────+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Reusable Module: Component Pipeline ([`terraform-roboshop-component/`](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-roboshop-component) | [Relative](../../terraform-roboshop-component))

#### [`locals.tf` - Dynamic Port & Path Polymorphism](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-roboshop-component/locals.tf#L1-L17) | [Relative](../../terraform-roboshop-component/locals.tf)

```hcl
1: locals {
2:   ami_id                    = data.aws_ami.joindevops.id
3:   vpc_id                    = data.aws_ssm_parameter.vpc_id.value
4:   private_subnet_id         = split(",", data.aws_ssm_parameter.private_subnet_ids.value)[0]
5:   sg_id                     = data.aws_ssm_parameter.sg_id.value
6:   health_check_path         = var.component == "frontend" ? "/" : "/health"
7:   port_number               = var.component == "frontend" ? 80 : 8080
8:   backend_alb_listener_arn  = data.aws_ssm_parameter.backend_alb_listener_arn.value
9:   frontend_alb_listener_arn = data.aws_ssm_parameter.frontend_alb_listener_arn.value
10:   alb_listener_arn          = var.component == "frontend" ? local.frontend_alb_listener_arn : local.backend_alb_listener_arn
11:   host_header               = var.component == "frontend" ? "${var.component}-${var.environment}.${var.domain_name}" : "${var.component}.backend-alb-${var.environment}.${var.domain_name}"
12:   common_tags = {
13:     Project     = var.project
14:     Environment = var.environment
15:     Terraform   = "true"
16:   }
17: }
```

##### Line-by-Line Breakdown:
- **Line 6 (`health_check_path`)**: Dynamically sets `/` for Nginx web frontend and `/health` for backend microservices.
- **Line 7 (`port_number`)**: Selects port `80` for web UI and port `8080` for backend microservices.
- **Line 10 (`alb_listener_arn`)**: Selects the external public ALB for frontend and internal ALB for backend services.
- **Line 11 (`host_header`)**: Formats `frontend-dev.domain.com` for public access vs `catalogue.backend-alb-dev.domain.com` for internal routing.

---

#### [`variables.tf` - Parameterized Contract](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-roboshop-component/variables.tf#L1-L28) | [Relative](../../terraform-roboshop-component/variables.tf)

```hcl
1: variable "project" {
2:   default = "roboshop"
3: }
4: variable "environment" {
5:   default = "dev"
6: }
7: variable "component" {
8:   type = string # e.g. "catalogue", "user", "frontend"
9: }
10: variable "app_version" {
11:   type    = string
12:   default = "v3"
13: }
14: variable "rule_priority" {
15:   type = number # e.g. 10, 20, 30
16: }
```
- Completely generalizes the infrastructure contract so any microservice can be launched in 10 lines of root HCL!

---

#### [`main.tf` - Full Immutable Lifecycle Pipeline](file:///Users/sriramcharankolla/Desktop/DevOps/terraform-roboshop-component/main.tf#L1-L209) | [Relative](../../terraform-roboshop-component/main.tf)

```hcl
58: resource "aws_lb_target_group" "main" {
59:   name                 = "${var.project}-${var.environment}-${var.component}"
60:   port                 = local.port_number
61:   protocol             = "HTTP"
62:   vpc_id               = local.vpc_id
63:   deregistration_delay = 60
64: 
65:   health_check {
66:     healthy_threshold   = 2
67:     unhealthy_threshold = 3
68:     interval            = 10
69:     timeout             = 2
70:     path                = local.health_check_path
71:     port                = local.port_number
72:     protocol            = "HTTP"
73:     matcher             = "200-299"
74:   }
75: }
...
179: resource "aws_lb_listener_rule" "main" {
180:   listener_arn = local.alb_listener_arn
181:   priority     = var.rule_priority
182: 
183:   action {
184:     type             = "forward"
185:     target_group_arn = aws_lb_target_group.main.arn
186:   }
187: 
188:   condition {
189:     host_header {
190:       values = [local.host_header]
191:     }
192:   }
193: }
```
- Standardizes target groups, health checks, launch templates, ASG scaling policies, and listener rules across the entire enterprise ecosystem.

---

## 4. Step-by-Step Hands-on Execution Walkthrough

All commands below are directly copyable:

### 1. Initializing and Applying Catalogue from Bastion
```bash
# SSH into Bastion Host
ssh ec2-user@<BASTION_PUBLIC_IP>

# Navigate to catalogue layer
cd /home/ec2-user/roboshop-infra-dev/60-catalogue

# Re-initialize to sync lock file
terraform init -reconfigure

# Apply configuration
terraform apply -auto-approve

# Verify that the service endpoint responds with HTTP 200:
curl -i http://catalogue.backend-alb-dev.aitechapp.fun/health
```

---

### 2. Simulating Heavy Traffic with `stress-ng`
Connect to the running Catalogue EC2 instance in the private subnet:

```bash
# Connect to Catalogue instance via Bastion jump proxy
ssh ec2-user@<CATALOGUE_PRIVATE_IP>

# Install stress-ng on RHEL 9
sudo dnf install -y epel-release
sudo dnf install -y stress-ng

# Generate synthetic CPU load across 2 cores at 80% capacity
stress-ng --cpu 2 --cpu-load 80
```

---

### 3. Observing CloudWatch Metrics & ASG Scale-Out
Open a second terminal window or check AWS Management Console:
1. Open EC2 Console $\rightarrow$ **Instances** $\rightarrow$ Select `roboshop-dev-catalogue`.
2. Click **Monitoring** tab $\rightarrow$ Inspect **CPU utilization** line graph.
3. Observe CPU utilization rise above `70%`.
4. Navigate to **Auto Scaling Groups** $\rightarrow$ Select `roboshop-dev-catalogue`.
5. Click **Activity** tab:
   - Notice activity status: **Updating capacity**.
   - Event description: `Launching a new EC2 instance: i-0abcdef...`.
6. Navigate back to **Instances**:
   - A second instance `roboshop-dev-catalogue` is now running in `us-east-1b`!
   - Total active capacity has dynamically doubled to 2.

---

### 4. Stress Cooldown & Scale-In Validation
```bash
# Inside the Catalogue server: Terminate stress-ng
# Press Ctrl + C to stop the process

# Observe CPU utilization drop back to < 5%
# Within 2-3 minutes, the ASG detects low demand:
# Activity Event: "Terminating EC2 instance: i-0abcdef..."
# Capacity scales back down to 1 instance!
```

---

### 5. Clean Up (Teardown)
```bash
# From Bastion Host:
cd /home/ec2-user/roboshop-infra-dev/60-catalogue
terraform destroy -auto-approve
```

---

## 5. Commands & CLI Flags Reference Table

| Command / Tool | Syntax Example | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `stress-ng` | `stress-ng --cpu 2 --cpu-load 80` | Generates controllable, deterministic CPU workload to test auto-scaling thresholds. |
| `aws autoscaling describe-scaling-activities` | `aws autoscaling describe-scaling-activities --auto-scaling-group-name roboshop-dev-catalogue` | Returns detailed audit history of scale-out and scale-in events with reason descriptions. |
| `terraform init -reconfigure` | `terraform init -reconfigure` | Disregards existing backend cache and reconfigures provider and state dependencies cleanly. |
| `aws ec2 terminate-instances` | `aws ec2 terminate-instances --instance-ids i-123` | Decommissions the temporary builder machine after Golden AMI baking completes. |
| `estimated_instance_warmup` | `estimated_instance_warmup = 120` | Prevents premature scaling calculations while a newly launched instance initializes. |

---

## 6. Official Documentation & References
- **Terraform Dependency Lock File**: [developer.hashicorp.com/terraform/language/files/dependency-lock](https://developer.hashicorp.com/terraform/language/files/dependency-lock)
- **AWS Auto Scaling Lifecycle**: [docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-lifecycle.html](https://docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-lifecycle.html)
- **Linux `stress-ng` Manual**: [manpages.ubuntu.com/manpages/jammy/man1/stress-ng.1.html](https://manpages.ubuntu.com/manpages/jammy/man1/stress-ng.1.html)
- **AWS Route53 Aliasing Guide**: [docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)

---

## 7. High-Yield Interview Questions & Answers

### Q1: Why is it critical to commit `.terraform.lock.hcl` to source control?
**Answer**:
- The `.terraform.lock.hcl` file guarantees that every member of the engineering team and all automated CI/CD runners use the **identical cryptographic provider plugin binaries**.
- Without the lock file, running `terraform init` can download a newer patch or minor release that introduces schema deprecations or subtle behavioral discrepancies, causing state drift and production outages.

---

### Q2: How do you safely cut over production DNS traffic with zero downtime using Route53?
**Answer**:
1. Exactly 24 to 48 hours before the cutover, edit the Route53 DNS record and reduce its **TTL from 300s/86400s to 1 second**.
2. This forces public DNS resolvers worldwide to flush their cached IPs and query authoritative name servers every second.
3. During the maintenance window, update the record to point to the new Load Balancer.
4. Traffic cuts over globally within seconds with zero cached DNS persistence.
5. Once migration stability is confirmed, increase TTL back to standard production values (`300s`).

---

### Q3: What is the purpose of `estimated_instance_warmup` in an Auto Scaling Policy?
**Answer**:
- When an ASG launches a new instance to relieve high CPU, the instance requires time to initialize the OS, start application runtimes (NodeJS/Java), and join the load balancer.
- If no warmup period is set, CloudWatch continues reading the high CPU load of the pre-existing instances and repeatedly adds more instances in rapid succession, causing **over-provisioning (flapping/thrashing)**.
- `estimated_instance_warmup` instructs CloudWatch to pause evaluation until the new instance is warm and contributing to metric averages.

---

### Q4: How does a single reusable module handle both public frontend and private backend services?
**Answer**:
By utilizing **Ternary HCL Expressions in `locals.tf`**:
- Port: `var.component == "frontend" ? 80 : 8080`
- Health check path: `var.component == "frontend" ? "/" : "/health"`
- Listener ARN: `var.component == "frontend" ? local.frontend_alb_listener_arn : local.backend_alb_listener_arn`
- Subnet selection: Public subnets for frontend web ingress, private subnets for backend data processing.

---

## 8. Production Mistakes & Troubleshooting Guide

### 1. Incurring Cloud Costs from Abandoned Builder Instances
- **Problem**: Dozens of stopped builder instances accumulate across AWS accounts, consuming EBS storage costs.
- **Cause**: Forgetting to terminate the builder instance after AMI baking.
- **Fix**: Automate cleanup directly in Terraform using `terraform_data` with a `local-exec` provisioner executing `aws ec2 terminate-instances --instance-ids ${aws_instance.main.id}`.

---

### 2. Auto Scaling Group Thrashing Due to Unrealistic Target Values
- **Problem**: ASG constantly adds and removes instances every 10 minutes.
- **Cause**: Setting `target_value` too low (e.g. 30%) or warmup period too short, causing minor traffic fluctuations to trigger continuous scaling events.
- **Fix**: Set realistic target values (`70%`) and provide adequate instance warmup periods (`120s`).

---

### 3. `stress-ng` Package Missing on Minimal Linux AMIs
- **Problem**: `sudo yum install stress-ng` returns `No package stress-ng available`.
- **Cause**: `stress-ng` resides in the **EPEL (Extra Packages for Enterprise Linux)** repository on RHEL/CentOS.
- **Fix**: Enable EPEL repository first: `sudo dnf install -y epel-release && sudo dnf install -y stress-ng`.

---

## 9. Session Metadata & Timestamps

- **Terraform Dependency Lock File Deep Dive**: `00:34:10`
- **CPU Stress Test & CloudWatch Auto Scaling Validation**: `00:55:00`
- **Reusable Component Module Architecture**: `01:10:00`
- **Session Q&A**: `01:24:45`

### Doubts & AI Clarification Link
- [Session 44 AI Clarification Chat](https://chat.z.ai/c/session-44-clarification)