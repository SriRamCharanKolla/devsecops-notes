# Friday, 13 March 2026
# Session 43 - Automated Microservice Deployment, Immutable Golden AMI Pipeline, Auto Scaling Groups (ASG) & ALB Listener Rules
## Comprehensive Class Notes, Multi-Tier Infrastructure Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [The 9-Step Immutable Microservice Pipeline](#the-9-step-immutable-microservice-pipeline)
   - [The Terraform Parallel Execution Race Condition & `depends_on`](#the-terraform-parallel-execution-race-condition--depends_on)
   - [Auto Scaling Dimensions: Min, Desired, and Max Capacity](#auto-scaling-dimensions-min-desired-and-max-capacity)
   - [Multi-AZ High Availability (1a & 1b Subnet Distribution)](#multi-az-high-availability-1a--1b-subnet-distribution)
   - [Dynamic Scaling: Target Tracking Policy on CPU Utilization](#dynamic-scaling-target-tracking-policy-on-cpu-utilization)
   - [Console Walkthrough: Manual Launch Template & ASG Creation](#console-walkthrough-manual-launch-template--asg-creation)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct File Links](#directory-structure--direct-file-links)
   - [Architectural Diagrams](#architectural-diagrams)
     - [A. End-to-End 9-Step Immutable Infrastructure & ASG Lifecycle Flow](#a-end-to-end-9-step-immutable-infrastructure--asg-lifecycle-flow)
     - [B. Multi-AZ High Availability Auto Scaling Topology](#b-multi-az-high-availability-auto-scaling-topology)
     - [C. Target Tracking Scaling Policy Control Loop](#c-target-tracking-scaling-policy-control-loop)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Layer 7: Catalogue Microservice Infrastructure (`roboshop-infra-dev/60-catalogue/`)](#layer-7-catalogue-microservice-infrastructure-roboshop-infra-dev60-catalogue)
     - [1. Temporary Builder EC2 Instance & Ansible Provisioning (`main.tf:1-38`)](#1-temporary-builder-ec2-instance--ansible-provisioning-maintf1-38)
     - [2. EC2 Shutdown & Golden AMI Baking (`main.tf:40-58`)](#2-ec2-shutdown--golden-ami-baking-maintf40-58)
     - [3. Target Group & Launch Template (`main.tf:60-122`)](#3-target-group--launch-template-maintf60-122)
     - [4. Auto Scaling Group & Target Tracking Policy (`main.tf:124-182`)](#4-auto-scaling-group--target-tracking-policy-maintf124-182)
     - [5. ALB Listener Rule & Builder Cleanup (`main.tf:185-211`)](#5-alb-listener-rule--builder-cleanup-maintf185-211)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Automated Multi-Layer Upstream Deployment](#1-automated-multi-layer-upstream-deployment)
   - [2. Deploying `60-catalogue` from Bastion Server](#2-deploying-60-catalogue-from-bastion-server)
   - [3. Verifying Local Process Status & Port Binding](#3-verifying-local-process-status--port-binding)
   - [4. End-to-End HTTP Testing via Backend ALB](#4-end-to-end-http-testing-via-backend-alb)
   - [5. Clean Up (Teardown)](#5-clean-up-teardown)
5. [Commands & CLI Flags Reference Table](#5-commands--cli-flags-reference-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### The 9-Step Immutable Microservice Pipeline
Modern enterprise infrastructure avoids in-place server patching by implementing an **Immutable Golden Image Pipeline**. In `60-catalogue`, this is orchestrated through 9 sequential steps:
1. **Launch Temporary Builder EC2**: Provisions a baseline VM in a private subnet.
2. **Bootstrap Application**: Runs Ansible via `terraform_data` and `remote-exec` to install NodeJS, clone code, and configure application version `v3`.
3. **Stop Builder Instance**: Powers down the instance cleanly (`aws_ec2_instance_state`) to guarantee filesystem consistency.
4. **Bake Golden AMI**: Creates an Amazon Machine Image (`aws_ami_from_instance`) containing the compiled application.
5. **Create Target Group**: Defines the load balancing destination (`aws_lb_target_group`) on port 8080 with health checks on `/health`.
6. **Create Launch Template**: Packages the newly baked Golden AMI ID, instance type, security groups, and tags into an AWS launch template.
7. **Deploy Auto Scaling Group (ASG)**: Launches instances across Availability Zones (`us-east-1a` and `us-east-1b`) into the target group.
8. **Create ALB Listener Rule**: Maps incoming requests with Host header `catalogue.backend-alb-dev.<domain>` to the target group.
9. **Terminate Builder Instance**: Decommissions the temporary builder VM via `local-exec` to eliminate unnecessary cloud expenditure.

---

### The Terraform Parallel Execution Race Condition & `depends_on`
A critical architectural lesson occurs during step 3:
- Both `terraform_data.catalogue` (which runs the Ansible provisioner) and `aws_ec2_instance_state.catalogue` (which stops the instance) depend on `aws_instance.catalogue`.
- **The Danger**: By default, Terraform executes independent child resources **in parallel**. As soon as the EC2 instance is created, Terraform simultaneously triggers the provisioner and attempts to shut down the instance! The VM powers off while Ansible is halfway through installing packages, causing deployment failure.
- **The Solution**: We enforce strict sequential execution using explicit dependency:
  ```hcl
  resource "aws_ec2_instance_state" "catalogue" {
    instance_id = aws_instance.catalogue.id
    state       = "stopped"
    depends_on  = [terraform_data.catalogue] # Forces Terraform to wait for Ansible!
  }
  ```

---

### Auto Scaling Dimensions: Min, Desired, and Max Capacity
An Auto Scaling Group governs capacity using three boundaries:
1. **Minimum Capacity (`min_size`)**:
   - The absolute minimum number of instances that must be running at any given time (e.g. `min_size = 1` or `2`).
   - Even if CPU drops to 0%, the ASG will never scale below this number to ensure baseline availability.
2. **Desired Capacity (`desired_capacity`)**:
   - The exact number of instances the ASG launches initially or maintains right now (e.g. `desired_capacity = 2`).
   - When traffic changes, scaling policies adjust `desired_capacity` dynamically.
3. **Maximum Capacity (`max_size`)**:
   - The upper ceiling the ASG is permitted to scale up to during heavy surges (e.g. `max_size = 10`).
   - Protects the organization from runaway cloud bills if a denial-of-service (DoS) attack occurs.

---

### Multi-AZ High Availability (1a & 1b Subnet Distribution)
By supplying multiple private subnet IDs spanning distinct data centers:
```hcl
vpc_zone_identifier = [local.private_subnet_ids] # us-east-1a and us-east-1b
```
The Auto Scaling Group automatically balances instances evenly across Availability Zones. If AWS suffers an outage in `us-east-1a`, instances in `us-east-1b` continue serving traffic seamlessly through the Application Load Balancer.

---

### Dynamic Scaling: Target Tracking Policy on CPU Utilization
Rather than writing complex step-scaling alarms, we configure a **Target Tracking Scaling Policy**:
- Metric: `ASGAverageCPUUtilization`
- Target Value: `70.0%`
- How it works:
  - If average CPU utilization exceeds 70%, CloudWatch automatically signals the ASG to launch additional instances.
  - If average CPU utilization drops below 70%, the ASG gracefully drains and terminates excess instances until the metric stabilizes back at 70%.

---

### Console Walkthrough: Manual Launch Template & ASG Creation

#### 1. Creating AWS Launch Template Manually
1. Open EC2 Console $\rightarrow$ **Launch Templates** $\rightarrow$ **Create launch template**.
2. **Name**: `roboshop-dev-catalogue`.
3. **Description**: `A dev catalogue server for roboshop`.
4. **AMI**: Go to **My AMIs** $\rightarrow$ **Owned by me** $\rightarrow$ Select `roboshop-dev-catalogue`.
5. **Instance type**: `t3.micro`.
6. **Key pair**: Select **Don't include in launch template**.
7. **Network settings**:
   - Subnet: `roboshop-dev-private-us-east-1a`.
   - Security groups: Select `roboshop-dev-catalogue`.
8. Click **Create launch template**.

#### 2. Creating AWS Auto Scaling Group Manually
1. EC2 Console $\rightarrow$ **Auto Scaling Groups** $\rightarrow$ **Create Auto Scaling group**.
2. **Name**: `roboshop-dev-catalogue`.
3. **Launch template**: Select `roboshop-dev-catalogue` (Version: `Latest(1)`). Click **Next**.
4. **Network**:
   - VPC: `roboshop-dev`.
   - Availability Zones & Subnets: Select both `private-us-east-1a` and `private-us-east-1b`. Click **Next**.
5. **Load balancing**:
   - Select **Attach to an existing load balancer**.
   - Select **Choose from your load balancer target groups** $\rightarrow$ `roboshop-dev-catalogue`.
   - **Health check type**: Turn on **ELB health checks**.
   - **Health check grace period**: `120` seconds. Click **Next**.
6. **Group size & scaling policies**:
   - Desired capacity: `2`.
   - Minimum capacity: `1`.
   - Maximum capacity: `10`.
   - Scaling policies: Select **Target tracking scaling policy**.
   - Metric type: **Average CPU utilization** | Target value: `70`.
   - Instance warmup: `120` seconds. Click **Next**.
7. Review settings and click **Create Auto Scaling group**.

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct File Links
Click any file link below to view it directly in your IDE:

#### Project: Microservice & Golden AMI Foundation (`roboshop-infra-dev/60-catalogue`)
- **Directory**: [`roboshop-infra-dev/60-catalogue/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue) | [Relative Path](../../roboshop-infra-dev/60-catalogue)
  - [`60-catalogue/main.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/main.tf#L1-L211) | [Relative Link](../../roboshop-infra-dev/60-catalogue/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/60-catalogue/main.tf)
  - [`60-catalogue/bootstrap.sh`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/bootstrap.sh#L1-L12) | [Relative Link](../../roboshop-infra-dev/60-catalogue/bootstrap.sh) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/60-catalogue/bootstrap.sh)
  - [`60-catalogue/variables.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/variables.tf#L1-L27) | [Relative Link](../../roboshop-infra-dev/60-catalogue/variables.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/60-catalogue/variables.tf)
  - [`60-catalogue/data.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/data.tf#L1-L32) | [Relative Link](../../roboshop-infra-dev/60-catalogue/data.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/60-catalogue/data.tf)
  - [`60-catalogue/locals.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/locals.tf#L1-L17) | [Relative Link](../../roboshop-infra-dev/60-catalogue/locals.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/60-catalogue/locals.tf)

#### Project: Upstream Load Balancer (`roboshop-infra-dev/50-backend-alb`)
- **Directory**: [`roboshop-infra-dev/50-backend-alb/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb) | [Relative Path](../../roboshop-infra-dev/50-backend-alb)
  - [`50-backend-alb/main.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb/main.tf#L1-L46) | [Relative Link](../../roboshop-infra-dev/50-backend-alb/main.tf)

---

### Architectural Diagrams

#### A. End-to-End 9-Step Immutable Infrastructure & ASG Lifecycle Flow
```
+───────────────────────────────────────────────────────────────────────────────────────────+
| 1. Temporary Builder EC2 (`aws_instance.catalogue`)                                       |
+───────────────────────────────────────────────────────────────────────────────────────────+
                                              │
                                              ▼
+───────────────────────────────────────────────────────────────────────────────────────────+
| 2. Ansible Bootstrap (`terraform_data.catalogue`) -> Configures NodeJS & App v3          |
+───────────────────────────────────────────────────────────────────────────────────────────+
                                              │
                                              │ depends_on = [terraform_data.catalogue]
                                              ▼
+───────────────────────────────────────────────────────────────────────────────────────────+
| 3. Safe Shutdown: `aws_ec2_instance_state.catalogue` = "stopped"                          |
+───────────────────────────────────────────────────────────────────────────────────────────+
                                              │
                                              ▼
+───────────────────────────────────────────────────────────────────────────────────────────+
| 4. Bake Golden AMI: `aws_ami_from_instance.catalogue`                                     |
+───────────────────────────────────────────────────────────────────────────────────────────+
                                              │
                                              ▼
+───────────────────────────────────────────────────────────────────────────────────────────+
| 5. Create Target Group: `aws_lb_target_group.catalogue` (Port 8080, /health)              |
+───────────────────────────────────────────────────────────────────────────────────────────+
                                              │
                                              ▼
+───────────────────────────────────────────────────────────────────────────────────────────+
| 6. Create Launch Template: `aws_launch_template.catalogue` with Baked Golden AMI          |
+───────────────────────────────────────────────────────────────────────────────────────────+
                                              │
                                              ▼
+───────────────────────────────────────────────────────────────────────────────────────────+
| 7. Auto Scaling Group: `aws_autoscaling_group.catalogue`                                  |
|    - Min: 1 | Desired: 1 (or 2) | Max: 10                                                 |
|    - Health Check Type: ELB | Rolling Refresh                                             |
+───────────────────────────────────────────────────────────────────────────────────────────+
                                              │
                                              ▼
+───────────────────────────────────────────────────────────────────────────────────────────+
| 8. ALB Listener Rule: `aws_lb_listener_rule.catalogue` (Priority 10 -> Forward to TG)     |
+───────────────────────────────────────────────────────────────────────────────────────────+
                                              │
                                              ▼
+───────────────────────────────────────────────────────────────────────────────────────────+
| 9. Cost Cleanup: `terraform_data.catalogue_delete` -> `aws ec2 terminate-instances`       |
+───────────────────────────────────────────────────────────────────────────────────────────+
```

---

#### B. Multi-AZ High Availability Auto Scaling Topology
```
+───────────────────────────────────────────────────────────────────────────────────+
| INTERNAL APPLICATION LOAD BALANCER (`roboshop-dev-backend-alb`)                   |
| Evaluates Rule: Host catalogue.backend-alb-dev.aitechapp.fun                      |
+───────────────────────────────────────────────────────────────────────────────────+
                                          │
                                          │ Forwards to Target Group: `roboshop-dev-catalogue`
                                          ▼
+───────────────────────────────────────────────────────────────────────────────────+
| AUTO SCALING GROUP: `roboshop-dev-catalogue`                                      |
|                                                                                   |
|   +───────────────────────────────────+   +───────────────────────────────────+   |
|   | PRIVATE SUBNET AZ-1A (10.0.11.0)  |   | PRIVATE SUBNET AZ-1B (10.0.12.0)  |   |
|   |                                   |   |                                   |   |
|   |   Catalogue Instance 1            |   |   Catalogue Instance 2            |   |
|   |   Port 8080 (Healthy)             |   |   Port 8080 (Healthy)             |   |
|   |   IP: 10.0.11.84                  |   |   IP: 10.0.12.195                 |   |
|   +───────────────────────────────────+   +───────────────────────────────────+   |
+───────────────────────────────────────────────────────────────────────────────────+
```

---

#### C. Target Tracking Scaling Policy Control Loop
```
                                 CloudWatch Metric:
                           ASGAverageCPUUtilization
                                       │
                         ┌─────────────┴─────────────┐
                         │                           │
                   CPU > 70.0%                 CPU < 70.0%
                         │                           │
                         ▼                           ▼
            +─────────────────────────+ +─────────────────────────+
            |      SCALE OUT EVENT    | |       SCALE IN EVENT    |
            | Launches +1 EC2 from    | | Drains connection (60s) |
            | Golden AMI Launch Temp  | | and terminates instance |
            +─────────────────────────+ +─────────────────────────+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Layer 7: Catalogue Microservice Infrastructure ([`roboshop-infra-dev/60-catalogue/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue) | [Relative](../../roboshop-infra-dev/60-catalogue))

#### 1. Temporary Builder EC2 Instance & Ansible Provisioning ([`main.tf:1-38`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/main.tf#L1-L38))

```hcl
1: resource "aws_instance" "catalogue" {
2:   ami                    = local.ami_id
3:   instance_type          = "t3.micro"
4:   subnet_id              = local.private_subnet_id
5:   vpc_security_group_ids = [local.catalogue_sg_id]
6:   tags = merge({ Name = "${var.project}-${var.environment}-catalogue" }, local.common_tags)
7: }
8: 
9: resource "terraform_data" "catalogue" {
10:   triggers_replace = [aws_instance.catalogue.id]
11: 
12:   connection {
13:     type     = "ssh"
14:     user     = "ec2-user"
15:     password = "DevOps321"
16:     host     = aws_instance.catalogue.private_ip
17:   }
18: 
19:   provisioner "file" {
20:     source      = "bootstrap.sh"
21:     destination = "/tmp/bootstrap.sh"
22:   }
23: 
24:   provisioner "remote-exec" {
25:     inline = [
26:       "chmod +x /tmp/bootstrap.sh",
27:       "sudo sh /tmp/bootstrap.sh catalogue ${var.environment} ${var.app_version}"
28:     ]
29:   }
30: }
```
- Launches the initial temporary builder machine and runs Ansible with `app_version=v3` over SSH.

---

#### 2. EC2 Shutdown & Golden AMI Baking ([`main.tf:40-58`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/main.tf#L40-L58))

```hcl
40: resource "aws_ec2_instance_state" "catalogue" {
41:   instance_id = aws_instance.catalogue.id
42:   state       = "stopped"
43:   depends_on  = [terraform_data.catalogue] # Resolves the parallel execution race condition!
44: }
45: 
46: resource "aws_ami_from_instance" "catalogue" {
47:   name               = "${var.project}-${var.environment}-catalogue-${var.app_version}-${aws_instance.catalogue.id}"
48:   source_instance_id = aws_instance.catalogue.id
49:   depends_on         = [aws_ec2_instance_state.catalogue]
50:   tags = merge({ Name = "${var.project}-${var.environment}-catalogue" }, local.common_tags)
51: }
```
- **Line 43 (`depends_on = [terraform_data.catalogue]`)**: Guarantees the instance is not stopped until Ansible finishes completely.
- **Line 46 (`aws_ami_from_instance`)**: Bakes the immutable Golden AMI with the exact instance ID and version in its name.

---

#### 3. Target Group & Launch Template ([`main.tf:60-122`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/main.tf#L60-L122))

```hcl
60: resource "aws_lb_target_group" "catalogue" {
61:   name                 = "${var.project}-${var.environment}-catalogue"
62:   port                 = 8080
63:   protocol             = "HTTP"
64:   vpc_id               = local.vpc_id
65:   deregistration_delay = 60
66: 
67:   health_check {
68:     healthy_threshold   = 2
69:     unhealthy_threshold = 3
70:     interval            = 10
71:     timeout             = 2
72:     path                = "/health"
73:     port                = 8080
74:     protocol            = "HTTP"
75:     matcher             = "200-299"
76:   }
77: }
78: 
79: resource "aws_launch_template" "catalogue" {
80:   name                                 = "${var.project}-${var.environment}-catalogue"
81:   image_id                             = aws_ami_from_instance.catalogue.id
82:   instance_initiated_shutdown_behavior = "terminate"
83:   instance_type                        = "t3.micro"
84:   vpc_security_group_ids               = [local.catalogue_sg_id]
85:   update_default_version               = true
86: }
```
- Sets up Target Group health checks probing `/health` every 10 seconds and creates the Launch Template pointing to the baked Golden AMI.

---

#### 4. Auto Scaling Group & Target Tracking Policy ([`main.tf:124-182`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/main.tf#L124-L182))

```hcl
124: resource "aws_autoscaling_group" "catalogue" {
125:   name                      = "${var.project}-${var.environment}-catalogue"
126:   max_size                  = 10
127:   min_size                  = 1
128:   desired_capacity          = 1
129:   health_check_type         = "ELB"
130:   health_check_grace_period = 120
131:   target_group_arns         = [aws_lb_target_group.catalogue.arn]
132:   vpc_zone_identifier       = [local.private_subnet_id]
133: 
134:   launch_template {
135:     id      = aws_launch_template.catalogue.id
136:     version = "$Latest"
137:   }
138: 
139:   instance_refresh {
140:     strategy = "Rolling"
141:     preferences { min_healthy_percentage = 50 }
142:     triggers = ["launch_template"]
143:   }
144: }
145: 
146: resource "aws_autoscaling_policy" "catalogue" {
147:   autoscaling_group_name = aws_autoscaling_group.catalogue.name
148:   name                   = "${var.project}-${var.environment}-catalogue"
149:   policy_type            = "TargetTrackingScaling"
150:   estimated_instance_warmup = 120
151: 
152:   target_tracking_configuration {
153:     predefined_metric_specification {
154:       predefined_metric_type = "ASGAverageCPUUtilization"
155:     }
156:     target_value = 70.0
157:   }
158: }
```
- Launches instances into the Target Group, attaches ELB health checks, and configures automatic scaling at 70% CPU utilization.

---

#### 5. ALB Listener Rule & Builder Cleanup ([`main.tf:185-211`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/main.tf#L185-L211))

```hcl
185: resource "aws_lb_listener_rule" "catalogue" {
186:   listener_arn = local.backend_alb_listener_arn
187:   priority     = 10
188: 
189:   action {
190:     type             = "forward"
191:     target_group_arn = aws_lb_target_group.catalogue.arn
192:   }
193: 
194:   condition {
195:     host_header {
196:       values = ["catalogue.backend-alb-${var.environment}.${var.domain_name}"]
197:     }
198:   }
199: }
200: 
201: resource "terraform_data" "catalogue_delete" {
202:   triggers_replace = [aws_instance.catalogue.id]
203:   depends_on       = [aws_autoscaling_policy.catalogue]
204: 
205:   provisioner "local-exec" {
206:     command = "aws ec2 terminate-instances --instance-ids ${aws_instance.catalogue.id}"
207:   }
208: }
```
- Routes traffic from the ALB to the Catalogue Target Group on Priority 10 and terminates the builder VM once the ASG is live.

---

## 4. Step-by-Step Hands-on Execution Walkthrough

All commands below are directly copyable:

### 1. Automated Multi-Layer Upstream Deployment
```bash
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev

for i in 00-vpc/ 10-sg/ 20-sg-rules/ 30-bastion/ 50-backend-alb/; do
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

### 2. Deploying `60-catalogue` from Bastion Server
```bash
# Connect to Bastion
ssh ec2-user@<BASTION_PUBLIC_IP>

# Navigate to catalogue layer
cd /home/ec2-user/roboshop-infra-dev/60-catalogue

# Pull latest code and apply
git pull
terraform init
terraform apply -auto-approve
```

---

### 3. Verifying Local Process Status & Port Binding
```bash
# Check if Catalogue systemd service is active:
ssh ec2-user@<CATALOGUE_PRIVATE_IP> 'systemctl status catalogue'

# Check if application is listening on port 8080:
ssh ec2-user@<CATALOGUE_PRIVATE_IP> 'netstat -lntp | grep 8080'

# Test local health endpoint:
ssh ec2-user@<CATALOGUE_PRIVATE_IP> 'curl http://localhost:8080/health'
# Expected Output: {"status":"ok"}
```

---

### 4. End-to-End HTTP Testing via Backend ALB
From inside the Bastion host:
```bash
# Health Check Endpoint through the Load Balancer:
curl -i http://catalogue.backend-alb-dev.aitechapp.fun/health
# Expected Output:
# HTTP/1.1 200 OK
# Content-Type: application/json
# {"status":"ok"}

# Catalogue Categories Endpoint:
curl -i http://catalogue.backend-alb-dev.aitechapp.fun/categories
# Expected Output:
# HTTP/1.1 200 OK
# [{"id":"1","name":"Electronics",...}]
```

---

### 5. Clean Up (Teardown)
> [!IMPORTANT]
> Destroy in reverse order: `60-catalogue` $\rightarrow$ `50-backend-alb` $\rightarrow$ `40-databases` $\rightarrow$ `30-bastion` $\rightarrow$ `20-sg-rules` $\rightarrow$ `10-sg` $\rightarrow$ `00-vpc`!

```bash
cd /home/ec2-user/roboshop-infra-dev/60-catalogue
terraform destroy -auto-approve
```

---

## 5. Commands & CLI Flags Reference Table

| Command / Tool | Syntax Example | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `aws autoscaling describe-auto-scaling-groups` | `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names roboshop-dev-catalogue` | Queries status, healthy instances, and target group attachments. |
| `aws autoscaling start-instance-refresh` | `aws autoscaling start-instance-refresh --auto-scaling-group-name roboshop-dev-catalogue` | Manually triggers a rolling update across all instances in an ASG. |
| `aws elbv2 describe-listener-rules` | `aws elbv2 describe-listener-rules --listener-arn <ARN>` | Lists all rules, priorities, and conditions configured on a listener. |
| `systemctl status catalogue` | `systemctl status catalogue` | Inspects systemd daemon health, memory, and recent journal log entries. |
| `depends_on` | `depends_on = [terraform_data.catalogue]` | Meta-argument enforcing strict sequential execution in the Terraform DAG. |

---

## 6. Official Documentation & References
- **AWS Auto Scaling Groups**: [docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-groups.html](https://docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-groups.html)
- **AWS Target Tracking Scaling Policies**: [docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html)
- **Terraform `aws_autoscaling_group` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/autoscaling_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/autoscaling_group)
- **Terraform `aws_launch_template` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/launch_template](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/launch_template)

---

## 7. High-Yield Interview Questions & Answers

### Q1: What happens if you omit `depends_on = [terraform_data.catalogue]` between `aws_ec2_instance_state` and `terraform_data`?
**Answer**:
- Terraform builds its Directed Acyclic Graph (DAG) based on resource references.
- Since both `aws_ec2_instance_state.catalogue` and `terraform_data.catalogue` reference `aws_instance.catalogue.id`, Terraform treats them as siblings with no dependency between them and executes them **in parallel**.
- As soon as the EC2 instance boots up, Terraform sends the AWS API call to stop the instance while simultaneously initiating the SSH connection to run Ansible.
- The instance shuts down mid-provisioning, breaking the bootstrap process. Specifying `depends_on = [terraform_data.catalogue]` guarantees Terraform waits until the Ansible provisioner successfully exits before executing the shutdown.

---

### Q2: How does an Auto Scaling Group decide when to scale out and scale in using Target Tracking?
**Answer**:
- Under Target Tracking (e.g. `ASGAverageCPUUtilization = 70%`), AWS automatically manages two CloudWatch Alarms:
  1. **High Alarm**: Triggers when average CPU $> 70\%$ for a consecutive period $\rightarrow$ Calls ASG to add instances.
  2. **Low Alarm**: Triggers when average CPU $< 70\%$ for a sustained period $\rightarrow$ Calls ASG to remove instances.
- The policy uses proportional mathematics: If current CPU is 90% across 2 instances ($180\%$ total), it calculates that 3 instances are needed ($180\% / 3 = 60\% \le 70\%$) and scales directly to 3 instances.

---

### Q3: Why is `health_check_grace_period` essential in Auto Scaling Groups?
**Answer**:
When an ASG launches a new instance, the operating system takes time to boot, initialize network interfaces, and start the application daemon (NodeJS, Java). If `health_check_grace_period` is set to 0, the ALB immediately flags the new instance as `Unhealthy` because port 8080 is not yet listening, causing the ASG to terminate and recreate the instance in an endless boot-crash loop. A grace period of `120` seconds ensures health checks are ignored until the service has fully warmed up.

---

### Q4: Explain the difference between `min_size`, `desired_capacity`, and `max_size`.
**Answer**:
- `min_size`: Hard floor; the ASG will never scale below this value even during complete zero-traffic idle periods.
- `desired_capacity`: Target number of active instances running at this moment; modified dynamically by scaling policies or manually by operators.
- `max_size`: Hard ceiling; the ASG will never scale above this value regardless of how high CPU or traffic spikes, preventing runaway cloud costs.

---

## 8. Production Mistakes & Troubleshooting Guide

### 1. Terraform Parallel Shutdown Race Condition
- **Problem**: Builder instance stops unexpectedly before Ansible finishes running.
- **Cause**: Missing explicit `depends_on = [terraform_data.catalogue]` on `aws_ec2_instance_state`.
- **Fix**: Always enforce explicit dependency between the provisioner and the instance shutdown resource.

---

### 2. Forgetting `update_default_version = true` on Launch Template
- **Problem**: Applying Terraform creates a new launch template version with the new Golden AMI, but the ASG continues launching instances with the old AMI version!
- **Cause**: By default, ASG points to the default template version unless configured with `$Latest` or `update_default_version = true`.
- **Fix**: Add `update_default_version = true` inside `aws_launch_template` and set `version = "$Latest"` inside `aws_autoscaling_group`.

---

### 3. Missing Security Group Ingress on Port 8080 from ALB
- **Problem**: ASG instances boot successfully, but remain permanently `Unhealthy` in the Target Group.
- **Cause**: Catalogue security group does not allow inbound TCP 8080 traffic from `local.backend_alb_sg_id`.
- **Fix**: Ensure `20-sg-rules` includes an ingress rule authorizing port 8080 with `source_security_group_id = local.backend_alb_sg_id`.

---

## 9. Session Metadata & Timestamps

- **Catalogue Instance & Provisioner Configuration**: `00:00 - 30:00`
- **Immutable AMI Bake & Launch Template Setup**: `30:00 - 55:00`
- **Auto Scaling Group Multi-AZ Setup & Scaling Policy**: `55:00 - 01:20:00`
- **Session Q&A**: `01:28:31`

### Doubts & AI Clarification Link
- [Session 43 AI Clarification Chat](https://chat.z.ai/c/session-43-clarification)