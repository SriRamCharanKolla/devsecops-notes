# Saturday, 28 March 2026
# Session 46 - SG Rules Completed, Roboshop Terraform Component, 90-Components Fleet Orchestration & NGINX Reverse Proxy Teardown
## Comprehensive Class Notes, Multi-Tier Infrastructure Teardown & Hands-on Guide

---

## Table of Contents
1. [Class Running Notes & Foundational Concepts](#1-class-running-notes--foundational-concepts)
   - [Original Running Notes & Teacher Analogies](#original-running-notes--teacher-analogies)
   - [PKI Trust Chain: Private Key, Certificate, and CA Bundle Architecture](#pki-trust-chain-private-key-certificate-and-ca-bundle-architecture)
   - [The 3-Point Production Failure Prevention Checklist](#the-3-point-production-failure-prevention-checklist)
   - [Multi-Layer Re-Orchestration with Bash Compounding](#multi-layer-re-orchestration-with-bash-compounding)
   - [Database Network Isolation & Security Group Chaining](#database-network-isolation--security-group-chaining)
   - [Bastion as the Trusted Execution Engine for Stateful Datastores](#bastion-as-the-trusted-execution-engine-for-stateful-datastores)
   - [Fleet Orchestration with Reusable Modules (`90-components`)](#fleet-orchestration-with-reusable-modules-90-components)
   - [Provider Lock File (`.terraform.lock.hcl`) Strategy](#provider-lock-file-terraformlockhcl-strategy)
   - [The NGINX Reverse Proxy Port 80 vs 8080 Routing Teardown](#the-nginx-reverse-proxy-port-80-vs-8080-routing-teardown)
2. [Workspace Projects Mapping & Architecture](#2-workspace-projects-mapping--architecture)
   - [Directory Structure & Direct Workspace File Links](#directory-structure--direct-workspace-file-links)
   - [Architectural Diagrams](#architectural-diagrams)
     - [A. PKI Certificate Chain Hierarchy & Asymmetric Security](#a-pki-certificate-chain-hierarchy--asymmetric-security)
     - [B. Multi-Layer Infrastructure Re-orchestration Pipeline](#b-multi-layer-infrastructure-re-orchestration-pipeline)
     - [C. Bastion Host Private Subnet Deployment Journey](#c-bastion-host-private-subnet-deployment-journey)
     - [D. Microservice Fleet Orchestration via Git-Sourced Module](#d-microservice-fleet-orchestration-via-git-sourced-module)
     - [E. NGINX Reverse Proxy Port Routing Dilemma (Port 80 vs 8080)](#e-nginx-reverse-proxy-port-routing-dilemma-port-80-vs-8080)
3. [End-to-End Line-by-Line Code Teardown](#3-end-to-end-line-by-line-code-teardown)
   - [Layer 1: Security Group Rules for Databases (`roboshop-infra-dev/20-sg-rules/`)](#layer-1-security-group-rules-for-databases-roboshop-infra-dev20-sg-rules)
     - [`main.tf`](#maintf-in-20-sg-rules)
     - [`data.tf`](#datatf-in-20-sg-rules)
     - [`locals.tf`](#localstf-in-20-sg-rules)
   - [Layer 2: Microservice Fleet Definition (`roboshop-infra-dev/90-components/`)](#layer-2-microservice-fleet-definition-roboshop-infra-dev90-components)
     - [`provider.tf`](#providertf-in-90-components)
     - [`variables.tf`](#variablestf-in-90-components)
     - [`main.tf`](#maintf-in-90-components)
   - [Layer 3: Polymorphic Module Logic (`terraform-roboshop-component/`)](#layer-3-polymorphic-module-logic-terraform-roboshop-component)
     - [`locals.tf`](#localstf-in-terraform-roboshop-component)
     - [`main.tf`](#maintf-in-terraform-roboshop-component)
   - [Layer 4: NGINX Reverse Proxy Template (`ansible-roboshop-roles-tf/`)](#layer-4-nginx-reverse-proxy-template-ansible-roboshop-roles-tf)
     - [`nginx.conf.j2`](#nginxconfj2-in-ansible-roboshop-roles-tf)
4. [Step-by-Step Hands-on Execution Walkthrough](#4-step-by-step-hands-on-execution-walkthrough)
   - [1. Re-initialize Foundation Layers via Compound Bash](#1-re-initialize-foundation-layers-via-compound-bash)
   - [2. Update and Apply Interservice Database Security Rules](#2-update-and-apply-interservice-database-security-rules)
   - [3. Deploy Bastion Host & Establish Private VPC Bridge](#3-deploy-bastion-host--establish-private-vpc-bridge)
   - [4. Clone Repository & Provision Databases from Inside Bastion](#4-clone-repository--provision-databases-from-inside-bastion)
   - [5. Publish and Version Reusable Component Module](#5-publish-and-version-reusable-component-module)
   - [6. Configure 90-Components & Lock File Synchronization](#6-configure-90-components--lock-file-synchronization)
   - [7. Pull and Apply 90-Components Fleet from Bastion](#7-pull-and-apply-90-components-fleet-from-bastion)
   - [8. Troubleshoot & Fix NGINX Reverse Proxy Port 8080 Issue](#8-troubleshoot--fix-nginx-reverse-proxy-port-8080-issue)
5. [AWS Console Step-by-Step Guides & Architectural Deep Dives](#5-aws-console-step-by-step-guides--architectural-deep-dives)
   - [How to Inspect ALB Listener Rules and Priorities](#how-to-inspect-alb-listener-rules-and-priorities)
   - [How to Trace Ingress Rules Across Security Groups in VPC Console](#how-to-trace-ingress-rules-across-security-groups-in-vpc-console)
6. [Commands & CLI Flags Reference Table](#6-commands--cli-flags-reference-table)
7. [Official Documentation & References](#7-official-documentation--references)
8. [High-Yield Interview Questions & Answers](#8-high-yield-interview-questions--answers)
9. [Production Mistakes & Troubleshooting Guide](#9-production-mistakes--troubleshooting-guide)
10. [Session Metadata & Timestamps](#10-session-metadata--timestamps)

---

## 1. Class Running Notes & Foundational Concepts

### Original Running Notes & Teacher Analogies
*Captured directly from classroom whiteboards, interactive debugging sessions, and architectural discussions:*

1. **The TLS Certificate Chain Formula**:
   - `private key`
   - `certificate`
   - `ca_bundle`
   - Formula:
     $$\text{certificate} + \text{ca\_bundle} = \text{final\_certificate\_chain} \longrightarrow \text{public}$$
   - The **Certificate** and the **CA Bundle** together form the full public certificate chain transmitted over the wire to connecting clients.
   - The **Private Key** is kept strictly secret on the server (or AWS KMS / ACM hardware security module) and is never transmitted over the network.
2. **The 3 Common Production Failure Points**:
   When launching multi-tier cloud infrastructure, mistakes almost always occur in three specific areas:
   1. **Check SG rules**: Missing or incorrect security group ingress/egress rules (e.g., forgotten database ports, wrong source security group, or restrictive CIDR blocks).
   2. **Ansible Roles service files**: Syntax errors, incorrect environment variables, or wrong working directories inside systemd unit files (`/etc/systemd/system/*.service`).
   3. **Ansible Roles host URLs & DNS**: Incorrect endpoints or port mismatches (e.g., hardcoding `:8080` when communicating through an ALB listener that expects Port 80).
3. **Compound Bash Automation for Layer Re-orchestration**:
   ```bash
   for in in 00-vpc/ 10-sg/ 20-sg-rules/ 50-backend-alb/ 70-acm/ 80-fronend-alb/; do cd $I; terraform init -reconfigure; terraform apply -auto-approve; cd ..;done
   ```
   - Enables fast, sequential state synchronization across all immutable layers when recovering from account teardowns or migrating environments.
4. **Bastion Host as the Trusted Intra-VPC Provisioner**:
   - Database instances (MongoDB, MySQL, Redis, RabbitMQ) live in private subnets with no public IP addresses.
   - Therefore, local workstations cannot directly run Ansible playbooks against them.
   - Solution: Launch `30-bastion/`, SSH into the public IP (`ssh ec2-user@<bastion-public-IP>`), clone `roboshop-infra-dev` directly inside the Bastion instance, and run `terraform init`, `terraform plan`, and `terraform apply -auto-approve` inside `40-databases/`.
5. **Reusable Microservice Module (`terraform-roboshop-component`)**:
   - Instead of writing separate Terraform code for Catalogue, Cart, User, Shipping, Payment, and Frontend, we extract common patterns into a single reusable child module.
   - Sourced via remote Git URL (`git::https://github.com/SriRamCharanKolla/terraform-roboshop-component.git`).
   - Sourced in `90-components/` using a dynamic `for_each` map.
6. **The Provider Lock File (`.terraform.lock.hcl`) Discipline**:
   - Always run `terraform init` locally to generate the lock file, then commit and push it to Git.
   - On the Bastion host, pull the repo and run `terraform init`. Terraform reads the committed lock file, ensuring exact provider binary compatibility between macOS and Linux.
7. **The NGINX Port 8080 Routing Bug**:
   - Frontend web server failed to communicate with backend microservices when using `proxy_pass http://<service-host>:8080/;` in `nginx.conf`.
   - **Root Cause**: The Backend Application Load Balancer listener listens on standard HTTP Port 80! Requesting `:8080` against the ALB listener caused immediate connection timeouts (`HTTP 502 Bad Gateway`).
   - **Fix**: Remove `:8080` from all backend URLs in `nginx.conf.j2`. Traffic hits the ALB on Port 80, and the ALB forwards it to backend target groups on Port 8080!

---

### PKI Trust Chain: Private Key, Certificate, and CA Bundle Architecture
Modern public key cryptography relies on a multi-tiered trust model:
- **Private Key**: Generated first on the server via cryptographic math (RSA 2048/4096 or ECC P-256). It is the mathematical counterpart to the public key.
- **Server Certificate (Leaf)**: Contains the server identity (Common Name, Subject Alternative Names) and public key, signed by an Intermediate CA.
- **CA Bundle (Chain)**: Contains the intermediate certificate(s) connecting the leaf certificate back to a trusted Root CA.
- **Client Verification**: When a client connects via HTTPS, the server sends the full bundle (`leaf + intermediate`). The client checks if the intermediate is signed by a Root CA pre-installed in the OS/browser trust store.

---

### The 3-Point Production Failure Prevention Checklist
```
+───────────────────────────────────────────────────────────────────────────────────────+
|                      ENTERPRISE PRODUCTION DEBUGGING CHECKLIST                       |
+───────────────────────────────────────────────────────────────────────────────────────+

  1. SECURITY GROUP RULES (Network Ingress):
     ├── Is the source security group correctly specified?
     ├── Is the target port open (e.g. 27017 for Mongo, 6379 for Redis, 3306 for MySQL)?
     └── Is traffic allowed from Bastion for management and Ansible provisioning?

  2. ANSILE ROLES SERVICE FILES (Systemd Unit Files):
     ├── Does /etc/systemd/system/<component>.service specify the correct ExecStart path?
     ├── Are environment variables (MONGO_URL, REDIS_HOST, CART_PORT) properly injected?
     └── Does the application user (roboshop) have file ownership over /app?

  3. ANSIBLE ROLES HOST URLs & PORTS:
     ├── Are microservices communicating via Route53 DNS names?
     ├── Do backend requests point to Port 80 (ALB listener) or Port 8080?
     └── Is NGINX proxy_pass forwarding clean URIs without unwanted port attachments?
```

---

### Multi-Layer Re-Orchestration with Bash Compounding
When multiple decoupled layers exist (`00-vpc`, `10-sg`, `20-sg-rules`, `50-backend-alb`, `70-acm`, `80-frontend-alb`), executing Terraform manually directory by directory is error-prone. A structured compound loop:
```bash
for layer in 00-vpc/ 10-sg/ 20-sg-rules/ 50-backend-alb/ 70-acm/ 80-frontend-alb/; do
  echo ">>> Reconfiguring and applying: $layer <<<"
  cd "$layer"
  terraform init -reconfigure
  terraform apply -auto-approve
  cd ..
done
```
This guarantees deterministic state reconciliation without manual intervention.

---

### Database Network Isolation & Security Group Chaining
In zero-trust architecture, database servers must never allow broad subnet CIDR ranges. Each database port accepts traffic **only** from the specific microservice security group that consumes it:
- `MongoDB (27017)`: Accepts traffic only from `catalogue_sg_id` and `user_sg_id`.
- `Redis (6379)`: Accepts traffic only from `user_sg_id` and `cart_sg_id`.
- `MySQL (3306)`: Accepts traffic only from `shipping_sg_id`.
- `RabbitMQ (5672)`: Accepts traffic only from `payment_sg_id`.
- `All Databases (Port 22)`: Accepts traffic only from `bastion_sg_id` for configuration management.

---

### Bastion as the Trusted Execution Engine for Stateful Datastores
Stateful database instances cannot be provisioned via public IPs. Because the Bastion server is placed inside the public subnet of the same VPC, it possesses direct network connectivity to private database instances. Running Terraform and Ansible from inside Bastion guarantees:
1. Complete isolation of stateful data from the internet.
2. Zero exposure of SSH management ports to the world (only Bastion Port 22 is public).
3. Seamless execution of database schema loaders (e.g. `mongo < /app/schema/catalogue.js`).

---

### Fleet Orchestration with Reusable Modules (`90-components`)
Rather than duplicating hundreds of lines of Terraform code for each component, `90-components/` declares a single map:
```hcl
variable "components" {
  default = {
    catalogue = { rule_priority = 10 }
    user      = { rule_priority = 20 }
    cart      = { rule_priority = 30 }
    shipping  = { rule_priority = 40 }
    payment   = { rule_priority = 50 }
    frontend  = { rule_priority = 10 }
  }
}
```
A single `module "component"` with `for_each = var.components` creates all 6 Auto Scaling Groups, Launch Templates, Target Groups, CloudWatch Alarms, and Listener Rules automatically!

---

### Provider Lock File (`.terraform.lock.hcl`) Strategy
- When writing Terraform code locally, `terraform init` computes SHA256 hashes of provider plugins and writes `.terraform.lock.hcl`.
- In team environments or Bastion execution workflows, committing this file ensures that running `terraform init` on Bastion downloads identical provider binary versions.
- If omitted, Bastion might pull a newer provider version that introduces breaking schema changes or deprecation errors.

---

### The NGINX Reverse Proxy Port 80 vs 8080 Routing Teardown
- **The Problem**: When accessing the Roboshop Web UI, the catalogue page remained empty, and browser developer tools showed `502 Bad Gateway`.
- **The Investigation**: Examining `/etc/nginx/nginx.conf` revealed:
  ```nginx
  location /api/catalogue/ {
      proxy_pass http://catalogue-dev.backend-alb-dev.aitechapp.fun:8080/;
  }
  ```
- **The Misconception**: The developer assumed that because the Catalogue NodeJS app runs on Port 8080, NGINX must forward requests to `:8080`.
- **The Reality**: The domain `catalogue-dev.backend-alb-dev.aitechapp.fun` resolves to the **Internal Application Load Balancer**. The ALB listener listens on **Port 80**, not Port 8080.
- **The Solution**: Update NGINX to proxy to standard Port 80:
  ```nginx
  location /api/catalogue/ {
      proxy_pass http://catalogue-dev.backend-alb-dev.aitechapp.fun/;
  }
  ```
  The ALB receives the request on Port 80, evaluates the host header `catalogue-dev...`, and forwards it to the Catalogue target group instances on Port 8080.

---

## 2. Workspace Projects Mapping & Architecture

### Directory Structure & Direct Workspace File Links
Click any file link below to view it directly in your IDE:

#### Project 1: Multi-Tier Foundation (`roboshop-infra-dev`)
- **Security Group Rules Layer (`20-sg-rules/`)**: [roboshop-infra-dev/20-sg-rules/](../../roboshop-infra-dev/20-sg-rules)
  - [20-sg-rules/provider.tf](../../roboshop-infra-dev/20-sg-rules/provider.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/20-sg-rules/provider.tf)
  - [20-sg-rules/main.tf](../../roboshop-infra-dev/20-sg-rules/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/20-sg-rules/main.tf)
  - [20-sg-rules/data.tf](../../roboshop-infra-dev/20-sg-rules/data.tf)
  - [20-sg-rules/locals.tf](../../roboshop-infra-dev/20-sg-rules/locals.tf)
- **Bastion Host Layer (`30-bastion/`)**: [roboshop-infra-dev/30-bastion/](../../roboshop-infra-dev/30-bastion)
  - [30-bastion/provider.tf](../../roboshop-infra-dev/30-bastion/provider.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/30-bastion/provider.tf)
  - [30-bastion/main.tf](../../roboshop-infra-dev/30-bastion/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/30-bastion/main.tf)
- **Database Provisioning Layer (`40-databases/`)**: [roboshop-infra-dev/40-databases/](../../roboshop-infra-dev/40-databases)
  - [40-databases/provider.tf](../../roboshop-infra-dev/40-databases/provider.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/40-databases/provider.tf)
  - [40-databases/main.tf](../../roboshop-infra-dev/40-databases/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/40-databases/main.tf)
- **Microservices Orchestrator Layer (`90-components/`)**: [roboshop-infra-dev/90-components/](../../roboshop-infra-dev/90-components)
  - [90-components/provider.tf](../../roboshop-infra-dev/90-components/provider.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/90-components/provider.tf)
  - [90-components/variables.tf](../../roboshop-infra-dev/90-components/variables.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/90-components/variables.tf)
  - [90-components/main.tf](../../roboshop-infra-dev/90-components/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/roboshop-infra-dev/blob/main/90-components/main.tf)

#### Project 2: Reusable Microservice Component (`terraform-roboshop-component`)
- **Module Root**: [terraform-roboshop-component/](../../terraform-roboshop-component)
  - [terraform-roboshop-component/main.tf](../../terraform-roboshop-component/main.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terraform-roboshop-component/blob/main/main.tf)
  - [terraform-roboshop-component/locals.tf](../../terraform-roboshop-component/locals.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terraform-roboshop-component/blob/main/locals.tf)
  - [terraform-roboshop-component/variables.tf](../../terraform-roboshop-component/variables.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terraform-roboshop-component/blob/main/variables.tf)
  - [terraform-roboshop-component/data.tf](../../terraform-roboshop-component/data.tf) | [GitHub Source](https://github.com/SriRamCharanKolla/terraform-roboshop-component/blob/main/data.tf)

#### Project 3: Ansible Configuration Management (`ansible-roboshop-roles-tf`)
- **Frontend Reverse Proxy Configuration**: [ansible-roboshop-roles-tf/roles/frontend/templates/nginx.conf.j2](../../ansible-roboshop-roles-tf/roles/frontend/templates/nginx.conf.j2) | [GitHub Source](https://github.com/SriRamCharanKolla/ansible-roboshop-roles-tf/blob/main/roles/frontend/templates/nginx.conf.j2)

---

### Architectural Diagrams

#### A. PKI Certificate Chain Hierarchy & Asymmetric Security
```
+───────────────────────────────────────────────────────────────────────────────────────+
|                      PKI TRUST CHAIN vs PRIVATE KEY BOUNDARY                         |
+───────────────────────────────────────────────────────────────────────────────────────+

  [ WEB BROWSER / CLIENT ]
             │
             │ HTTPS Request (ClientHello)
             ▼
  +───────────────────────────────────────────────────────────────────────────────────+
  |              PUBLIC CERTIFICATE CHAIN TRANSMITTED BY SERVER                       |
  |                                                                                   |
  |  +─────────────────────────────────────────────────────────────────────────────+  |
  |  | 1. Server Leaf Certificate (*.aitechapp.fun)                                |  |
  |  |    - Contains Public Key + Subject Alternative Names                        |  |
  |  |    - Cryptographically signed by Intermediate CA                            |  |
  |  +─────────────────────────────────────────────────────────────────────────────+  |
  |                                ▲ (Signed by)                                     |
  |  +─────────────────────────────────────────────────────────────────────────────+  |
  |  | 2. Intermediate CA Certificate (ca_bundle.crt)                              |  |
  |  |    - Issued to AWS ACM / Let's Encrypt / DigiCert                           |  |
  |  |    - Cryptographically signed by Root CA                                    |  |
  |  +─────────────────────────────────────────────────────────────────────────────+  |
  |                                ▲ (Signed by)                                     |
  |  +─────────────────────────────────────────────────────────────────────────────+  |
  |  | 3. Root CA Certificate (Pre-loaded in OS / Browser Trust Store)             |  |
  |  +─────────────────────────────────────────────────────────────────────────────+  |
  +───────────────────────────────────────────────────────────────────────────────────+
                                           │
                                           │ Authenticated!
                                           ▼
  +───────────────────────────────────────────────────────────────────────────────────+
  |                       PRIVATE BOUNDARY (NEVER TRANSMITTED)                        |
  |                                                                                   |
  |  [ PRIVATE KEY (.key) ]                                                           |
  |  - Held securely inside AWS ACM Hardware Security Module (HSM)                    |
  |  - Decrypts pre-master secret to establish high-speed AES symmetric tunnel        |
  +───────────────────────────────────────────────────────────────────────────────────+
```

#### B. Multi-Layer Infrastructure Re-orchestration Pipeline
```
+---------------------------------------------------------------------------------------+
|                 10-TIER RE-ORCHESTRATION PIPELINE (SEQUENTIAL ORDER)                  |
+---------------------------------------------------------------------------------------+

  [00-vpc]            -> Provisions VPC, Subnets, Internet & NAT Gateways, Route Tables
     │
     ▼
  [10-sg]             -> Creates Base Security Groups for all tiers (Empty Shells)
     │
     ▼
  [20-sg-rules]       -> Establishes Inter-service Ingress & Interservice Chaining Rules
     │
     ▼
  [30-bastion]        -> Provisions Public Ingress Bastion Jump Host
     │
     ▼
  [40-databases]      -> Provisions Stateful Databases from Inside Bastion (MongoDB, etc.)
     │
     ▼
  [50-backend-alb]    -> Provisions Internal Backend Application Load Balancer & Listeners
     │
     ▼
  [70-acm]            -> Requests & Validates TLS Certificate (*.aitechapp.fun)
     │
     ▼
  [80-frontend-alb]   -> Provisions Public Application Load Balancer with HTTPS Listener
     │
     ▼
  [90-components]     -> Provisions Microservice Fleet ASGs, AMIs, Target Groups, Rules
```

#### C. Bastion Host Private Subnet Deployment Journey
```
+───────────────────────────────────────────────────────────────────────────────────────+
|                  BASTION-MEDIATED PRIVATE SUBNET PROVISIONING                        |
+───────────────────────────────────────────────────────────────────────────────────────+

  [ Developer Laptop ]
           │
           │ SSH :22 (Public Internet)
           ▼
  +──────────────────────────────────────────+
  | BASTION HOST (Public Subnet)             |
  | IP: 54.x.x.x                             |
  |                                          |
  | 1. git clone roboshop-infra-dev          |
  | 2. terraform init / apply in 40-databases|
  | 3. terraform apply in 90-components      |
  +──────────────────────────────────────────+
           │
           │ Private VPC Network (10.0.0.0/16)
           ├────────────────────────┬────────────────────────┐
           ▼                        ▼                        ▼
  +─────────────────+      +─────────────────+      +─────────────────+
  | MongoDB (10.0.2)|      | MySQL (10.0.2)  |      | Redis (10.0.2)  |
  | Private Subnet  |      | Private Subnet  |      | Private Subnet  |
  | Port: 27017     |      | Port: 3306      |      | Port: 6379      |
  +─────────────────+      +─────────────────+      +─────────────────+
```

#### D. Microservice Fleet Orchestration via Git-Sourced Module
```
+───────────────────────────────────────────────────────────────────────────────────────+
|               90-COMPONENTS DYNAMIC ORCHESTRATION ARCHITECTURE                        |
+───────────────────────────────────────────────────────────────────────────────────────+

  [ roboshop-infra-dev/90-components/main.tf ]
                       │
                       │ for_each = var.components
                       ▼
       ┌───────────────┬───────────────┬───────────────┬───────────────┐
       ▼               ▼               ▼               ▼               ▼
  [catalogue]       [user]          [cart]        [shipping]       [frontend]
  Priority: 10   Priority: 20    Priority: 30    Priority: 40    Priority: 10
       │               │               │               │               │
       └───────────────┴───────────────┴───────────────┴───────────────┘
                                       │
                                       │ Sources Remote Child Module:
                                       ▼
  +─────────────────────────────────────────────────────────────────────────────+
  |        git::https://github.com/.../terraform-roboshop-component.git         |
  |                                                                             |
  |  1. Temporary EC2 Builder Instance -> Bakes Golden AMI                     |
  |  2. AWS Launch Template & Target Group (Port 80 or 8080)                   |
  |  3. Auto Scaling Group (Min: 1, Max: 5, Desired: 2)                        |
  |  4. Target Tracking Auto Scaling Policy (CPU Utilization 50%)              |
  |  5. ALB Listener Rule with Assigned Priority Number                        |
  +─────────────────────────────────────────────────────────────────────────────+
```

#### E. NGINX Reverse Proxy Port Routing Dilemma (Port 80 vs 8080)
```
+───────────────────────────────────────────────────────────────────────────────────────+
|                  THE NGINX REVERSE PROXY PORT ROUTING RESOLUTION                      |
+───────────────────────────────────────────────────────────────────────────────────────+

                              [ Client Browser ]
                                      │
                                      │ HTTPS :443
                                      ▼
                           [ Public Frontend ALB ]
                                      │
                                      │ Port 80 (Target Group)
                                      ▼
                        [ NGINX Web Server on Frontend ]
                                      │
                                      │ Evaluates /api/catalogue/
                                      │
        ┌─────────────────────────────┴─────────────────────────────┐
        │                                                           │
        ▼ [ BROKEN CONFIGURATION ]                                  ▼ [ RESOLVED CONFIGURATION ]
  proxy_pass http://catalogue...:8080/;                       proxy_pass http://catalogue.../;
        │                                                           │
        ▼ (Explicitly queries Port 8080)                            ▼ (Queries standard HTTP Port 80)
  +─────────────────────────────────────────+                 +─────────────────────────────────────────+
  | Internal Backend ALB                    |                 | Internal Backend ALB                    |
  | Listener: PORT 80 ONLY                  |                 | Listener: PORT 80 ONLY                  |
  |                                         |                 |                                         |
  | ❌ CONNECTION REFUSED!                  |                 |  REQUEST ACCEPTED!                     |
  | No listener active on Port 8080         |                 | Evaluates Host: catalogue-dev...        |
  | Returns HTTP 502 Bad Gateway            |                 | Rule Priority 10 matches                |
  +─────────────────────────────────────────+                 +─────────────────────────────────────────+
                                                                            │
                                                                            │ ALB Target Group Forwards
                                                                            ▼
                                                              +─────────────────────────────────────────+
                                                              | Catalogue NodeJS App (Listens on :8080) |
                                                              +─────────────────────────────────────────+
```

---

## 3. End-to-End Line-by-Line Code Teardown

### Layer 1: Security Group Rules for Databases (roboshop-infra-dev/20-sg-rules/)
> **File**: [roboshop-infra-dev/20-sg-rules/main.tf](../../roboshop-infra-dev/20-sg-rules/main.tf)

```hcl
 1: # MongoDB Ingress from Catalogue
 2: resource "aws_security_group_rule" "mongodb_catalogue" {
 3:   type                     = "ingress"
 4:   from_port                = 27017
 5:   to_port                  = 27017
 6:   protocol                 = "tcp"
 7:   source_security_group_id = local.catalogue_sg_id
 8:   security_group_id        = local.mongodb_sg_id
 9: }
10: 
11: # MongoDB Ingress from User
12: resource "aws_security_group_rule" "mongodb_user" {
13:   type                     = "ingress"
14:   from_port                = 27017
15:   to_port                  = 27017
16:   protocol                 = "tcp"
17:   source_security_group_id = local.user_sg_id
18:   security_group_id        = local.mongodb_sg_id
19: }
20: 
21: # Redis Ingress from User
22: resource "aws_security_group_rule" "redis_user" {
23:   type                     = "ingress"
24:   from_port                = 6379
25:   to_port                  = 6379
26:   protocol                 = "tcp"
27:   source_security_group_id = local.user_sg_id
28:   security_group_id        = local.redis_sg_id
29: }
30: 
31: # MySQL Ingress from Shipping
32: resource "aws_security_group_rule" "mysql_shipping" {
33:   type                     = "ingress"
34:   from_port                = 3306
35:   to_port                  = 3306
36:   protocol                 = "tcp"
37:   source_security_group_id = local.shipping_sg_id
38:   security_group_id        = local.mysql_sg_id
39: }
40: 
41: # RabbitMQ Ingress from Payment
42: resource "aws_security_group_rule" "rabbitmq_payment" {
43:   type                     = "ingress"
44:   from_port                = 5672
45:   to_port                  = 5672
46:   protocol                 = "tcp"
47:   source_security_group_id = local.payment_sg_id
48:   security_group_id        = local.rabbitmq_sg_id
49: }
```

#### Line-by-Line Breakdown:
- **Lines 1-9 (`mongodb_catalogue`)**: Restricts MongoDB access on Port 27017 strictly to instances carrying the Catalogue security group.
- **Lines 11-19 (`mongodb_user`)**: Grants User service access to MongoDB on Port 27017 for user profile storage.
- **Lines 21-29 (`redis_user`)**: Opens Port 6379 on Redis strictly for the User service to manage authentication sessions.
- **Lines 31-39 (`mysql_shipping`)**: Restricts relational database access on Port 3306 exclusively to the Java-based Shipping service.
- **Lines 41-49 (`rabbitmq_payment`)**: Opens AMQP queue Port 5672 on RabbitMQ strictly for the Payment microservice.

---

### Layer 2: Microservice Fleet Definition (roboshop-infra-dev/90-components/)

#### variables.tf in 90-components/
> **File**: [roboshop-infra-dev/90-components/variables.tf](../../roboshop-infra-dev/90-components/variables.tf)

```hcl
 1: variable "components" {
 2:   default = {
 3:     # backend components are attaching to backend ALB
 4:     catalogue = {
 5:       rule_priority = 10
 6:     }
 7:     user = {
 8:       rule_priority = 20
 9:     }
10:     cart = {
11:       rule_priority = 30
12:     }
13:     shipping = {
14:       rule_priority = 40
15:     }
16:     payment = {
17:       rule_priority = 50
18:     }
19:     # this is attaching to frontend ALB, there is only component there
20:     frontend = {
21:       rule_priority = 10
22:     }
23:   }
24: }
```

#### main.tf in 90-components/
> **File**: [roboshop-infra-dev/90-components/main.tf](../../roboshop-infra-dev/90-components/main.tf)

```hcl
1: module "component" {
2:   for_each      = var.components
3:   source        = "git::https://github.com/SriRamCharanKolla/terraform-roboshop-component.git"
4:   component     = each.key
5:   rule_priority = each.value.rule_priority
6: }
```

#### Line-by-Line Breakdown:
- **Line 2 (`for_each = var.components`)**: Clones and initializes 6 distinct child module states.
- **Line 3 (`source = "git::https://..."`)**: Instructs Terraform to fetch the reusable module from GitHub.
- **Lines 4-5 (`component`, `rule_priority`)**: Injects the component identity and ALB evaluation priority dynamically into the child module.

---

### Layer 3: Polymorphic Module Logic (terraform-roboshop-component/)
> **File**: [terraform-roboshop-component/locals.tf](../../terraform-roboshop-component/locals.tf)

```hcl
 1: locals {
 2:   is_frontend = var.component == "frontend" ? true : false
 3:   app_port    = local.is_frontend ? 80 : 8080
 4:   
 5:   alb_listener_arn = local.is_frontend ? data.aws_ssm_parameter.frontend_alb_listener_arn.value : data.aws_ssm_parameter.backend_alb_listener_arn.value
 6:   alb_dns_name     = local.is_frontend ? data.aws_ssm_parameter.frontend_alb_dns_name.value : data.aws_ssm_parameter.backend_alb_dns_name.value
 7:   
 8:   health_check_path = local.is_frontend ? "/" : "/health"
 9: }
```

#### Line-by-Line Breakdown:
- **Line 2 (`is_frontend`)**: Evaluates whether the current module iteration represents the Web UI.
- **Line 3 (`app_port`)**: Evaluates to Port 80 for Frontend, and Port 8080 for all backend microservices.
- **Lines 5-6 (`alb_listener_arn`, `alb_dns_name`)**: Switches target listener from the internal ALB to the public-facing ALB automatically.
- **Line 8 (`health_check_path`)**: Assigns `/` for NGINX root and `/health` for backend health probes.

---

### Layer 4: NGINX Reverse Proxy Template (ansible-roboshop-roles-tf/)
> **File**: [ansible-roboshop-roles-tf/roles/frontend/templates/nginx.conf.j2](../../ansible-roboshop-roles-tf/roles/frontend/templates/nginx.conf.j2)

```nginx
50:         location /api/catalogue/ { proxy_pass http://{{ CATALOGUE_HOST }}/; }
51:         location /api/user/ { proxy_pass http://{{ USER_HOST }}/; }
52:         location /api/cart/ { proxy_pass http://{{ CART_HOST }}/; }
53:         location /api/shipping/ { proxy_pass http://{{ SHIPPING_HOST }}/; }
54:         location /api/payment/ { proxy_pass http://{{ PAYMENT_HOST }}/; }
```

#### Line-by-Line Breakdown:
- **Lines 50-54**: Proxies API traffic to the internal ALB without appending `:8080`. The request enters the ALB on Port 80, matches the host header (`{{ CATALOGUE_HOST }}`), and routes to the target group on Port 8080.

---

## 4. Step-by-Step Hands-on Execution Walkthrough

```
+───────────────────────────────────────────────────────────────────────────────────────────+
|                           THE 8-STEP PRODUCTION PLAYBOOK                                  |
+───────────────────────────────────────────────────────────────────────────────────────────+
```

### 1. Re-initialize Foundation Layers via Compound Bash
Execute this compound loop to ensure state synchronization across all immutable layers:
```bash
for layer in 00-vpc/ 10-sg/ 20-sg-rules/ 50-backend-alb/ 70-acm/ 80-frontend-alb/; do
  echo "========================================="
  echo ">>> Reconfiguring Layer: $layer"
  echo "========================================="
  cd "$layer"
  terraform init -reconfigure
  terraform apply -auto-approve
  cd ..
done
```

### 2. Update and Apply Interservice Database Security Rules
Navigate to `20-sg-rules/` and create ingress rules for MongoDB, Redis, MySQL, and RabbitMQ:
```bash
cd roboshop-infra-dev/20-sg-rules/
terraform plan
terraform apply -auto-approve
```

### 3. Deploy Bastion Host & Establish Private VPC Bridge
Provision the Bastion host in the public subnet:
```bash
cd ../30-bastion/
terraform init
terraform apply -auto-approve

# Capture Bastion Public IP:
BASTION_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=bastion-dev" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
echo "Bastion Public IP: $BASTION_IP"
```

### 4. Clone Repository & Provision Databases from Inside Bastion
Log into Bastion and run the stateful database layer:
```bash
ssh -i ~/.ssh/id_rsa ec2-user@"$BASTION_IP"

# Inside Bastion Host:
git clone https://github.com/SriRamCharanKolla/roboshop-infra-dev.git
cd roboshop-infra-dev/40-databases/
terraform init
terraform plan
terraform apply -auto-approve
```

### 5. Publish and Version Reusable Component Module
On your local developer workstation, ensure the reusable component module is pushed to GitHub:
```bash
cd /Users/sriramcharankolla/Desktop/DevOps/terraform-roboshop-component
git status
git add .
git commit -m "feat: polymorphic microservice module supporting both frontend and backend"
git push origin main
```

### 6. Configure 90-Components & Lock File Synchronization
In your local workstation terminal, initialize `90-components/` to download the Git module and generate the provider lock file:
```bash
cd /Users/sriramcharankolla/Desktop/DevOps/roboshop-infra-dev/90-components/
terraform init

# Push code along with lock file:
git add provider.tf main.tf variables.tf .terraform.lock.hcl
git commit -m "feat: declare 6-service fleet with lock file"
git push origin main
```

### 7. Pull and Apply 90-Components Fleet from Bastion
Switch back to the Bastion SSH session and deploy the fleet:
```bash
# Inside Bastion Host:
cd ~/roboshop-infra-dev/
git pull origin main
cd 90-components/
terraform init
terraform plan
terraform apply -auto-approve
```
*Note: This provisioning run takes 12 to 15 minutes to bake AMIs, configure launch templates, and launch Auto Scaling Groups across all 6 services.*

### 8. Troubleshoot & Fix NGINX Reverse Proxy Port 8080 Issue
When frontend fails with `502 Bad Gateway`:
1. Verify NGINX proxy pass configuration in `ansible-roboshop-roles-tf/roles/frontend/templates/nginx.conf.j2`.
2. Ensure no `:8080` port suffix is appended to internal ALB URLs.
3. If corrected, re-run Ansible or restart NGINX:
   ```bash
   sudo systemctl restart nginx
   ```

---

## 5. AWS Console Step-by-Step Guides & Architectural Deep Dives

### How to Inspect ALB Listener Rules and Priorities
1. Open the **AWS EC2 Console** and navigate to **Load Balancers** under *Load Balancing*.
2. Select `roboshop-dev-backend-alb`.
3. Select the **Listeners** tab and click on **HTTP: 80**.
4. Click **Manage rules**.
5. Verify the numerical priority order matches [variables.tf](../../roboshop-infra-dev/90-components/variables.tf):
   - Priority 10: `Host is catalogue-dev.backend-alb-dev.aitechapp.fun` -> Forward to `catalogue-dev` Target Group.
   - Priority 20: `Host is user-dev.backend-alb-dev.aitechapp.fun` -> Forward to `user-dev` Target Group.
   - Priority 30: `Host is cart-dev.backend-alb-dev.aitechapp.fun` -> Forward to `cart-dev` Target Group.
   - Priority 40: `Host is shipping-dev.backend-alb-dev.aitechapp.fun` -> Forward to `shipping-dev` Target Group.
   - Priority 50: `Host is payment-dev.backend-alb-dev.aitechapp.fun` -> Forward to `payment-dev` Target Group.

### How to Trace Ingress Rules Across Security Groups in VPC Console
1. Open the **AWS VPC Console** and click on **Security Groups**.
2. Filter for `roboshop-dev-mongodb`.
3. In the **Inbound rules** tab, verify that:
   - Port 27017 allows traffic from source security group `roboshop-dev-catalogue`.
   - Port 27017 allows traffic from source security group `roboshop-dev-user`.
   - Port 22 allows traffic from source security group `roboshop-dev-bastion`.
   - Port 0-65535 from `0.0.0.0/0` is **NOT present**.

---

## 6. Commands & CLI Flags Reference Table

| Command | Working Directory / Context | Purpose |
| :--- | :--- | :--- |
| `terraform init -reconfigure` | Any layer root | Re-initializes backend S3 state ignoring cached local backend configurations. |
| `terraform apply -auto-approve` | Any layer root | Deploys planned changes without requiring interactive `yes` input. |
| `ssh ec2-user@<bastion-ip>` | Local developer machine | Connects to Bastion jump host in public subnet. |
| `git pull origin main` | Bastion host (`~/roboshop-infra-dev/`) | Synchronizes Bastion workspace with latest committed configurations and lock files. |
| `curl -Iv https://dev.aitechapp.fun` | Local terminal / Browser | Verifies public SSL handshake and HTTP response headers. |
| `curl -s http://<internal-alb>/health` | Bastion / EC2 instance | Probes internal backend microservice health endpoint through the ALB listener. |
| `sudo systemctl restart nginx` | Frontend EC2 instance | Reloads updated NGINX reverse proxy routing rules. |

---

## 7. Official Documentation & References
- [AWS Application Load Balancer Listener Rules Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-update-rules.html)
- [HashiCorp Terraform Dependency Lock File (`.terraform.lock.hcl`)](https://developer.hashicorp.com/terraform/language/files/dependency-lock)
- [Terraform Module Sources (Git Repositories)](https://developer.hashicorp.com/terraform/language/modules/sources#generic-git-repository)
- [AWS Security Group Ingress Rules Reference](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html)
- [NGINX Reverse Proxy Module Guide (`ngx_http_proxy_module`)](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)

---

## 8. High-Yield Interview Questions & Answers

### Q1: Why do we commit `.terraform.lock.hcl` to Git?
> **Answer**: The lock file records the exact cryptographic checksums and versions of provider plugins selected during `terraform init`. Committing it ensures that team members, CI/CD runners, and Bastion servers download identical provider binaries, preventing breaking changes or unintended resource replacement caused by upstream provider releases.

### Q2: Why did the NGINX reverse proxy return 502 Bad Gateway when routing to `:8080`?
> **Answer**: The domain configured in `proxy_pass` pointed to an internal Application Load Balancer. The ALB listener was configured on Port 80. Requesting Port 8080 resulted in connection refused because the ALB has no listener on 8080. The ALB listener accepts traffic on Port 80, matches the host header, and forwards it to the backend instances on Port 8080 via the target group.

### Q3: Why provision database instances from inside a Bastion host rather than your local laptop?
> **Answer**: In a production VPC, database instances reside in isolated private subnets with no public IPs or internet ingress. Running Terraform with Ansible provisioners requires direct IP reachability to configure MySQL, MongoDB, Redis, and RabbitMQ. Bastion resides inside the same VPC and can reach private IP addresses directly.

### Q4: How does `for_each` in `90-components` reduce infrastructure maintenance overhead?
> **Answer**: It consolidates microservice infrastructure into a single declarative map. To add a new service (e.g. `ratings` or `recommendations`), an engineer adds one block to the map with its rule priority. Terraform automatically generates the AMI pipeline, launch template, target group, ASG, and ALB listener rule without copying HCL files.

---

## 9. Production Mistakes & Troubleshooting Guide

### Mistake 1: Hardcoding Port 8080 in NGINX Reverse Proxy
- **Symptom**: Web frontend loads UI, but clicking products yields `502 Bad Gateway`.
- **Cause**: NGINX `proxy_pass` directed to `http://catalogue.backend...:8080/`.
- **Solution**: Remove `:8080`. Target the internal ALB on standard Port 80.

### Mistake 2: Missing Interservice Security Group Ingress Rules
- **Symptom**: Catalogue microservice health check fails with database timeout.
- **Cause**: Port 27017 ingress from `catalogue_sg_id` was missing in `20-sg-rules`.
- **Solution**: Ensure [20-sg-rules/main.tf](../../roboshop-infra-dev/20-sg-rules/main.tf) explicitly allows Port 27017 from `local.catalogue_sg_id`.

### Mistake 3: Provider Version Drift Between Local Machine and Bastion
- **Symptom**: `terraform apply` succeeds on macOS but fails on Amazon Linux Bastion with `unsupported argument`.
- **Cause**: Developer machine had a newer AWS provider version and omitted `.terraform.lock.hcl` from Git.
- **Solution**: Always track `.terraform.lock.hcl` in Git and run `terraform init` on Bastion.

---

## 10. Session Metadata & Timestamps
- **Session Date**: Saturday, 28 March 2026
- **Topic**: Security Group Rules Completed, Roboshop Reusable Component, 90-Components Multi-Service Fleet Orchestration & NGINX Reverse Proxy Teardown
- **Timestamps**:
  - `QA Session`: `01:20:00`
- **Mistakes & Learning**:
  - Check SG rules for interservice database connectivity.
  - Check Ansible roles service unit files for environment variables and working directories.
  - Check Ansible roles host URLs and remove `:8080` port suffixes in NGINX proxy configs.
- **Doubts & Clarifications Link**: [DevSecOps Architecture Discussions & Clarifications](https://github.com/SriRamCharanKolla/devsecops-notes)