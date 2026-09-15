# Friday, 13 March 2026
# Session 42 - Application Load Balancers (ALB), Target Groups, Health Checks, Host/Path Routing & Immutable Golden AMI Architecture
## Comprehensive Class Notes, Multi-Tier Infrastructure Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [Core Architecture of an Application Load Balancer (ALB)](#core-architecture-of-an-application-load-balancer-alb)
   - [Target Group Health Checks: The Mechanics of Availability](#target-group-health-checks-the-mechanics-of-availability)
   - [Routing Strategies: Host-Based Routing vs Context/Path-Based Routing](#routing-strategies-host-based-routing-vs-contextpath-based-routing)
   - [The Rolling Update Strategy & Simultaneous Version Serving](#the-rolling-update-strategy--simultaneous-version-serving)
   - [The Immutable Golden AMI Pattern](#the-immutable-golden-ami-pattern)
   - [Deconstructing the HTTP 503 "Service Temporarily Unavailable" Error](#deconstructing-the-http-503-service-temporarily-unavailable-error)
   - [Console Walkthrough: Manual ALB & Target Group Creation](#console-walkthrough-manual-alb--target-group-creation)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct File Links](#directory-structure--direct-file-links)
   - [Architectural Diagrams](#architectural-diagrams)
     - [A. ALB Host-Based Microsegmentation Routing Topology](#a-alb-host-based-microsegmentation-routing-topology)
     - [B. Immutable Golden AMI Baking & Auto Scaling Lifecycle](#b-immutable-golden-ami-baking--auto-scaling-lifecycle)
     - [C. Target Group Health Check Evaluation State Machine](#c-target-group-health-check-evaluation-state-machine)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Layer 6: Internal Application Load Balancer (`roboshop-infra-dev/50-backend-alb/`)](#layer-6-internal-application-load-balancer-roboshop-infra-dev50-backend-alb)
     - [`main.tf`](#maintf-in-50-backend-alb)
   - [Layer 7: Catalogue Microservice & Immutable AMI (`roboshop-infra-dev/60-catalogue/`)](#layer-7-catalogue-microservice--immutable-ami-roboshop-infra-dev60-catalogue)
     - [`main.tf` - Temporary Builder Instance & Ansible Execution](#maintf---temporary-builder-instance--ansible-execution)
     - [`main.tf` - EC2 State & AMI Baking](#maintf---ec2-state--ami-baking)
     - [`main.tf` - Target Group & Launch Template](#maintf---target-group--launch-template)
     - [`main.tf` - Auto Scaling Group & Scaling Policy](#maintf---auto-scaling-group--scaling-policy)
     - [`main.tf` - Listener Rule & Builder Teardown](#maintf---listener-rule--builder-teardown)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Testing the Initial Backend ALB Fixed-Response](#1-testing-the-initial-backend-alb-fixed-response)
   - [2. Manual Verification: Catalogue Service & Health Endpoint](#2-manual-verification-catalogue-service--health-endpoint)
   - [3. Registering Targets & Overcoming the HTTP 503 Error](#3-registering-targets--overcoming-the-http-503-error)
   - [4. Automated Deployment via Terraform (`50-backend-alb` & `60-catalogue`)](#4-automated-deployment-via-terraform-50-backend-alb--60-catalogue)
   - [5. Clean Up (Teardown)](#5-clean-up-teardown)
5. [Commands & CLI Flags Reference Table](#5-commands--cli-flags-reference-table)
6. [Official Documentation & References](#6-official-documentation--references)
7. [High-Yield Interview Questions & Answers](#7-high-yield-interview-questions--answers)
8. [Production Mistakes & Troubleshooting Guide](#8-production-mistakes--troubleshooting-guide)
9. [Session Metadata & Timestamps](#9-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### Core Architecture of an Application Load Balancer (ALB)
An AWS Application Load Balancer (Layer 7) functions as an intelligent traffic dispatcher composed of four distinct decoupled components:
1. **Load Balancer (Delivery Manager)**: Evaluates incoming requests across availability zones and terminates incoming client sessions.
2. **Listener (Reception Desk)**: Checks for connection requests matching a specific protocol (HTTP/HTTPS) and port (80/443).
3. **Listener Rules (Routing Logic)**: Evaluates conditions (Host headers, Path patterns, HTTP headers, Query strings) in priority order to route traffic to the appropriate backend target group.
4. **Target Group (The Team)**: A logical container holding registered compute instances, containers, or IP addresses that receive forwarded requests and undergo health checks.

---

### Target Group Health Checks: The Mechanics of Availability
To ensure zero traffic is sent to dead or degraded servers, the ALB polls each target's health endpoint (e.g. `http://<PRIVATE_IP>:8080/health`):
- **Success Criteria**: HTTP status code matching `200-299`.
- **Healthy Threshold (e.g. 3 Consecutive Successes)**: A target starting in `initial` or `unhealthy` state must pass 3 consecutive health checks before the ALB will route user traffic to it.
- **Unhealthy Threshold (e.g. 2 Consecutive Failures)**: If an active target fails 2 consecutive health checks, the ALB immediately marks it `unhealthy` and removes it from the routing pool.
- **Timeout (5 seconds)**: Time the ALB waits for the server to return an HTTP response before counting the check as a failure.
- **Interval (10 seconds)**: Time between consecutive health check probes.

---

### Routing Strategies: Host-Based Routing vs Context/Path-Based Routing

Modern microservices use two primary Layer 7 routing patterns:

#### 1. Host-Based Routing (Domain / Subdomain Level)
The ALB routes requests based on the **HTTP `Host` Header**:
- **Banking Example**:
  - `retailbanking.icicibank.com` $\rightarrow$ Routes to Retail Banking Target Group.
  - `corporatebanking.icicibank.com` $\rightarrow$ Routes to Corporate Banking Target Group.
- **RoboShop Architecture**:
  - `catalogue.backend-alb-dev.aitechapp.fun` $\rightarrow$ Routes to Catalogue Target Group.
  - `user.backend-alb-dev.aitechapp.fun` $\rightarrow$ Routes to User Target Group.

#### 2. Context / Path-Based Routing (URL Path Level)
The ALB routes requests based on the **URL Request Path Pattern**:
- `icicibank.com/retailbanking` $\rightarrow$ Retail Banking Target Group.
- `icicibank.com/corporatebanking` $\rightarrow$ Corporate Banking Target Group.
- `api.roboshop.fun/api/catalogue/*` $\rightarrow$ Catalogue Target Group.
- `api.roboshop.fun/api/user/*` $\rightarrow$ User Target Group.

---

### The Rolling Update Strategy & Simultaneous Version Serving
When deploying a new release (e.g. version `v4`) to replace an existing fleet of 10 instances running `v3`:
1. **Rolling Step 1**: Launch 4 instances running `v4` and register them into the Target Group.
2. **Rolling Step 2**: Once healthy, terminate 4 old `v3` instances.
3. **Rolling Step 3**: Launch the next 4 `v4` instances $\rightarrow$ Terminate 4 `v3` instances.
4. **Rolling Step 4**: Launch final 2 `v4` instances $\rightarrow$ Terminate remaining 2 `v3` instances.

> [!IMPORTANT]
> During a rolling deployment, the application **simultaneously serves both v3 and v4 traffic** for a window of time! Backend database schemas must remain backwards-compatible to prevent crashes when older and newer application versions query the database concurrently.

---

### The Immutable Golden AMI Pattern
Rather than having every newly autoscaled EC2 instance run slow, failure-prone `yum install` and `git clone` commands at boot time, high-performance production environments use **Immutable Golden AMIs**:
1. **Builder Stage**: Launch a single temporary builder EC2 instance.
2. **Provision Stage**: Run Ansible playbooks to install runtime dependencies (NodeJS, Java, Python), clone code, and configure application version `v4`.
3. **Stop & Freeze**: Stop the builder EC2 instance (`aws_ec2_instance_state.catalogue` = "stopped") to prevent disk writes.
4. **Bake Golden AMI**: Create an Amazon Machine Image (`aws_ami_from_instance`) containing the fully installed, pre-compiled application.
5. **Launch Template**: Reference the newly baked Golden AMI in `aws_launch_template`.
6. **Auto Scaling**: ASG provisions instances using the Golden AMI in under **45 seconds** with zero runtime compilation or internet download dependencies.
7. **Cleanup**: Terminate the temporary builder instance (`aws ec2 terminate-instances`).

---

### Deconstructing the HTTP 503 "Service Temporarily Unavailable" Error
When testing an ALB endpoint (`curl http://catalogue.backend-alb-dev.aitechapp.fun/health`), encountering `HTTP 503 Service Temporarily Unavailable` indicates a specific ALB failure mode:
- **Root Cause**: The ALB received the request, matched a Listener Rule, and attempted to forward to the target group—**but the target group has no registered targets, or all registered targets are unhealthy or still in pending registration**.
- **Resolution**:
  1. Verify the instance is running and listening on port 8080 (`netstat -lntp`).
  2. Verify the instance is registered into the target group.
  3. Wait for the required consecutive healthy threshold checks (e.g. 2 successful checks $\times$ 10s interval = 20 seconds).
  4. Once AWS marks the target `Healthy`, the ALB immediately returns HTTP 200 `OK`.

---

### Console Walkthrough: Manual ALB & Target Group Creation

#### 1. Creating the Application Load Balancer
1. Open EC2 Console $\rightarrow$ **Load Balancers** $\rightarrow$ **Create load balancer**.
2. Select **Application Load Balancer (ALB)** $\rightarrow$ Click **Create**.
3. **Name**: `roboshop-dev-backend-alb`.
4. **Scheme**: Select **Internal** (Internal microservice load balancer; use **Internet-facing** only for edge web ingress).
5. **IP address type**: IPv4.
6. **Network mapping**:
   - VPC: `roboshop-dev`.
   - Mappings: Select at least 2 Availability Zones:
     - `us-east-1a`: Select `roboshop-dev-private-us-east-1a`.
     - `us-east-1b`: Select `roboshop-dev-private-us-east-1b`.
7. **Security Groups**: Select `roboshop-dev-backend-alb`.
8. **Listeners and routing**:
   - Protocol: `HTTP` | Port: `80`.
   - Default action: Select **Return fixed response**.
   - Response code: `200`.
   - Content type: `text/html`.
   - Response body: `<h1>Hi, I am from Backend HTTP ALB</h1>`.
9. Click **Create load balancer**.

#### 2. Creating the Target Group
1. EC2 Console $\rightarrow$ **Target Groups** $\rightarrow$ **Create target group**.
2. **Target type**: Select **Instances**.
3. **Target group name**: `roboshop-dev-catalogue`.
4. **Protocol / Port**: `HTTP` on port `8080`.
5. **VPC**: Select `roboshop-dev`.
6. **Health checks**:
   - Health check protocol: `HTTP`.
   - Health check path: `/health`.
7. **Advanced health check settings**:
   - Healthy threshold: `2`.
   - Unhealthy threshold: `3`.
   - Timeout: `2` seconds.
   - Interval: `10` seconds.
   - Success codes: `200-299`.
8. **Target deregistration management**:
   - Deregistration delay: Set to `60` seconds (Connection draining).
9. Click **Create target group**.

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct File Links
Click any file link below to view it directly in your IDE:

#### Project 1: Backend Load Balancer (`roboshop-infra-dev/50-backend-alb`)
- **Directory**: [`roboshop-infra-dev/50-backend-alb/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb) | [Relative Path](../../roboshop-infra-dev/50-backend-alb)
  - [`50-backend-alb/main.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb/main.tf#L1-L46) | [Relative Link](../../roboshop-infra-dev/50-backend-alb/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/50-backend-alb/main.tf)
  - [`50-backend-alb/locals.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb/locals.tf#L1-L10) | [Relative Link](../../roboshop-infra-dev/50-backend-alb/locals.tf)
  - [`50-backend-alb/parameters.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb/parameters.tf#L1-L7) | [Relative Link](../../roboshop-infra-dev/50-backend-alb/parameters.tf)

#### Project 2: Catalogue Service & Golden AMI Pipeline (`roboshop-infra-dev/60-catalogue`)
- **Directory**: [`roboshop-infra-dev/60-catalogue/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue) | [Relative Path](../../roboshop-infra-dev/60-catalogue)
  - [`60-catalogue/main.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/main.tf#L1-L211) | [Relative Link](../../roboshop-infra-dev/60-catalogue/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/60-catalogue/main.tf)
  - [`60-catalogue/bootstrap.sh`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/bootstrap.sh#L1-L12) | [Relative Link](../../roboshop-infra-dev/60-catalogue/bootstrap.sh)
  - [`60-catalogue/data.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/data.tf#L1-L32) | [Relative Link](../../roboshop-infra-dev/60-catalogue/data.tf)
  - [`60-catalogue/locals.tf`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/locals.tf#L1-L17) | [Relative Link](../../roboshop-infra-dev/60-catalogue/locals.tf)

---

### Architectural Diagrams

#### A. ALB Host-Based Microsegmentation Routing Topology
```
                     +-----------------------------------+
                     | CLIENT APPLICATION / FRONTEND UI  |
                     +-----------------------------------+
                                       │
                                       │ HTTP Host: catalogue.backend-alb-dev...
                                       ▼
                     +───────────────────────────────────+
                     | ROUTE53 WILDCARD ALIAS RECORD     |
                     | *.backend-alb-dev.aitechapp.fun   |
                     +───────────────────────────────────+
                                       │
                                       ▼
                     +───────────────────────────────────+
                     | INTERNAL BACKEND ALB (Port 80)    |
                     +───────────────────────────────────+
                                       │
                     +─────────────────┴─────────────────+
                     | LISTENER RULES EVALUATION         |
                     | Priority 10: Host = catalogue.*   |
                     | Priority 20: Host = user.*        |
                     | Default: Fixed-Response 200       |
                     +───────────────────────────────────+
                                       │
                 ┌─────────────────────┴─────────────────────┐
                 │ Priority 10 matches                       │ No rule matches
                 ▼                                           ▼
+──────────────────────────────────+       +──────────────────────────────────+
| TARGET GROUP: roboshop-dev-cat   |       | FIXED-RESPONSE ACTION            |
| Port: 8080 | Health: /health     |       | Status: 200 OK                   |
| ──────────────────────────────── |       | Body: "Hi, I am from Backend     |
| [ASG Instance 1: 10.0.11.15:8080]|       |        HTTP ALB"                 |
| [ASG Instance 2: 10.0.12.82:8080]|       +──────────────────────────────────+
+──────────────────────────────────+
```

---

#### B. Immutable Golden AMI Baking & Auto Scaling Lifecycle
```
+---------------------------------------------------------------------------------------+
| 1. Launch temporary EC2 instance: `aws_instance.catalogue`                            |
+---------------------------------------------------------------------------------------+
                                           │
                                           ▼
+---------------------------------------------------------------------------------------+
| 2. `terraform_data.catalogue` runs `bootstrap.sh` via SSH:                            |
|    - Installs NodeJS, dependencies, and application code version `v3`                 |
+---------------------------------------------------------------------------------------+
                                           │
                                           ▼
+---------------------------------------------------------------------------------------+
| 3. Clean stop: `aws_ec2_instance_state.catalogue` = "stopped"                         |
+---------------------------------------------------------------------------------------+
                                           │
                                           ▼
+---------------------------------------------------------------------------------------+
| 4. Bake Golden AMI: `aws_ami_from_instance.catalogue`                                 |
|    - Image Name: `roboshop-dev-catalogue-v3-i-012345`                                 |
+---------------------------------------------------------------------------------------+
                                           │
                                           ▼
+---------------------------------------------------------------------------------------+
| 5. Create Launch Template (`aws_launch_template.catalogue`) referencing Golden AMI    |
+---------------------------------------------------------------------------------------+
                                           │
                                           ▼
+---------------------------------------------------------------------------------------+
| 6. Attach to Auto Scaling Group (`aws_autoscaling_group.catalogue`)                   |
|    - Rolling Instance Refresh: min_healthy_percentage = 50%                           |
|    - Dynamic Target Tracking Policy: 70% Average CPU Utilization                      |
+---------------------------------------------------------------------------------------+
                                           │
                                           ▼
+---------------------------------------------------------------------------------------+
| 7. Cleanup builder: `terraform_data.catalogue_delete` runs:                           |
|    `aws ec2 terminate-instances --instance-ids ${aws_instance.catalogue.id}`          |
+---------------------------------------------------------------------------------------+
```

---

#### C. Target Group Health Check Evaluation State Machine
```
           +------------------+
           | Target Added     |
           +------------------+
                     │
                     ▼
           +------------------+
           |  Initial State   |
           +------------------+
                     │
         ┌───────────┴───────────┐
         │ 2 Consecutive Successes│ 3 Consecutive Failures
         ▼                       ▼
+──────────────────+   Failure   +──────────────────+
|  HEALTHY STATE   | ──────────> | UNHEALTHY STATE  |
| (Receives Traffic|             | (Traffic Stopped |
|  from ALB)       | <────────── |  Alert Triggered)|
+──────────────────+   2 Success +──────────────────+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Layer 6: Internal Application Load Balancer ([`roboshop-infra-dev/50-backend-alb/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb) | [Relative](../../roboshop-infra-dev/50-backend-alb))

#### [`main.tf` in `50-backend-alb/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/50-backend-alb/main.tf#L1-L46) | [Relative](../../roboshop-infra-dev/50-backend-alb/main.tf)

```hcl
1: resource "aws_lb" "backend_alb" {
2:   name               = "${var.project}-${var.environment}"
3:   internal           = true
4:   load_balancer_type = "application"
5:   security_groups    = [local.backend_alb_sg_id]
6:   subnets            = local.private_subnet_ids
7:   enable_deletion_protection = false
8:   tags = merge({ Name = "${var.project}-${var.environment}" }, local.common_tags)
9: }
10: 
11: resource "aws_lb_listener" "http" {
12:   load_balancer_arn = aws_lb.backend_alb.arn
13:   port              = "80"
14:   protocol          = "HTTP"
15: 
16:   default_action {
17:     type = "fixed-response"
18:     fixed_response {
19:       content_type = "text/html"
20:       message_body = "<h1>Hi, I am from HTTP Backend ALB</h1>"
21:       status_code  = "200"
22:     }
23:   }
24: }
25: 
26: resource "aws_route53_record" "www" {
27:   zone_id = var.zone_id
28:   name    = "*.backend-alb-${var.environment}.${var.domain_name}"
29:   type    = "A"
30:   alias {
31:     name                   = aws_lb.backend_alb.dns_name
32:     zone_id                = aws_lb.backend_alb.zone_id
33:     evaluate_target_health = true
34:   }
35: }
```

##### Line-by-Line Breakdown:
- **Lines 1-9 (`aws_lb "backend_alb"`)**: Sets up an internal Layer 7 load balancer spanning private subnets across multiple AZs.
- **Lines 11-24 (`aws_lb_listener "http"`)**: Ingests traffic on Port 80 and returns a fixed baseline response when no specific microservice rule matches.
- **Lines 26-35 (`aws_route53_record "www"`)**: Wildcard DNS alias resolving `*.backend-alb-dev.aitechapp.fun` to the ALB's canonical DNS endpoint without extra query costs.

---

### Layer 7: Catalogue Microservice & Immutable AMI ([`roboshop-infra-dev/60-catalogue/`](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue) | [Relative](../../roboshop-infra-dev/60-catalogue))

#### [`main.tf` - Temporary Builder Instance & Ansible Execution](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/main.tf#L1-L38) | [Relative](../../roboshop-infra-dev/60-catalogue/main.tf#L1-L38)

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
- Launches a temporary builder instance and triggers Ansible to configure application version `v3`.

---

#### [`main.tf` - EC2 State & AMI Baking](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/main.tf#L40-L58) | [Relative](../../roboshop-infra-dev/60-catalogue/main.tf#L40-L58)

```hcl
40: resource "aws_ec2_instance_state" "catalogue" {
41:   instance_id = aws_instance.catalogue.id
42:   state       = "stopped"
43:   depends_on  = [terraform_data.catalogue]
44: }
45: 
46: resource "aws_ami_from_instance" "catalogue" {
47:   name               = "${var.project}-${var.environment}-catalogue-${var.app_version}-${aws_instance.catalogue.id}"
48:   source_instance_id = aws_instance.catalogue.id
49:   depends_on         = [aws_ec2_instance_state.catalogue]
50:   tags = merge({ Name = "${var.project}-${var.environment}-catalogue" }, local.common_tags)
51: }
```
- **Line 42 (`state = "stopped"`)**: Shuts down the VM safely to avoid baking an image with locked database files or dirty caches.
- **Line 46 (`aws_ami_from_instance`)**: Freezes the configured operating system into an immutable Golden AMI.

---

#### [`main.tf` - Target Group & Launch Template](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/main.tf#L60-L122) | [Relative](../../roboshop-infra-dev/60-catalogue/main.tf#L60-L122)

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
- Configures Target Group health check probing `/health` on port 8080 and creates the Launch Template pointing to the baked Golden AMI.

---

#### [`main.tf` - Auto Scaling Group & Scaling Policy](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/main.tf#L124-L182) | [Relative](../../roboshop-infra-dev/60-catalogue/main.tf#L124-L182)

```hcl
124: resource "aws_autoscaling_group" "catalogue" {
125:   name                      = "${var.project}-${var.environment}-catalogue"
126:   max_size                  = 10
127:   min_size                  = 1
128:   desired_capacity          = 1
129:   health_check_type         = "ELB" # Replaces instances if ALB marks them unhealthy!
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
150: 
151:   target_tracking_configuration {
152:     predefined_metric_specification {
153:       predefined_metric_type = "ASGAverageCPUUtilization"
154:     }
155:     target_value = 70.0
156:   }
157: }
```
- **Line 129 (`health_check_type = "ELB"`)**: Crucial production setting—if the application crashes on port 8080, the ALB marks it unhealthy, and the ASG automatically terminates and replaces the instance!
- **Lines 139-143 (`instance_refresh`)**: Automates rolling updates whenever a new Golden AMI is attached to the launch template.
- **Lines 146-157 (`aws_autoscaling_policy`)**: Adds or removes instances to keep average CPU utilization pinned at 70%.

---

#### [`main.tf` - Listener Rule & Builder Teardown](file:///Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/60-catalogue/main.tf#L185-L211) | [Relative](../../roboshop-infra-dev/60-catalogue/main.tf#L185-L211)

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
- **Lines 185-199 (`aws_lb_listener_rule`)**: Connects the ALB to the Target Group using Host Header matching (`catalogue.backend-alb-dev.aitechapp.fun`).
- **Lines 201-208 (`terraform_data "catalogue_delete"`)**: Terminates the initial builder VM to save cloud costs once the Golden AMI and ASG are healthy!

---

## 4. Step-by-Step Hands-on Execution Walkthrough

All commands below are directly copyable:

### 1. Testing the Initial Backend ALB Fixed-Response
From inside the Bastion server:
```bash
# Connect to Bastion
ssh ec2-user@<BASTION_PUBLIC_IP>

# Query the internal ALB DNS directly
curl -i http://roboshop-dev-123456789.us-east-1.elb.amazonaws.com

# Or query using wildcard DNS
curl -i http://test.backend-alb-dev.aitechapp.fun

# Expected Output:
# HTTP/1.1 200 OK
# <h1>Hi, I am from HTTP Backend ALB</h1>
```

---

### 2. Manual Verification: Catalogue Service & Health Endpoint
```bash
# SSH into Catalogue instance
ssh ec2-user@<CATALOGUE_PRIVATE_IP>

# Verify application is listening locally on port 8080:
curl http://localhost:8080/health
# Expected Output: {"status":"ok"}
```

---

### 3. Registering Targets & Overcoming the HTTP 503 Error
When a listener rule forwards to an empty or unhealthy target group:

```bash
# Test through the Load Balancer before registration:
curl -i http://catalogue.backend-alb-dev.aitechapp.fun/health
# Result: HTTP/1.1 503 Service Temporarily Unavailable

# Register instance target into target group:
TARGET_GROUP_ARN=$(aws elbv2 describe-target-groups \
  --names "roboshop-dev-catalogue" \
  --query "TargetGroups[0].TargetGroupArn" \
  --output text)

aws elbv2 register-targets \
  --target-group-arn $TARGET_GROUP_ARN \
  --targets Id=<CATALOGUE_INSTANCE_ID>

# Wait 20 seconds for healthy threshold checks to pass, then re-test:
curl -i http://catalogue.backend-alb-dev.aitechapp.fun/health
# Result:
# HTTP/1.1 200 OK
# {"status":"ok"}
```

---

### 4. Automated Deployment via Terraform (`50-backend-alb` & `60-catalogue`)
```bash
# Inside Bastion:
cd /home/ec2-user/roboshop-infra-dev/50-backend-alb
terraform init
terraform apply -auto-approve

cd ../60-catalogue
terraform init
terraform apply -auto-approve

# Watch the automated pipeline bake the AMI, launch ASG, and destroy the builder!
```

---

### 5. Clean Up (Teardown)
> [!IMPORTANT]
> Destroy in reverse dependency order: `60-catalogue` $\rightarrow$ `50-backend-alb` $\rightarrow$ `40-databases` $\rightarrow$ `30-bastion` $\rightarrow$ `20-sg-rules` $\rightarrow$ `10-sg` $\rightarrow$ `00-vpc`!

```bash
cd /home/ec2-user/roboshop-infra-dev/60-catalogue
terraform destroy -auto-approve

cd /home/ec2-user/roboshop-infra-dev/50-backend-alb
terraform destroy -auto-approve
```

---

## 5. Commands & CLI Flags Reference Table

| Command / Tool | Syntax Example | Purpose & Real-World Use Case |
| :--- | :--- | :--- |
| `aws elbv2 register-targets` | `aws elbv2 register-targets --target-group-arn <ARN> --targets Id=i-123` | Manually attaches an EC2 instance to a target group. |
| `aws elbv2 describe-target-health` | `aws elbv2 describe-target-health --target-group-arn <ARN>` | Inspects whether targets are `healthy`, `unhealthy`, or `initial`. |
| `aws_ec2_instance_state` | `state = "stopped"` | Declaratively manages EC2 power state to safely prepare instances for AMI baking. |
| `aws_ami_from_instance` | `resource "aws_ami_from_instance" "ami"` | Freezes an existing EC2 instance into a bootable Golden AMI. |
| `instance_refresh` | `instance_refresh { strategy = "Rolling" }` | Automates rolling replacement of ASG instances when a new launch template version is published. |
| `target_tracking_configuration` | `target_value = 70.0` | Dynamically provisions or terminates instances to keep average CPU utilization at 70%. |

---

## 6. Official Documentation & References
- **AWS Application Load Balancer Routing Rules**: [docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html)
- **Terraform `aws_lb_target_group` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group)
- **Terraform `aws_ami_from_instance` Resource**: [registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ami_from_instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ami_from_instance)
- **AWS Auto Scaling Instance Refresh Guide**: [docs.aws.amazon.com/autoscaling/ec2/userguide/asg-instance-refresh.html](https://docs.aws.amazon.com/autoscaling/ec2/userguide/asg-instance-refresh.html)

---

## 7. High-Yield Interview Questions & Answers

### Q1: What causes an Application Load Balancer to return `HTTP 503 Service Temporarily Unavailable`?
**Answer**:
1. **Empty Target Group**: A listener rule successfully matched the incoming request and forwarded it to a target group, but the target group contains zero registered targets.
2. **All Targets Failing Health Checks**: Targets are registered, but all are failing health checks and are in the `unhealthy` state.
3. **Targets Still in `Initial` State**: Newly registered targets have not yet accumulated enough consecutive passing checks to satisfy `healthy_threshold`.

---

### Q2: Why is the Immutable Golden AMI Pattern preferred over configuring instances at boot time via User Data?
**Answer**:
- **Speed**: Booting from a Golden AMI takes $\sim$30-45 seconds, whereas running `yum update`, `dnf install`, and `git clone` during boot takes 5-10 minutes, making auto-scaling too slow to handle traffic surges.
- **Reliability**: User Data relies on external internet package repositories (npm, pip, yum). If an external mirror is down or slow during a scale-out event, instance launch fails. Golden AMIs have zero external runtime dependencies.
- **Consistency**: Guarantees identical binary representations across development, staging, and production fleets.

---

### Q3: What is the difference between `health_check_type = "EC2"` vs `health_check_type = "ELB"` in an Auto Scaling Group?
**Answer**:
- **`EC2`**: The ASG only checks if the virtual machine hardware and hypervisor are responding (system status and instance status checks). If the microservice process crashes on port 8080 but the OS kernel is alive, the ASG will **not** replace the instance.
- **`ELB`**: The ASG checks both EC2 hardware status **and** target group health checks. If the application process fails health checks on `/health`, the ALB flags it as unhealthy, and the ASG immediately terminates and replaces the instance.

---

### Q4: Why must database schemas be backwards-compatible during Rolling Deployments?
**Answer**:
In a rolling deployment, new instances (`v4`) are brought up alongside existing instances (`v3`). During the transition window, user requests are routed to both versions concurrently. If `v4` drops a column or alters a table schema in a non-backwards-compatible manner, running `v3` instances will immediately crash when attempting to query the database.

---

## 8. Production Mistakes & Troubleshooting Guide

### 1. HTTP 503 Due to Unregistered or Pending Targets
- **Problem**: Accessing microservice URLs returns `HTTP 503 Service Temporarily Unavailable`.
- **Cause**: Target group has no registered instances or targets are still satisfying `healthy_threshold`.
- **Fix**: Check status using `aws elbv2 describe-target-health`. Ensure instances listen on `0.0.0.0:8080` (not `127.0.0.1:8080`) and security groups permit ingress from the ALB security group.

---

### 2. Baking an AMI from a Running EC2 Instance
- **Problem**: Golden AMI boots with corrupted files or locks.
- **Cause**: Taking an EBS snapshot while active write operations or database writes are in progress.
- **Fix**: Always enforce `aws_ec2_instance_state` with `state = "stopped"` and add `depends_on = [aws_ec2_instance_state.catalogue]` to `aws_ami_from_instance`.

---

### 3. Missing `health_check_type = "ELB"` on Auto Scaling Groups
- **Problem**: Application crashes inside container, but ASG never replaces the broken instance.
- **Cause**: Default `health_check_type = "EC2"` only monitors VM hardware, ignoring application crashes.
- **Fix**: Set `health_check_type = "ELB"` on all production Auto Scaling Groups attached to Target Groups.

---

## 9. Session Metadata & Timestamps

- **Recently Faced Problem (Interview Question Discussion)**: `00:21:00`
- **Manual vs Terraform Load Balancer Workflow**: `00:35:00`
- **Catalogue Microservice & Route53 Alias Setup**: `01:00:00`
- **Session Q&A**: `01:22:07`

### Doubts & AI Clarification Link
- [Session 42 AI Clarification Chat](https://chat.z.ai/c/session-42-clarification)